---
title: "Ant で Hono を動かして、単一バイナリと VM サンドボックスまで試す"
emoji: "🐜"
type: "tech"
topics: ["ant", "hono", "javascript", "typescript", "runtime"]
published: false
---

[Ant](https://antjs.org) は、V8 や JavaScriptCore を使わずに自前のエンジンで動く JavaScript ランタイムです。Ginza.js #11 の [小粒でもパワフルな JS runtime AntJS について](https://speakerdeck.com/comamoca/kotsubu-demo-pawafuruna-js-runtime-antjs-nitsuite) で知って、Hono を載せて触ってみました。

デモは https://github.com/te2wow/ant-hono-demo に置いています。確認した環境は Ant `14.0.ff84a70d.0` / Hono `4.13.2` / `@ant/hono` `0.0.4` / macOS (arm64) です。

## Ant とは

[README](https://github.com/theMackabu/ant/blob/ff84a70d887e507066bd6f4e0f03b626e4ddb2fc/README.md#L28-L36) の比較表からの抜粋です。

| | Ant | Node | Bun | Deno |
| --- | --- | --- | --- | --- |
| Binary size | ~9 MB | ~140 MB | ~61 MB | ~77 MB |
| Cold start | ~5 ms | ~31 ms | ~13 ms | ~25 ms |
| Engine | Ant Silver | V8 | JSC | V8 |

エンジンの Ant Silver は自作で、JIT には [MIR](https://github.com/themackabu/mir) の fork を使っています。API は [WinterTC Minimum Common API](https://min-common-api.proposal.wintertc.org/) を対象にしていて、`fetch` / `Request` / `Response` / `ReadableStream` などが最初からあります。手元で `ant -e` で確認したところ `WebSocket` や `Temporal` もグローバルに生えていました。

`Request` / `Response` / `fetch` あたりだけに依存するコードなら、そのまま動く見込みがあります。Hono はそのタイプです。

## Hono を動かす

インストールは 1 行です。`~/.ant/bin/ant` に 8.8MB のバイナリが 1 つ入ります。

```bash
curl -fsSL https://antjs.org/install | bash
```

パッケージマネージャも内蔵しています。`ant add hono` で `node_modules` に展開され、`ant.lockb` というバイナリのロックファイルができます。

```bash
ant add hono
```

アプリはこれだけです。TypeScript のまま実行できます。

```typescript:src/app.ts
import { Hono } from "hono";
import { logger } from "hono/logger";

export const app = new Hono();
app.use(logger());

app.get("/", (c) => c.text(`hello from Hono on Ant ${Ant.version}\n`));
app.get("/json", (c) =>
  c.json({ runtime: "ant", version: Ant.version, target: Ant.target }),
);
```

```typescript:src/server.ts
import { app } from "./app";

export default {
  port: 8080,
  fetch: (req: Request, ctx: unknown) => app.fetch(req, ctx),
};
```

`export default { port, fetch }` の形でオブジェクトを返すと、Ant がそれをサーバ定義として起動します。[server.c](https://github.com/theMackabu/ant/blob/ff84a70d887e507066bd6f4e0f03b626e4ddb2fc/src/modules/server.c#L1598-L1605) で default export に呼び出し可能な `fetch` があるかを見て、あれば listen する作りです。Bun の `export default { port, fetch }` と同じ形です。

![ant --version のあと ant src/server.ts で起動し、curl で / と /json を叩き、WebSocket の echo も返る](/images/antjs-hono-sandbox/01-run.gif)

```
$ ant src/server.ts &
$ curl -s -w '\n' localhost:8080/
hello from Hono on Ant 14.0.ff84a70d.0
$ curl -s -w '\n' localhost:8080/json
{"runtime":"ant","version":"14.0.ff84a70d.0","target":"arm64-apple-darwin24.6.0"}
```

`Ant` というグローバルオブジェクトがあり、`Ant.version` や `Ant.target` が取れます。

### WebSocket は `@ant/hono` を使う

HTTP だけなら `hono` 本体で足りますが、WebSocket は Ant のネイティブ実装につなぐ必要があるので、[`@ant/hono`](https://github.com/theMackabu/ant/tree/ff84a70d887e507066bd6f4e0f03b626e4ddb2fc/packages/hono) が用意されています。使うのは `upgradeWebSocket` です（ほかに `serve` / `createWSContext` / `WSContext` も export されています）。Ant 独自のレジストリ `npm.ants.land` から配布されているので、`.npmrc` にスコープを書いておきます。

```
@ant:registry=https://npm.ants.land
```

```typescript
import { upgradeWebSocket } from "@ant/hono";

app.get(
  "/ws",
  upgradeWebSocket(() => ({
    onMessage(event, ws) {
      ws.send(`echo: ${event.data}`);
    },
  })),
);
```

`server.ts` で `fetch` の第 2 引数まで `app.fetch` に渡しているのは、Ant がそこにサーバコンテキストを載せていて、`upgradeWebSocket` がそれを `c.env` 経由で見るためです。

## コールドスタートを測る

README と同じ形のスクリプト（Hono を import してルートを 2 つ登録し、`process.exit(0)` する）を hyperfine で測りました。README の計測とはハードウェアもバージョンも違うので絶対値は違いますが、傾向は同じでした。

```bash
hyperfine -N --warmup 10 --runs 100 \
  'ant bench-coldstart.js' 'bun bench-coldstart.js' 'node bench-coldstart.js'
```

| ランタイム | Mean ± σ | 相対 |
| --- | --- | --- |
| Ant | 15.2 ± 2.0 ms | 1.0 |
| Bun | 20.5 ± 2.4 ms | 1.3× |
| Node | 61.0 ± 5.4 ms | 4.0× |

Apple M4 Pro / Node 24.14.1 / Bun 1.3.11 での値です。同じ `bench-coldstart.js` を次の `ant compile` で実行ファイルにしたものは 12.0 ± 1.7 ms でした。モジュール解決が要らないぶん速くなっています。

## 単一バイナリにする

`ant compile` で、スクリプトと import 先を 1 つの実行ファイルにまとめられます。

```bash
ant compile -o dist/server src/server.ts
```

![ant compile で 7.5MB の dist/server ができ、node_modules の無い別ディレクトリで起動して curl が返る](/images/antjs-hono-sandbox/02-compile.gif)

```
$ ant compile -o dist/server src/server.ts
Compiled dist/server
33 modules, 80 KB embedded, 7652 KB total
$ ls -lh dist/server
-rwxr-xr-x@ 1 teppei0717  staff   7.5M  8月 18 11:08 dist/server
$ cp dist/server /tmp/standalone/ && cd /tmp/standalone && ./server &
$ curl -s -w '\n' localhost:8080/json
{"runtime":"ant","version":"14.0.ff84a70d.0","target":"arm64-apple-darwin24.6.0"}
$ bun ws-client.ts
received: echo: hello ant
```

Hono とそのミドルウェアを含む 33 モジュールが 80 KB として埋め込まれ、ランタイム込みで 7.5MB です。`node_modules` の無いディレクトリに `server` だけコピーして起動しても、HTTP も WebSocket も動きました。`otool -L` で見える依存は macOS のシステムライブラリだけです。

`ant compile` は [v14.0](https://github.com/theMackabu/ant/releases/tag/v14.0.ff84a70d.0) で追加されたコマンドです。スライドの時点では「シングルバイナリにする単一のコマンドは現状ない」という説明でしたが、その後のリリースで入りました。

## Sandbox は VM で動く

`ant:sandbox` の `Sandbox` は Worker のような隔離ではなく、Hypervisor.framework でゲストカーネルを起動する仮想マシンでした。`verbose: true` で起動ログを出すと、こう出ます（一部省略）。

```
ant: Hypervisor.framework backend image=~/.ant/sandbox/…/ant-sandbox-aarch64.img kernel=~/.ant/sandbox/…/ant-kernel-aarch64.img memory=256 MiB mounts=1 forwards=0
ant: mount[0] host=~/src/github.com/te2wow/ant-hono-demo guest=/workspace tag=0:/workspace:ro ro
…
ant: opened disk image (11931648 bytes)
ant: created VM
ant: created GIC
ant: mapped guest RAM base=0x40000000 size=256 MiB
ant: loaded sandbox kernel (1528424 bytes)
…
ant: created vCPU
ant: running guest request-timeout=10000 ms
en1: assigned 10.0.2.15
…
result: 3
```

12MB のディスクイメージと 1.5MB のゲストカーネルを読み込んで VM と vCPU を作り、ホストのディレクトリを 9P で read-only マウントして、ゲスト側にネットワークまで割り当てています。それで `1 + 2` を評価して返ってくるまで、手元では 20ms 前後でした。バックエンドの実装は [src/sandbox/backends/darwin/backend.c](https://github.com/theMackabu/ant/blob/ff84a70d887e507066bd6f4e0f03b626e4ddb2fc/src/sandbox/backends/darwin/backend.c#L57-L71) で、`hv_vm_create` を直接呼んでいます。

この速さなら「リクエストごとに VM を起こす」ことができそうです。POST されたコードを隔離実行して返す API を Hono で作りました。

```typescript:src/index.ts
import { Sandbox } from "ant:sandbox";
import { app } from "./app";

app.post("/run", async (c) => {
  const { code } = await c.req.json<{ code: string }>();
  const sandbox = new Sandbox({ mount: ".:/workspace", cpuTimeMs: 200 });
  try {
    const result = await sandbox.eval(
      `export default await (async () => { ${code} })()`,
    );
    return c.json({ ok: true, result, stats: sandbox.stats() });
  } catch (e) {
    const err = e instanceof Error ? e : new Error(String(e));
    return c.json({ ok: false, error: err.name, message: err.message }, 400);
  } finally {
    await sandbox.terminate().catch(() => {});
  }
});

export default {
  port: 8080,
  fetch: (req: Request, ctx: unknown) => app.fetch(req, ctx),
};
```

![POST /run に普通のコード・無限ループ・ホストのファイル読み取りを順に投げ、それぞれ結果・SandboxCpuTimeLimit・no such file が返る](/images/antjs-hono-sandbox/03-sandbox.gif)

```
$ curl -s -w '\n' -X POST localhost:8080/run -d '{"code":"return [1,2,3].map(n => n * 2)"}'
{"ok":true,"result":"[ 2, 4, 6 ]","stats":{"cpuTimeMs":26.284,"wallTimeMs":35.085,"residentMemory":28364800}}
$ curl -s -w '\n' -X POST localhost:8080/run -d '{"code":"while (true) {}"}'
{"ok":false,"error":"SandboxCpuTimeLimit","message":"sandbox VM exceeded its CPU time budget"}
$ curl -s -w '\n' -X POST localhost:8080/run -d '{"code":"const fs = await import(\"ant:fs\"); return fs.readFile(\"/etc/hosts\").catch(e => e.message)"}'
{"ok":true,"result":"no such file or directory","stats":{"cpuTimeMs":16.927,"wallTimeMs":27.071,"residentMemory":32436224}}
```

普通のコードは VM 起動込みで 35ms、無限ループは `cpuTimeMs: 200` で `SandboxCpuTimeLimit` になって止まります。ホストの `/etc/hosts` は「拒否された」ではなく「存在しない」になります。ゲストは自前の rootfs（`/etc` には `passwd` と `resolv.conf` と `ssl` だけ）を持っていて、ホスト側で見えるのは `mount` で渡したディレクトリだけだからです。

今回は逐次リクエストしか試していません。同時リクエストの数だけ VM が立つので、実際に使うならメモリの見積もりは別途必要です。オプションの一覧は [sandbox.d.ts](https://github.com/theMackabu/ant/blob/ff84a70d887e507066bd6f4e0f03b626e4ddb2fc/src/types/modules/sandbox.d.ts#L2-L40) にあります。

## 触って分かった注意点

v14.0 時点の挙動です。

- `Sandbox` の `memory` は 64mb 未満だと `RangeError` になりますが、この環境では 64mb や 80mb を指定するとゲストが `SandboxTimeout` で起動せず、安定して起動したのは 112mb 以上でした。既定値の 256MiB に任せています。
- `Sandbox#eval` に Promise を返すと await されず、`Promise { … }` の inspect 文字列が返ります。`export default await …` の形にします。
- CPU 制限に達した `Sandbox` は `close()` も同じエラーで reject します。後始末は `terminate()` にしました。
- `ant compile` した実行ファイルの中では `ant:sandbox` を解決できませんでした（`Cannot resolve module: ant:sandbox`）。Sandbox を使うエントリは `ant` コマンドで動かしています。
- `export default { fetch: app.fetch }` と書くと、`.ts` ファイルに限って `app` が未定義になりました。`.js` だと通ります。原因は追っていません。関数式で包めば動くので、そうしています。
- `ant compile -o dist/server` は出力先ディレクトリを作りません。

## まとめ

Hono は素の `hono` パッケージがそのまま動きました。単一バイナリはコマンド 1 つで 7.5MB になり、コールドスタートは手元でも Node の 4〜5 倍速く出ました。

いちばん面白かったのは Sandbox で、Hypervisor.framework 上の VM を数十 ms で起動できるので、ユーザーコードの隔離実行をリクエスト単位でやれる目処が立ちます。ここは他のランタイムには無い部分だと思います。

## 参考

- [Ant, a lightweight JavaScript runtime](https://antjs.org/)
- https://github.com/theMackabu/ant
- [小粒でもパワフルな JS runtime AntJS について](https://speakerdeck.com/comamoca/kotsubu-demo-pawafuruna-js-runtime-antjs-nitsuite)
- [デモアプリ: te2wow/ant-hono-demo](https://github.com/te2wow/ant-hono-demo)
