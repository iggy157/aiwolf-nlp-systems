# エージェント内部アーキテクチャ

> このドキュメントでは、LLMエージェントの内部構造を説明します。エージェントがサーバからリクエストを受け取り、LLMに問い合わせ、応答を返すまでの流れを中心に解説します。

## エージェントの全体像

エージェントは「人狼ゲームのプレイヤー」です。自分では考えず、LLM（ChatGPTやGeminiなど）に状況を伝えて「何を発言すべきか」「誰に投票すべきか」を考えさせます。

エージェントの主な仕事は以下の3つです:
1. **サーバからの指示を受け取る** — 「発言して」「投票して」などのリクエスト
2. **LLMに状況を伝えて判断を仰ぐ** — ゲーム状態・会話履歴をプロンプトにまとめてLLM APIを呼び出す
3. **LLMの応答をサーバに返す** — 発言テキストや投票先をWebSocketで送信

## クラス階層

全ての役職は共通の `Agent` ベースクラスを継承しています。ベースクラスにLLM呼び出し・状態管理などの中核ロジックが集約されており、各役職クラスはその役職だけが使う特殊なアクション（占い・護衛・襲撃など）のメソッドをオーバーライドしています。

```mermaid
classDiagram
    class Agent {
        ゲーム状態やLLM接続を管理する共通基盤
        --
        サーバとの通信で受け取った情報
        info: ゲーム状態（日数・生死・結果など）
        setting: ゲーム設定（人数・制限など）
        talk_history: 全議論の会話履歴
        whisper_history: 人狼同士の密談履歴
        role: 自分の役職
        --
        LLM関連
        llm_model: LLMへの接続（OpenAI/Google/Ollama）
        llm_message_history: LLMとの全会話履歴
        --
        主要メソッド
        set_packet() パケットから内部状態を更新
        action() リクエスト種別に応じて処理を分岐
        _send_message_to_llm() プロンプト生成→LLM呼出→応答取得
        talk() / vote() / divine() etc. 各アクション
    }

    class Villager {
        村人：特殊能力なし
        ベースクラスの talk() と vote() をそのまま使用
    }

    class Werewolf {
        人狼：仲間との密談＋襲撃
        whisper() 人狼同士で相談
        attack() 襲撃対象を選ぶ
    }

    class Seer {
        占い師：毎晩1人を占う
        divine() 占い対象を選ぶ
    }

    class Medium {
        霊媒師：追放者の正体を知る
        info.medium_result で結果を受け取る
    }

    class Bodyguard {
        騎士：毎晩1人を護衛
        guard() 護衛対象を選ぶ
    }

    class Possessed {
        狂人：人狼チームだが能力なし
        村人と同じインターフェース
    }

    Agent <|-- Villager
    Agent <|-- Werewolf
    Agent <|-- Seer
    Agent <|-- Medium
    Agent <|-- Bodyguard
    Agent <|-- Possessed
```

## データの流れ：サーバ → エージェント → LLM → サーバ

以下の図は、サーバからパケットを受信してからLLMに問い合わせて応答を返すまでの、エージェント内部のデータの流れを示しています。

**ポイント:**
- サーバからは毎回「差分」だけが送られてくる（新しい発言のみ）
- エージェント側で過去の履歴と結合して「全体」を保持する
- LLMには必要な情報だけをテンプレートで整形して送る

```mermaid
flowchart LR
    subgraph Receive["① サーバからパケットを受信"]
        Packet["Packet (JSON形式)"]
        Packet --> Request["request<br/>何をしてほしいか<br/>例: TALK, VOTE"]
        Packet --> InfoData["info<br/>今のゲーム状態<br/>日数・誰が生きてるか・結果等"]
        Packet --> TalkHist["talk_history<br/>新しい発言だけ（差分）<br/>前回送信以降の会話"]
    end

    subgraph Store["② エージェント内部で蓄積"]
        AgentState["エージェントの内部状態"]
        AgentState --> SelfInfo["info<br/>最新のゲーム状態に更新"]
        AgentState --> SelfTalk["talk_history<br/>過去の全発言＋今回の差分<br/>を結合して保持"]
        AgentState --> SelfRole["role<br/>自分の役職<br/>（ゲーム中ずっと同じ）"]
        AgentState --> LLMHist["llm_message_history<br/>LLMとのやり取り全履歴<br/>（文脈保持のため）"]
    end

    subgraph Generate["③ プロンプトを生成してLLMに送信"]
        Template["Jinja2テンプレート<br/>設定ファイルに定義された<br/>プロンプトの雛形"]
        Render["テンプレートに情報を埋め込む<br/>「今日は2日目。昨夜の占い結果は…<br/>最近の会話: Agent[03]が怪しいと…」"]
        LLMCall["LLM API呼び出し<br/>過去の会話履歴も含めて送信<br/>→ LLMが発言や判断を生成"]
        Response["応答テキスト<br/>例: 「Agent[03]に投票します」"]

        Template --> Render
        SelfInfo & SelfTalk & SelfRole --> Render
        Render --> LLMCall --> Response
    end

    InfoData --> SelfInfo
    TalkHist -->|"差分を追加"| SelfTalk
    Response -->|"④ サーバに送信"| ServerSend["WebSocketで応答を返す"]

    style Store fill:#e3f2fd,stroke:#1976d2
    style Generate fill:#f3e5f5,stroke:#9c27b0
```

## プロンプトテンプレートの仕組み

LLMに送るプロンプトは、**設定ファイル（YAML）内にJinja2テンプレートとして定義**されています。エージェントは毎回のリクエストに対して、テンプレートにゲーム情報を埋め込んで最終的なプロンプト文字列を生成します。

### テンプレートで使える情報の全体像

```mermaid
mindmap
    root((プロンプトに<br/>埋め込める情報))
        ゲーム状態 (info)
            自分の名前 — info.agent
            現在の日数 — info.day
            キャラ設定 — info.profile
            誰が生きてるか — info.status_map
            自分の役職 — info.role_map
            占い結果 — info.divine_result
            霊媒結果 — info.medium_result
            昨日の追放者 — info.executed_agent
            昨夜の襲撃者 — info.attacked_agent
            残り発言回数 — info.remain_count
        ゲーム設定 (setting)
            参加人数 — setting.agent_count
            役職の配分 — setting.role_num_map
            発言制限 — setting.talk
        自分の役職 (role)
            役職名 — role.value
            所属チーム — role.team
            種族 — role.species
        会話履歴
            全議論の履歴 — talk_history
            前回以降の新着 — talk_history[sent_talk_count:]
            人狼密談の履歴 — whisper_history
```

### テンプレートの具体例

設定ファイルにはこのようなテンプレートが書かれています:

```yaml
prompt:
  # ゲーム開始時：LLMに役職とルールを伝える
  initialize: |
    あなたは人狼ゲームのエージェントです。
    あなたの名前は{{ info.agent }}です。
    あなたの役職は{{ role.value }}です。
    ...

  # 毎朝：前日の結果を伝える
  daily_initialize: |
    {{ info.day }}日目の昼になりました。
    {% if info.divine_result %}占い結果: {{ info.divine_result }}{% endif %}
    {% if info.executed_agent %}昨日の追放: {{ info.executed_agent }}{% endif %}

  # 議論：最近の会話を見せて発言を求める
  talk: |
    発言してください。最近の会話:
    {% for w in talk_history[sent_talk_count:] %}
    {{ w.agent }}: {{ w.text }}
    {% endfor %}

  # 投票：生存者リストを見せて投票先を求める
  vote: |
    誰に投票しますか？ 候補:
    {% for k, v in info.status_map.items() %}
    {% if v == 'ALIVE' %}{{ k }}{% endif %}
    {% endfor %}
```

**`sent_talk_count` の役割**: LLMに送る会話履歴の「どこまで送ったか」を記録するマーカーです。`talk_history[sent_talk_count:]` とすることで、前回のtalk()呼び出し以降に追加された新しい発言だけをLLMに伝えます。これにより、同じ会話を何度も送らずに済みます。

## マルチプロセス構成

エージェントは **複数の独立したプロセス** として起動します。各プロセスが1体のエージェントに対応し、それぞれが独自にサーバとWebSocket接続を持ちます。

```mermaid
flowchart TD
    Main["main.py<br/>メインプロセス<br/>（設定ファイルを読み込む）"]
    Main -->|"プロセス1を起動"| C1["Agent 'kanolab1'<br/>WebSocket接続 → ゲーム参加"]
    Main -->|"プロセス2を起動"| C2["Agent 'kanolab2'"]
    Main -->|"プロセス3を起動"| C3["Agent 'kanolab3'"]
    Main -->|"プロセス4を起動"| C4["Agent 'kanolab4'"]
    Main -->|"プロセス5を起動"| C5["Agent 'kanolab5'"]

    subgraph "各エージェントプロセスの中身"
        direction TB
        WS["WebSocket接続<br/>サーバに接続してパケットを受信"]
        Loop["ゲームセッションループ<br/>パケット受信 → 処理 → 応答<br/>を FINISH が届くまで繰り返す"]
        Thread["タイムアウト制御<br/>LLM呼び出しを別スレッドで実行し<br/>制限時間を超えたら強制停止"]

        WS --> Loop --> Thread
    end

    C1 --> WS
```

## タイムアウト処理

LLM APIの応答が遅い場合に備えて、エージェントにはタイムアウト機構があります。サーバ側にも応答待ちのタイムアウトがあるため、エージェントはそれに間に合うように応答を返す必要があります。

```mermaid
flowchart TD
    Action["action() が呼ばれる"] --> Decorator["@timeout デコレータ<br/>LLM呼び出しを別スレッドで実行"]
    Decorator --> Wait{"サーバ設定のタイムアウト時間内に<br/>LLMから応答が返った？"}

    Wait -->|"時間内に完了"| Success["LLMの応答テキストを取得<br/>→ 通常通りサーバに送信"]
    Wait -->|"タイムアウト"| Kill["LLM呼び出しスレッドを強制停止"]
    Kill --> Fallback["フォールバック処理で応答を生成"]

    Fallback -->|"TALK / WHISPER の場合"| SkipText["空文字 or 'Over'を返す<br/>（発言をスキップ）"]
    Fallback -->|"VOTE / DIVINE /<br/>GUARD / ATTACK の場合"| Random["生存者の中からランダムに<br/>1人を選んで返す"]

    Success --> Send["サーバに応答を送信"]
    SkipText --> Send
    Random --> Send

    style Kill fill:#ffcdd2
    style Fallback fill:#fff9c4
```
