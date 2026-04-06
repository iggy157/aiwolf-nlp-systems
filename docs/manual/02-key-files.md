# 重要ファイル・関数リファレンス

## ディレクトリ構成

```
systems/
├── aiwolf-nlp-server/          # ゲームサーバー (Go) ※通常は編集不要
├── aiwolf-nlp-common/          # 共通ライブラリ (Python) ※通常は編集不要
│   └── src/aiwolf_nlp_common/
│       ├── client.py           # WebSocket通信クライアント
│       └── packet/             # データクラス定義
│           ├── packet.py       # Packet (ルートコンテナ)
│           ├── request.py      # Request Enum
│           ├── info.py         # Info (ゲーム状態)
│           ├── setting.py      # Setting (ゲーム設定)
│           ├── talk.py         # Talk (会話エントリ)
│           ├── vote.py         # Vote (投票エントリ)
│           ├── judge.py        # Judge (占い/霊媒結果)
│           ├── role.py         # Role, Team, Species Enum
│           └── status.py       # Status Enum
│
└── aiwolf-nlp-agent-llm/      # ★ エージェント本体 (開発対象)
    └── src/
        ├── main.py             # エントリーポイント
        ├── starter.py          # 接続・セッション管理
        ├── agent/
        │   ├── agent.py        # ★★★ 基底 Agent クラス
        │   ├── villager.py     # 村人
        │   ├── werewolf.py     # 人狼
        │   ├── seer.py         # 占い師
        │   ├── medium.py       # 霊媒師
        │   ├── bodyguard.py    # 騎士
        │   └── possessed.py    # 狂人
        ├── utils/
        │   ├── agent_utils.py  # 役職→クラスのマッピング
        │   ├── agent_logger.py # ログ出力
        │   └── stoppable_thread.py # タイムアウト対応スレッド
        └── config/
            ├── config.yml      # ★★ エージェント設定 (プロンプト含む)
            ├── .env            # APIキー
            ├── config.jp.yml.example  # 日本語設定テンプレート
            └── config.en.yml.example  # 英語設定テンプレート
```

**凡例:** ★★★ = 最重要, ★★ = 重要, ★ = 開発対象

---

## 重要ファイル詳細

### 1. `agent/agent.py` - 基底Agentクラス (★★★ 最重要)

エージェントの全ロジックが集約されたファイル。全役職がこのクラスを継承します。

#### 主要メソッド

| メソッド | 説明 | 戻り値 | カスタマイズ頻度 |
|---|---|---|---|
| `__init__(config, name, game_id, role)` | 初期化。内部状態をセットアップ | - | 中 |
| `set_packet(packet)` | サーバーからのパケットを内部状態に反映 | `None` | 低 |
| `action()` | メインディスパッチャー。requestに応じたメソッドを呼ぶ | `str \| None` | 低 |
| `name()` | NAMEリクエストに応答 | `str` | 低 |
| `initialize()` | ゲーム開始時の初期化処理 | `None` | 中 |
| `daily_initialize()` | 毎日の開始処理 | `None` | 中 |
| **`talk()`** | **トーク発言を生成** | **`str`** | **高** |
| **`whisper()`** | **囁き発言を生成 (人狼のみ)** | **`str`** | **高** |
| **`vote()`** | **投票先を決定** | **`str`** | **高** |
| **`divine()`** | **占い対象を決定 (占い師のみ)** | **`str`** | **高** |
| **`guard()`** | **護衛対象を決定 (騎士のみ)** | **`str`** | **高** |
| **`attack()`** | **襲撃対象を決定 (人狼のみ)** | **`str`** | **高** |
| `daily_finish()` | 毎日の終了処理 | `None` | 中 |
| `finish()` | ゲーム終了処理 | `None` | 低 |
| `_send_message_to_llm(request)` | LLMにプロンプトを送信して応答を取得 | `str \| None` | 高 |
| `get_alive_agents()` | 生存エージェントのリストを取得 | `list[str]` | 低 |

#### `_send_message_to_llm()` の処理フロー

```python
def _send_message_to_llm(self, request):
    # 1. configからプロンプトテンプレートを取得
    prompt_template = self.config["prompt"][request.lower()]
    
    # 2. Jinja2でテンプレートをレンダリング
    #    (info, setting, talk_history等の変数を注入)
    rendered = Template(prompt_template).render(...)
    
    # 3. LLMの会話履歴に追加
    self.llm_message_history.append(HumanMessage(content=rendered))
    
    # 4. LLMに送信して応答取得
    response = (self.llm_model | StrOutputParser()).invoke(self.llm_message_history)
    
    # 5. 応答を履歴に保存
    self.llm_message_history.append(AIMessage(content=response))
    
    return response
```

#### `talk()` の実装例

```python
def talk(self) -> str:
    response = self._send_message_to_llm(self.request)
    self.sent_talk_count = len(self.talk_history)
    return response or ""
```

#### `vote()` の実装例

```python
def vote(self) -> str:
    response = self._send_message_to_llm(self.request)
    # LLMの応答がエージェント名でなければランダム選択にフォールバック
    return response or random.choice(self.get_alive_agents())
```

---

### 2. `config/config.yml` - エージェント設定 (★★)

エージェントの振る舞いを決める設定ファイル。**プロンプトもここに記述**します。

#### 構造

```yaml
# ===== 接続設定 =====
web_socket:
  url: ws://127.0.0.1:8080/ws    # サーバーURL
  token: null                     # 認証トークン (オプション)
  auto_reconnect: false           # 自動再接続

# ===== エージェント設定 =====
agent:
  num: 5                          # 起動するエージェント数
  team: kanolab                   # チーム名
  kill_on_timeout: true           # タイムアウト時にスレッド強制終了

# ===== LLM設定 =====
llm:
  type: google                    # 'openai', 'google', 'ollama'
  sleep_time: 3                   # API呼び出し間隔 (秒)

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

# ===== ★ プロンプト設定 (最も重要) =====
prompt:
  initialize: |
    あなたは人狼ゲームのエージェントです。
    ...
  daily_initialize: |
    ...
  talk: |
    ...
  whisper: |
    ...
  vote: |
    ...
  divine: |
    ...
  guard: |
    ...
  attack: |
    ...
  daily_finish: |
    ...

# ===== ログ設定 =====
log:
  console_output: true
  file_output: true
  output_dir: ./log
  level: debug
  request:
    name: false
    initialize: false
    talk: true
    whisper: true
    vote: true
    divine: true
    guard: true
    attack: true
    finish: false
```

---

### 3. 役職サブクラス (`agent/*.py`)

各役職はAgentを継承し、役職固有のメソッドだけをオーバーライドします。

| ファイル | クラス | オーバーライドメソッド | ポイント |
|---|---|---|---|
| `villager.py` | `Villager` | なし | 全てベースクラスのまま |
| `werewolf.py` | `Werewolf` | `whisper()`, `attack()` | 囁き・襲撃ロジック |
| `seer.py` | `Seer` | `divine()` | 占い対象選択ロジック |
| `medium.py` | `Medium` | なし | 霊媒結果は`info.medium_result`で自動取得 |
| `bodyguard.py` | `Bodyguard` | `guard()` | 護衛対象選択ロジック |
| `possessed.py` | `Possessed` | なし | 村人と同じインターフェース |

---

### 4. `starter.py` - 接続・セッション管理

エージェントの接続確立からゲーム終了までのセッション管理を担当。

#### 重要関数

| 関数 | 説明 |
|---|---|
| `execute(config_path)` | 設定を読み込み、N個のエージェントプロセスを起動 |
| `connect(config, agent_index)` | 1つのエージェントのWebSocket接続・ゲームセッション管理 |
| `handle_game_session(client, config, name)` | ターンベースのゲームループ |
| `handle_game_session_async(client, config, name)` | フリーフォームモードのゲームループ |

---

### 5. `utils/agent_utils.py` - ユーティリティ

| 関数 | 説明 |
|---|---|
| `init_agent_from_packet(config, name, packet)` | パケットの役職情報から適切なサブクラスを生成 |

```python
def init_agent_from_packet(config, name, packet):
    role = packet.info.role_map[packet.info.agent]
    match role:
        case Role.WEREWOLF:  return Werewolf(config, name, ...)
        case Role.SEER:      return Seer(config, name, ...)
        case Role.BODYGUARD: return Bodyguard(config, name, ...)
        case Role.MEDIUM:    return Medium(config, name, ...)
        case Role.POSSESSED: return Possessed(config, name, ...)
        case _:              return Villager(config, name, ...)
```

---

### 6. Common Library - 参照用

直接編集することは少ないですが、データ構造を理解するために参照します。

| ファイル | 定義内容 | 参照場面 |
|---|---|---|
| `packet/info.py` | `Info` dataclass | エージェントが受け取るゲーム状態 |
| `packet/setting.py` | `Setting` dataclass | ゲーム設定・制限 |
| `packet/talk.py` | `Talk` dataclass | 会話エントリの構造 |
| `packet/role.py` | `Role`, `Team`, `Species` Enum | 役職・チーム・種族 |
| `packet/judge.py` | `Judge` dataclass | 占い/霊媒結果 |
| `client.py` | `Client` class | WebSocket通信 (send/receive) |

---

## サーバー設定ファイル (参考)

`aiwolf-nlp-server/config/` 内の設定ファイル:

| ファイル | 内容 |
|---|---|
| `default_5.yml` | 5人用デフォルト設定 |
| `default_9.yml` | 9人用設定 |
| `default_13.yml` | 13人用設定 |
| `freeform_5.yml` | 5人用フリーフォームモード |

主な設定項目:

```yaml
game:
  agent_count: 5           # プレイヤー数
  vote_visibility: true    # 投票公開
  talk:
    max_count:
      per_agent: 4         # 1人の最大発言数/日
      per_day: 20          # 全体の最大発言数/日
    max_length:
      per_talk: -1         # 1発言の最大文字数 (-1=無制限)
  roles:
    5:                     # 5人の場合の役職配分
      WEREWOLF: 1
      POSSESSED: 1
      SEER: 1
      BODYGUARD: 0
      VILLAGER: 2
      MEDIUM: 0
```
