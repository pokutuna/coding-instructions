# Mesopを使ったアプリケーション開発ガイド

## 1. 概要

Mesopは、Pythonを使用してウェブアプリケーションを簡単に構築できるUIフレームワークです。特にAI/MLのデモアプリケーションや内部ツールの迅速な開発に適しています。フロントエンド開発の知識が少ない開発者でも、Pythonだけを使用して美しく機能的なウェブインターフェースを構築できます。

Mesopの主な特徴：
- **Pythonのみでの開発**: JavaScript、HTML、CSSを直接書く必要がありません
- **コンポーネントベース**: 再利用可能なコンポーネントを組み合わせてUI構築
- **ステート管理**: セッションごとのステート管理が簡単に実装可能
- **AI向けコンポーネント**: チャットインターフェースなどAIアプリケーション向けの高レベルコンポーネントを標準提供
- **ホットリロード**: 開発時の変更が即座に反映される

## 3. 主要概念

Mesopでアプリケーションを構築する際の主要な概念を理解しましょう。

### 3.1 コンポーネント

Mesopアプリケーションは、コンポーネントのツリーで構成されます。コンポーネントには以下の種類があります：

1. **ネイティブコンポーネント**: Mesopに組み込まれたUIコンポーネント（button, text, inputなど）
2. **ユーザー定義コンポーネント**: 開発者が作成するPython関数で、他のコンポーネントを呼び出す
3. **コンテンツコンポーネント**: 子コンポーネントを受け入れる特殊なコンポーネント

### 3.2 ステート管理

Mesopでは、`@me.stateclass`デコレータを使用してアプリケーションの状態（ステート）を管理します。ステートクラスは、セッションごとに個別のインスタンスが作成されます。

### 3.3 イベントハンドラ

ユーザーの操作（ボタンクリックなど）に応答するためのイベントハンドラ関数を定義します。イベントハンドラ内でのみステートを変更できます。

## 4. 基本的な使用方法とコード例

### 4.1 Hello Worldアプリケーション

最も基本的なMesopアプリケーションの例です：

```python
import mesop as me

@me.page(path="/hello_world")
def app():
    me.text("Hello World")
```

このコードを`hello_world.py`として保存し、以下のコマンドで実行します：

```bash
mesop hello_world.py
```

ブラウザで`http://localhost:32123`を開くと、「Hello World」と表示されます。

### 4.2 カウンターアプリケーション

ステートとイベントハンドラを使用した簡単なカウンターアプリケーション：

```python
import mesop as me

@me.stateclass
class State:
    clicks: int = 0  # 初期値を0に設定

def button_click(event: me.ClickEvent):
    state = me.state(State)  # 現在のセッションのステートを取得
    state.clicks += 1  # クリック数を増加

@me.page(path="/counter")
def main():
    state = me.state(State)  # 現在のセッションのステートを取得
    me.text(f"Clicks: {state.clicks}")  # クリック数を表示
    me.button("Increment", on_click=button_click)  # ボタンを表示
```

このアプリケーションでは：

1. `State`クラスでクリック回数を追跡
2. `button_click`イベントハンドラでクリック時にカウントを増加
3. `main`関数で現在のカウントを表示し、クリック可能なボタンを提供

### 4.3 テキスト入力と出力

ユーザー入力を受け取り、処理する例：

```python
import mesop as me

@me.stateclass
class State:
    input: str = ""
    output: str = ""

def on_blur(e: me.InputBlurEvent):
    state = me.state(State)
    state.input = e.value  # 入力値を保存

def on_submit(e: me.InputSubmitEvent):
    state = me.state(State)
    state.input = e.value  # 入力値を保存
    state.output = e.value.upper()  # 大文字に変換して出力

@me.page(path="/text_io")
def app():
    state = me.state(State)
    
    # 入力フィールド
    me.input(
        label="テキストを入力", 
        value=state.input,
        on_blur=on_blur,
        on_submit=on_submit
    )
    
    # 「変換」ボタン
    if state.input:
        with me.box(style=me.Style(margin=me.Margin(top=10, bottom=10))):
            me.button("大文字に変換", on_click=lambda e: on_submit(me.InputSubmitEvent(value=state.input)))
    
    # 出力表示
    if state.output:
        with me.box(style=me.Style(margin=me.Margin(top=10))):
            me.text("変換結果:")
            me.text(state.output, style=me.Style(font_weight="bold"))
```

### 4.4 レイアウトとスタイリング

`box`コンポーネントとスタイルを使用したレイアウト例：

```python
import mesop as me

@me.page(path="/layout")
def app():
    # ヘッダー
    with me.box(style=me.Style(
        background_color="#f5f5f5",
        padding=me.Padding.all(20),
        margin=me.Margin(bottom=20)
    )):
        me.text("アプリケーションヘッダー", type="headline-4")
    
    # メインコンテンツ
    with me.box(style=me.Style(
        display="flex",
        flex_direction="row",
        gap=20
    )):
        # サイドバー
        with me.box(style=me.Style(
            width="30%",
            background_color="#e0e0e0",
            padding=me.Padding.all(15),
            border_radius=5
        )):
            me.text("サイドバー", type="headline-5")
            me.text("ナビゲーションリンク1")
            me.text("ナビゲーションリンク2")
            me.text("ナビゲーションリンク3")
        
        # メインエリア
        with me.box(style=me.Style(
            flex_grow=1,
            padding=me.Padding.all(15),
            border=me.Border.all(width=1, style="solid", color="#ccc"),
            border_radius=5
        )):
            me.text("メインコンテンツエリア", type="headline-5")
            me.text("ここにコンテンツが表示されます。")
            
            with me.box(style=me.Style(margin=me.Margin(top=15))):
                me.button("アクション")
```

## 5. 高度なトピック

### 5.1 カスタムコンポーネントの作成

再利用可能なコンポーネントを作成する例：

```python
import mesop as me

# 通常のコンポーネント
@me.component
def user_card(name: str, role: str, avatar_url: str = None):
    with me.box(style=me.Style(
        border=me.Border.all(width=1, style="solid", color="#ddd"),
        border_radius=8,
        padding=me.Padding.all(16),
        display="flex",
        flex_direction="row",
        align_items="center",
        gap=16
    )):
        # アバター画像（オプション）
        if avatar_url:
            me.image(src=avatar_url, style=me.Style(
                width=50,
                height=50,
                border_radius=25  # 円形に
            ))
        
        # ユーザー情報
        with me.box():
            me.text(name, style=me.Style(font_weight="bold"))
            me.text(role, style=me.Style(color="#666"))

# コンテンツコンポーネント
@me.content_component
def card(title: str):
    with me.box(style=me.Style(
        border=me.Border.all(width=1, style="solid", color="#ddd"),
        border_radius=8,
        padding=me.Padding.all(16),
        margin=me.Margin(bottom=16)
    )):
        me.text(title, type="headline-6", style=me.Style(margin=me.Margin(bottom=8)))
        me.slot()  # 子コンポーネントはここに挿入される

@me.page(path="/custom_components")
def app():
    # 通常のコンポーネントの使用
    user_card("田中太郎", "エンジニア", "https://example.com/avatar1.jpg")
    user_card("鈴木花子", "デザイナー")
    
    # コンテンツコンポーネントの使用
    with card("重要なお知らせ"):
        me.text("これは重要なお知らせの内容です。")
        me.text("詳細は以下のリンクをご覧ください。")
    
    with card("統計情報"):
        me.text("今月のアクセス数: 1,234")
        me.text("先月比: +5.6%")
```

### 5.2 ナビゲーション

複数ページを持つアプリケーションの例：

```python
import mesop as me

# 共通のナビゲーションメニュー
@me.component
def nav_menu(current_page: str):
    pages = [
        {"path": "/", "name": "ホーム"},
        {"path": "/about", "name": "概要"},
        {"path": "/contact", "name": "お問い合わせ"}
    ]
    
    with me.box(style=me.Style(
        display="flex",
        gap=16,
        margin=me.Margin(bottom=20)
    )):
        for page in pages:
            # 現在のページのスタイルを変える
            style = me.Style(
                font_weight="bold" if page["path"] == current_page else "normal",
                color="#007bff" if page["path"] == current_page else "#333",
                text_decoration="underline" if page["path"] == current_page else "none",
                cursor="pointer"
            )
            
            # クリック時に別ページに移動
            def navigate_to(e, path=page["path"]):
                me.navigate(path)
            
            me.text(page["name"], style=style, on_click=navigate_to)

# ホームページ
@me.page(path="/")
def home():
    nav_menu("/")
    me.text("ホームページへようこそ", type="headline-4")
    me.text("これはMesopで構築されたマルチページアプリケーションのデモです。")

# 概要ページ
@me.page(path="/about")
def about():
    nav_menu("/about")
    me.text("概要", type="headline-4")
    me.text("このアプリケーションは、Mesopフレームワークを使用して構築されています。")
    me.text("Mesopは、Pythonだけを使用してウェブアプリケーションを構築できるUIフレームワークです。")

# お問い合わせページ
@me.page(path="/contact")
def contact():
    nav_menu("/contact")
    me.text("お問い合わせ", type="headline-4")
    me.text("以下のフォームからお問い合わせください。")
    
    @me.stateclass
    class ContactState:
        name: str = ""
        email: str = ""
        message: str = ""
        submitted: bool = False
    
    def update_name(e: me.InputBlurEvent):
        state = me.state(ContactState)
        state.name = e.value
    
    def update_email(e: me.InputBlurEvent):
        state = me.state(ContactState)
        state.email = e.value
    
    def update_message(e: me.InputBlurEvent):
        state = me.state(ContactState)
        state.message = e.value
    
    def submit_form(e: me.ClickEvent):
        state = me.state(ContactState)
        # ここで実際にはフォームデータを処理
        state.submitted = True
    
    state = me.state(ContactState)
    
    if not state.submitted:
        with me.box(style=me.Style(
            display="flex",
            flex_direction="column",
            gap=16,
            max_width=500
        )):
            me.input(label="お名前", on_blur=update_name)
            me.input(label="メールアドレス", on_blur=update_email)
            me.textarea(label="メッセージ", on_blur=update_message)
            me.button("送信", on_click=submit_form)
    else:
        me.text("お問い合わせありがとうございます。まもなくご連絡いたします。", 
                style=me.Style(color="green"))
```

### 5.3 ダイアログ（モーダル）の使用

ダイアログを表示する例：

```python
import mesop as me

@me.stateclass
class State:
    is_dialog_open: bool = False
    selected_option: str = ""

def open_dialog(e: me.ClickEvent):
    state = me.state(State)
    state.is_dialog_open = True

def close_dialog(e: me.ClickEvent):
    state = me.state(State)
    state.is_dialog_open = False

def select_option(e: me.ClickEvent):
    state = me.state(State)
    state.selected_option = e.key  # ボタンのkeyを選択値として使用
    state.is_dialog_open = False

@me.content_component
def dialog(is_open: bool):
    with me.box(style=me.Style(
        background="rgba(0,0,0,0.4)",
        display="block" if is_open else "none",
        height="100%",
        overflow="auto",
        position="fixed",
        width="100%",
        z_index=1000,
        top=0,
        left=0
    )):
        with me.box(style=me.Style(
            align_items="center",
            display="grid",
            height="100vh",
            justify_items="center"
        )):
            with me.box(style=me.Style(
                background="#fff",
                border_radius=20,
                padding=me.Padding.all(20),
                max_width=500,
                width="80%"
            )):
                me.slot()  # ダイアログの内容

@me.page(path="/dialog")
def app():
    state = me.state(State)
    
    # メインコンテンツ
    me.button("オプションを選択", on_click=open_dialog)
    
    if state.selected_option:
        me.text(f"選択されたオプション: {state.selected_option}")
    
    # ダイアログ
    with dialog(state.is_dialog_open):
        me.text("オプションを選択してください", type="headline-5")
        
        with me.box(style=me.Style(margin=me.Margin(top=15, bottom=15))):
            for option in ["オプション1", "オプション2", "オプション3"]:
                with me.box(style=me.Style(margin=me.Margin(bottom=10))):
                    me.button(option, key=option, on_click=select_option)
        
        with me.box(style=me.Style(
            display="flex",
            justify_content="flex-end",
            margin=me.Margin(top=15)
        )):
            me.button("キャンセル", on_click=close_dialog)
```

## 6. AIアプリケーションの例

MesopはもともとGoogle内部でAIデモや内部ツールの構築に使用されており、AIアプリケーション向けの機能が豊富です。以下はチャットインターフェースの例です：

```python
import mesop as me
from dataclasses import dataclass, field
from typing import Literal

# チャットメッセージの型定義
@dataclass
class ChatMessage:
    role: Literal["user", "assistant"] = "user"
    content: str = ""

@me.stateclass
class State:
    messages: list[ChatMessage] = field(default_factory=list)
    input_text: str = ""
    is_loading: bool = False

def on_input_change(e: me.InputBlurEvent):
    state = me.state(State)
    state.input_text = e.value

def send_message(e: me.ClickEvent = None):
    state = me.state(State)
    
    # 空メッセージは送信しない
    if not state.input_text.strip():
        return
    
    # ユーザーメッセージを追加
    user_message = ChatMessage(role="user", content=state.input_text)
    state.messages.append(user_message)
    state.input_text = ""  # 入力をクリア
    
    # AIの応答を生成（ここでは簡単なエコー）
    state.is_loading = True
    
    # 実際のアプリではここでAI APIを呼び出す
    # ここでは単純なエコーで代用
    ai_response = f"あなたのメッセージ「{user_message.content}」を受け取りました。"
    
    # 疑似的な遅延（実際のAPIレスポンスを模倣）
    import time
    time.sleep(1)
    
    # アシスタントの応答を追加
    state.messages.append(ChatMessage(role="assistant", content=ai_response))
    state.is_loading = False

def on_key_press(e: me.InputSubmitEvent):
    state = me.state(State)
    state.input_text = e.value
    send_message()

@me.component
def message_bubble(message: ChatMessage):
    is_user = message.role == "user"
    
    # メッセージのスタイル
    with me.box(style=me.Style(
        display="flex",
        justify_content="flex-end" if is_user else "flex-start",
        margin=me.Margin(bottom=10)
    )):
        with me.box(style=me.Style(
            background_color="#007bff" if is_user else "#f1f0f0",
            color="white" if is_user else "black",
            border_radius=10,
            padding=me.Padding.all(10),
            max_width="70%"
        )):
            me.text(message.content)

@me.page(path="/chat")
def app():
    state = me.state(State)
    
    # ヘッダー
    with me.box(style=me.Style(
        background_color="#f5f5f5",
        padding=me.Padding.all(15),
        border_bottom=me.BorderSide(width=1, style="solid", color="#ddd")
    )):
        me.text("Mesopチャットデモ", type="headline-5")
    
    # メッセージエリア
    with me.box(style=me.Style(
        height="calc(100vh - 150px)",
        overflow_y="auto",
        padding=me.Padding.all(15)
    )):
        # 初期メッセージ
        if not state.messages:
            with me.box(style=me.Style(
                display="flex",
                justify_content="center",
                margin=me.Margin(top=20, bottom=20)
            )):
                me.text("メッセージを送信してください")
        
        # メッセージを表示
        for message in state.messages:
            message_bubble(message)
    
    # 入力エリア
    with me.box(style=me.Style(
        position="fixed",
        bottom=0,
        width="100%",
        padding=me.Padding.all(15),
        background_color="white",
        border_top=me.BorderSide(width=1, style="solid", color="#ddd"),
        display="flex",
        gap=10
    )):
        # テキスト入力
        me.input(
            value=state.input_text,
            placeholder="メッセージを入力...",
            on_blur=on_input_change,
            on_submit=on_key_press,
            style=me.Style(flex_grow=1)
        )
        
        # 送信ボタン
        me.button(
            "送信", 
            on_click=send_message,
            disabled=state.is_loading or not state.input_text.strip()
        )
        
        # ローディングインジケータ
        if state.is_loading:
            me.text("応答生成中...")
```

## 7. デプロイメント

Mesopアプリケーションをデプロイする方法の概要です。

### 7.1 Google Cloud Runへのデプロイ（推奨）

Mesopの公式ドキュメントでは、Google Cloud Runを使用したデプロイが推奨されています。

1. Google Cloud SDKをインストールして設定する
2. Dockerfileを作成する：

```dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

ENV PORT 8080
CMD exec mesop main.py --host 0.0.0.0 --port $PORT
```

3. `requirements.txt`を作成する：

```
mesop
# 他に必要なパッケージがあれば追加
```

4. デプロイコマンドを実行：

```bash
gcloud run deploy YOUR_SERVICE_NAME --source .
```

### 7.2 他のクラウドサービスへのデプロイ

Docker対応の任意のクラウドサービス（AWS、Azure、Herokuなど）にデプロイすることも可能です。上記のDockerfileを使用し、各サービスのデプロイ手順に従ってください。

## 8. トラブルシューティング

### よくある問題と解決策

1. **問題**: `AttributeError: module 'mesop' has no attribute...`
   **解決策**: Mesopのバージョンを確認し、最新バージョンにアップグレードしてください。
   ```bash
   pip install --upgrade mesop
   ```

2. **問題**: ステート更新後にUIが更新されない
   **解決策**: イベントハンドラ内でのみステートを更新していることを確認してください。コンポーネント関数内でのステート変更は反映されません。

3. **問題**: コンポーネントが正しく再利用されない
   **解決策**: 複数のインスタンスを使用する場合は、`key`パラメータを使用して各インスタンスに一意のIDを割り当ててください。

4. **問題**: クロージャ変数がイベントハンドラで正しく取得できない
   **解決策**: イベントハンドラでは`e.key`を使用して関連付けられた値を取得するようにし、クロージャ変数への直接アクセスを避けてください。

## 9. 参考リンク

- [Mesop公式ドキュメント](https://mesop-dev.github.io/mesop/)
- [Mesop GitHub リポジトリ](https://github.com/mesop-dev/mesop)
- [Mesopインストールガイド](https://mesop-dev.github.io/mesop/getting-started/installing/)
- [Mesopコア概念ガイド](https://mesop-dev.github.io/mesop/getting-started/core-concepts/)
- [MesopコンポーネントAPI](https://google.github.io/mesop/components/)
- [Mesopチュートリアル(DuoChat Codelab)](https://google.github.io/mesop/codelab/)
