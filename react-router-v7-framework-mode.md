# React Router v7 Framework モード実装ガイド

## 1. 概要

React Router v7は、シングルページアプリケーション（SPA）からサーバーサイドレンダリング（SSR）まで幅広くサポートする、React向けのルーティングライブラリです。特にv7で導入された「Framework モード」は、Remix（React Routerの開発元が作成したフレームワーク）の機能をReact Routerに取り入れ、フルスタックアプリケーションの構築を可能にします。

このドキュメントでは、React Router v7のFrameworkモードを使用してアプリケーションを実装する方法を解説します。

## 2. 主要概念

React Router v7のFrameworkモードには、以下の主要な概念があります：

* **ルートモジュール**: Frameworkモードの基盤となるファイルで、コードの分割、データ読み込み、アクション、エラーバウンダリなどの機能を定義します。
* **ルート設定**: `app/routes.ts`ファイルでURLパターンとルートモジュールのファイルパスを指定します。
* **データローディング**: サーバーサイドまたはクライアントサイドでデータを取得し、ルートコンポーネントに提供します。
* **アクション**: フォーム送信などのデータ変更操作を処理します。
* **レンダリング戦略**: クライアントサイドレンダリング、サーバーサイドレンダリング、静的プリレンダリングをサポートします。

## 3. セットアップとインストール

### プロジェクトの作成

React Router v7のFrameworkモードを使用するには、以下のコマンドで新しいプロジェクトを作成します：

```bash
# 基本的なテンプレートを使用して新しいプロジェクトを作成
npx create-react-router@latest my-react-router-app

# プロジェクトディレクトリに移動し、依存関係をインストールして開発サーバーを起動
cd my-react-router-app
npm i
npm run dev
```

これにより、`http://localhost:5173`でアプリケーションにアクセスできるようになります。

### 既存のプロジェクトの場合

既存のプロジェクトで React Router v7を使用するには、以下のコマンドでインストールします：

```bash
# react-router-domは不要になりました（v7からは単一のパッケージに統合）
npm uninstall react-router-dom
npm install react-router@latest
```

## 4. ルーティングの設定

### ルートの設定

Frameworkモードでは、`app/routes.ts`ファイルでルートを設定します：

```typescript
// app/routes.ts
import { type RouteConfig, route, index, layout, prefix } from "@react-router/dev/routes";

export default [
  // ルートパス（/）へのルート
  index("./home.tsx"),
  
  // 静的パス（/about）へのルート
  route("about", "./about.tsx"),
  
  // レイアウトルート（URLパスに影響しない）
  layout("./marketing/layout.tsx", [
    index("./marketing/home.tsx"),
    route("contact", "./marketing/contact.tsx"),
  ]),
  
  // プレフィックスを使用した複数のルート
  ...prefix("projects", [
    index("./projects/home.tsx"),
    route(":projectId", "./projects/project.tsx"),
    route(":projectId/edit", "./projects/edit-project.tsx"),
  ]),
] satisfies RouteConfig;
```

### ルートのタイプ

React Router v7では、様々なタイプのルートを定義できます：

1. **インデックスルート (`index`)**: 親ルートのURLでレンダリングされるデフォルトのルート
2. **通常ルート (`route`)**: 特定のパスパターンに一致するルート
3. **レイアウトルート (`layout`)**: URLパスに影響せず、子ルート用のレイアウトを提供
4. **プレフィックスルート (`prefix`)**: 複数のルートに共通のパスプレフィックスを追加

### 動的セグメント

URLの一部を動的に変化させることができます：

```typescript
// 動的セグメント
route("teams/:teamId", "./team.tsx")

// 複数の動的セグメント
route("c/:categoryId/p/:productId", "./product.tsx")

// オプショナルセグメント
route(":lang?/categories", "./categories.tsx")

// スプラット（残りのパスをすべてキャプチャ）
route("files/*", "./files.tsx")
```

### ファイルベースのルーティング

設定ファイルの代わりにファイル名の規則でルートを定義することもできます：

```typescript
import { type RouteConfig, route } from "@react-router/dev/routes";
import { flatRoutes } from "@react-router/fs-routes";

export default [
  route("/", "./home.tsx"),
  ...(await flatRoutes()),
] satisfies RouteConfig;
```

## 5. ルートモジュール

ルートモジュールは、ルートの動作を定義するファイルです。以下に主要なエクスポートタイプを示します：

### 基本的なコンポーネント（デフォルトエクスポート）

```typescript
// app/routes/dashboard.tsx
import type { Route } from "./+types/dashboard";

export default function Dashboard({ 
  loaderData,
  actionData,
  params,
  matches,
}: Route.ComponentProps) {
  return (
    <div>
      <h1>ダッシュボード</h1>
      <Outlet /> {/* 子ルートがここにレンダリングされる */}
    </div>
  );
}
```

### データローディング

#### サーバーサイドローダー

```typescript
// app/routes/projects.tsx
import type { Route } from "./+types/projects";
import { fakeDb } from "../db";

export async function loader({ params }: Route.LoaderArgs) {
  const projects = await fakeDb.getProjects();
  return projects;
}

export default function Projects({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <h1>プロジェクト一覧</h1>
      <ul>
        {loaderData.map(project => (
          <li key={project.id}>{project.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

#### クライアントサイドローダー

```typescript
// app/routes/products/[id].tsx
import type { Route } from "./+types/product";

export async function clientLoader({ params }: Route.ClientLoaderArgs) {
  const res = await fetch(`/api/products/${params.id}`);
  const product = await res.json();
  return product;
}

// ローダー実行中はHydrateFallbackが表示される
export function HydrateFallback() {
  return <div>読み込み中...</div>;
}

export default function Product({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <h1>{loaderData.name}</h1>
      <p>{loaderData.description}</p>
    </div>
  );
}
```

### データ変更（アクション）

#### サーバーサイドアクション

```typescript
// app/routes/projects/new.tsx
import type { Route } from "./+types/new-project";
import { Form } from "react-router";
import { fakeDb } from "../../db";

export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData();
  const title = formData.get("title");
  
  const project = await fakeDb.createProject({ title });
  return project;
}

export default function NewProject({ actionData }: Route.ComponentProps) {
  return (
    <div>
      <h1>新規プロジェクト</h1>
      <Form method="post">
        <input type="text" name="title" />
        <button type="submit">作成</button>
      </Form>
      {actionData && <p>{actionData.title}が作成されました</p>}
    </div>
  );
}
```

#### クライアントサイドアクション

```typescript
// app/routes/tasks/[id].tsx
import type { Route } from "./+types/task";
import { Form } from "react-router";
import { clientApi } from "../../api";

export async function clientAction({ request }: Route.ClientActionArgs) {
  const formData = await request.formData();
  const title = formData.get("title");
  
  const task = await clientApi.updateTask({ title });
  return task;
}

export default function Task({ actionData }: Route.ComponentProps) {
  return (
    <div>
      <h1>タスク編集</h1>
      <Form method="post">
        <input type="text" name="title" />
        <button type="submit">更新</button>
      </Form>
      {actionData && <p>{actionData.title}に更新されました</p>}
    </div>
  );
}
```

### エラーハンドリング

```typescript
// app/routes/dashboard.tsx
import { isRouteErrorResponse, useRouteError } from "react-router";

export function ErrorBoundary() {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  } else if (error instanceof Error) {
    return (
      <div>
        <h1>エラーが発生しました</h1>
        <p>{error.message}</p>
      </div>
    );
  } else {
    return <h1>不明なエラーが発生しました</h1>;
  }
}
```

### メタデータと追加リソース

```typescript
// app/routes/about.tsx
export function meta() {
  return [
    { title: "会社概要" },
    { name: "description", content: "当社の概要ページです" },
    { property: "og:title", content: "会社概要" },
  ];
}

export function links() {
  return [
    { rel: "stylesheet", href: "/styles/about.css" },
    { rel: "preload", href: "/images/team.jpg", as: "image" },
  ];
}

export const handle = {
  breadcrumb: "会社概要",
};
```

## 6. レンダリング戦略

React Router v7では、3つのレンダリング戦略をサポートしています：

### クライアントサイドレンダリング（SPA）

SPA（シングルページアプリケーション）として構築する場合、`react-router.config.ts`でサーバーサイドレンダリングを無効にします：

```typescript
// react-router.config.ts
import type { Config } from "@react-router/dev/config";

export default {
  ssr: false,
} satisfies Config;
```

### サーバーサイドレンダリング

SSRを有効にするには：

```typescript
// react-router.config.ts
import type { Config } from "@react-router/dev/config";

export default {
  ssr: true,
} satisfies Config;
```

### 静的プリレンダリング

ビルド時に特定のページを静的生成する場合：

```typescript
// react-router.config.ts
import type { Config } from "@react-router/dev/config";

export default {
  // ビルド時にプリレンダリングするURLのリストを返す
  async prerender() {
    return ["/", "/about", "/contact"];
  },
} satisfies Config;
```

動的なURLを静的に生成するためのより複雑な例：

```typescript
// react-router.config.ts
import type { Config } from "@react-router/dev/config";

export default {
  async prerender() {
    // データソースから動的URLのリストを生成
    const products = await readProductsFromDatabase();
    return [
      "/",
      "/about",
      ...products.map(product => `/products/${product.id}`)
    ];
  },
} satisfies Config;
```

## 7. ナビゲーション

### リンクとナビゲーション

#### 通常のリンク

```tsx
import { Link } from "react-router";

function HomeLink() {
  return <Link to="/">ホームに戻る</Link>;
}
```

#### アクティブスタイル付きのナビゲーションリンク

```tsx
import { NavLink } from "react-router";

function Navigation() {
  return (
    <nav>
      <NavLink to="/" end>
        ホーム
      </NavLink>
      <NavLink to="/projects">
        {({ isActive, isPending }) => (
          <span className={isActive ? "active" : ""}>
            プロジェクト {isPending && "読み込み中..."}
          </span>
        )}
      </NavLink>
    </nav>
  );
}
```

#### プログラムによるナビゲーション

```tsx
import { useNavigate } from "react-router";

function LogoutButton() {
  const navigate = useNavigate();
  
  const handleLogout = () => {
    // ログアウト処理
    logout();
    // ログインページへリダイレクト
    navigate("/login");
  };
  
  return <button onClick={handleLogout}>ログアウト</button>;
}
```

#### リダイレクト

```tsx
import { redirect } from "react-router";

export async function loader({ request }) {
  const user = await getUser(request);
  
  if (!user) {
    return redirect("/login");
  }
  
  return { user };
}
```

### フォームの送信

#### 通常のフォーム送信（ナビゲーションあり）

```tsx
import { Form } from "react-router";

function LoginForm() {
  return (
    <Form method="post" action="/login">
      <input type="text" name="username" />
      <input type="password" name="password" />
      <button type="submit">ログイン</button>
    </Form>
  );
}
```

#### フェッチャーによるフォーム送信（ナビゲーションなし）

```tsx
import { useFetcher } from "react-router";

function CommentForm() {
  const fetcher = useFetcher();
  const busy = fetcher.state !== "idle";
  
  return (
    <fetcher.Form method="post" action="/comments/new">
      <textarea name="text"></textarea>
      <button type="submit" disabled={busy}>
        {busy ? "送信中..." : "コメントを投稿"}
      </button>
    </fetcher.Form>
  );
}
```

## 8. 高度なトピック

### 楽観的UI更新

ユーザーのアクションに対して即座に応答するUIを実装できます：

```tsx
function Task({ task }) {
  const fetcher = useFetcher();
  
  // フォームデータに基づいて楽観的に状態を表示
  let isComplete = task.status === "complete";
  if (fetcher.formData) {
    isComplete = fetcher.formData.get("status") === "complete";
  }
  
  return (
    <div>
      <div>{task.title}</div>
      <fetcher.Form method="post">
        <button
          name="status"
          value={isComplete ? "incomplete" : "complete"}
        >
          {isComplete ? "未完了にする" : "完了にする"}
        </button>
      </fetcher.Form>
    </div>
  );
}
```

### テスト

React Router v7のコンポーネントをテストするには、`createRoutesStub`を使用します：

```tsx
import { createRoutesStub } from "react-router";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { LoginForm } from "./LoginForm";

test("LoginForm エラーメッセージを表示できること", async () => {
  const USER_MESSAGE = "ユーザー名は必須です";
  const PASSWORD_MESSAGE = "パスワードは必須です";
  
  const Stub = createRoutesStub([
    {
      path: "/login",
      Component: LoginForm,
      action() {
        return {
          errors: {
            username: USER_MESSAGE,
            password: PASSWORD_MESSAGE,
          },
        };
      },
    },
  ]);
  
  // /loginパスでスタブをレンダリング
  render(<Stub initialEntries={["/login"]} />);
  
  // インタラクションのシミュレーション
  userEvent.click(screen.getByText("ログイン"));
  
  // エラーメッセージが表示されることを確認
  await waitFor(() => screen.findByText(USER_MESSAGE));
  await waitFor(() => screen.findByText(PASSWORD_MESSAGE));
});
```

### デプロイ

React Router v7のアプリケーションは、以下のいずれかの方法でデプロイできます：

1. **フルスタックホスティング**
   - Dockerを使用したNode.js
   - Vercel
   - Cloudflare Workers
   - Netlify

2. **静的ホスティング**
   - ビルド時に静的に生成されたファイルをデプロイ

公式テンプレートには、様々なデプロイ環境向けの設定が含まれています：

```bash
# Node.js + Docker
npx create-react-router@latest --template remix-run/react-router-templates/default

# Vercel
npx create-react-router@latest --template remix-run/react-router-templates/vercel

# Cloudflare Workers
npx create-react-router@latest --template remix-run/react-router-templates/cloudflare

# Netlify
npx create-react-router@latest --template remix-run/react-router-templates/netlify
```

## 9. トラブルシューティング

### 共通の問題と解決策

1. **v6からのアップグレード時の問題**
   - インポートを `react-router-dom` から `react-router` に変更する必要があります
   - すべての future フラグを有効にしてから移行すると、破壊的な変更を避けられます

2. **ベースネームの設定**
   - Frameworkモードでベースネームを設定するには、`react-router.config.ts`に`basename`プロパティを追加します
   ```typescript
   // react-router.config.ts
   import type { Config } from "@react-router/dev/config";
   export default {
     basename: "/my-app",
   } satisfies Config;
   ```

3. **サーバーレンダリングとクライアントレンダリングの不一致**
   - `clientLoader.hydrate = true as const` を使用して、ハイドレーション前にクライアントローダーを実行させる
   - `HydrateFallback` コンポーネントを提供してローディング中のUIを表示する

4. **他のバンドラーの利用**
   - Frameworkモードは現在主にViteをサポートしていますが、Rspackなど他のバンドラー向けのプラグインも開発されています

## 参考リンク

- [React Router 公式ドキュメント](https://reactrouter.com/)
- [Frameworkモードインストール](https://reactrouter.com/start/framework/installation)
- [Frameworkモードルーティング](https://reactrouter.com/start/framework/routing)
- [ルートモジュール](https://reactrouter.com/start/framework/route-module)
- [レンダリング戦略](https://reactrouter.com/start/framework/rendering)
- [データローディング](https://reactrouter.com/start/framework/data-loading)
- [アクション](https://reactrouter.com/start/framework/actions)
- [ナビゲーション](https://reactrouter.com/start/framework/navigating)
- [ペンディングUI](https://reactrouter.com/start/framework/pending-ui)
- [テスト](https://reactrouter.com/start/framework/testing)
- [デプロイ](https://reactrouter.com/start/framework/deploying)
