# Building AI（enexus-agent）実装仕様

> 本ファイルは仕様の親文書。各節は別ファイルに分割している。
> 【未確定】と記した項目は推測で実装せず、ユーザに確認するか環境変数で外出しにすること。
> 背景・狙いは [concept.md](./concept.md) を参照。

## 目次

| 節 | ファイル | 内容 |
|---|---|---|
| 1 | （本ファイル） | システム構成・技術スタック・リポジトリ構成・フェーズ計画・環境変数 |
| 2 | [spec_datasource.md](./spec_datasource.md) | Elasticsearch（センサーデータ・機器台帳） |
| 3 | [spec_api.md](./spec_api.md) | e-Nexus 設備制御 API v1、基幹 API v0（暫定）、ガードレール、BEMS との関係 |
| 4 | [spec_llm.md](./spec_llm.md) | 学内 LLM（Qwen 3.5 / vLLM）、エージェント構成 |
| 5 | spec_policy.md（未作成） | ポリシーの構造化表現、ポリシーエンジン、リプレイ検証 |
| 6 | spec_ui.md（未作成） | 画面構成、妖怪キャラクター、チャット UI |

5・6 はフェーズ1完了後に作成する。

---

## 1. システム構成

### 1.1 全体図

```
                ┌─────────────────────────────────────────────────────┐
                │  enexus-agent (VM#2, Ubuntu 24.04, Docker Compose)   │
                │                                                       │
 Browser ──────▶│  frontend (React/Vite, nginx :80)                    │
                │      │ REST / WebSocket                              │
                │  backend (FastAPI :8000)                             │
                │   ├ agents/        3エージェント会話 (LangGraph)       │
                │   ├ policy/        ポリシーエンジン・リプレイ検証        │
                │   ├ control/       Guardrail → DeviceController      │
                │   ├ datasource/    ES クライアント・ポイント台帳        │
                │   └ llm/           vLLM アダプタ                       │
                │      │                                                │
                │  postgres :5432   (ポリシー履歴・承認ログ・制御監査ログ) │
                └──────┬──────────────────┬──────────────────┬─────────┘
                       │ https             │ http              │ http
                       ▼                   ▼                   ▼
          Elasticsearch 8.11         e-Nexus API v1        vLLM (Qwen3.5)
          elastic.i4s...:9200        openapi.i4s...        genai.i4s...:8000
          （読み取りのみ）            （GET / PUT）          （OpenAI 互換）
```

開発は VM#1（ai-connect）から `ssh enexus-agent` 経由、または enexus-agent 上の Claude Code で行う。

### 1.2 技術スタック（確定）

| 層 | 採用 | 理由 |
|---|---|---|
| バックエンド | Python 3.12 / FastAPI / uv | LLM・ES 周辺ライブラリが Python に集中 |
| エージェント | LangGraph + `openai` SDK | 会話→承認待ち→ポリシー生成の状態遷移を明示的に書ける |
| ポリシーエンジン | 自作（Pydantic モデル + 評価器） | 依存を増やさず、図示・リプレイと同じ型を共有するため |
| フロントエンド | React 18 / Vite / TypeScript / Tailwind | 単一画面ダッシュボードに十分。キャラクターは SVG |
| DB | PostgreSQL 16 | ポリシー履歴・承認・監査ログ。JSONB でポリシー本体を保存 |
| リアルタイム通知 | WebSocket（FastAPI） | エージェント会話のストリーミング表示 |
| 実行環境 | Docker Compose | backend / frontend / postgres の3サービス |
| テスト | pytest + Vitest | 制御レイヤとガードレールは必ずユニットテストを書く |

### 1.3 リポジトリ構成

```
building-ai/
├── CLAUDE.md
├── docs/
│   ├── concept.md
│   ├── spec.md                 ← 本ファイル
│   ├── spec_datasource.md
│   ├── spec_api.md
│   ├── spec_llm.md
│   └── api/
│       ├── e11-v1.json         ← e-Nexus API v1（本命）
│       └── e11-v0.json         ← 基幹 API v0（暫定バックエンド）
├── config/
│   ├── areas.yaml              ← エリアコード → 日本語名
│   ├── enums.yaml              ← API enum ↔ シンボル
│   ├── guardrails.yaml         ← 値域・変化量上限・レート制限・安全状態
│   └── agents.yaml             ← 3エージェントの名前・性格・担当・参照データ
├── data/
│   └── points.csv              ← 自動生成のポイント台帳（scripts/build_points.py）
├── backend/
│   ├── pyproject.toml
│   ├── app/
│   │   ├── main.py
│   │   ├── datasource/         ← es_client.py, points.py, power.py
│   │   ├── control/            ← device_controller.py, guardrail.py, backends/{base,v1,v0,mock}.py
│   │   ├── llm/                ← qwen_adapter.py
│   │   ├── agents/             ← graph.py, prompts/
│   │   ├── policy/             ← model.py, engine.py, replay.py
│   │   ├── api/                ← FastAPI ルータ
│   │   └── db/                 ← SQLAlchemy モデル, Alembic
│   └── tests/
├── frontend/
│   ├── package.json
│   └── src/
├── scripts/
│   ├── inspect_es.py           ← ES の未確定項目を調査
│   ├── build_points.py         ← ポイント台帳生成
│   ├── check_api.py            ← e-Nexus API 疎通・ulid 突合
│   └── check_llm.py            ← tool call / thinking / JSON 出力の検証
├── infra/
│   └── enexus-agent/setup.sh   ← VM#2 初期セットアップ
├── docker-compose.yml
├── .env.example
└── .gitignore                  ← .env, data/*.csv 以外の生成物
```

### 1.4 環境変数（`.env.example`）

```
# Elasticsearch（spec_datasource.md 2.1）
ES_URL=https://elastic.i4s.uec.ac.jp:9200
ES_USER=
ES_PASSWORD=
ES_CA_CERT=                     # 空なら ES_VERIFY_CERTS に従う
ES_VERIFY_CERTS=false           # 開発時のみ false

# 設備制御バックエンド（spec_api.md 3.6 / 3.11）
DEVICE_BACKEND=v0               # v1 | v0 | mock。v1 復旧まで v0

# e-Nexus API v1（spec_api.md 3.1）
E11_API_URL=http://openapi.i4s.uec.ac.jp/api/v1/e11
E11_API_KEY_READ=
E11_API_KEY_WRITE=              # DRY_RUN では空のまま

# 基幹 API v0（spec_api.md 3.11）
E11_V0_API_URL=http://192.168.13.10:8080   # 認証なし。パスは /webapi/...

# LLM（spec_llm.md 4.1）
LLM_BASE_URL=http://genai.i4s.uec.ac.jp:8000/v1
LLM_API_KEY=
LLM_MODEL=Qwen/Qwen3.5-122B-A10B-FP8

# 制御モード（spec_api.md 3.6）
CONTROL_MODE=DRY_RUN            # DRY_RUN | LIVE
LIVE_WINDOW_START=              # ISO8601 JST。LIVE 許可期間
LIVE_WINDOW_END=
CONTROL_LOOP_INTERVAL_SEC=300

# DB
DATABASE_URL=postgresql+psycopg://app:app@postgres:5432/building_ai
```

### 1.5 フェーズ計画

| フェーズ | ゴール | 制御 | 完了条件 |
|---|---|---|---|
| **1. 基盤** | 3サービスが起動し、ES の実データからセンサー数と最新値が画面に出る | なし | `docker compose up` で画面表示。`scripts/inspect_es.py` で 2.8 の未確定が埋まる。`check_api.py` で ulid 突合、`check_llm.py` で LLM 検証が通る |
| **2. エージェント（読み取り専用）** | 3エージェントが実データを見て会話し「ポリシー案＋根拠」を出す。人間が承認/修正/棄却でき履歴が残る | DRY_RUN | 会話がストリーミング表示され、承認したポリシーが DB に保存される |
| **3. ポリシーエンジン** | 構造化ポリシーの評価・図示・使用センサー表示・リプレイ検証 | DRY_RUN | 過去1週間のデータでリプレイし、制御回数と推定消費電力量が出る |
| **4. 実制御** | ガードレール・ロールバック・実験開始/終了手順・安全状態復帰 | LIVE（限定エリア） | BEMS 停止下で1日の実験が事故なく完了し、監査ログが揃う |

各フェーズは Claude Code に対する独立したタスク群として `docs/tasks/phaseN.md` に分解する（フェーズ1は [tasks/phase1.md](./tasks/phase1.md)）。

### 1.6 横断的な設計原則

1. **LLM は機器を直接動かさない。** LLM の出力はポリシー（構造化データ）まで。実行はポリシーエンジン → ガードレール → DeviceController。
2. **LLM が止まっても建物は動く。** 承認済みポリシーの評価は LLM に依存しない。
3. **デフォルトは DRY_RUN。** LIVE は環境変数・実験ウィンドウ・WRITE キーの3条件が揃った時だけ。
4. **制御対象はホワイトリスト。** `aircons` / `ventilators` / `lights` / `lighting-interlocks` 以外は呼び出せない実装にする。Elevator は絶対に触らない。
5. **人間優先。** 人感連動 OFF や手動操作された機器は一定時間ポリシー対象外。
6. **すべて記録する。** 制御コマンド・ポリシー変更・承認・ガードレール拒否は監査ログに残す。
7. **enum はシンボルで扱う。** API の数値は API 呼び出し直前でのみ変換する。
