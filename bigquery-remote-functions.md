# BigQuery リモート関数の実装ガイド

## 1. 概要

BigQuery リモート関数は、BigQuery の SQL 機能を外部サービスと連携させる強力な機能です。このドキュメントでは、Google Cloud Functions と Python を使用して BigQuery から呼び出せるリモート関数を作成する方法について説明します。リモート関数を使うことで、SQL や JavaScript では実現が難しい処理や、BigQuery のユーザー定義関数では利用できないライブラリやサービスを活用することができます。

主な利点:

- Python、Node.js、Go、Java、.NET、Ruby、PHP など、複数の言語でリモート関数を実装可能
- 外部 API やサービスとの連携が容易
- セキュリティやコンプライアンス機能の強化（データの暗号化、トークン化など）
- 複雑なビジネスロジックや機械学習モデルの活用

## 2. 主要概念

### リモート関数の仕組み

BigQuery リモート関数は以下のワークフローで動作します：

1. Cloud Functions または Cloud Run に HTTP エンドポイントを作成
2. BigQuery に CLOUD_RESOURCE 接続を作成
3. BigQuery にリモート関数を定義
4. SQL クエリから通常のユーザー定義関数と同様にリモート関数を呼び出し

### データの流れ

1. BigQuery から Cloud Functions へのリクエスト：
   - JSON 形式の HTTP POST リクエスト
   - 複数の行をバッチ処理

2. Cloud Functions から BigQuery へのレスポンス：
   - JSON 形式の HTTP レスポンス
   - 各行に対する結果を返す

### サポートされるデータ型

リモート関数でサポートされるデータ型は以下の通りです：

- Boolean
- Bytes
- Numeric
- String
- Date
- Datetime
- Time
- Timestamp
- JSON

**注意**: ARRAY、STRUCT、INTERVAL、GEOGRAPHYタイプはサポートされていません。

## 3. セットアップと準備

### 前提条件

- Google Cloud プロジェクト
- 課金が有効なアカウント
- 以下の API が有効になっていること：
  - BigQuery API
  - BigQuery Connection API
  - Cloud Functions API

### 必要な権限

リモート関数を作成および実行するには、以下の権限が必要です：

- Cloud Functions Developer または Cloud Run Developer ロール
- BigQuery Connection API に関連する権限
- 作成したリモート関数に対するアクセス権限

## 4. 実装手順

### 1. Cloud Functions でエンドポイントを作成

以下は、整数を2つ受け取って加算するシンプルな Python 関数の例です。

まず、`main.py` ファイルを作成します：

```python
import functions_framework
from flask import jsonify

# JSON でエンコードされる最大の整数値
_MAX_LOSSLESS = 9007199254740992

@functions_framework.http
def batch_add(request):
    try:
        return_value = []
        request_json = request.get_json()
        calls = request_json['calls']
        
        for call in calls:
            # null 値を除外して整数の合計を計算
            return_value.append(sum([int(x) if isinstance(x, str) else x for x in call if x is not None]))
        
        # 大きな整数値は文字列として返す
        replies = [str(x) if x > _MAX_LOSSLESS or x < -_MAX_LOSSLESS else x for x in return_value]
        return_json = jsonify({"replies": replies})
        return return_json
    
    except Exception as e:
        return jsonify({"errorMessage": str(e)}), 400
```

次に、`requirements.txt` ファイルを作成します：

```
functions-framework==3.*
flask==2.*
```

Google Cloud Console または gcloud コマンドラインツールを使用して、この関数をデプロイします：

```bash
gcloud functions deploy batch_add \
  --runtime python39 \
  --trigger-http \
  --allow-unauthenticated=false
```

デプロイ後、関数の URL をメモしておきます（例: `https://us-central1-myproject.cloudfunctions.net/batch_add`）。

### 2. BigQuery に接続を作成

BigQuery と Cloud Functions 間の通信には CLOUD_RESOURCE 接続が必要です。

**Google Cloud Console で接続を作成する場合:**

1. BigQuery ページに移動
2. 「データを追加」をクリック
3. 「データソースタイプ」から「ビジネスアプリケーション」を選択
4. Vertex AI カードをクリック
5. 接続タイプで「Vertex AI リモートモデル、リモート関数、BigLake（Cloud Resource）」を選択
6. 接続 ID を入力
7. 「接続を作成」をクリック
8. 「接続に移動」をクリックし、サービスアカウント ID をコピー

**bq コマンドラインで接続を作成する場合:**

```bash
bq mk --connection --location=REGION --project_id=PROJECT_ID \
  --connection_type=CLOUD_RESOURCE CONNECTION_ID
```

サービスアカウント ID を取得します：

```bash
bq show --connection PROJECT_ID.REGION.CONNECTION_ID
```

### 3. サービスアカウントにアクセス権を付与

1. IAM & 管理ページに移動
2. 「追加」をクリック
3. 「新しいプリンシパル」フィールドに、コピーしたサービスアカウント ID を入力
4. ロールの選択：
   - 1st-gen Cloud Functions の場合は「Cloud Function Invoker」ロール
   - 2nd-gen Cloud Functions の場合は「Cloud Run Invoker」ロール
   - Cloud Run サービスの場合は「Cloud Run Invoker」ロール
5. 「保存」をクリック

### 4. BigQuery にリモート関数を作成

SQL ステートメントを実行して、リモート関数を作成します：

```sql
CREATE FUNCTION
`PROJECT_ID.DATASET_ID.remote_add`(x INT64, y INT64)
RETURNS INT64
REMOTE WITH CONNECTION `PROJECT_ID.LOCATION.CONNECTION_NAME`
OPTIONS (
  endpoint = 'https://us-central1-myproject.cloudfunctions.net/batch_add'
);
```

## 5. リモート関数の使用例

### 基本的な使用方法

リモート関数は通常のユーザー定義関数と同様に使用できます：

```sql
SELECT
  val,
  `PROJECT_ID.DATASET_ID`.remote_add(val, 2) AS result
FROM
  UNNEST([NULL, 2, 3, 5, 8]) AS val;
```

結果：

```
+------+--------+
| val  | result |
+------+--------+
| NULL | 2      |
| 2    | 4      |
| 3    | 5      |
| 5    | 7      |
| 8    | 10     |
+------+--------+
```

### ユーザー定義コンテキストの活用例

1つのエンドポイントで複数の機能を提供する例（暗号化と復号化）：

```sql
CREATE FUNCTION `PROJECT_ID.DATASET_ID`.encrypt(x BYTES)
RETURNS BYTES
REMOTE WITH CONNECTION `PROJECT_ID.LOCATION.CONNECTION_NAME`
OPTIONS (
  endpoint = 'ENDPOINT_URL',
  user_defined_context = [("mode", "encryption")]
);

CREATE FUNCTION `PROJECT_ID.DATASET_ID`.decrypt(x BYTES)
RETURNS BYTES
REMOTE WITH CONNECTION `PROJECT_ID.LOCATION.CONNECTION_NAME`
OPTIONS (
  endpoint = 'ENDPOINT_URL',
  user_defined_context = [("mode", "decryption")]
);
```

対応する Cloud Functions のコード：

```python
import functions_framework
import json
import base64
from cryptography.fernet import Fernet

# 実際のアプリケーションでは、安全な方法でキーを管理する必要があります
KEY = Fernet.generate_key()
cipher_suite = Fernet(KEY)

@functions_framework.http
def crypto_functions(request):
    try:
        request_json = request.get_json()
        calls = request_json['calls']
        context = request_json.get('userDefinedContext', {})
        mode = context.get('mode', '')
        
        replies = []
        for call in calls:
            data = call[0]  # 最初の引数を取得
            if data is None:
                replies.append(None)
                continue
                
            # バイト型をBase64でデコード
            if isinstance(data, str):
                data = base64.b64decode(data)
                
            if mode == 'encryption':
                # データを暗号化
                encrypted = cipher_suite.encrypt(data)
                replies.append(base64.b64encode(encrypted).decode('utf-8'))
            elif mode == 'decryption':
                # データを復号
                decrypted = cipher_suite.decrypt(data)
                replies.append(base64.b64encode(decrypted).decode('utf-8'))
            else:
                replies.append(None)
        
        return json.dumps({"replies": replies})
    except Exception as e:
        return json.dumps({"errorMessage": str(e)}), 400
```

## 6. 高度なトピック

### バッチ処理の最適化

パフォーマンスを向上させるため、`max_batching_rows` オプションを設定して、1つのリクエストで処理する行数を制限できます：

```sql
CREATE FUNCTION `PROJECT_ID.DATASET_ID.my_function`(...)
RETURNS ...
REMOTE WITH CONNECTION `PROJECT_ID.LOCATION.CONNECTION_NAME`
OPTIONS (
  endpoint = 'ENDPOINT_URL',
  max_batching_rows = 1000
);
```

### 外部 API と連携する例

Google Translate API を使用してテキストを翻訳する例：

```python
import functions_framework
import json
from google.cloud import translate_v2 as translate

client = translate.Client()

@functions_framework.http
def translate_text(request):
    try:
        request_json = request.get_json()
        calls = request_json['calls']
        context = request_json.get('userDefinedContext', {})
        target_language = context.get('target_language', 'en')
        
        replies = []
        for call in calls:
            text = call[0]  # 最初の引数を取得
            if text is None or text == '':
                replies.append('')
                continue
                
            # テキストを翻訳
            result = client.translate(text, target_language=target_language)
            replies.append(result['translatedText'])
        
        return json.dumps({"replies": replies})
    except Exception as e:
        return json.dumps({"errorMessage": str(e)}), 400
```

BigQuery での定義：

```sql
CREATE FUNCTION `PROJECT_ID.DATASET_ID`.translate_to_english(text STRING)
RETURNS STRING
REMOTE WITH CONNECTION `PROJECT_ID.LOCATION.CONNECTION_NAME`
OPTIONS (
  endpoint = 'ENDPOINT_URL',
  user_defined_context = [("target_language", "en")]
);
```

### オブジェクトテーブルの分析

BigQuery のオブジェクトテーブル内の非構造化データを分析する例：

```python
import functions_framework
import json
import urllib.request

@functions_framework.http
def object_length(request):
    calls = request.get_json()['calls']
    replies = []
    for call in calls:
        object_content = urllib.request.urlopen(call[0]).read()
        replies.append(len(object_content))
    return json.dumps({'replies': replies})
```

BigQuery での定義：

```sql
CREATE FUNCTION mydataset.object_length(signed_url STRING)
RETURNS INT64
REMOTE WITH CONNECTION `us.myconnection`
OPTIONS(
  endpoint = "https://us-central1-myproject.cloudfunctions.net/object_length",
  max_batching_rows = 1
);
```

使用例：

```sql
SELECT uri, mydataset.object_length(signed_url) AS content_length
FROM EXTERNAL_OBJECT_TRANSFORM(TABLE my_dataset.object_table, ["SIGNED_URL"])
LIMIT 10;
```

## 7. ベストプラクティス

1. **入力データのフィルタリング**：リモート関数に渡す前にデータをフィルタリングすると、クエリのパフォーマンスとコストが向上します。

2. **スケーラビリティの確保**：Cloud Functions の設定を最適化します：
   - 最小インスタンス数
   - 最大インスタンス数
   - 同時実行数

3. **エラーハンドリングの実装**：
   - 適切な HTTP レスポンスコードを返す
   - すべての例外をキャッチ
   - 408、429、500、503、504 以外のエラーコードを使用して BigQuery の再試行を最小化

4. **バッチ処理の最適化**：
   - `max_batching_rows` オプションを設定
   - 適切なバッチサイズを選択

5. **セキュリティ対策**：
   - 未認証呼び出しを許可しない
   - 必要に応じて VPC Service Controls を使用

## 8. トラブルシューティング

### 一般的な問題と解決策

1. **接続エラー**：
   - サービスアカウントに適切な権限が付与されているか確認
   - 接続 ID とリージョンが正しいか確認
   - Google Cloud SDK が最新バージョンであることを確認

2. **タイムアウトエラー**：
   - `max_batching_rows` の値を小さくする
   - Cloud Functions のタイムアウト設定を増やす
   - 処理を最適化する

3. **権限エラー**：
   - IAM 権限を確認
   - サービスアカウントが適切なロールを持っているか確認

4. **データ型の不一致**：
   - サポートされているデータ型を使用しているか確認
   - 大きな整数値を文字列として処理

## 9. 制限事項

- リモート関数は、ARRAY、STRUCT、INTERVAL、GEOGRAPHYタイプをサポートしていません。
- テーブル値を返すリモート関数は作成できません。
- マテリアライズドビューでリモート関数は使用できません。
- リモート関数の戻り値は常に非決定的とみなされるため、クエリの結果はキャッシュされません。
- 一時的なネットワークエラーやBigQuery内部エラーにより、成功したレスポンスの後でも同じデータで繰り返しリクエストが発生することがあります。

## 10. 料金

- 標準の [BigQuery 料金](https://cloud.google.com/bigquery/pricing) が適用されます。
- また、Cloud Functions や Cloud Run の使用に応じて追加料金が発生します。詳細は [Cloud Functions の料金](https://cloud.google.com/functions/pricing) および [Cloud Run の料金](https://cloud.google.com/run/pricing) を参照してください。

## 11. 関連ドキュメント

- [BigQuery リモート関数の公式ドキュメント](https://cloud.google.com/bigquery/docs/remote-functions)
- [Cloud Functions の概要](https://cloud.google.com/functions/docs/concepts/overview)
- [Cloud Run の概要](https://cloud.google.com/run/docs/overview/what-is-cloud-run)
- [BigQuery 接続の作成方法](https://cloud.google.com/bigquery/docs/create-cloud-resource-connection)
- [リモート関数と翻訳 API のチュートリアル](https://cloud.google.com/bigquery/docs/remote-functions-translation-tutorial)
- [オブジェクトテーブルをリモート関数で分析する方法](https://cloud.google.com/bigquery/docs/object-table-remote-function)
- [Cloud Functions の Python クイックスタート](https://cloud.google.com/functions/docs/create-deploy-http-python)
- [BigQuery の Python ユーザー定義関数](https://cloud.google.com/bigquery/docs/user-defined-functions-python)
- [BigQuery DataFrames を使用したリモート関数のデプロイ](https://cloud.google.com/bigquery/docs/samples/bigquery-dataframes-remote-function)
- [BigQuery のクォータと制限](https://cloud.google.com/bigquery/quotas#remote_function_limits)
