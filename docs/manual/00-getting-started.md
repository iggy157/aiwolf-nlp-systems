# はじめに - AIWolf NLP エージェント開発マニュアル

## このマニュアルについて

このマニュアルは、AIWolf NLP（人狼知能コンテスト）のエージェント開発を初めて行う方向けのガイドです。
LLMを使った人狼ゲームAIの開発手順を、難易度別に解説します。

## 全体像

AIWolf NLPは、LLM（大規模言語モデル）を使ったエージェントが自然言語で人狼ゲームをプレイするシステムです。

### 3つのコンポーネント

| コンポーネント | 言語 | 役割 |
|---|---|---|
| **aiwolf-nlp-server** | Go | ゲームサーバー。ゲーム進行・ルール管理 |
| **aiwolf-nlp-common** | Python | 共通ライブラリ。通信プロトコル・データクラス |
| **aiwolf-nlp-agent-llm** | Python | エージェント本体。LLMを使った意思決定 |

**開発者が主に編集するのは `aiwolf-nlp-agent-llm` です。**

### 人狼ゲームの役職

| 役職 | チーム | 能力 |
|---|---|---|
| **村人 (VILLAGER)** | 村人 | なし。議論と投票で人狼を見つける |
| **占い師 (SEER)** | 村人 | 毎晩1人を占い、人間か人狼か判定 |
| **騎士 (BODYGUARD)** | 村人 | 毎晩1人を護衛。襲撃から守る |
| **霊媒師 (MEDIUM)** | 村人 | 処刑された人の正体がわかる |
| **人狼 (WEREWOLF)** | 人狼 | 夜に1人を襲撃。人狼同士で囁きあう |
| **狂人 (POSSESSED)** | 人狼 | 特殊能力なし。人狼チームの勝利を目指す |

## クイックスタート

### 前提条件

- Python 3.11+
- Go 1.21+ (サーバー起動用)
- LLM API キー (OpenAI, Google, または Ollama)

### 手順

#### 1. 環境構築

```bash
# 共通ライブラリのインストール
cd aiwolf-nlp-common
pip install -e .

# エージェントの依存関係インストール
cd ../aiwolf-nlp-agent-llm
pip install -e .

# 環境変数の設定
cp src/config/.env.example src/config/.env
# .env に API キーを記入
```

#### 2. 設定ファイルの準備

```bash
# エージェント設定
cp src/config/config.jp.yml.example src/config/config.yml
# config.yml を編集 (LLMの種類、モデル名など)
```

#### 3. サーバー起動

```bash
cd ../aiwolf-nlp-server
go run . -c ./config/default_5.yml
```

#### 4. エージェント起動

```bash
cd ../aiwolf-nlp-agent-llm
python src/main.py -c src/config/config.yml
```

5人のエージェントが接続され、自動的にゲームが開始されます。

## ドキュメント構成

| ファイル | 内容 |
|---|---|
| [00-getting-started.md](./00-getting-started.md) | このファイル。概要とクイックスタート |
| [01-available-data.md](./01-available-data.md) | エージェントが使える変数・データ一覧 |
| [02-key-files.md](./02-key-files.md) | 重要ファイル・関数の解説 |
| [03-development-guide.md](./03-development-guide.md) | 難易度別の開発ガイド |
| [../architecture/](../architecture/) | Mermaidアーキテクチャ図 |

## 次のステップ

まずは [01-available-data.md](./01-available-data.md) でエージェントが利用できるデータを把握し、
[03-development-guide.md](./03-development-guide.md) の難易度0（プロンプト改善）から始めることをおすすめします。
