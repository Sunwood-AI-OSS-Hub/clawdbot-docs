# Clawdbot アーキテクチャ

……ふふ、Clawdbotの全体像をまとめてみたよ。

## 概要

Clawdbotは、複数のメッセージングプラットフォーム（Discord、Slack、Telegram、WhatsAppなど）とAIエージェントを接続するためのフレームワークです。

## 全体アーキテクチャ

![全体アーキテクチャ](diagrams/clawdbot-overall-architecture.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
graph TB
    subgraph "Discord Platform"
        D1[Discord Server]
        D2[Discord Channel]
        D3[User Message]
    end

    subgraph "Clawdbot - Discord Layer"
        DC1[discord/monitor/provider.ts<br/>Discord Provider]
        DC2[DiscordListener<br/>Gateway Events]
        DC3[DiscordMessageHandler]
        DC4[message-handler.preflight.ts<br/>Validation & Auth]
        DC5[message-handler.process.ts<br/>Main Processing]
    end

    subgraph "Clawdbot - Core"
        R1[routing/resolve-route.ts<br/>Route Resolution]
        S1[channels/session.ts<br/>Session Management]
        AR1[auto-reply/dispatch.ts<br/>Reply Dispatcher]
        TI[Typing Indicator]
    end

    subgraph "Clawdbot - Agent"
        PI1[agents/pi-embedded-runner.ts<br/>Agent Execution]
        M1[Model Provider<br/>OpenAI/Anthropic/etc]
        MEM[(Session Memory<br/>JSONL)]
    end

    subgraph "Clawdbot - Media"
        MP1[media/processor.ts<br/>Media Processing]
        TMP[(Temp Files)]
    end

    subgraph "Clawdbot - Gateway"
        GW1[gateway/server.impl.ts<br/>Gateway Server]
        GW2[gateway/server/ws-connection.ts<br/>WebSocket]
    end

    D3 -->|"Gateway Event"| DC2
    DC2 --> DC3
    DC3 --> DC4
    DC4 -->|"Validated"| DC5
    DC5 -->|"Extract Text/Media"| R1
    DC5 -->|"Attachments"| MP1
    MP1 --> TMP
    R1 --> S1
    S1 --> AR1
    AR1 --> PI1
    PI1 --> M1
    PI1 <-->|"Read/Write"| MEM
    AR1 --> TI
    TI -->|"Reply"| DC1
    DC1 --> D2

    GW1 <-->|"WebSocket"| S1
    GW1 <-->|"WebSocket"| PI1

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef primary fill:#2B4B7E,stroke:#D4AF37,stroke-width:2px,color:#F7F3EA

    class PI1,M1,AR1,DC5 highlight
    class D3,D2 primary
```

## メッセージフロー

![メッセージフロー](diagrams/clawdbot-message-flow.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP', 'actorLineColor':'#2B4B7E', 'actorBkgColor':'#F7F3EA', 'actorTextColor':'#2B4B7E', 'actorBorderColor':'#2B4B7E', 'actorStereotypeColor':'#D4AF37', 'noteBkgColor':'#D4AF37', 'noteTextColor':'#2B4B7E', 'noteBorderColor':'#2B4B7E', 'signalColor':'#2B4B7E', 'signalTextColor':'#2B4B7E' } } }%%
sequenceDiagram
    participant U as Discord User
    participant DG as Discord Gateway
    participant DP as Discord Provider
    participant MH as Message Handler
    participant RT as Routing
    participant SS as Session Manager
    participant AR as Auto-Reply
    participant AG as Agent
    participant M as Model Provider

    U->>DG: Send Message
    DG->>DP: Gateway Event
    DP->>MH: DiscordMessageListener

    MH->>MH: Preflight Check
    Note over MH: • Validation<br/>• Auth Check<br/>• Debounce

    alt Valid Message
        MH->>MH: Parse Message
        Note over MH: • Extract Text<br/>• Process Media<br/>• Resolve Thread

        MH->>RT: Resolve Route
        RT->>SS: Get/Create Session
        SS->>SS: Load Session History

        RT->>AR: Dispatch Message
        AR->>AR: Start Typing Indicator

        AR->>AG: Send to Agent
        AG->>M: API Request
        M-->>AG: Response

        AG-->>AR: Agent Response
        AR->>AR: Format Reply
        AR->>DP: Deliver Reply
        DP->>DG: Send to Discord
        DG->>U: Bot Reply

        AR->>SS: Update Session History
    else Invalid/Debounced
        MH-->>DG: Ignore
    end
```

## チャネル共通レイヤー

![チャネル共通レイヤー](diagrams/clawdbot-common-layer.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
graph LR
    subgraph "Channel Providers"
        D[Discord Provider]
        S[Slack Provider]
        T[Telegram Provider]
        W[Web Provider<br/>WhatsApp]
    end

    subgraph "Common Layer"
        CH[channels/ - Shared Logic]
        AL[AllowList]
        RP[Reply Prefix]
        TY[Typing Indicator]
        AC[Ack Reaction]
    end

    subgraph "Core Processing"
        RT[Routing]
        SS[Session Management]
        AR[Auto-Reply Dispatcher]
    end

    subgraph "Agent Layer"
        AG[Agent Runner]
        PRV[Provider Selector]
        CFG[Agent Config]
    end

    D --> CH
    S --> CH
    T --> CH
    W --> CH

    CH --> AL
    CH --> RP
    CH --> TY
    CH --> AC

    AL --> RT
    RP --> RT
    TY --> AR
    AC --> AR

    RT --> SS
    SS --> AR
    AR --> AG

    AG --> PRV
    AG --> CFG

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef primary fill:#2B4B7E,stroke:#D4AF37,stroke-width:2px,color:#F7F3EA

    class AR,AG,CH highlight
    class D,S,T,W primary
```

## ディレクトリ構造

```
clawdbot/src/
├── discord/          # Discord連携のコア
├── channels/         # チャネル共通処理
├── gateway/         # ゲートウェイサーバー
├── routing/          # ルーティング
├── agents/           # エージェント処理
├── web/             # Webプロバイダー
└── auto-reply/      # 自動返信機能
```

## 主要コンポーネント

### Discord Layer

| ファイル | 主要な関数/エクスポート | 役割 |
|---------|---------------------|------|
| `discord/monitor/provider.ts` | `monitorDiscordProvider` | Discordプロバイダのメインエントリーポイント。Discordクライアントの初期化、コマンドデプロイ、メッセージハンドラー登録 |
| `discord/monitor/message-handler.process.ts` | `processDiscordMessage` | メッセージ処理の本体。テキスト/メディアの抽出、ルーティング、セッション管理、AIへのディスパッチ |
| `discord/monitor/message-handler.preflight.ts` | `preflightDiscordMessage` | メッセージの事前チェック（認証・検証）。Botメッセージ除外、AllowListチェック、メンション検証 |

### Core Processing

| ファイル | 主要な関数/エクスポート | 役割 |
|---------|---------------------|------|
| `routing/resolve-route.ts` | `resolveAgentRoute`, `buildAgentSessionKey` | 送信元から宛先のルーティング決定。セッションキー生成、バインディング解決 |
| `channels/session.ts` | `recordInboundSession` | セッション管理と履歴保存。セッションメタデータの記録、ルート更新 |
| `auto-reply/dispatch.ts` | `dispatchInboundMessage`, `createReplyDispatcherWithTyping` | メッセージのディスパッチと返信制御。Typingインジケーター付きディスパッチャー作成 |

### Agent Layer

| ファイル | 主要な関数/エクスポート | 役割 |
|---------|---------------------|------|
| `agents/pi-embedded-runner.ts` | `runEmbeddedPiAgent` | エージェント実行エンジン。Claude Embedded Piとのやり取り、セッション管理 |
| `agents/pi-embedded-runner/run.ts` | `runEmbeddedPiAgent` (実装) | エージェント実行の実装詳細 |
| `agents/auth-profiles.ts` | 認証プロファイル管理 | AIプロバイダーの認証情報管理 |

### Gateway Layer

| ファイル | 主要な関数/エクスポート | 役割 |
|---------|---------------------|------|
| `gateway/server.impl.ts` | `startGateway` | Gatewayサーバーのメイン実装。WebSocketサーバー起動、チャネル管理 |
| `gateway/server/ws-connection.ts` | WebSocket接続管理 | WebSocket接続の確立、メッセージハンドリング |

### Common Layer

| 機能 | 役割 |
|------|------|
| AllowList | アクセス制御（誰がBotを使えるか） |
| Reply Prefix | 返信プレフィックス設定 |
| Typing Indicator | 「入力中...」インジケーター |
| Ack Reaction | 処理開始のリアクション（👍） |

## メッセージ処理の詳細

### 1. 事前チェック（Preflight）

```
discord/monitor/message-handler.preflight.ts
```

- メッセージ検証（空メッセージの除外）
- メンションチェック
- 認証・権限確認
- デバウンス処理（重複メッセージの抑制）

### 2. メッセージ解析

- テキスト抽出（リプライ含む）
- メディアアタッチメント処理
- スレッド情報解決

### 3. ルーティング決定

```typescript
// 送信元情報から宛先を決定
const effectiveFrom = isDirectMessage
  ? `discord:${author.id}`
  : `discord:channel:${message.channelId}`;
```

### 4. セッション管理

- セッションキー生成
- 履歴保存（JSONL形式）
- 既存セッションの継続/新規作成判断

### 5. エージェントへのディスパッチ

```typescript
const { dispatcher, replyOptions, markDispatchIdle } = createReplyDispatcherWithTyping({
  deliver: async (payload: ReplyPayload) => {
    // Discordへの返信処理
    await deliverDiscordReply({ /* ... */ });
  }
});
```

## 特徴的な仕組み

### デバウンス機構

- 短時間内の重複メッセージを処理を抑制
- 非同期キューでバッチ処理

### スレッド対応

- Discordスレッドの子メッセージを親セッションに関連付け
- 自動スレッド作成機能

### リアクションフィードバック

- 処理開始時のリアクション（👍）
- 処理完了後のリアクション削除

### メンション検知

- Botへのメンションの有効チェック
- 通常メッセージとコマンドメッセージの区別

## レイヤー別の役割

| レイヤー | 役割 |
|---------|------|
| **Discord Layer** | Discord Gatewayのイベント監視、メッセージ受信 |
| **Common Layer** | 全チャネル共通処理（AllowList、Typing等） |
| **Core Processing** | ルーティング、セッション管理、ディスパッチ |
| **Agent Layer** | エージェント実行、モデル呼び出し |

## 関連リンク

- [Clawdbot Repository](https://github.com/clawdbot/clawdbot)
- [Clawdbot Docs](https://docs.clawd.bot/)

---

……ふふ、これでClawdbotの全体が見えたかな。
