---
title: "better-auth の organization plugin は何を解決するのか"
emoji: "🏢"
type: "tech"
topics: ["betterauth", "nextjs", "typescript", "auth", "drizzle"]
published: false
---

SaaS を作っていると、ほぼ必ず「組織」という概念が必要になります。ユーザーが個人として存在するだけでなく、会社やチームに属して、その中で権限が分かれて、他の人を招待できる、というやつです。

[better-auth](https://better-auth.com) の [organization plugin](https://better-auth.com/docs/plugins/organization) は、この一式をプラグイン 1 つで用意してくれます。この記事では、組織を作ったときにデータベースで実際に何が起きているのかを中心に、注意点をまとめます。

動作確認用のデモアプリを作ったので、そちらも合わせて参照してください。

- デモアプリ: https://github.com/te2wow/better-auth-org-demo

検証に使ったバージョンは `better-auth@1.6.25` / `next@16.2.12` / `drizzle-orm@0.45.2` です。データベースは SQLite を使っています。

## マルチテナントを自前で作ると何が面倒なのか

「組織」を自前で実装しようとすると、やることは意外と多くなります。

- 組織テーブルと、ユーザーと組織を結ぶ中間テーブルを設計する
- 「いま操作している組織はどれか」をセッションに持たせる
- 招待メールを送り、トークンの有効期限を管理し、承諾時にメンバーへ変換する
- owner / admin / member のような役割ごとに、操作できることを分ける
- 上記すべてについて「他人の組織のデータを触れないこと」を保証する

organization plugin はこの領域をまとめて引き受けてくれます。逆に言うと、課金・プラン管理・組織ごとのリソース分離までは面倒を見てくれません。プラグインが持つのはあくまで「組織・メンバー・招待・権限」の 4 つです。

この薄さが better-auth らしいところだと思います。組織まわりで必ず要る土台だけを用意して、その先のドメイン固有の判断はアプリ側に委ねる、という切り分けになっています。テーブルもプラグインを有効にした分しか増えませんし、権限モデルも既定のロールをそのまま使うことも、独自に定義し直すこともできます。フルスタックな SaaS フレームワークを導入して要らない機能ごと抱えるのとは、この点が違います。

## 最小構成

サーバ側はプラグインを足すだけです。

```typescript
import { betterAuth } from "better-auth";
import { organization } from "better-auth/plugins";

export const auth = betterAuth({
  plugins: [organization()],
});
```

クライアント側も同様です。

```typescript
import { createAuthClient } from "better-auth/react";
import { organizationClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  plugins: [organizationClient()],
});
```

この状態で `@better-auth/cli generate` を実行すると、必要なテーブル定義が生成されます。

```bash
npx @better-auth/cli generate --config lib/auth.ts --output db/schema.ts
```

## 生成されるスキーマを読む

生成された Drizzle スキーマから、organization plugin が追加する部分だけを抜き出します。

```typescript
export const organization = sqliteTable("organization", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  slug: text("slug").notNull().unique(),
  logo: text("logo"),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull(),
  metadata: text("metadata"),
});

export const member = sqliteTable("member", {
  id: text("id").primaryKey(),
  organizationId: text("organization_id").notNull()
    .references(() => organization.id, { onDelete: "cascade" }),
  userId: text("user_id").notNull()
    .references(() => user.id, { onDelete: "cascade" }),
  role: text("role").default("member").notNull(),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull(),
});

export const invitation = sqliteTable("invitation", {
  id: text("id").primaryKey(),
  organizationId: text("organization_id").notNull()
    .references(() => organization.id, { onDelete: "cascade" }),
  email: text("email").notNull(),
  role: text("role"),
  status: text("status").default("pending").notNull(),
  expiresAt: integer("expires_at", { mode: "timestamp_ms" }).notNull(),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull(),
  inviterId: text("inviter_id").notNull()
    .references(() => user.id, { onDelete: "cascade" }),
});
```

読みどころは 3 つあります。

（1）`member` が本体である。ユーザーと組織の関係は `member` テーブルが持ちます。ロールもここに乗ります。つまり「同じユーザーが組織 A では owner、組織 B では member」という状態が自然に表現できます。

（2）外部キーが `onDelete: "cascade"` になっている。組織を削除すると、その組織の `member` と `invitation` は SQL レベルで消えます。アプリ側で消し漏らす心配はない代わりに、組織削除は思っているより破壊的です。論理削除にしたい場合は `disableOrganizationDeletion: true` で削除自体を塞いで、独自の運用を組む必要があります。

（3）`session` にカラムが 1 本増えている。既存の `session` テーブルに `activeOrganizationId` が追加されます。

```typescript
export const session = sqliteTable("session", {
  // ...既存のカラム
  activeOrganizationId: text("active_organization_id"),
});
```

「いまどの組織を見ているか」をセッションが持つ設計です。

なお `teams` を有効にしていない場合、`team` / `teamMember` テーブルと `session.activeTeamId` は生成されません。必要なテーブルだけが増える作りになっています。

## 組織を作ると DB では何が起きるか

ここが本題です。`createOrganization` を 1 回呼ぶと、2 つのテーブルに 1 行ずつ入ります。

デモアプリでは画面の右半分に `demo.db` の中身を 1 秒ごとに表示していて、新しく増えた行が緑色に光るようにしています。ボタンを 1 回押した結果がこれです。

![createOrganization を押すと organization テーブルと member テーブルに同時に 1 行ずつ追加され、session の activeOrganizationId が埋まる](/images/better-auth-organization-db/01-create-org.gif)

API のレスポンスを見ると、作られた組織と一緒に `members` が返ってきていることが分かります。

```json
{
  "name": "Acme Inc",
  "slug": "acme-inc",
  "logo": null,
  "createdAt": "2026-08-16T23:29:23.185Z",
  "id": "XPak6XwEiFkbZU3m5QYmuYmhgw4TpELE",
  "members": [
    {
      "organizationId": "XPak6XwEiFkbZU3m5QYmuYmhgw4TpELE",
      "userId": "gvY62eemm1VxqZLRTqRBVt1QgWhaEyAL",
      "role": "owner",
      "createdAt": "2026-08-16T23:29:23.185Z",
      "id": "0XiANXw9BDZE02seoMWBdV3gWPKcbS2p"
    }
  ]
}
```

DB を直接見ても同じです。

```
sqlite> select * from organization;
        id = XPak6XwEiFkbZU3m5QYmuYmhgw4TpELE
      name = Acme Inc
      slug = acme-inc
created_at = 1786922963185

sqlite> select * from member;
             id = 0XiANXw9BDZE02seoMWBdV3gWPKcbS2p
organization_id = XPak6XwEiFkbZU3m5QYmuYmhgw4TpELE
        user_id = gvY62eemm1VxqZLRTqRBVt1QgWhaEyAL
           role = owner
     created_at = 1786922963185
```

`created_at` が両者で完全に一致していることに注目してください。これは「組織を作る」と「作った人を owner として登録する」が 1 つの操作として扱われていることを示しています。

つまり `member` を自分で作る必要はありません。よくある「組織を作ったあとに、作成者を管理者として INSERT する」というコードは不要です。

作成者のロールは `creatorRole` で変えられます。既定は `owner` です。

```typescript
organization({
  creatorRole: "admin", // 既定は "owner"
});
```

### 作成直後に何かしたいときは hook を使う

「組織を作ったら初期データを流し込みたい」という要求はよくあります。その場合、API を叩いたあとにアプリ側で処理を書くのではなく、`organizationHooks` を使うほうが安全です。組織がどの経路で作られても必ず通るためです。

```typescript
organization({
  organizationHooks: {
    afterCreateOrganization: async ({ organization, member, user }) => {
      await setupDefaultResources(organization.id);
    },
  },
});
```

`before` 系の hook で例外を投げると、その操作自体が実行されません。「特定ドメインのメールアドレスの人しか組織を作れない」といった制約はここで表現できます。

## 招待フローで DB が動く順番

招待は「招待を作る」と「招待を受ける」の 2 段階で、それぞれ別のタイミングで DB が動きます。

![inviteMember で invitation が pending として作られ、承諾すると status が accepted になって member が 1 行増える](/images/better-auth-organization-db/02-invite-accept.gif)

（1）`inviteMember` を呼ぶと `invitation` に 1 行入る。この時点では `member` は増えません。状態は `pending` です。

```json
{
  "organizationId": "XPak6XwEiFkbZU3m5QYmuYmhgw4TpELE",
  "email": "bob@example.com",
  "role": "member",
  "status": "pending",
  "expiresAt": "2026-08-18T23:29:32.561Z",
  "inviterId": "gvY62eemm1VxqZLRTqRBVt1QgWhaEyAL",
  "id": "SovbVKKXBfXzwE3z255LZoSRmDbeWALp"
}
```

`expiresAt` は既定で 48 時間後です。`invitationExpiresIn` で秒単位で変更できます。

（2）`acceptInvitation` を呼ぶと `member` が増え、`invitation` の状態が変わる。

```
sqlite> select id, user_id, role from member;
0XiANXw9BDZE02seoMWBdV3gWPKcbS2p  gvY62eemm1VxqZLRTqRBVt1QgWhaEyAL  owner
HdhFfB7TZPrDUvNqYY8GRcoI0Opa7xRT  6KmEPqHcLG0XCdDSo8NRaApyrkRVp3zB  member

sqlite> select id, status from invitation;
SovbVKKXBfXzwE3z255LZoSRmDbeWALp  accepted
```

ここで注意したいのは、承諾しても `invitation` の行は消えず、`status` が `accepted` に変わるだけという点です。招待の履歴が残るのは監査上ありがたいところです。

### メール送信は自前で用意する

`sendInvitationEmail` を設定しないと、招待は作られてもメールは飛びません。プラグインが渡してくれるのは招待 ID までで、送信そのものはアプリの責任です。

```typescript
organization({
  async sendInvitationEmail(data) {
    const inviteLink = `https://example.com/accept-invitation/${data.id}`;
    await sendMail({
      to: data.email,
      subject: `${data.organization.name} に招待されています`,
      body: inviteLink,
    });
  },
});
```

招待リンクを受け取る側のページは自分で作ります。デモアプリでは `/accept-invitation/[id]` で `acceptInvitation` を呼ぶだけの画面を置いています。

## 権限モデル

既定のロールは owner / admin / member の 3 つで、権限は次のように分かれています。

| ロール | できること |
| --- | --- |
| owner | すべて。組織の削除も可能 |
| admin | 組織の削除と owner の変更以外 |
| member | 参照のみ |

実際に member ロールのユーザーで操作するとどうなるかを撮ったのがこちらです。

![member ロールのユーザーは hasPermission が拒否を返し、組織削除も 403 で弾かれる](/images/better-auth-organization-db/03-permission.gif)

`hasPermission` はサーバ側で判定を返します。

```typescript
const res = await authClient.organization.hasPermission({
  permissions: { organization: ["delete"] },
});
// member ロールなら { error: null, success: false }
```

そして、判定を無視して実際に削除を叩いても、サーバ側で止まります。

```json
{
  "message": "You are not allowed to delete this organization",
  "code": "YOU_ARE_NOT_ALLOWED_TO_DELETE_THIS_ORGANIZATION"
}
```

招待も同様に弾かれます。

```json
{
  "message": "You are not allowed to invite users to this organization",
  "code": "YOU_ARE_NOT_ALLOWED_TO_INVITE_USERS_TO_THIS_ORGANIZATION"
}
```

`hasPermission` は UI の出し分けのためのもので、防御そのものはサーバ側のエンドポイントが担当しているという構造です。

### 独自の権限を足すときは defaultStatements を混ぜる

`createAccessControl` で独自リソースを定義できます。ここで既定の statement をマージし忘れると、organization / member / invitation に対する既定の権限が消えます。

```typescript
import { createAccessControl } from "better-auth/plugins/access";
import {
  defaultStatements,
  ownerAc,
  adminAc,
  memberAc,
} from "better-auth/plugins/organization/access";

export const statement = {
  ...defaultStatements, // これを忘れると既定の権限が消える
  project: ["create", "update", "delete"],
} as const;

export const ac = createAccessControl(statement);

export const owner = ac.newRole({
  ...ownerAc.statements,
  project: ["create", "update", "delete"],
});

export const admin = ac.newRole({
  ...adminAc.statements,
  project: ["create", "update"],
});

export const member = ac.newRole({
  ...memberAc.statements,
  project: ["create"],
});
```

上の定義では member にも `project: ["create"]` を与えているので、同じユーザーでも結果が変わります。

```
hasPermission(organization:delete) -> success: false
hasPermission(project:create)      -> success: true
```

定義した `ac` と `roles` は、サーバとクライアントの両方に渡す必要があります。

```typescript
// サーバ
organization({ ac, roles: { owner, admin, member } });

// クライアント
organizationClient({ ac, roles: { owner, admin, member } });
```

なお `better-auth/plugins/access` から import することが推奨されています。`better-auth/plugins` 全体を経由するとバンドルサイズが膨らみます。

## その他の注意点

ロールは文字列 1 列に入る。`member.role` は `text` です。複数ロールを持たせるとカンマ区切りで 1 つの列に入ります。`role = 'admin'` のような素朴な SQL は、複数ロールを使い始めた瞬間に壊れます。

組織の作成数・メンバー数には既定の上限がある。`membershipLimit` の既定は 100、`invitationLimit` の既定も 100 です。大きな組織を扱うなら明示的に引き上げる必要があります。`organizationLimit` でユーザーごとの組織数も制限できます。

`slug` は unique 制約付きである。全体で一意なので、ユーザーが自由に入力する場合は `checkSlug` で事前確認するか、衝突エラーを画面に出す作りが要ります。デモでは組織名から機械的に生成しているだけなので、実運用ではもう少し丁寧に扱う必要があります。

サーバから `createOrganization` を呼ぶときは `userId` とセッションを併用できない。サーバ側でセッションヘッダを渡さずに呼ぶ場合は `userId` を指定します。両方同時には使えません。

## まとめ

organization plugin が引き受けてくれるのは「組織・メンバー・招待・権限」の 4 つで、テーブル設計から認可の実施までが込みになっています。特に、組織作成時に owner の `member` 行まで面倒を見てくれる点と、権限チェックがサーバ側のエンドポイントで実際に効いている点が、自前実装と比べたときの価値だと思います。

必要なテーブルだけが増え、その先の判断はアプリ側に残るという薄さも、扱いやすいところだと思います。

## 参考

- [Organization | Better Auth](https://better-auth.com/docs/plugins/organization)
- [organization.mdx（ドキュメントのソース）](https://github.com/better-auth/better-auth/blob/main/docs/content/docs/plugins/organization.mdx)
- [デモアプリ: te2wow/better-auth-org-demo](https://github.com/te2wow/better-auth-org-demo)
