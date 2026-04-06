# 使用可能なデータ・変数リファレンス

エージェントが各リクエスト時にアクセスできるデータの一覧です。
これらは主にプロンプトテンプレート（Jinja2）内で使用します。

## テンプレート変数一覧

### `info` - ゲーム状態

サーバーから毎回送られるゲームの現在状態です。

| 変数 | 型 | 説明 | 使用例 |
|---|---|---|---|
| `info.game_id` | `str` | ゲームの一意識別子 | `"01JQXYZ..."` |
| `info.day` | `int` | 現在の日数 (1から開始) | `1`, `2`, `3` |
| `info.agent` | `str` | 自分のエージェント名 | `"Agent[01]"` |
| `info.profile` | `str \| None` | キャラクター設定 (名前・性格等) | `"ミナト (10歳, 男性...)"` |
| `info.status_map` | `dict[str, Status]` | 全エージェントの生死状況 | `{"Agent[01]": "ALIVE", "Agent[02]": "DEAD"}` |
| `info.role_map` | `dict[str, Role]` | 役職マップ (**自分の役職のみ**) | `{"Agent[01]": "SEER"}` |
| `info.divine_result` | `Judge \| None` | 最新の占い結果 (**占い師のみ**) | 下記参照 |
| `info.medium_result` | `Judge \| None` | 最新の霊媒結果 (**霊媒師のみ**) | 下記参照 |
| `info.executed_agent` | `str \| None` | 前日の処刑者 | `"Agent[03]"` |
| `info.attacked_agent` | `str \| None` | 前夜の襲撃被害者 | `"Agent[05]"` |
| `info.vote_list` | `list[Vote] \| None` | 前日の投票リスト | 下記参照 |
| `info.attack_vote_list` | `list[Vote] \| None` | 襲撃投票リスト (**人狼のみ**) | 下記参照 |
| `info.remain_count` | `int \| None` | 残り発言回数 | `20` |
| `info.remain_length` | `int \| None` | 残り文字数 | `500` |
| `info.remain_skip` | `int \| None` | 残りスキップ回数 | `0` |

### `setting` - ゲーム設定

ゲーム開始時に一度だけ送られるルール情報です。

| 変数 | 型 | 説明 |
|---|---|---|
| `setting.agent_count` | `int` | 参加エージェント数 |
| `setting.max_day` | `int \| None` | 最大日数 (-1 = 無制限) |
| `setting.role_num_map` | `dict[Role, int]` | 各役職の人数 |
| `setting.vote_visibility` | `bool` | 投票が公開されるか |
| `setting.talk.max_count.per_agent` | `int` | 1日あたりの1人の最大発言数 |
| `setting.talk.max_count.per_day` | `int` | 1日あたりの全体最大発言数 |
| `setting.talk.max_length.per_talk` | `int` | 1発言の最大文字数 |
| `setting.whisper.*` | | 囁きの制限 (talkと同構造) |
| `setting.timeout.action` | `int` | アクションタイムアウト (ms) |
| `setting.timeout.response` | `int` | レスポンスタイムアウト (ms) |

### `role` - 自分の役職

| 変数 | 型 | 説明 |
|---|---|---|
| `role` | `Role` | 役職 Enum |
| `role.value` | `str` | 役職名の文字列 (`"WEREWOLF"`, `"SEER"` 等) |
| `role.team` | `Team` | 所属チーム (`VILLAGER` or `WEREWOLF`) |
| `role.species` | `Species` | 種族 (`HUMAN` or `WEREWOLF`) |

### `talk_history` / `whisper_history` - 会話履歴

| 変数 | 型 | 説明 |
|---|---|---|
| `talk_history` | `list[Talk]` | 全トーク履歴 (累積) |
| `whisper_history` | `list[Talk]` | 全囁き履歴 (人狼のみ, 累積) |
| `sent_talk_count` | `int` | 前回までに処理済みのトーク数 |
| `sent_whisper_count` | `int` | 前回までに処理済みの囁き数 |

**Talk オブジェクトの構造:**

| フィールド | 型 | 説明 |
|---|---|---|
| `talk.idx` | `int` | 通し番号 |
| `talk.day` | `int` | 発言日 |
| `talk.turn` | `int` | 発言ターン |
| `talk.agent` | `str` | 発言者 |
| `talk.text` | `str` | 発言内容 |
| `talk.skip` | `bool` | スキップかどうか |
| `talk.over` | `bool` | 終了宣言かどうか |

### サブオブジェクト

**Judge (占い/霊媒結果):**

```python
judge.day      # 占った/判定した日
judge.agent    # 占い師/霊媒師のエージェント名
judge.target   # 対象エージェント名
judge.result   # "HUMAN" or "WEREWOLF"
```

**Vote (投票):**

```python
vote.day       # 投票日
vote.agent     # 投票者
vote.target    # 投票先
```

## 役職別のアクセス可能データ

| データ | 村人 | 占い師 | 騎士 | 霊媒師 | 人狼 | 狂人 |
|---|---|---|---|---|---|---|
| info.status_map | o | o | o | o | o | o |
| info.role_map (自分のみ) | o | o | o | o | o | o |
| info.divine_result | - | **o** | - | - | - | - |
| info.medium_result | - | - | - | **o** | - | - |
| info.executed_agent | o | o | o | o | o | o |
| info.attacked_agent | o | o | o | o | o | o |
| info.vote_list | o | o | o | o | o | o |
| info.attack_vote_list | - | - | - | - | **o** | - |
| talk_history | o | o | o | o | o | o |
| whisper_history | - | - | - | - | **o** | - |
| DIVINE リクエスト | - | **o** | - | - | - | - |
| GUARD リクエスト | - | - | **o** | - | - | - |
| ATTACK リクエスト | - | - | - | - | **o** | - |
| WHISPER リクエスト | - | - | - | - | **o** | - |

## プロンプトテンプレートでの使い方

```jinja2
{# 基本情報 #}
あなたは{{ info.agent }}です。役職は{{ role.value }}です。
現在{{ info.day }}日目です。

{# 生存者リスト #}
生存中のエージェント:
{% for agent, status in info.status_map.items() %}
{% if status == 'ALIVE' %}
- {{ agent }}
{% endif %}
{% endfor %}

{# 占い結果 (占い師のみ) #}
{% if info.divine_result %}
占い結果: {{ info.divine_result.target }}は{{ info.divine_result.result }}でした。
{% endif %}

{# 新しい会話のみ表示 #}
最新の会話:
{% for talk in talk_history[sent_talk_count:] %}
{{ talk.agent }}: {{ talk.text }}
{% endfor %}

{# 投票フェーズで生存者を選択肢として提示 #}
以下から投票先を1人選んでください:
{% for agent, status in info.status_map.items() %}
{% if status == 'ALIVE' %}
{{ agent }}
{% endif %}
{% endfor %}
```

## 応答形式

| リクエスト | 応答内容 | 例 |
|---|---|---|
| NAME | エージェント名 (文字列) | `"kanolab1"` |
| TALK | 自然言語の発言 or `"Over"` | `"Agent[03]が怪しいです"` |
| WHISPER | 自然言語の囁き or `"Over"` | `"Agent[02]を襲撃しましょう"` |
| VOTE | エージェント名 | `"Agent[03]"` |
| DIVINE | エージェント名 | `"Agent[02]"` |
| GUARD | エージェント名 | `"Agent[04]"` |
| ATTACK | エージェント名 | `"Agent[02]"` |
| INITIALIZE | 応答不要 | - |
| DAILY_INITIALIZE | 応答不要 | - |
| DAILY_FINISH | 応答不要 | - |
| FINISH | 応答不要 | - |
