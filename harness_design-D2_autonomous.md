# AI Harness Design D2 — Autonomous Architecture

**作成日**: 2026-04-10
**前提**: Design D の思想を継承し、「プロンプトを書かなくても成立する」設計に進化させる
**方針**: 役割を構造で決定し、人間の介入を最小化し、自律的な並列運転を可能にする

---

# 0. 設計思想

## 0-1. Design D からの継承（不変）

以下の5要素は Design D と同一であり、変更しない。

1. **Queue State Machine** — タスクの状態遷移を機械的に管理する
2. **Task Packet** — 1タスクの全情報を1フォルダに閉じ込める
3. **1 Task = 1 Worktree = 1 Branch** — 実装の物理的隔離
4. **Handoff-first Recovery** — 会話ではなくファイルで継続する
5. **Structured Failure Classification** — 失敗を機械的に分類し再判定する

## 0-2. Design D2 の追加原則

Design D の問題は「構造は正しいが、人間が各フェーズの進行役になっている」ことだった。D2 は以下を追加する。

### 原則 1: 役割は構造から自動決定される

セッションに「あなたは○○です」と伝えない。セッションは以下の3つの情報から自分の役割を認識する。

| 情報源 | 決定されるもの |
|---|---|
| **cwd がどこか** | worktree内 → Executor / メインrepo → Triage or Review |
| **queue のどの状態にタスクがあるか** | ready → 着手可能 / running → 継続 / review → レビュー対象 |
| **permission mode** | plan → 読み取り専用の役割 / acceptEdits → 実装役割 |

これが成立するために、CLAUDE.md に「cwd と queue state から役割を判定するルール」を書く。hooks が強制する。subagent 定義が権限を制約する。プロンプトは不要になる。

### 原則 2: デフォルト行動が定義されている

各役割には「何も指示されなかった場合にやること」が定義されている。セッションが起動し、構造から役割を認識したら、デフォルト行動を開始する。

| 役割 | デフォルト行動 |
|---|---|
| Triage | queue 全状態をスキャンし、TRIAGE_REPORT と CANDIDATES を生成する |
| Executor | running/ にある自分の task packet の HANDOFF.md を読み、「次の1〜3手」から再開する |
| Reviewer | review/ にある task packet の diff と VERIFY.md を読み、REVIEW.md を書く |

### 原則 3: task packet が自己完結している

task packet を読めば「何をすべきか」「どこまで終わったか」「次に何をするか」が分かる。外部の指示は不要。これは以下を意味する:

- BRIEF.md に「なぜ」と「何を」が書かれている
- PLAN.md に「どうやって」が書かれている
- HANDOFF.md に「今どこにいて、次に何をするか」が書かれている
- task.yaml に機械可読なメタデータがある
- definition_of_done に完了判定基準がある

### 原則 4: 自動進行と人間ゲートが明確に分離されている

AIが自律的に進めてよい遷移と、人間の承認が必要な遷移を明確に分ける。

| 遷移 | 実行者 | 理由 |
|---|---|---|
| inbox → proposed | **AI自動** | 構造化と優先度付けは機械的判断 |
| proposed → ready | **人間** | 「今日何を回すか」は経営判断 |
| ready → running | **AI自動** | worktree作成と起動は機械的 |
| running → review | **AI自動** | review-gate の機械的チェック通過で自動遷移 |
| running → blocked | **AI自動** | failure-classifier の判定に基づく |
| review → done | **人間** | merge は最終承認 |
| review → blocked | **AI自動** | unsafe change 検出時 |
| blocked → ready | **人間** | ブロック解消の判断は人間 |
| blocked → ready (auto-unblock) | **AI自動（条件付き）** | 後述の auto-unblock ルールに該当する場合のみ |

### 原則 5: プロンプトは「何をするか」だけで成立する

上記の原則 1〜4 により、ワークフローのプロンプトは以下のレベルまで短縮される:

```
# 最小プロンプト例

## Triage 起動
triage して

## タスク実行
Q-P1-0042 を実行して

## レビュー
review/ にあるタスクをレビューして

## 状況確認
状況を教えて
```

これが成立する理由:
- 「triage して」→ CLAUDE.md のルールで triage-from-state skill が呼ばれ、デフォルト行動が実行される
- 「Q-P1-0042 を実行して」→ task.yaml から worktree path と状態を読み、HANDOFF.md から再開点を取得し、PLAN.md に従って実装する
- 「review/ にあるタスクをレビューして」→ review/ 内の task packet を順に読み、REVIEW.md を書く
- 「状況を教えて」→ queue 状態 + 直近の TRIAGE_REPORT を読み、人間向けサマリーを返す

---

# 1. 全体アーキテクチャ

## 1-1. 4層構造（Design D と同一）

```
┌──────────────────────────────────────────────────────────┐
│  Layer 0: 可視化・追跡層                                   │
│  GitHub Projects / Issues / PR                            │
├──────────────────────────────────────────────────────────┤
│  Layer 1: 長期計画・統治層                                  │
│  docs repo + project docs                                 │
├──────────────────────────────────────────────────────────┤
│  Layer 2: 運用管理層                                       │
│  .ops/ (queue, runs, reports, templates)                   │
├──────────────────────────────────────────────────────────┤
│  Layer 3: 実行層                                          │
│  git worktree + Claude session                            │
└──────────────────────────────────────────────────────────┘
```

## 1-2. 3つのAgent役割（Design D と同一、決定方法が異なる）

```
Human (最終意思決定者)
  │
  ├── Triage Agent — メインrepo + plan mode で自動認識
  ├── Execution Agent — worktree + acceptEdits mode で自動認識
  └── Review Agent — メインrepo + plan mode + review/ 対象で自動認識
```

## 1-3. 人間の責務（Design D から縮小）

**必ずやること（3つだけ）:**
1. proposed を見て ready に昇格させる（「今日何を回すか」を決める）
2. review を見て merge / close / hold を決める
3. blocked を見て判断する（破壊的操作・課金操作・本番操作・仕様確定）

**やらなくていいこと（Design D より拡大）:**
- タスク候補の洗い出し → Triage Agent が自動
- 実装 → Execution Agent が自動
- レビュー初期分析 → Review Agent が自動
- 失敗分類と再試行判断 → failure-classifier が自動
- ready → running の起動 → 自動
- running → review の遷移 → review-gate 通過で自動
- 次タスクの生成 → run 終了時に Triage が自動再評価
- 仕様変更の影響分析 → impact-analyzer が自動
- GitHub Issue/Project の更新 → issue-sync が自動
- 日次レポート生成 → /schedule で自動

---

# 2. Claude Code スコープ設計 — 何をどこに持たせるか

## 2-1. スコープの性質と役割分担

Claude Code には性質の異なる複数のスコープがある。Design D2 ではこれらを明確に使い分ける。

| スコープ | 性質 | 強制力 | 用途 |
|---|---|---|---|
| **settings.json permissions** | client enforced | **強制** | 許可/拒否の境界。違反不可能 |
| **settings.json hooks** | client enforced | **強制** | ライフサイクルの自動処理。必ず実行される |
| **rules/** | context injection | 中（読まれるが無視されうる） | 状況依存のガイダンス。条件付きで注入される |
| **CLAUDE.md** | context | 低（読まれるが強制力なし） | プロジェクトの憲法。方針と構造の説明 |
| **skills/** | invocable workflow | なし（呼ばれた時だけ） | 再利用可能な手順のパッケージ |
| **agents/** | subagent definition | 中（権限で制約） | 役割分離と context 保護 |

## 2-2. 各スコープに持たせるもの

### permissions（強制。違反不可能）

「何ができて何ができないか」の物理的境界。

```
持たせるもの:
- secrets/** への deny（絶対）
- worktree 外書き込みの deny
- 本番系コマンドの deny
- 破壊的 git 操作の deny
- テスト・lint 等の安全コマンドの allow
- gh CLI の allow
- git read 系の allow

持たせないもの:
- 役割の定義（permissions では表現できない）
- ワークフローの手順
- 判断基準
```

### hooks（強制。必ず実行される）

「いつ何が自動的に起きるか」のライフサイクル制御。

```
持たせるもの:
- セッション開始時の文脈チェック（UserPromptSubmit）
- 危険操作の遮断（PreToolUse）
- 変更の追跡・ログ記録（PostToolUse）
- セッション終了時の handoff 保証（Stop）
- subagent 結果の自動反映（SubagentStop）
- 状態遷移の自動トリガー（Stop → 再triage）

持たせないもの:
- 複雑なビジネスロジック（hooks は shell script、重い処理は skill に委譲）
- 人間への通知（hooks は同期実行、通知は別機構）
```

### rules/（条件付き注入。状況に応じて読まれる）

「この状況ではこう振る舞え」という状況依存のガイダンス。

```
持たせるもの:
- safety.md: 安全ルール（停止条件、エスカレーション基準）
- testing.md: テスト方針（何をどこまでテストするか）
- github-ops.md: GitHub 運用ルール（Issue/PR の書き方）
- docs-output.md: ドキュメント出力ルール（RUNLOG の書き方等）
- role-detection.md: 役割自動判定ルール（★D2 で追加）
- autonomous-ops.md: 自律運転ルール（★D2 で追加）

持たせないもの:
- プロジェクト固有の仕様（それは 00-docs/ に書く）
- タスク固有の手順（それは task packet に書く）
```

### CLAUDE.md（context。方針と構造の説明）

「このプロジェクトは何で、どういう構造で、何が大事か」の全体像。

```
持たせるもの:
- Mission（このリポジトリの目的）
- Source of Truth（何がどこにあるか）
- Non-negotiables（絶対に守ること）
- Queue State Machine の状態定義
- Task Packet の必須ファイル定義
- Done / Review の定義
- 役割判定の基本ルール（cwd + queue + permission → 役割）
- デフォルト行動の定義

持たせないもの:
- 長いワークフロー手順（それは skill に書く）
- 実装の詳細手順（それは task packet に書く）
- 設定値（それは settings.json / config/ に書く）
```

### skills/（呼ばれた時だけ実行される再利用手順）

「この手順をパッケージとして呼び出す」。

```
持たせるもの:
- triage-from-state: 状態スキャンと候補生成
- make-task-packet: task packet 雛形生成
- failure-classifier: 失敗分類
- impact-analyzer: 仕様変更の影響分析
- worktree-execution: worktree 作成と起動
- review-gate: レビュー前チェック
- task-handoff: HANDOFF.md 更新
- issue-sync: GitHub 同期
- dashboard: 人間向け状況サマリー生成（★D2 で追加）
- auto-advance: 自動状態遷移（★D2 で追加）

持たせないもの:
- 1回限りのアドホック手順
- 判断基準（それは rules/ に書く）
```

### agents/（役割分離と context 保護）

「この役割にはこの権限とツールだけ」。

```
持たせるもの:
- triage.md: Triage Agent の定義（plan mode、read + write to .ops/）
- executor.md: Execution Agent の定義（acceptEdits、worktree 内のみ）
- reviewer.md: Review Agent の定義（plan mode、REVIEW.md のみ write）
- log-summarizer.md: ログ要約（plan mode、read only）
- research.md: 調査（plan mode、read + gh CLI）

持たせないもの:
- 具体的なタスクの指示（それは task packet に書く）
```

## 2-3. 「短いプロンプトで成立する」メカニズムの全体像

```
人間が「triage して」と入力
  ↓
[UserPromptSubmit hook] cwd と queue 状態をチェックし、文脈情報を注入
  ↓
[CLAUDE.md] 「メインrepo + plan mode → Triage 役割」と判定
  ↓
[rules/role-detection.md] Triage のデフォルト行動を確認
  ↓
[skills/triage-from-state] 実際の手順を実行
  ↓
[agents/triage.md] subagent として起動される場合の権限制約
  ↓
[hooks/Stop] 終了時に handoff を保証
```

人間が書くプロンプトは最初の1行だけ。残りは構造が補完する。

---

# 3. 役割の自動認識

## 3-1. 判定ルール

セッション起動時に以下の順で役割を判定する。これは `rules/role-detection.md` に記述し、`UserPromptSubmit` hook が文脈情報を提供する。

```
1. cwd が _worktrees/ 配下か？
   YES → worktree 内セッション
     → running/ にこの worktree に対応する task packet があるか？
       YES → Executor（デフォルト行動: HANDOFF.md から再開）
       NO  → 警告を出す（orphan worktree）

2. cwd がメインrepo か？
   YES → メインrepoセッション
     → プロンプトまたは queue 状態から役割を判定:
       - review/ にタスクがある + プロンプトに "review" → Reviewer
       - それ以外 → Triage（デフォルト行動: 状態スキャン）

3. permission mode は何か？
   plan → 読み取り中心の役割（Triage / Review）を強化
   acceptEdits → 実装役割（Executor）を強化
```

## 3-2. 役割判定に必要な前提条件

| 前提 | 誰が用意するか | いつ |
|---|---|---|
| cwd が正しい場所である | 人間 or 自動起動スクリプト | セッション起動時 |
| task packet が正しい queue 状態にある | 前のフェーズの agent | 前フェーズ完了時 |
| task.yaml に worktree_path が記載されている | make-task-packet skill | タスク生成時 |
| HANDOFF.md が最新である | Executor / Stop hook | 実行中・終了時 |
| permission mode が適切に設定されている | 起動コマンド or settings.json | セッション起動時 |

## 3-3. 各役割のデフォルト行動（詳細）

### Triage のデフォルト行動

```
起動条件: メインrepo + plan mode + 明示的な triage 指示 or デフォルト
処理:
  1. queue 全状態をスキャンする
     - inbox/: 新規投入物の有無
     - proposed/: 既存候補の状態
     - ready/: 着手可能タスクの有無
     - running/: 実行中タスクの進捗（HANDOFF.md の更新時刻）
     - review/: レビュー待ちタスクの有無
     - blocked/: 停滞タスクの有無と原因
     - done/: 直近完了タスク
  2. 異常を検出する
     - stale running（24時間以上 HANDOFF.md 未更新）
     - 同種 blocked の蓄積（3件以上）
     - orphan worktree（running task がないのに worktree が残っている）
  3. TRIAGE_REPORT と CANDIDATES を生成する
  4. 人間向けサマリーを出力する
```

### Executor のデフォルト行動

```
起動条件: worktree 内 + running task packet が存在
処理:
  1. task.yaml を読む（スコープ・forbidden paths・definition_of_done を確認）
  2. HANDOFF.md を読む（現在地と次の手を確認）
  3. PLAN.md を読む（全体計画を確認）
  4. HANDOFF.md の「次の1〜3手」から実装を再開する
  5. 完了条件を1つずつ潰す
  6. 区切りごとに HANDOFF.md を更新する
  7. 全完了条件を満たしたら VERIFY.md を更新し、review-gate を通す
```

### Reviewer のデフォルト行動

```
起動条件: メインrepo + plan mode + review/ にタスクがある
処理:
  1. review/ 内の task packet を1つ選ぶ（古い順）
  2. diff を読む（git diff main...{branch}）
  3. PLAN.md と diff の整合を確認する
  4. VERIFY.md の検証結果を確認する
  5. definition_of_done との照合
  6. REVIEW.md を書く
  7. approve / fix / blocked / reject の判定を出す
```

---

# 4. 自律運転設計

## 4-1. 自動進行ルール

以下の状態遷移は人間の介入なしに自動で行われる。

### ready → running（自動着手）

```
トリガー: ready/ にタスクが存在し、並列度上限に達していない
処理:
  1. worktree-execution skill を実行
  2. task packet を running/ に移動
  3. task.yaml の status を "running" に更新
  4. Executor セッションを起動（or 起動コマンドを出力）
```

### running → review（自動遷移）

```
トリガー: review-gate skill の全チェック項目が pass
処理:
  1. task packet を review/ に移動
  2. task.yaml の status を "review" に更新
  3. issue-sync skill で GitHub を更新
  4. Reviewer セッションの起動を通知（or 自動起動）
```

### run 終了 → 再triage（自動再評価）

```
トリガー: Stop hook の発火
処理:
  1. failure-classifier で結果を分類
  2. task.yaml の next_action を更新
  3. next_action に応じて:
     - continue_same_task → running のまま、HANDOFF.md を更新
     - retry_same_task → retry カウントを確認し、上限内なら再実行
     - split_new_task → 新 task packet を proposed/ に生成
     - blocked_waiting_human → blocked/ に移動
     - done → review/ に移動（review-gate 通過済みの場合）
```

## 4-2. 自動 unblock ルール

以下の条件に合致する blocked タスクは、人間の介入なしに自動で ready に戻す。

| 条件 | 自動 unblock | 理由 |
|---|---|---|
| flaky で retry 上限未達 | YES | 一時的障害は自動再試行 |
| environment で、同種の infra-fix が done になった | YES | 根本原因が解消された |
| implementation_bug で、3回連続失敗していない | YES | まだ修正の余地がある |
| spec_ambiguity | NO | 人間の仕様判断が必要 |
| needs_human_decision | NO | 人間の意思決定が必要 |
| permission | NO | 人間の権限変更が必要 |

## 4-3. 並列実行ルール

### 基本ルール

| ルール | 値 | 理由 |
|---|---|---|
| 同時 running 上限 | 3（デフォルト、.ops/config/parallel.yaml で変更可） | リソース制約と人間の把握可能量 |
| 同一ファイルへの同時編集 | 禁止 | コンフリクト防止 |
| 同一ディレクトリへの同時編集 | 警告付きで許可 | target_paths の重複をチェック |
| 依存関係のあるタスクの同時実行 | 禁止 | related_tasks でブロック判定 |

### コンフリクト検出

```
ready → running の遷移時に:
  1. 新タスクの target_paths を取得
  2. 既存 running タスクの target_paths と比較
  3. 重複がある場合:
     - 完全一致 → blocked（先行タスクの完了を待つ）
     - 部分一致 → 警告付きで running（ただし人間に通知）
  4. related_tasks に running 中のタスクがある場合 → blocked
```

### 並列度の自動調整

```
以下の場合に並列度を下げる:
- blocked 率が 50% を超えた → 上限を 1 に下げる
- 同種 failure が 3件蓄積 → 該当ディレクトリのタスクを一時停止
- review/ が 5件以上溜まった → 新規着手を停止し review 消化を優先
```

## 4-4. 失敗時の自律的振る舞い

### retry ポリシー

| 失敗カテゴリ | 最大 retry | 間隔 | エスカレーション |
|---|---|---|---|
| flaky | 3回 | 即時 | 3回失敗 → environment に再分類 → blocked |
| implementation_bug | 2回 | 即時 | 2回失敗 → needs_human_decision に再分類 |
| environment | 0回 | - | 即 blocked |
| permission | 0回 | - | 即 blocked |
| spec_ambiguity | 0回 | - | 即 blocked |

### stale running の自動処理

```
閾値: HANDOFF.md が 2時間以上未更新
処理:
  1. セッションが生きているか確認（可能な場合）
  2. 生きていない場合:
     - task.yaml の last_run を確認
     - HANDOFF.md の内容を読む
     - failure-classifier で分類
     - 分類結果に応じて blocked or retry
  3. 通知: 人間向けダッシュボードに警告を記録
```

### セッション断絶時の復帰

Design D と同一の原則: 会話が飛んでも作業は飛ばない。

```
復帰手順（自動）:
  1. running/ にタスクがある
  2. worktree が存在する
  3. HANDOFF.md が存在する
  → 新しいセッションが Executor として自動認識し、HANDOFF.md から再開

復帰手順（手動、最悪の場合）:
  1. queue/running/ を確認
  2. 該当 worktree に cd
  3. claude を起動
  → 以降は自動
```

## 4-5. 長時間無人運転のためのタスク粒度

### タスク分割の基準

| 基準 | 推奨粒度 | 理由 |
|---|---|---|
| 想定実行時間 | 30分〜2時間 | context 膨張を防ぎ、handoff の頻度を上げる |
| 変更ファイル数 | 10ファイル以下 | レビューの容易さ |
| 完了条件数 | 3〜5個 | 進捗の可視性 |
| 依存関係 | 0〜1個 | 並列実行の阻害を減らす |

### 自動分割の判断

Triage Agent が以下の場合にタスク分割を提案する:

```
- PLAN.md のステップ数が 10 を超える
- target_paths が 3ディレクトリ以上にまたがる
- definition_of_done が 7個以上
- 過去の同種タスクが 3時間以上かかった記録がある
```

---

# 5. 人間向け最小インターフェース

## 5-1. 設計方針

非エンジニアが「5分で状況を把握し、判断し、指示を出す」ためのインターフェース。

## 5-2. ダッシュボード（.ops/DASHBOARD.md）

Triage Agent が自動生成し、状態変化のたびに更新する。

```markdown
# Dashboard — 最終更新: {datetime}

## 今すぐ判断が必要なもの
- [ ] Q-P1-0045 がレビュー完了 → merge するか？ → [REVIEW.md](queue/review/Q-P1-0045/REVIEW.md)
- [ ] Q-P1-0043 が仕様曖昧で停止中 → [spec-question.md](queue/blocked/Q-P1-0043/spec-question.md)

## 進捗サマリー
| 状態 | 件数 | 詳細 |
|---|---|---|
| done（今日） | 2 | Q-P1-0040, Q-P1-0041 |
| review | 1 | Q-P1-0045 |
| running | 2 | Q-P1-0042（70%完了）, Q-P1-0044（着手直後） |
| blocked | 1 | Q-P1-0043（仕様曖昧） |
| ready | 3 | 次の着手候補 |
| proposed | 5 | ready 昇格待ち |

## 警告
- Q-P1-0042 の HANDOFF.md が 1.5時間未更新（閾値: 2時間）
- blocked が 1件あり（Q-P1-0043: spec_ambiguity）

## 直近完了タスクのサマリー
- Q-P1-0040: Nginx設定修正 → 成功、merge済み
- Q-P1-0041: ログローテーション追加 → 成功、merge済み
```

## 5-3. 人間が見るべきファイルの導線

| 人間がやりたいこと | 見るファイル |
|---|---|
| 全体の状況を知りたい | `.ops/DASHBOARD.md` |
| 判断が必要なものを知りたい | DASHBOARD.md の「今すぐ判断が必要なもの」 |
| レビューして merge したい | `queue/review/{task}/REVIEW.md` → diff を見て判断 |
| blocked の理由を知りたい | `queue/blocked/{task}/blocked_reason.md` or `spec-question.md` |
| 完了したタスクの結果を知りたい | `queue/done/{task}/RESULT.md` |
| 次に何が実行されるか知りたい | DASHBOARD.md の ready 欄 or `.ops/reports/triage/CANDIDATES_{date}.yaml` |

## 5-4. 人間の操作パターン（3つだけ）

### パターン A: 朝の確認と指示（5分）

```
1. DASHBOARD.md を開く
2. 「今すぐ判断が必要なもの」を処理する
   - review → merge or 差し戻し
   - blocked → 判断して ready に戻す or 保留
3. proposed の上位を見て ready に昇格させる
4. 終了
```

### パターン B: 就寝前の確認（2分）

```
1. DASHBOARD.md を開く
2. running のタスクが正常に進行しているか確認
3. blocked が発生していないか確認
4. 問題なければそのまま放置
```

### パターン C: 問題発生時の介入（10分）

```
1. DASHBOARD.md の警告欄を確認
2. 該当タスクの HANDOFF.md or blocked_reason.md を読む
3. 判断して指示を出す
   - blocked を解除して ready に戻す
   - タスクをキャンセルして done に移す
   - 仕様を確定して再開させる
```

---

# 6. Queue State Machine

Design D と同一の7状態。遷移ルールに自動進行ルールを追加。

## 6-1. 7状態（Design D と同一）

```
inbox → proposed → ready → running → review → done
                     ↑        │         │
                     │        ↓         │
                     └──── blocked ←────┘
```

## 6-2. 遷移ルール（自動進行を明記）

| 遷移 | 実行者 | トリガー | 処理 |
|---|---|---|---|
| inbox → proposed | AI | inbox に新規投入 | 構造化 + スコアリング + task.yaml 生成 |
| proposed → ready | **人間** | 人間の判断 | 今日の実行本数を決定 |
| ready → running | AI | ready に存在 + 並列度上限未達 | worktree 作成 + session 起動 |
| running → review | AI | review-gate 全項目 pass | task packet 移動 + GitHub 更新 |
| running → blocked | AI | failure-classifier 判定 | 分類結果に応じた処理 |
| review → done | **人間** | 人間の merge 判断 | merge + cleanup |
| review → blocked | AI | unsafe change 検出 | REVIEW.md に理由記載 |
| blocked → ready | **人間** or AI | ブロック解消 | auto-unblock ルール or 人間判断 |

## 6-3. イベント駆動の再トリアージ（Design D と同一）

| イベント | 再トリアージの内容 |
|---|---|
| run が終了 | 該当taskの再分類 + 次候補の再計算 |
| blocked が発生 | unblock task の自動生成検討 |
| review で差し戻し | fix task の生成 or 同task内での修正判断 |
| inbox に仕様変更が投入 | impact.md 生成 + 影響先タスクの状態見直し |
| stale running が閾値超過 | blocked への自動移行検討 |

---

# 7. Task Packet

## 7-1. 構成（Design D と同一）

```text
.ops/queue/{state}/Q-{project}-{id}-{slug}/
├── task.yaml          # 構造化メタデータ（機械可読の正本）
├── BRIEF.md           # 背景・目的・完了条件
├── PLAN.md            # 実行計画（差分更新する）
├── HANDOFF.md         # 再開点（最重要ファイル）
├── VERIFY.md          # 事実のみ（テスト結果等）
├── RESULT.md          # 最終サマリー（完了時のみ）
├── REVIEW.md          # Review Agent の出力
├── RUNLOG.md          # 構造化実行ログ
├── DECISIONS.md       # このtask内の判断記録
├── RISKS.md           # 未解決リスク・既知制約
├── issue.md           # Issue化用の整形済み文章
└── artifacts/
```

## 7-2. task packet が自己完結であるための必須情報

task packet を読んだセッションが「何をすべきか」を判断するために、以下の情報が必須。

### task.yaml の必須フィールド

| フィールド | 用途 | 省略可否 |
|---|---|---|
| id | タスク識別 | 必須 |
| title | 人間可読な概要 | 必須 |
| status | 現在の queue 状態 | 必須 |
| target_paths | 編集対象ディレクトリ | 必須 |
| forbidden_paths | 編集禁止ディレクトリ | 任意（なければ制約なし） |
| definition_of_done | 完了条件リスト | 必須 |
| worktree_path | worktree の物理パス | running 以降は必須 |
| branch | ブランチ名 | running 以降は必須 |
| priority | 6軸スコアリング結果 | proposed 以降は必須 |
| related_tasks | 依存タスク | 任意（並列実行の判定に使用） |
| next_action | 次の行動指示 | run 終了時に必須 |
| failure_classification | 失敗分類 | failure 時のみ必須 |

### BRIEF.md の必須セクション

| セクション | 用途 | 省略可否 |
|---|---|---|
| 背景 | なぜこのタスクが必要か | 必須 |
| 目的 | 何を達成するか | 必須 |
| 完了条件 | いつ「終わり」か | 必須（task.yaml の definition_of_done と一致） |
| 触る範囲 | どこを編集するか | 必須（task.yaml の target_paths と一致） |

### HANDOFF.md の必須セクション

| セクション | 用途 | 省略可否 |
|---|---|---|
| 現在地 | 何がどこまで終わったか | 必須 |
| 次の1〜3手 | 最優先でやること | 必須 |
| 環境状態 | branch, worktree, テスト状態 | 必須 |

### 省略可能な情報

以下は必須ではないが、あると品質が上がる。

- PLAN.md の詳細ステップ（なければ Executor が BRIEF.md から自分で計画を立てる）
- RISKS.md（なければリスクなしと判断）
- DECISIONS.md（判断が不要なタスクでは空でよい）
- issue.md（Issue 化しない小タスクでは不要）
- related_docs（関連ドキュメントがなければ省略可）

---

# 8. ディレクトリ設計

Design D と同一。追加分のみ記載。

## 8-1. 追加ファイル

```text
{project}/
├── .ops/
│   ├── DASHBOARD.md                   # ★D2追加: 人間向けダッシュボード
│   ├── config/
│   │   ├── scoring-weights.yaml       # Design D と同一
│   │   └── parallel.yaml              # ★D2追加: 並列実行設定
│   └── ...（Design D と同一）
├── .claude/
│   ├── rules/
│   │   ├── ...（Design D と同一）
│   │   ├── role-detection.md          # ★D2追加: 役割自動判定ルール
│   │   └── autonomous-ops.md          # ★D2追加: 自律運転ルール
│   ├── skills/
│   │   ├── ...（Design D と同一）
│   │   ├── dashboard/
│   │   │   └── SKILL.md              # ★D2追加: ダッシュボード生成
│   │   └── auto-advance/
│   │       └── SKILL.md              # ★D2追加: 自動状態遷移
│   └── ...（Design D と同一）
└── ...（Design D と同一）
```

## 8-2. parallel.yaml

```yaml
# 並列実行設定
max_concurrent_running: 3
conflict_detection:
  same_file: block
  same_directory: warn
  related_task: block
auto_throttle:
  blocked_rate_threshold: 0.5      # blocked 率がこれを超えたら並列度を 1 に下げる
  review_backlog_threshold: 5      # review 待ちがこれを超えたら新規着手を停止
stale_threshold_minutes: 120       # HANDOFF.md 未更新の閾値（分）
retry_limits:
  flaky: 3
  implementation_bug: 2
  environment: 0
  permission: 0
  spec_ambiguity: 0
```

---

# 9. Settings 設計

Design D と同一。変更なし。

---

# 10. Hooks 設計

Design D の hooks を基盤とし、以下を拡張する。

## 10-1. UserPromptSubmit（拡張）

Design D の機能に加え、以下を追加:

```bash
# 役割判定のための文脈情報を注入
CURRENT_DIR=$(pwd)
if [[ "$CURRENT_DIR" == *"_worktrees"* ]]; then
  ROLE="executor"
  TASK_DIR=$(find .ops/queue/running/ -maxdepth 1 -mindepth 1 -type d 2>/dev/null | head -1)
  if [ -n "$TASK_DIR" ]; then
    echo "CONTEXT: role=executor, task=$(basename $TASK_DIR), handoff=$TASK_DIR/HANDOFF.md"
  fi
else
  REVIEW_COUNT=$(ls -d .ops/queue/review/*/ 2>/dev/null | wc -l)
  RUNNING_COUNT=$(ls -d .ops/queue/running/*/ 2>/dev/null | wc -l)
  echo "CONTEXT: role=triage_or_review, review_pending=$REVIEW_COUNT, running=$RUNNING_COUNT"
fi
```

## 10-2. Stop（拡張）

Design D の機能に加え、以下を追加:

```bash
# 自動再triage のトリガー
# run 終了時に failure-classifier を呼び、next_action を設定
# DASHBOARD.md を更新
bash .ops/hooks/update-dashboard.sh
```

## 10-3. 新規: update-dashboard.sh

```bash
#!/bin/bash
# .ops/hooks/update-dashboard.sh
# queue の状態から DASHBOARD.md を自動生成する

DASHBOARD=".ops/DASHBOARD.md"
NOW=$(date "+%Y-%m-%d %H:%M")

# 各状態の件数カウント
DONE_TODAY=$(find .ops/queue/done/ -maxdepth 1 -mindepth 1 -type d -newer /tmp/.dashboard_daily_marker 2>/dev/null | wc -l)
REVIEW=$(ls -d .ops/queue/review/*/ 2>/dev/null | wc -l)
RUNNING=$(ls -d .ops/queue/running/*/ 2>/dev/null | wc -l)
BLOCKED=$(ls -d .ops/queue/blocked/*/ 2>/dev/null | wc -l)
READY=$(ls -d .ops/queue/ready/*/ 2>/dev/null | wc -l)
PROPOSED=$(ls -d .ops/queue/proposed/*/ 2>/dev/null | wc -l)

# DASHBOARD.md を更新（実際の生成は dashboard skill に委譲）
echo "Dashboard updated at $NOW: done=$DONE_TODAY review=$REVIEW running=$RUNNING blocked=$BLOCKED ready=$READY proposed=$PROPOSED"
```

---

# 11. Skills 設計

Design D の skills を基盤とし、以下を追加。

## 11-1. dashboard（★D2 追加）

```markdown
---
name: dashboard
description: 人間向けの状況サマリーを.ops/DASHBOARD.mdに生成する
---

# Dashboard Generator

## 読むもの
1. .ops/queue/ の全状態
2. 各 task packet の task.yaml と HANDOFF.md
3. .ops/runs/ の直近ログ
4. blocked/ の blocked_reason.md

## 出力
.ops/DASHBOARD.md を上書き更新

## 内容
1. 「今すぐ判断が必要なもの」（review 待ち、blocked で human decision 待ち）
2. 進捗サマリー（各状態の件数と詳細）
3. 警告（stale running、blocked 蓄積、レビュー滞留）
4. 直近完了タスクのサマリー

## ルール
- 人間が5分で読める量に収める
- 判断に必要なファイルへのリンクを必ず含める
- 技術用語は最小限にする
```

## 11-2. auto-advance（★D2 追加）

```markdown
---
name: auto-advance
description: 自動状態遷移のルールに基づいてqueueの状態を進める
---

# Auto Advance

## チェック項目
1. ready/ にタスクがあり、並列度上限に達していない → running への遷移候補
2. running/ で review-gate 全項目 pass のタスク → review への遷移候補
3. blocked/ で auto-unblock 条件を満たすタスク → ready への遷移候補

## コンフリクト検出
- 新タスクの target_paths と既存 running の target_paths を比較
- 重複 → blocked（先行タスクの完了待ち）

## 出力
- 遷移を実行した場合: task.yaml の status 更新 + task packet の移動
- 遷移できない場合: 理由をログに記録
```

## 11-3. Design D の skills（変更なし）

以下は Design D と同一のため、詳細は Design D を参照。

- triage-from-state
- make-task-packet
- failure-classifier
- impact-analyzer
- worktree-execution
- review-gate
- task-handoff
- issue-sync

---

# 12. Subagents 設計

Design D と同一。変更なし。

---

# 13. Permissions 設計

Design D と同一。変更なし。

---

# 14. rules/ の設計（★D2 追加）

## 14-1. role-detection.md

```markdown
# Role Detection Rules

## 判定ロジック

セッション起動時に以下の順で役割を判定し、対応するデフォルト行動を取る。

### 1. cwd による判定
- cwd が `_worktrees/` 配下 → **Executor**
- cwd がメインrepo → **Triage** or **Reviewer**（次の判定へ）

### 2. queue 状態による判定（メインrepoの場合）
- プロンプトに "review" を含む or review/ にタスクがある → **Reviewer**
- それ以外 → **Triage**

### 3. permission mode による補強
- plan mode → 読み取り中心（Triage / Review に適合）
- acceptEdits mode → 実装中心（Executor に適合）

## デフォルト行動

### Executor
1. running/ で自分の worktree に対応する task packet を探す
2. task.yaml を読む
3. HANDOFF.md を読む
4. PLAN.md を読む
5. HANDOFF.md の「次の1〜3手」から実装を開始する

### Triage
1. triage-from-state skill を実行する

### Reviewer
1. review/ の最も古い task packet を選ぶ
2. review-gate の出力と diff を読む
3. REVIEW.md を書く
```

## 14-2. autonomous-ops.md

```markdown
# Autonomous Operations Rules

## 自動進行の原則
- AI は定義された自動遷移を躊躇なく実行する
- 人間ゲートは飛ばさない（proposed → ready、review → done）
- 不確実な場合は blocked にして停止する（進むより止まる）

## 停止条件（以下のいずれかに該当したら即座に blocked）
- 破壊的操作が必要（データ削除、volume 削除、本番操作）
- 課金リスクがある（外部 API 呼び出し、高コスト処理）
- セキュリティ問題を検出（秘匿情報の露出、認証情報の漏洩）
- 仕様が曖昧で複数の解釈がある
- 3回連続で同じエラーが発生
- target_paths 外のファイルを編集する必要がある

## retry ルール
- flaky: 最大3回、即時 retry
- implementation_bug: 最大2回、RUNLOG に root cause を記録してから retry
- その他: retry しない、即 blocked

## 並列実行ルール
- parallel.yaml の設定に従う
- コンフリクト検出は target_paths の比較で行う
- 依存タスク（related_tasks）の同時実行は禁止

## HANDOFF.md の更新頻度
- 完了条件を1つ潰すごとに更新
- 30分ごとに更新（最低頻度）
- session 終了時に必ず更新（Stop hook で強制）
- /compact 前に必ず更新

## ダッシュボード更新
- 状態遷移が発生するたびに .ops/DASHBOARD.md を更新する
```

---

# 15. GitHub Issues / Projects 連携

Design D と同一。変更なし。

---

# 16. ドキュメント管理

Design D と同一。変更なし。

---

# 17. 日次ワークフロー（D2 版 — 最小化）

## 朝の確認（人間: 5分）

```
1. .ops/DASHBOARD.md を開く
2. 「今すぐ判断が必要なもの」を処理する
3. proposed の上位を ready に昇格させる
4. 終了（残りは AI が自動で進める）
```

## AI の自律運転（人間の介入なし）

```
以下は AI が自動で行う:
- ready → running: worktree 作成、session 起動
- 実装: BRIEF → PLAN → HANDOFF に従って実装
- テスト: VERIFY.md を更新
- review-gate 通過: running → review に自動遷移
- 失敗時: failure-classifier で分類、retry or blocked
- 再triage: run 終了時に自動
- DASHBOARD.md: 状態変化時に自動更新
```

## 就寝前の確認（人間: 2分）

```
1. .ops/DASHBOARD.md を開く
2. 異常がないか確認
3. 問題なければ放置
```

## 起床後の確認（人間: 5分）

```
1. .ops/DASHBOARD.md を開く
2. 夜間の進捗を確認（done に入ったタスク、blocked に入ったタスク）
3. 必要な判断を行う
4. 次のサイクルへ
```

---

# 18. 失敗からの再開設計

Design D と同一の思想。自律的な復帰を追加。

## 18-1. 自律復帰マトリクス

| 何が飛んでも | 残るもの | 復帰方法 | 人間の介入 |
|---|---|---|---|
| Claude session | worktree, queue, runs, HANDOFF | 新 session が HANDOFF.md から自動再開 | 不要 |
| tmux session | 同上 | 同上 | 不要 |
| Mac 再起動 | 同上 | 同上 | 不要 |
| Git 操作ミス | worktree は独立 | worktree から復帰 | 不要 |
| 全体混乱 | queue structure | 全 running を blocked に、clean triage | **必要** |

## 18-2. blocked 蓄積時の自動対応

```
1. Triage Agent が blocked/ を定期スキャン
2. 類似原因を束ねる
3. 3件以上の同種 blocked → 高優先度の infra-fix task を proposed に自動追加
4. auto-unblock 条件を満たすものは自動で ready に戻す
5. それ以外は DASHBOARD.md の「今すぐ判断が必要なもの」に表示
```

---

# 19. 安全設計

Design D と同一。変更なし。

---

# 20. 導入チェックリスト

Design D のチェックリストに以下を追加:

### Step 2.5: D2 追加設定
- [ ] .claude/rules/role-detection.md
- [ ] .claude/rules/autonomous-ops.md
- [ ] .claude/skills/dashboard/SKILL.md
- [ ] .claude/skills/auto-advance/SKILL.md
- [ ] .ops/config/parallel.yaml
- [ ] .ops/DASHBOARD.md（初期状態: 空テンプレート）
- [ ] .ops/hooks/update-dashboard.sh

---

# 21. Design D との差分サマリー

| 観点 | Design D | Design D2 |
|---|---|---|
| 役割認識 | プロンプトで宣言 | cwd + queue + permission で自動判定 |
| デフォルト行動 | 未定義（毎回指示が必要） | 各役割に定義済み |
| 状態遷移 | 多くが人間トリガー | 人間ゲートは 3箇所のみ、残りは自動 |
| 並列実行 | ルールなし | parallel.yaml で上限・コンフリクト・自動調整を定義 |
| 失敗復帰 | handoff ベースの手動復帰 | 自動 retry + auto-unblock + 自動復帰 |
| 人間インターフェース | queue フォルダ構造を直接見る | DASHBOARD.md で 5分把握 |
| タスク粒度 | 未定義 | 30分〜2時間、自動分割提案あり |
| セッション断絶 | HANDOFF.md から手動再開 | 新 session が自動認識して再開 |
| プロンプト | 各フェーズに詳細プロンプトが必要 | 「triage して」「実行して」で成立 |
| 追加ファイル | なし | rules/ 2個、skills/ 2個、config 1個、DASHBOARD.md |

## 変更していないもの

以下は Design D と完全に同一であり、変更していない:

- 4層構造
- Queue State Machine の7状態
- Task Packet の構成と各ファイルの仕様
- task.yaml の詳細仕様
- Run Log 管理
- Settings 設計
- Hooks の基本設計（拡張はしたが基盤は同一）
- Skills の既存8個（追加はしたが変更なし）
- Subagents 設計
- Permissions 設計
- GitHub Issues / Projects 連携
- ドキュメント管理と正本ルール
- 安全設計
- CLI / コマンド運用

---

# 22. 最終判断

## Design D2 で追加すべきもの

- rules/role-detection.md（役割自動判定）
- rules/autonomous-ops.md（自律運転ルール）
- skills/dashboard（人間向けサマリー生成）
- skills/auto-advance（自動状態遷移）
- .ops/config/parallel.yaml（並列実行設定）
- .ops/DASHBOARD.md（人間向けダッシュボード）
- hooks の拡張（役割判定の文脈注入、ダッシュボード更新）
- CLAUDE.md へのデフォルト行動の記述
- UserPromptSubmit hook への文脈情報注入

## 一番危険なもの（Design D と同一 + 追加）

Design D の「絶対にやらない」リストに加え:

- 人間ゲートを飛ばして proposed → ready を自動化する運用
- 並列度上限なしで無制限に task を走らせる運用
- DASHBOARD.md なしで queue フォルダを直接見させる運用（非エンジニアには厳しい）
- auto-unblock を全カテゴリに適用する運用（spec_ambiguity の自動解消は危険）
- 失敗時に retry 上限なしで無限リトライする運用

---

この設計は Design D の骨格を一切壊さず、「人間がプロンプトを書かなくても構造が役割と行動を決定する」層を追加したものである。Design D の普遍的構造（queue / task packet / worktree / handoff）はそのまま、Claude Code 固有の構造的決定メカニズム（CLAUDE.md / rules / hooks / skills）を活用して自律性を実現している。
