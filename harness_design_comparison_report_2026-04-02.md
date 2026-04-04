# ハーネス設計案 A / B / C 比較評価レポート

作成日: 2026-04-02
対象:
- `harness_design-A_master.md`
- `harness_design-B_master.md`
- `harness_design-C_master.md`

## 1. 結論

総合判断は以下です。

1. 現時点で最も実運用に近い土台は **C案**
2. 人間の統治設計と責務境界の明快さは **A案** が最も強い
3. Claude Code の機能を最も深く活かせるのは **B案**
4. 実際に採るべきは **C案をベースに、A案の統治モデルとB案の実装規律を合成したハイブリッド案**

理由は明確です。

- あなたが求めているのは、単なるタスク管理ではなく、仕事中や就寝中に実装を回し、終了後に「続行」「やり直し」「新規化」を再判定し、全体状況から次タスク提案まで行う運用です。
- この要件には、単なる役割分担や文書整理だけでは足りず、**機械的に再判定できる状態遷移** と **再トリアージ可能な構造化状態** が必要です。
- その点で、`queue state machine`、`task.yaml`、`blocked` 再走査、`failure-classifier` を持つ C 案が最も要件に近いです。
- ただし C 案だけだと、全体統治の責任分界と、Claude Code の settings / hooks / skills / permissions の厳密設計がやや薄いので、A と B の補完が必要です。

## 2. 比較結果サマリー

5点満点で評価します。

| 評価軸 | A案 | B案 | C案 | コメント |
|---|---:|---:|---:|---|
| 全体統治の分かりやすさ | 5 | 3 | 4 | A は Layer1/2/3 の責務が最も明確 |
| Claude Code 実装適合性 | 2 | 5 | 4 | B は settings / hooks / skills / subagents まで具体化 |
| 夜間・離席中の連続運用 | 3 | 4 | 5 | C は queue と blocked 再処理が強い |
| 起床後・帰宅後の状況把握のしやすさ | 3 | 4 | 5 | C は queue 状態と run 単位ログが見やすい |
| 失敗タスクの再分類能力 | 2 | 3 | 5 | C は `failure-classifier` と `blocked` 運用がある |
| 次タスク提案のしやすさ | 3 | 5 | 5 | B は triage scoring、C は queue 再生成が強い |
| 機械可読性・自動化しやすさ | 2 | 4 | 5 | C の `task.yaml` は強い |
| GitHub 連携のしやすさ | 2 | 5 | 4 | B が最も整理されている |
| ベンダー/モデル変更への耐性 | 5 | 2 | 4 | A は抽象度が高く、C は骨格が移植しやすい |
| 初期導入の現実性 | 3 | 4 | 4 | B/C は導入順序が具体的 |

総合所見:

- **A案**: 戦略とガバナンスは良いが、夜間自律運用の制御面が弱い
- **B案**: Claude Code 向けの運用設計としては非常に強いが、将来のツール変更にはやや弱い
- **C案**: 実行と再判定の運用機械として最も優秀で、今回の要件に最も合う

## 3. 各案の詳細評価

## 3-1. A案の評価

### 強み

- Layer1 / Layer2 / Layer3 の責務が明確
- `status.md`、issue ファイル、`runs/` を使った再開設計が分かりやすい
- `ready_for_close / continue_current_issue / needs_followup / blocked` の判定語彙が整理されている
- 「会話ではなく状態ファイルで継続する」という思想が正しい
- 特定ツールへの依存が薄く、将来のモデルやエージェント変更に強い

### 弱み

- 状態管理が人間向け Markdown 中心で、機械判定には弱い
- `open/closed/_index` と issue 追記だけでは、夜間に大量の試行が走った後の自動再トリアージが重い
- `1 issue = 1 Layer2 = 1 Layer3` は分かりやすい一方、将来の複数サブエージェント協調には硬い
- Claude Code の settings / permissions / hooks / skills への落とし込みが薄い
- 「全体のリアルタイム状況から毎回適切な次タスク提案」を回すには、状態が構造化不足

### あなたの要件への適合

- 仕事中・就寝中に回す: `可`
- 終了後に続行/やり直し/新規を判断する: `半分可`
- 毎回全体状況から適切な次タスク提案をする: `手動色が強い`

判断:

- A案は **人間の司令塔を強く残す運用** には向いています。
- ただし、あなたの要求はもう一段自律度が高いので、A案単独では不足です。

## 3-2. B案の評価

### 強み

- Claude Code の現実機能に最も忠実
- `.claude/settings.json`、`skills`、`subagents`、`hooks`、`worktree` の接続が具体的
- `Triage / Executor / Reviewer` の三役分離が実践的
- `TRIAGE_REPORT.md` と `CANDIDATES.yaml` による候補抽出が強い
- GitHub Projects / Issues / `gh` CLI と相性が良い
- `HANDOFF.md`、`VERIFY.md`、`RUNLOG.md` の役割分離が良い

### 弱み

- Claude Code 固有機能への依存が強く、将来別ツールへ移る時に移植コストが高い
- `task pack` は良いが、状態遷移が C 案ほど厳密な state machine になっていない
- 失敗タスクの分類が明示ルール化されておらず、`retry / continue / new issue` がやや運用依存
- `blocked` の扱いが C 案ほど中心概念になっていない
- 「リアルタイム全体状況」を見て次を選ぶための常時再評価モデルは、triage セッション頼み

### あなたの要件への適合

- 仕事中・就寝中に回す: `かなり可`
- 終了後に続行/やり直し/新規を判断する: `可`
- 毎回全体状況から適切な次タスク提案をする: `可`

判断:

- B案は **今すぐ Claude Code 中心で運用を固める** にはかなり強いです。
- ただし、運用の中核を Claude 固有概念に寄せすぎているため、将来のモデル進化やツール差し替えには弱めです。

## 3-3. C案の評価

### 強み

- `inbox / proposed / ready / running / review / blocked / done` の queue が明快
- `task.yaml` により、状態が Markdown だけでなく構造化されている
- `failure-classifier`、`blocked` 再走査、`proposed` 再生成があり、失敗後の再判定に強い
- `1 task = 1 worktree` と `1 run = 1 folder` が実運用に強い
- `Local-first` から `GitHub-integrated` へ段階的に移行できる
- 将来 Agent SDK や別ツールへ拡張する余地を残している
- 起床後や帰宅後に、「何が終わり、何が止まり、何を次に動かすか」を把握しやすい

### 弱み

- A案ほどの全体統治モデルの説明力はない
- B案ほど Claude Code の hooks / permissions / settings の具体化が厚くない
- queue が増えるぶん、初期導入時の設計とテンプレート整備を雑にすると運用が崩れる
- `real-time` を本当にやるには、日次 triage だけでは足りず、イベント起点の再トリアージが追加で必要

### あなたの要件への適合

- 仕事中・就寝中に回す: `非常に可`
- 終了後に続行/やり直し/新規を判断する: `非常に可`
- 毎回全体状況から適切な次タスク提案をする: `最も可`

判断:

- C案は今回の要件に最も近いです。
- 特に「失敗タスクを整理し直し、次タスク候補を再提案する」という循環に強いです。

## 4. あなたの要件に対する厳密評価

あなたの要件を、運用上は次の5能力に分解できます。

1. 離席中に安全に実装を継続できる
2. 終了後に成果物と失敗を整理できる
3. 失敗を `再試行 / 継続 / 新規タスク化 / 人間判断待ち` に分類できる
4. 全体の現在地から次タスク候補を再計算できる
5. 次に回すと危険なタスクを止められる

この5能力に対する評価は以下です。

| 能力 | A案 | B案 | C案 |
|---|---|---|---|
| 1. 離席中継続 | 中 | 高 | 最高 |
| 2. 終了後整理 | 中 | 高 | 最高 |
| 3. 失敗分類 | 低 | 中 | 高 |
| 4. 次タスク再計算 | 中 | 高 | 高 |
| 5. 危険停止 | 中 | 高 | 高 |

重要な指摘があります。

**A/B/C のどれも、そのままでは「リアルタイム再判定」を完全には満たしていません。**

理由:

- A は人間更新の `_index.md` と issue ベースなので、即時再計算に向かない
- B は triage は強いが、基本的にセッション起点であり、イベントドリブンな状態更新が弱い
- C は最も近いが、日次 triage と blocked 再走査が中心で、`run終了イベント`、`レビュー差し戻し`、`仕様変更投入` のたびに自動再評価する仕組みまでは書かれていない

つまり、本当にあなたの求める運用にするには、C案ベースに次の4点を追加する必要があります。

## 5. 必須追加要素

### 5-1. タスク再分類ルールの明文化

最低でも各 run 終了時に、task を次のどれかへ必ず分類する必要があります。

- `continue_same_task`
- `retry_same_task`
- `split_new_task`
- `blocked_waiting_human`
- `done`

この判定は `task.yaml` か `result.md` に機械可読で残すべきです。

### 5-2. イベント駆動の再トリアージ

再トリアージを「朝だけ」ではなく、次のイベント発生時にも回すべきです。

- run が `success / partial / failure / aborted` で終わった時
- `blocked` が発生した時
- review で差し戻された時
- `inbox` に仕様変更が入った時
- stale `running` が閾値を超えた時

### 5-3. 全体優先度の再計算フィールド

次タスク提案を毎回安定させるため、少なくとも以下の構造化フィールドが必要です。

- `impact`
- `urgency`
- `unblock_value`
- `effort`
- `confidence`
- `risk`
- `human_dependency`
- `last_run_result`
- `next_action_type`

この点は B 案の scoring と C 案の `task.yaml` を組み合わせるのが最適です。

### 5-4. ハーネス内部の adapter 分離

将来のモデルやツール進化に耐えるには、役割そのものを Claude 固有名にしないほうが良いです。

推奨:

- `Triage Agent`
- `Execution Agent`
- `Review Agent`
- `Log Summarizer`

として設計し、Claude / Codex / 将来の agent platform は adapter として差し替える形にすることです。

## 6. 近い将来の性能向上・機能追加への耐性

この観点では順位が少し変わります。

1. **A案**
2. **C案**
3. **B案**

### A案が強い理由

- 概念がツール非依存
- 層構造、正本、再開性、判定フローが抽象化されている
- Claude 以外の AI にも移しやすい

### A案の弱点

- 抽象度が高いため、今すぐ自動運用に使うには追加実装が多い

### B案が弱い理由

- `CLAUDE.md`、Claude の `/commands`、permissions mode、hooks、skills への依存が強い
- 将来別ツールに置き換える時、設計をかなり再解釈し直す必要がある

### C案がバランスが良い理由

- queue、task packet、worktree、runs という骨格はツール非依存
- 一方で Claude Code の現実運用にも落とせる
- 将来、モデルや AI エージェントの性能が上がっても、内部の executor を差し替えればよい

結論として、**将来耐性は A の思想を持ちつつ、実運用の骨格は C に置く** のが最も合理的です。

## 7. 最終推奨: 採るべき設計

### 推奨方針

**C案ベース + A案の統治 + B案のClaude運用部品**

### A案から採るべきもの

- Layer1 / Layer2 / Layer3 の責務分離
- `ready_for_close / continue / needs_followup / blocked` の判定思想
- `status + issue + runs` による再開原則
- 正本を少なく保つ設計

### B案から採るべきもの

- `.claude/settings.json` の中心化
- hooks / skills / subagents の活用
- `HANDOFF.md`、`VERIFY.md`、`RUNLOG.md` の厳密運用
- triage scoring
- GitHub Projects / Issues / `gh` CLI 連携

### C案から採るべきもの

- queue state machine
- `task.yaml` を含む task packet
- `blocked` 再走査
- `failure-classifier`
- `1 task = 1 worktree`
- `1 run = 1 folder`
- Local-first から GitHub-integrated への段階導入

## 8. 実務上の最終判定

### A案をそのまま採用する場合

- ガバナンス重視
- 自律運用は弱い
- 人間が強く管理し続ける前提

向いている状況:

- まず事故なく回したい
- AI の自律度はまだ抑えたい

### B案をそのまま採用する場合

- Claude Code で今すぐ回しやすい
- かなり実践的
- 将来の他ツール展開は弱い

向いている状況:

- 当面は Claude Code 一本で行く
- GitHub と強く結びたい

### C案をそのまま採用する場合

- あなたの要件に最も近い
- 離席中運用と再判定に強い
- 少し設計を足せば本命になれる

向いている状況:

- 夜間/勤務中に流し続けたい
- 次タスク提案まで AI に寄せたい
- 将来別モデルも視野に入れたい

## 9. 最終結論

**最有力は C案です。**

ただし、**そのまま C 案を採用するのではなく、A 案の Layer 統治と B 案の Claude Code 実装規律を必ず混ぜるべきです。**

一言でまとめると、こうです。

- A案は「運用思想の骨」
- B案は「Claude Code で回すための筋肉」
- C案は「夜間・離席中に回り続ける運用機械」

あなたの要件を最も満たすのは、**C を主軸にしたハイブリッド案** です。

## 10. 次に設計へ反映すべき具体項目

優先度順に挙げます。

1. `task.yaml` に再分類フィールドを追加する
2. run 終了時の自動再判定フローを定義する
3. `blocked` 発生時の `unblock task` 自動生成ルールを作る
4. B案の triage scoring を C案の queue に移植する
5. A案の Layer1 責務を C案に重ねる
6. Claude 固有名を role abstraction に置き換える
7. 将来の Codex などに備えて adapter 層を分ける

## 11. 要約

- 今すぐ最も強い土台: `C案`
- 最も抽象設計として良い: `A案`
- 最も Claude Code 実装に強い: `B案`
- 実際に採るべき: `C + A + B` のハイブリッド

この結論なら、今の Claude Code 運用にも耐え、近い将来のモデル性能向上や AI ツール追加にも設計を壊さず追従しやすいです。
