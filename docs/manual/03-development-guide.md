# 開発ガイド - 難易度別アプローチ

エージェント開発の取り組み方を、難易度0（最も簡単）から難易度4（上級）まで段階的に解説します。
初学者は難易度0から始め、徐々にステップアップすることを推奨します。

---

## 難易度0: プロンプト改善 (コード変更なし)

**対象ファイル:** `config/config.yml` の `prompt` セクションのみ

**概要:** コードを一切変更せず、LLMに渡すプロンプトを改善するだけで強いエージェントを作ります。最も手軽で効果的なアプローチです。

### やること

`config.yml` の `prompt:` セクション内のテンプレートを編集します。

### 改善ポイント

#### A. 役職ごとの戦略を明示する

```yaml
# Before (弱い例)
prompt:
  talk: |
    トークしてください。

# After (改善例)
prompt:
  talk: |
    あなたは{{ info.agent }}（役職: {{ role.value }}）です。{{ info.day }}日目のトークフェーズです。
    
    【戦略ガイドライン】
    {% if role.value == 'SEER' %}
    - あなたは占い師です。占い結果を適切なタイミングで公開してください。
    - ただし初日にすぐカミングアウトするかは状況次第です。
    {% elif role.value == 'WEREWOLF' %}
    - あなたは人狼です。村人のふりをしてください。
    - 占い師を騙ることも有効な戦略です。
    {% elif role.value == 'POSSESSED' %}
    - あなたは狂人です。人狼チームの勝利を目指してください。
    - 偽の占い師として振る舞うのが効果的です。
    {% endif %}
    
    最新の会話:
    {% for talk in talk_history[sent_talk_count:] %}
    {{ talk.agent }}: {{ talk.text }}
    {% endfor %}
    
    自然な日本語で発言してください。発言のみを出力してください。
```

#### B. 投票ロジックを改善する

```yaml
prompt:
  vote: |
    {{ info.day }}日目の投票フェーズです。以下の情報をもとに投票先を決めてください。
    
    【本日の議論まとめ】
    {% for talk in talk_history %}
    {% if talk.day == info.day %}
    {{ talk.agent }}: {{ talk.text }}
    {% endif %}
    {% endfor %}
    
    {% if info.vote_list %}
    【前回の投票結果】
    {% for vote in info.vote_list %}
    {{ vote.agent }} → {{ vote.target }}
    {% endfor %}
    {% endif %}
    
    【生存中のエージェント】
    {% for agent, status in info.status_map.items() %}
    {% if status == 'ALIVE' and agent != info.agent %}
    - {{ agent }}
    {% endif %}
    {% endfor %}
    
    最も人狼の可能性が高いエージェント名だけを出力してください（例: Agent[03]）。
```

#### C. 占い結果・霊媒結果を活用する

```yaml
prompt:
  daily_initialize: |
    {{ info.day }}日目が始まりました。
    
    {% if info.divine_result %}
    【占い結果】{{ info.divine_result.target }}を占った結果: {{ info.divine_result.result }}
    {% endif %}
    
    {% if info.medium_result %}
    【霊媒結果】処刑された{{ info.medium_result.target }}の正体: {{ info.medium_result.result }}
    {% endif %}
    
    {% if info.executed_agent %}
    【処刑】{{ info.executed_agent }}が処刑されました。
    {% endif %}
    
    {% if info.attacked_agent %}
    【襲撃】{{ info.attacked_agent }}が襲撃されました。
    {% endif %}
    
    この情報を今後の判断に活用してください。
```

### Tips

- プロンプトの末尾に「発言のみを出力してください」等の制約を入れると余計な出力を抑えられる
- `role.value` で条件分岐することで、1つのテンプレートで全役職に対応できる
- `info.remain_count` を使って「残り発言数が少ないので重要なことだけ話してください」といった指示が可能

---

## 難易度1: LLM設定の調整

**対象ファイル:** `config/config.yml` の LLM 設定セクション

### やること

- モデルの変更 (例: `gpt-4o-mini` → `gpt-4o`)
- `temperature` の調整 (低い=安定、高い=創造的)
- `sleep_time` の調整

```yaml
# より賢いモデルに変更
openai:
  model: gpt-4o
  temperature: 0.3    # 低めで安定した判断

# または Google
google:
  model: gemini-2.0-flash
  temperature: 0.5
```

### ポイント

- 高性能モデルほど戦略的な判断ができるが、レスポンスが遅い場合がある
- `timeout.action` (サーバー設定) 内に応答する必要があるため、速度とのバランスが重要
- `sleep_time` はAPIレート制限を避けるための待機時間

---

## 難易度2: 役職サブクラスのカスタマイズ

**対象ファイル:** `agent/villager.py`, `agent/werewolf.py` 等

### やること

各役職のサブクラスにロジックを追加します。基底クラスのメソッドをオーバーライドして、役職固有の意思決定を強化します。

### 例: 占い師の占い対象選択を改善

```python
# agent/seer.py
class Seer(Agent):
    def __init__(self, config, name, game_id, role):
        super().__init__(config, name, game_id, Role.SEER)
        self.divine_results = {}  # 占い済みの結果を記録
    
    def daily_initialize(self):
        """占い結果を蓄積"""
        super().daily_initialize()
        if self.info.divine_result:
            target = self.info.divine_result.target
            result = self.info.divine_result.result
            self.divine_results[target] = result
    
    def divine(self) -> str:
        """まだ占っていない生存者を優先的に占う"""
        alive = self.get_alive_agents()
        # 自分と占い済みを除外
        not_divined = [a for a in alive 
                       if a != self.info.agent 
                       and a not in self.divine_results]
        
        if not_divined:
            # LLMに未占い者から選ばせる
            # (プロンプトに占い済み情報を追加するのも有効)
            response = self._send_message_to_llm(self.request)
            if response and response.strip() in not_divined:
                return response.strip()
            return random.choice(not_divined)
        
        return super().divine()
```

### 例: 人狼の囁きで戦略共有

```python
# agent/werewolf.py
class Werewolf(Agent):
    def __init__(self, config, name, game_id, role):
        super().__init__(config, name, game_id, Role.WEREWOLF)
        self.suspected_seer = None  # 占い師と疑うエージェント
    
    def whisper(self) -> str:
        """他の人狼と戦略を共有"""
        response = self._send_message_to_llm(self.request)
        return response or "Over"
    
    def attack(self) -> str:
        """占い師と思しきエージェントを優先的に襲撃"""
        response = self._send_message_to_llm(self.request)
        if response and response.strip() in self.get_alive_agents():
            return response.strip()
        return random.choice(self.get_alive_agents())
```

---

## 難易度3: 基底クラスの改造・新機能追加

**対象ファイル:** `agent/agent.py`, 新規ユーティリティファイル

### アイデア一覧

#### A. 会話分析モジュールの追加

```python
# utils/conversation_analyzer.py (新規作成)
class ConversationAnalyzer:
    """会話履歴を分析して各エージェントの怪しさスコアを算出"""
    
    def __init__(self):
        self.suspicion_scores = {}  # {agent: score}
    
    def analyze_talks(self, talk_history, my_agent):
        """発言パターンから怪しさを分析"""
        for talk in talk_history:
            agent = talk.agent
            text = talk.text
            if agent == my_agent:
                continue
            
            # 例: 他人を強く攻撃するエージェントは怪しい
            # 例: 発言が少なすぎるエージェントは怪しい
            # ... 分析ロジック
        
        return self.suspicion_scores
```

#### B. 投票パターン分析

```python
# agent/agent.py に追加
class Agent:
    def __init__(self, ...):
        ...
        self.vote_history = {}  # {day: {agent: target}}
    
    def daily_initialize(self):
        """投票履歴を蓄積"""
        if self.info.vote_list:
            day_votes = {}
            for vote in self.info.vote_list:
                day_votes[vote.agent] = vote.target
            self.vote_history[self.info.day - 1] = day_votes
    
    def get_vote_pattern_summary(self):
        """投票パターンの要約を生成 (プロンプトに注入可能)"""
        summary = []
        for day, votes in self.vote_history.items():
            for agent, target in votes.items():
                summary.append(f"Day{day}: {agent} → {target}")
        return "\n".join(summary)
```

#### C. LLMの会話履歴管理の改善

```python
# agent/agent.py の _send_message_to_llm を改造
def _send_message_to_llm(self, request):
    # 会話履歴が長くなりすぎたら要約する
    if len(self.llm_message_history) > 20:
        self._summarize_history()
    
    # 通常の処理...
    ...

def _summarize_history(self):
    """過去の会話履歴をLLMで要約して圧縮"""
    summary_prompt = "以下の会話を簡潔に要約してください:\n"
    for msg in self.llm_message_history[:-5]:  # 最新5件は保持
        summary_prompt += f"{msg.content}\n"
    
    # 要約をLLMに依頼...
    # self.llm_message_history を要約で置換
```

#### D. プロンプトテンプレートの動的生成

```python
# Jinja2テンプレートに追加の変数を注入
def _send_message_to_llm(self, request):
    prompt_template = self.config["prompt"][request.lower()]
    template = Template(prompt_template)
    
    # 追加の分析データを注入
    rendered = template.render(
        info=self.info,
        setting=self.setting,
        talk_history=self.talk_history,
        whisper_history=self.whisper_history,
        role=self.role,
        sent_talk_count=self.sent_talk_count,
        sent_whisper_count=self.sent_whisper_count,
        # ↓ 追加データ
        suspicion_scores=self.analyzer.get_scores(),
        vote_patterns=self.get_vote_pattern_summary(),
        alive_count=len(self.get_alive_agents()),
    )
    ...
```

---

## 難易度4: システムレベルの改造

**対象ファイル:** `starter.py`, `agent/agent.py`, 新規モジュール群

### アイデア一覧

#### A. マルチエージェント協調 (同じチーム間で情報共有)

```python
# utils/shared_memory.py (新規)
import multiprocessing

class SharedMemory:
    """プロセス間で共有するゲーム分析結果"""
    def __init__(self):
        self.manager = multiprocessing.Manager()
        self.data = self.manager.dict()
    
    def update(self, agent_name, key, value):
        self.data[f"{agent_name}:{key}"] = value
    
    def get(self, agent_name, key):
        return self.data.get(f"{agent_name}:{key}")
```

#### B. 応答のバリデーション・後処理

```python
# utils/response_validator.py (新規)
class ResponseValidator:
    """LLM応答をゲームプロトコルに適合させる"""
    
    @staticmethod
    def validate_vote(response, alive_agents):
        """投票先がエージェント名として有効か検証"""
        response = response.strip()
        # Agent[XX] 形式を抽出
        import re
        match = re.search(r'Agent\[\d+\]', response)
        if match and match.group() in alive_agents:
            return match.group()
        return random.choice(alive_agents)
    
    @staticmethod
    def validate_talk(response, max_length=None):
        """発言が制限内に収まるか検証"""
        if max_length and len(response) > max_length:
            return response[:max_length]
        return response
```

#### C. ゲームログ分析・学習パイプライン

```python
# utils/game_analyzer.py (新規)
class GameAnalyzer:
    """過去のゲームログから勝率パターンを分析"""
    
    def analyze_logs(self, log_dir):
        """ログを読んで戦略の成功率を計算"""
        ...
    
    def suggest_strategy(self, role, day, context):
        """過去のデータに基づいて戦略を提案"""
        ...
```

#### D. 独自のLLM呼び出しパイプライン

LangChainに頼らず、直接API呼び出しやRAGを構築する。

```python
# agent/agent.py を改造
def _send_message_to_llm(self, request):
    # LangChainの代わりに直接API呼び出し
    # Function CallingやStructured Outputを活用
    # RAGで人狼戦略データベースを参照
    ...
```

---

## 開発の進め方まとめ

```
難易度0 (コード変更なし)
  └─ config.yml のプロンプト改善
       ↓
難易度1 (設定変更のみ)
  └─ LLMモデル・パラメータ変更
       ↓
難易度2 (サブクラス編集)
  └─ 役職固有のロジック追加
       ↓
難易度3 (基底クラス改造)
  └─ 分析モジュール追加、プロンプト動的生成
       ↓
難易度4 (システム改造)
  └─ マルチエージェント協調、独自パイプライン
```

### 共通のコツ

1. **ログを活用する**: `log.request.talk: true` にして会話ログを確認。LLMが何を返しているか把握する
2. **小さく始める**: まずはプロンプト改善で効果を確認してからコードを触る
3. **フォールバックを意識する**: LLMが想定外の応答を返した場合のハンドリングを常に考える
4. **タイムアウトに注意**: サーバーの `timeout.action` 内に応答しないとエラー扱いになる
5. **テスト実行**: 5人ゲームで動作確認してから設定を変更する
6. **差分送信を理解する**: `talk_history` は差分で送られ、エージェント側で累積。`sent_talk_count` でオフセット管理

### デバッグのやり方

```bash
# エージェントのログを確認
tail -f aiwolf-nlp-agent-llm/log/*/kanolab1.log

# サーバーのログを確認
ls aiwolf-nlp-server/log/json/     # JSONログ
ls aiwolf-nlp-server/log/game/     # ゲームログ

# 特定のリクエスト/レスポンスだけログに出す
# config.yml の log.request セクションで制御
```
