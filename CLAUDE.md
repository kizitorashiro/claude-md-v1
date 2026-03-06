# CLAUDE.md（インフラ設計・構築エージェントルール）

## 概要

クラウドアーキテクトがクラウドインフラをスペック駆動で設計・構築するためのルールを定義します。
スペック駆動とは、**コードより先にドキュメントで仕様・設計・タスクを定義し、承認を得てから実装を始める**アプローチです。
IaCツールはTerraformを使用します。AWS / GCP / Azure いずれにも適用できますが、**1リポジトリは1クラウドプロバイダーを対象**とします。

私たちのチームが扱うインフラはKubernetesベースの社内用システムが中心であり、クローズドなネットワーク構成を基本とする。そのため、**Kubernetesクラスターの検証**と**通信セキュリティの確認**を特に重視する。

---

## プロジェクト構造

### リポジトリ全体の構成

```
リポジトリルート
├── CLAUDE.md                        # 本ファイル（エージェントルール）
├── terraform-guidelines-base.md     # Terraform開発ガイドラインの共通テンプレート
├── docs/                            # 永続的ドキュメント（プロジェクト固有）
├── .steering/                       # 作業単位のドキュメント
└── infra/                           # Terraformコード
```

`terraform-guidelines-base.md` はプロジェクト横断で再利用する共通テンプレートです。プロジェクト開始時にこれをベースとして `docs/terraform-guidelines.md` を作成します。

### ドキュメントの分類

#### 1. 永続的ドキュメント（`docs/`）

インフラ全体の「**何を作るか**」「**どう作るか**」を定義する恒久的なドキュメント。
基本方針やアーキテクチャが変わらない限り更新されません。

- **infrastructure-requirements.md** - インフラ要件定義書
  - ビジネス要件・背景、対象クラウドプロバイダーと利用目的
  - SLA / 可用性要件、セキュリティ・コンプライアンス要件
  - コスト方針、非機能要件（スケーラビリティ、DR、バックアップ等）

- **architecture.md** - アーキテクチャ設計書
  - 全体構成図（Mermaid推奨）、コンポーネント設計
  - ネットワーク設計、セキュリティ設計（IAM方針、通信制御等）
  - 環境構成（dev / staging / prod）

- **terraform-guidelines.md** - Terraform開発ガイドライン
  - `terraform-guidelines-base.md` をベースとし、プロジェクト固有の設定を追記して作成する
  - プロジェクト固有の内容: stateバックエンド設定、Provider種類・バージョン、タグ戦略の具体値、CLIによるリソース検証コマンド例

#### 2. 作業単位のドキュメント（`.steering/[YYYYMMDD]-[作業タイトル]/`）

特定の構築・変更作業における「**今回何をするか**」を定義する一時的なドキュメント。
作業完了後のディレクトリは削除せず履歴として保持し、新しい作業では新しいディレクトリを作成します。

- **requirements.md** - 今回の作業要求（要件・制約事項・受け入れ条件）
- **design.md** - 変更内容の設計（構成変更詳細・影響範囲・実装アプローチ・ロールバック計画）
- **tasklist.md** - タスクリスト（実装タスク・進捗状況・完了条件）

**作業完了の定義:**
1. `tasklist.md` の全タスクが完了チェックされている
2. CLIによるリソース検証が完了している
3. `terraform plan` で差分がないこと（冪等性の確認）
4. `docs/` の更新要否を確認し、必要な更新が完了している

### ステアリングディレクトリの命名規則

```
.steering/[YYYYMMDD]-[作業タイトル]/
```

例: `.steering/20260301-initial-network-setup/`, `.steering/20260310-add-gke-cluster/`

### Terraformディレクトリ構成

`terraform-guidelines-base.md`「ディレクトリ構成」を参照。
`infra/modules/` 配下のサブディレクトリは `docs/architecture.md` の設計に基づき作業ごとに作成する。

---

## 開発プロセス

### 初回セットアップ時の手順

#### 1. リポジトリ初期化

```bash
git init
mkdir -p docs .steering infra/envs/dev infra/envs/staging infra/envs/prod infra/modules
```

`.gitignore` を作成し、以下を含める:
- `.terraform/` - プロバイダーキャッシュ
- `tfplan` / `*.tfplan` - planファイル
- `terraform.tfvars` / `*.auto.tfvars` - 環境変数の実値
- `*.tfstate` / `*.tfstate.*` - ローカルstateファイル

**注: `.terraform.lock.hcl` はGitに含める（コミット対象）。**

#### 2. stateバックエンドの準備

リモートバックエンド（S3 / GCS / Azure Blob）のリソースは、**Terraformで管理する対象インフラとは別に事前準備する**。
方法はプロジェクトの状況に応じてユーザーと相談して決める（手動作成、別Terraformプロジェクト、クラウドCLI等）。

#### 3. 永続的ドキュメント作成（`docs/`）

**1ファイルごとに作成後、必ず確認・承認を得てから次に進む。**

1. `docs/infrastructure-requirements.md` - インフラ要件定義書
2. `docs/architecture.md` - アーキテクチャ設計書
3. `docs/terraform-guidelines.md` - Terraform開発ガイドライン（`terraform-guidelines-base.md` をベースに作成）

#### 4. 初回実装用のステアリングファイル作成

```bash
mkdir -p .steering/[YYYYMMDD]-initial-implementation
```

**1ファイルごとに作成後、必ず確認・承認を得てから次に進む。**

```
requirements.md（承認） → design.md（承認） → tasklist.md（承認） → 実装
```

#### 5. Terraform実装・検証

`tasklist.md` に基づいて実装。検証は `docs/terraform-guidelines.md` のルールに従う。

---

### 構成追加・変更時の手順

#### 1. 影響分析

- 永続的ドキュメント（`docs/`）への影響を確認
- 「docs/ 永続ドキュメントの更新ルール」に該当する場合は先に `docs/` を更新し承認を得る

#### 2. ステアリングディレクトリ作成・ドキュメント作成

```bash
mkdir -p .steering/[YYYYMMDD]-[作業タイトル]
```

**1ファイルごとに作成後、必ず確認・承認を得てから次に進む。**

```
requirements.md（承認） → design.md（承認） → tasklist.md（承認） → 実装
```

#### 3. Terraform実装・検証

`tasklist.md` に基づいて実装。検証は `docs/terraform-guidelines.md` のルールに従う。

#### 4. 作業完了時

- 作業完了の定義（4条件）を満たしていることを確認する

---

### 既存インフラの取り込み（import）

既存リソースをTerraform管理下に置く場合:

1. 通常の作業と同様に `.steering/` でドキュメントを作成・承認
2. `design.md` にimport対象リソースの一覧とimport方法（`import` ブロック or `terraform import` コマンド）を記載
3. import操作は**必ずユーザーの承認を得てから実行する**（state操作のため）
4. import後に `terraform plan` で差分がないことを確認

---

## Claudeの行動ルール

### 承認が必要な操作

以下の操作は**必ずユーザーに内容を提示し、承認を得てから実行する**。

| 操作 | 理由 |
|---|---|
| `terraform apply` | リソース作成・変更・削除が発生するため |
| `terraform destroy` | 全リソースの破壊的削除のため |
| `terraform import` / import操作 | state操作はリスクが高いため |
| `docs/` の更新 | インフラ全体の方針に影響するため |

### 自律的に進めてよい操作

- `terraform fmt` / `terraform validate` / `terraform plan`
- `terraform init`（初回セットアップ時・プロバイダー/モジュール追加後・バックエンド変更後）
- 各クラウドCLIによる読み取り系コマンド（describe / list / get）
- `.steering/` 配下のドキュメント作成・更新
- `tasklist.md` のステータス更新

### エラー発生時の対処

- **apply失敗・plan失敗**: エラー内容を分析しユーザーに報告、修正方針を提示してから対処する。`terraform force-unlock` や `state push` などのstate操作は勝手に行わない
- **認証エラー（SSOトークン切れ等）**: ユーザーに再認証を依頼し、完了後に処理を再開する
- **想定外のリソース差分**: `plan` で意図しない削除・再作成が検出された場合はapply前にユーザーへ報告して方針を確認する

---

## ドキュメント管理ルール

### 作業前にドキュメントを作る

コードより先に `.steering/` のドキュメントを作成し、各ファイルの承認を得てから次へ進む。

### tasklist.md をリアルタイムに更新する

- タスク完了ごとにチェックボックスを更新する
- `terraform apply` の結果（作成されたリソースIDなど）も記録する

### `docs/` 永続ドキュメントの更新ルール

以下のいずれかに該当する場合、**作業開始前**に該当ドキュメントを更新し承認を得る。
また、**作業完了時**にも `docs/` への反映漏れがないか確認する。

| 変更内容 | 更新対象 |
|---|---|
| 新しいクラウドサービスの採用・廃止 | `architecture.md` |
| ネットワーク構成・セキュリティ方針の変更 | `architecture.md` |
| Terraformのディレクトリ構成・命名規則の変更 | `terraform-guidelines.md` |
| 新しい開発ルール・方針の追加 | `terraform-guidelines.md` |
| ビジネス要件・SLA・コスト方針の変更 | `infrastructure-requirements.md` |
