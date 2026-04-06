# AIWolf NLP System

LLMエージェントが自然言語で人狼ゲームをプレイする対戦システムです。

## システム構成

```
systems/
├── aiwolf-nlp-server/      # ゲームサーバー (Go)
├── aiwolf-nlp-common/      # 共通ライブラリ (Python)
├── aiwolf-nlp-agent-llm/   # エージェント本体 (Python)
└── docs/
    ├── architecture/        # アーキテクチャ図
    └── manual/              # 開発マニュアル
```

## セットアップ

### 1. サーバー起動

#### ソースからビルドする場合

```bash
cd aiwolf-nlp-server
go run . -c ./config/default_5.yml
```

#### バイナリをダウンロードする場合

```bash
curl -LO https://github.com/aiwolfdial/aiwolf-nlp-server/releases/latest/download/aiwolf-nlp-server-linux-amd64
curl -LO https://github.com/aiwolfdial/aiwolf-nlp-server/releases/latest/download/default_5.yml
curl -Lo .env https://github.com/aiwolfdial/aiwolf-nlp-server/releases/latest/download/example.env
chmod u+x ./aiwolf-nlp-server-linux-amd64
./aiwolf-nlp-server-linux-amd64 -c ./default_5.yml
```

> 起動する設定ファイルによってゲームの種類（人数・役職構成・通信モード等）が変わります。
> `default_5.yml`(5人), `default_9.yml`(9人), `default_13.yml`(13人), `freeform_5.yml`(フリーフォーム) などがあります。

### 2. エージェント起動

```bash
cd aiwolf-nlp-agent-llm

# 設定ファイルの準備
cp config/.env.example config/.env
cp config/config.jp.yml.example config/config.yml

# .env に LLM の API キーを設定
vi config/.env

# config.yml の設定をサーバーの設定ファイルに合わせて編集
vi config/config.yml

# 環境構築
python -m venv .venv
source .venv/bin/activate
pip install -e .

# 起動
python src/main.py
```

サーバーとエージェントの設定により、自己対戦も他者対戦も可能です。

## ドキュメント

- [アーキテクチャ図](docs/architecture/) - システム構成・ゲームフロー・エージェント内部構造の Mermaid 図
- [開発マニュアル](docs/manual/00-getting-started.md) - エージェント開発の始め方から難易度別ガイドまで
