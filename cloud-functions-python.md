# Cloud RunでシンプルなアプリケーションでのAPIサービス提供ガイド

## 1. 概要
このガイドでは、Google Cloud Runを使用して、PythonベースのシンプルなAPIサービスをデプロイする方法について説明します。Cloud Runは、フルマネージドのコンピューティングプラットフォームで、HTTPリクエストに応答するステートレスなコンテナをデプロイして実行できます。このドキュメントでは、Python関数をCloud Runにデプロイし、簡単なAPIとして公開する方法を示します。

## 2. 主要概念

### Cloud Run関数とは
Cloud Run関数は、標準のHTTPリクエストに応答する「HTTPタイプ」と、Google CloudインフラストラクチャからのイベントをハンドルするためのPublic/Subやストレージの変更などを処理する「イベント駆動タイプ」の2種類があります。

### 関数のライフサイクル
Cloud Run関数はステートレスであり、実行環境はコールドスタートと呼ばれるゼロからの初期化が行われることがあります。コールドスタートは完了までに時間がかかることがあるため、不要なコールドスタートを避け、可能な限りコールドスタートプロセスを効率化することがベストプラクティスです。

## 3. 環境のセットアップ

### 必要なAPIを有効化する
Cloud Run関数を使用するには、以下のAPIを有効化する必要があります：
- Cloud Functions
- Cloud Build
- Artifact Registry
- Cloud Run
- Cloud Logging API

以下のコマンドを使用してAPIを有効化します：

```bash
gcloud services enable artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  logging.googleapis.com
```

必要に応じて、イベントトリガーを使用する場合は、Eventarc APIも有効化します：

```bash
gcloud services enable eventarc.googleapis.com
```

### Google Cloud CLIのインストール

開発環境にGoogle Cloud CLIをインストールするか、Cloud Shellを使用します。Cloud Shellは、すでにGoogle Cloud CLIがプリインストールされたコマンドライン環境です。

## 4. Pythonプロジェクトの構築

### プロジェクト構造
シンプルなAPIサービス用のプロジェクト構造は以下のようになります：

```
project/
├── main.py        # メイン関数コード
├── requirements.txt  # 依存関係
└── README.md      # プロジェクトの説明
```

### HTTP関数の実装
main.pyファイルに、HTTPリクエストを処理する関数を作成します：

```python
import functions_framework
from markupsafe import escape

@functions_framework.http
def hello_http(request):
    """HTTP Cloud Run関数
    
    Args:
        request (flask.Request): リクエストオブジェクト
        
    Returns:
        レスポンステキスト
    """
    request_json = request.get_json(silent=True)
    request_args = request.args
    
    if request_json and 'name' in request_json:
        name = request_json["name"]
    elif request_args and 'name' in request_args:
        name = request_args["name"]
    else:
        name = "World"
        
    return f"Hello {escape(name)}!"
```

### 依存関係の管理
Pythonの依存関係はpipで管理され、requirements.txtというメタデータファイルで表現されます。

以下のように必要な依存関係をrequirements.txtに追加します：

```
functions-framework==3.0.0
markupsafe==2.1.1
```

## 5. Cloud Runへの関数のデプロイ

### Google Cloud コンソールを使ったデプロイ
Google Cloud コンソールからデプロイする場合：

1. Cloud Runページに移動します
2. [関数を作成]をクリックします
3. [サービス名]フィールドに、関数を表す名前を入力します
4. [リージョン]リストで、コンテナをデプロイするリージョンを選択します
5. [ランタイム]リストで、Pythonランタイムバージョンを選択します

### Google Cloud CLIを使ったデプロイ
Google Cloud CLIを使用してデプロイする場合は、以下のコマンドを実行します：

```bash
gcloud run deploy FUNCTION \
  --source . \
  --function FUNCTION_ENTRYPOINT \
  --base-image python312 \
  --region REGION \
  --allow-unauthenticated
```

以下のように置き換えます：
- FUNCTION: デプロイする関数の名前
- FUNCTION_ENTRYPOINT: ソースコード内の関数のエントリポイント（例: hello_http）
- REGION: 関数をデプロイするGoogle Cloudリージョン（例: us-central1）

`--allow-unauthenticated`フラグは、認証なしで関数を呼び出せるようにします。APIを公開する場合に使用します。

## 6. ベストプラクティス

### 関数の最適化
関数を最適化するためのいくつかのベストプラクティスは以下の通りです：

1. **べき等関数を作成する**: 何回呼び出されても結果が同じになることが必要です。これにより、前の呼び出しがコードの途中で失敗した場合は、呼び出しを再試行できます。

2. **HTTPレスポンスの確実な送信**: HTTP関数の場合は、常にHTTPレスポンスを送信することを確認してください。通知を行わないと、タイムアウトまで関数の実行が継続し、タイムアウト時間全体が課金されます。

3. **バックグラウンドアクティビティを開始しない**: 関数の終了後に実行されるコードはCPUにアクセスできず、処理を続行できません。

4. **一時ファイルを削除する**: 一時ディレクトリ内のローカルディスクストレージはメモリ内ファイルシステムです。書き込んだファイルを明示的に削除しないと、最終的にメモリ不足エラーが発生する可能性があります。

### パフォーマンスの最適化

パフォーマンスを最適化するためのベストプラクティス：

1. **同時実行数を適切に設定する**: 同時実行数を制限すると、既存のインスタンスの利用方法が制限されるため、コールドスタートの発生数が増えます。同時実行数を増やすと、1つのインスタンスで複数のリクエストを処理できるようになり、負荷の急増に対処しやすくなります。

2. **グローバル変数を使用して将来の呼び出しでオブジェクトを再利用する**: 変数をグローバルスコープで宣言すると、その値は再計算せずに後続の呼び出しで再利用できます。特にネットワーク接続、ライブラリ参照、APIクライアントオブジェクトをグローバルスコープでキャッシュに保存することが重要です。

3. **インスタンスの最小数を設定してコールドスタートを減らす**: インスタンスの最小数を設定すると、アプリケーションのコールドスタートが減少します。レイテンシの影響を受けやすいアプリケーションの場合は、インスタンスの最小数を設定し、読み込み時に初期化を完了することをおすすめします。

以下は、グローバル変数を使用してオブジェクトを再利用する例です：

```python
import functions_framework
from google.cloud import storage

# グローバルスコープでクライアントを初期化
storage_client = storage.Client()

@functions_framework.http
def storage_api(request):
    """Cloud Storageと連携するAPI"""
    # グローバルな storage_client を再利用
    bucket = storage_client.get_bucket('my-bucket')
    # 以下、処理を続行...
    return "Storage API response"
```

## 7. デプロイされた関数のテスト

### 関数のURLの取得

デプロイ後、以下のコマンドで関数のURLを取得できます：

```bash
gcloud run services describe FUNCTION --region=REGION --format="value(status.url)"
```

### HTTPリクエストの送信

cURLを使用して関数をテストします：

```bash
curl -X POST https://YOUR_FUNCTION_URL -H "Content-Type: application/json" -d '{"name": "John"}'
```

### ログの確認

関数のログはCloud Logging UIまたはGoogle Cloud CLIを使用して表示できます。gcloud CLIでログを表示するには、次のコマンドを使用します：

```bash
gcloud functions logs read \
  --gen2 \
  --limit=10 \
  --region=REGION \
  FUNCTION_NAME
```

## 8. 関数の更新と管理

### 関数の更新

関数のコードや構成を更新する場合は、以下のコマンドを使用します：

```bash
gcloud run deploy FUNCTION \
  --source . \
  --function FUNCTION_ENTRYPOINT \
  --base-image python312 \
  --region REGION
```

### スケーリング設定の変更

必要に応じて、関数のスケーリング設定を更新できます：
- 最小インスタンス数を指定して、コールドスタートを減らす
- 最大インスタンス数を設定して、コスト制御を行う

## 9. トラブルシューティング

### 一般的な問題と解決策

1. **デプロイエラー**: 必要なAPIがすべて有効化されていることを確認します
2. **関数のタイムアウト**: タイムアウト設定を調整するか、処理を最適化します
3. **メモリ不足エラー**: 一時ファイルを削除するか、メモリ割り当てを増やします
4. **コールドスタートの遅延**: 最小インスタンス数を設定するか、初期化コードを最適化します

## 10. 関連ドキュメント

- [Cloud Run 関数のデプロイ](https://cloud.google.com/run/docs/deploy-functions?hl=ja)
- [Cloud Run 関数のベストプラクティス](https://cloud.google.com/run/docs/tips/functions-best-practices?hl=ja)
- [Python ランタイム](https://cloud.google.com/run/docs/runtimes/python)
- [HTTPリクエストを処理する関数の作成](https://cloud.google.com/functions/docs/create-deploy-http-python)
