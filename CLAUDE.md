# Building AI プロジェクト — Claude Code 運用ルール

## このリポジトリについて

e-Nexus（電気通信大学 東11号館）のビル設備を、3体の AI エージェント（照明・空調・換気）が
会話しながら制御するための Web アプリケーション。詳細は `docs/` を参照。

- 背景と狙い: `docs/concept.md`
- 実装仕様（親）: `docs/spec.md` → 各節 `docs/spec_*.md`
- 現在のフェーズと作業リスト: `docs/tasks/phase1.md`

仕様に【未確定】と書かれた項目は**推測で埋めない**。ユーザに確認するか、環境変数・設定ファイルに外出しする。

## 環境

- このマシンは **VM#2 enexus-agent**（Ubuntu 24.04）。アプリの実行環境であり、開発もここで行う。
  - VM#1 ai-connect から `ssh enexus-agent` で入ってくる場合もある。その場合の作業ディレクトリは `~/building-ai`。
- 外部接続先（すべて学内 NW）:
  - Elasticsearch: `https://elastic.i4s.uec.ac.jp:9200`（読み取りのみ）
  - e-Nexus API: `http://openapi.i4s.uec.ac.jp/api/v1/e11`
  - vLLM: `http://genai.i4s.uec.ac.jp:8000/v1`（学内共用。大量リクエストは事前確認）
- 認証情報は `.env` のみ。コード・ログ・コミット・チャット出力に鍵や password を含めない。`.env` は `.gitignore` 済み。

## 絶対に守ること（安全）

1. **`CONTROL_MODE` のデフォルトは `DRY_RUN`。** `LIVE` への変更はユーザの明示的な指示があった時のみ。自分で `.env` を LIVE に書き換えない。
2. **設備制御 API の PUT を直接呼ぶコードを書かない。** すべて `backend/app/control/device_controller.py` を経由する。テストでも実 PUT は呼ばず、モックを使う。
3. **制御対象は `aircons` / `ventilators` / `lights` / `lighting-interlocks` のみ。** Elevator / Displaywall / Patlite など他のエンドポイントを呼ぶコードは書かない（将来 API に追加されても同じ）。
4. **`E11_API_KEY_WRITE` を DRY_RUN 中に読み込むコードを書かない。**
5. Elasticsearch には**書き込まない**（新 index の作成も含む）。
6. 学内 LLM へのベンチマーク・並列負荷試験は実行前にユーザに確認する。
7. 実機の状態を変える可能性のあるスクリプト（`scripts/check_api.py` の PUT テスト等）は、実行前にコマンドを提示して確認を取る。

## 開発の進め方

- 作業前に `docs/tasks/phaseN.md` の該当タスクを読み、完了したらチェックを付ける。
- 複数ファイルにまたがる変更や、新しい依存パッケージの追加は、着手前に方針を1〜3行で提示する。
- 仕様と矛盾する実装が必要になったら、実装せずに矛盾点を報告する。
- 疑問があれば推測せず質問する。特に enum の意味、単位、時刻の基準（JST/UTC）。

## コーディング規約

**Python（backend）**
- Python 3.12、`uv` でパッケージ管理。`pip` は使わない。
- 型ヒント必須。データ構造は Pydantic v2。
- ES / API / LLM の各クライアントは `backend/app/{datasource,control,llm}/` に閉じ込め、他モジュールから直接 HTTP を呼ばない。
- 時刻は内部 UTC（`datetime` aware）、表示時のみ JST。ES の `@timestamp` を正とし、機器側 `timestamp` 文字列は使わない。
- API の enum はアプリ内でシンボル（`config/enums.yaml`）として扱い、数値変換は `e11_client.py` 内だけ。
- ログは構造化（`structlog`）。制御・ガードレール判定・LLM 呼び出しは必ずログに残す。
- テスト: `pytest`。`control/` と `policy/` は変更のたびにテストを実行。

**TypeScript（frontend）**
- React 18 + Vite + TypeScript strict。スタイルは Tailwind。
- API 型は backend の OpenAPI から生成（`openapi-typescript`）。手書きしない。
- キャラクター画像は SVG をリポジトリ内で管理。外部 CDN・画像生成 API は使わない。

**共通**
- コミットメッセージは英語で `type(scope): summary`（例 `feat(datasource): add points builder`）。
- 生成物（`data/points.csv` 以外）・`.env`・`node_modules`・`__pycache__` はコミットしない。

## コマンド

```bash
# 起動
docker compose up -d --build
docker compose logs -f backend

# backend 単体
cd backend && uv sync && uv run uvicorn app.main:app --reload

# テスト
cd backend && uv run pytest
cd frontend && npm test

# 調査スクリプト（DRY_RUN 前提、READ キーのみ使用）
cd backend && uv run python ../scripts/inspect_es.py
cd backend && uv run python ../scripts/check_api.py
cd backend && uv run python ../scripts/check_llm.py
```

## 用語

| 用語 | 意味 |
|---|---|
| ポイント / point_id | ES のフィールド名（例 `E11F01CWS1IAQMONITOR009_co2`）。命名規則は `docs/spec_datasource.md` 2.3 |
| ulid | device-info / e-Nexus API の機器 ID。point_id とは device-info の `param1` で対応 |
| ポリシー | エージェントが提案し人間が承認する構造化された制御ルール |
| DRY_RUN / LIVE | 制御コマンドを記録だけするか、実機に送るか |
| ガードレール | ポリシーエンジンと DeviceController の間の安全チェック層 |
| 安全状態 | アプリ停止時に機器を戻す既定値（`config/guardrails.yaml`） |
