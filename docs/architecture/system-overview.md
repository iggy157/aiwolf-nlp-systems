# システム全体アーキテクチャ

> このドキュメントでは、AIWolf NLPシステムを構成する3つのコンポーネントの役割と関係を、初めてシステムを見る人が直感的に理解できるよう説明します。

## システムの概要

AIWolf NLPシステムは、**LLM（大規模言語モデル）を使ったAIエージェント同士が人狼ゲームを対戦するためのプラットフォーム**です。

システムは以下の3つのコンポーネントで構成されています:

| コンポーネント | 役割の一言説明 | 言語 |
|---|---|---|
| **aiwolf-nlp-server** | ゲームの進行を管理する「ゲームマスター」 | Go |
| **aiwolf-nlp-agent-llm** | LLMを使って人狼ゲームをプレイする「プレイヤー」 | Python |
| **aiwolf-nlp-common** | サーバとエージェント間の通信を橋渡しする「通訳」 | Python |

## システム構成図

```mermaid
graph TB
    subgraph "AIWolf NLP System"
        subgraph Server["🖥️ Game Server (Go)"]
            S["aiwolf-nlp-server<br/>ゲームの司会進行役<br/>Gin + gorilla/websocket"]
            WR["WaitingRoom<br/>エージェントが全員揃うまで<br/>接続を受け付けるプール"]
            GL["Game Logic<br/>昼夜のフェーズ進行<br/>投票集計・勝敗判定"]
            SV["Services<br/>ログ記録<br/>リアルタイム配信"]
        end

        subgraph Agents["🤖 Agent (Python) × 5プロセス"]
            A1[Agent 1<br/>例: 占い師]
            A2[Agent 2<br/>例: 人狼]
            A3[Agent 3<br/>例: 騎士]
            A4[Agent 4<br/>例: 村人]
            A5[Agent 5<br/>例: 狂人]
        end

        subgraph Common["📦 Common Library (Python)"]
            CL["aiwolf-nlp-common<br/>サーバからのJSONを<br/>Pythonオブジェクトに変換する<br/>通信の橋渡し役"]
        end

        subgraph LLMProvider["🧠 LLM Provider"]
            LLM["OpenAI / Google / Ollama<br/>エージェントの「頭脳」<br/>発言・投票・判断を生成"]
        end
    end

    A1 & A2 & A3 & A4 & A5 -->|"WebSocket通信<br/>サーバとの全やり取り<br/>ws://host:8080/ws"| S
    S --> WR --> GL
    GL --> SV
    A1 & A2 & A3 & A4 & A5 -.->|"import<br/>通信処理に利用"| CL
    A1 & A2 & A3 & A4 & A5 -->|"API呼び出し<br/>発言や判断を<br/>LLMに考えさせる"| LLM
```

## 各コンポーネントの役割

### aiwolf-nlp-server（ゲームサーバ）

ゲーム全体の司会進行を担当します。具体的には:
- エージェントのWebSocket接続を受け付け、全員が揃うまで待機する
- 役職をランダムに割り当ててゲームを開始する
- 昼（議論→投票→追放）と夜（占い→護衛→囁き→襲撃）のフェーズを順番に進行する
- 各エージェントにリクエストを送信し、応答を待つ（タイムアウト管理つき）
- 勝利条件を判定してゲームを終了する

### aiwolf-nlp-agent-llm（LLMエージェント）

LLMを頭脳として使い、人狼ゲームをプレイする「プレイヤー」です:
- 5つの独立したプロセスとして起動し、それぞれがサーバに接続する
- サーバからリクエスト（「発言して」「投票して」等）が届くたびに、ゲーム状況をLLMに伝えて応答を生成する
- 割り当てられた役職に応じて、占い・護衛・襲撃などの固有アクションも実行する
- LLMとの会話履歴をゲーム全体で保持し、文脈のある発言・判断を行う

### aiwolf-nlp-common（共通ライブラリ）

サーバ（Go）とエージェント（Python）の間の通信プロトコルを抽象化する「通訳」です:
- WebSocket接続の確立・管理を行う（`Client`クラス）
- サーバから送られてくるJSON形式のパケットをPythonのデータクラスに変換する
- エージェントがサーバにテキストを送信するインターフェースを提供する

## コンポーネント関係図（ファイルレベル）

各コンポーネントの主要ファイルがどのように連携しているかを示します。

```mermaid
graph LR
    subgraph "aiwolf-nlp-server"
        direction TB
        Server["core/server.go<br/>WebSocket接続を受け付ける入口"]
        WaitingRoom["core/waiting_room.go<br/>全員揃うまで接続を保持"]
        Game["logic/game.go<br/>ゲームのメインループ<br/>昼→夜を繰り返す"]
        Phases["logic/<br/>各フェーズの処理<br/>divine, guard, attack,<br/>vote, execution,<br/>communication"]
        Models["model/<br/>データ構造の定義<br/>Packet, Info, Setting,<br/>Agent, Role, Status"]
        Services["service/<br/>ゲームログ記録<br/>リアルタイム配信<br/>音声合成(TTS)"]

        Server --> WaitingRoom --> Game --> Phases
        Game --> Models
        Game --> Services
    end

    subgraph "aiwolf-nlp-common"
        direction TB
        Client["client.py<br/>WebSocket通信の管理<br/>send() / receive()"]
        Packet["packet/<br/>JSON→Pythonオブジェクト変換<br/>Packet, Info, Setting,<br/>Talk, Vote, Judge, Role等"]

        Client --> Packet
    end

    subgraph "aiwolf-nlp-agent-llm"
        direction TB
        Main["main.py<br/>エントリポイント<br/>N個のプロセスを起動"]
        Starter["starter.py<br/>サーバ接続の確立と<br/>ゲームセッションの管理"]
        AgentBase["agent/agent.py<br/>全役職の共通処理<br/>LLM呼び出しの中核"]
        Roles["agent/<br/>役職ごとのクラス<br/>villager, werewolf,<br/>seer, medium,<br/>bodyguard, possessed"]
        Utils["utils/<br/>エージェント生成<br/>ロギング<br/>タイムアウト制御"]

        Main --> Starter --> AgentBase --> Roles
        AgentBase --> Utils
    end

    Starter -.->|"Clientを使って<br/>サーバと通信"| Client
    AgentBase -.->|"Packetを使って<br/>データを解釈"| Packet
```

## 通信プロトコル（1ゲームの流れ）

サーバとエージェント間の通信は **WebSocket** で行われ、データは **JSON形式** でやり取りされます。

- **サーバ → エージェント**: JSON形式のパケット（リクエスト種別 + ゲーム状態 + 会話履歴）
- **エージェント → サーバ**: テキスト文字列（発言内容 or エージェント名）

```mermaid
sequenceDiagram
    participant Agent as 🤖 Agent (Python)
    participant Server as 🖥️ Server (Go)
    participant LLM as 🧠 LLM API

    Agent->>Server: WebSocket接続
    Server->>Agent: NAME リクエスト
    Agent->>Server: エージェント名 (例: "kanolab1")

    Note over Server: WaitingRoomで待機<br/>N人揃ったらゲーム作成

    Server->>Agent: INITIALIZE パケット<br/>（あなたの役職・ゲーム設定）
    Agent->>LLM: 初期化プロンプト<br/>（「あなたは占い師です…」）
    LLM-->>Agent: 応答

    loop ゲームループ (昼→夜を繰り返す)
        Server->>Agent: DAILY_INITIALIZE<br/>（今日の日数・前日の結果）

        loop 議論フェーズ
            Server->>Agent: TALK<br/>（他の人の発言履歴を添付）
            Agent->>LLM: 発言を考えさせる
            LLM-->>Agent: 発言テキスト
            Agent->>Server: 発言 or "Over"（発言終了）
        end

        Server->>Agent: VOTE（投票依頼）
        Agent->>LLM: 投票先を考えさせる
        LLM-->>Agent: 投票先
        Agent->>Server: "Agent[XX]"

        Note over Server: 投票集計 → 追放実行

        Server->>Agent: DAILY_FINISH（夜の開始）

        Note over Server: 夜のフェーズ実行<br/>（占い → 護衛 → 囁き → 襲撃）<br/>各役職に個別リクエスト
    end

    Server->>Agent: FINISH（ゲーム終了）<br/>全員の役職を公開
    Note over Agent: ログ出力・接続終了
```

## デプロイ構成

実際にシステムを動かすときの構成です。1台のマシン上でサーバとエージェントの両方を起動します。

```mermaid
graph TB
    subgraph "実行環境"
        subgraph "サーバプロセス（1つ）"
            GoServer["Go Server<br/>ポート :8080 でWebSocket待機"]
        end

        subgraph "エージェントプロセス群（5つ）"
            MP["Main Process<br/>main.py が5つの子プロセスを起動"]
            MP --> P1[Agent 1]
            MP --> P2[Agent 2]
            MP --> P3[Agent 3]
            MP --> P4[Agent 4]
            MP --> P5[Agent 5]
        end

        subgraph "設定ファイル"
            SC["Server Config<br/>default_5.yml<br/>（ゲームルール・タイムアウト等）"]
            AC["Agent Config<br/>config.yml<br/>（LLM設定・プロンプト等）"]
            ENV[".env<br/>（LLM APIキー）"]
        end
    end

    GoServer -.->|読込| SC
    MP -.->|読込| AC
    MP -.->|読込| ENV
    P1 & P2 & P3 & P4 & P5 -->|WebSocket| GoServer
```
