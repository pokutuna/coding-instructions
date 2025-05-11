# Pythonを使用したGoogle Cloud Run Functionsの実装ガイド

このドキュメントは、コーディングエージェント（Claude Codeなど）がPythonでCloud Run Functions（HTTP関数）を実装する際に参考となる情報をまとめたものです。

## 目次

1. [概要](#概要)
2. [前提条件](#前提条件)
3. [プロジェクト構造](#プロジェクト構造)
4. [HTTP関数の作成](#http関数の作成)
5. [依存関係の管理](#依存関係の管理)
6. [ローカルでのテスト](#ローカルでのテスト)
7. [デプロイ方法](#デプロイ方法)
8. [関数のテスト](#関数のテスト)
9. [ログの表示](#ログの表示)
10. [発展的なトピック](#発展的なトピック)
11. [トラブルシューティング](#トラブルシューティング)

## 概要

Google Cloud Run Functionsは、サーバーレスコンピューティングサービスで、コードを実行するためのインフラストラクチャを管理せずに関数を実行できます。関数には主に2つのタイプがあります：

- **HTTP関数**: 標準的なHTTPリクエストから呼び出します
- **イベントドリブン関数**: Pub/SubトピックのメッセージやCloud Storageバケットの変更などのイベントを処理します

このガイドではPythonを使用したHTTP関数の実装に焦点を当てます。

## 前提条件

- Googleアカウント
- Google Cloudプロジェクト（新規または既存）
- プロジェクトの課金が有効化されていること
- 以下のAPIが有効化されていること:
  - Cloud Functions API
  - Cloud Build API
  - Artifact Registry API
  - Cloud Run API
  - Cloud Logging API
- gcloud CLIのインストールと初期化

## プロジェクト構造

Cloud Run Functionsのための基本的なディレクトリ構造は次のとおりです：

```
プロジェクト/
├── main.py          # 関数コード（必須）
└── requirements.txt # 依存関係（推奨）
```

**注意点:**
- Cloud Run Functionsはデフォルトで`main.py`を探します
- 関数のメイン実行ファイルは必ず`main.py`という名前にする必要があります

## HTTP関数の作成

以下は、基本的なHTTP関数の例です：

```python
import functions_framework
from markupsafe import escape

@functions_framework.http
def hello_http(request):
    """HTTP Cloud Function.
    Args:
        request (flask.Request): リクエストオブジェクト。
        https://flask.palletsprojects.com/en/1.1.x/api/#incoming-request-data
    Returns:
        応答テキスト、またはflask.make_responseを使用してレスポンスオブジェクトに
        変換できる値の集合。
        https://flask.palletsprojects.com/en/1.1.x/api/#flask.make_response
    """
    request_json = request.get_json(silent=True)
    request_args = request.args

    if request_json and "name" in request_json:
        name = request_json["name"]
    elif request_args and "name" in request_args:
        name = request_args["name"]
    else:
        name = "World"
    
    return f"Hello {escape(name)}!"
```

この関数は以下を行います：
- HTTPリクエストからパラメータ「name」を取得します
- JSONボディまたはクエリパラメータから「name」を探します
- 「name」が提供されない場合は「World」をデフォルト値として使用します
- フォーマットされた挨拶を返します

### 重要なポイント

- `@functions_framework.http`デコレータを使用して、HTTP関数であることを示します
- リクエストはFlaskの`request`オブジェクトとして渡されます
- セキュリティのために、ユーザー入力（`name`）に対しては`escape()`関数を使用してHTMLエスケープを行います

## 依存関係の管理

Pythonの依存関係は`pip`で管理され、`requirements.txt`ファイルで指定します。

```
# requirements.txtの例
functions-framework==3.*
```

最低限、`functions-framework`パッケージが必要です。その他のライブラリも必要に応じて追加できます。

## ローカルでのテスト

関数をデプロイする前にローカルでテストする手順：

1. 依存関係をインストールします：
   ```bash
   pip install -r requirements.txt
   PATH=$PATH:~/.local/bin
   ```

2. Functions Frameworkを使用して関数をローカルで実行します：
   ```bash
   functions-framework-python --target hello_http
   ```

3. 関数をテストします：
   ```bash
   curl localhost:8080
   ```
   または、ブラウザで`http://localhost:8080`にアクセスします。

## デプロイ方法

関数をデプロイするには、以下のコマンドを実行します：

```bash
gcloud functions deploy python-http-function \
  --gen2 \
  --runtime=python312 \
  --region=REGION \
  --source=. \
  --entry-point=hello_http \
  --trigger-http \
  --allow-unauthenticated
```

パラメータ説明：
- `python-http-function`: デプロイされる関数の名前
- `--gen2`: Cloud Run Functions第2世代を使用
- `--runtime=python312`: Python 3.12ランタイムを使用
- `--region=REGION`: 関数をデプロイするGoogleクラウドリージョン（例：`us-west1`）
- `--source=.`: ソースコードの場所（カレントディレクトリ）
- `--entry-point=hello_http`: 実行するエントリーポイント関数の名前
- `--trigger-http`: HTTP関数として設定
- `--allow-unauthenticated`: 認証なしでアクセス可能に設定（オプション）

**注意:** 本番環境では、`--allow-unauthenticated`フラグの使用を慎重に検討してください。認証が必要な場合は、このフラグを省略します。

## 関数のテスト

デプロイが完了したら、以下のコマンドで関数のURLを取得できます：

```bash
gcloud functions describe python-http-function \
  --region=REGION
```

出力されたURLにブラウザでアクセスするか、curlコマンドを使用してテストできます。

例：
```bash
curl https://REGION-PROJECT_ID.cloudfunctions.net/python-http-function
```

パラメータを渡してテストする場合：
```bash
curl https://REGION-PROJECT_ID.cloudfunctions.net/python-http-function?name=YourName
```

## ログの表示

### コマンドラインでログを表示

```bash
gcloud functions logs read \
  --gen2 \
  --limit=10 \
  --region=REGION \
  python-http-function
```

### ロギングダッシュボードでログを表示

1. [Cloud Run functions の概要ページ](https://console.cloud.google.com/functions/list)を開く
2. 関数の名前をクリック
3. 「ログ」タブをクリック

## 発展的なトピック

### CloudEventsの処理

CloudEventsを処理する関数を作成するには：

```python
import functions_framework
from cloudevents.http.event import CloudEvent

@functions_framework.cloud_event
def hello_cloud_event(cloud_event: CloudEvent) -> None:
    print(f"イベントID: {cloud_event['id']}、データ: {cloud_event.data}")
```

### カスタムエラーハンドリング

特定のエラータイプを処理するカスタムハンドラーを設定できます：

```python
import functions_framework

@functions_framework.errorhandler(ZeroDivisionError)
def handle_zero_division(e):
    return "エラーが発生しました", 500

@functions_framework.http
def function_with_error(request):
    1 / 0  # エラー発生
    return "成功", 200
```

### 異なるContent-Typeの処理

様々なContent-Typeを処理する例：

```python
import functions_framework
from markupsafe import escape

@functions_framework.http
def hello_content(request):
    content_type = request.headers["content-type"]
    
    if content_type == "application/json":
        request_json = request.get_json(silent=True)
        if request_json and "name" in request_json:
            name = request_json["name"]
        else:
            raise ValueError("JSONが無効か、'name'プロパティがありません")
    elif content_type == "application/octet-stream":
        name = request.data
    elif content_type == "text/plain":
        name = request.data
    elif content_type == "application/x-www-form-urlencoded":
        name = request.form.get("name")
    else:
        raise ValueError(f"未知のcontent type: {content_type}")
    
    return f"Hello {escape(name)}!"
```

## トラブルシューティング

### 一般的な問題と解決策

1. **デプロイエラー**
   - 依存関係が正しく記述されているか確認
   - リージョンが存在するか確認
   - プロジェクトに十分な権限があるか確認

2. **関数が呼び出されない**
   - URLが正しいか確認
   - 認証設定を確認
   - ログを確認してエラーメッセージを探す

3. **パフォーマンスの問題**
   - メモリ割り当てを増やす
   - タイムアウト設定を調整する
   - コールドスタートの影響を考慮する

4. **依存関係の問題**
   - `requirements.txt`で正確なバージョンを指定する
   - 互換性のある依存関係を使用する

### デバッグのヒント

- ローカルでの実行時に`--debug`フラグを使用する
- コードに`print`ステートメントを追加してログに出力する
- 複雑な関数は段階的にテストする

## 関連ドキュメント

以下は、Google Cloud Run FunctionsをPythonで実装する際に参考になる公式ドキュメントとリソースのリストです：

1. [Pythonで Cloud Run functionsを作成してデプロイする](https://cloud.google.com/functions/docs/create-deploy-http-python?hl=ja) - Google Cloud公式ドキュメント
2. [HTTP関数の作成](https://cloud.google.com/functions/docs/writing/write-http-functions?hl=ja) - HTTP関数に特化した詳細ガイド
3. [イベントドリブン関数の作成](https://cloud.google.com/functions/docs/writing/write-event-driven-functions?hl=ja) - イベント駆動型関数の実装ガイド
4. [Python Functions Framework](https://github.com/GoogleCloudPlatform/functions-framework-python) - オープンソースのFaaSフレームワーク（GitHub）
5. [Cloud Run Functions ロケーション](https://cloud.google.com/functions/docs/locations?hl=ja) - 利用可能なリージョン一覧
6. [Cloud Run Functions Python ランタイム](https://cloud.google.com/functions/docs/concepts/python-runtime) - Pythonランタイムの詳細情報
7. [Cloud Run Functions のセキュリティ](https://cloud.google.com/functions/docs/securing/managing-access-iam?hl=ja) - IAMとアクセス管理
8. [Cloud Pub/Subとの連携](https://cloud.google.com/functions/docs/tutorials/pubsub) - Pub/Subを使ったイベント処理
9. [Cloud Storageとの連携](https://cloud.google.com/functions/docs/tutorials/storage) - Storageイベントの処理
10. [Cloud Functions クイックスタート](https://cloud.google.com/functions/docs/quickstart) - 迅速に始めるためのガイド
11. [Functions Framework CLI リファレンス](https://github.com/GoogleCloudPlatform/functions-framework-python#command-line-flags) - コマンドラインオプションの詳細
12. [CloudEvents SDK for Python](https://github.com/cloudevents/sdk-python) - CloudEventの詳細な実装例
