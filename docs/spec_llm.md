# 4. 学内 LLM（Qwen 3.5 on vLLM）

> 本節は spec.md の一部。【未確定】と記した項目は確認後に埋めること。
> Claude Code への注意：LLM は学内サーバの共用資源。ベンチマークや大量リクエストを投げる前にユーザに確認する。

## 4.1 基本情報

| 項目 | 値 |
|---|---|
| フロントエンド（人間用） | Open WebUI `http://genai.i4s.uec.ac.jp:8080/` |
| 推論サーバ | vLLM（`vllm serve`）。OpenAI 互換 API |
| vLLM エンドポイント | `http://genai.i4s.uec.ac.jp:8000/v1`（ポート確定。enexus-agent から疎通可） |
| Open WebUI API エンドポイント | `http://genai.i4s.uec.ac.jp:8080/api`（OpenAI 互換。`/api/chat/completions`, `/api/models`） |
| アプリからの接続経路 | **B. vLLM 直結（確定）** `http://genai.i4s.uec.ac.jp:8000/v1` |
| モデル ID（`model` パラメータ） | `Qwen/Qwen3.5-122B-A10B-FP8` |
| 認証 | Bearer トークン。`Authorization: Bearer <key>`。鍵は `vllm serve --api-key` の値（確定） |
| 認証情報の保管 | `.env` の `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL` |

### 4.1.1 接続経路と鍵（確定）

**vLLM 直結**を採用。Open WebUI（:8080）は人間がプロンプトを試す UI としてのみ使い、アプリは経由しない。

- 鍵は `vllm serve --api-key` の固定値をアプリと共有する。vLLM 側で鍵をローテーションした場合は `.env` の `LLM_API_KEY` を更新して再起動する。
- 機器制御 API キー（SEP Portal 発行）とは完全に別物。`.env` でも `E11_API_KEY_*` と `LLM_API_KEY` を混同しない。
- 将来 Open WebUI 経由や別モデルに切り替える可能性に備え、`LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL` の差し替えだけで済むよう経路依存のコードは書かない。

起動スクリプト（`~/vllm/run_vllm_qwen35.sh`、参考）:

```
vllm serve Qwen/Qwen3.5-122B-A10B-FP8
  --enable-auto-tool-choice --tool-call-parser qwen3_coder
  --tensor-parallel-size 2
  --max-model-len 262144 --max-num-seqs 512
  --reasoning-parser qwen3
  --disable-uvicorn-access-log --api-key {APIKey}
```

疎通確認:

```bash
curl -s -H "Authorization: Bearer $LLM_API_KEY" "$LLM_BASE_URL/models"
curl -s -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  "$LLM_BASE_URL/chat/completions" \
  -d '{"model":"Qwen/Qwen3.5-122B-A10B-FP8","messages":[{"role":"user","content":"こんにちは"}],"max_tokens":64}'
```

## 4.2 起動オプションから読み取れる能力と制約

| オプション | 意味 | 本アプリへの影響 |
|---|---|---|
| `Qwen3.5-122B-A10B-FP8` | MoE（総パラメータ 122B、アクティブ 10B）、FP8 量子化 | 応答速度は 10B クラス相当で軽快。3エージェントの往復会話に耐える |
| `--enable-auto-tool-choice` + `--tool-call-parser` | OpenAI 形式の **function calling が使える** | エージェントの Tool（ES 検索、機器状態取得、制御コマンド発行）を `tools=[...]` で渡せる。**エージェント設計の前提が成立** |
| `--reasoning-parser qwen3` | 思考過程（thinking）を `reasoning_content` として本文と分離して返す | UI で「妖怪の思考中…」として思考過程を見せられる。通常応答から thinking を除外できるので出力が汚れない |
| `--max-model-len 262144` | コンテキスト 256K トークン | 1日分のセンサー要約や過去ポリシー履歴をまるごとプロンプトに入れられる。ただし長いほど遅いので実運用は 32K 以内を目標にする |
| `--max-num-seqs 512` | 同時 512 シーケンス | 並列度は十分。3エージェント同時呼び出しも問題なし |
| `--tensor-parallel-size 2` | GPU 2枚 | 共用サーバ。他利用者と帯域を分け合う |
| `--api-key` | 単一キー | 【未確定】アプリ用に別キーを発行できるか、同じキーを共用するか |

【未確定・要検証】
- `--tool-call-parser qwen3_coder` が Qwen3.5 で安定して動くか（Qwen3 系は `hermes` / `qwen3_xml` 等の選択肢がある）。**並列 tool call（1応答で複数 tool を呼ぶ）** が正しくパースされるかを最初に検証する。
- thinking の ON/OFF 切り替え方法。Qwen3 系は `extra_body={"chat_template_kwargs": {"enable_thinking": false}}` で無効化できるのが通例。会話 UI では ON、ポリシー生成の構造化出力では OFF にしたい。
- 構造化出力（`response_format: {"type": "json_schema", ...}`）が有効か。vLLM は guided decoding をサポートしているが、起動オプションで明示されていないのでデフォルト設定を確認。
- Embedding モデルは起動されていない → RAG（過去の判断根拠の検索等）が必要になったら別途 embedding サーバが要る。フェーズ2までは不要な設計にする。

## 4.3 本アプリでの利用方針

### 接続はどこから

```
enexus-agent (VM#2)  ──http──▶  genai.i4s.uec.ac.jp:8000 (vLLM, OpenAI 互換 API)
```

### クライアント

- Python `openai` SDK（`base_url` を差し替えるだけ）。フレームワークは **LangGraph**（エージェント間の会話・承認待ちなどの状態遷移を明示的に書ける）。
- モデル固有の挙動（thinking 切替、tool parser の癖）は `llm/qwen_adapter.py` の1ファイルに閉じ込め、将来モデルが変わっても他を触らずに済むようにする。

### 用途ごとの呼び出し設定

| 用途 | thinking | tools | 出力形式 | temperature |
|---|---|---|---|---|
| エージェント間会話（妖怪同士の議論） | ON | ES 検索、機器状態取得 | 自由文（日本語） | 0.7 |
| ポリシー案生成 | OFF | なし（会話の結論から生成） | JSON Schema（`Policy` 型） | 0.2 |
| 人間との対話チャット | ON | ES 検索、機器状態取得、ポリシー参照 | 自由文 | 0.7 |
| リプレイ検証結果の解説 | OFF | なし | 自由文 | 0.3 |

**制御コマンドの発行は LLM の tool として直接は渡さない。** LLM が生成するのは構造化された `Policy`（ルール記述）であり、実際の PUT はポリシーエンジン → ガードレール → DeviceController が行う（spec_api.md 3.6）。これにより「LLM が勝手に機器を動かす」経路を構造的に無くす。

### 3エージェントの構成

| エージェント | 担当 | 参照するデータ | 生成するポリシー |
|---|---|---|---|
| 照明 | `lights`, `lighting-interlocks` | 人感（occupancy）、電力（LIGHT 系統）、時刻 | 点灯 / 消灯 / 調光率 / 人感連動 ON/OFF |
| 空調 | `aircons` | 温度、放射温度、湿度、人感、電力（AIRCON 系統）、外気【未確定: 外気温データの有無】 | モード / 設定温度 / 風量 |
| 換気 | `ventilators` | CO2、人感、電力（ERV 系統） | 運転状態 / 換気モード / 風量 |

3体は同じモデル・同じエンドポイントで、system prompt（役割・性格・参照可能データ・制約）だけが異なる。会話は LangGraph 上で「司会役（コーディネータ）」が順番に発言させ、合意形成後にポリシー案を出す。

## 4.4 障害時の振る舞い

LLM サーバは共用・実験用のため、停止やメンテナンスが前提。

| 状況 | 振る舞い |
|---|---|
| LLM 応答なし / タイムアウト（30 秒） | エージェント会話を中断し UI に表示。**承認済みの現行ポリシーはそのまま動き続ける**（ポリシーエンジンは LLM に依存しない） |
| LLM が不正な JSON を返す | 1回だけ再試行。それでも失敗ならポリシー案は「生成失敗」として記録し、現行ポリシーを維持 |
| LLM 長期停止 | DRY_RUN / LIVE いずれもポリシーエンジンは継続。新規ポリシー提案だけが止まる |

「機能3（未来予測により安全性を保障）」の観点で、**LLM が止まっても建物は安全に動き続ける**ことを設計原則にする。

## 4.5 確認事項まとめ

### 確定済み
- vLLM 直結 `http://genai.i4s.uec.ac.jp:8000/v1`、enexus-agent から疎通可
- 鍵は `vllm serve --api-key` の値。機器制御 API キー（SEP Portal）とは別物

### 残り【未確定】
1. `qwen3_coder` パーサーでの tool call 動作確認（単発・並列）
2. thinking ON/OFF の切り替え方法の動作確認
3. `response_format: json_schema` の動作確認
4. 同時利用者との負荷分担（研究室内での利用ルール、優先度）
5. 外気温・天候データの有無（空調エージェントの判断材料として重要。無ければ気象庁 API 等の外部データを検討）
6. サーバ停止・再起動の連絡経路（実験中の停止を避けるため）

1〜3 は Claude Code に `scripts/check_llm.py` を書かせて一括検証する。
