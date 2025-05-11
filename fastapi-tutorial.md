# FastAPI による シンプルな API サーバーの実装方法

## 1. 概要

FastAPI は現代的で高性能な Python Web フレームワークで、標準的な Python の型ヒントを活用して API を簡単に構築できるツールです。高速な開発プロセス、自動的なデータ検証、インタラクティブなドキュメント生成など、多くの優れた機能を提供しています。このドキュメントでは、FastAPI を用いてシンプルな API サーバーを実装する方法について説明します。

## 2. 主要概念

### FastAPI の特徴

- **高速**: Starlette と Pydantic のおかげで、NodeJS や Go 言語と同等の高いパフォーマンスを実現
- **開発速度**: 開発速度が 200-300% 向上するとされており、コード作成が非常に速い
- **少ないバグ**: 人的エラーを約 40% 削減
- **直感的**: エディタサポートが優れており、コード補完が可能で、デバッグ時間が短縮
- **簡単**: 学習と使用が容易で、ドキュメントを読む時間が少なくて済む
- **簡潔**: コードの重複を最小限に抑え、パラメータ宣言から複数の機能を提供
- **堅牢**: 本番環境に対応したコードと自動インタラクティブドキュメントを生成
- **標準ベース**: OpenAPI や JSON Schema などの API のオープン標準に基づいている

## 3. セットアップとインストール

### 必要条件

- Python 3.6 以上
- 仮想環境（推奨）

### インストール手順

仮想環境を作成して有効化した後、以下のコマンドでインストールします：

```bash
pip install "fastapi[standard]"
```

FastAPI を実行するには ASGI サーバーが必要です。FastAPI では Uvicorn が推奨されています。`standard` オプションでインストールすると、Uvicorn も含まれます。

## 4. 基本的な使用方法とコード例

### 最小限の FastAPI アプリケーション

以下は最もシンプルな FastAPI アプリケーションの例です：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

このコードを `main.py` という名前のファイルに保存し、以下のコマンドで実行します：

```bash
fastapi dev main.py
```

ブラウザで http://127.0.0.1:8000 にアクセスすると、`{"message": "Hello World"}` という JSON レスポンスが表示されます。

また、自動生成された API ドキュメントは以下の URL で確認できます：
- Swagger UI: http://127.0.0.1:8000/docs
- ReDoc: http://127.0.0.1:8000/redoc

### パスパラメータの使用

URL のパスの一部を変数として使用するパスパラメータを定義できます：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

ここで `item_id` はパスパラメータとして定義され、`:int` という型注釈によって整数型として検証されます。http://127.0.0.1:8000/items/5 にアクセスすると、`{"item_id": 5}` というレスポンスが返されます。

### クエリパラメータの使用

クエリパラメータは URL の `?` の後に続く key-value ペアです：

```python
from fastapi import FastAPI

app = FastAPI()

fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

この例では、`skip` と `limit` がクエリパラメータとして定義されています。http://127.0.0.1:8000/items/?skip=1&limit=1 にアクセスすると、配列の一部のみが返されます。

### Pydantic モデルとリクエストボディ

クライアントからサーバーにデータを送信するには、リクエストボディを使用します。FastAPI では Pydantic モデルを使ってリクエストボディを宣言します：

```python
from fastapi import FastAPI
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str = None
    price: float
    tax: float = None

app = FastAPI()

@app.post("/items/")
async def create_item(item: Item):
    return item
```

この例では、`Item` クラスが Pydantic の `BaseModel` を継承して定義されています。POST リクエストのボディは自動的に検証され、`Item` オブジェクトに変換されます。

### パスパラメータ、クエリパラメータ、リクエストボディの組み合わせ

これらのパラメータタイプを組み合わせることも可能です：

```python
from fastapi import FastAPI
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str = None
    price: float
    tax: float = None

app = FastAPI()

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, q: str = None):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    return result
```

この例では：
- `item_id` はパスパラメータ
- `item` はリクエストボディ (Pydantic モデル)
- `q` はオプションのクエリパラメータ

## 5. 高度なトピック/ベストプラクティス

### 非同期サポート

FastAPI は非同期処理をネイティブにサポートしています。パフォーマンスを向上させるために `async def` を使用できます：

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    result = await some_async_operation()
    return result
```

また、通常の同期関数も使用できます：

```python
@app.get("/items/{item_id}")
def read_item(item_id: int):
    result = some_operation()
    return result
```

### 依存性注入

依存性注入はコードの再利用と分離を可能にする強力な機能です：

```python
from fastapi import Depends, FastAPI

app = FastAPI()

def common_parameters(q: str = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons
```

### バリデーション

FastAPI では Pydantic のバリデーション機能を活用できます：

```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel, Field

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str = Field(None, max_length=300)
    price: float = Field(..., gt=0)
    tax: float = None

@app.get("/items/{item_id}")
async def read_items(
    item_id: int = Path(..., title="The ID of the item", ge=1),
    q: str = Query(None, min_length=3, max_length=50)
):
    return {"item_id": item_id, "q": q}
```

## 6. APIリファレンス (抜粋)

### パスオペレーションデコレータ

- `@app.get()`: GET リクエストを処理
- `@app.post()`: POST リクエストを処理
- `@app.put()`: PUT リクエストを処理
- `@app.delete()`: DELETE リクエストを処理
- `@app.options()`: OPTIONS リクエストを処理
- `@app.head()`: HEAD リクエストを処理
- `@app.patch()`: PATCH リクエストを処理
- `@app.trace()`: TRACE リクエストを処理

### レスポンスモデル

レスポンスのデータモデルを定義できます：

```python
from fastapi import FastAPI
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

class ResponseItem(BaseModel):
    name: str
    price_with_tax: float

app = FastAPI()

@app.post("/items/", response_model=ResponseItem)
async def create_item(item: Item):
    return {"name": item.name, "price_with_tax": item.price * 1.1}
```

## 7. トラブルシューティング

### よくある問題と解決策

1. **CORS (Cross-Origin Resource Sharing) エラー**

   フロントエンドからの API アクセスで CORS エラーが発生した場合は、CORSMiddleware を追加します：

   ```python
   from fastapi import FastAPI
   from fastapi.middleware.cors import CORSMiddleware

   app = FastAPI()

   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],  # 本番環境では特定のオリジンのみを許可
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )
   ```

2. **デバッグモードの有効化**

   開発時にはデバッグモードを有効にすると便利です：

   ```bash
   fastapi dev main.py
   ```

3. **本番環境での実行**

   本番環境では以下のようにして実行することが推奨されています：

   ```bash
   fastapi run
   ```

## 8. まとめ

FastAPI は、標準的な Python の型ヒントを活用した現代的な Web フレームワークであり、高速なパフォーマンス、自動ドキュメント生成、データ検証などの機能を提供しています。この技術ドキュメントでは、FastAPI を使用してシンプルな API サーバーを実装する方法について説明しました。

パスパラメータ、クエリパラメータ、リクエストボディなどの基本的な概念から、依存性注入やデータバリデーションなどの高度なトピックまで、FastAPI の主要な機能を網羅しています。

## 関連ドキュメント

- [FastAPI 公式ドキュメント](https://fastapi.tiangolo.com/)
- [Pydantic ドキュメント](https://docs.pydantic.dev/)
- [Starlette ドキュメント](https://www.starlette.io/)
- [Uvicorn ドキュメント](https://www.uvicorn.org/)
