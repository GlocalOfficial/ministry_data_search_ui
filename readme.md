# 省庁資料検索ツール (β版_v2)

省庁が公開する予算資料・会議資料を横断的に検索できるStreamlitアプリケーションです。

現行のアプリURL：https://ministry-data-search.streamlit.app/

## 概要

このツールは、各省庁の以下の資料を統合的に検索できます：

- **予算資料**: 概算要求、当初予算、補正予算（2013年度以降）
- **会議資料**: 各種本部・連絡会議、審議会の資料（2023年度以降）

BigQueryをバックエンドとして使用し、キーワード検索や詳細な条件絞り込みが可能です。

## 主な機能

- **キーワード検索**: AND検索/OR検索に対応
- **多次元フィルタリング**:
  - 省庁・外局選択（階層的に選択可能）
  - カテゴリ（予算/会議資料）
  - 資料形式（予算概要、議事録など）
  - 対象年度
  - 会議体名（会議資料の場合）
- **認証機能**: BigQueryベースのユーザー認証
- **検索ログ記録**: 検索履歴やログイン履歴をBigQueryに保存

## 必要な環境

- Python 3.8以上
- Google Cloud Platform (GCP) プロジェクト
- BigQuery APIの有効化
- サービスアカウントキー（権限: BigQuery管理者または適切な読み書き権限）

## セットアップ

### 1. リポジトリのクローン

```bash
git clone <repository-url>
cd <repository-name>
```

### 2. 仮想環境の作成と有効化

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
```

### 3. 依存パッケージのインストール

```bash
pip install -r requirements.txt
```

### 4. GCPサービスアカウントキーの準備

プロジェクトルートに `service_account.json` を配置してください（このファイルはGit管理外）。

### 5. BigQueryのデータ準備

以下のデータセットとテーブルを準備してください：

#### 必要なデータセット

- `rawdata_dataset`: 生データを格納するデータセット
- `config_dataset`: 設定・認証・ログを格納するデータセット

#### 必要なテーブル

**rawdata_dataset内:**
- `budget_table`: 予算資料データ
- `council_table`: 会議資料データ
- `council_list`: 会議体マスタ
※データ格納システムは別リポジトリで作成・運用

**config_dataset内:**
- `auth_table`: 認証情報テーブル
    - テーブルの構成は下記の通り
    - ※is_adminは現在使用していないが、今後UI上でユーザ改廃ができるようにする機能などを追加する想定で作成

|フィールド名|データ型|内容・説明|
|:----|:----|:----|
|id|STRING|ユーザーID|
|pw|STRING|ユーザーパスワード（ランダム生成）|
|is_alive|BOOLEAN|アカウント利用可能状況（trueでログイン可能）|
|create_dt|TIMESTAMP|アカウント作成日時|
|update_dt|TIMESTAMP|アカウント情報変更日時（デフォルトはcreate_dtと同一）|
|is_admin|BOOLEAN|管理者フラグ|
|code|STRING|コード（同一の法人などをまとめて管理する場合に使用）|



- `log_login_table`: ログイン履歴
    - テーブルの構成は下記の通り

|フィールド名|データ型|内容|
|:----|:----|:----|
|timestamp|TIMESTAMP|ログイン時のタイムスタンプ|
|id|STRING|ログインしたユーザーID|
|password|STRING|ログインしたユーザーパスワード|
|result|STRING|ログイン成否（"success" or "failed" ）|
|sessionId|STRING|セッションID（擬似的にidとタイムスタンプから生成）|



- `log_search_table`: 検索履歴
    - テーブルの構成は下記の通り
    - ※keywordは過去使用していたが現在使用していないので、新規にテーブルを作る際は作成不要

|フィールド名|データ型|内容・説明|
|:----|:----|:----|
|timestamp|TIMESTAMP|検索実行時のタイムスタンプ|
|sessionId|STRING|セッションID（ログイン時に擬似的にidとタイムスタンプから生成）|
|keyword|STRING|検索キーワード|
|filter_ministries|STRING|検索条件として指定した省庁|
|filter_category|STRING|検索条件として指定した資料カテゴリ|
|filter_subcategory|STRING|検索条件として指定した資料形式カテゴリ|
|filter_year|STRING|検索条件として指定した年度|
|filter_councils|STRING|検索条件として指定した会議体|
|keyword_and|STRING|検索キーワード（and）|
|keyword_or|STRING|検索キーワード（or）|



### 6. 選択肢JSONファイルの準備

`choices/` ディレクトリに以下のファイルが含まれています：
- `ministry_tree.json`: 省庁・外局の階層構造
- `category.json`: カテゴリ選択肢
- `sub_category.json`: 資料形式選択肢
- `year.json`: 年度選択肢

### 7. マニュアルの準備

`docs/manual.md` に使用方法とデータ情報が記載されています。

### 8. Streamlit上でのアプリ作成
Streamlit管理画面の「Create app」からアプリを作成し、Githubと連携して作成してください。

### 9. Streamlit Secretsの設定
StreamlitのApp settingsの「Secrets」にて、以下の形式でSecretsを設定してください：

```toml
[gcp_service_account]
type = "service_account"
project_id = "your-project-id"
private_key_id = "xxxxx"
private_key = "-----BEGIN PRIVATE KEY-----\nxxxxx\n-----END PRIVATE KEY-----\n"
client_email = "xxxxx@xxxxx.iam.gserviceaccount.com"
client_id = "xxxxx"
auth_uri = "https://accounts.google.com/o/oauth2/auth"
token_uri = "https://oauth2.googleapis.com/token"
auth_provider_x509_cert_url = "https://www.googleapis.com/oauth2/v1/certs"
client_x509_cert_url = "https://www.googleapis.com/robot/v1/metadata/x509/xxxxx"

[bigquery]
project_id = "your-project-id"
rawdata_dataset = "rawdata_dataset_name"
config_dataset = "config_dataset_name"
budget_table = "budget_table_name"
council_table = "council_table_name"
council_list = "council_list_table_name"
auth_table = "auth_table_name"
log_login_table = "log_login_table_name"
log_search_table = "log_search_table_name"
```

## 使用方法

### ログイン

1. 認証テーブルに登録されたユーザーIDとパスワードを入力
2. ログインに成功すると検索画面に遷移

### 検索

1. **キーワード入力**:
   - AND検索: すべてのキーワードを含む資料を検索
   - OR検索: いずれかのキーワードを含む資料を検索
   - 両方を組み合わせることも可能

2. **条件絞り込み**:
   - 省庁、カテゴリ、資料形式、年度、会議体を選択

3. **検索実行**:
   - 「🔍 検索」ボタンをクリック

4. **結果の確認**:
   - 「予算」「会議資料」タブで結果を確認
   - URLリンクから元資料にアクセス可能

### アカウントの改廃
auth_tableを操作してください。
1. **アカウントの作成**:
   - Bigquery上でクエリを使用してInsertしてください。
   - 例：x1234というidを発行する場合（パスワードはランダム生成したものを使用してください）
```
INSERT INTO `project_id-config_dataset_name.auth_table` 
  VALUES
    ('x1234','password',true,TIMESTAMP(CURRENT_DATETIME('Asia/Tokyo')),TIMESTAMP(CURRENT_DATETIME('Asia/Tokyo')),false,'code_a')
```

2. **アカウントの情報変更**:
   - Bigquery上でクエリを使用してUpdateしてください。
   - 例：ユーザーID：x1234のパスワードを変更する場合
```
Update `project_id-config_dataset_name.auth_table` 
  SET pw = 'new_password', update_dt = TIMESTAMP(CURRENT_DATETIME('Asia/Tokyo'))
  WHERE id = x1234
```

3. **アカウントの利用停止**:
   - Bigquery上でクエリを使用して「is_alive」をfalseにUpdateしてください。
   - 例：ユーザーID：x1234のアカウントを利用停止する場合
```
Update `project_id-config_dataset_name.auth_table` 
  SET is_alive = false, update_dt = TIMESTAMP(CURRENT_DATETIME('Asia/Tokyo'))
  WHERE id = x1234
```


## プロジェクト構成

```
.
├── app.py                      # メインアプリケーション
├── requirements.txt            # 依存パッケージ
├── .gitignore                  # Git管理外ファイルの定義
├── choices/                    # 選択肢マスタデータ
│   ├── ministry_tree.json
│   ├── category.json
│   ├── sub_category.json
│   └── year.json
├── docs/
│   └── manual.md              # 使用方法マニュアル
└── readme.md                  # プロジェクトの説明（本ファイル）
```

## データ収録範囲

### 予算資料
- **概算要求**: 2013年度〜2026年度
- **当初予算**: 2013年度〜2025年度
- **補正予算**: 2013年度〜2025年度（公開分）

### 会議資料
- **各種本部・連絡会議等**: 2023年度以降
- **審議会**: 2023年度以降（内閣府、復興庁、総務省、文部科学省、厚生労働省、農林水産省、経済産業省、国土交通省）

## 注意事項

- 会議体を選択した場合、予算タブの検索は実行されません
- 検索結果件数が多い場合、表示に時間がかかることがあります
- `service_account.json` や `.streamlit/secrets.toml` は絶対にGitリポジトリにコミットしないでください

---

**開発環境**: Python 3.8+, Streamlit, Google Cloud BigQuery