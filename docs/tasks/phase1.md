# フェーズ1：基盤

> 配置先: `docs/tasks/phase1.md`
> ゴール: 3サービスが起動し、ES の実データからセンサー数と最新値が画面に出る。未確定項目を調査スクリプトで埋める。
> 制御なし。使う鍵は `E11_API_KEY_READ` と `LLM_API_KEY` のみ。設備 API は v1 が未稼働のため `DEVICE_BACKEND=v0` で進める。

## 前提

- `.env` はユーザが用意する（`.env.example` からコピーして鍵を記入）。
- enexus-agent に Docker / Docker Compose / uv / Node.js 22 が入っていること（Task 0 で確認）。

## タスク

各タスクは独立した Claude Code セッションで実行できる粒度。上から順に。

### Task 0: 環境セットアップ
- [ ] `infra/enexus-agent/setup.sh` を作成: Docker Engine + Compose plugin、uv、Node.js 22（nodesource）、git、jq をインストール。冪等に書く
- [ ] 実行前にスクリプトを提示して確認を取り、実行
- [ ] `docker --version`, `uv --version`, `node --version` を確認

### Task 1: リポジトリ骨組み
- [ ] `docs/spec.md` 1.3 の構成でディレクトリ・空ファイルを作る
- [ ] `.env.example`（spec.md 1.4）、`.gitignore`
- [ ] `docker-compose.yml`: backend / frontend / postgres の3サービス。backend は `.env` を読む。postgres はボリューム永続化
- [ ] `backend/pyproject.toml`: fastapi, uvicorn, pydantic, pydantic-settings, elasticsearch (8.11系), httpx, openai, sqlalchemy, psycopg, alembic, structlog, pyyaml, pytest, pytest-asyncio
- [ ] `backend/app/main.py`: `/health` が `{"status":"ok"}` を返す
- [ ] `frontend`: Vite + React + TS + Tailwind の初期化。トップに "Building AI" と backend の `/health` 結果を表示
- [ ] `docker compose up -d --build` で3サービスが起動することを確認

### Task 2: ES クライアントと調査スクリプト
- [ ] `backend/app/datasource/es_client.py`: `.env` から接続。`ES_CA_CERT` があれば使い、無ければ `ES_VERIFY_CERTS` に従う。接続確認メソッド
- [ ] `scripts/inspect_es.py`: 以下を調査して Markdown で標準出力する
  - ES バージョン、`e11_*` と `device-info` の index 一覧・件数・サイズ
  - `e11_sens-*` の直近1日のドキュメント数（環境 / 電力を `exists` で分けて）→ サンプリング間隔を推定
  - `e11_hds_detection-*` の直近1日の件数
  - `device-info` の総件数、`Device Category` × `Device_type` の件数表
  - 電力ドキュメントで `message` にあって `_source` に無いフィールドの有無（直近1件で比較）
  - `_cat/indices` から保持期間（最古の index 名）
- [ ] 実行し、結果を `docs/spec_datasource.md` 2.8 の該当項目に反映する提案を出す（編集はユーザ確認後）

### Task 3: ポイント台帳
- [ ] `config/areas.yaml`: spec_datasource.md 2.3 のエリアコード表
- [ ] `backend/app/datasource/points.py`: point_id の正規表現パーサ（spec_datasource.md 2.3）。単体テスト付き（表の5例＋不正な名前）
- [ ] `scripts/build_points.py`: `e11_sens-*` と `e11_hds_detection-*` の直近ドキュメントからフィールド名を収集 → パース → `device-info` の `param1` と突合して `device_ulid` を埋める → `data/points.csv` を出力。突合できなかった device-info の機器と、device-info に無いポイントを一覧で報告
- [ ] 実行して `data/points.csv` を生成

### Task 4: 最新値の取得
- [ ] `backend/app/datasource/latest.py`: 環境センサー最新値・電力最新値（各1クエリ）、人感の現在状態（直近 N 時間の全件から畳み込み。N は設定値、初期 24h）。戻り値は `{point_id: {value, timestamp}}`
- [ ] `backend/app/datasource/power.py`: 累積カウンタ → 本日 kWh（00:00 JST 基準、spec_datasource.md 2.3）。負の差分は異常として返す
- [ ] `backend/app/api/sensors.py`: `GET /api/sensors/summary`（階×種別の件数）、`GET /api/sensors/latest`、`GET /api/power/today`
- [ ] 単体テスト（ES はモック）

### Task 5: 設備 API バックエンド（読み取りのみ）
- [ ] `docs/api/e11-v1.json`, `docs/api/e11-v0.json` をリポジトリに配置
- [ ] `config/enums.yaml`（spec_api.md 3.3）。`v0_value` 列は空で作る
- [ ] `backend/app/control/backends/base.py`: `DeviceBackend` 抽象クラス。語彙は v1（ulid + シンボル値）。**この Task では GET 系（`list_devices`, `get_status`）のみ定義**し、PUT 系メソッドは作らない
- [ ] `backends/v1.py`: `httpx` ラッパー。enum → シンボル変換。エラー応答（3.4）を例外にマップ。v1 が未稼働なので単体テストはモックで
- [ ] `backends/v0.py`: device-info の `param1` から `{系統}/{deviceId}` を引き、`GET /webapi/.../commands/{controlKey}` で状態取得。`Result` のパース、`N/R` / `ErrorType` の例外化。**最初の1回はレスポンス生 JSON を保存して形を確認**してからパーサを書く
- [ ] `backends/mock.py`: 固定値を返す
- [ ] `scripts/check_api.py --backend v0|v1`: 疎通、系統 / deviceId 一覧、device-info の Actuator との突合、各 controlKey の `Type` / `RangeLo` / `RangeHi` 一覧（v0）
- [ ] v0 で実行し、spec_api.md 3.11.7 の 1〜4、3.10 の 6 に反映する提案。`config/enums.yaml` の `v0_value` を埋める提案

### Task 6: LLM アダプタと検証
- [ ] `backend/app/llm/qwen_adapter.py`: `openai` SDK で vLLM に接続。`chat(messages, tools=None, thinking=True, response_schema=None)` の1インターフェース。thinking の ON/OFF は `extra_body.chat_template_kwargs.enable_thinking` で切り替え（動かなければ報告）
- [ ] `scripts/check_llm.py`: (1) `/models` 一覧、(2) 単発 tool call（ダミー tool `get_temperature(point_id)`）、(3) 並列 tool call（2つ同時）、(4) thinking ON/OFF で `reasoning_content` の有無、(5) `response_format: json_schema` で Pydantic モデルに合致する JSON が返るか。各項目 OK/NG と所要時間を表で出力
- [ ] 実行して spec_llm.md 4.5 の 1〜3 に反映する提案

### Task 7: ダッシュボード v0
- [ ] `GET /api/sensors/summary` を使い、階ごと（1F/2F/3F）×種別（温度・湿度・CO2・放射温度・人感・電力）のセンサー数をカード表示
- [ ] エリア別の最新値テーブル（`config/areas.yaml` の日本語名で表示。LAB と POC は `実験室（LAB1）` のようにコード併記）
- [ ] 本日の電力量（エリア合計ポイントのみ）
- [ ] 30 秒ごとに自動更新
- [ ] デザインは簡素でよい。妖怪キャラクターはフェーズ2

### 完了条件
- [ ] `docker compose up -d --build` → ブラウザで実データが表示される
- [ ] `inspect_es.py` / `check_api.py` / `check_llm.py` の結果が spec に反映されている
- [ ] `pytest` が通る
- [ ] `git log` に各 Task が独立したコミットとして残っている

---

## Claude Code への最初の指示文（コピー用）

enexus-agent 上の `~/building-ai` で `claude` を起動し、以下を貼る。

```
docs/tasks/phase1.md の Task 0 と Task 1 を進めてください。

進め方:
1. まず CLAUDE.md と docs/spec.md を読み、不明点があれば作業前に質問してください。
2. Task 0 の setup.sh は、実行前に内容を見せて私の確認を待ってください。
3. Task 1 は docker compose up で3サービスが起動するところまで。
   フロントは "Building AI" の文字と backend の /health の結果が出れば十分です。
4. 各 Task 完了時にコミットし、phase1.md のチェックボックスを更新してください。
5. .env は私が用意します。.env.example を作ったら教えてください。
```

Task 2 以降は、前の Task が完了したセッションで「次は Task N をお願いします」と続ける。
Task 2・5・6 の調査スクリプトは、結果を spec に反映する前に必ず結果を見せてもらう。
