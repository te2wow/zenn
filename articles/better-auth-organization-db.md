---
title: "better-auth の organization plugin は何を解決するのか"
emoji: "🏢"
type: "tech"
topics: ["betterauth", "nextjs", "typescript", "auth", "drizzle"]
published: false
---

SaaS を作っていると、「組織」という概念が必要になることがあります。ユーザーが個人として存在するだけでなく、会社やチームに属して、その中で権限が分かれて、他の人を招待できる、という機能です。

[better-auth](https://better-auth.com) の [organization plugin](https://better-auth.com/docs/plugins/organization) は、この一式をプラグイン 1 つで用意してくれます。この記事では、組織を作ったときにどのテーブルへ何が書き込まれるのかを中心に、注意点をまとめます。

動作確認用のデモアプリを作ったので、そちらも合わせて参照してください。

- デモアプリ: https://github.com/te2wow/better-auth-org-demo

検証に使ったバージョンは `better-auth@1.6.25` / `next@16.2.12` / `drizzle-orm@0.45.2` です。データベースは SQLite（better-sqlite3）を使っています。

## マルチテナントを自前で実装する場合の作業範囲

「組織」を自前で実装する場合、必要な作業は次のように広がります。

- 組織テーブルと、ユーザーと組織を結ぶ中間テーブルを設計する
- 「いま操作している組織はどれか」をセッションに持たせる
- 招待メールを送り、招待の有効期限を管理し、承諾時にメンバーへ変換する
- owner / admin / member のような役割ごとに、操作できることを分ける
- 上記のテーブルすべてについて、他の組織のデータへアクセスできないようにする

organization plugin はこの領域をまとめて引き受けます。逆に言うと、課金・プラン管理・組織ごとのリソース分離は対象外です。プラグインが扱うのは「組織・メンバー・招待・権限」の 4 つです。

この範囲の狭さが better-auth らしいところだと思います。組織機能に共通して必要な部分だけを提供し、その先のドメイン固有の判断はアプリ側に委ねる、という切り分けになっています。

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

この状態で CLI の `generate` を実行すると、必要なテーブル定義が生成されます。

```bash
npx auth@latest generate --config lib/auth.ts --output db/schema.ts
```

## 生成されるスキーマを読む

生成された Drizzle スキーマから、organization plugin が追加する部分だけを抜き出します。`member` と `invitation` に付くインデックス定義と、`invitation.createdAt` の SQL デフォルト式は省略しています。

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

（1）ユーザーと組織の関連は `member` テーブルが保持する。ロールも `member` のカラムとして持ちます。`(userId, organizationId)` の組に対してロールが決まるので、「同じユーザーが組織 A では owner、組織 B では member」という状態を表現できます。

（2）外部キーが `onDelete: "cascade"` になっている。ただし、組織を削除したときに `member` と `invitation` が消えるのは、この外部キー制約が主因ではありません。プラグインの `deleteOrganization` が `member`、`invitation`、`organization` の順に明示的に DELETE を発行しています。外部キーの cascade は、外部キー制約が有効な DB では二重の保険として働く、という位置づけです。いずれにせよ組織削除は破壊的なので、論理削除にしたい場合は `disableOrganizationDeletion: true` で削除自体を塞いで、独自の運用を組む必要があります。

（3）`session` テーブルにカラムが追加される。既存の `session` テーブルに `activeOrganizationId` が追加されます。

```typescript
export const session = sqliteTable("session", {
  // ...既存のカラム
  activeOrganizationId: text("active_organization_id"),
});
```

選択中の組織をセッションの状態として保持する設計です。

なお `teams` を有効にしていない場合、`team` / `teamMember` テーブル、`session.activeTeamId`、`invitation.teamId` は生成されません。必要なテーブルだけが増える作りになっています。

## 組織を作ると何が INSERT されるか

ここが本題です。`createOrganization` を 1 回呼ぶと、2 つのテーブルにそれぞれ 1 行が INSERT されます。

デモアプリでは画面の右半分に `demo.db` の中身を 1 秒ごとに読み直して表示し、新しく追加された行を緑色で強調しています。ボタンを 1 回押した結果がこれです。

![createOrganization を押すと organization テーブルと member テーブルにそれぞれ 1 行が INSERT され、session の activeOrganizationId が更新される](/images/better-auth-organization-db/01-create-org.gif)

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

`created_at` は両者で一致していますが、これは同じミリ秒内に処理されたためで、値を共有しているわけではありません。実装上は、`createOrganization` エンドポイントが組織を INSERT したあと、続けて作成者の `member` を INSERT しています。

つまり作成者の `member` 行をアプリ側で INSERT する必要はありません。「組織を作ったあとに、作成者を管理者として登録する」という処理を自前で書く必要はないということです。

作成者のロールは `creatorRole` で変えられます。既定は `owner` です。

```typescript
organization({
  creatorRole: "admin", // 既定は "owner"
});
```

### 作成直後に処理を挟むときは hook を使う

「組織を作ったら初期データを投入したい」という要件はよくあります。その場合、API のレスポンスを受け取ったあとにアプリ側で処理を書くより、`organizationHooks` に寄せるほうが漏れにくいと思います。呼び出し元がクライアントでもサーバでも、組織の作成処理を経由する限り実行されるためです。

```typescript
organization({
  organizationHooks: {
    afterCreateOrganization: async ({ organization, member, user }) => {
      await setupDefaultResources(organization.id);
    },
  },
});
```

`setupDefaultResources` は説明用の仮の関数です。デモアプリではサーバログへの出力に置き換えています。

`beforeCreateOrganization` で例外を投げると、組織の INSERT 自体が実行されません。「特定ドメインのメールアドレスの人しか組織を作れない」といった制約はここで表現できます。一方 `beforeAddMember` は組織の INSERT が終わったあとに呼ばれるので、そこで例外を投げると組織だけが残ります。制約を入れる位置には注意が必要です。

## 招待フローでの書き込み順序

招待は「招待を作る」と「招待を受ける」の 2 段階に分かれ、それぞれ別のタイミングで書き込みが発生します。

![inviteMember で invitation に status=pending の行が INSERT され、承諾すると status が accepted に UPDATE されて member に 1 行 INSERT される](/images/better-auth-organization-db/02-invite-accept.gif)

（1）`inviteMember` を呼ぶと `invitation` に 1 行 INSERT される。この時点では `member` には何も INSERT されません。`status` は `pending` です。

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

`expiresAt` は既定で 48 時間後です（レスポンスの `createdAt` は省略しています）。`invitationExpiresIn` で秒単位で変更できます。

（2）`acceptInvitation` を呼ぶと、まず `invitation` の `status` が `accepted` に UPDATE され、続けて `member` に 1 行 INSERT される。あわせてセッションの `activeOrganizationId` もその組織に更新されます。

```
sqlite> select id, user_id, role from member;
0XiANXw9BDZE02seoMWBdV3gWPKcbS2p  gvY62eemm1VxqZLRTqRBVt1QgWhaEyAL  owner
HdhFfB7TZPrDUvNqYY8GRcoI0Opa7xRT  6KmEPqHcLG0XCdDSo8NRaApyrkRVp3zB  member

sqlite> select id, status from invitation;
SovbVKKXBfXzwE3z255LZoSRmDbeWALp  accepted
```

ここで注意したいのは、承諾しても `invitation` の行は DELETE されず、`status` が `accepted` に UPDATE されるだけという点です。履歴が残るのは追跡のうえでは便利だと思いますが、行は自動では消えず、期限切れの招待も `pending` のまま残ります。メールアドレスを含むテーブルなので、保持期間はアプリ側で決める必要があります。

### メール送信は自前で用意する

`sendInvitationEmail` を設定しない場合、`invitation` の行は作成されますがメールは送信されません。プラグインが渡すのは招待 ID を含むデータまでで、送信処理はアプリ側で実装します。

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

`sendMail` は説明用の仮の関数です。デモアプリでは招待リンクをサーバログに出力するだけにしています。

招待リンクを受け取る側のページもアプリ側で用意します。デモアプリでは `/accept-invitation/[id]` で `acceptInvitation` を呼ぶだけの画面を置いています。

## 権限モデル

既定のロールは owner / admin / member の 3 つで、権限は次のように分かれています。

| ロール | できること |
| --- | --- |
| owner | 組織の削除を含むすべての操作 |
| admin | 組織の削除と owner の変更を除く操作 |
| member | 既定では参照のみ。作成・更新・削除の権限を持たない |

member ロールのユーザーで操作した場合の挙動が次のとおりです。

![member ロールのユーザーでは hasPermission が success: false を返し、組織削除も 403 で拒否される](/images/better-auth-organization-db/03-permission.gif)

`hasPermission` はサーバ側で権限を評価した結果を返します。

```typescript
const res = await authClient.organization.hasPermission({
  permissions: { organization: ["delete"] },
});
// member ロールなら res.data は { error: null, success: false }
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

プラグインが持つテーブルへの操作は、各エンドポイントがサーバ側で権限を確認しています。一方、`project` のようにアプリ側で追加したリソースについては、サーバ側で `auth.api.hasPermission` を呼んで判定するのがアプリの責任です。クライアントの `hasPermission` の結果だけを根拠に独自 API の認可を省かないようにしてください。

### 独自の権限を追加するときは defaultStatements をマージする

`createAccessControl` で独自リソースを定義できます。このとき既定の statement をマージしないと、organization / member / invitation / team / ac に対する既定の権限が定義から失われます。

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

なお access 関連は `better-auth/plugins` ではなく `better-auth/plugins/access` から import します。プラグイン全体を経由するとバンドルサイズが大きくなるためです。

## その他の注意点

ロールは単一のカラムに格納される。`member.role` は `text` 型です。複数ロールを付与した場合もカンマ区切りで同じカラムに格納されます。そのため `role = 'admin'` のような完全一致の条件では、複数ロールを持つ行が該当しなくなります。

メンバー数と保留中の招待数には既定の上限がある。`membershipLimit`（組織あたりのメンバー数）の既定は 100、`invitationLimit` の既定も 100 です。`invitationLimit` はドキュメント上「ユーザーあたりの招待数」と説明されていますが、1.6.25 の実装では組織あたりの `pending` な招待数を数えています。大きな組織を扱うなら明示的に引き上げる必要があります。なお `membershipLimit` は `listMembers` の既定の取得件数も兼ねています。ユーザーごとの組織数は `organizationLimit` で制限できますが、こちらは既定では無制限です。

`slug` には unique 制約が付く。テーブル全体で一意になるため、ユーザーが自由に入力する場合は `checkSlug` で事前確認したうえで、それでも競合したときの一意制約違反をハンドリングする必要があります。事前確認と INSERT の間に別のリクエストが割り込む可能性があるためです。デモでは組織名から機械的に生成しているだけなので、実運用ではもう少し丁寧に扱う必要があります。

サーバから `createOrganization` を呼ぶときの `userId` はセッションが無いときだけ使われる。セッションヘッダを渡さずに呼ぶ場合は `userId` で対象ユーザーを指定します。セッションがある場合はセッションのユーザーが優先され、`userId` はエラーにならず無視されます。

## まとめ

organization plugin が担当するのは「組織・メンバー・招待・権限」の 4 つで、スキーマの定義からアクセス制御の実施までが含まれます。特に、組織の作成時に owner の `member` 行まで INSERT される点と、権限チェックがサーバ側のエンドポイントで実施される点が、自前実装と比べたときの利点だと思います。

有効にしたプラグインの分だけテーブルが増え、その先の判断はアプリ側に残るという薄さも、扱いやすいところだと思います。

## 参考

- [Organization | Better Auth](https://better-auth.com/docs/plugins/organization)
- [organization.mdx（ドキュメントのソース）](https://github.com/better-auth/better-auth/blob/main/docs/content/docs/plugins/organization.mdx)
- [デモアプリ: te2wow/better-auth-org-demo](https://github.com/te2wow/better-auth-org-demo)
