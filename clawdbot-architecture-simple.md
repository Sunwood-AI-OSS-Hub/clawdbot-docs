# Clawdbot って何？

……ふふ、まずは全体像から見ていこうか。

## ざっくり言うと

**Clawdbot = 複数のチャットサービスとAIを繋ぐ架け橋**

![全体像](diagrams/clawdbot-simple-overview.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
graph LR
    subgraph "チャットサービス"
        D[Discord]
        S[Slack]
        T[Telegram]
        W[WhatsApp]
    end

    subgraph "Clawdbot"
        CB[ルーター]
    end

    subgraph "AI"
        LLM[Claude / GPT / etc]
    end

    D --> CB
    S --> CB
    T --> CB
    W --> CB
    CB <--> LLM

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E

    class CB highlight
```

## メッセージがどう動いてるか

![メッセージフロー](diagrams/clawdbot-simple-message-flow.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
flowchart TD
    A[あなたがDiscordで話しかける] --> B[Clawdbotが受け取る]
    B --> C[セッションを管理<br/>履歴を覚える]
    C --> D[AIに渡す]
    D --> E[AIが考えた答えを返す]
    E --> F[Discordであなたに届く]

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E

    class D,F highlight
```

## 中身は4つに分かれてる

![レイヤー構成](diagrams/clawdbot-simple-layers.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
graph TB
    subgraph L1["① チャネル層"]
        D1[Discord]
        S1[Slack]
        T1[Telegram]
        W1[WhatsApp]
    end

    subgraph L2["② 共通処理層"]
        AL[AllowList<br/>誰が使えるか]
        RP[Reply Prefix<br/>返信のプレフィックス]
        TY[Typing<br/>入力中インジケーター]
    end

    subgraph L3["③ コア処理層"]
        RT[ルーティング<br/>どこから来たか]
        SS[セッション管理<br/>履歴を覚える]
        AR[ディスパッチャー<br/>AIに渡す]
    end

    subgraph L4["④ AIエージェント層"]
        AG[AIエージェント<br/>Claude / GPT]
        PR[プロバイダー選択]
        CFG[設定管理]
    end

    L1 --> L2
    L2 --> L3
    L3 --> L4

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E

    class AR,AG highlight
```

## Discordで話しかけるとどうなる？

![Discordフロー](diagrams/clawdbot-simple-discord-flow.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
flowchart TD
    Start["あなた: 「こんにちは」"]

    Step1["① Discordがイベントを通知<br/>Clawdbotが受け取る"]
    Step2["② チェックする<br/>• 空メッセージじゃない？<br/>• Botへのメンションが含まれてる？<br/>• 使うことが許可されてる人？"]
    Step3["③ セッションを管理<br/>以前の会話履歴も思い出す<br/>（この機能で会話が続く！）"]
    Step4["④ AIに投げる<br/>ClaudeやGPTに質問を送る"]
    Step5["⑤ AIの答えを受け取る<br/>整形してDiscordに送り返す"]

    End["Bot: 「こんにちは！何かお手伝いできる？」"]

    Start --> Step1
    Step1 --> Step2
    Step2 --> Step3
    Step3 --> Step4
    Step4 --> Step5
    Step5 --> End

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef user fill:#2B4B7E,stroke:#D4AF37,stroke-width:2px,color:#F7F3EA

    class Start,End user
    class Step4,Step5 highlight
```

## 主な機能

| 機能 | 説明 |
|------|------|
| **マルチャネル対応** | Discord、Slack、Telegram、WhatsAppなど |
| **セッション管理** | 会話の履歴を覚えてくれる |
| **AIプロバイダ切り替え** | Claude、GPT、ローカルモデルなど選べる |
| **メディア対応** | 画像やファイルも送れる |
| **AllowList** | 誰がBotを使えるか制御できる |
| **スレッド対応** | Discordスレッドでも会話できる |

## ファイルの場所（ざっくり）

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

## まとめ

![まとめ](diagrams/clawdbot-simple-summary.png)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#F7F3EA', 'primaryTextColor':'#2B4B7E', 'primaryBorderColor':'#2B4B7E', 'lineColor':'#2B4B7E', 'secondaryColor':'#D4AF37', 'tertiaryColor':'#F7F3EA', 'background':'#F7F3EA', 'fontFamily':'Noto Sans JP' }} }%%
graph LR
    Chat[いろんな<br/>チャットサービス]
    Bot[Clawdbot]
    AI[AI]

    Chat --> Bot
    Bot <--> AI

    classDef default fill:#F7F3EA,stroke:#2B4B7E,stroke-width:2px,color:#2B4B7E
    classDef highlight fill:#D4AF37,stroke:#2B4B7E,stroke-width:3px,color:#2B4B7E
    classDef primary fill:#2B4B7E,stroke:#D4AF37,stroke-width:2px,color:#F7F3EA

    class Bot highlight
    class AI,Chat primary
```

**Clawdbotは**

「いろんなチャットサービス」と「AI」を
つなぐための便利なツール

会話の履歴を覚えてくれたり、
画像を送ったりもできる

……ふふ、そんな感じかな

---

……これなら少しわかりやすいかな？
