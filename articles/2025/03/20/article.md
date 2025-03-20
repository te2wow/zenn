---
title: "obs-websokcet-jsを使ってOBSと疎通をとる" # 記事のタイトルを設定してください
emoji: "🎬" # 記事のアイコンとして使用する絵文字を選択してください
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["obs", "websocket", "javascript", "streaming"] # タグを追加してください（例：["zenn", "markdown", "programming"]）
published: false # true: 公開記事 / false: 下書き
---

## はじめに

配信やスクリーンキャプチャなどで広く使われているOBS Studio（Open Broadcaster Software）は、多くのクリエイターにとって必須のツールとなっています。OBSを外部から制御できれば、配信の自動化やカスタムコントロールの作成など、様々な可能性が広がります。

本記事では、OBSとWebSocketを使った連携方法、特に「obs-websocket-js」ライブラリを使用してJavaScriptからOBSを制御する方法について解説します。

## 本文

### OBSとは

OBS Studio（Open Broadcaster Software）は、無料でオープンソースの配信・録画ソフトウェアです。ライブストリーミングやビデオ録画に必要な機能を備えており、YouTubeやTwitchなどの配信プラットフォームで広く使用されています。

OBSの主な特徴：
- 複数のシーンとソースの管理
- リアルタイムでの映像・音声のミキシング
- 様々な入力ソース（カメラ、マイク、画面キャプチャなど）のサポート
- 拡張性の高いプラグインシステム

### WebSocketとは

WebSocketは、クライアントとサーバー間で双方向通信を可能にするプロトコルです。HTTP接続と異なり、一度接続が確立されると、クライアントとサーバーの両方がいつでもデータを送信できる状態が維持されます。

WebSocketの主な利点：
- リアルタイム通信が可能
- 低レイテンシー
- 効率的なデータ転送
- ブラウザやサーバーで広くサポートされている

### OBS WebSocketとは

OBS WebSocketは、OBS Studioに対してWebSocket経由で制御を行うためのプラグインです。OBS 28.0以降では標準で組み込まれています。このプラグインを使用することで、外部アプリケーションからOBSの操作が可能になります。

### obs-websocket-jsの基本

obs-websocket-jsは、JavaScriptからOBS WebSocketを簡単に利用するためのライブラリです。Node.jsアプリケーションやブラウザから使用できます。

#### インストール方法
obs-websocket-jsを使用するには、まずnpmまたはyarnを使ってパッケージをインストールします。

```sh
npm install obs-websocket-js
# または
yarn add obs-websocket-js
```

## 基本的な接続方法

以下のコードは、obs-websocket-jsを使ってOBSに接続する基本的な例です。

```javascript
import OBSWebSocket from 'obs-websocket-js';

const obs = new OBSWebSocket();

async function connectOBS() {
  try {
    await obs.connect('ws://localhost:4455', 'パスワード');
    console.log('OBSに接続しました');
  } catch (error) {
    console.error('OBSへの接続に失敗しました:', error);
  }
}

connectOBS();
```

OBS WebSocketはデフォルトで`ws://localhost:4455`でリスニングしています。パスワードは、OBSのWebSocket設定で設定したものを使用します。

## 主なメソッドとサンプルコード

### シーンを変更する

```javascript
async function changeScene(sceneName) {
  try {
    await obs.call('SetCurrentProgramScene', { sceneName });
    console.log(`シーンを「${sceneName}」に変更しました`);
  } catch (error) {
    console.error('シーンの変更に失敗しました:', error);
  }
}

changeScene('ゲーム画面');
```

### 配信の開始・停止

```javascript
async function startStreaming() {
  try {
    await obs.call('StartStream');
    console.log('配信を開始しました');
  } catch (error) {
    console.error('配信開始に失敗しました:', error);
  }
}

async function stopStreaming() {
  try {
    await obs.call('StopStream');
    console.log('配信を停止しました');
  } catch (error) {
    console.error('配信停止に失敗しました:', error);
  }
}

startStreaming();
// stopStreaming(); を適宜呼び出す
```

### ソースの可視化/非表示

```javascript
async function setSourceVisibility(sourceName, visible) {
  try {
    await obs.call('SetSourceFilterEnabled', {
      sourceName,
      filterName: 'フィルタ名',
      filterEnabled: visible,
    });
    console.log(`ソース「${sourceName}」の表示状態を${visible ? 'オン' : 'オフ'}にしました`);
  } catch (error) {
    console.error('ソースの可視化/非表示の切り替えに失敗しました:', error);
  }
}

setSourceVisibility('Webcam', true);
```

### 利用可能なリクエストの確認方法

obs-websocket-jsで使用できるリクエストの完全なリストは、GitHubリポジトリの`src/types.ts`ファイルで確認できます。

具体的には以下のURLで確認できます：
https://github.com/obs-websocket-community-projects/obs-websocket-js/blob/master/src/types.ts

このファイルには`OBSRequestTypes`というタイプ定義があり、すべての利用可能なリクエストメソッドとそのパラメータが詳細に記載されています。これを参照することで、以下のような情報を確認できます：

- 各リクエストメソッドの名前
- 必要なパラメータとその型
- レスポンスの型と構造

例えば、先ほど使用した`SetCurrentProgramScene`や`StartStream`などのメソッドもこのファイルで定義されています。新しい機能を実装する際は、このファイルを参照することで利用可能なAPIを確認できます。

## obs-websocket-jsを使わない開発との比較

obs-websocket-jsを使用しない場合、WebSocket APIを直接扱う必要があります。

### 直接WebSocketを使用する場合のコード例

```javascript
const ws = new WebSocket('ws://localhost:4455');

ws.onopen = () => {
  console.log('WebSocket接続が確立しました');
  ws.send(JSON.stringify({
    'op': 1,
    'd': { 'rpcVersion': 1 }
  }));
};

ws.onmessage = (event) => {
  console.log('受信:', event.data);
};

ws.onerror = (error) => {
  console.error('WebSocketエラー:', error);
};
```

WebSocketを直接使う場合は、コマンドを適切なJSON形式で送信し、レスポンスを処理する必要があります。そのため、obs-websocket-jsを使用する方が開発が簡単になります。

## まとめ

obs-websocket-jsを使うことで、JavaScriptから簡単にOBSを操作できます。本記事では基本的な接続方法、シーンの変更、配信の開始・停止、ソースの可視化・非表示の方法を紹介しました。

WebSocketを直接使う方法もありますが、obs-websocket-jsを利用することでより簡潔なコードで開発できるため、特にJavaScriptでの開発におすすめです。

余談ですが、筆者はobs-websocket-jsのドキュメントのタイポを修正してコントリビュートしており、ライブラリのコントリビューターとして名を連ねています（笑）。オープンソースへの貢献は、タイポの修正のような小さなことからでも始められるということを付け加えておきたいと思います。
```
