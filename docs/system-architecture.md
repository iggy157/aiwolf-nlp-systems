# AIWolf NLP システム全体アーキテクチャ詳細ドキュメント

本ドキュメントは、AIWolf NLP（自然言語処理人狼）対戦システムを構成する3つのリポジトリ（`aiwolf-nlp-server`、`aiwolf-nlp-common`、`aiwolf-nlp-agent-llm`）の構造・データフロー・関数レベルの動きを、フロー図作成の下地としてまとめたものである。

---

## 目次

1. [システム全体像](#1-システム全体像)
2. [aiwolf-nlp-server（ゲームサーバ）](#2-aiwolf-nlp-serverゲームサーバ)
3. [aiwolf-nlp-common（共通パッケージ）](#3-aiwolf-nlp-common共通パッケージ)
4. [aiwolf-nlp-agent-llm（LLMエージェント）詳細](#4-aiwolf-nlp-agent-llmllmエージェント詳細)
5. [システム全体のデータフロー](#5-システム全体のデータフロー)
6. [1ゲームの完全なライフサイクル](#6-1ゲームの完全なライフサイクル)

---

## 1. システム全体像

### 1.1 3つのコンポーネントの関係

```
┌─────────────────────────────┐
│   aiwolf-nlp-agent-llm      │  ← LLMを使って人狼ゲームをプレイするエージェント (Python)
│   (複数プロセスで複数体起動)  │
└──────────┬──────────────────┘
           │ WebSocket通信 (JSON)
           │ ↕ パケット送受信
┌──────────┴──────────────────┐
│   aiwolf-nlp-server          │  ← ゲーム進行を管理するサーバ (Go/Gin)
│   (ゲームロジック・通信管理)  │
└─────────────────────────────┘

┌─────────────────────────────┐
│   aiwolf-nlp-common          │  ← エージェント側で使う共通ライブラリ (Python)
│   (パケット解析・WebSocket    │     サーバからのJSONをdataclassに変換
│    クライアント)              │
└─────────────────────────────┘
```

**通信プロトコル**: WebSocket (`ws://host:port/ws`)
**データ形式**: JSON（サーバ→エージェント: Packetオブジェクト、エージェント→サーバ: 文字列）

### 1.2 技術スタック

| コンポーネント | 言語 | 主要ライブラリ |
|---|---|---|
| server | Go | Gin, gorilla/websocket |
| common | Python >=3.11 | websocket-client |
| agent-llm | Python >=3.11 | LangChain (OpenAI/Google/Ollama), Jinja2, aiwolf-nlp-common |

---

## 2. aiwolf-nlp-server（ゲームサーバ）

### 2.1 サーバ起動フロー

```
main.go
  ↓ コマンドライン引数解析 (-c config.yml)
  ↓ YAML設定ファイル読み込み
  ↓ Server構造体を生成
  ↓ server.Run()
      ↓ HTTPサーバ起動 (Gin)
      ↓ /ws エンドポイントでWebSocket接続待機
      ↓ SIGTERM/SIGHUP/SIGINT シグナルハンドラ登録
```

### 2.2 サーバの主要構造体

```go
type Server struct {
    config              model.Config       // YAML設定
    upgrader            websocket.Upgrader // WebSocketアップグレーダ
    waitingRoom         *WaitingRoom       // 接続待ちエージェントのプール
    matchOptimizer      *MatchOptimizer    // マッチング最適化（任意）
    gameSetting         *model.Setting     // ゲーム設定
    games               sync.Map           // 進行中のゲーム一覧（スレッドセーフ）
    realtimeBroadcaster *service.RealtimeBroadcaster // リアルタイム配信
}
```

### 2.3 接続〜ゲーム開始の流れ

```
エージェントがWebSocket接続
  ↓
handleConnections(w, r)
  ↓ WebSocketアップグレード
  ↓ (認証が有効なら) トークン検証
  ↓ NAME リクエスト送信 → エージェント名を受信
  ↓ チーム名を抽出（名前末尾の数字を除去: "kanolab1" → "kanolab"）
  ↓ Connection オブジェクトを WaitingRoom に追加
  ↓
WaitingRoom がエージェント数の条件を満たしたら
  ↓ 接続を取り出し
  ↓ Game 構造体を生成（役職ランダム割当）
  ↓ 別 goroutine で game.Start() を実行
```

**マッチングモード**:
- **セルフマッチ（デフォルト）**: 同じチームのエージェントN体でゲームを構成
- **非セルフマッチ**: 異なるチームから1体ずつN体を集める
- **マッチオプティマイザ**: 事前スケジュールに基づくマッチング

### 2.4 ゲーム進行ループ（`game.Start()`）

```
1. INITIALIZE パケットを全エージェントに送信
   （setting, info, profile を含む）

2. メインループ: while !shouldFinish()
   ├─ progressDay()    // 昼フェーズ
   │   ├─ isDaytime = true
   │   ├─ DAILY_INITIALIZE を全エージェントに送信
   │   └─ 設定されたフェーズを順次実行:
   │       ├─ talk（議論フェーズ）
   │       ├─ vote（投票フェーズ）
   │       └─ execution（追放処理）
   │
   ├─ progressNight()  // 夜フェーズ
   │   ├─ isDaytime = false
   │   ├─ DAILY_FINISH を全エージェントに送信
   │   └─ 設定されたフェーズを順次実行:
   │       ├─ divine（占い）
   │       ├─ guard（護衛）
   │       ├─ whisper（人狼同士の密談）
   │       └─ attack（襲撃）
   │
   └─ currentDay++

3. FINISH パケットを全エージェントに送信
4. 全接続をクローズ
```

**ゲーム終了条件** (`shouldFinish()`):
- エラーエージェント率が閾値を超過
- 村人陣営勝利: 人狼が全員死亡
- 人狼陣営勝利: 人狼の数 >= 村人の数

### 2.5 通信モード

#### ターン制通信（`communication_turn.go`）

```
エージェント順序をシャッフル
for turn in 0..MaxCount.PerDay:
  for agent in shuffled_agents:
    ├─ TALK（またはWHISPER）リクエスト送信
    │   パケット内容: request, info(ゲーム状態), talk_history(差分のみ)
    ├─ エージェントからテキスト応答を受信
    ├─ テキスト処理:
    │   ├─ 文字数制限チェック・トリム
    │   ├─ メンション(@AgentName)の長さ別計算
    │   ├─ "ForceSkip" → "Skip" に変換
    │   ├─ スキップ上限超過時 → "Over" に変換
    │   └─ 空テキスト → "Over" に変換
    ├─ Talk オブジェクト構築
    └─ ログ出力・ブロードキャスト
  全エージェントが "Over" → ループ終了
```

#### フリーフォーム通信（`communication_freeform.go`）

```
TALK_PHASE_START を全エージェントに送信
タイムアウト付きコンテキスト作成
  ├─ 各エージェントごとにgoroutine起動
  │   └─ エージェントのmsgChanからメッセージを読み取り
  │       └─ talkChannelに送信
  ├─ メインコーディネータgoroutine:
  │   └─ talkChannelからメッセージを受信
  │       ├─ バリデーション（残カウント、残文字数）
  │       ├─ Talk オブジェクト構築
  │       └─ TALK_BROADCAST として全エージェントに配信
  └─ タイムアウト or 全エージェント完了
TALK_PHASE_END を全エージェントに送信
```

### 2.6 パケット構造（サーバ → エージェント）

```json
{
  "request": "TALK",
  "info": {
    "game_id": "01JQXYZ...",
    "day": 2,
    "agent": "Agent[01]",
    "profile": "...",
    "medium_result": { "day": 1, "agent": "Agent[03]", "target": "Agent[05]", "result": "HUMAN" },
    "divine_result": { "day": 1, "agent": "Agent[01]", "target": "Agent[04]", "result": "WEREWOLF" },
    "executed_agent": "Agent[05]",
    "attacked_agent": "Agent[02]",
    "vote_list": [{ "day": 1, "agent": "Agent[01]", "target": "Agent[05]" }, ...],
    "attack_vote_list": [...],
    "status_map": { "Agent[01]": "ALIVE", "Agent[02]": "DEAD", ... },
    "role_map": { "Agent[01]": "SEER" },
    "remain_count": 5,
    "remain_length": 500,
    "remain_skip": 2
  },
  "setting": { "agent_count": 5, "talk": { "max_count": {...}, ... }, ... },
  "talk_history": [{ "idx": 0, "day": 1, "turn": 0, "agent": "Agent[03]", "text": "...", "skip": false, "over": false }, ...],
  "whisper_history": [...]
}
```

**エージェント → サーバの応答形式**:
- TALK/WHISPER: 自然言語テキスト（例: `"Agent[03]が怪しいと思います"`）
- VOTE/DIVINE/GUARD/ATTACK: エージェント名（例: `"Agent[03]"`）

### 2.7 タイムアウト処理

```
1. actionTimeout + acceptableTimeout でレスポンス待ち
2. タイムアウト発生時:
   ├─ NAME リクエストを送信して生存確認
   ├─ responseTimeout で応答待ち
   ├─ 応答なし or 不正 → agent.HasError = true
   └─ 以降このエージェントへのリクエストをスキップ
```

### 2.8 ゲームメカニクス

| フェーズ | 処理内容 |
|---|---|
| **投票 (vote)** | 全生存エージェントが投票 → 最多得票者を追放。同数の場合は再投票（最大MaxCount回）→ それでも決まらなければランダム |
| **占い (divine)** | 占い師がターゲットを選択 → ターゲットの種族（HUMAN/WEREWOLF）を結果として保存 → 翌日のInfoで占い師に通知 |
| **護衛 (guard)** | 騎士がターゲットを選択 → 夜の襲撃対象と一致すれば襲撃を阻止 |
| **襲撃 (attack)** | 人狼が襲撃対象を投票で決定 → 護衛されていなければターゲットを死亡に |
| **霊能 (medium)** | 追放されたエージェントの種族（HUMAN/WEREWOLF）を霊能者に翌日通知 |

### 2.9 差分配信の仕組み（`minimize()`）

サーバは各エージェントごとに `lastTalkIdxMap` / `lastWhisperIdxMap` を保持し、前回送信済みのインデックス以降の新しいトーク/ウィスパーのみを `talk_history` / `whisper_history` に含める。これにより帯域を削減している。

---

## 3. aiwolf-nlp-common（共通パッケージ）

### 3.1 役割

エージェント側で使用するPythonライブラリ。サーバからのJSON通信をPythonのdataclassオブジェクトに変換し、WebSocket通信を抽象化する。

### 3.2 パッケージ構造

```
src/aiwolf_nlp_common/
├── __init__.py          # Client をエクスポート
├── client.py            # WebSocketクライアント
└── packet/
    ├── __init__.py      # 全パケットクラスをエクスポート
    ├── packet.py        # Packet（ルートコンテナ）
    ├── request.py       # Request（リクエスト種別enum）
    ├── info.py          # Info（ゲーム状態）
    ├── setting.py       # Setting（ゲーム設定）
    ├── talk.py          # Talk（発言データ）
    ├── role.py          # Role, Team, Species（役職・陣営・種族enum）
    ├── status.py        # Status（生死状態enum）
    ├── judge.py         # Judge（占い・霊能結果）
    └── vote.py          # Vote（投票データ）
```

### 3.3 Client クラス（`client.py`）

```python
class Client:
    socket: websocket.WebSocket   # WebSocket接続
    url: str                      # サーバURL
    headers: list[str]            # HTTPヘッダ（User-Agent, Bearerトークン）

    def connect() -> None         # WebSocket接続を確立
    def receive() -> Packet       # JSONを受信しPacketオブジェクトに変換
    def send(req: str) -> None    # 文字列をサーバに送信（末尾に改行付与）
    def close() -> None           # 接続をクローズ
```

**`receive()` の内部動作**:
1. WebSocketからJSON文字列を受信
2. `json.loads()` でdict化
3. `Packet.from_dict(obj)` でdataclassに変換
4. Packet オブジェクトを返却

### 3.4 主要データクラス

#### Packet（ルートコンテナ）

```python
@dataclass
class Packet:
    request: Request              # リクエスト種別
    info: Info | None             # ゲーム状態
    setting: Setting | None       # ゲーム設定
    talk_history: list[Talk] | None    # トーク履歴
    whisper_history: list[Talk] | None # ウィスパー履歴
    new_talk: Talk | None         # 最新トーク（フリーフォーム時のブロードキャスト）
    new_whisper: Talk | None      # 最新ウィスパー（フリーフォーム時）
```

#### Request（リクエスト種別enum）

| リクエスト | 応答要否 | 説明 |
|---|---|---|
| `NAME` | 要 | エージェント名を返す |
| `INITIALIZE` | 不要 | ゲーム開始通知 |
| `DAILY_INITIALIZE` | 不要 | 昼開始通知（前日の結果を含む） |
| `TALK` | 要 | 発言リクエスト |
| `WHISPER` | 要 | 人狼ウィスパーリクエスト |
| `VOTE` | 要 | 投票リクエスト |
| `DIVINE` | 要 | 占いリクエスト |
| `GUARD` | 要 | 護衛リクエスト |
| `ATTACK` | 要 | 襲撃リクエスト |
| `DAILY_FINISH` | 不要 | 昼終了通知 |
| `FINISH` | 不要 | ゲーム終了通知 |
| `TALK_PHASE_START` | 不要 | フリーフォームトーク開始 |
| `TALK_PHASE_END` | 不要 | フリーフォームトーク終了 |
| `TALK_BROADCAST` | 不要 | 他エージェントの新しいトーク通知 |
| `WHISPER_PHASE_START` | 不要 | フリーフォームウィスパー開始 |
| `WHISPER_PHASE_END` | 不要 | フリーフォームウィスパー終了 |
| `WHISPER_BROADCAST` | 不要 | 新しいウィスパー通知 |

#### Info（ゲーム状態）

```python
@dataclass
class Info:
    game_id: str                          # ゲーム一意ID（ULID形式）
    day: int                              # 現在の日数
    agent: str                            # 自分のエージェント名
    profile: str | None                   # エージェントの経歴（INITIALIZE時のみ）
    medium_result: Judge | None           # 霊能結果（霊能者のみ）
    divine_result: Judge | None           # 占い結果（占い師のみ）
    executed_agent: str | None            # 前日追放されたエージェント
    attacked_agent: str | None            # 前夜襲撃されたエージェント
    vote_list: list[Vote] | None          # 投票結果リスト
    attack_vote_list: list[Vote] | None   # 襲撃投票結果（人狼のみ）
    status_map: dict[str, Status]         # 全エージェントの生死状態
    role_map: dict[str, Role]             # 役職マップ（自分の役職のみ、FINISH時は全員）
    remain_count: int | None              # 残り発言回数
    remain_length: int | None             # 残り文字数
    remain_skip: int | None               # 残りスキップ回数
```

#### Role, Team, Species

```python
class Role(Enum):      # team / species
    WEREWOLF  = ...    # WEREWOLF / WEREWOLF（人狼）
    POSSESSED = ...    # WEREWOLF / HUMAN（狂人）
    SEER      = ...    # VILLAGER / HUMAN（占い師）
    MEDIUM    = ...    # VILLAGER / HUMAN（霊能者）
    BODYGUARD = ...    # VILLAGER / HUMAN（騎士）
    VILLAGER  = ...    # VILLAGER / HUMAN（村人）

class Team(Enum):      VILLAGER, WEREWOLF
class Species(Enum):   HUMAN, WEREWOLF
class Status(Enum):    ALIVE, DEAD
```

#### Talk, Judge, Vote

```python
@dataclass
class Talk:
    idx: int       # メッセージ通番
    day: int       # 発言日
    turn: int      # ターン番号
    agent: str     # 発言者名
    text: str      # 発言内容
    skip: bool     # スキップかどうか
    over: bool     # 発言終了かどうか

@dataclass
class Judge:
    day: int       # 判定日
    agent: str     # 判定者名
    target: str    # 判定対象名
    result: Species  # HUMAN or WEREWOLF

@dataclass
class Vote:
    day: int       # 投票日
    agent: str     # 投票者名
    target: str    # 投票対象名
```

### 3.5 commonパッケージの役割まとめ

commonは**サーバとエージェント間の通信プロトコルを抽象化するアダプタ層**である。エージェント（agent-llm）はcommonを通じて：
1. WebSocket接続の確立・管理（`Client`）
2. サーバからのJSONパケットの受信・パース（`Client.receive()` → `Packet`）
3. ゲーム状態の型安全なアクセス（`Info`, `Setting`, `Talk`, etc.）
4. サーバへの応答送信（`Client.send()`）

を行う。サーバ側のGo実装とエージェント側のPython実装の間の**JSON ↔ dataclass変換を一元的に管理**している。

---

## 4. aiwolf-nlp-agent-llm（LLMエージェント）詳細

### 4.1 ディレクトリ構造

```
aiwolf-nlp-agent-llm/
├── src/
│   ├── main.py                    # エントリポイント
│   ├── starter.py                 # 接続・ゲームセッション管理
│   ├── agent/
│   │   ├── agent.py               # Agentベースクラス（全ロジックの中核）
│   │   ├── werewolf.py            # 人狼エージェント
│   │   ├── seer.py                # 占い師エージェント
│   │   ├── bodyguard.py           # 騎士エージェント
│   │   ├── medium.py              # 霊能者エージェント
│   │   ├── possessed.py           # 狂人エージェント
│   │   └── villager.py            # 村人エージェント
│   └── utils/
│       ├── agent_utils.py         # エージェントファクトリ
│       ├── agent_logger.py        # ロギング
│       └── stoppable_thread.py    # タイムアウト用強制停止スレッド
├── config/
│   ├── config.jp.yml.example      # 設定ファイル例（日本語プロンプト）
│   ├── config.en.yml.example      # 設定ファイル例（英語プロンプト）
│   └── .env                       # 環境変数（APIキー）
└── pyproject.toml
```

### 4.2 エントリポイントとプロセス起動（`main.py`）

```python
# main.py の流れ
def main():
    paths = sys.argv[1:]  # コマンドライン引数として設定ファイルパスを受け取る
    multiprocessing.set_start_method("spawn")
    threads = []
    for path in paths:
        thread = multiprocessing.Process(target=execute, args=(Path(path),))
        threads.append(thread)
        thread.start()
    for thread in threads:
        thread.join()

def execute(config_path: Path) -> None:
    config = yaml.safe_load(config_path.read_text())  # YAML読み込み
    agent_num = int(config["agent"]["num"])             # エージェント数取得
    threads = []
    for i in range(agent_num):
        thread = multiprocessing.Process(
            target=connect,           # starter.py の connect を呼び出し
            args=(config, i + 1)      # config と エージェントインデックス(1始まり)
        )
        threads.append(thread)
        thread.start()
    for thread in threads:
        thread.join()
```

**データの流れ**:
- `sys.argv[1:]` → 設定ファイルパス（複数指定可能）
- `config["agent"]["num"]` → エージェント数（例: 5）
- 各エージェントは独立した`multiprocessing.Process`で起動
- エージェント名 = `config["agent"]["team"]` + インデックス（例: `"kanolab1"`, `"kanolab2"`, ...）

### 4.3 接続・ゲームセッション管理（`starter.py`）

#### `connect(config, idx)` — エージェントのメインループ

```python
def connect(config: dict, idx: int = 1) -> None:
    name = str(config["agent"]["team"]) + str(idx)  # エージェント名構築

    while True:
        client = create_client(config)          # Clientオブジェクト生成
        connect_to_server(client, name)         # WebSocket接続（リトライあり）
        try:
            handle_game_session(client, config, name)  # ゲーム1セッション実行
        finally:
            client.close()                      # 接続クローズ

        if not bool(config["web_socket"]["auto_reconnect"]):
            break  # auto_reconnectがfalseなら終了
```

**変数の流れ**:
- `name`: `"kanolab"` + `"1"` → `"kanolab1"`
- `config["web_socket"]["url"]`: WebSocket URL（例: `"ws://127.0.0.1:8080/ws"`）
- `config["web_socket"]["token"]`: 認証トークン（null可）
- `config["web_socket"]["auto_reconnect"]`: ゲーム終了後に再接続するか

#### `create_client(config)` — WebSocketクライアント生成

```python
def create_client(config: dict) -> Client:
    url = str(config["web_socket"]["url"])
    token = config["web_socket"]["token"]  # None or str
    return Client(url, token)              # aiwolf_nlp_common.Client を使用
```

#### `connect_to_server(client, name)` — 接続リトライ

```python
def connect_to_server(client: Client, name: str) -> None:
    while True:
        try:
            client.connect()   # WebSocket接続確立（common.Client.connect()）
            break
        except Exception:
            sleep(15)          # 15秒待って再試行
```

#### `handle_game_session_async(client, config, name)` — ゲームセッションの非同期メインループ

これがゲーム1回分の全体制御を行う**最も重要な関数**である。

```python
async def handle_game_session_async(client, config, name):
    agent: Agent | None = None
    talk_task: asyncio.Task | None = None      # フリーフォームトーク用非同期タスク
    whisper_task: asyncio.Task | None = None   # フリーフォームウィスパー用非同期タスク
    send_lock = threading.Lock()               # 送信時の排他ロック

    def send_with_lock(text: str) -> None:
        with send_lock:
            client.send(text)

    while True:
        # ① サーバからパケット受信（ブロッキング、別スレッドで実行）
        packet: Packet = await asyncio.to_thread(client.receive)

        # ② NAME リクエスト処理
        if packet.request == Request.NAME:
            send_with_lock(name)
            continue

        # ③ INITIALIZE リクエストでエージェント初期化
        if packet.request == Request.INITIALIZE:
            agent = init_agent_from_packet(config, name, packet)
        if not agent:
            raise ValueError("エージェントが初期化されていません")

        # ④ パケットデータをエージェントに反映
        agent.set_packet(packet)

        # ⑤ リクエスト種別に応じた処理分岐
        match packet.request:
            # --- フリーフォーム通信モード ---
            case Request.TALK_PHASE_START:
                agent.in_talk_phase = True
                await cancel_task(talk_task)
                talk_task = asyncio.create_task(
                    agent.handle_talk_phase(send_with_lock)
                )

            case Request.TALK_PHASE_END:
                agent.in_talk_phase = False
                await cancel_task(talk_task)
                talk_task = None

            case Request.WHISPER_PHASE_START:
                agent.in_whisper_phase = True
                await cancel_task(whisper_task)
                whisper_task = asyncio.create_task(
                    agent.handle_whisper_phase(send_with_lock)
                )

            case Request.WHISPER_PHASE_END:
                agent.in_whisper_phase = False
                await cancel_task(whisper_task)
                whisper_task = None

            # --- ターン制通信モード & ライフサイクルイベント ---
            case _:
                req = agent.action()   # @timeout デコレータ付き
                agent.agent_logger.packet(agent.request, req)
                if req:
                    send_with_lock(req)

        # ⑥ FINISH ならループ終了
        if packet.request == Request.FINISH:
            agent.in_talk_phase = False
            agent.in_whisper_phase = False
            await cancel_task(talk_task)
            await cancel_task(whisper_task)
            break
```

**データフローの要点**:
1. `client.receive()` → `Packet` オブジェクト（commonパッケージがJSON→dataclass変換）
2. `agent.set_packet(packet)` → エージェント内部の `info`, `talk_history` 等を更新
3. `agent.action()` or `agent.handle_talk_phase()` → LLMに問い合わせ → 応答テキスト生成
4. `send_with_lock(text)` → `client.send()` → WebSocket経由でサーバに送信

### 4.4 エージェントファクトリ（`agent_utils.py`）

```python
ROLE_TO_AGENT_CLS: dict[Role, type[Agent]] = {
    Role.WEREWOLF:  Werewolf,
    Role.POSSESSED: Possessed,
    Role.SEER:      Seer,
    Role.BODYGUARD: Bodyguard,
    Role.VILLAGER:  Villager,
    Role.MEDIUM:    Medium,
}

def init_agent_from_packet(config, name, packet) -> Agent:
    role = packet.info.role_map.get(packet.info.agent)  # 自分の役職を取得
    return ROLE_TO_AGENT_CLS[role](                     # 役職に応じたクラスをインスタンス化
        config=config,
        name=name,
        game_id=packet.info.game_id,
        role=role,
    )
```

**データの流れ**:
- `packet.info.agent` → 自分のエージェント名（例: `"Agent[01]"`）
- `packet.info.role_map` → `{"Agent[01]": Role.SEER, ...}` → 自分の役職を取得
- 役職に対応するクラス（`Seer`, `Werewolf`, etc.）をインスタンス化

### 4.5 Agentベースクラス（`agent/agent.py`）— 中核ロジック

#### コンストラクタ `__init__(config, name, game_id, role)`

```python
def __init__(self, config, name, game_id, role):
    load_dotenv(Path("../../config/.env"))   # 環境変数（APIキー）読み込み

    self.config = config                      # YAML設定全体
    self.agent_name = name                    # エージェント名（例: "kanolab1"）
    self.agent_logger = AgentLogger(config, name, game_id)  # ロガー
    self.request: Request | None = None       # 現在のリクエスト種別
    self.info: Info | None = None             # 現在のゲーム状態
    self.setting: Setting | None = None       # ゲーム設定（タイムアウト等）
    self.talk_history: list[Talk] = []        # 全トーク履歴（ゲーム通算）
    self.whisper_history: list[Talk] = []     # 全ウィスパー履歴
    self.role: Role = role                    # 自分の役職
    self.in_talk_phase: bool = False          # フリーフォームトーク中フラグ
    self.in_whisper_phase: bool = False       # フリーフォームウィスパー中フラグ
    self.sent_talk_count: int = 0             # 前回talk()時点のtalk_historyの長さ
    self.sent_whisper_count: int = 0          # 前回whisper()時点のwhisper_historyの長さ
    self.llm_model: BaseChatModel | None = None          # LLMインスタンス
    self.llm_message_history: list[BaseMessage] = []     # LLMとの会話履歴
```

**各変数の役割**:

| 変数名 | 型 | 用途 | 更新タイミング |
|---|---|---|---|
| `config` | dict | YAML設定 | 初期化時のみ |
| `agent_name` | str | エージェント識別名 | 初期化時のみ |
| `request` | Request\|None | 現在処理中のリクエスト種別 | `set_packet()` 毎 |
| `info` | Info\|None | ゲーム状態（日数、生死、結果等） | `set_packet()` 毎 |
| `setting` | Setting\|None | ゲーム設定 | `set_packet()` 毎 |
| `talk_history` | list[Talk] | ゲーム通算の全トーク | `set_packet()` で追加、INITIALIZE時リセット |
| `whisper_history` | list[Talk] | ゲーム通算の全ウィスパー | `set_packet()` で追加、INITIALIZE時リセット |
| `role` | Role | 自分の役職 | 初期化時のみ |
| `in_talk_phase` | bool | フリーフォームトーク中か | starter.pyから設定 |
| `in_whisper_phase` | bool | フリーフォームウィスパー中か | starter.pyから設定 |
| `sent_talk_count` | int | 前回のtalk()呼び出し時のtalk_historyの長さ | `talk()` 実行後 |
| `sent_whisper_count` | int | 前回のwhisper()呼び出し時のwhisper_historyの長さ | `whisper()` 実行後 |
| `llm_model` | BaseChatModel\|None | LLM接続（OpenAI/Google/Ollama） | `initialize()` |
| `llm_message_history` | list[BaseMessage] | LLMとのマルチターン会話履歴 | `_send_message_to_llm()` で追加、INITIALIZE時リセット |

#### `set_packet(packet)` — パケットデータの反映

```python
def set_packet(self, packet: Packet) -> None:
    self.request = packet.request                           # ① リクエスト種別更新

    if packet.info:
        self.info = packet.info                             # ② ゲーム状態更新

    if packet.setting:
        self.setting = packet.setting                       # ③ ゲーム設定更新

    if packet.talk_history:
        self.talk_history.extend(packet.talk_history)       # ④ トーク履歴を追加

    if packet.whisper_history:
        self.whisper_history.extend(packet.whisper_history) # ⑤ ウィスパー履歴を追加

    # フリーフォームモード: 新着メッセージを追加
    if packet.new_talk:
        self.talk_history.append(packet.new_talk)           # ⑥ 新着トークを追加
        self.on_talk_received(packet.new_talk)              # フック呼び出し

    if packet.new_whisper:
        self.whisper_history.append(packet.new_whisper)     # ⑦ 新着ウィスパーを追加
        self.on_whisper_received(packet.new_whisper)        # フック呼び出し

    # INITIALIZE時は全履歴をリセット
    if self.request == Request.INITIALIZE:
        self.talk_history: list[Talk] = []                  # ⑧ トーク履歴リセット
        self.whisper_history: list[Talk] = []               # ⑨ ウィスパー履歴リセット
        self.llm_message_history: list[BaseMessage] = []    # ⑩ LLM会話履歴リセット
```

**サーバの差分配信との関係**:
- サーバは差分（新しいトーク/ウィスパーのみ）を送る
- エージェント側では `extend()` で蓄積していく
- テンプレートで `talk_history[sent_talk_count:]` とスライスすることで「前回のtalk()以降の新着」のみをLLMに渡す

#### `initialize()` — LLM初期化とゲーム開始プロンプト

```python
def initialize(self) -> None:
    match str(self.config["llm"]["type"]):
        case "openai":
            self.llm_model = ChatOpenAI(
                model=str(self.config["openai"]["model"]),        # 例: "gpt-4o-mini"
                temperature=float(self.config["openai"]["temperature"]),
                api_key=SecretStr(os.environ["OPENAI_API_KEY"]),
            )
        case "google":
            self.llm_model = ChatGoogleGenerativeAI(
                model=str(self.config["google"]["model"]),        # 例: "gemini-2.0-flash-lite"
                temperature=float(self.config["google"]["temperature"]),
                api_key=SecretStr(os.environ["GOOGLE_API_KEY"]),
            )
        case "ollama":
            self.llm_model = ChatOllama(
                model=str(self.config["ollama"]["model"]),        # 例: "llama3.1"
                temperature=float(self.config["ollama"]["temperature"]),
                base_url=str(self.config["ollama"]["base_url"]),
            )

    self._send_message_to_llm(self.request)  # INITIALIZEプロンプトをLLMに送信
```

**データの流れ**:
- `config["llm"]["type"]` → LLMプロバイダの選択
- 環境変数 `OPENAI_API_KEY` / `GOOGLE_API_KEY` → APIキー
- `self.llm_model` にLangChainモデルインスタンスを格納
- 初期化プロンプトを送信 → LLMにゲームのルールと役職を伝える → `llm_message_history` に最初のやり取りが記録される

#### `_send_message_to_llm(request)` — LLMへのメッセージ送信（中核メソッド）

全てのLLM問い合わせがこのメソッドを通る。

```python
def _send_message_to_llm(self, request: Request | None) -> str | None:
    # ① リクエストに対応するプロンプトが設定にあるか確認
    if request is None:
        return None
    if request.lower() not in self.config["prompt"]:
        return None

    # ② プロンプトテンプレート取得
    prompt_template = self.config["prompt"][request.lower()]

    # ③ レート制限対策のスリープ
    if float(self.config["llm"]["sleep_time"]) > 0:
        sleep(float(self.config["llm"]["sleep_time"]))  # デフォルト3秒

    # ④ テンプレートに注入するコンテキスト辞書を構築
    key = {
        "info":                self.info,               # ゲーム状態
        "setting":             self.setting,             # ゲーム設定
        "talk_history":        self.talk_history,        # 全トーク履歴
        "whisper_history":     self.whisper_history,     # 全ウィスパー履歴
        "role":                self.role,                # 自分の役職
        "sent_talk_count":     self.sent_talk_count,     # トーク位置マーカ
        "sent_whisper_count":  self.sent_whisper_count,  # ウィスパー位置マーカ
    }

    # ⑤ Jinja2テンプレートのレンダリング
    template: Template = Template(prompt_template)
    prompt = template.render(**key).strip()

    # ⑥ LLMモデルの存在確認
    if self.llm_model is None:
        return None

    # ⑦ LLMに送信し、会話履歴を更新
    try:
        self.llm_message_history.append(HumanMessage(content=prompt))   # プロンプトを履歴に追加
        response = (
            self.llm_model | StrOutputParser()
        ).invoke(self.llm_message_history)                              # 全履歴をLLMに送信
        self.llm_message_history.append(AIMessage(content=response))    # 応答を履歴に追加
    except Exception:
        return None

    return response
```

**データフローの詳細図**:

```
config["prompt"]["talk"] (Jinja2テンプレート文字列)
       ↓
Template(prompt_template)
       ↓
template.render(info=self.info, talk_history=self.talk_history, ...)
       ↓
prompt (レンダリング済みのプロンプト文字列)
       ↓
HumanMessage(content=prompt)
       ↓
self.llm_message_history に追加 → [HumanMessage1, AIMessage1, HumanMessage2, AIMessage2, ..., HumanMessageN]
       ↓
self.llm_model.invoke(self.llm_message_history)
       ↓  LLM API呼び出し (OpenAI / Google / Ollama)
       ↓
response (文字列)
       ↓
AIMessage(content=response)
       ↓
self.llm_message_history に追加 → [..., HumanMessageN, AIMessageN]
       ↓
return response
```

**`llm_message_history`（LLMマルチターン会話履歴）の重要性**:

この変数はゲーム全体を通してLLMとの全やり取りを蓄積する。LLMはこの履歴全体をコンテキストとして参照するため：
- ゲーム初期化時の役職説明を記憶
- 過去の発言・判断の文脈を保持
- 日ごとの結果や推理の積み重ねを理解

```
llm_message_history の構造:
[
  HumanMessage("あなたは人狼ゲームのエージェントです。役職は占い師..."),     # initialize
  AIMessage("了解しました..."),
  HumanMessage("昼開始。1日目。占い結果: Agent[03] は HUMAN"),            # daily_initialize
  AIMessage("情報を整理します..."),
  HumanMessage("トークリクエスト\n履歴:\nAgent[02]: 私は村人です..."),      # talk
  AIMessage("Agent[04]が怪しいと思います"),
  HumanMessage("投票リクエスト\n対象: Agent[02], Agent[03], Agent[04]"),   # vote
  AIMessage("Agent[04]"),
  ...
]
```

#### `sent_talk_count` / `sent_whisper_count` の役割

これらのインデックス変数は、**LLMに送るトーク履歴の差分制御**に使われる。

```
例: talk_history = [Talk0, Talk1, Talk2, Talk3, Talk4]

1回目のtalk()呼び出し時:
  sent_talk_count = 0
  テンプレート: talk_history[0:] → [Talk0, Talk1, Talk2] をLLMに送る
  talk()実行後: sent_talk_count = 3 (len(talk_history)=3の時点)

2回目のtalk()呼び出し時:
  sent_talk_count = 3
  テンプレート: talk_history[3:] → [Talk3, Talk4] をLLMに送る（新着のみ）
  talk()実行後: sent_talk_count = 5
```

#### アクションメソッド群

```python
def talk(self) -> str:
    response = self._send_message_to_llm(self.request)
    self.sent_talk_count = len(self.talk_history)    # 位置マーカ更新
    return response or ""

def whisper(self) -> str:
    response = self._send_message_to_llm(self.request)
    self.sent_whisper_count = len(self.whisper_history)  # 位置マーカ更新
    return response or ""

def vote(self) -> str:
    return self._send_message_to_llm(self.request) or random.choice(self.get_alive_agents())

def divine(self) -> str:   # 占い師のみ
    return self._send_message_to_llm(self.request) or random.choice(self.get_alive_agents())

def guard(self) -> str:    # 騎士のみ
    return self._send_message_to_llm(self.request) or random.choice(self.get_alive_agents())

def attack(self) -> str:   # 人狼のみ
    return self._send_message_to_llm(self.request) or random.choice(self.get_alive_agents())

def daily_initialize(self) -> None:
    self._send_message_to_llm(self.request)  # 応答不要（状態更新のみ）

def daily_finish(self) -> None:
    self._send_message_to_llm(self.request)  # 応答不要
```

**フォールバック動作**: `vote`, `divine`, `guard`, `attack` はLLMが応答に失敗した場合、`random.choice(self.get_alive_agents())` で生存エージェントからランダムに選択する。

#### `action()` — メインディスパッチャ（@timeout デコレータ付き）

```python
@timeout
def action(self) -> str | None:
    match self.request:
        case Request.NAME:              return self.name()
        case Request.TALK:              return self.talk()
        case Request.WHISPER:           return self.whisper()
        case Request.VOTE:              return self.vote()
        case Request.DIVINE:            return self.divine()
        case Request.GUARD:             return self.guard()
        case Request.ATTACK:            return self.attack()
        case Request.INITIALIZE:        self.initialize()
        case Request.DAILY_INITIALIZE:  self.daily_initialize()
        case Request.DAILY_FINISH:      self.daily_finish()
        case Request.FINISH:            self.finish()
        case _:                         pass
    return None
```

#### `@timeout` デコレータの仕組み

```python
def timeout(func):
    def _wrapper(*args, **kwargs):
        res = Exception("No result")

        def execute_with_timeout():
            nonlocal res
            res = func(*args, **kwargs)

        thread = StoppableThread(target=execute_with_timeout)
        thread.start()

        self = args[0]  # Agent インスタンス
        timeout_value = (self.setting.timeout.action if self.setting else 0) // 1000

        if timeout_value > 0:
            thread.join(timeout=timeout_value)
            if thread.is_alive():
                if bool(self.config["agent"]["kill_on_timeout"]):
                    thread.stop()  # ctypes で強制終了
        else:
            thread.join()

        if isinstance(res, Exception):
            raise res
        return res
    return _wrapper
```

**StoppableThread**: `ctypes.pythonapi.PyThreadState_SetAsyncExc` を使ってスレッドに `SystemExit` を送ることで強制終了する特殊スレッド。LLM APIの応答が遅い場合にサーバのタイムアウトに間に合わせるための仕組み。

#### フリーフォームチャットハンドラ

```python
async def handle_talk_phase(self, send: Callable[[str], None]) -> None:
    while self.in_talk_phase:
        if self.info and self.info.remain_count is not None and self.info.remain_count <= 0:
            break
        text = self.talk()             # LLMに問い合わせ → テキスト生成
        if not self.in_talk_phase:
            break
        send(text)                     # サーバに送信
        await asyncio.sleep(5)         # 5秒間隔

async def handle_whisper_phase(self, send: Callable[[str], None]) -> None:
    while self.in_whisper_phase:
        if self.info and self.info.remain_count is not None and self.info.remain_count <= 0:
            break
        text = self.whisper()          # LLMに問い合わせ → テキスト生成
        if not self.in_whisper_phase:
            break
        send(text)                     # サーバに送信
        await asyncio.sleep(5)         # 5秒間隔
```

**フリーフォーム時のデータの流れ**:
1. サーバから `TALK_PHASE_START` 受信
2. `asyncio.create_task(agent.handle_talk_phase(send_with_lock))` で非同期タスク起動
3. タスクが5秒ごとにLLM問い合わせ → テキスト生成 → サーバ送信を繰り返す
4. サーバは受信したテキストを `TALK_BROADCAST` として全エージェントに配信
5. 各エージェントは `TALK_BROADCAST` を受信 → `set_packet()` → `talk_history` に追加
6. サーバから `TALK_PHASE_END` 受信 → タスクをキャンセル

### 4.6 役職別エージェントクラス

全クラスは `Agent` を継承し、その役職で使用可能なメソッドのみをオーバーライドする（実装は `super()` を呼ぶだけ）。

```
Agent（ベースクラス）
  ├── Werewolf    : talk(), whisper(), vote(), attack()
  ├── Seer        : talk(), divine(), vote()
  ├── Bodyguard   : talk(), guard(), vote()
  ├── Villager    : talk(), vote()
  ├── Medium      : talk(), vote()
  └── Possessed   : talk(), vote()
```

**各役職が受け取る特別な情報**:

| 役職 | 特別に受け取る情報 | 特別なアクション |
|---|---|---|
| 人狼 (WEREWOLF) | `whisper_history`（人狼同士の密談）、`attack_vote_list` | `whisper()`, `attack()` |
| 占い師 (SEER) | `info.divine_result`（占い結果: HUMAN/WEREWOLF） | `divine()` |
| 騎士 (BODYGUARD) | なし | `guard()` |
| 霊能者 (MEDIUM) | `info.medium_result`（霊能結果: HUMAN/WEREWOLF） | なし |
| 村人 (VILLAGER) | なし | なし |
| 狂人 (POSSESSED) | なし（占いではHUMANと判定される） | なし |

### 4.7 プロンプトテンプレート（Jinja2）

設定ファイル（YAML）内に定義される。テンプレートには `_send_message_to_llm()` で構築した `key` 辞書の変数が注入される。

```yaml
prompt:
  initialize: |
    あなたは人狼ゲームのエージェントです。
    あなたの名前は{{ info.agent }}です。
    あなたの役職は{{ role.value }}です。
    ...

  daily_initialize: |
    昼開始リクエスト
    {{ info.day }}日目
    {% if info.medium_result is not none %}霊能結果: {{ info.medium_result }}{% endif %}
    {% if info.divine_result is not none %}占い結果: {{ info.divine_result }}{% endif %}
    ...

  talk: |
    トークリクエスト
    履歴:
    {% for w in talk_history[sent_talk_count:] %}{{ w.agent }}: {{ w.text }}
    {% endfor %}

  vote: |
    投票リクエスト
    対象:
    {% for k, v in info.status_map.items() %}{% if v == 'ALIVE' %}{{ k }}
    {% endif %}{% endfor %}

  divine: |
    占いリクエスト
    対象:
    {% for k, v in info.status_map.items() %}{% if v == 'ALIVE' %}{{ k }}
    {% endif %}{% endfor %}

  whisper: |
    囁きリクエスト
    履歴:
    {% for w in whisper_history[sent_whisper_count:] %}{{ w.agent }}: {{ w.text }}
    {% endfor %}
```

**テンプレート変数 → 実データの変換例**:

```
テンプレート: {{ info.agent }}          → "Agent[01]"
テンプレート: {{ role.value }}          → "SEER"
テンプレート: {{ info.day }}            → 2
テンプレート: {{ info.divine_result }}  → Judge(day=1, agent="Agent[01]", target="Agent[04]", result=Species.WEREWOLF)
テンプレート: info.status_map.items()  → [("Agent[01]","ALIVE"), ("Agent[02]","DEAD"), ...]
テンプレート: talk_history[sent_talk_count:] → [Talk(agent="Agent[03]", text="怪しい"), ...]
```

### 4.8 設定ファイル構造

```yaml
web_socket:
  url: ws://127.0.0.1:8080/ws   # サーバのWebSocket URL
  token: null                    # 認証トークン（null=なし）
  auto_reconnect: false          # ゲーム後の自動再接続

agent:
  num: 5                         # 起動するエージェント数
  team: kanolab                  # チーム名（エージェント名のプレフィクス）
  kill_on_timeout: true          # タイムアウト時にスレッドを強制終了

llm:
  type: google                   # LLMプロバイダ: openai / google / ollama
  sleep_time: 3                  # LLM呼び出し前のスリープ（秒）

openai:
  model: gpt-4o-mini
  temperature: 0.7

google:
  model: gemini-2.0-flash-lite
  temperature: 0.7

ollama:
  model: llama3.1
  temperature: 0.7
  base_url: http://localhost:11434

log:
  console_output: true
  file_output: true
  output_dir: ./log
  level: debug
  request:                        # リクエスト種別ごとのログ出力設定
    name: false
    initialize: false
    talk: true
    whisper: true
    vote: true
    divine: true
    guard: true
    attack: true
    ...

prompt:                           # Jinja2プロンプトテンプレート（上記参照）
  initialize: ...
  daily_initialize: ...
  talk: ...
  vote: ...
  ...
```

### 4.9 ロギング（`agent_logger.py`）

```python
class AgentLogger:
    def __init__(self, config, name, game_id):
        self.logger = logging.getLogger(name)
        # コンソールハンドラ（設定に応じて追加）
        # ファイルハンドラ（game_idのタイムスタンプでディレクトリ作成）
        #   → log/{timestamp}/{agent_name}.log

    def packet(self, req, res):
        # 設定でログ対象のリクエストかチェック
        if config["log"]["request"][req.lower()] == True:
            self.logger.info([str(req), res])
```

---

## 5. システム全体のデータフロー

### 5.1 完全なデータフロー図

```
┌────────────────────────────────────────────────────────────────────────┐
│ aiwolf-nlp-agent-llm (Python)                                         │
│                                                                        │
│  main.py                                                               │
│    ↓ YAML読み込み                                                      │
│    ↓ multiprocessing.Process × N体                                     │
│                                                                        │
│  starter.py::connect()                                                 │
│    ↓                                                                   │
│    ↓ Client(url, token)  ←── aiwolf_nlp_common.Client                 │
│    ↓ client.connect()    ←── WebSocket接続確立                         │
│    ↓                                                                   │
│  starter.py::handle_game_session_async()                               │
│    │                                                                   │
│    │  ┌──────────────────────── メインループ ──────────────────────┐   │
│    │  │                                                            │   │
│    │  │  packet = client.receive()                                 │   │
│    │  │    ↓ JSON受信 → Packet.from_dict() → Packet オブジェクト   │   │
│    │  │                                                            │   │
│    │  │  agent.set_packet(packet)                                  │   │
│    │  │    ↓ info, talk_history, whisper_history 等を更新           │   │
│    │  │                                                            │   │
│    │  │  agent.action() or handle_talk_phase()                     │   │
│    │  │    ↓                                                       │   │
│    │  │  _send_message_to_llm(request)                             │   │
│    │  │    ↓ config["prompt"][request] からテンプレート取得          │   │
│    │  │    ↓ Jinja2レンダリング (info, talk_history, role 等を注入) │   │
│    │  │    ↓ llm_message_history に HumanMessage 追加              │   │
│    │  │    ↓ llm_model.invoke(llm_message_history)                 │   │
│    │  │    ↓ LLM API呼び出し (OpenAI / Google / Ollama)            │   │
│    │  │    ↓ 応答テキスト取得                                      │   │
│    │  │    ↓ llm_message_history に AIMessage 追加                 │   │
│    │  │    ↓ return response                                       │   │
│    │  │                                                            │   │
│    │  │  client.send(response)                                     │   │
│    │  │    ↓ WebSocket経由でサーバにテキスト送信                    │   │
│    │  │                                                            │   │
│    │  └────────────────────────────────────────────────────────────┘   │
│                                                                        │
└───────────────────────────────┬────────────────────────────────────────┘
                                │ WebSocket (JSON ↕ テキスト)
┌───────────────────────────────┴────────────────────────────────────────┐
│ aiwolf-nlp-server (Go)                                                 │
│                                                                        │
│  WebSocket接続受付                                                     │
│    ↓ NAME リクエスト送信 → エージェント名受信                          │
│    ↓ WaitingRoom に追加                                                │
│    ↓ N体揃ったらGame生成                                               │
│                                                                        │
│  game.Start()                                                          │
│    ↓ INITIALIZE パケット送信（setting, info, role_map）                │
│    ↓                                                                   │
│    ↓ メインループ:                                                     │
│    │  昼: DAILY_INITIALIZE → TALK (or TALK_PHASE_START/END)            │
│    │       → VOTE → 追放処理                                           │
│    │  夜: DAILY_FINISH → DIVINE → GUARD → WHISPER → ATTACK            │
│    │                                                                   │
│    ↓ FINISH パケット送信                                               │
│                                                                        │
│  各リクエストで:                                                       │
│    buildInfo() → 差分トーク履歴を含むInfoオブジェクト構築              │
│    Packet{request, info, setting, talk_history} をJSON化               │
│    WebSocketで送信 → 応答受信（タイムアウト管理付き）                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.2 1回のTALKリクエストにおけるデータの流れ

```
[サーバ]
  buildInfo(agent, R_TALK)
    ↓ Info構築: game_id, day, agent名, status_map, role_map(自分のみ), remain_count, ...
    ↓ minimize(): lastTalkIdxMap[agent] 以降のトークのみ抽出
  Packet{request:"TALK", info:Info, talk_history:[差分Talk...]}
    ↓ JSON化
    ↓ WebSocket送信
        ↓
[common: Client.receive()]
  JSON文字列受信
    ↓ json.loads() → dict
    ↓ Packet.from_dict(dict)
    ↓   ├─ Request("TALK")
    ↓   ├─ Info.from_dict(dict["info"])
    ↓   ├─ [Talk.from_dict(t) for t in dict["talk_history"]]
    ↓   └─ Packet オブジェクト
        ↓
[agent-llm: handle_game_session_async]
  agent.set_packet(packet)
    ↓ self.request = Request.TALK
    ↓ self.info = Info(...)
    ↓ self.talk_history.extend([Talk1, Talk2, ...])  # 差分を蓄積
        ↓
  agent.action()  →  agent.talk()
    ↓
  _send_message_to_llm(Request.TALK)
    ↓ template = config["prompt"]["talk"]
    ↓ key = {info, talk_history, sent_talk_count, ...}
    ↓ prompt = template.render(**key)
    ↓   └─ talk_history[sent_talk_count:] で「前回以降の新着トーク」のみ展開
    ↓ llm_message_history.append(HumanMessage(prompt))
    ↓ response = llm_model.invoke(llm_message_history)
    ↓ llm_message_history.append(AIMessage(response))
    ↓ return response (例: "Agent[03]の発言は矛盾しています。怪しいです。")
        ↓
  self.sent_talk_count = len(self.talk_history)  # 次回用の位置マーカ更新
        ↓
  client.send(response)
    ↓ WebSocket送信 (文字列)
        ↓
[サーバ]
  応答テキスト受信
    ↓ テキスト処理（文字数制限、メンション処理、Skip/Over判定）
    ↓ Talk オブジェクト構築
    ↓ ゲーム状態に記録
    ↓ ログ出力・ブロードキャスト
```

---

## 6. 1ゲームの完全なライフサイクル

### 6.1 時系列フロー（5人戦の例）

```
[接続フェーズ]
  エージェント × 5 が WebSocket 接続
  各エージェントに NAME リクエスト → 名前応答
  5体揃ったら Game 生成、役職ランダム割当
  例: Agent[01]=占い師, Agent[02]=人狼, Agent[03]=騎士, Agent[04]=村人, Agent[05]=狂人

[ゲーム開始]
  サーバ → 全エージェント: INITIALIZE パケット
    └─ info.role_map = {自分の名前: 自分の役職} （自分の役職のみ見える）
    └─ info.status_map = 全員ALIVE
    └─ setting = ゲーム設定
  エージェント側: agent.initialize()
    └─ LLMモデル生成
    └─ INITIALIZEプロンプトをLLMに送信（役職通知）

[Day 1 — 昼]
  サーバ → 全エージェント: DAILY_INITIALIZE
    └─ info.day = 1, 結果は全てnull（初日なので）
  エージェント側: agent.daily_initialize()
    └─ DAILY_INITIALIZEプロンプトをLLMに送信

  --- 議論フェーズ ---
  ターン制の場合:
    サーバ → Agent[03]: TALK (talk_history=[])
    Agent[03] → サーバ: "私は騎士です"
    サーバ → Agent[01]: TALK (talk_history=[Agent[03]の発言])
    Agent[01] → サーバ: "私は占い師です。Agent[04]を占いました"
    ...（複数ターン繰り返し）

  フリーフォームの場合:
    サーバ → 全エージェント: TALK_PHASE_START
    各エージェントは5秒ごとにtalk()を呼び出し → サーバに送信
    サーバは受信した発言をTALK_BROADCASTで全エージェントに配信
    制限時間後 → サーバ → 全エージェント: TALK_PHASE_END

  --- 投票フェーズ ---
  サーバ → 全エージェント: VOTE (info.status_map)
  各エージェント → サーバ: ターゲット名 (例: "Agent[02]")
  サーバ: 得票数集計 → 最多得票者を追放
  例: Agent[04]が追放される → status_map[Agent[04]] = DEAD

[Day 1 — 夜]
  サーバ → 全エージェント: DAILY_FINISH
  エージェント側: agent.daily_finish()

  --- 占いフェーズ ---
  サーバ → Agent[01](占い師): DIVINE (生存エージェントリスト)
  Agent[01] → サーバ: "Agent[02]"
  サーバ: Agent[02]の種族はWEREWOLF → divine_result保存

  --- 護衛フェーズ ---
  サーバ → Agent[03](騎士): GUARD (生存エージェントリスト)
  Agent[03] → サーバ: "Agent[01]"
  サーバ: guard_target = Agent[01]

  --- 襲撃フェーズ ---
  サーバ → Agent[02](人狼): ATTACK (生存エージェントリスト)
  Agent[02] → サーバ: "Agent[01]"
  サーバ: guard_target == attack_target → 襲撃阻止！ attacked_agent = null

  --- 霊能判定 ---
  Agent[04]が追放された → 種族はHUMAN → medium_result保存

[Day 2 — 昼]
  サーバ → 全エージェント: DAILY_INITIALIZE
    └─ info.day = 2
    └─ info.executed_agent = "Agent[04]"
    └─ info.attacked_agent = null (護衛成功)
    └─ Agent[01]のinfo.divine_result = Judge(target="Agent[02]", result=WEREWOLF)
    └─ 霊能者のinfo.medium_result = Judge(target="Agent[04]", result=HUMAN)

  --- 議論フェーズ ---（占い師が結果を共有、推理が進む）
  --- 投票フェーズ ---
  例: Agent[02]（人狼）が追放される

[ゲーム終了判定]
  人狼が全滅 → 村人陣営の勝利

[ゲーム終了]
  サーバ → 全エージェント: FINISH
    └─ info.role_map = 全エージェントの役職（全公開）
  エージェント側: agent.finish()
  全接続クローズ
```

### 6.2 エージェント内部の状態遷移

```
[未初期化] ─── INITIALIZE ──→ [初期化済み]
                                 │ LLMモデル生成
                                 │ llm_message_history = [初期化プロンプト, 応答]
                                 │
                    ┌────────────┴────────────┐
                    ↓                         ↓
            [ターン制モード]           [フリーフォームモード]
                    │                         │
      DAILY_INITIALIZE                DAILY_INITIALIZE
        ↓ info更新                      ↓ info更新
        ↓ LLMに状況報告                ↓ LLMに状況報告
                    │                         │
      TALK リクエスト              TALK_PHASE_START
        ↓ talk()                      ↓ handle_talk_phase()起動
        ↓ LLM問い合わせ               ↓ 5秒ごとにtalk()ループ
        ↓ sent_talk_count更新          │
        ↓ テキスト応答                 │ TALK_BROADCAST受信
                    │                  ↓ set_packet() → talk_history追加
      VOTE リクエスト                  │
        ↓ vote()                    TALK_PHASE_END
        ↓ LLM問い合わせ               ↓ タスクキャンセル
        ↓ エージェント名応答           │
                    │                  ↓ VOTE リクエスト
      DAILY_FINISH                     │
        ↓ LLMに1日の振り返り           │
                    │                         │
      [夜のアクション]                [夜のアクション]
      DIVINE/GUARD/ATTACK             DIVINE/GUARD/ATTACK
        ↓ 役職に応じたアクション        ↓ 同左
                    │                         │
                    └────────────┬────────────┘
                                 ↓
                    次の日のDAILY_INITIALIZE
                    （繰り返し）
                                 ↓
                              FINISH
                    llm_message_historyは保持されたまま
                    接続クローズ
```

---

## 付録: 依存関係まとめ

### agent-llm の依存パッケージ

| パッケージ | バージョン | 用途 |
|---|---|---|
| aiwolf-nlp-common | ==0.7.0 | サーバ通信・パケット解析 |
| langchain-openai | >=0.3.9 | OpenAI LLM統合 |
| langchain-google-genai | >=2.1.0 | Google Gemini LLM統合 |
| langchain-ollama | >=0.3.0 | Ollama LLM統合 |
| jinja2 | >=3.1.6 | プロンプトテンプレートエンジン |
| pyyaml | >=6.0.2 | 設定ファイル読み込み |
| python-dotenv | >=1.1.0 | 環境変数読み込み |
| python-ulid | >=3.0.0 | ゲームIDのタイムスタンプ解析 |

### common の依存パッケージ

| パッケージ | バージョン | 用途 |
|---|---|---|
| websocket-client | >=1.8.0 | WebSocket通信 |

### server の依存パッケージ

| パッケージ | 用途 |
|---|---|
| gin-gonic/gin | HTTPフレームワーク |
| gorilla/websocket | WebSocket通信 |
| yaml.v3 | 設定ファイル読み込み |
