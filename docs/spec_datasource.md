# 2. データソース（Elasticsearch）

> 本節は spec.md の一部。【未確定】と記した項目は確認後に埋めること。
> Claude Code への注意：【未確定】の項目は推測で実装せず、ユーザに確認するか環境変数で外出しにする。

## 2.1 接続情報

| 項目 | 値 |
|---|---|
| Kibana（人間用 UI） | `http://elastic.i4s.uec.ac.jp:5601/` |
| Elasticsearch REST エンドポイント（アプリ用） | `https://elastic.i4s.uec.ac.jp:9200/`（確定。HTTPS） |
| 認証方式 | Basic 認証（ID / password） |
| 認証情報の保管 | `.env` の `ES_URL` / `ES_USER` / `ES_PASSWORD`。リポジトリにコミットしない |
| Elasticsearch バージョン | 8.11.0（確定） |
| enexus-agent からの到達性 | あり（確定） |
| アクセス権限 | 読み取り専用で十分（本アプリは ES に書き込まない。書き込むなら別 index を新設） |

### TLS 証明書の扱い【未確定：CA 証明書を配置するか、検証を無効にするか】

9200 は HTTPS（確定）。http で接続すると `curl: (52) Empty reply from server` になる。
自己署名証明書の場合、CA 証明書（ES サーバの `config/certs/http_ca.crt`）を enexus-agent に配置するのが正。
入手できるまでは開発時に限り検証無効（`ES_VERIFY_CERTS=false`）で進める。

```bash
# 疎通・index 確認（-k は自己署名を許容。CA 配置後は --cacert に置き換える）
curl -sk -u "$ES_USER:$ES_PASSWORD" "https://elastic.i4s.uec.ac.jp:9200/"
curl -sk -u "$ES_USER:$ES_PASSWORD" "https://elastic.i4s.uec.ac.jp:9200/_cat/indices/e11_*?v"
curl -sk -u "$ES_USER:$ES_PASSWORD" "https://elastic.i4s.uec.ac.jp:9200/device-info/_count"
openssl s_client -connect elastic.i4s.uec.ac.jp:9200 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer
```

アプリ側の設定（Python `elasticsearch` クライアント 8.11 系を使用）:

```
ES_URL=https://elastic.i4s.uec.ac.jp:9200
ES_USER=...
ES_PASSWORD=...
ES_CA_CERT=/path/to/http_ca.crt   # 自己署名なら ES サーバから取得して配置。無ければ ES_VERIFY_CERTS=false（開発時のみ）
```

Kibana が http:5601 で動いているなら、Kibana → ES 間は証明書検証を緩めている可能性が高く、`http_ca.crt` は ES サーバの `config/certs/` にある。

## 2.2 index 一覧

| index | 用途 | 粒度 | ドキュメント形式 | 備考 |
|---|---|---|---|---|
| `device-info` | 機器台帳（センサー・アクチュエータ） | 1機器 = 1ドキュメント | 固定スキーマ | `ulid` が機器の一意 ID。制御 API に渡すキー |
| `e11_sens-{yyyy}` | 環境センサー（温度・湿度・CO2・放射温度） | 1時刻 = 1ドキュメント（全ポイント横持ち） | ワイド形式 | 年ごとにローテーション。検索は `e11_sens-*` |
| `e11_sens-{yyyy}` | 電力量（同一 index） | 1時刻 = 1ドキュメント（全ポイント横持ち） | ワイド形式 | 環境センサーとは別ドキュメント。フィールド接尾辞 `_powerconsumption` で判別 |
| `e11_hds_detection-{yyyy}` | 人感センサー | 1状態変化 = 1ドキュメント（ポイント1つだけ） | ナロー形式 | **値変化時のみ発火（確定）**。現在状態は「各ポイントの最新1件」で決まる。長時間変化がないポイントは古いドキュメントまで遡る必要がある |

### 共通メタデータ（全 index に付与）

Filebeat（MQTT input）経由で投入されており、以下が全ドキュメントに付く。

```
@timestamp              ISO8601 UTC（ES 投入時刻）※検索・範囲指定にはこちらを使う
timestamp               "2026/09/08 01:32:47.350"（JST、機器側の送信時刻、文字列）
mqtt.topic              "/E11/SENS/" or "/E11/HDS/"
custom.index_name       "e11_sens" / "e11_hds_detection"
custom.meta.site        "E11"
custom.meta.sensor_domain "environment"
custom.meta.value_schema  "mixed" / "boolean"
message                 元の MQTT ペイロード（JSON 文字列）。フィールド数が多いと message.keyword は _ignored される
agent.* / host.* / ecs.* / input.*   Filebeat 由来。アプリでは無視
```

## 2.3 ポイント ID 命名規則（最重要）

環境センサー・電力・人感のフィールド名、および device-info の `param1` は共通の命名規則に従う。

```
E11 F01 CWS1 IAQMONITOR009 _co2
│   │   │    │              └ 計測項目（measurement）
│   │   │    └ 機器種別 + 連番（device type + no.）
│   │   └ エリアコード + 連番（area）
│   └ 階（floor: F01/F02/F03）
└ 建物（building）
```

正規表現（暫定）:

```
^(?P<building>E\d{2})(?P<floor>F\d{2})(?P<area>[A-Z]+\d)(?P<device>[A-Z]+\d*)?_(?P<measurement>[a-z]+)$
```

例:

| フィールド名 | building | floor | area | device | measurement |
|---|---|---|---|---|---|
| `E11F01CWS1IAQMONITOR009_co2` | E11 | F01 | CWS1 | IAQMONITOR009 | co2 |
| `E11F01CWS1TRAD003_radtemp` | E11 | F01 | CWS1 | TRAD003 | radtemp |
| `E11F01CWS1HDS013_detection` | E11 | F01 | CWS1 | HDS013 | detection |
| `E11F02LAB1LIGHT1_powerconsumption` | E11 | F02 | LAB1 | LIGHT1 | powerconsumption（パルス値） |
| `E11F01CWS1_powerconsumption` | E11 | F01 | CWS1 | （なし＝エリア合計） | powerconsumption |

### 機器種別コード（サンプルから観測）

| コード | 意味 | 計測項目 |
|---|---|---|
| `IAQMONITOR` | 室内空気質モニタ | `temp`(℃), `humidity`(%), `co2`(ppm) |
| `TRAD` | 放射温度センサー | `radtemp`(℃) |
| `WHMETER` | 電力量計（v0 仕様に記載。ES では未観測） | `powerconsumption` |
| `HDS` | 人感センサー | `detection`(bool) |
| `AIRCON` | 空調機 | `powerconsumption` |
| `ERV` | 全熱交換器（換気） | `powerconsumption` |
| `LIGHT` | 照明系統 | `powerconsumption` |
| `MAIN` | 主幹（エリア電源） | `powerconsumption` |
| （なし） | エリア合計 | `powerconsumption` |

`powerconsumption` は **積算電力量計の累積パルスカウント値（確定）**。`× 0.1` で kWh に換算する。

既存 BEMS での「本日の使用電力量」の算出方法（確定）:

```
本日kWh = (count[now] - count[本日 00:00 JST 時点]) × 0.1
```

本アプリでも同じ定義を採用する。汎用化すると:

```
kWh(区間)  = (count[t2] - count[t1]) × 0.1
kW(平均)   = kWh(区間) ÷ (t2 - t1)[h]
```

実装上の注意:
- 「00:00 時点のカウンタ値」は `@timestamp >= 当日00:00 JST` で昇順 `size:1` を取る（JST 基準。UTC では前日 15:00）。
- 区間の始点・終点でドキュメントが欠落している場合は最近傍のドキュメントを使い、その旨を結果に含める。
- 負の差分が出た場合（カウンタリセット・機器交換など）は 0 として扱わず、異常値としてログに記録し UI に表示する。ラップアラウンドの上限値は【未確定】。
- エリア合計ポイント（例 `E11F01CWS1_powerconsumption`）と機器別ポイントの和は一致しない可能性がある（計測系統が別）。UI ではエリア合計を正とする。

### エリアコード（確定。正式定義は基幹 API v0 仕様 → spec_api.md 3.11.6）

| コード | 階（観測） | 正式名称 | UI 表示名（案） |
|---|---|---|---|
| HALL1 | F01 | エントランスホール | エントランスホール |
| CWS1 | F01, F02 | コワーキングスペース | コワーキング |
| COR1 | F01, F02 | 廊下 | 廊下 |
| STG1 | F01 | 倉庫 | 倉庫 |
| EXT1 | — | 外構 | 外構 |
| KIT1 | — | 給湯室 | 給湯室 |
| LAB1 / LAB2 / LAB3 | F02 | 共同研究実験室１〜３ | 共同研究室１〜３ |
| POC1 / POC2 / POC3 | F02, F03 | 実証実験室１〜３ | 実証実験室１〜３ |
| CMR1 | F02 | 前室中央監視室 | 中央監視室 |
| SVR1 | F02 | サーバー室 | サーバー室 |
| EVH1 | — | EV ホール | EV ホール |
| MTG1 / MTG2 | F03 | 会議室１〜２ | 会議室１〜２ |
| ROOF1 | — | 屋上 | 屋上 |
| ATR1 | — | 階段吹抜け | 吹抜け |
| EPSxL1 / EPSxP1 | F01〜F03 | x階 EPS室 照明分電盤 / 動力分電盤 | EPS（照明）/ EPS（動力） |

「—」は ES サンプルで未観測。`scripts/build_points.py` の実行結果で埋める。
【未確定】ES では `E11F01EPS1LIGHT1` / `E11F01EPS1MAIN1` と表記され、v0 の `EPS1L1` / `EPS1P1` と異なる。突合結果で対応を確認する。
この表を `config/areas.yaml` の初期値とし、正規表現の `area` グループは `[A-Z]+\d` に加えて `EPS\d[LP]\d` も受理する。

### ポイント台帳（points.csv）の方針

- 手作業の CSV を正とせず、**ES の実データからフィールド名を収集して自動生成**する（`scripts/build_points.py`）。
- 人手で管理するのはエリアコード→日本語名の対応表（`config/areas.yaml`）のみ。
- 生成した台帳を `data/points.csv` としてリポジトリに置き、UI の「センサー数（種別ごと・階ごと）」はここから集計する。

```
point_id,building,floor,area,device_type,device_no,measurement,unit,index,device_ulid
E11F01CWS1IAQMONITOR009_co2,E11,F01,CWS1,IAQMONITOR,009,co2,ppm,e11_sens,
E11F01HALL1AIRCON001_powerconsumption,E11,F01,HALL1,AIRCON,001,powerconsumption,pulse(x0.1=kWh),e11_sens,01JBGZV7HVYW87FT1C0YT34TMB
```

`device_ulid` は device-info の `param1` の `/` 以降（例 `E11F01HALL1AIRCON001`）と point_id の接頭辞を突合して埋める。

## 2.4 device-info のスキーマ

```json
{
  "Device Category": "Actuator",          // "Actuator" | "Sensor"（確定）
  "Device_type": "Air Conditioner",       // 確定: Light, Air Conditioner, Ventilation, HDS-Controlstop, Displaywall, Elevator, Patlite
  "Description": "Control an air conditioner.",
  "Location": "UEC", "Region": "East", "Building": "E11",
  "Area": "エントランスホール",           // 日本語エリア名（エリアコードとの対応表の元ネタ）
  "Device": "ACP-1-1",                    // 設備側の呼称
  "param1": "HVAC01/E11F01HALL1AIRCON001", // "<コントローラ>/<ポイント接頭辞>" → sens index との結合キー
  "param2": "ー", "param3": "ー",
  "ulid": "01JBGZV7HVYW87FT1C0YT34TMB",   // ★ 機器の一意 ID。制御 API に渡す
  "point": [139.54321, 35.65824],         // [lon, lat]
  "altitude": "0",
  "API": "TBF",                           // 【未確定】制御 API の定義先。現状 "TBF"（To Be Filled）
  "access_log_url": "TBF", "error_log_url": "TBF",
  "map": "...", "GoogleEarth": "...",
  "administrator": "..."
}
```

### Device_type と本アプリの担当エージェント

| Device_type | 担当エージェント | 本アプリでの扱い |
|---|---|---|
| Light | 照明 | 制御対象 |
| Air Conditioner | 空調 | 制御対象 |
| Ventilation | 換気 | 制御対象 |
| HDS-Controlstop | （共通） | 人感連動制御の停止スイッチ（確定）。ON の間は本アプリのポリシーも人感連動を発動させない＝**手動優先のガードレール入力**として扱う |
| Displaywall | 対象外 | 一覧には表示するが制御しない |
| Elevator | 対象外 | 同上。**安全上、本アプリから絶対に制御しない**（CLAUDE.md にも明記） |
| Patlite | 通知用途（将来） | 警告灯（確定）。**現時点で e-Nexus には未設置**。将来設置されればガードレール発動・ロールバック時の物理通知先に採用。フェーズ4の通知インターフェースは Patlite を差し込める抽象（`Notifier`）にしておく |

アプリ起動時に `device-info` を全件取得（`size: 1000` で十分と推定、【未確定】総件数）し、`ulid` ⇔ `param1` ⇔ point_id の対応表をメモリに持つ。
制御対象は `Device Category == "Actuator"` かつ `Device_type ∈ {Light, Air Conditioner, Ventilation}` に限定し、それ以外はホワイトリスト外として制御 API 呼び出しを拒否する。

## 2.5 代表的なクエリ

### (a) 環境センサーの最新値（全ポイント）

```json
POST /e11_sens-*/_search
{
  "size": 1,
  "sort": [{"@timestamp": "desc"}],
  "query": {"exists": {"field": "E11F01CWS1IAQMONITOR009_temp"}},
  "_source": {"excludes": ["message", "agent", "host", "ecs", "input", "mqtt"]}
}
```

環境センサーと電力は同じ index の別ドキュメントなので、`exists` で代表フィールドを指定して振り分ける。

### (b) 電力の最新値（全ポイント）

同上、`exists` を `E11F01EPS1MAIN1_powerconsumption` 等に変更。

### (c) 人感センサーの現在状態

ナロー形式なので「各ポイントの最新1件」を集約で取る。

```json
POST /e11_hds_detection-*/_search
{
  "size": 0,
  "query": {"range": {"@timestamp": {"gte": "now-1h"}}},
  "aggs": {
    "by_point": {
      "terms": {"field": "custom.index_name", "size": 1},   // ← プレースホルダ
      "aggs": {"latest": {"top_hits": {"size": 1, "sort": [{"@timestamp": "desc"}]}}}
    }
  }
}
```

【設計メモ】人感はフィールド名がポイントごとに異なるため terms 集約が素直に書けない。
選択肢: (1) ポイントごとに `exists` クエリを N 回投げる、(2) `now-Xmin` の全件を取ってアプリ側で最新値を畳む、
(3) 投入側で `point_id` / `value` の共通フィールドを追加してもらう。フェーズ1は (2) で実装し、(3) を要望として記録。

### (d) 時系列（リプレイ検証用）

```json
POST /e11_sens-*/_search
{
  "size": 10000,
  "sort": [{"@timestamp": "asc"}],
  "query": {"bool": {"filter": [
    {"range": {"@timestamp": {"gte": "2026-09-01T00:00:00Z", "lt": "2026-09-08T00:00:00Z"}}},
    {"exists": {"field": "E11F01CWS1IAQMONITOR009_temp"}}
  ]}},
  "_source": ["@timestamp", "E11F01CWS1IAQMONITOR009_temp", "E11F01CWS1IAQMONITOR009_co2"]
}
```

10,000 件超は `search_after` または `date_histogram` 集約で間引く。

## 2.6 サンプリング間隔・データ量（【未確定】要計測）

| index | 観測からの推定 | 確認方法 |
|---|---|---|
| e11_sens（環境） | 約2分間隔?（サンプル2件が 16:30:48 / 16:32:48） | `date_histogram` で1日分を数える |
| e11_sens（電力） | 同上 | 同上 |
| e11_hds_detection | 値変化時のみ（確定） | 1日の件数のみ確認（負荷見積り用） |
| 保持期間 | 【未確定】 | `_cat/indices` で過去年の index 有無 |

## 2.7 データ品質メモ

- 湿度が 83〜88% と高め（9月の雨天なら妥当だが、校正ズレの可能性も）。UI に「異常値フラグ」の仕組みを入れる余地あり。
- `timestamp`（機器側 JST）と `@timestamp`（投入時 UTC）に約 1 秒のズレ。アプリ内は `@timestamp` に統一し、表示時に JST 変換。
- 電力ドキュメントの `message` にはフィールド化されていないポイント（例 `E11F01EPS1LIGHT1`, `E11F01HALL1`）が含まれる場合がある。
  【未確定】`_source` 側にも全て存在するか、フィールド数上限で落ちていないか（`index.mapping.total_fields.limit`）を確認。

## 2.8 確認事項まとめ

### 確定済み
- REST エンドポイント `https://elastic.i4s.uec.ac.jp:9200/`、ES 8.11.0、enexus-agent から到達可
- 人感センサーは値変化時のみ送信
- `powerconsumption` は累積パルスカウンタ、×0.1 で kWh。日次は 00:00 JST との差分
- エリアコードの日本語名
- device-info の `Device Category`（Actuator / Sensor）と `Device_type`（7種）
- `HDS-Controlstop` = 人感連動制御の停止スイッチ、`Patlite` = 警告灯（e-Nexus には未設置、将来の通知先候補）

### 残り【未確定】
1. TLS 証明書の扱い（`http_ca.crt` を入手して配置するか、開発中は検証無効か）
2. 累積カウンタのラップアラウンド上限
3. device-info の総件数
4. e11_sens のサンプリング間隔と各 index の保持期間
5. 電力ドキュメントで `message` にあって `_source` に無いポイントがあるか（total_fields.limit）
6. 制御 API の仕様（device-info の `API` フィールドは "TBF"）→ spec.md の別節で扱う
7. `HDS-Controlstop` の状態を ES から読めるか（device-info にはあるが、状態値がどの index に入るかは未確認）

3〜5 は Claude Code に調査スクリプト（`scripts/inspect_es.py`）を書かせて自動で埋める。
