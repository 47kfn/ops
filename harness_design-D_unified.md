# AI Harness Design D — Unified Architecture

**作成日**: 2026-04-06
**前提**: A案の統治思想 + B案のClaude Code実装規律 + C案の運用機械を統合した単一設計
**方針**: 段階的アプローチではなく、最初から完成形を1つ定義する

---

# 0. 設計思想

## 0-1. 根幹原則: 非ツール依存の骨格

このハーネスの骨格は **特定のAIツールに依存しない** 。以下の5要素はClaude Code、Codex、Cursor、将来のどのAIエージェントでも成立する普遍的構造である。

1. **Queue State Machine** — タスクの状態遷移を機械的に管理する
2. **Task Packet** — 1タスクの全情報を1フォルダに閉じ込める
3. **1 Task = 1 Worktree = 1 Branch** — 実装の物理的隔離
4. **Handoff-first Recovery** — 会話ではなくファイルで継続する
5. **Structured Failure Classification** — 失敗を機械的に分類し再判定する

## 0-2. 実装原則: Claude Codeの全機能を全開で使う

骨格は非ツール依存だが、**実装レイヤーではClaude Codeの現行機能を制限なく使い切る** 。

- `CLAUDE.md` — プロジェクトの憲法
- `settings.json` — 権限とモードの強制
- `hooks` — ライフサイクルの自動化
- `skills` — 再利用ワークフローのパッケージ化
- `subagents` — 役割分離と context 保護
- `sandbox` — OS レベルの隔離
- `permissions` — deny-first の安全設計
- `gh` CLI — GitHub との直接統合
- `/schedule` + remote triggers — 非同期・自動実行
- GitHub Actions + `@claude` — CI/CD レベルの自動化

## 0-3. 進化前提の設計姿勢

以下のAI進化を **起きるものとして** 設計に織り込む。

| 想定される進化 | 設計上の対応 |
|---|---|
| コンテキスト長の大幅拡大 | handoff/task packet による分割設計は維持。ただし triage の入力量を増やせる |
| モデル精度の継続的向上 | failure-classifier、impact分析、triage scoringの精度が自動的に上がる |
| 永続エージェント / 常駐セッション | triage の常駐化に直接対応。queue + hooks がそのまま活きる |
| Agent Team / 並列実行の公式サポート | 1 task = 1 worktree が並列のベース。subagent 構成をそのまま拡張 |
| auto permission mode の成熟 | deny rules と hooks の安全装置はそのまま残し、中間の承認を省略 |
| MCP / 外部ツール統合の拡大 | `.mcp.json` のスロットは確保済み。必要時に追加 |
| Claude Agent SDK の成熟 | triage の daemon 化、イベント駆動の再トリアージに直接移行可能 |
| GitHub Copilot / Codex 等の競合進化 | 骨格(queue/packet/worktree/handoff)は共通。executor adapter を差し替えるだけ |

## 0-4. 役割はファイル配置・権限・worktree境界で決める

**毎回「あなたは第N層です」と宣言しない** 。Claude Code の各セッションは fresh context で始まる。`CLAUDE.md` と auto memory は読まれるが強制力はない。強制力があるのは settings / permissions / sandbox / hooks である。だから:

- **どの queue ディレクトリを読んでいるか** で triage / execution / review が決まる
- **どの worktree にいるか** で編集スコープが決まる
- **どの permission mode で走っているか** で安全境界が決まる

---

# 1. 全体アーキテクチャ

## 1-1. 4層構造

```
┌──────────────────────────────────────────────────────────┐
│  Layer 0: 可視化・追跡層                                   │
│  GitHub Projects / Issues / PR                            │
│  → 全体の見える化、優先順位UI、議論と追跡                     │
├──────────────────────────────────────────────────────────┤
│  Layer 1: 長期計画・統治層                                  │
│  docs repo + project docs                                 │
│  → phase設計、仕様、architecture、decisions                 │
│  → 正本管理、変更管理、上位→下位の更新順序                     │
├──────────────────────────────────────────────────────────┤
│  Layer 2: 運用管理層                                       │
│  .ops/ (queue, runs, reports, templates)                   │
│  → queue state machine, task packet, triage, handoff       │
│  → AIが直接回す中心                                        │
├──────────────────────────────────────────────────────────┤
│  Layer 3: 実行層                                          │
│  git worktree + Claude session                            │
│  → 1 task = 1 worktree = 1 branch                        │
│  → 実装、テスト、結果記録                                   │
└──────────────────────────────────────────────────────────┘
```

## 1-2. 3つのAgent役割

```
Human (最終意思決定者)
  │
  ├── Triage Agent
  │   計画書・repo状態・進捗・失敗ログ・仕様変更を見て次タスクを提案・優先順位付け
  │   → input:  docs, queue/*, runs/*, GitHub Issues/Projects, recent diffs
  │   → output: TRIAGE_REPORT.md, CANDIDATES.yaml, impact.md, issue drafts
  │
  ├── Execution Agent
  │   ready にある task を 1 task = 1 worktree で実行
  │   → input:  task packet (brief/plan/handoff)
  │   → output: 実装、VERIFY.md, HANDOFF.md, RESULT.md, RUNLOG.md
  │
  └── Review Agent
      実行結果を読み、差し戻し・完了判定の材料を作る
      → input:  diff, plan, verify, result, runlog
      → output: REVIEW.md, merge/hold/reject recommendation
```

これらは「Claude固有の役割名」ではなく **抽象的なAgent役割** である。現在はClaude Codeで実装するが、将来は任意のAIエージェントプラットフォームで差し替え可能。

## 1-3. 人間の責務

人間がやることを明確に限定する。

**必ずやること:**
- `proposed` を見て今日回す本数を決める
- `ready` に昇格させる
- `review` を見て merge / close / hold を決める
- 破壊的操作・課金操作・本番操作の最終承認
- 仕様変更の確定
- blocked タスクの人間判断

**やらなくていいこと:**
- タスク候補の洗い出し（Triage Agent）
- 実装（Execution Agent）
- コードレビューの初期分析（Review Agent）
- 失敗の分類（failure-classifier）
- 仕様変更の影響分析（impact analyzer）
- Issue/Project の定型更新（issue-sync skill）

---

# 2. ディレクトリ設計

## 2-1. メインrepo構成

```text
{project}/
├── CLAUDE.md                          # プロジェクトの憲法
├── .claude/
│   ├── settings.json                  # チーム共有設定 (git管理)
│   ├── settings.local.json            # 個人ローカル設定 (gitignore)
│   ├── rules/
│   │   ├── safety.md                  # 安全ルール
│   │   ├── testing.md                 # テスト方針
│   │   ├── github-ops.md             # GitHub運用ルール
│   │   └── docs-output.md            # ドキュメント出力ルール
│   ├── skills/
│   │   ├── triage-from-state/
│   │   │   └── SKILL.md
│   │   ├── make-task-packet/
│   │   │   └── SKILL.md
│   │   ├── worktree-execution/
│   │   │   └── SKILL.md
│   │   ├── failure-classifier/
│   │   │   └── SKILL.md
│   │   ├── issue-sync/
│   │   │   └── SKILL.md
│   │   ├── review-gate/
│   │   │   └── SKILL.md
│   │   ├── impact-analyzer/
│   │   │   └── SKILL.md
│   │   └── task-handoff/
│   │       └── SKILL.md
│   └── agents/
│       ├── triage.md
│       ├── executor.md
│       ├── reviewer.md
│       ├── log-summarizer.md
│       └── research.md
├── .ops/
│   ├── queue/
│   │   ├── inbox/                     # 雑多な観測、メモ、追加要求
│   │   ├── proposed/                  # Triage Agentが出した候補
│   │   ├── ready/                     # いつでも実行開始可
│   │   ├── running/                   # 実行中
│   │   ├── review/                    # 実装完了、レビュー待ち
│   │   ├── blocked/                   # 人間判断・権限・環境・不確実性で停止
│   │   └── done/                      # 完了
│   ├── runs/                          # 実行ログ (task ID単位)
│   ├── reports/
│   │   ├── triage/                    # TRIAGE_REPORT.md, CANDIDATES.yaml
│   │   └── daily/                     # 日次サマリー
│   ├── templates/
│   │   ├── task-packet/               # task packet雛形
│   │   ├── issue/                     # Issue雛形
│   │   └── review/                    # レビュー雛形
│   └── config/
│       └── scoring-weights.yaml       # triageスコアリング重み設定
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/                     # Claude GitHub Actions
├── 00-docs/                           # プロジェクト別ドキュメント
│   ├── project_overview.md
│   ├── requirements.md
│   ├── roadmap.md
│   ├── status.md
│   ├── decisions.md
│   └── architecture.md
├── 10-apps/
│   └── {app}/CLAUDE.md               # サブディレクトリ別ルール
├── 20-configs/
│   └── CLAUDE.md
├── 30-scripts/
└── .mcp.json                          # MCP設定 (必要時に有効化)
```

## 2-2. Worktree置き場所

```text
/Users/{user}/GitHub/_worktrees/
└── {project}/
    ├── Q-{project}-{id}-{slug}/
    ├── Q-{project}-{id}-{slug}/
    └── Q-{project}-{id}-{slug}/
```

Worktreeはメインrepo外に置く。理由:
- メインrepoのgit状態を汚さない
- 複数worktreeの並列が自然
- 削除時にメインrepoに影響しない

## 2-3. User scope

```text
~/.claude/
├── CLAUDE.md                          # 個人の普遍的な好み
├── settings.json                      # 個人のデフォルト設定
├── skills/                            # 個人用スキル
└── agents/                            # 個人用エージェント
```

User CLAUDE.md には **repo固有の話を書かない** 。書くのは:
- 破壊的変更は必ず確認
- 実装より前に現状把握を優先
- ログとhandoffを必ず残す
- diffは小さくする
- 説明は初心者向けにする

---

# 3. Queue State Machine

## 3-1. 7状態

```
inbox → proposed → ready → running → review → done
                     ↑        │         │
                     │        ↓         │
                     └──── blocked ←────┘
```

| 状態 | 意味 | 遷移条件 |
|---|---|---|
| `inbox` | 雑多な観測、メモ、追加要求、仕様変更通知 | 人間またはAIが投入 |
| `proposed` | Triage Agentが「次これ」と出した候補 | triageスコアリング通過 |
| `ready` | いつでも実行開始可能 | **人間が承認して昇格** |
| `running` | Execution Agentが実行中 | worktree作成 + Claude起動 |
| `review` | 実装完了、レビュー待ち | VERIFY.md + HANDOFF.md更新済み |
| `blocked` | 停止中 | 人間判断/権限/環境/仕様曖昧/失敗 |
| `done` | 完了 | merge済み or 不要判定 |

## 3-2. 遷移ルール

### inbox → proposed
- Triage Agentが inbox を読み、構造化されたtask候補に変換
- `task.yaml` を生成し `proposed/` に配置
- スコアリングを実施

### proposed → ready
- **人間のみが実行可能**
- 今日回す本数を決めて昇格

### ready → running
- worktree 作成
- task packet の `running/` への移動
- Claude session 起動

### running → review
- VERIFY.md の全項目が記入済み
- HANDOFF.md が更新済み
- RESULT.md の下書きが存在

### running → blocked
- failure-classifier が分類した結果 `needs_human_decision` の場合
- 権限不足
- 環境問題
- 仕様曖昧

### review → done
- **人間が merge / close を決定**

### review → blocked
- Review Agent が unsafe change / spec deviation を検出

### blocked → ready
- ブロック原因が解消された場合
- 人間が判断して再投入

### done → archive
- 一定期間後に `.ops/queue/done/` から削除可能
- runs/ のログは保持

## 3-3. イベント駆動の再トリアージ

朝のtriageだけでなく、以下のイベント発生時にも自動で再トリアージを起動する:

| イベント | 再トリアージの内容 |
|---|---|
| run が success/partial/failure/aborted で終了 | 該当taskの再分類 + 次候補の再計算 |
| `blocked` が発生 | unblock task の自動生成検討 |
| review で差し戻し | fix task の生成 or 同task内での修正判断 |
| `inbox` に仕様変更が投入 | impact.md 生成 + 影響先タスクの状態見直し |
| stale `running` が閾値超過 | blocked への自動移行検討 |

これは現在は hooks (Stop, SubagentStop) + skills (triage-from-state) で実装する。将来 Agent SDK が成熟すれば daemon 化する。

---

# 4. Task Packet

## 4-1. 構成

1つのtaskは1つのフォルダで管理する。

```text
.ops/queue/{state}/Q-{project}-{id}-{slug}/
├── task.yaml          # 構造化メタデータ (機械可読の正本)
├── BRIEF.md           # 背景・目的・完了条件
├── PLAN.md            # 実行計画 (差分更新する)
├── HANDOFF.md         # 再開点 (最重要ファイル)
├── VERIFY.md          # 事実のみ (テスト結果等)
├── RESULT.md          # 最終サマリー (完了時のみ)
├── REVIEW.md          # Review Agentの出力
├── RUNLOG.md          # 構造化実行ログ
├── DECISIONS.md       # このtask内の判断記録
├── RISKS.md           # 未解決リスク・既知制約
├── issue.md           # Issue化用の整形済み文章
└── artifacts/
    ├── screenshots/
    ├── test-outputs/
    ├── raw/
    └── diffs/
```

## 4-2. task.yaml 詳細仕様

```yaml
# === 識別 ===
id: "Q-P1-0042"
title: "Dify compose設定の修正"
slug: "dify-compose-fix"
project: "P1"
phase: "phase-1"

# === 状態 ===
status: "running"                     # inbox/proposed/ready/running/review/blocked/done
created_at: "2026-04-06T09:00:00+09:00"
updated_at: "2026-04-06T14:30:00+09:00"

# === 優先度 (Triage Agent が算出) ===
priority:
  score: 91                           # 0-100
  impact: 30                          # 0-30: ゴールへの直接的影響度
  urgency: 20                         # 0-20: 時間的な緊急度
  unblock_value: 18                   # 0-20: 他タスクのブロック解除価値
  effort: 8                           # 0-10: 作業量の少なさ (少ない=高スコア)
  confidence: 10                      # 0-10: 成功確度
  cross_project_leverage: 5           # 0-10: 他プロジェクトへの波及効果

# === スコープ ===
target_paths:
  - "10-apps/dify/"
  - "20-configs/dify/"
forbidden_paths:
  - "20-configs/secrets/"
  - ".env*"

# === 完了定義 ===
definition_of_done:
  - "compose up が正常に起動する"
  - "ヘルスチェックが全サービスpassする"
  - "既存のvolume dataが保持されている"

# === 関連 ===
related_docs:
  - "swe_p1_docs/phase1/dify-setup.md"
related_issues:
  - "#42"
related_tasks:
  - "Q-P1-0041"

# === 実行情報 ===
worktree_path: "/Users/{user}/GitHub/_worktrees/swe_p1/Q-P1-0042-dify-compose-fix"
branch: "task/Q-P1-0042-dify-compose-fix"
issue_id: "#42"
pr_id: ""

# === 実行結果 (run終了時に更新) ===
last_run:
  run_id: "run-003"
  result: "partial"                   # success/partial/failure/aborted
  ended_at: "2026-04-06T14:30:00+09:00"

# === 再分類 (run終了時に必ず設定) ===
next_action:
  type: "continue_same_task"          # continue_same_task/retry_same_task/split_new_task/blocked_waiting_human/done
  reason: "unit tests pass, integration test pending"
  human_dependency: false
  blocked_reason: ""

# === 失敗分類 (failure時のみ) ===
failure_classification:
  category: ""                        # environment/permission/flaky/spec_ambiguity/implementation_bug/needs_human_decision
  detail: ""
  retry_viable: true
  suggested_action: ""
```

## 4-3. 各ドキュメントの詳細仕様

### BRIEF.md

```markdown
# Q-{id}: {title}

## 背景
なぜこのタスクが必要なのか。現在の問題点。

## 目的
このタスクで達成すること。明確に。

## 完了条件
- [ ] 条件1
- [ ] 条件2
- [ ] 条件3

## 触る範囲
- path/to/target

## 触らない範囲
- path/to/forbidden

## 関連
- docs: path/to/spec
- issue: #42
- 前提タスク: Q-{prev-id}
```

### PLAN.md

```markdown
# 実行計画: Q-{id}

## 事前確認
- [ ] 現状のファイル構成を確認
- [ ] 依存関係を確認
- [ ] 既存テストの状態を確認

## 実装ステップ
1. ステップ1の説明
2. ステップ2の説明
3. ステップ3の説明

## テスト方針
- unit: 対象と方法
- integration: 対象と方法
- manual: 確認手順

## リスク
- リスク1とその対策

## 変更履歴
- {date}: 初版作成
- {date}: 仕様変更により ステップ2 を修正 (DEC-xxx参照)
```

### HANDOFF.md (最重要)

```markdown
# Handoff: Q-{id}

## 最終更新: {datetime}

## 現在地
何がどこまで終わったか。具体的に。

## 次の1〜3手
1. 最優先で次にやること
2. その次
3. その次

## 保留事項
- 未決定の事項
- 確認待ちの事項

## 要承認事項
- 人間の判断が必要なこと

## 環境状態
- branch: task/Q-{id}-{slug}
- worktree: /path/to/worktree
- compose状態: up/down
- テスト状態: pass/fail/not-run

## 警告
- 注意すべき状態変化や問題
```

### VERIFY.md

```markdown
# 検証結果: Q-{id}

## 最終更新: {datetime}

| 検証項目 | 結果 | 備考 |
|---|---|---|
| lint | pass/fail/not-run | |
| typecheck | pass/fail/not-run | |
| unit tests | pass/fail/not-run | 失敗テスト名 |
| integration tests | pass/fail/not-run | |
| manual check | pass/fail/not-run | 確認手順と結果 |
| security check | pass/fail/not-run | |
| existing tests regression | pass/fail/not-run | |

## 残存リスク
- リスク1
```

### RUNLOG.md

```markdown
# Run Log: Q-{id}

## Run {number} — {datetime}

### Action
何を試したか（事実のみ）

### Result
どうなったか（事実のみ）

### Root Cause Candidate
原因の仮説

### Next Action
次に試すべきこと

---

## Run {number-1} — {datetime}
(同じ構造で積み上げる。最新が一番上)
```

### REVIEW.md

```markdown
# Review: Q-{id}

## 最終更新: {datetime}
## Reviewer: {agent/human}

## 差分サマリー
- 変更ファイル数: N
- 追加行: N
- 削除行: N

## チェック項目

| 観点 | 判定 | コメント |
|---|---|---|
| PLAN.md との整合 | ok/ng | |
| 完了条件の達成 | ok/ng | |
| 破壊的変更の有無 | ok/ng | |
| セキュリティ | ok/ng | |
| テストの十分性 | ok/ng | |
| ドキュメント更新 | ok/ng | |
| 不要な変更の混入 | ok/ng | |

## 判定
- [ ] approve candidate
- [ ] fix before merge
- [ ] blocked by ambiguity
- [ ] unsafe change — reject

## コメント
詳細な指摘事項
```

---

# 5. Run Log管理

## 5-1. 1 run = 1 フォルダ (task ID単位)

```text
.ops/runs/
└── Q-P1-0042/
    ├── run-001/
    │   ├── manifest.json
    │   ├── summary.md
    │   ├── review-input.md
    │   ├── changed-files.txt
    │   └── artifacts/
    ├── run-002/
    │   └── ...
    └── run-003/
        └── ...
```

task ID 単位でフォルダを切る（日付単位ではない）。理由: タスクの追跡が容易になる。

## 5-2. manifest.json

```json
{
  "task_id": "Q-P1-0042",
  "run_id": "run-003",
  "worktree_path": "/path/to/worktree",
  "branch": "task/Q-P1-0042-dify-compose-fix",
  "model": "claude-sonnet-4-6",
  "permission_mode": "acceptEdits",
  "started_at": "2026-04-06T14:00:00+09:00",
  "ended_at": "2026-04-06T14:30:00+09:00",
  "result": "partial",
  "next_action_type": "continue_same_task",
  "failure_classification": null,
  "linked_issue": "#42",
  "linked_pr": "",
  "changed_files_count": 5,
  "test_results": {
    "lint": "pass",
    "unit": "pass",
    "integration": "fail",
    "manual": "not_run"
  }
}
```

## 5-3. summary.md

```markdown
# Run Summary: Q-P1-0042 / run-003

## 何をしたか
compose.yaml のservice定義を修正し、healthcheckを追加

## 何が終わったか
- compose.yaml の修正完了
- unit tests pass

## 何が残ったか
- integration test で接続タイムアウト
- manual verification 未実施

## 次の1手
.env のDB接続先を確認し、integration testを再実行
```

---

# 6. Claudeスコープ設計

## 6-1. スコープ優先順位

```
Managed (企業配布) > User (~/.claude/) > Project (.claude/) > Local (.claude/settings.local.json) > Subdirectory (subdir/CLAUDE.md)
```

**重要**: `CLAUDE.md` は context（読まれるが強制力なし）。`settings.json` の permissions / hooks は client enforced（強制力あり）。

## 6-2. root CLAUDE.md の内容

```markdown
# Project Constitution

## Mission
この repo の目的と現在フェーズ

## Source of Truth
- 実行の truth: .ops/queue/ と .ops/runs/
- 計画の truth: 00-docs/
- 管理の dashboard: GitHub Projects
- 議論: GitHub Issues

## Non-negotiables
- main working tree で実装しない
- 1 task = 1 worktree
- task開始前に BRIEF.md + PLAN.md + HANDOFF.md を読む
- 実装終了時に VERIFY.md + HANDOFF.md を必ず更新
- 破壊的変更・依存追加・本番操作は止まって確認
- 20-configs/secrets/ は絶対に読まない・編集しない

## Queue State Machine
inbox → proposed → ready → running → review → done
                                        ↕
                                     blocked

## Task Packet Required Files
task.yaml, BRIEF.md, PLAN.md, HANDOFF.md, VERIFY.md

## Done Definition
- VERIFY.md の全項目が pass or justified skip
- HANDOFF.md が最終状態に更新済み
- RESULT.md が記入済み
- Review Agent の approve candidate

## Review Definition
- diff が PLAN.md と整合
- 完了条件を全て満たす
- 破壊的変更がない
- テストが十分

## Docs Repo
swe_p1_docs の場所と参照方法

## Output Rules
- PLAN は差分更新
- VERIFY は事実のみ
- RESULT は完了時のみ
- RUNLOG は「何を試し、どうなったか」のみ（「考えた」は書かない）
```

## 6-3. Subdirectory CLAUDE.md

例: `10-apps/dify/CLAUDE.md`

```markdown
# Dify Subdirectory Rules

## このディレクトリの対象
- compose / env / Dockerfile

## 禁止事項
- volume の直接削除
- 本番の認証情報を含むファイルの編集

## 確認コマンド
- docker compose ps
- docker compose logs --tail=50
- curl http://localhost:3000/health
```

---

# 7. Settings設計

## 7-1. `.claude/settings.json` (チーム共有)

```json
{
  "permissions": {
    "defaultMode": "plan",
    "allow": [
      "Read(**)",
      "Glob(**)",
      "Grep(**)",
      "Bash(npm test*)",
      "Bash(pnpm test*)",
      "Bash(pytest*)",
      "Bash(ruff*)",
      "Bash(eslint*)",
      "Bash(tsc*)",
      "Bash(docker compose ps*)",
      "Bash(docker compose logs*)",
      "Bash(gh issue*)",
      "Bash(gh pr*)",
      "Bash(gh project*)",
      "Bash(git status*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git branch*)",
      "Bash(git worktree*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(docker compose down -v*)",
      "Bash(docker volume rm*)",
      "Edit(20-configs/secrets/**)",
      "Read(20-configs/secrets/**)",
      "Edit(.env*)",
      "Read(.env.production*)"
    ]
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "type": "command",
        "command": "bash .ops/hooks/check-task-context.sh"
      }
    ],
    "PreToolUse": [
      {
        "type": "command",
        "command": "bash .ops/hooks/pre-tool-safety.sh \"$TOOL_NAME\" \"$TOOL_INPUT\""
      }
    ],
    "PostToolUse": [
      {
        "type": "command",
        "command": "bash .ops/hooks/post-tool-audit.sh \"$TOOL_NAME\" \"$TOOL_INPUT\""
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "bash .ops/hooks/session-end-handoff.sh"
      }
    ],
    "SubagentStop": [
      {
        "type": "command",
        "command": "bash .ops/hooks/subagent-result-sync.sh"
      }
    ]
  },
  "model": "sonnet"
}
```

## 7-2. `.claude/settings.local.json` (個人ローカル、gitignore)

```json
{
  "permissions": {
    "allow": [
      "Bash(curl http://localhost*)"
    ]
  },
  "claudeMdExcludes": []
}
```

## 7-3. `~/.claude/settings.json` (個人グローバル)

```json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
```

---

# 8. Hooks設計

Hooks はこのハーネスの **安全装置かつ自動化エンジン** 。`CLAUDE.md` は読まれるが無視されうる。Hooks は client enforced で必ず実行される。

## 8-1. UserPromptSubmit

**目的**: セッション開始時の文脈チェック

```bash
#!/bin/bash
# .ops/hooks/check-task-context.sh

# running task があるか確認
RUNNING_COUNT=$(ls -d .ops/queue/running/*/ 2>/dev/null | wc -l)

# worktree内にいるか確認
CURRENT_DIR=$(pwd)
if [[ "$CURRENT_DIR" == *"_worktrees"* ]]; then
  IS_WORKTREE=true
else
  IS_WORKTREE=false
fi

# worktree内にいるのにrunning taskがない場合は警告
if [ "$IS_WORKTREE" = true ] && [ "$RUNNING_COUNT" -eq 0 ]; then
  echo "WARNING: worktree内にいますが running task がありません。先にtask packetを確認してください。"
fi

# handoff.md の存在確認
if [ "$IS_WORKTREE" = true ]; then
  TASK_DIR=$(find .ops/queue/running/ -maxdepth 1 -mindepth 1 -type d 2>/dev/null | head -1)
  if [ -n "$TASK_DIR" ] && [ ! -f "$TASK_DIR/HANDOFF.md" ]; then
    echo "WARNING: HANDOFF.md が存在しません。先に作成してください。"
  fi
fi
```

## 8-2. PreToolUse

**目的**: 危険操作の遮断

```bash
#!/bin/bash
# .ops/hooks/pre-tool-safety.sh
TOOL_NAME="$1"
TOOL_INPUT="$2"

# secrets ディレクトリへのアクセス遮断
if echo "$TOOL_INPUT" | grep -q "secrets/"; then
  echo "BLOCKED: secrets ディレクトリへのアクセスは禁止されています"
  exit 1
fi

# worktree外への書き込み遮断 (Edit/Write時)
if [[ "$TOOL_NAME" == "Edit" || "$TOOL_NAME" == "Write" ]]; then
  CURRENT_DIR=$(pwd)
  if [[ "$CURRENT_DIR" == *"_worktrees"* ]]; then
    # worktree内の場合、worktree外への書き込みを遮断
    WORKTREE_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
    if [ -n "$WORKTREE_ROOT" ] && ! echo "$TOOL_INPUT" | grep -q "$WORKTREE_ROOT"; then
      echo "BLOCKED: worktree外への書き込みは禁止されています"
      exit 1
    fi
  fi
fi

# deploy系コマンド遮断
if echo "$TOOL_INPUT" | grep -qE "(deploy|migrate.*production|push.*main)"; then
  echo "BLOCKED: 本番系操作は手動で実行してください"
  exit 1
fi
```

## 8-3. PostToolUse

**目的**: 変更の追跡と品質チェック

```bash
#!/bin/bash
# .ops/hooks/post-tool-audit.sh
TOOL_NAME="$1"
TOOL_INPUT="$2"

# ファイル編集後にdiff statを記録
if [[ "$TOOL_NAME" == "Edit" || "$TOOL_NAME" == "Write" ]]; then
  DIFF_STAT=$(git diff --stat 2>/dev/null)
  if [ -n "$DIFF_STAT" ]; then
    # running taskのRUNLOG.mdに追記 (存在すれば)
    TASK_DIR=$(find .ops/queue/running/ -maxdepth 1 -mindepth 1 -type d 2>/dev/null | head -1)
    if [ -n "$TASK_DIR" ] && [ -f "$TASK_DIR/RUNLOG.md" ]; then
      echo -e "\n### File Change $(date +%H:%M)\n\`\`\`\n$DIFF_STAT\n\`\`\`" >> "$TASK_DIR/RUNLOG.md"
    fi
  fi
fi
```

## 8-4. Stop

**目的**: セッション終了時のhandoff保証

```bash
#!/bin/bash
# .ops/hooks/session-end-handoff.sh

TASK_DIR=$(find .ops/queue/running/ -maxdepth 1 -mindepth 1 -type d 2>/dev/null | head -1)

if [ -n "$TASK_DIR" ]; then
  # HANDOFF.md の最終更新時刻チェック
  if [ -f "$TASK_DIR/HANDOFF.md" ]; then
    LAST_MOD=$(stat -f %m "$TASK_DIR/HANDOFF.md" 2>/dev/null || stat -c %Y "$TASK_DIR/HANDOFF.md" 2>/dev/null)
    NOW=$(date +%s)
    DIFF=$((NOW - LAST_MOD))
    # 30分以上更新されていなければ警告
    if [ "$DIFF" -gt 1800 ]; then
      echo "WARNING: HANDOFF.md が30分以上更新されていません。セッション終了前に更新してください。"
    fi
  else
    echo "WARNING: HANDOFF.md が存在しません。"
  fi

  # summary.md を runs/ に保存
  RUN_DIR=".ops/runs/$(basename $TASK_DIR)"
  mkdir -p "$RUN_DIR"
  RUN_COUNT=$(ls -d "$RUN_DIR"/run-* 2>/dev/null | wc -l)
  NEXT_RUN=$((RUN_COUNT + 1))
  NEXT_RUN_DIR="$RUN_DIR/run-$(printf '%03d' $NEXT_RUN)"
  mkdir -p "$NEXT_RUN_DIR"

  # changed files を記録
  git diff --name-only > "$NEXT_RUN_DIR/changed-files.txt" 2>/dev/null
fi
```

## 8-5. SubagentStop

**目的**: subagent結果の自動反映

```bash
#!/bin/bash
# .ops/hooks/subagent-result-sync.sh

# triage結果をqueueに反映
# reviewer結果をREVIEW.mdに反映
# (具体的な実装はsubagentの出力形式に依存)
echo "SubagentStop: 結果の同期を実行"
```

---

# 9. Skills設計

Skills は再利用ワークフローのパッケージ。長いプロンプトを毎回手打ちしない。

## 9-1. triage-from-state

```markdown
---
name: triage-from-state
description: 現在のrepo状態からトリアージレポートと候補リストを生成する
---

# Triage from Current State

## 読むもの
1. CLAUDE.md
2. 00-docs/status.md
3. 00-docs/roadmap.md
4. 00-docs/requirements.md
5. .ops/queue/ の全状態 (特にblocked, running, review)
6. .ops/runs/ の直近ログ (失敗パターンの傾向)
7. .ops/reports/triage/ の前回レポート
8. GitHub Project の現在状態 (`gh project list`, `gh project item-list`)
9. Open issues (`gh issue list`)
10. 最近のPRとコメント (`gh pr list`)
11. recent commits (`git log --oneline -20`)
12. docs repo の関連ドキュメント

## 出力
1. `.ops/reports/triage/TRIAGE_REPORT_{date}.md`
   - blocked の再スキャン結果
   - stale running の検出
   - 仕様変更の影響分析
   - Top 10 候補と優先順位
   - 推奨実行本数
   - 新規 task か継続 task かの判定

2. `.ops/reports/triage/CANDIDATES_{date}.yaml`
   - 各候補の6軸スコアリング
   - task.yaml 形式のメタデータ

## スコアリング基準
scoring-weights.yaml を参照:
- impact (0-30): ゴールへの直接的影響度
- urgency (0-20): 時間的な緊急度
- unblock_value (0-20): 他タスクのブロック解除価値
- effort (0-10): 作業量の少なさ
- confidence (0-10): 成功確度
- cross_project_leverage (0-10): 他プロジェクトへの波及効果

## blocked 再スキャンルール
- 類似原因を束ねる
- 3件以上同種の blocked があれば高優先度の infra-fix task を proposed に追加
- 環境起因の blocked は unblock_value を +5 する
```

## 9-2. make-task-packet

```markdown
---
name: make-task-packet
description: 新しいtask packetの雛形を指定されたqueue状態に生成する
---

# Make Task Packet

## Input
- task_id (自動採番)
- title
- project
- target queue state (proposed/ready)

## 生成するファイル
- task.yaml (テンプレートから生成、priority は空)
- BRIEF.md (背景/目的/完了条件の雛形)
- PLAN.md (事前確認/ステップ/テスト方針の雛形)
- HANDOFF.md (初期状態: "未開始")
- VERIFY.md (検証項目テーブルの雛形)
- REVIEW.md (空)
- RUNLOG.md (空)
- DECISIONS.md (空)
- RISKS.md (空)
- issue.md (空)

## 命名規則
Q-{project}-{4桁連番}-{slug}
例: Q-P1-0042-dify-compose-fix
```

## 9-3. failure-classifier

```markdown
---
name: failure-classifier
description: 実行失敗を自動分類し、次のアクションを提案する
---

# Failure Classifier

## 分類カテゴリ

| カテゴリ | 説明 | 推奨アクション |
|---|---|---|
| environment | compose起動失敗、依存不足、path不整合、auth問題 | blocked + infra-fix task proposed |
| permission | permission deny、sandbox制限、hook block | blocked + needs-human-decision |
| flaky | 一時的なネットワーク、タイミング依存 | retry_same_task |
| spec_ambiguity | どちらのデータ構造が正か不明、acceptance criteria曖昧 | blocked + spec-question.md作成 |
| implementation_bug | コードのロジックエラー | continue_same_task |
| needs_human_decision | 複数の妥当な選択肢があり、方針決定が必要 | blocked_waiting_human |

## 判定手順
1. stderr / stdout の最終50行を読む
2. exit code を確認
3. エラーメッセージのパターンマッチ
4. 直近3 runの結果を比較 (同じエラーの繰り返しか)
5. 分類結果を task.yaml の failure_classification に書き込む
6. 分類結果に応じて next_action を設定

## 出力
- task.yaml の failure_classification と next_action を更新
- blocked の場合は blocked_reason.md を生成
```

## 9-4. impact-analyzer

```markdown
---
name: impact-analyzer
description: 仕様変更の影響範囲を分析し、既存タスクへの影響を判定する
---

# Impact Analyzer

## トリガー
inbox に仕様変更が投入された時

## 手順
1. 変更内容を読む
2. 影響を受ける既存 task を洗い出す (.ops/queue/ready/*, running/*, review/*)
3. 各影響先を分類:
   - `continue`: 変更が小さく、既存変更と整合する
   - `suspend`: 一旦止めて仕様を確認すべき → blocked
   - `discard_and_replan`: 根本的に方針が変わった → blocked + 新 proposed
4. impact.md を生成

## 出力
```text
.ops/queue/inbox/{change-id}/
└── impact.md
```

impact.md には:
- 変更の要約
- 影響を受けるtask一覧と判定
- 新規task候補
- 推奨アクション
```

## 9-5. worktree-execution

```markdown
---
name: worktree-execution
description: worktree生成からClaude起動までの一連の手順を実行する
---

# Worktree Execution

## 手順
1. task.yaml から branch名とworktree pathを決定
2. git worktree add {path} -b {branch}
3. task packet を ready/ → running/ に移動
4. task.yaml の status を "running" に更新
5. worktreeディレクトリ内でClaude起動コマンドを出力

## 起動コマンド例
```bash
cd {worktree_path}
claude --permission-mode acceptEdits --model sonnet
```

## 完了後の片付け
1. review後、merge済みの場合:
   - git worktree remove {path}
   - git branch -d {branch}
   - task packet を done/ に移動
2. blocked の場合:
   - worktreeは残す (凍結)
   - task packet を blocked/ に移動
```

## 9-6. review-gate

```markdown
---
name: review-gate
description: review前に最低限の機械的チェックを実施する
---

# Review Gate

## チェック項目
1. VERIFY.md が存在し、全項目が記入済みか
2. HANDOFF.md が直近30分以内に更新されているか
3. RESULT.md の下書きが存在するか
4. git diff --stat でchanged filesが存在するか
5. task.yaml の definition_of_done の各項目に対応する検証が存在するか
6. PLAN.md と実際の変更の乖離がないか (大まかに)

## 判定
- 全項目 pass → review session 開始可能
- 1項目でも fail → Execution Agent に差し戻し、不足項目を指摘
```

## 9-7. task-handoff

```markdown
---
name: task-handoff
description: セッション終了時にHANDOFF.mdを更新する
---

# Task Handoff

## 更新内容
1. 現在地: 何がどこまで終わったか
2. 次の1〜3手: 最優先の次アクション
3. 保留事項: 未決定のこと
4. 要承認事項: 人間判断が必要なこと
5. 環境状態: branch, worktree, compose, テスト状態
6. 警告: 注意すべき変化

## ルール
- session終了前に必ず実行
- compact前にも実行
- 区切りの良いポイントでも随時更新
```

## 9-8. issue-sync

```markdown
---
name: issue-sync
description: task packetの状態をGitHub Issue/Projectに同期する
---

# Issue Sync

## 同期方向
task packet → GitHub (one-way が基本)

## 同期内容
- task.yaml の status → Issue labels / Project Status field
- task.yaml の priority.score → Project Priority field
- RESULT.md → Issue close comment
- REVIEW.md → PR comment

## 使用コマンド
- gh issue create / edit / close
- gh project item-edit
- gh pr create / comment
```

---

# 10. Subagents設計

Subagentsは **main sessionを軽く保つため** に使う。常時使うのではなく、context を汚しやすい作業を切り出す。

## 10-1. triage (subagent)

```markdown
---
name: triage
description: 状況整理と候補作成を行うTriageエージェント
permissionMode: plan
tools:
  - Read
  - Glob
  - Grep
  - Bash(gh *)
  - Write(.ops/reports/**)
  - Write(.ops/queue/inbox/**)
  - Write(.ops/queue/proposed/**)
model: sonnet
---

# Triage Agent

## 役割
- docs repo と .ops/queue/ を見て proposed task を作る
- 優先度を6軸スコアリングで付ける
- blocked を再スキャンし、類似原因を束ねる
- Issue化すべきものを判断する
- 仕様変更の影響を分析する

## 制約
- コードの編集はしない
- queue の ready 以降への移動はしない (人間の権限)
- network access は gh CLI のみ
```

## 10-2. executor (subagent)

```markdown
---
name: executor
description: 1 taskを1 worktreeで実装する実行エージェント
permissionMode: default
tools:
  - Read
  - Edit
  - Write
  - Bash
  - Glob
  - Grep
model: sonnet
---

# Execution Agent

## 役割
- ready の task を 1 worktree で実装する
- PLAN.md に従う
- VERIFY.md, HANDOFF.md, RESULT.md, RUNLOG.md を更新する

## 制約
- 自分の worktree 配下のみ編集可
- 親 repo と他 worktree は deny
- 20-configs/secrets/** は deny
- network はデフォルト off (task.yaml で明示された場合のみ)
- main branch への直接コミット禁止
```

## 10-3. reviewer (subagent)

```markdown
---
name: reviewer
description: 差分レビューと完了判定を行うReviewエージェント
permissionMode: plan
tools:
  - Read
  - Glob
  - Grep
  - Write(.ops/queue/**/REVIEW.md)
model: sonnet
---

# Review Agent

## 役割
- diff, PLAN.md, VERIFY.md, RESULT.md, RUNLOG.md を読む
- 仕様逸脱、破壊的変更、ログ不足、テスト不足を指摘
- REVIEW.md を書く
- approve / fix / blocked / reject の判定を出す

## 制約
- コードの編集はしない
- 判定は出すが最終決定は人間
```

## 10-4. log-summarizer (subagent)

```markdown
---
name: log-summarizer
description: 長いログを要約し、blocked原因を抽出する
permissionMode: plan
tools:
  - Read
  - Grep
model: sonnet
---

# Log Summarizer

## 役割
- 長い stdout/stderr を短くまとめる
- blocked 原因を抽出する
- 同種のエラーパターンを検出する
```

## 10-5. research (subagent)

```markdown
---
name: research
description: ログ探索と既存コード構造把握を行う調査エージェント
permissionMode: plan
tools:
  - Read
  - Glob
  - Grep
  - Bash(gh *)
  - Bash(git log *)
  - Bash(git blame *)
model: sonnet
---

# Research Agent

## 役割
- ログ探索
- 既存コードの構造把握
- Issue/PR コメント読み取り
- 高ボリューム探索の context 汚染を防ぐ
```

---

# 11. Permissions設計

## 11-1. Agent役割別の権限マトリクス

| 権限 | Triage Agent | Execution Agent | Review Agent | Log Summarizer |
|---|---|---|---|---|
| permissionMode | plan | default/acceptEdits | plan | plan |
| Read (全般) | allow | allow | allow | allow |
| Edit (worktree内) | deny | allow | deny | deny |
| Edit (.ops/queue/**) | allow (proposed/inboxのみ) | deny | allow (REVIEW.mdのみ) | deny |
| Write (worktree内) | deny | allow | deny | deny |
| Write (.ops/**) | allow (reports/queue) | allow (HANDOFF等) | allow (REVIEW.md) | deny |
| Bash (テスト系) | deny | allow | deny | deny |
| Bash (gh CLI) | allow | allow | allow | deny |
| Bash (git操作) | allow (read系のみ) | allow | allow (read系のみ) | deny |
| Bash (docker) | deny | allow (compose ps/logs) | deny | deny |
| network | gh CLIのみ | task.yamlで明示時のみ | gh CLIのみ | deny |
| secrets/** | deny | deny | deny | deny |

## 11-2. deny-first の原則

permissions は deny → ask → allow の順で評価される。最初にマッチしたルールが勝つ。したがって:

1. 最初に全てのdenyを定義
2. 次にaskのデフォルト
3. 最後に明示的なallow

---

# 12. GitHub Issues / Projects 連携

## 12-1. 役割分担

| 要素 | 役割 |
|---|---|
| **GitHub Project** | 全体管理と優先順位のUI。custom fieldsでメタデータ管理 |
| **GitHub Issue** | 1 taskの外向き管理単位。議論・追跡 |
| **task packet** | Agentが読む実行ドキュメントの真実 |
| **worktree** | 実装の隔離現場 |

**Issue は管理、task packet は実行の真実** 。実行の詳細ログや handoff を Issue に全部書かない。

## 12-2. Issue にする基準

### Issue にする
- 数時間〜数日かかる
- PR単位になる
- 他projectと比較対象になる
- 仕様変更の議論がある
- 後で検索したい

### queue だけで済ませる
- 設定名1個修正
- path typo 修正
- flaky 再試行
- 軽微な cleanup

## 12-3. Project の custom fields

| Field | Values | 用途 |
|---|---|---|
| `Project` | P1 / P2 / P3 | プロジェクト識別 |
| `Phase` | phase-1 / phase-2 / ... | 現在フェーズ |
| `Status` | triage/ready/running/review/done/blocked | queue状態との同期 |
| `Priority` | Critical/High/Medium/Low | 優先度 |
| `Priority Score` | 0-100 | 6軸スコアの合計 |
| `Effort` | XS/S/M/L/XL | 作業規模 |
| `Area` | app名/infra/docs | 対象領域 |
| `AI Suggested` | yes/no | AI提案か人間起案か |
| `Needs Human` | yes/no | 人間判断待ちか |
| `Spec Changed` | yes/no | 仕様変更あり |
| `Branch` | task/Q-xxx | ブランチ名 |
| `Task Pack Path` | .ops/queue/xxx | task packetの場所 |
| `Linked Task ID` | Q-P1-xxxx | task ID |

## 12-4. Issue → task packet の接続

```
Issue #42 "Dify compose修正"
  ↕ (双方向リンク)
Q-P1-0042-dify-compose-fix/
  ├── task.yaml (issue_id: "#42")
  └── issue.md (Issue作成用の整形文)
```

PR は Issue にリンクし、merge 時に自動 close させる。

## 12-5. Claude GitHub Actions

local terminal 運用が本命だが、以下の用途でGitHub Actionsを使う:

- Issue で `@claude` に初動整理をさせる
- PR で `@claude review` 的な運用
- CI failure 時の自動分析
- nightly review / triage

`/install-github-app` でセットアップ。必要な権限: Contents, Issues, Pull requests の read/write。

---

# 13. Triage ワークフロー

## 13-1. 定例triage (朝 or 作業開始時)

```
1. Triage Agent 起動
   claude --permission-mode plan --model sonnet

2. 入力を読む
   - 00-docs/status.md, roadmap.md, requirements.md
   - .ops/queue/ の全状態
   - .ops/runs/ の直近失敗ログ
   - GitHub Project / Issues / PR
   - docs repo の関連ドキュメント
   - recent commits

3. 分析
   - blocked の再スキャン → 類似原因の束ね
   - stale running の検出 (閾値: 24時間更新なし)
   - 仕様変更の影響分析
   - 完了タスクの後続候補の検出

4. 出力
   - TRIAGE_REPORT_{date}.md
   - CANDIDATES_{date}.yaml
   - 必要に応じて impact.md
   - 必要に応じて inbox への新規候補投入

5. 人間の介入
   - proposed 上位を見る
   - 今日回す本数を決める
   - ready に昇格させる
```

## 13-2. イベント駆動triage

以下のイベントで自動起動:

| イベント | 実装方法 (現在) | 実装方法 (将来) |
|---|---|---|
| run 終了 | Stop hook → triage skill 呼び出し | Agent SDK daemon |
| blocked 発生 | PostToolUse hook → failure-classifier | Agent SDK event handler |
| review 差し戻し | SubagentStop hook → triage update | Agent SDK event handler |
| inbox に仕様変更 | UserPromptSubmit hook → impact-analyzer | Agent SDK watcher |
| stale running 検出 | `/schedule` による定期チェック | Agent SDK cron |

## 13-3. スコアリング重み設定

`.ops/config/scoring-weights.yaml`:

```yaml
# 各軸の最大値と重み
# score = impact + urgency + unblock_value + effort + confidence + leverage

impact:
  max: 30
  description: "ゴールへの直接的影響度"
  guide:
    30: "フェーズ完了に直結"
    20: "主要機能の実装"
    10: "品質向上・改善"
    0: "nice-to-have"

urgency:
  max: 20
  description: "時間的な緊急度"
  guide:
    20: "今日やらないと他タスクが全停止"
    15: "今週中に必要"
    10: "今フェーズ中に必要"
    5: "いつかやる"
    0: "急がない"

unblock_value:
  max: 20
  description: "他タスクのブロック解除価値"
  guide:
    20: "3件以上の blocked を解除"
    15: "2件の blocked を解除"
    10: "1件の blocked を解除"
    5: "間接的に他タスクを助ける"
    0: "独立タスク"

effort:
  max: 10
  description: "作業量の少なさ (少ない=高スコア)"
  guide:
    10: "30分以内"
    7: "1-2時間"
    5: "半日"
    3: "1日"
    1: "数日"

confidence:
  max: 10
  description: "成功確度"
  guide:
    10: "確実にできる"
    7: "高確率"
    5: "やってみないとわからない"
    3: "かなり不確実"
    1: "実験的"

cross_project_leverage:
  max: 10
  description: "他プロジェクトへの波及効果"
  guide:
    10: "全プロジェクトに直接影響"
    5: "1-2プロジェクトに間接影響"
    0: "このプロジェクトのみ"
```

---

# 14. 実行ワークフロー

## 14-1. 標準成功フロー

```
ready
  ↓
1. worktree 作成
   git worktree add {path} -b {branch}

2. task packet を running/ に移動

3. Claude 起動 (plan mode)
   cd {worktree_path}
   claude --permission-mode plan --model sonnet

4. 事前確認
   BRIEF.md → PLAN.md → HANDOFF.md を読む
   対象ファイルの現状確認
   依存関係の確認
   既存テストの状態確認

5. PLAN.md 更新 (必要なら)

6. permission mode 切り替え
   /permissions → acceptEdits

7. 実装

8. テスト
   VERIFY.md を事実で埋める

9. HANDOFF.md 更新
   RESULT.md 下書き
   RUNLOG.md 更新

10. review-gate skill で機械的チェック

11. task packet を review/ に移動
  ↓
review
  ↓
12. Review Agent 起動 (別 session, plan mode)
    REVIEW.md を書く

13. 人間が PR / merge 判断

14. merge → Issue close → Project 更新

15. task packet を done/ に移動
    worktree 削除
  ↓
done
```

## 14-2. 失敗パターンと対応

### 環境失敗 (compose起動、依存不足、path不整合、auth問題)

```
1. failure-classifier skill で "environment" に分類
2. blocked/ に移動
3. blocked_reason.md に原因を書く
4. 同種失敗が3件以上 → 高優先度の infra-fix task を proposed に追加
5. task.yaml: next_action.type = "blocked_waiting_human"
```

### 権限失敗 (permission deny, sandbox制限, hook block)

```
1. failure-classifier skill で "permission" に分類
2. blocked/ に移動
3. needs-human-decision: true
4. 人間が local settings で一時開放するか判断
5. bypass は原則しない
```

### 仕様曖昧

```
1. failure-classifier skill で "spec_ambiguity" に分類
2. blocked/ に移動
3. spec-question.md を作る
4. Issue に落として議論
```

### flaky (一時的なネットワーク、タイミング依存)

```
1. failure-classifier skill で "flaky" に分類
2. task.yaml: next_action.type = "retry_same_task"
3. 自動retry (最大3回)
4. 3回失敗 → blocked + "environment" に再分類
```

### 実装バグ

```
1. failure-classifier skill で "implementation_bug" に分類
2. task.yaml: next_action.type = "continue_same_task"
3. RUNLOG.md に root cause candidate を記録
4. 次の run で修正を試みる
```

### Claude自体の運用失敗 (context膨張、迷走、command不調)

```
1. /compact で圧縮
2. /rewind で巻き戻し
3. HANDOFF.md を正として更新
4. 必要なら session を切り替え
5. /debug, /doctor で診断
6. 最悪の場合: 新 session を plan で開始し、HANDOFF.md から再開
```

---

# 15. タスク継続パターン

## 15-1. 同じ日で継続

- session そのまま継続
- 定期的に `/compact`
- HANDOFF.md は区切りごとに更新
- RUNLOG.md に run 単位で記録

## 15-2. 翌日継続

```
1. 同じ worktree に入る
   cd {worktree_path}

2. Claude 起動
   claude

3. /resume (前回 session の再開を試みる)

4. HANDOFF.md を読む (これが source of truth)

5. PLAN.md を再確認

6. VERIFY.md の最新状態を確認

7. permissions 再承認 (session-scoped のため)

8. 続行
```

## 15-3. Session切り替えでの継続

**継続の基点は会話ではなく handoff** 。`/resume` は便利だが source of truth にはしない。

```
1. 新 session 起動
   claude --permission-mode plan

2. まず HANDOFF.md を読む

3. PLAN.md と VERIFY.md を確認

4. 前回の session がどこで終わったかは HANDOFF.md に書いてある

5. そこから続行
```

## 15-4. Mac 再起動 / tmux 切断からの復帰

致命傷にならない理由:
- worktree は残る
- queue state は残る
- runs/ のログは残る
- HANDOFF.md も残る
- task.yaml も残る

会話が飛んでも作業は飛ばない。

---

# 16. 仕様変更パターン

## 16-1. 仕様変更が来た時

```
1. 変更要求を inbox/ に置く
   .ops/queue/inbox/CHANGE-{id}-{slug}/
   └── change-request.md

2. Triage Agent (または impact-analyzer skill) が影響を洗う

3. impact.md を作る
   影響先を分類:
   - continue: 変更が小さく、既存変更と整合する
   - suspend: 一旦止めて仕様確認 → blocked
   - discard_and_replan: 根本的に変わった → blocked + 新 proposed

4. 必要なら既存 ready/running を blocked に移す

5. 変更理由を 00-docs/decisions.md に記録

6. 新しい proposed を生成

7. PLAN.md を更新 (影響を受ける running task がある場合)
```

## 16-2. ルール

- 走っている task を silent update しない
- 仕様変更が大きい時は task ID を分ける
- 変更理由を必ず decisions.md に残す
- 実装を続ける前に PLAN.md を更新する

---

# 17. ドキュメント管理

## 17-1. 正本ルール

| 種類 | 正本 |
|---|---|
| 目的 | 00-docs/project_overview.md |
| 仕様 | 00-docs/requirements.md |
| 優先順位・段階 | 00-docs/roadmap.md |
| 現在地 | 00-docs/status.md |
| 判断理由 | 00-docs/decisions.md |
| 実行の状態 | .ops/queue/ (task.yaml が機械可読の正本) |
| 実行の詳細 | .ops/runs/ (manifest.json + summary.md) |
| 管理のUI | GitHub Projects |
| 議論 | GitHub Issues |
| ハーネス設計 | ops/harness_design-D_unified.md (人間用) |

## 17-2. 更新順序

変更が起きた場合は上位から下位へ更新する:

```
project_overview.md (必要なら)
  → requirements.md
    → roadmap.md
      → decisions.md
        → status.md
          → 関連 Issue
            → 関連 task packet
              → 関連ログ
```

Issue だけ先に書き換える運用は禁止。上位方針とのズレを生むため。

## 17-3. docs repoに残すもの

- phase設計
- long-lived decisions
- architecture
- schema rationale
- acceptance criteriaの変更
- 重要な失敗学習

## 17-4. main repoに残すもの

- queue
- runs
- task packets
- issue drafts
- operational reports
- templates

---

# 18. ヘルスチェック

## 18-1. 定期チェック項目

Triage Agent がセッション開始時に確認:

- [ ] 00-docs/status.md と queue の実態が一致しているか
- [ ] blocked/ に1週間以上放置されたタスクがないか
- [ ] running/ に24時間以上更新のないタスクがないか
- [ ] review/ にレビュー未着手のタスクがないか
- [ ] HANDOFF.md のない running task がないか
- [ ] 同種の failure が3件以上蓄積していないか
- [ ] proposed/ のスコアが最新状態を反映しているか

## 18-2. 実施タイミング

- Triage session 開始時 (毎日)
- `/schedule` による定期チェック (将来は4時間ごと)
- イベント駆動の再トリアージ時

---

# 19. 安全設計

## 19-1. 重点リスク

| リスク | 対策レイヤー |
|---|---|
| ファイル削除・移動 | permissions deny + hooks PreToolUse |
| 秘密情報の露出 | permissions deny (secrets/**) + sandbox |
| 外部送信 | sandbox network isolation |
| 勝手な push | permissions deny (git push --force) |
| 課金操作 | permissions deny + 人間承認 |
| 本番系接続 | sandbox + permissions deny |
| main branch直接実装 | hooks PreToolUse + CLAUDE.md |
| worktree外書き込み | hooks PreToolUse + sandbox |
| volume破壊 | permissions deny (docker compose down -v) |

## 19-2. エスカレーションルール

以下の場合、Execution Agent は即座に blocked にして人間に報告:

- **課金リスク**: 想定外のAPI呼び出し・高コスト処理
- **データ削除リスク**: 取り返しのつかないデータ削除
- **セキュリティ問題**: 秘匿情報の露出・認証情報の漏洩
- **致命的ブロッカー**: 環境が壊れて全体進行が止まる

## 19-3. コスト管理

- 全ツールをサブスクリプション範囲内で使用
- Triage / Review は plan mode (低コスト)
- Execution は task に応じてモデル選択 (sonnet が基本、重い task は opus)
- 不要なリトライでトークンを消費しない (flaky は3回まで)
- `/usage` で定期的にトークン使用量を確認

---

# 20. CLI / コマンド運用

## 20-1. 毎日使う

| コマンド | 用途 |
|---|---|
| `claude` | 起動 |
| `claude --permission-mode plan` | triage/review用の安全起動 |
| `claude --permission-mode acceptEdits` | 実装用の起動 |
| `claude --model sonnet` | モデル指定 |
| `/help` | コマンド一覧確認 |
| `/resume` | 過去session再開 |
| `/compact` | context圧縮 |
| `/agents` | subagent読み込み確認 |
| `/permissions` | 権限確認・調整 |
| `/model` | モデル切り替え |
| `/tasks` | background tasks確認 |
| `/skills` | skill一覧確認 |
| `/memory` | auto memory確認 |
| `/rename` | session名変更 |

## 20-2. 問題が出た時

| コマンド | 用途 |
|---|---|
| `/debug` | Claude Code自体の不調確認 |
| `/doctor` | 環境診断 |
| `/sandbox` | sandbox状態確認 |
| `/hooks` | hook設定確認・調整 |
| `/rewind` | 会話/コードの巻き戻し |

## 20-3. GitHub連携

| コマンド | 用途 |
|---|---|
| `/install-github-app` | GitHub Actions連携セットアップ |
| `/pr-comments` | PRコメント取得 |
| `/security-review` | セキュリティレビュー |
| `gh issue create/edit/close` | Issue操作 |
| `gh pr create/comment` | PR操作 |
| `gh project item-list/item-edit` | Project操作 |

## 20-4. 自動化

| コマンド | 用途 |
|---|---|
| `/schedule` | 定期タスク (triage, health check) |
| remote triggers | イベント駆動の自動実行 |

## 20-5. 注意

- `/help` を正とする。fixed な一覧を前提にしない
- version と auth によって hidden になる slash commands がある
- `auto` permission mode は成熟するまで主役にしない

---

# 21. 日次ワークフロー

## 朝 or 作業開始時

```
1. Triage session 起動
   claude --permission-mode plan --model sonnet

2. /agents でsubagent読み込み確認

3. triage-from-state skill 実行
   → TRIAGE_REPORT, CANDIDATES 生成
   → blocked 再スキャン
   → stale running 検出

4. TRIAGE_REPORT を読む

5. 今日回す本数を決める

6. proposed → ready に昇格

7. ready の各taskについて worktree-execution skill 実行
   → worktree作成、running移動、Claude起動コマンド出力
```

## 実装中

```
1. 各task の worktree で Claude 起動
   cd {worktree_path}
   claude --permission-mode acceptEdits

2. BRIEF.md → PLAN.md → HANDOFF.md を読む

3. 実装 → テスト → VERIFY.md 更新

4. 定期的に /compact

5. 区切りごとに task-handoff skill 実行
   → HANDOFF.md 更新

6. 完了したら review-gate skill
   → 機械的チェック通過後 review/ に移動
```

## 終了前

```
1. Review session 起動 (別session)
   claude --permission-mode plan

2. review 対象の task pack を読む

3. REVIEW.md を書く

4. issue-sync skill で GitHub 更新

5. merge 判断

6. done に移動、worktree cleanup
```

## 起床後 / 帰宅後

```
1. Triage session で状況確認

2. 確認すべきもの:
   - done/ に入ったタスク → RESULT.md を読む
   - blocked/ に入ったタスク → blocked_reason.md を読む
   - review/ に入ったタスク → REVIEW.md を読む
   - running/ で stale なタスク → HANDOFF.md を読む

3. queue のフォルダ構造がそのまま進捗を示す:
   done/: 3件 → 3件完了
   review/: 1件 → 1件レビュー待ち
   blocked/: 1件 → 1件停止中
   running/: 0件 → 実行中なし

4. 次のサイクルへ
```

---

# 22. 失敗からの再開設計

## 22-1. このハーネスで一番大事なこと

**会話が飛んでも作業は飛ばない** 構造にする。

| 何が飛んでも | 残るもの | 再開方法 |
|---|---|---|
| Claude session | worktree, queue, runs, HANDOFF | HANDOFF.md から新session |
| tmux session | 同上 | 同上 |
| Mac再起動 | 同上 | 同上 |
| Git操作ミス | worktree は独立 | worktree から復帰 |

## 22-2. blocked が溜まった時

```
1. Triage Agent が毎朝 blocked/ を再スキャン
2. 類似原因を束ねる
3. 束ねた原因に対して:
   - 環境系: 高優先度の infra-fix task を proposed に追加
   - 権限系: 人間に permission 変更を提案
   - 仕様系: Issue で議論を開始
4. 3件以上の同種 blocked は自動的に unblock_value +15 で proposed 生成
```

## 22-3. 全体が混乱した時

```
1. 全 running を一旦 blocked に移す
2. Triage session を clean に起動
3. 00-docs/status.md を人間が更新
4. Triage Agent に再スキャンさせる
5. 最小限の ready から再開
```

---

# 23. 導入チェックリスト

この設計は段階的アプローチではなく完成形として定義しているが、物理的なセットアップの順序は以下:

### Step 1: 基盤ファイル作成
- [ ] root CLAUDE.md
- [ ] .claude/settings.json
- [ ] .claude/settings.local.json (gitignore)
- [ ] .ops/ ディレクトリ構造 (queue/runs/reports/templates/config)
- [ ] 00-docs/ (project_overview, roadmap, status, requirements, decisions)
- [ ] .ops/config/scoring-weights.yaml
- [ ] .ops/templates/task-packet/ (雛形)

### Step 2: Claude Code機能の設定
- [ ] .claude/agents/ (triage, executor, reviewer, log-summarizer, research)
- [ ] .claude/skills/ (triage-from-state, make-task-packet, failure-classifier, impact-analyzer, worktree-execution, review-gate, task-handoff, issue-sync)
- [ ] .claude/rules/ (safety, testing, github-ops, docs-output)
- [ ] .ops/hooks/ (check-task-context, pre-tool-safety, post-tool-audit, session-end-handoff, subagent-result-sync)

### Step 3: GitHub連携
- [ ] .github/ISSUE_TEMPLATE/
- [ ] .github/PULL_REQUEST_TEMPLATE.md
- [ ] GitHub Project の作成と custom fields 設定
- [ ] /install-github-app (GitHub Actions)

### Step 4: 運用開始
- [ ] 最初の triage session を実行
- [ ] 最初の task packet を生成
- [ ] 最初の worktree で実装を実行
- [ ] 最初の review を実施
- [ ] 最初の done サイクルを完了

### Step 5: 自動化
- [ ] /schedule で定期 triage を設定
- [ ] remote triggers の設定
- [ ] GitHub Actions の自動化ルール

---

# 24. 最終判断

## 採用すべきもの (全て初日から)

- root / subdir CLAUDE.md
- .claude/settings.json + settings.local.json
- subagents (triage, executor, reviewer, log-summarizer, research)
- skills (8個)
- hooks (5種類)
- queue state machine (7状態)
- task packet (10ファイル + artifacts)
- 1 task = 1 worktree = 1 branch
- 6軸スコアリング
- failure-classifier
- impact-analyzer
- イベント駆動の再トリアージ
- run終了時の自動再分類
- Issue / Projects 併用 (issue-sync skill)
- HANDOFF.md を source of truth にする
- manifest.json による構造化ログ
- deny-first permissions
- ヘルスチェック
- decisions.md による判断記録

## やるが、AI/ツール進化に応じて置き換わるもの

| 現在の実装 | 将来の置き換え候補 |
|---|---|
| 手動session起動 | 永続エージェント / Agent SDK daemon |
| /schedule による定期triage | Agent SDK cron / event handler |
| hooks による安全装置 | auto permission mode の成熟 |
| subagent 手動定義 | Agent Team 公式サポート |
| gh CLI による Issue 同期 | GitHub Actions の深い統合 |
| plan/acceptEdits の手動切替 | auto mode の安定化 |

## 一番危険なもの (絶対にやらない)

- 毎回プロンプトで「あなたは第N層です」と宣言するだけの運用
- 会話履歴を記憶の本体にする運用
- micro task まで全部 issue 化する運用
- main working tree で直接実装する運用
- HANDOFF.md なしで session 依存する運用
- deny rules なしで全許可する運用
- task.yaml なしで状態をMarkdownだけで管理する運用

---

この設計の骨格 (queue / task packet / worktree / handoff / failure-classifier / scoring) は **どのAIエージェントプラットフォームでも成立する** 。Claude Code 固有部分 (skills / hooks / subagents / permissions) は、将来ツールが変わっても「同等の概念」が存在する可能性が極めて高く、完全な書き直しにはならない。

今の Claude Code 運用にも耐え、モデル性能向上・Agent Team・永続エージェント・auto mode の成熟にもそのまま追従できる設計である。
