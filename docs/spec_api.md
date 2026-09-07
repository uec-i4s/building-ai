# 3. 設備制御 API（e-Nexus Building API v1）

> 本節は spec.md の一部。【未確定】と記した項目は確認後に埋めること。
> Claude Code への注意：本節の API は**実機を動かす**。PUT 系は必ず dry-run 層を経由させ、フェーズ3までは実呼び出しを無効にしておく。

## 3.1 基本情報

| 項目 | 値 |
|---|---|
| 仕様書（OpenAPI 3.0.4） | `https://api.sep.i4s.uec.ac.jp/docs/specs/e11-v1.json` |
| 仕様書のローカルコピー | `docs/api/e11-v1.json`（リポジトリに同梱。クライアントはこれから生成する） |
| タイトル / バージョン | UEC East 11 (e-Nexus) Building API v1 / 1.0.0 |
| Production サーバ | `http://openapi.i4s.uec.ac.jp/api/v1/e11`（API Gateway 経由） |
| Development サーバ | `http://openapi.i4s.uec.ac.jp:8889/v1/e11` |
| 認証 | API キー。HTTP ヘッダ `X-API-Key: <key>` |
| API キー発行 | SEP Portal `https://portal.sep.i4s.uec.ac.jp/`（学内アカウント認証）のダッシュボードから発行 |
| 認証情報の保管 | `.env` の `E11_API_URL` / `E11_API_KEY_READ` / `E11_API_KEY_WRITE`。リポジトリにコミットしない |
| ライセンス | MIT |

実際に叩く URL は `servers` に記載の `openapi.i4s.uec.ac.jp` で確定（API 自体が開発中のため暫定。変更に備え `E11_API_URL` は環境変数で外出し、ハードコードしない）。
仕様書のホスト `api.sep.i4s.uec.ac.jp` はドキュメント配信用。

API キーは SEP Portal で管理する。キーは**個人の学内アカウントに紐づく**ため、本アプリ用のキーを誰のアカウントで発行するか（担当者個人か、共用アカウントか）を決めておく。担当者交代時のキー再発行手順も運用文書に残す。

Portal で発行できるキー（確定）: 権限 = GET のみ / PUT のみ / GET+PUT、有効期限設定可、スコープ設定可（E11 のみ等）。

本アプリでは**2本のキーを使い分ける**:

| キー | 権限 | スコープ | 用途 | 有効期限 |
|---|---|---|---|---|
| `E11_API_KEY_READ` | GET のみ | E11 | 状態取得・監視。常時使用 | 長め（例: 1年） |
| `E11_API_KEY_WRITE` | PUT のみ（または GET+PUT） | E11 | LIVE モードの制御のみ。`CONTROL_MODE=LIVE` かつ実験ウィンドウ内でのみロード | **実験期間に合わせ短く**（例: 実験日当日〜1週間） |

DRY_RUN モードでは `E11_API_KEY_WRITE` を環境に置かない（置いてあっても読まない）。これによりフェーズ1〜2でコードのバグで PUT が飛んでも 401 で止まる。

【未確定】
- Portal に API 呼び出しログ・利用量の確認画面があるか（device-info の `access_log_url` / `error_log_url` が "TBF" なので、Portal 側に統合される可能性）。
- Production / Development の使い分け（Development は実機に繋がっているか、モックか）。
- enexus-agent から `openapi.i4s.uec.ac.jp` への到達性。HTTPS ではなく HTTP である点も要確認（Gateway 前段で TLS 終端しているか）。
- API が開発中のため、仕様変更を検知する仕組みが必要。起動時に仕様書 URL を取得し、同梱の `docs/api/e11-v1.json` とハッシュ比較して差分があれば警告する。

疎通確認:

```bash
curl -s -H "X-API-Key: $E11_API_KEY_READ" "$E11_API_URL/health"        # → "OK"
curl -s -H "X-API-Key: $E11_API_KEY_READ" "$E11_API_URL/aircons"       # → {"devices":[{ulid,location,region,building,room},...]}
```

## 3.2 エンドポイント一覧

### 制御対象（読み書き）

| リソース | 一覧 | 状態取得 | 設定（PUT） | 状態スキーマ |
|---|---|---|---|---|
| 空調 `aircons` | `GET /aircons` | `GET /aircons/{ulid}` | `/mode`, `/temperature`, `/fan-speed` | `{mode, fan_speed, setpoint_temperature, room_temperature}` |
| 全熱交換器（換気） `ventilators` | `GET /ventilators` | `GET /ventilators/{ulid}` | `/power`, `/mode`, `/fan-speed` | `{power, mode, fan_speed}` |
| 照明 `lights` | `GET /lights` | `GET /lights/{ulid}` | `/power`, `/brightness` | `{power, brightness}` |
| 照明人感連動 `lighting-interlocks` | `GET /lighting-interlocks` | `GET /lighting-interlocks/{ulid}` | `/enabled` | `{enabled}` |

### センサー（読み取り専用）

| リソース | 一覧 | 状態取得 | 値 |
|---|---|---|---|
| `co2` | `GET /co2` | `GET /co2/{ulid}` | `{co2}` ppm |
| `humidity` | `GET /humidity` | `GET /humidity/{ulid}` | `{humidity}` %RH |
| `temperature` | `GET /temperature` | `GET /temperature/{ulid}` | `{temperature}` ℃ |
| `radiant-temperature` | `GET /radiant-temperature` | `GET /radiant-temperature/{ulid}` | `{radiant_temperature}` ℃ |
| `occupancy` | `GET /occupancy` | `GET /occupancy/{ulid}` | `{occupancy}` 0/1 |
| `power-consumption` | `GET /power-consumption` | `GET /power-consumption/{ulid}` | `{power_consumption}` 積算値 |
| `health` | `GET /health` | — | `"OK"` |

PUT のリクエストボディはいずれも `{"<フィールド名>": <値>}` の1項目のみ。成功時は `200 {"message": "..."}`。

## 3.3 値の定義（enum）— 実装上の落とし穴

**enum の数値は「大小に意味がない」ものが多い。** LLM に直接数値を生成させず、アプリ内では必ずシンボル（`COOL`, `AUTO` 等）で扱い、API 呼び出し直前にマッピングする。

### 空調

| フィールド | 値 | 意味 | 備考 |
|---|---|---|---|
| `mode` | 0 / 1 / 2 / 3 / 5 | 停止 / 冷房 / 暖房 / 送風 / ドライ | **4 は存在しない**。`mode=0` で停止、`mode≠0` でモード設定＋運転開始 |
| `fan_speed` | 1 / 2 / 3 / 4 / 5 | 微弱 / **強** / 弱 / 中 / 自動 | **順序が強さと一致しない**（弱→強は 1,3,4,2） |
| `setpoint_temperature` | 17.0〜28.0 | 設定温度 ℃ | float。範囲外は 400 |
| `room_temperature` | -99〜99 | 室内温度 ℃ | 読み取り専用。`/temperature` の値とは別センサー |

### 全熱交換器（換気）

| フィールド | 値 | 意味 | 備考 |
|---|---|---|---|
| `power` | 1 / 2 / 3 | **停止** / 運転 / 24時間換気 | **1 が停止**（照明の 0/1 と逆感覚） |
| `mode` | 1 / 2 / 3 | 熱交換 / 普通 / 自動 | |
| `fan_speed` | 1 / 2 / 3 / 5 | 微弱 / 強 / 弱 / 自動 | **4（中）が無い**。空調の fan_speed と enum が異なり、相互変換禁止 |

### 照明

| フィールド | 値 | 意味 |
|---|---|---|
| `power` | 0 / 1 | 消灯 / 点灯 |
| `brightness` | 0.0〜100.0 | 調光率 % |

### 照明人感連動

| フィールド | 値 | 意味 | 備考 |
|---|---|---|---|
| `enabled` | 0 / 1 | 連動停止 / 連動動作 | device-info の `HDS-Controlstop` に対応。**手動優先スイッチ** |

【未確定】`brightness` を設定した時 `power` も自動で 1 になるか。`power=0` で `brightness` は保持されるか。

### アプリ内シンボル定義（`config/enums.yaml` 案）

```yaml
aircon:
  mode:      {OFF: 0, COOL: 1, HEAT: 2, FAN: 3, DRY: 5}
  fan_speed: {QUIET: 1, HIGH: 2, LOW: 3, MID: 4, AUTO: 5}
  fan_speed_order: [QUIET, LOW, MID, HIGH]      # 強さの順序（AUTO は除く）
ventilator:
  power:     {OFF: 1, ON: 2, ALWAYS_ON_24H: 3}
  mode:      {HEAT_EXCHANGE: 1, NORMAL: 2, AUTO: 3}
  fan_speed: {QUIET: 1, HIGH: 2, LOW: 3, AUTO: 5}
  fan_speed_order: [QUIET, LOW, HIGH]
light:
  power:     {OFF: 0, ON: 1}
interlock:
  enabled:   {OFF: 0, ON: 1}
```

## 3.4 エラー応答

| コード | 意味 | アプリでの扱い |
|---|---|---|
| 400 | リクエスト不正（範囲外の値など） | バグ扱い。ガードレール側でここに到達させない |
| 401 | API キー不正 | 起動時ヘルスチェックで検出 |
| 404 | ulid 不在 | device-info との不整合として記録 |
| 500 | API サーバ内部エラー | リトライ1回、失敗で警告 |
| 502 | 上流設備との通信失敗 | **設備側障害**。リトライせず警告。制御は「未反映」として扱う |
| 504 | 上流設備タイムアウト | 同上。状態を再取得して反映有無を確認 |

エラーボディは `{"title": "...", "detail": "..."}`。

【未確定】PUT は同期か非同期か（200 が返った時点で設備に反映済みか、コマンド受理のみか）。反映遅延があるなら、PUT 後 N 秒待って GET で検証する。

## 3.5 Elasticsearch との役割分担

| 用途 | 使うもの | 理由 |
|---|---|---|
| 現在状態の取得（設定温度・モード・点灯状態） | **API** `GET /{resource}/{ulid}` | ES には設定値（setpoint 等）が入っていない |
| 現在のセンサー値 | API または ES のどちらでも可。**フェーズ1は ES** | ES は1リクエストで全ポイント取れる。API は ulid ごとに1リクエスト |
| 履歴・統計・リプレイ検証 | **ES** | API に履歴取得は無い |
| 制御 | **API** `PUT` | ES は読み取り専用 |
| 機器台帳 | device-info（ES）を正とし、API の `GET /{resource}` で存在確認 | |

### ulid とポイント ID の対応

API の一覧応答 `{ulid, location, region, building, room}` には ES のポイント接頭辞（`E11F01HALL1AIRCON001`）が含まれない。
対応は device-info の `ulid` ⇔ `param1` で取る。起動時に以下を突合し、不一致をログに出す。

- device-info の Actuator（Light / Air Conditioner / Ventilation / HDS-Controlstop）の ulid 集合
- API `GET /aircons`, `/ventilators`, `/lights`, `/lighting-interlocks` の ulid 集合

【未確定】API のセンサー系 ulid（`/co2` 等の測定点）も device-info に `Sensor` として登録されているか。

## 3.6 制御レイヤの設計（ガードレール前提）

すべての PUT は `DeviceController` を経由し、直接 HTTP を呼ばない。

```
Agent / Policy engine
   │  ControlCommand(symbolic)  例: {target: ulid, action: set_temperature, value: 26.0, reason: "..."}
   ▼
Guardrail ── 拒否 → 監査ログ + UI 警告
   │  ・ホワイトリスト（ulid が制御対象か）
   │  ・値域（17〜28℃、0〜100% 等。enum は定義済みシンボルのみ）
   │  ・変化量上限（1回の変更で設定温度 ±2℃ まで、等。config で定義）
   │  ・レート制限（同一機器へ N 分に1回まで）
   │  ・人感連動 OFF（interlock.enabled=0）の機器には照明ポリシーを適用しない
   │  ・時間帯制約（夜間の暖房 ON 禁止、等）
   ▼
DeviceController
   │  mode = DRY_RUN | LIVE   （.env の CONTROL_MODE。デフォルト DRY_RUN）
   │  DRY_RUN: 監査ログに「実行したはずのリクエスト」を記録し、200 を模擬
   │  LIVE:    実 PUT → 監査ログ → N 秒後 GET で反映確認 → 不一致なら警告
   ▼
DeviceBackend（抽象。.env の DEVICE_BACKEND で選択）
   ├ E11V1Backend   … e-Nexus API v1（本命。3.1〜3.5）
   ├ E11V0Backend   … 基幹 API v0（暫定。3.11）
   └ MockBackend    … テスト・オフラインデモ用
```

`DeviceBackend` のインターフェースは v1 の語彙（ulid + シンボル値）で統一し、v0 への変換は `E11V0Backend` 内に閉じ込める。上位層（ガードレール・ポリシーエンジン・UI）はどのバックエンドかを知らない。

監査ログ（PostgreSQL `control_log`）の必須項目:
`timestamp, agent, ulid, point_id, action, before_value, after_value, reason, policy_id, mode(DRY_RUN/LIVE), http_status, verified`

ロールバック: 各制御コマンドは `before_value` を持つため、`policy_id` 単位で逆順に PUT し直すことでロールバックする。

## 3.7 既存 BEMS との関係（確定）

**本アプリによる実制御の実験時は BEMS を停止する。** つまり LIVE モード中は本アプリが e-Nexus 設備の唯一の制御主体になる。これは競合の問題を解消する一方で、以下の責任が本アプリに移る。

| 観点 | 要件 |
|---|---|
| フェイルセーフ | 本アプリの停止・クラッシュ・LLM 応答不能時に設備が放置されないこと。**「安全状態（safe state）」を機器ごとに定義**し（例: 空調 = 自動運転 26℃、換気 = 24時間換気、照明 = 人感連動 ON）、`CONTROL_MODE=LIVE` を終了する時と watchdog がアプリ死活を検知した時に安全状態へ戻す |
| 実験の開始・終了手順 | (1) BEMS 停止 → (2) 本アプリで全機器の現在状態をスナップショット（`experiment_snapshot`）→ (3) LIVE 有効化 → … → (4) LIVE 無効化 → (5) スナップショットまたは安全状態へ復元 → (6) BEMS 再開。UI に「実験開始 / 終了」ボタンとしてこの手順を実装する |
| 手動介入 | LIVE 中でも人間が UI から任意の機器を直接操作でき、その機器は一定時間ポリシーの対象外になる（手動優先ロック） |
| 実験ウィンドウ | LIVE を許可する期間（開始・終了時刻）を明示的に設定し、期間外は自動で DRY_RUN に戻す |
| 制御ループ周期 | BEMS 不在時の制御周期は本アプリが決める。初期値 5 分（ES のサンプリング約 2 分より長く、空調の応答より短い） |

【未確定】
- BEMS の停止・再開は誰がどう行うか（本アプリから操作できる API は無い前提で、手順書として運用する）。
- 実験対象を建物全体にするか、特定エリア（例: F03 会議室のみ）に限定するか。**初回実験は限定エリアを強く推奨。**
- 安全状態の具体値（機器種別ごと、季節ごと）。

## 3.8 Elevator 等の除外

API には Elevator / Displaywall / Patlite のエンドポイントは存在しない（v1 時点）。
将来追加されても、本アプリの `DeviceController` は `aircons` / `ventilators` / `lights` / `lighting-interlocks` 以外のパスを呼び出せない実装にする。

## 3.9 クライアント実装方針

- `docs/api/e11-v1.json` から Python クライアントを生成（`openapi-python-client`）するか、エンドポイントが少ないので `httpx` で薄いラッパーを手書きする。**推奨: 手書き**（enum のシンボル変換とガードレールを同じ層に置きやすい）。
- タイムアウト: 接続 3 秒 / 読み取り 10 秒（504 の上流タイムアウトより短くならないよう要調整）。
- 一覧系 GET は起動時 + 10 分ごとにキャッシュ更新。状態系 GET はキャッシュしない。

## 3.11 基幹 API v0（暫定バックエンド）

> 仕様: `docs/api/e11-v0.json`（RESTfulAPIwrapper 1.0）。設備導入時に業者から提供された BACnet ラッパー。
> v1 は本 API をさらにラップして API キー認証・ulid・型付き enum を付けたもの。
> **v1 が不具合修正中のため、デモ・検証では v0 を `E11V0Backend` として使う。v1 復旧後は `DEVICE_BACKEND=v1` に切り替える。**

### 3.11.1 基本情報

| 項目 | 値 |
|---|---|
| ベース URL | `http://192.168.13.10:8080`（確定）。パスは `/webapi/...`。Swagger UI: `http://192.168.13.10:8080/swagger/index.html` |
| 認証 | **なし**（確定）。学内 NW 制限のみ。enexus-agent から到達可 |
| GET | `GET /webapi/{系統}/{deviceId}/points/{pointKey}` または `.../commands/{controlKey}` |
| PUT | `PUT /webapi/{系統}/{deviceId}/commands` ボディ `{"command": "<controlKey>", "value": "<文字列>"}` |
| 応答 | `Result` オブジェクト: `Request, ReadValue("N/R"=無応答), Name, Type, RangeLo, RangeHi, Comment, EquipmentSystemList, EquipmentDeviceList, PointKey, ControlKey, ErrorType, ErrorMessage` |
| 一覧取得 | `GET /webapi/root` → 系統一覧、`/webapi/{系統}` → deviceId 一覧、`/webapi/{系統}/{deviceId}/points|commands` → キー一覧 |

v0 は BACnet コントローラの前にある業者提供のラッパーで、プライベート IP（`192.168.13.0/24`）上にある。enexus-agent がこの NW に到達できる＝**設備を直接動かせる位置にいる**ことを意味する。

**認証がないため、v0 使用時は DRY_RUN 原則と WRITE 側の抑止をアプリ内で担保するしかない。** v1 のように「鍵を置かなければ PUT が通らない」保険が効かないことを前提に、`E11V0Backend` の PUT は `CONTROL_MODE=LIVE` かつ実験ウィンドウ内でなければメソッド呼び出し自体を例外にする。

### 3.11.2 系統（BACnet コントローラ）

| 系統 | 内容 | v1 リソース |
|---|---|---|
| HVAC01 | 空調機 | aircons |
| VENT01 | 全熱交換器 | ventilators |
| LIGHT01 / 02 / 03 | 1〜3階 照明系統（照明と人感センサ） | lights, lighting-interlocks, occupancy |
| SENS01 | 1階 温湿度 CO2、放射温度 | temperature, humidity, co2, radiant-temperature |
| SENS02 | 1〜3階 電力量、2階 温湿度 CO2 | power-consumption ほか |

### 3.11.3 deviceId と ulid の対応

deviceId は `[棟][階][部屋][設備nnn]`（例 `E11F01HALL1AIRCON001`）で、ES のポイント接頭辞と同一。
device-info の `param1` は **`{系統}/{deviceId}` そのもの**（例 `HVAC01/E11F01HALL1AIRCON001`）なので、

```
ulid  ──device-info.param1──▶  {系統}/{deviceId}  ──"/" 分割──▶  v0 パス
```

で v1 ⇔ v0 の対応が取れる。追加の対応表は不要。

### 3.11.4 controlKey / pointKey と v1 の対応

| v0 キー | 意味 | v1 での対応 | 備考 |
|---|---|---|---|
| `power` | 発停・点灯消灯 | 照明 `power`、換気 `power`。**空調は v1 に無い**（v1 は `mode=0` が停止） | 空調の停止を v0 で行う場合 `power` と `mode` のどちらを使うか【未確定】 |
| `mode` | モード | 空調 `mode`、換気 `mode` | |
| `fanspeed` | 風量 | `fan_speed` | |
| `tempset` | 室温設定 | `setpoint_temperature` | |
| `dimmer` | 照明調光 | `brightness` | |
| `controlstop` | 人感センサ照明連動 | `lighting-interlocks.enabled` | 極性は v1 と同じ（確定）: `0`=連動停止 / `1`=連動有効。名前に反して `1` が有効なので注意 |
| `temp` / `humidity` / `co2` / `radtemp` / `detection` / `powerconsumption` | 計測値（読み取り） | 各センサーリソース | |

**値の型・enum は v0 仕様に記載がない。** `value` は文字列。v1 の enum（`mode: 0/1/2/3/5` 等）は BACnet の生値と推定されるが、【未確定】として以下で実機確認する:

```
GET /webapi/HVAC01/E11F01HALL1AIRCON001/commands/mode
→ Result.Type / RangeLo / RangeHi / Comment を読み、v1 enum と突合
```

`config/enums.yaml` に `v0_value` 列を追加し、確認結果を記録する。確認が取れていない controlKey への PUT はガードレールで拒否する。

### 3.11.5 v0 固有の注意

- `ReadValue == "N/R"` は無応答。v1 の 502/504 相当として扱う。
- `ErrorType` / `ErrorMessage` が非空なら失敗。HTTP ステータスは 200 のまま返る可能性があるため、**HTTP 200 を成功と見なさない**。
- レスポンスの JSON 形が仕様書に無い（`Result` の記述のみ）。最初の GET で実際の形を確認し、`E11V0Backend` のパーサを書く。
- v0 は BEMS も使っている経路の可能性が高い。LIVE 実験時の BEMS 停止は v0 でも同様に必要。

### 3.11.6 部屋コード一覧（v0 仕様より。ES エリアコードの正式定義）

| コード | 名称 | | コード | 名称 |
|---|---|---|---|---|
| HALL1 | エントランスホール | | POC1〜3 | 実証実験室１〜３ |
| CWS1 | コワーキングスペース | | CMR1 | 前室中央監視室 |
| STG1 | 倉庫 | | SVR1 | サーバー室 |
| COR1 | 廊下 | | EVH1 | EV ホール |
| EXT1 | 外構 | | MTG1〜2 | 会議室１〜２ |
| KIT1 | 給湯室 | | ROOF1 | 屋上 |
| LAB1〜3 | 共同研究実験室１〜３ | | ATR1 | 階段吹抜け |
| EPSxL1 | x階 EPS室 照明分電盤 | | EPSxP1 | x階 EPS室 動力分電盤 |

設備コードには `WHMETERnnn`（電力量計）が追加。
【未確定】ES の電力ポイント `E11F01EPS1LIGHT1` / `E11F01EPS1MAIN1` は v0 の `EPS1L1` / `EPS1P1` と命名が異なる。ES 側は別の整形を経ている可能性があり、`scripts/build_points.py` の突合結果で確認する。

### 3.11.7 確認事項

**確定済み**: ベース URL `http://192.168.13.10:8080`、認証なし（NW 制限のみ）、`controlstop` は `0`=停止 / `1`=有効（v1 と同じ極性）

**残り【未確定】**
1. GET 応答の実際の JSON 形
2. 各 controlKey の値の型・enum（`Type` / `RangeLo` / `RangeHi` から）と v1 enum との一致
3. 空調の停止方法（`power` か `mode=0` か）
4. ES の `EPS1LIGHT1` / `EPS1MAIN1` と v0 の `EPSxL1` / `EPSxP1` の対応

1〜2 は `scripts/check_api.py --backend v0` で GET のみで確認できる。3 は `HVAC01/{deviceId}/commands` のキー一覧に `power` があるかで判断する。

## 3.11 基幹 API v0（暫定バックエンド）

> 仕様: `docs/api/e11-v0.json`（RESTfulAPIwrapper 1.0）。設備導入時に業者から提供された BACnet ラッパー。
> v1 は本 API をさらにラップして API キー認証・ulid・型付き enum を付けたもの。
> **v1 が不具合修正中のため、デモ・検証では v0 を `E11V0Backend` として使う。v1 復旧後は `DEVICE_BACKEND=v1` に切り替える。**

### 3.11.1 基本情報

| 項目 | 値 |
|---|---|
| ベース URL | `http://192.168.13.10:8080`（確定）。パスは `/webapi/...`。Swagger UI: `http://192.168.13.10:8080/swagger/index.html` |
| 認証 | **なし**（確定）。学内 NW 制限のみ。enexus-agent から到達可 |
| GET | `GET /webapi/{系統}/{deviceId}/points/{pointKey}` または `.../commands/{controlKey}` |
| PUT | `PUT /webapi/{系統}/{deviceId}/commands` ボディ `{"command": "<controlKey>", "value": "<文字列>"}` |
| 応答 | `Result` オブジェクト: `Request, ReadValue("N/R"=無応答), Name, Type, RangeLo, RangeHi, Comment, EquipmentSystemList, EquipmentDeviceList, PointKey, ControlKey, ErrorType, ErrorMessage` |
| 一覧取得 | `GET /webapi/root` → 系統一覧、`/webapi/{系統}` → deviceId 一覧、`/webapi/{系統}/{deviceId}/points|commands` → キー一覧 |

v0 は BACnet コントローラの前にある業者提供のラッパーで、プライベート IP（`192.168.13.0/24`）上にある。enexus-agent がこの NW に到達できる＝**設備を直接動かせる位置にいる**ことを意味する。

**認証がないため、v0 使用時は DRY_RUN 原則と WRITE 側の抑止をアプリ内で担保するしかない。** v1 のように「鍵を置かなければ PUT が通らない」保険が効かないことを前提に、`E11V0Backend` の PUT は `CONTROL_MODE=LIVE` かつ実験ウィンドウ内でなければメソッド呼び出し自体を例外にする。

### 3.11.2 系統（BACnet コントローラ）

| 系統 | 内容 | v1 リソース |
|---|---|---|
| HVAC01 | 空調機 | aircons |
| VENT01 | 全熱交換器 | ventilators |
| LIGHT01 / 02 / 03 | 1〜3階 照明系統（照明と人感センサ） | lights, lighting-interlocks, occupancy |
| SENS01 | 1階 温湿度 CO2、放射温度 | temperature, humidity, co2, radiant-temperature |
| SENS02 | 1〜3階 電力量、2階 温湿度 CO2 | power-consumption ほか |

### 3.11.3 deviceId と ulid の対応

deviceId は `[棟][階][部屋][設備nnn]`（例 `E11F01HALL1AIRCON001`）で、ES のポイント接頭辞と同一。
device-info の `param1` は **`{系統}/{deviceId}` そのもの**（例 `HVAC01/E11F01HALL1AIRCON001`）なので、

```
ulid  ──device-info.param1──▶  {系統}/{deviceId}  ──"/" 分割──▶  v0 パス
```

で v1 ⇔ v0 の対応が取れる。追加の対応表は不要。

### 3.11.4 controlKey / pointKey と v1 の対応

| v0 キー | 意味 | v1 での対応 | 備考 |
|---|---|---|---|
| `power` | 発停・点灯消灯 | 照明 `power`、換気 `power`。**空調は v1 に無い**（v1 は `mode=0` が停止） | 空調の停止を v0 で行う場合 `power` と `mode` のどちらを使うか【未確定】 |
| `mode` | モード | 空調 `mode`、換気 `mode` | |
| `fanspeed` | 風量 | `fan_speed` | |
| `tempset` | 室温設定 | `setpoint_temperature` | |
| `dimmer` | 照明調光 | `brightness` | |
| `controlstop` | 人感センサ照明連動 | `lighting-interlocks.enabled` | 極性は v1 と同じ（確定）: `0`=連動停止 / `1`=連動有効。名前に反して `1` が有効なので注意 |
| `temp` / `humidity` / `co2` / `radtemp` / `detection` / `powerconsumption` | 計測値（読み取り） | 各センサーリソース | |

**値の型・enum は v0 仕様に記載がない。** `value` は文字列。v1 の enum（`mode: 0/1/2/3/5` 等）は BACnet の生値と推定されるが、【未確定】として以下で実機確認する:

```
GET /webapi/HVAC01/E11F01HALL1AIRCON001/commands/mode
→ Result.Type / RangeLo / RangeHi / Comment を読み、v1 enum と突合
```

`config/enums.yaml` に `v0_value` 列を追加し、確認結果を記録する。確認が取れていない controlKey への PUT はガードレールで拒否する。

### 3.11.5 v0 固有の注意

- `ReadValue == "N/R"` は無応答。v1 の 502/504 相当として扱う。
- `ErrorType` / `ErrorMessage` が非空なら失敗。HTTP ステータスは 200 のまま返る可能性があるため、**HTTP 200 を成功と見なさない**。
- レスポンスの JSON 形が仕様書に無い（`Result` の記述のみ）。最初の GET で実際の形を確認し、`E11V0Backend` のパーサを書く。
- v0 は BEMS も使っている経路の可能性が高い。LIVE 実験時の BEMS 停止は v0 でも同様に必要。

### 3.11.6 部屋コード一覧（v0 仕様より。ES エリアコードの正式定義）

| コード | 名称 | | コード | 名称 |
|---|---|---|---|---|
| HALL1 | エントランスホール | | POC1〜3 | 実証実験室１〜３ |
| CWS1 | コワーキングスペース | | CMR1 | 前室中央監視室 |
| STG1 | 倉庫 | | SVR1 | サーバー室 |
| COR1 | 廊下 | | EVH1 | EV ホール |
| EXT1 | 外構 | | MTG1〜2 | 会議室１〜２ |
| KIT1 | 給湯室 | | ROOF1 | 屋上 |
| LAB1〜3 | 共同研究実験室１〜３ | | ATR1 | 階段吹抜け |
| EPSxL1 | x階 EPS室 照明分電盤 | | EPSxP1 | x階 EPS室 動力分電盤 |

設備コードには `WHMETERnnn`（電力量計）が追加。
【未確定】ES の電力ポイント `E11F01EPS1LIGHT1` / `E11F01EPS1MAIN1` は v0 の `EPS1L1` / `EPS1P1` と命名が異なる。ES 側は別の整形を経ている可能性があり、`scripts/build_points.py` の突合結果で確認する。

### 3.11.7 確認事項

**確定済み**: ベース URL `http://192.168.13.10:8080`、認証なし（NW 制限のみ）、`controlstop` は `0`=停止 / `1`=有効（v1 と同じ極性）

**残り【未確定】**
1. GET 応答の実際の JSON 形
2. 各 controlKey の値の型・enum（`Type` / `RangeLo` / `RangeHi` から）と v1 enum との一致
3. 空調の停止方法（`power` か `mode=0` か）
4. ES の `EPS1LIGHT1` / `EPS1MAIN1` と v0 の `EPSxL1` / `EPSxP1` の対応

1〜2 は `scripts/check_api.py --backend v0` で GET のみで確認できる。3 は `HVAC01/{deviceId}/commands` のキー一覧に `power` があるかで判断する。

## 3.10 確認事項まとめ

### 確定済み
- ベース URL は `http://openapi.i4s.uec.ac.jp/api/v1/e11`（暫定。環境変数で外出し）
- 実制御の実験時は BEMS を停止する（本アプリが唯一の制御主体）
- API キーは SEP Portal（`portal.sep.i4s.uec.ac.jp`、学内アカウント認証）で発行。GET / PUT / GET+PUT、有効期限、スコープを指定可 → READ 用と WRITE 用の2本運用

- **v1 は不具合修正中で現時点では実行不可**。復旧まで v0 バックエンド（3.11）で検証する

### 残り【未確定】
0. v1 の復旧時期
1. 本アプリ用キーを発行するアカウント（個人 / 共用）
2. Development サーバの実体（実機 / モック）
3. HTTP のままか、Gateway で TLS 終端するか
4. PUT の同期・非同期と反映遅延
5. `brightness` 設定時の `power` の挙動
6. センサー系 ulid が device-info に登録されているか
7. レート制限の有無（Gateway 側）
8. BEMS 停止・再開の手順と担当
9. 初回実験の対象エリア
10. 機器種別ごとの安全状態の具体値

4〜7 は API キーが手に入れば Claude Code に検証スクリプトを書かせて埋められる。8〜10 は運用側で決める。

