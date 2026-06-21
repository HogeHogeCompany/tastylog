# tastylog インフラ destroy / 復旧 ランブック

コスト削減のために**インフラを一時的に全削除（destroy）**し、後日**復旧（apply）**するための手順書です。
DB データを失わないよう、**RDS スナップショット取得 → destroy → スナップショットから復元**の流れを取ります。

- 対象: `10_infra/` の Terraform で構築したリソース
- リージョン: 東京 (`ap-northeast-1`) / プロファイル: `terraform`
- 主なリソース識別子:
  - RDS インスタンス: `tastylog-dev-mysql-standalone`
  - ECS クラスター: `tastylog-dev-webapp-cluster`
  - ECS サービス: `tastylog-dev-webapp-service`
  - S3 バックエンド（tfstate 保管）: `tastylog-bucket-at-practice`

---

## 0. 前提知識：なぜこの手順なのか

| 用語 | 意味 |
|---|---|
| **destroy** | Terraform がコードで管理する AWS リソースを**全削除**する操作。停止ではなく削除。 |
| **手動スナップショット** | RDS のある時点のデータ丸ごとのバックアップ。**インスタンスを削除しても残る**（自動スナップショットは削除と一緒に消える）。だから destroy 前に手動取得が必須。 |
| **tfstate** | Terraform が「今どのリソースを管理しているか」を記録するファイル。本プロジェクトでは S3 に保管され、destroy しても残るので再 apply にそのまま使える。 |

> 💰 **コスト感**: フル稼働で月 約$64。destroy すれば残るのは**スナップショット保管費（20GBで月 数十円程度）だけ**になり、ほぼ $0 にできる。

---

## 1. destroy 手順

### Step 1-1. RDS 手動スナップショットを取得

```bash
aws rds create-db-snapshot --db-instance-identifier tastylog-dev-mysql-standalone --db-snapshot-identifier tastylog-dev-mysql-manual-20260622 --region ap-northeast-1 --profile terraform
```

**コマンドの意味（1つずつ）:**

| 部分 | 意味 |
|---|---|
| `aws rds create-db-snapshot` | RDS の「手動スナップショットを作成する」API を呼ぶサブコマンド。 |
| `--db-instance-identifier tastylog-dev-mysql-standalone` | **どの DB インスタンスを**バックアップするかの指定。`rds.tf` の `identifier` と一致させる。 |
| `--db-snapshot-identifier tastylog-dev-mysql-manual-20260622` | 作成する**スナップショットに付ける名前**。日付を入れておくと後で識別しやすい（`manual` は手動の意。`20260622` は取得日 2026/06/22）。 |
| `--region ap-northeast-1` | 操作対象の**リージョン**（東京）。 |
| `--profile terraform` | `~/.aws/credentials` の**どの認証情報を使うか**。Terraform 用プロファイルを使用。 |

> ⚠️ スナップショット名は**アカウント内で一意**。同じ名前で2回実行するとエラーになるので、再取得時は日付や連番を変える。

### Step 1-2. スナップショット完了を待つ

```bash
aws rds wait db-snapshot-completed --db-snapshot-identifier tastylog-dev-mysql-manual-20260622 --region ap-northeast-1 --profile terraform
```

**コマンドの意味:**

| 部分 | 意味 |
|---|---|
| `aws rds wait db-snapshot-completed` | スナップショットのステータスが `available`（完了）になるまで**コマンドをブロックして待機**する。完了するとプロンプトが返る。 |
| `--db-snapshot-identifier ...` | 待機対象のスナップショット名。 |

確認したい場合は次で `Status` を見られる:

```bash
aws rds describe-db-snapshots --db-snapshot-identifier tastylog-dev-mysql-manual-20260622 --query "DBSnapshots[0].Status" --region ap-northeast-1 --profile terraform
```

- `aws rds describe-db-snapshots` … スナップショットの詳細情報を取得。
- `--query "DBSnapshots[0].Status"` … 取得結果(JSON)から**ステータスだけ**を抜き出す（JMESPath という絞り込み記法）。`available` と表示されれば完了。

### Step 1-3. destroy 実行

```bash
terraform destroy
```

**コマンドの意味:**

| 部分 | 意味 |
|---|---|
| `terraform destroy` | tfstate に記録された**全リソースの削除計画**を作り、確認後に削除する。 |

実行すると `Plan: 0 to add, 0 to change, N to destroy.` と削除対象が表示され、`Enter a value:` で **`yes`** を入力すると削除が始まる。

確認を省略して一気に実行したい場合（自動化向け・要注意）:

```bash
terraform destroy -auto-approve
```

- `-auto-approve` … 確認プロンプトをスキップする。**誤実行の危険があるため手動運用では非推奨**。

特定リソースだけ消したい場合（例: ALB と RDS だけ止めたい等の応用）:

```bash
terraform destroy -target=aws_lb.front -target=aws_db_instance.mysql_standalone
```

- `-target=...` … 指定したリソースのみを対象にする。依存関係で巻き込まれるものもあるため上級者向け。

> ⚠️ **destroy で消えるもの**
> - RDS インスタンス本体（※データは Step 1-1 のスナップショットに退避済み）
> - ECR リポジトリと中の**コンテナイメージ**（`30_app/Dockerfile` から再ビルド可能）
> - ALB / Fargate / VPC など全リソース
>
> **残るもの**: 手動スナップショット、S3 の tfstate。

---

## 2. 復旧（後日インフラを戻す）手順

### Step 2-1. `rds.tf` にスナップショット復元の設定を追加

`aws_db_instance "mysql_standalone"` ブロックに以下の1行を追加する:

```hcl
  snapshot_identifier = "tastylog-dev-mysql-manual-20260622"
```

**意味:** 「RDS を**空で作るのではなく、このスナップショットから復元して作る**」という指示。これを入れないと apply で**データの入っていない新規 DB** が立ち上がる。

> 📝 補足
> - `snapshot_identifier` を使うと、`username` / `password` / DB 名は**スナップショット時点の値**が使われる。
> - 本プロジェクトは `name = "tastylog"` を指定しているが、スナップショット復元時はスナップショット側が優先される。plan で差分や警告が出たら `name` 行を一時的にコメントアウトして調整する。

### Step 2-2. 初期化と差分確認

```bash
terraform init
terraform plan
```

**コマンドの意味:**

| コマンド | 意味 |
|---|---|
| `terraform init` | プロバイダ（AWS）や S3 バックエンドの初期化。新しい環境・久しぶりの作業時に必要。既に初期化済みなら省略可。 |
| `terraform plan` | 実際には変更せず、「これから**何が作られるか**」を表示するドライラン。`Plan: N to add` を確認する。 |

### Step 2-3. 復元実行

```bash
terraform apply
```

`Plan` 内容を確認し、`yes` で実行するとインフラ一式が再作成され、RDS はスナップショットのデータ入りで復元される。

### Step 2-4.（復元後）`snapshot_identifier` を外す

RDS が無事復元できたら、Step 2-1 で追加した行を**削除またはコメントアウト**しておく。
理由: 残したままだと次回以降の `plan` で不要な差分・再作成の引き金になり得るため。

```hcl
  # snapshot_identifier = "tastylog-dev-mysql-manual-20260622"  # 復元完了後はコメントアウト
```

### Step 2-5. アプリイメージの再投入

ECR は destroy で空になっているため、アプリを動かすにはイメージを再 push する（CI/CD パイプライン or 手動ビルド）。

```bash
# 例: ローカルからビルドして push する場合の流れ（詳細は CI 設定に従う）
# 1) ECR にログイン → 2) docker build → 3) docker tag → 4) docker push
```

---

## 3. クイックリファレンス（コマンドだけ）

```bash
# --- 停止（destroy）---
aws rds create-db-snapshot --db-instance-identifier tastylog-dev-mysql-standalone --db-snapshot-identifier tastylog-dev-mysql-manual-YYYYMMDD --region ap-northeast-1 --profile terraform
aws rds wait db-snapshot-completed --db-snapshot-identifier tastylog-dev-mysql-manual-YYYYMMDD --region ap-northeast-1 --profile terraform
terraform destroy

# --- 復旧（restore）---
# rds.tf に snapshot_identifier = "tastylog-dev-mysql-manual-YYYYMMDD" を追記してから
terraform init
terraform plan
terraform apply
# 復元確認後、snapshot_identifier をコメントアウト
```

---

## 4. よくある注意点（ハマりどころ）

| 症状 | 原因 / 対処 |
|---|---|
| スナップショット作成がエラー | 同名スナップショットが既存。名前の日付・連番を変える。 |
| destroy が RDS で止まる | `deletion_protection = true` だと消せない。本プロジェクトは `false` なので問題なし。 |
| 復元後に DB が空 | `snapshot_identifier` を付け忘れて apply した。`rds.tf` を直して再 apply（既存を一度 destroy する必要あり）。 |
| apply で `name` 関連の差分が出る | スナップショット復元時は DB 名がスナップショット側優先。`name` を一時コメントアウトして調整。 |
| アプリが 503 / 起動しない | ECR にイメージが無い（destroy で消えた）。イメージを再ビルド & push する。 |
| スナップショット保管費が気になる | 不要になったら `aws rds delete-db-snapshot --db-snapshot-identifier <名前>` で削除。 |
