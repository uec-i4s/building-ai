# フェーズ1 無人実行手順

> 配置先: `docs/tasks/phase1_unattended.md`
> 目的: 人間が席を外している間に、Claude Code がフェーズ1（Task 0〜7）を確認なしで完走する。

## 1. なぜ無人実行してよいか

- フェーズ1は **GET のみ・DRY_RUN**。PUT 系のメソッドはこのフェーズでは実装自体を作らない（phase1.md Task 5）。
- `.env` には READ 用の鍵しか置かない。`E11_API_KEY_WRITE` は空。
- enexus-agent は本プロジェクト専用 VM で、壊れても再構築できる。
- `.claude/settings.json` の deny ルールで、`.env` の読み書き、`curl` による PUT、`git push`、破壊的コマンドを機械的に止める。

## 2. 事前準備（人間がやること。5分）

```bash
ssh enexus-agent
cd ~/building-ai

# (a) .env を用意。WRITE キーは空のまま
cp .env.example .env
nano .env        # ES_USER / ES_PASSWORD / E11_API_KEY_READ / LLM_API_KEY を記入。CONTROL_MODE=DRY_RUN, DEVICE_BACKEND=v0 を確認

# (b) 権限設定を配置
mkdir -p .claude
cp docs/tasks/settings.json .claude/settings.json    # ← 本ファイルと一緒に配布した settings.json

# (c) sudo がパスワード無しで通ることを確認（Task 0 の apt install に必要）
sudo -n true && echo OK

# (d) Claude Code が動くことを確認
claude --version

# (e) bypass モードの初回警告を一度だけ対話で受諾しておく（無人実行では警告が出せないため）
claude --dangerously-skip-permissions
#   → 警告ダイアログを受諾 → /exit で抜ける
```

## 3. 実行

tmux 内で起動し、ログをファイルにも残す。SSH が切れてもセッションは生き続ける。

```bash
tmux new -s phase1
cd ~/building-ai
claude --dangerously-skip-permissions -p "$(cat docs/tasks/phase1_unattended_prompt.txt)" \
  --output-format text 2>&1 | tee -a logs/phase1_$(date +%Y%m%d_%H%M).log
```

`Ctrl-b d` でデタッチ。戻る時は `tmux attach -t phase1`。

`--dangerously-skip-permissions` は全ての確認を省略するフラグ。`.claude/settings.json` の deny ルールは bypass モードでも効くため、`.env` 読み取りや PUT はブロックされる（詳細は Claude Code のドキュメント https://code.claude.com/docs/en/permission-modes を参照）。

## 4. 戻った時の確認

1. `docs/tasks/phase1_report.md` を読む（Claude Code が最後に書く完了報告）
2. `git log --oneline` で Task ごとのコミットがあるか
3. `docker compose ps` で3サービスが up か → ブラウザで `http://<enexus-agent>/` を開く
4. `docs/spec_*.md` への変更提案が `phase1_report.md` に列挙されているので、内容を見て採否を決める（無人実行では spec を直接編集させない）
5. `logs/phase1_*.log` で止まった箇所・エラーを確認

## 5. 途中で止まった場合

`claude --dangerously-skip-permissions -p "docs/tasks/phase1_report.md と phase1.md を読み、未完了の Task から再開してください。ルールは phase1_unattended_prompt.txt と同じです。"` で再開できる。

---

# Claude Code への指示文

以下を `docs/tasks/phase1_unattended_prompt.txt` として保存する。

```
docs/tasks/phase1.md の Task 0 から Task 7 までを、人間の確認を待たずに順番に完走してください。
私は席を外しており、質問には答えられません。

## 無人実行中の特別ルール（CLAUDE.md・phase1.md の「確認を取る」より優先）

1. 「実行前に確認を取る」と書かれている箇所は、確認せずに進めてよい。ただし対象はフェーズ1の範囲（GET・DRY_RUN・setup.sh の apt install）に限る。
2. 質問が生じたら、最も安全で保守的な選択肢を採り、その判断を docs/tasks/phase1_report.md に「判断ログ」として記録して進める。
3. docs/spec_*.md は編集しない。調査スクリプトの結果から spec に反映すべき内容は、phase1_report.md に「spec 更新提案」として書く。
4. .env は読まない・書かない。設定値が必要なら .env.example の変数名を使い、値は実行時に環境から読む。
5. PUT・書き込み系のコードは書かない（phase1.md Task 5 の通り、DeviceBackend に PUT メソッドを定義しない）。curl でも PUT は打たない。
6. CONTROL_MODE / DEVICE_BACKEND / 鍵の値を変更しない。
7. 学内 LLM への検証は scripts/check_llm.py の5項目のみ。ベンチマークや並列負荷は行わない。
8. git push はしない。ローカルコミットのみ。
9. 外部サービス（ES / v0 API / vLLM）に到達できない・認証に失敗する場合は、その Task をスキップして理由を報告し、次の Task に進む。3回リトライして駄目なら諦める。
10. 同じエラーで3回以上ループしたら、その Task を「未完了」として報告に残し、次の Task に進む。無限に粘らない。

## 進め方

- Task ごとに: 着手 → 実装 → テスト → コミット（英語、type(scope): summary）→ phase1.md のチェックボックス更新。
- 各 Task の所要時間と結果を phase1_report.md に追記していく（最後にまとめて書くのではなく、Task 完了ごとに追記）。
- Task 2・5・6 の調査スクリプトの出力は、そのまま docs/tasks/artifacts/ に保存する（例: inspect_es_result.md, check_api_v0_result.md, check_llm_result.md）。
- Task 5 の v0 バックエンドは、最初の GET の生レスポンスを docs/tasks/artifacts/v0_raw_response.json に保存してからパーサを書く。
- Task 7 のダッシュボードは、実データが取れなければモックデータで画面だけ作り、その旨を報告する。

## 完了時

docs/tasks/phase1_report.md を以下の構成で完成させる:
1. サマリ（完了した Task / 未完了の Task と理由）
2. 判断ログ（確認なしで決めた事項とその根拠）
3. spec 更新提案（spec_datasource.md 2.8, spec_api.md 3.10・3.11.7, spec_llm.md 4.5 の未確定項目ごとに、判明した値と根拠）
4. 起動方法と確認手順
5. 次に人間が判断すべきこと

最後に git log --oneline の出力を報告に含めて終了してください。
```
