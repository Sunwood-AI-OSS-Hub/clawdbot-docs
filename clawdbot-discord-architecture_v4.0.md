# Clawdbot の中身を覗いてみよう 🦞 〜Discord で話しかけたら何が起きる？〜

> ⚠️ **この記事を読む前に**
> この記事は 2026年1月時点の情報をもとにしています。Clawdbot は活発に開発が進んでいるプロジェクトのため、最新情報は [公式ドキュメント](https://docs.clawd.bot/concepts/architecture) をご確認ください。

## ざっくり言うと

- Discord でメッセージを送ると、 **4つの層** （Discord層 → 共通層 → コア処理層 → AIエージェント層）を順番に通って返信が届く
- 「誰が話しかけてきたか」「会話の続きか」を **セッション管理** で記憶している
- 主要な処理は `src/discord/monitor/` 配下の TypeScript ファイルで行われている

## 📖 この記事について

この記事は、[前回の記事「Clawdbot を Fly.io にデプロイしよう！」](https://note.com/sunwood_ai_labs/n/n211d0c2cca18) と [「Agent Workspace とは何か」](https://note.com/sunwood_ai_labs/n/nb9ea5760f15d) の続編です。

今回は **Clawdbot の内部アーキテクチャ** を、Discord を例に詳しく解説していきます。

「Discord で話しかけたら、中で何が起きてるの？」という疑問に答えます。

---

## そもそも、どんな構造になってるの？

Clawdbot の全体像を図で見てみましょう。

### 全体概要

**Clawdbot = 複数のチャットサービスとAIを繋ぐ架け橋** です。

![全体像](diagrams/clawdbot-simple-overview.png)

Discord、Slack、Telegram、WhatsApp など、いろんなチャットサービスからのメッセージを受け取って、Claude や GPT などの AI に渡してくれます。

### たとえ話で説明すると...

会社の **コールセンター** をイメージしてください。

1. **受付（Discord Layer）**：電話を取り次ぐ人。「お名前は？」「ご用件は？」
2. **セキュリティ（Common Layer）**：「この人、お客様リストに載ってる？」
3. **オペレーター管理（Core Processing）**：「この人は前にも問い合わせてきた人だ。前回の会話記録を出そう」
4. **専門スタッフ（Agent Layer）**：実際に質問に答える AI

この 4 層が連携して、あなたの Discord メッセージに返信しているわけです。

---

## 🔄 メッセージがどう動いてるか

まずはシンプルな流れから見てみましょう。

![メッセージフロー](diagrams/clawdbot-simple-message-flow.png)

1. あなたが Discord で話しかける
2. Clawdbot が受け取る
3. セッションを管理（履歴を覚える）
4. AI に渡す
5. AI が考えた答えを返す
6. Discord であなたに届く

---

## 🏗️ 中身は4つに分かれてる

Clawdbot の内部は **4つの層** に分かれています。

![レイヤー構成](diagrams/clawdbot-simple-layers.png)

| レイヤー | 役割 |
|---------|------|
| **① チャネル層** | Discord、Slack、Telegram などからメッセージを受け取る |
| **② 共通処理層** | AllowList（誰が使えるか）、Reply Prefix（返信のプレフィックス）、Typing（入力中インジケーター） |
| **③ コア処理層** | ルーティング（どこから来たか）、セッション管理（履歴を覚える）、ディスパッチャー（AIに渡す） |
| **④ AIエージェント層** | AIエージェント（Claude / GPT）、プロバイダー選択、設定管理 |

---

## 🎯 Discord で話しかけるとどうなる？

具体的な流れを見てみましょう。

![Discordフロー](diagrams/clawdbot-simple-discord-flow.png)

**① Discord がイベントを通知** → Clawdbot が受け取る

**② チェックする**
- 空メッセージじゃない？
- Bot へのメンションが含まれてる？
- 使うことが許可されてる人？

**③ セッションを管理** → 以前の会話履歴も思い出す（この機能で会話が続く！）

**④ AI に投げる** → Claude や GPT に質問を送る

**⑤ AI の答えを受け取る** → 整形して Discord に送り返す

---

## 📊 全体アーキテクチャ（詳細版）

すべてを統合した図がこちらです。

![全体アーキテクチャ](diagrams/clawdbot-overall-architecture.png)

この図のポイント：

- **黄色でハイライト** されているのが主要な処理コンポーネント
- **青色** がユーザーとのインターフェース（User Message、Discord Channel）
- **Session Memory（JSONL）** で会話履歴を永続化
- **Gateway** 経由で WebSocket 通信も可能

---

## 📂 メッセージフロー（シーケンス図）

時系列でより詳しく見てみましょう。

![メッセージフロー](diagrams/clawdbot-message-flow.png)

1. **Discord User** がメッセージを送信
2. **Discord Gateway** → **Discord Provider** → **Message Handler** と伝達
3. **Preflight Check** で Validation / Auth Check / Debounce
4. **Valid なら** Parse → Routing → Session → Auto-Reply → Agent → Model Provider
5. **Response** が戻ってきたら Discord に送り返す
6. **Invalid/Debounced なら** 無視

---

## 🔗 チャネル共通レイヤー

Discord だけでなく、Slack や Telegram でも使う **共通機能** がここにあります。

![チャネル共通レイヤー](diagrams/clawdbot-common-layer.png)

| 機能 | 役割 |
|------|------|
| **AllowList** | アクセス制御（誰が Bot を使えるか） |
| **Reply Prefix** | 返信プレフィックス設定 |
| **Typing Indicator** | 「入力中...」インジケーター |
| **Ack Reaction** | 処理開始のリアクション（👍） |

### なぜ共通化するの？

こうすることで、新しいチャットサービスに対応するとき、 **共通処理を書き直さなくて済む** んです。Discord Provider、Slack Provider、Telegram Provider、Web Provider（WhatsApp）すべてが同じ Common Layer を通ります。

---

## 🔍 主要コンポーネントの詳細

### Discord Layer

Discord からメッセージが来たとき、最初に動くのが **Discord Layer** です。

[`discord/monitor/provider.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/discord/monitor/provider.ts) が `monitorDiscordProvider` 関数をエクスポートしています。これが Discord プロバイダのメインエントリーポイントで、Discord クライアントの初期化、コマンドデプロイ、メッセージハンドラー登録を行います。

[`discord/monitor/message-handler.preflight.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/discord/monitor/message-handler.preflight.ts) の `preflightDiscordMessage` 関数がメッセージの事前チェック（認証・検証）を担当します。Bot メッセージ除外、AllowList チェック、メンション検証などを行います。

[`discord/monitor/message-handler.process.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/discord/monitor/message-handler.process.ts) の `processDiscordMessage` 関数がメッセージ処理の本体です。テキスト/メディアの抽出、ルーティング、セッション管理、AI へのディスパッチを行います。

### Core Processing

[`routing/resolve-route.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/routing/resolve-route.ts) では `resolveAgentRoute` と `buildAgentSessionKey` 関数が定義されています。送信元から宛先のルーティング決定、セッションキー生成、バインディング解決を行います。

[`channels/session.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/channels/session.ts) の `recordInboundSession` 関数がセッション管理と履歴保存を担当します。セッションメタデータの記録、ルート更新を行います。

[`auto-reply/dispatch.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/auto-reply/dispatch.ts) では `dispatchInboundMessage` と `createReplyDispatcherWithTyping` 関数が定義されています。メッセージのディスパッチと返信制御、Typing インジケーター付きディスパッチャー作成を行います。

### Agent Layer

[`agents/pi-embedded-runner.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/agents/pi-embedded-runner.ts) の `runEmbeddedPiAgent` 関数がエージェント実行エンジンです。Claude Embedded Pi とのやり取り、セッション管理を行います。実装の詳細は [`agents/pi-embedded-runner/run.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/agents/pi-embedded-runner/run.ts) にあります。

[`agents/auth-profiles.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/agents/auth-profiles.ts) が認証プロファイル管理を担当し、AI プロバイダーの認証情報管理を行います。

### Gateway Layer

[`gateway/server.impl.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/gateway/server.impl.ts) の `startGateway` 関数が Gateway サーバーのメイン実装です。WebSocket サーバー起動、チャネル管理を行います。

[`gateway/server/ws-connection.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/gateway/server/ws-connection.ts) が WebSocket 接続管理を担当し、WebSocket 接続の確立、メッセージハンドリングを行います。

---

## ⚙️ メッセージ処理の詳細

### 1. 事前チェック（Preflight）

[`discord/monitor/message-handler.preflight.ts`](https://github.com/clawdbot/clawdbot/blob/main/src/discord/monitor/message-handler.preflight.ts) で以下の処理を行います：

- メッセージ検証（空メッセージの除外）
- メンションチェック
- 認証・権限確認
- デバウンス処理（重複メッセージの抑制）

```typescript
// preflightDiscordMessage の主な処理（簡略化）
export async function preflightDiscordMessage(message: Message) {
  // 1. Bot のメッセージは無視
  if (message.author.bot) return null;
  
  // 2. 空メッセージは無視
  if (!message.content && message.attachments.size === 0) return null;
  
  // 3. AllowList チェック（許可されたユーザーか？）
  if (!isAllowed(message.author.id)) return null;
  
  // 4. メンションチェック（Bot 宛てか？）
  if (!isMentionedOrDM(message)) return null;
  
  return { valid: true, message };
}
```

> 💡 **ポイント**
> ここで弾かれたメッセージは、その後の処理に一切進みません。これにより **不要な API 呼び出し** を防いでいます。

### 2. メッセージ解析

- テキスト抽出（リプライ含む）
- メディアアタッチメント処理
- スレッド情報解決

```typescript
// processDiscordMessage の主な処理（簡略化）
export async function processDiscordMessage(message: Message) {
  // 1. テキストを抽出（リプライ元も含む）
  const text = extractText(message);
  
  // 2. 添付ファイルがあれば処理
  const media = await processAttachments(message.attachments);
  
  // 3. スレッド情報を解決
  const threadInfo = resolveThread(message);
  
  // 4. ルーティングへ渡す
  return { text, media, threadInfo };
}
```

### 3. ルーティング決定

送信元情報から宛先を決定します。

```typescript
// routing/resolve-route.ts より
// 送信元情報から宛先を決定
const effectiveFrom = isDirectMessage
  ? `discord:${author.id}`
  : `discord:channel:${message.channelId}`;
```

> 💡 **ポイント**
> DM とチャンネルで **セッションが分かれる** のはこの仕組みのおかげです。プライベートな会話と、サーバーでの会話が混ざりません。

### 4. セッション管理

- セッションキー生成
- 履歴保存（JSONL 形式）
- 既存セッションの継続/新規作成判断

```typescript
// channels/session.ts より（簡略化）
export async function recordInboundSession(params: {
  sessionKey: string;
  message: string;
  timestamp: Date;
}) {
  // 1. 既存セッションがあれば読み込み
  const existing = await loadSession(params.sessionKey);
  
  // 2. 履歴に追加
  existing.history.push({
    role: 'user',
    content: params.message,
    timestamp: params.timestamp
  });
  
  // 3. 保存（JSONL 形式）
  await saveSession(params.sessionKey, existing);
}
```

これが「前の会話を覚えている」機能の正体です。

### 5. エージェントへのディスパッチ

```typescript
// auto-reply/dispatch.ts より
const { dispatcher, replyOptions, markDispatchIdle } = createReplyDispatcherWithTyping({
  deliver: async (payload: ReplyPayload) => {
    // Discordへの返信処理
    await deliverDiscordReply({ /* ... */ });
  }
});
```

---

## 🛠️ ファイルの場所（ざっくり）

```
clawdbot/src/
├── discord/      # Discordとの連携部分
│   └── monitor/
│       ├── provider.ts              # Discordプロバイダのメイン
│       ├── message-handler.process.ts   # メッセージ処理
│       └── message-handler.preflight.ts # 事前チェック
├── slack/        # Slackとの連携部分
├── channels/     # どのチャネルでも使う共通処理
│   └── session.ts                   # セッション管理
├── routing/      # 「どのAIに渡すか」を判断するルーター
│   └── resolve-route.ts             # ルーティング決定
├── agents/       # AIエージェントの実行部分
│   ├── pi-embedded-runner.ts        # エージェント実行エンジン
│   └── auth-profiles.ts             # 認証プロファイル管理
├── auto-reply/   # 自動返信機能
│   └── dispatch.ts                  # ディスパッチャー
├── gateway/      # WebSocketサーバー（リアルタイム通信用）
│   ├── server.impl.ts               # Gatewayサーバーメイン
│   └── server/ws-connection.ts      # WebSocket接続
└── media/        # 画像やファイルの処理
```

---

## 💡 特徴的な仕組み

### デバウンス機構

短時間内の重複メッセージの処理を抑制します。非同期キューでバッチ処理することで、「あ」「あ、」「あ、そうだ」と連続で送っても、最後のメッセージだけが処理されます。API コストの節約になります。

### スレッド対応

- Discord スレッドの子メッセージを親セッションに関連付け
- 自動スレッド作成機能

### リアクションフィードバック

- 処理開始時のリアクション（👍）
- 処理完了後のリアクション削除

これにより、ユーザーは「Bot が処理を始めた」ことを視覚的に確認できます。

### メンション検知

- Bot へのメンションの有効チェック
- 通常メッセージとコマンドメッセージの区別

---

## 📋 主な機能まとめ

| 機能 | 説明 |
|------|------|
| **マルチチャネル対応** | Discord、Slack、Telegram、WhatsApp など |
| **セッション管理** | 会話の履歴を覚えてくれる |
| **AI プロバイダ切り替え** | Claude、GPT、ローカルモデルなど選べる |
| **メディア対応** | 画像やファイルも送れる |
| **AllowList** | 誰が Bot を使えるか制御できる |
| **スレッド対応** | Discord スレッドでも会話できる |

---

## ⚠️ 注意点・制限事項

### メンション必須（グループチャンネルの場合）

デフォルトでは、グループチャンネルでは **Bot へのメンション** が必要です。これは意図しない反応を防ぐためです。

```
❌ こんにちは          → 反応しない
✅ @Clawdbot こんにちは → 反応する
```

### セッションの分離

| 送信元 | セッションキー | 備考 |
|--------|---------------|------|
| DM | `agent:main:discord:{userId}` | ユーザーごとに独立 |
| チャンネル | `agent:main:discord:channel:{channelId}` | チャンネルごとに独立 |
| スレッド | 親チャンネルに紐付け | スレッド内は同一セッション |

---

## まとめ

![まとめ](diagrams/clawdbot-simple-summary.png)

つまり、Discord でメッセージを送ると、 **4 つの層** を順番に通っていました：

1. **Discord Layer**：メッセージを受け取り、処理していいか事前チェック
2. **Common Layer**：AllowList チェック、入力中表示など共通処理
3. **Core Processing**：セッションキー生成、会話履歴の読み書き
4. **Agent Layer**：AI に質問して返答をもらう

**Clawdbot は**「いろんなチャットサービス」と「AI」をつなぐための便利なツールです。会話の履歴を覚えてくれたり、画像を送ったりもできます。

この設計により、Discord だけでなく Slack や Telegram など **複数のチャットサービス** に対応しつつ、 **会話履歴を統一管理** できているんです。

次回は、この仕組みをカスタマイズする方法（スキルの追加など）を解説予定です 🦞

---

## もっと詳しく知りたい人へ

### 参考にした資料

1. [Clawdbot Architecture Overview](https://docs.clawd.bot/concepts/architecture) - 公式ドキュメント
2. [Clawdbot GitHub Repository](https://github.com/clawdbot/clawdbot) - ソースコード
3. [Discord Channel Setup](https://docs.clawd.bot/channels/discord) - Discord 設定ガイド

### 関連記事

- [Clawdbot を Fly.io にデプロイしよう！](https://note.com/sunwood_ai_labs/n/n211d0c2cca18)
- [Clawdbot（Clawd）の Agent Workspace とは何か](https://note.com/sunwood_ai_labs/n/nb9ea5760f15d)

### 関連リンク

- GitHub リポジトリ：https://github.com/clawdbot/clawdbot
- 公式ドキュメント：https://docs.clawd.bot/
- Discord コミュニティ：https://discord.gg/clawd

---

*この記事は 2026年1月27日 時点の情報です。*
