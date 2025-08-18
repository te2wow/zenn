---
title: "TypeScriptのas型アサーションをZodで型安全に置き換える"
emoji: "🛡️"
type: "tech"
topics: ["typescript", "zod", "type-safety"]
published: false
---
Effective Typescriptな勢いで書いています

TypeScriptで開発していると、外部APIのレスポンスや設定ファイルの読み込みなど、実行時まで型が確定しないデータを扱うことがよくあります。このような場面で`as`を使った型アサーションに頼ってしまうと、ランタイムエラーのリスクが高まります。

本記事では、Zodの`z.infer`（型推論）、`parse`（ランタイム検証）、`refine`（カスタムバリデーション）、`transform`（データ変換）を使って、`as`型アサーションを避ける方法をまとめます。

## 従来の問題：型アサーションの危険性

### Before: 型アサーション（危険）

```typescript
// 危険な例：型アサーション
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  
  // ここが危険：実際のデータ構造を検証していない
  return data as User;
}

// 実行時エラーの可能性
const user = await fetchUser("123");
console.log(user.name.toUpperCase()); // user.nameがundefinedの場合エラー
```

### 問題点

1. **ランタイム検証がない**：APIから返されるデータの形式が変わってもエラーが発生しない
2. **型の不整合**：TypeScriptは型が正しいと信じてしまう
3. **デバッグが困難**：実行時まで問題が発覚しない

## Zodの基本概念

Zodは「Schema first」のアプローチを取ります。まずスキーマを定義し、そこから型を推論するという流れです。これにより、データの検証と型定義を一箇所で管理できます。

### z.inferによる型推論の仕組み

`z.infer`は、Zodスキーマから対応するTypeScript型を自動生成するユーティリティです。スキーマを変更すれば型も自動的に更新されるため、型とスキーマの不整合を防げます。

```typescript
// スキーマから型を推論
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
});

// z.inferで型を自動生成
type User = z.infer<typeof UserSchema>;
// => { id: number; name: string; email: string; }
```

この仕組みにより、interfaceの重複定義が不要になり、[信頼できる唯一の情報源](https://ja.wikipedia.org/wiki/%E4%BF%A1%E9%A0%BC%E3%81%A7%E3%81%8D%E3%82%8B%E5%94%AF%E4%B8%80%E3%81%AE%E6%83%85%E5%A0%B1%E6%BA%90)を実現できます。

## 解決策：Zodによる型安全なアプローチ

### After: Zodによる型安全な実装

```typescript
import { z } from "zod";

// Zodスキーマの定義
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().optional(),
});

// z.inferで型を自動生成
type User = z.infer<typeof UserSchema>;

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  
  // ランタイムでデータを検証
  const validatedUser = UserSchema.parse(data);
  return validatedUser; // 型安全が保証された状態
}

// 安全に使用可能
const user = await fetchUser("123");
console.log(user.name.toUpperCase()); // user.nameは確実にstring
```

## 実践例1：設定ファイルの読み込み

### Before: 型アサーション版

```typescript
interface Config {
  port: number;
  database: {
    host: string;
    port: number;
    name: string;
  };
  features: string[];
}

function loadConfig(): Config {
  const configFile = fs.readFileSync("config.json", "utf-8");
  const config = JSON.parse(configFile);
  
  // 設定ファイルの内容が正になってしまう
  return config as Config;
}
```

### After: Zodを使った実装

```typescript
import { z } from "zod";

const ConfigSchema = z.object({
  port: z.number().min(1).max(65535),
  database: z.object({
    host: z.string().min(1),
    port: z.number().min(1).max(65535),
    name: z.string().min(1),
  }),
  features: z.array(z.string()),
});

type Config = z.infer<typeof ConfigSchema>;

function loadConfig(): Config {
  const configFile = fs.readFileSync("config.json", "utf-8");
  const rawConfig = JSON.parse(configFile);
  
  // 設定値を検証してから返す
  return ConfigSchema.parse(rawConfig);
}
```

## 実践例2：フォームバリデーション

### Before: 手動バリデーション

```typescript
interface FormData {
  username: string;
  email: string;
  age: number;
}

function validateForm(data: any): FormData | null {
  if (typeof data.username !== "string" || data.username.length < 3) {
    return null;
  }
  if (typeof data.email !== "string" || !data.email.includes("@")) {
    return null;
  }
  if (typeof data.age !== "number" || data.age < 0) {
    return null;
  }
  
  return data as FormData; // まだ型アサーションが必要
}
```

### After: Zodを使った実装

```typescript
import { z } from "zod";

const FormSchema = z.object({
  username: z.string().min(3, "ユーザー名は3文字以上である必要があります"),
  email: z.string().email("有効なメールアドレスを入力してください"),
  age: z.number().min(0, "年齢は0以上である必要があります"),
});

type FormData = z.infer<typeof FormSchema>;

function validateForm(data: unknown): FormData {
  return FormSchema.parse(data); // バリデーションと型変換を同時に実行
}

// 使用例
function handleFormSubmission(rawData: unknown) {
  const validData = validateForm(rawData);
  // validDataは確実にFormData型
  console.log(`ユーザー: ${validData.username}, 年齢: ${validData.age}`);
}
```

## refineでカスタムバリデーション

Zodの`refine`メソッドは、基本的なバリデーション（文字列の長さ、数値の範囲など）を超えた、より複雑な検証ができます。

### refineの動作原理

`refine`は以下の流れで動作します：
1. 基本的なスキーマバリデーションが成功
2. `refine`で指定した関数が実行される
3. 関数が`false`を返すかエラーを投げると、バリデーション失敗

```typescript
const schema = z.string().refine(
  (value) => value.includes('@'), // バリデーション関数
  { message: 'メールアドレス形式ではありません' } // エラーメッセージ
);
```

### Before: 手動でバリデーション

```typescript
interface UserData {
  username: string;
  email: string;
  age: number;
}

function validateUser(data: any): UserData | null {
  if (typeof data.username !== "string" || data.username.length < 3) {
    return null;
  }
  if (typeof data.email !== "string" || !data.email.endsWith("@company.com")) {
    return null;
  }
  if (typeof data.age !== "number" || data.age < 18) {
    return null;
  }
  
  return data as UserData; // 手動バリデーション後もasが必要
}
```

### After: refineでカスタムバリデーション

```typescript
const UserSchema = z.object({
  username: z.string().min(3),
  email: z.string().email().refine(
    email => email.endsWith("@company.com"),
    "会社のメールアドレスを使用してください"
  ),
  age: z.number().min(18, "18歳以上である必要があります"),
});

type UserData = z.infer<typeof UserSchema>;

function validateUser(data: unknown): UserData {
  return UserSchema.parse(data); // カスタムバリデーション込みで安全に変換
}
```

## transformでデータ変換

`transform`メソッドは、バリデーション成功後にデータを別の形に変換する機能です。単なる型キャストではなく、実際のデータ操作を伴います。

### transformの特徴

1. **パイプライン処理**：複数のtransformを連鎖させて段階的に変換可能
2. **型安全性**：変換後の型も正しく推論される
3. **エラーハンドリング**：変換中にエラーが発生した場合も適切に処理

```typescript
// 文字列 → 数値 → 文字列の変換パイプライン
const schema = z.string()
  .transform(str => parseInt(str, 10)) // string → number
  .refine(num => !isNaN(num), 'Invalid number')
  .transform(num => `ID: ${num.toString().padStart(4, '0')}`); // number → string

// 使用例
schema.parse("123"); // "ID: 0123"
```

`as`では型変換しかできませんが、Zodなら実際のデータ変換も同時に行えます。

### Before: 手動でデータ変換

```typescript
interface ProcessedData {
  email: string;
  normalizedName: string;
  ageGroup: "adult" | "minor";
}

function processData(raw: any): ProcessedData {
  const email = (raw.email as string).toLowerCase();
  const normalizedName = (raw.name as string).trim().toLowerCase();
  const ageGroup = (raw.age as number) >= 18 ? "adult" : "minor";
  
  return { email, normalizedName, ageGroup } as ProcessedData;
}
```

### After: transformで変換

```typescript
const ProcessedDataSchema = z.object({
  email: z.string().email().transform(email => email.toLowerCase()),
  name: z.string().transform(name => name.trim().toLowerCase()),
  age: z.number().min(0),
}).transform(data => ({
  email: data.email,
  normalizedName: data.name,
  ageGroup: data.age >= 18 ? "adult" as const : "minor" as const,
}));

type ProcessedData = z.infer<typeof ProcessedDataSchema>;

function processData(raw: unknown): ProcessedData {
  return ProcessedDataSchema.parse(raw); // バリデーションと変換を同時実行
}
```

## safeParse を使ったエラーハンドリング

Zodには2つの主要なパース方法があります：

### parseとsafeParseの違い

| メソッド | エラー時の動作 | 戻り値の型 | 使用場面 |
|---------|-------------|----------|---------|
| `parse` | 例外を投げる | `T` | エラーが予期されない場面 |
| `safeParse` | Result型を返す | `SafeParseResult<T>` | エラーハンドリングが必要な場面 |

### safeParseの戻り値構造

```typescript
// 成功時
{ success: true, data: T }

// 失敗時  
{ success: false, error: ZodError }
```

この構造により、TypeScriptの型ガードを活用した安全なエラーハンドリングが可能になります。`parse`メソッドはバリデーションエラー時に例外を投げますが、`safeParse`を使うことで、エラーハンドリングをより柔軟に行えます。

### Before: parseメソッド

```typescript
function processApiResponse(data: unknown): User | null {
  // parseは例外を投げる可能性がある
  const user = UserSchema.parse(data);
  return user;
}
```

### After: safeParseメソッド

```typescript
function processApiResponse(data: unknown): User | null {
  const result = UserSchema.safeParse(data);
  
  if (result.success) {
    // バリデーション成功時はresult.dataに型安全なデータが入る
    return result.data; // User型として確実に使用可能
  } else {
    // エラー情報はresult.errorに含まれる
    console.error("バリデーションエラー:", result.error.errors);
    return null;
  }
}
```

### safeParseの実践例

```typescript
const ApiResponseSchema = z.object({
  users: z.array(UserSchema),
  total: z.number(),
});

type ApiResponse = z.infer<typeof ApiResponseSchema>;

async function fetchUsers(): Promise<ApiResponse | null> {
  const response = await fetch("/api/users");
  const data = await response.json();
  
  const result = ApiResponseSchema.safeParse(data);
  
  if (result.success) {
    return result.data; // 型安全なApiResponse
  }
  
  // エラーログを出力して null を返す
  console.error("API応答の形式が不正です:");
  result.error.errors.forEach(err => {
    console.error(`- ${err.path.join(".")}: ${err.message}`);
  });
  
  return null;
}

// 使用例
const users = await fetchUsers();
if (users) {
  console.log(`取得したユーザー数: ${users.total}`);
  users.users.forEach(user => {
    console.log(user.name); // 型安全にアクセス可能
  });
}
```

## まとめ

## パフォーマンスとベストプラクティス

### スキーマの再利用

同じスキーマを何度も作成するのではなく、モジュール化して再利用することが重要です：

```typescript
// 共通スキーマ
export const EmailSchema = z.string().email();
export const IdSchema = z.number().positive();

// 他のスキーマで再利用
const UserSchema = z.object({
  id: IdSchema,
  email: EmailSchema,
  name: z.string().min(1),
});
```

### エラーメッセージのカスタマイズ

Zodでは詳細なエラーメッセージを設定できます：

```typescript
const schema = z.object({
  password: z.string()
    .min(8, "パスワードは8文字以上である必要があります")
    .regex(/[A-Z]/, "大文字を含む必要があります")
    .regex(/[0-9]/, "数字を含む必要があります"),
});
```

## まとめ

Zodを活用することで：

1. **ランタイム検証**：実行時にデータの整合性を確認
2. **型安全性**：`as`を使わずに正確な型を取得
3. **保守性**：スキーマから型が自動生成されるため、変更に強い
4. **エラーハンドリング**：具体的で分かりやすいエラーメッセージ
5. **開発効率**：スキーマと型定義の一元管理

型アサーションを使わずに済むZodの活用で、ランタイムエラーのリスクを減らし、より安全なTypeScriptコードを書けます。特に外部データを扱う際は、Zodによる検証層を設けることで、予期しないデータ形式によるバグを事前に防げます。