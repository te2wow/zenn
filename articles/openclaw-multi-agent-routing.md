---
title: "OpenClawマルチエージェントとroutingによるミニmoltbookの運用"
emoji: "🤖"
type: "tech"
topics: ["openclaw", "discord", "multiagent", "ai", "agent"]
published: false
---

![](/images/openclaw-multi-agent-routing/01-title.png)

## OpenClaw とは

[OpenClaw](https://docs.openclaw.ai) は、自分のマシンで動かす **self-hosted gateway** です。「a self-hosted gateway that connects your favorite chat apps and channel surfaces」を掲げており、Discord / Slack / Telegram / WhatsApp / iMessage / Microsoft Teams / Signal / Google Chat / Matrix / Zalo などのチャットチャネルと、AI コーディングエージェントを橋渡しする役割を持ちます。

公式が掲げている特徴は次の3つです。

- **Self-hosted** — runs on your hardware, your rules
- **Multi-channel** — one Gateway serves built-in channels plus bundled or external channel plugins simultaneously
- **Agent-native** — built for coding agents with tool use, sessions, memory, and multi-agent routing

つまり「1 つの Gateway が複数のチャネルと複数のエージェントを束ねる」アーキテクチャになっています。本記事では、その中でも **マルチエージェント** と **チャネルルーティング** に焦点を当てます。

参考:

- https://docs.openclaw.ai/concepts/multi-agent
- https://docs.openclaw.ai/channels/channel-routing
- https://docs.openclaw.ai/channels/discord

## なぜマルチエージェントを使いたくなるのか

最近の楽しみは、エージェント同士の会話を眺めること（たまに乱入する）です。やりたいことを並べると、自然とマルチエージェント構成に行き着きます。

- Discord Bot を n 個使いたい
  - 「一般 Channel 用」と「コーディング専用」でそれぞれ違うエージェント・違う Channel に接続したい
- Bot ごとに設定を分けたい
  - Bot Token・Guild・Channel を別管理
  - セッションを混在させたくない
  - 会話履歴・認証情報を完全分離したい
- 特定の Channel だけ上位の Claude モデルに
  - 深い作業用 Bot だけモデルを変えたい

これらはすべて OpenClaw のマルチエージェント設計で実現できます。

## マルチエージェントとは — 1 Gateway で複数の独立した脳

![](/images/openclaw-multi-agent-routing/02-multi-agent-concept.png)

公式ドキュメント（[concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent)）では、Agent は「完全にスコープされた脳」と定義されており、独立した workspace、state directory（`agentDir`）、session store を持ちます。複数の isolated agents を 1 つの Gateway で動かせるのが OpenClaw のマルチエージェントです。

ポイントは3つです。

1. **各エージェントは完全独立**
   - workspace / agentDir / sessions がそれぞれ独立。認証情報も混在しない。
2. **1つの Gateway が全部を束ねる**
   - WhatsApp / Discord / Telegram / Slack などのチャネルを1つのサーバーで管理。Bot を増やしてもサーバーは1台でよい。
3. **ユースケース例**
   - 雑談 Bot とコーディング Bot を同じサーバーで運用
   - 家族用・仕事用を分離
   - エージェントごとに Claude のモデルを変える（軽量モデル vs 上位モデル）

エージェントは次のコマンドで追加することができます。

```bash
openclaw agents add <agentId>
```

`openclaw agents list --bindings` で一覧と binding を表示できます。

### エージェントごとに独立しているもの

公式ドキュメント上、各 agent は次のディレクトリを所有します。

| 用途 | パス |
|---|---|
| メイン設定 | `~/.openclaw/openclaw.json` |
| State 全体 | `~/.openclaw`（`OPENCLAW_STATE_DIR`） |
| Workspace | `~/.openclaw/workspace[-<agentId>]` |
| agentDir | `~/.openclaw/agents/<agentId>/agent` |
| Sessions | `~/.openclaw/agents/<agentId>/sessions` |
| 認証プロファイル | `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` |

認証情報は per-agent で共有されません。別 agent に認証をコピーしたい場合は `auth-profiles.json` を明示的にコピーする必要があります。`agentDir` を agent 間で再利用すると auth/session collisions が起きるため避けてくださいとされています。

## Discord Bot の設定例

![](/images/openclaw-multi-agent-routing/03-discord-bot-config.png)

Discord 用の設定を、`~/.openclaw/openclaw.json` に書きます。

参加サーバーの guild ID を設定する例:

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          guilds: {
            "123456789012345678": { // ← guild ID
              channels: {
                "222222222222": { allow: true },
              },
            },
          },
        },
      },
    },
  },
}
```

Bot 同士で会話させる場合は `allowBots: true` を有効にします。

```json5
{
  channels: {
    discord: {
      accounts: {
        coding: {
          token: "BOT_TOKEN_CODING",
          allowBots: true, // ← Bot 同士で会話させる場合
          guilds: { /* ... */ },
        },
      },
    },
  },
}
```

公式ドキュメント（[channels/discord](https://docs.openclaw.ai/channels/discord)）には、`channels.discord.allowBots=true` でも有効化できると記載があります。Bot 同士の会話を許可するときは「ループ動作を避けるため、strict mention と allowlist を併用すべき」とも書かれているので、`requireMention: true` などと組み合わせて運用するのが安全です。


## 1 つの OpenClaw から複数のチャネル/Bot へどう振り分けているか

ここからが本題のチャネルルーティングです。

### Routing の基本思想

![](/images/openclaw-multi-agent-routing/04-routing-philosophy.png)

公式ドキュメント（[channels/channel-routing](https://docs.openclaw.ai/channels/channel-routing)）の冒頭にこうあります。

> "OpenClaw routes replies back to the channel where a message came from. The model does not choose a channel; routing is deterministic and controlled by the host configuration."

要点は3つです。

1. **AI はルーティングしない**
   - モデルはチャンネルを選ばない。ルーティングは config が完全に制御する。
2. **決定論的**
   - 同じ入力に対して常に同じエージェントへ。挙動が予測・デバッグしやすい。
3. **完全分離**
   - エージェントごとに workspace / auth / sessions が独立して存在する。

「LLM にルートを判断させない」という割り切りは、運用に乗せるうえでとても重要です。

### Binding の基本構造

![](/images/openclaw-multi-agent-routing/05-binding-structure.png)

受信メッセージは `channel` / `accountId` / `peer` を持ち、`bindings` のルールに対して順に照合され、最終的に `agentId` が決まります。決まった agent の workspace + sessions にメッセージがルーティングされます。

binding は次のように書きます。

```json5
// openclaw.json — bindings の構造
{
  bindings: [
    {
      agentId: "coding",        // → このエージェントへ
      match: {
        channel:   "discord",   // チャンネル
        accountId: "coding",    // Bot アカウント（省略=デフォルトのみマッチ）
        peer: { kind: "group", id: "333333333333" }, // 特定チャンネル ID
      },
    },
  ],
}
```

公式ドキュメントには次のように明記されています。

> "When a binding includes multiple match fields, all provided fields must match for that binding to apply."

つまり複数フィールドを指定した場合は **すべて一致する必要がある（AND 条件）** です。

### Routing ルール — most-specific wins（8段階）

![](/images/openclaw-multi-agent-routing/06-most-specific-wins.png)

複数の binding が候補になり得るとき、もっとも具体的なルールが勝ちます。公式の優先度は以下の8段階です。

| 優先度 | ルール | 条件 |
|---|---|---|
| 1 | Exact peer match | `peer.kind` + `peer.id` の完全一致（最高優先） |
| 2 | Parent peer match | スレッド継承（thread 内返信） |
| 3 | Guild + roles match | Discord — `guildId` + `roles`（AND） |
| 4 | Guild match | Discord — `guildId` のみ |
| 5 | Team match | Slack — `teamId` |
| 6 | Account match | `accountId` on channel |
| 7 | Channel match | `accountId: "*"` — チャンネル全体 |
| 8 | Default agent | `agents.list[].default` → 先頭エントリ → `main`（fallback） |

同一段でマッチが複数発生した場合は「first one in config order wins」、つまり設定順で先に書いたものが勝ちます。

直感的には「メッセージが具体的であればあるほど、より specific なルールが当たる」と捉えると分かりやすいです。例えば DM の peer.id が一致するなら 1 が当たり、guild にだけ反応させたいなら 4、チャンネル全体に1つのエージェントを割り当てるなら 7、何もマッチしなければ 8 の default agent に落ちます。

### スライドで触れていない、知っておくと便利な公式機能

スライドには出てこなかったものの、運用するうえで知っておくと便利な公式機能をいくつか補足します。

- **`groupChat.mentionPatterns`**: グループでの mention gating。特定のエージェントだけ反応させたいときに使う。
- **`tools.agentToAgent.enabled`**: agent 間通信は **off by default**。Bot 同士で会話させるときに必要。
- **`replyToMode`**（Discord）: Discord ネイティブのリプライ挙動。`off` / `first` / `all` / `batched`。
- **`requireMention`** / **`dmPolicy`**: 反応条件と DM のアクセス制御。
- **`historyLimit`**: Guild チャネルの履歴取得上限（デフォルト 20）。
- **`threadBindings`**: スレッド単位の session バインド。

特に「`tools.agentToAgent.enabled` は off by default」は、`allowBots: true` だけ設定して動かないときに踏みやすいので注意です。

## 運用してみての所感

ミニ moltbook の運用は、エージェントを増やすこと自体が目的になりがちです。最低限「**ルーティングを LLM に任せない**」「**workspace/sessions/auth は agent 単位で分離する**」「**Bot 同士の会話には上限を入れる**」の3つを守れば、楽しく運用できる構成だと思います。

enjoy!! マルチエージェント!!
