# Success Workflow — Ready to Next Task

**前提**: AI Harness Design D (harness_design-D_unified.md)
**対象**: 成功パターンのみ
**範囲**: ready → running → review → done → cleanup → 次タスク生成

---

## 全体フロー概観

```
[Session 1: Triage/Ops] ready にある task を worktree で起動準備
         ↓
[Session 2: Execution]  worktree 内で実装・テスト・handoff 完了
         ↓
[Session 3: Review]     review 判定 → 人間が merge 決定
         ↓
[Session 1 再開 or 4]   cleanup → 次タスク生成
```

---

## Phase 0: 前提確認（Session 1 — Triage/Ops セッション）

**場所**: メインrepo (`{project}/`)
**起動**: `claude --permission-mode plan --model sonnet`

### 人間がやること

1. `.ops/queue/ready/` を確認する

```bash
ls .ops/queue/ready/
```

2. 実行するタスクを1つ選ぶ
   - **見るファイル**: `ready/{task-dir}/task.yaml` — priority.score と definition_of_done
   - **見るファイル**: `ready/{task-dir}/BRIEF.md` — 背景・目的・完了条件
   - **判断**: 「このタスクを今から実行するか？」

選んだら Phase 1 へ。

---

## Phase 1: Worktree 作成と実行準備（Session 1 続き）

### AI にやらせること

以下のプロンプトを投入する:

```
タスク Q-{project}-{id}-{slug} の実行準備をしてください。
1. git worktree add で worktree を作成
2. task packet を ready/ → running/ に移動
3. task.yaml の status を "running" に更新
4. task.yaml の worktree_path と branch を記入
5. HANDOFF.md の環境状態を初期化
6. 起動コマンドを出力
```

### AI が実行する手順

```bash
# 1. worktree 作成
git worktree add /Users/{user}/GitHub/_worktrees/{project}/Q-{project}-{id}-{slug} \
  -b task/Q-{project}-{id}-{slug}

# 2. task packet 移動
mv .ops/queue/ready/Q-{project}-{id}-{slug} .ops/queue/running/

# 3. task.yaml 更新 (status, worktree_path, branch, updated_at)

# 4. HANDOFF.md 初期化
#    現在地: "未開始 — worktree 作成済み"
#    環境状態: branch名, worktree path を記入

# 5. runs/ にディレクトリ作成
mkdir -p .ops/runs/Q-{project}-{id}-{slug}/run-001
```

### AI の出力（人間が使う）

```
worktree 作成完了。以下のコマンドで実行セッションを開始してください:

cd /Users/{user}/GitHub/_worktrees/{project}/Q-{project}-{id}-{slug}
claude --permission-mode plan --model sonnet
```

### セッション切り替えポイント

> **ここで Session 1 を終了し、Session 2 を worktree 内で開始する。**
> 理由: 実行セッションは worktree 内で起動する必要があり、permission mode も異なる。

---

## Phase 2: 実装（Session 2 — Execution セッション）

**場所**: worktree (`/Users/{user}/GitHub/_worktrees/{project}/Q-{project}-{id}-{slug}/`)
**起動**: `claude --permission-mode plan --model sonnet`

### Step 2-1: 事前読み込み（plan mode）

以下のプロンプトを投入する:

```
このタスクの実行を開始します。まず以下を読んで実行計画を確認してください:
1. .ops/queue/running/Q-{project}-{id}-{slug}/BRIEF.md
2. .ops/queue/running/Q-{project}-{id}-{slug}/PLAN.md
3. .ops/queue/running/Q-{project}-{id}-{slug}/HANDOFF.md
対象ファイルの現状も確認し、PLAN.md に更新が必要なら更新してください。
```

### AI が実行する手順

- BRIEF.md を読み、目的と完了条件を把握
- PLAN.md を読み、実装ステップを確認
- HANDOFF.md を読み、現在地を確認
- target_paths のファイルを実際に読み、PLAN.md と齟齬があれば差分更新
- PLAN.md の「事前確認」チェックリストを埋める

### Step 2-2: Permission 切り替え → 実装開始

**人間がやること**:

```
/permissions
```

→ `acceptEdits` に切り替える（または起動時に `--permission-mode acceptEdits` で再起動）

その後、以下のプロンプトを投入する:

```
PLAN.md のステップに従って実装を開始してください。
各ステップ完了ごとに RUNLOG.md に記録してください。
```

### AI が実行する手順

- PLAN.md のステップを順に実装
- 各ステップ完了時に RUNLOG.md へ記録（何をした / どうなった / 次のアクション）
- HANDOFF.md を区切りごとに更新（最低30分に1回）
- DECISIONS.md にこの task 内で判断した事項を記録（該当する場合のみ）

### Step 2-3: テスト・検証

プロンプト（実装完了後、AI が自然に遷移するか、以下を投入）:

```
実装が完了したらテストを実行し、VERIFY.md を事実で埋めてください。
```

### AI が実行する手順

- lint / typecheck / unit test / integration test を実行
- VERIFY.md の各行を事実で埋める（pass / fail / not-run + 備考）
- 全て pass なら次へ。fail があれば修正して再テスト

### Step 2-4: Handoff・Result 確定

プロンプト:

```
テストが通ったら以下を完了してください:
1. HANDOFF.md を最終状態に更新
2. RESULT.md を記入
3. RUNLOG.md の最終 run を記録
4. git add / git commit（worktree 内のみ）
5. task.yaml の last_run を更新し、next_action.type を "done" にする
```

### AI が実行する手順

```bash
# 1. HANDOFF.md 更新
#    現在地: "実装完了、テスト全 pass、レビュー待ち"
#    次の1〜3手: "レビューを受けて merge"

# 2. RESULT.md 記入
#    何をしたか / 何が終わったか / 変更ファイル一覧

# 3. RUNLOG.md 最終記録

# 4. 実装のコミット
git add -A
git commit -m "Q-{id}: {title} — 実装完了"

# 5. task.yaml 更新
#    last_run.result: "success"
#    next_action.type: "done"

# 6. runs/ に manifest.json と summary.md を保存
#    .ops/runs/Q-{project}-{id}-{slug}/run-001/manifest.json
#    .ops/runs/Q-{project}-{id}-{slug}/run-001/summary.md
#    .ops/runs/Q-{project}-{id}-{slug}/run-001/changed-files.txt
```

### Step 2-5: Review Gate チェック

プロンプト:

```
review-gate のチェックを実施してください:
- VERIFY.md が全項目記入済みか
- HANDOFF.md が最新か
- RESULT.md が存在するか
- definition_of_done の各項目に対応する検証があるか
全て pass なら task packet を review/ に移動してください。
```

### AI が実行する手順

- 各チェック項目を確認
- 全 pass → task packet を `running/` → `review/` に移動

```bash
mv .ops/queue/running/Q-{project}-{id}-{slug} .ops/queue/review/
# task.yaml の status を "review" に更新
```

- ブランチを push（PR 作成のため）

```bash
git push -u origin task/Q-{project}-{id}-{slug}
```

### AI の出力（人間が使う）

```
Review Gate 全項目 pass。task packet を review/ に移動しました。
ブランチ push 済み。

次のステップ:
1. この Execution セッションを終了
2. メインrepo で Review セッションを開始
```

### セッション切り替えポイント

> **ここで Session 2 を終了し、Session 3 を開始する。**
> 理由: Review Agent は実装とは独立した視点で判定する必要がある。plan mode で起動する。

---

## Phase 3: レビュー（Session 3 — Review セッション）

**場所**: メインrepo (`{project}/`)
**起動**: `claude --permission-mode plan --model sonnet`

### AI にやらせること

以下のプロンプトを投入する:

```
タスク Q-{project}-{id}-{slug} のレビューを実施してください。
1. .ops/queue/review/Q-{project}-{id}-{slug}/ の全ファイルを読む
2. worktree の diff を確認（git diff main...task/Q-{project}-{id}-{slug}）
3. REVIEW.md を書く（テンプレートに従う）
4. PR を作成する
```

### AI が実行する手順

```bash
# 1. task packet の全ファイルを読む
#    task.yaml, BRIEF.md, PLAN.md, VERIFY.md, RESULT.md, RUNLOG.md

# 2. diff を確認
git diff main...task/Q-{project}-{id}-{slug} --stat
git diff main...task/Q-{project}-{id}-{slug}

# 3. REVIEW.md を書く
#    差分サマリー（変更ファイル数、追加行、削除行）
#    チェック項目テーブル（PLAN整合、完了条件達成、破壊的変更、セキュリティ、テスト十分性等）
#    判定: approve candidate / fix before merge / blocked / unsafe

# 4. PR 作成
gh pr create \
  --base main \
  --head task/Q-{project}-{id}-{slug} \
  --title "Q-{id}: {title}" \
  --body "$(cat .ops/queue/review/Q-{project}-{id}-{slug}/RESULT.md)"

# 5. task.yaml に pr_id を記入

# 6. Issue がある場合、PR に "Closes #{issue_id}" を含める
```

### AI の出力（人間が使う）

```
レビュー完了。

■ REVIEW.md 判定: approve candidate
■ PR: https://github.com/{owner}/{repo}/pull/{N}
■ 変更ファイル: N files (+XX, -YY)

REVIEW.md の内容:
（チェック項目テーブルの要約）

人間の判断をお願いします:
- PR を見て merge するか判断してください
- merge する場合: gh pr merge {N} --merge
- 修正が必要な場合: 指摘内容をお知らせください
```

### 人間がやること

1. **見るファイル**: `.ops/queue/review/Q-{project}-{id}-{slug}/REVIEW.md` — AI のレビュー判定
2. **見るもの**: GitHub PR の diff（リンクは AI が出力する）
3. **判断**: merge するか / 差し戻すか

**merge する場合**:

```bash
gh pr merge {N} --merge
```

または GitHub UI で merge ボタンを押す。

---

## Phase 4: 完了処理と Cleanup（Session 3 続き or Session 4）

**場所**: メインrepo (`{project}/`)

### 人間がやること

merge 完了後、以下のプロンプトを投入する:

```
Q-{project}-{id}-{slug} が merge されました。完了処理を実行してください:
1. task packet を review/ → done/ に移動
2. task.yaml の status を "done" に更新
3. worktree を削除
4. ブランチを削除
5. Issue がある場合は close
6. GitHub Project のステータスを更新
7. .ops/ の変更を commit
```

### AI が実行する手順

```bash
# 1. task packet 移動
mv .ops/queue/review/Q-{project}-{id}-{slug} .ops/queue/done/

# 2. task.yaml 更新
#    status: "done"
#    updated_at: 現在時刻

# 3. worktree 削除
git worktree remove /Users/{user}/GitHub/_worktrees/{project}/Q-{project}-{id}-{slug}

# 4. ブランチ削除（merge 済みなので安全）
git branch -d task/Q-{project}-{id}-{slug}

# 5. Issue close（issue_id がある場合）
gh issue close {issue_id} --comment "完了: Q-{id} merged via PR #{pr_id}"

# 6. GitHub Project 更新（Project が設定されている場合）
# gh project item-edit で Status を "Done" に更新

# 7. main を最新に更新
git pull origin main

# 8. 運用ファイルの commit
git add .ops/
git commit -m "ops: Q-{id} done — cleanup complete"
git push origin main
```

---

## Phase 5: 次タスク生成（Session 3 続き or 新規 Session）

**場所**: メインrepo (`{project}/`)
**mode**: plan

完了処理の直後に、次のタスク候補を生成する。

### AI にやらせること

以下のプロンプトを投入する:

```
Q-{project}-{id}-{slug} が完了しました。
イベント駆動の再トリアージを実行してください:
1. 完了タスクの後続候補を検出
2. blocked/ の再スキャン（この完了で unblock されるものがあるか）
3. 00-docs/status.md を更新
4. 次の候補を proposed/ に生成
5. TRIAGE_REPORT を更新
```

### AI が実行する手順

```bash
# 1. 完了タスクの関連を確認
#    task.yaml の related_tasks を読む
#    → 後続タスクが blocked/ にあれば ready 候補に

# 2. blocked/ 再スキャン
#    この完了で解消される blocked があるか判定
#    → あれば blocked/ → ready/ に移動提案（人間が最終判断）

# 3. 00-docs/status.md 更新
#    完了タスクを「完了」欄に追記
#    進捗の数値を更新

# 4. inbox/ と 00-docs/roadmap.md を見て次の候補を proposed/ に生成
#    make-task-packet skill 相当の処理:
#    - task.yaml（6軸スコアリング付き）
#    - BRIEF.md（背景・目的・完了条件）
#    - PLAN.md（実行計画の雛形）
#    - HANDOFF.md（初期状態: "未開始"）
#    - VERIFY.md（検証項目テーブルの雛形）
#    - その他空ファイル

# 5. TRIAGE_REPORT 更新
#    .ops/reports/triage/TRIAGE_REPORT_{date}.md に追記
#    - 完了タスクの結果
#    - 新規候補のスコアリング
#    - 推奨次タスク

# 6. Issue 化が必要なら issue.md を生成
#    → 人間に Issue 作成を提案

# 7. 運用ファイルの commit
git add .ops/ 00-docs/
git commit -m "ops: post-completion triage after Q-{id}"
git push origin main
```

### AI の出力（人間が使う）

```
再トリアージ完了。

■ 完了: Q-{id} ({title})
■ unblock された blocked タスク: N 件（→ ready 昇格候補）
■ 新規 proposed: N 件

proposed 上位:
1. Q-{project}-{next-id}-{slug} (score: XX) — {title}
2. Q-{project}-{next-id}-{slug} (score: XX) — {title}

次のアクション:
- proposed を確認し、ready に昇格させるタスクを選んでください
- 昇格コマンド例: mv .ops/queue/proposed/Q-xxx .ops/queue/ready/
```

### 人間がやること

1. **見るファイル**: `.ops/reports/triage/TRIAGE_REPORT_{date}.md` — 再トリアージ結果
2. **見るファイル**: `.ops/queue/proposed/` 配下の各 `task.yaml` — スコアと概要
3. **判断**: 「次にどのタスクを実行するか？」「今日さらに回すか？」

**次タスクを実行する場合**:

```bash
mv .ops/queue/proposed/Q-{project}-{next-id}-{slug} .ops/queue/ready/
```

→ Phase 0 に戻る（サイクル再開）

---

## セッション一覧と切り替えタイミング

| Session | 場所 | Permission Mode | 目的 | 終了トリガー |
|---|---|---|---|---|
| 1: Triage/Ops | メインrepo | plan | worktree 作成・タスク準備 | worktree 作成完了 |
| 2: Execution | worktree 内 | plan → acceptEdits | 実装・テスト・handoff | review/ 移動完了 |
| 3: Review | メインrepo | plan | レビュー・PR作成 | 人間が merge 判断 |
| 3 続き or 4 | メインrepo | plan | cleanup・次タスク生成 | proposed 出力完了 |

**原則**:
- Execution セッションは worktree 内で閉じる
- Review セッションは Execution と独立した context で行う
- Triage/Ops とCleanup は同じメインrepoセッションで連続可能
- いつ session が飛んでも HANDOFF.md から再開できる

---

## 人間の介入ポイントまとめ

| Phase | 介入内容 | 見るもの | 操作 |
|---|---|---|---|
| 0 | 実行タスク選択 | `ready/*/task.yaml`, `BRIEF.md` | 「実行準備してください」プロンプト |
| 2-2 | Permission 切替 | なし | `/permissions` → acceptEdits |
| 3 | Merge 判断 | `REVIEW.md`, GitHub PR diff | `gh pr merge {N} --merge` |
| 4 | Cleanup 指示 | なし | 「完了処理を実行してください」プロンプト |
| 5 | 次タスク選択 | `TRIAGE_REPORT`, `proposed/*/task.yaml` | `mv proposed/Q-xxx ready/` |

**人間がやらないこと**: worktree 作成・削除の実コマンド、task.yaml の手動編集、REVIEW.md の記入、runs/ へのログ保存、status.md の更新、Issue/Project の定型更新

---

## ファイル変更の時系列

```
Phase 1 (準備):
  CREATE  /Users/{user}/GitHub/_worktrees/{project}/Q-xxx/  (worktree)
  MOVE    .ops/queue/ready/Q-xxx → .ops/queue/running/Q-xxx
  UPDATE  running/Q-xxx/task.yaml (status, worktree_path, branch)
  UPDATE  running/Q-xxx/HANDOFF.md (環境状態初期化)
  CREATE  .ops/runs/Q-xxx/run-001/

Phase 2 (実装):
  UPDATE  running/Q-xxx/PLAN.md (差分更新、必要な場合のみ)
  EDIT    worktree 内の対象ファイル群
  UPDATE  running/Q-xxx/RUNLOG.md (各ステップ記録)
  UPDATE  running/Q-xxx/HANDOFF.md (区切りごと)
  UPDATE  running/Q-xxx/DECISIONS.md (判断があれば)
  UPDATE  running/Q-xxx/VERIFY.md (テスト結果)
  CREATE  running/Q-xxx/RESULT.md
  UPDATE  running/Q-xxx/task.yaml (last_run)
  CREATE  .ops/runs/Q-xxx/run-001/manifest.json
  CREATE  .ops/runs/Q-xxx/run-001/summary.md
  CREATE  .ops/runs/Q-xxx/run-001/changed-files.txt
  COMMIT  worktree 内: "Q-{id}: {title}"
  PUSH    task/Q-xxx branch
  MOVE    .ops/queue/running/Q-xxx → .ops/queue/review/Q-xxx

Phase 3 (レビュー):
  CREATE  review/Q-xxx/REVIEW.md
  CREATE  GitHub PR
  UPDATE  review/Q-xxx/task.yaml (pr_id)

Phase 4 (cleanup):
  MOVE    .ops/queue/review/Q-xxx → .ops/queue/done/Q-xxx
  UPDATE  done/Q-xxx/task.yaml (status: done)
  DELETE  worktree
  DELETE  branch (local)
  CLOSE   GitHub Issue (あれば)
  COMMIT  "ops: Q-{id} done — cleanup complete"
  PUSH    main

Phase 5 (次タスク生成):
  UPDATE  00-docs/status.md
  CREATE  .ops/queue/proposed/Q-{next}/ (task packet 一式)
  UPDATE  .ops/reports/triage/TRIAGE_REPORT_{date}.md
  COMMIT  "ops: post-completion triage after Q-{id}"
  PUSH    main
```

---

## クイックリファレンス: コピペ用プロンプト集

### Phase 1: 実行準備

```
タスク Q-{project}-{id}-{slug} の実行準備をしてください。
git worktree add で worktree を作成し、task packet を running/ に移動し、
task.yaml と HANDOFF.md を更新して、起動コマンドを出力してください。
```

### Phase 2: 実装開始

```
このタスクの実行を開始します。
BRIEF.md, PLAN.md, HANDOFF.md を読んで対象ファイルの現状を確認し、
PLAN.md のステップに従って実装してください。
各ステップごとに RUNLOG.md に記録し、HANDOFF.md を随時更新してください。
テスト完了後は VERIFY.md を事実で埋め、RESULT.md を記入し、
review-gate チェック後に task packet を review/ に移動してください。
```

### Phase 3: レビュー

```
タスク Q-{project}-{id}-{slug} のレビューを実施してください。
task packet の全ファイルと diff を確認し、REVIEW.md を書いて PR を作成してください。
```

### Phase 4: 完了処理

```
Q-{project}-{id}-{slug} が merge されました。
task packet を done/ に移動、worktree とブランチを削除、
Issue close と .ops/ の commit/push を実行してください。
```

### Phase 5: 次タスク生成

```
Q-{project}-{id}-{slug} の完了を受けて再トリアージしてください。
blocked の再スキャン、status.md 更新、次の候補を proposed/ に生成し、
TRIAGE_REPORT を更新してください。
```
