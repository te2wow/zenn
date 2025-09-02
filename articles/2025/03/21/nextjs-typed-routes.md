---
title: "Next.jsの型付きルートが安定版に！機能と使い方をまとめてみた"
emoji: "🔗"
type: "tech"
topics: ["nextjs", "typescript", "react"]
published: false
---

## はじめに

Next.jsの「型付きルート（Typed Routes）」機能が、遂にNext.js 15.5で安定版となりました！

この機能を使うと、`<Link>`コンポーネントのhrefに存在しないパスを書いた時に、TypeScriptがコンパイル時にエラーを出してくれます。つまり、リンク切れを本番環境にデプロイする前に防げるようになります。

本記事では、安定版になった型付きルートの設定方法と基本的な使い方をまとめて紹介します。

## 型付きルートとは

型付きルートは、Next.jsのファイルベースルーティングを基に、TypeScriptの型を自動生成する機能です。これにより、`<Link>`コンポーネントや`router.push()`で指定するパスが、実際に存在するルートかどうかをTypeScriptが検証してくれます。

### 従来の問題点

```tsx
// 従来：存在しないルートでもコンパイルは通る
<Link href="/non-existent-page">壊れたリンク</Link>

// 本番環境で404エラーになって初めて気づく...
```

### 型付きルートを使った場合

```tsx
// 型付きルート：存在しないルートは型エラーになる
<Link href="/non-existent-page">壊れたリンク</Link>
//     ^^^^ Type error: Argument of type '"/non-existent-page"' is not assignable to parameter of type 'Route'
```

## セットアップ

### Next.js 15.5以降（安定版）

型付きルートの設定は非常にシンプルです。`next.config.ts`に1行追加するだけで有効になります。

```typescript
// next.config.ts
const nextConfig = {
  typedRoutes: true,
};

export default nextConfig;
```

### 従来のバージョン（実験的）

Next.js 13.2〜14.xでは実験的フラグが必要でした。

```typescript
// next.config.ts（13.2〜14.x）
const nextConfig = {
  experimental: {
    typedRoutes: true,
  },
};
```

設定後、`next dev`や`next build`を実行すると、Next.jsが自動的にルートの型定義を生成します。

## 実際の使用例

### 基本的な使い方

```tsx
import Link from 'next/link';

export default function Navigation() {
  return (
    <nav>
      {/* 存在するルートは問題なし */}
      <Link href="/">ホーム</Link>
      <Link href="/about">About</Link>
      <Link href="/blog">ブログ</Link>
      
      {/* 存在しないルートは型エラー */}
      <Link href="/invalid-route">エラーになるリンク</Link>
      //     ^^^^ 型エラー！
    </nav>
  );
}
```

### 動的ルートの場合

動的ルートも型安全に扱えます。15.5では`/blog/${string}`のようなテンプレートリテラル型として生成されます。

```tsx
// app/blog/[slug]/page.tsx が存在する場合

<Link href="/blog/my-first-post">記事を読む</Link>  // OK
<Link href={`/blog/${slug}`}>動的リンク</Link>       // OK
<Link href="/blog">ブログ一覧</Link>               // OK
```

### クエリパラメータ付きのリンク

クエリパラメータも含めて型安全に記述できます。

```tsx
<Link href="/search?q=typescript&category=tech">
  TypeScript記事を検索
</Link>

// router.pushでも同様
import { useRouter } from 'next/navigation';

const router = useRouter();
router.push('/search?q=nextjs');
```

## 開発体験の向上

### エディタの補完機能

型付きルートを有効にすると、VSCodeなどのエディタで`href`属性の補完が効くようになります。

```tsx
<Link href="/">
         {/* ↑ ここで補完候補が表示される */}
```

### リファクタリング時の安全性

ルート構造を変更した際も、TypeScriptの型チェックにより影響範囲を漏れなく把握できます。

```bash
# /blog を /articles に変更した場合
# 型エラーが発生し、修正が必要な箇所が一目瞭然
```

## 注意点とベストプラクティス

### 1. 型定義の生成タイミング

型定義は開発サーバー起動時やビルド時に生成されます。15.5では`next typegen`コマンドでビルドとは独立して型定義のみを生成することも可能です。

```bash
# CI環境での型チェック用
next typegen && tsc --noEmit
```

### 2. 外部リンクの扱い

型付きルートは内部リンクのみを対象とします。外部リンクは通常通り文字列として扱います。

```tsx
// 内部リンク：型チェックあり
<Link href="/about">About</Link>

// 外部リンク：通常の文字列
<a href="https://example.com">外部サイト</a>
```

### 3. 条件付きルーティング

動的にルートを決定する場合も型安全性を保てます。

```tsx
import type { Route } from 'next';

const getRoute = (isLoggedIn: boolean): Route => {
  return isLoggedIn ? '/dashboard' : '/login';
};

<Link href={getRoute(user.isAuthenticated)}>
  {user.isAuthenticated ? 'ダッシュボード' : 'ログイン'}
</Link>
```

## WebpackとTurbopackの対応

Next.js 15以降、型付きルートはWebpackとTurbopackの両方で動作します。どちらのビルドツールを使っていても、同じ設定で型安全性を享受できます。

## まとめ

型付きルートは、Next.jsアプリケーションの信頼性を大幅に向上させる機能です。特に以下のような場合に効果を発揮します：

- 大規模なアプリケーションでリンクが多数存在する
- チーム開発でルート構造の把握が困難
- 頻繁にページ構造をリファクタリングする

Next.js 13.2で実験的機能として登場し、15.5で遂に安定版となりました。設定も1行追加するだけと簡単で、既存のコードベースへの導入も容易です。リンク切れによる本番環境でのエラーを防ぐために、ぜひ活用してみてください。

## 参考リンク

- [Next.js 公式ドキュメント - TypeScript](https://nextjs.org/docs/app/building-your-application/configuring/typescript#statically-typed-routes)
- [Next.js 15.0 リリースノート](https://nextjs.org/blog/next-15)