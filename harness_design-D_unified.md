# AI Harness Design D — Unified Architecture

**作成日**: 2026-04-06
**D2統合日**: 2026-04-10
**前提**: A案の統治思想 + B案のClaude Code実装規律 + C案の運用機械を統合した単一設計
**方針**: 段階的アプローチではなく、最初から完成形を1つ定義する
**D2方針**: 役割を構造で決定し、人間の介入を最小化し、自律的な並列運転を可能にする

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

## 0-3. 自律運転のための追加原則

Design D の骨格の上に、「プロンプトを書かなくても成立する」設計を実現するための原則を追加する。

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
| blocked → ready (auto-unblock) | **AI自動（条件付き）** | auto-unblock ルールに該当する場合のみ |

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

## 0-4. 進化前提の設計姿勢

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

## 0-5. 役割はファイル配置・権限・worktree境界で決める

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
  ├── Triage Agent — メインrepo + plan mode で自動認識
  │   計画書・repo状態・進捗・失敗ログ・仕様変更を見て次タスクを提案・優先順位付け
  │   → input:  docs, queue/*, runs/*, GitHub Issues/Projects, recent diffs
  │   → output: TRIAGE_REPORT.md, CANDIDATES.yaml, impact.md, issue drafts
  │
  ├── Execution Agent — worktree + acceptEdits mode で自動認識
  │   ready にある task を 1 task = 1 worktree で実行
  │   → input:  task packet (brief/plan/handoff)
  │   → output: 実装、VERIFY.md, HANDOFF.md, RESULT.md, RUNLOG.md
  │
  └── Review Agent — メインrepo + plan mode + review/ 対象で自動認識
      実行結果を読み、差し戻し・完了判定の材料を作る
      → input:  diff, plan, verify, result, runlog
      → output: REVIEW.md, merge/hold/reject recommendation
```

これらは「Claude固有の役割名」ではなく **抽象的なAgent役割** である。現在はClaude Codeで実装するが、将来は任意のAIエージェントプラットフォームで差し替え可能。役割の決定は cwd + queue state + permission mode から自動的に行われる。

## 1-3. 人間の責務

人間がやることを明確に限定する。

**必ずやること（3つだけ）:**
1. `proposed` を見て `ready` に昇格させる（「今日何を回すか」を決める）
2. `review` を見て merge / close / hold を決める
3. `blocked` を見て判断する（破壊的操作・課金操作・本番操作・仕様確定）

**やらなくていいこと:**
- タスク候補の洗い出し → Triage Agent が自動
- 実装 → Execution Agent が自動
- コードレビューの初期分析 → Review Agent が自動
- 失敗の分類と再試行判断 → failure-classifier が自動
- ready → running の起動 → 自動
- running → review の遷移 → review-gate 通過で自動
- 次タスクの生成 → run 終了時に Triage が自動再評価
- 仕様変更の影響分析 → impact-analyzer が自動
- Issue/Project の定型更新 → issue-sync skill が自動
- 日次レポート生成 → /schedule で自動

---

# 2. Claude Code スコープ設計 — 何をどこに持たせるか

## 2-1. スコープの性質と役割分担

Claude Code には性質の異なる複数のスコープがある。これらを明確に使い分ける。

| スコープ | 性質 | 強制力 | 用途 |
|---|---|---|---|
| **settings.json permissions** | client enforced | **強制** | 許可/拒否の境界。違反不可能 |
| **settings.json hooks** | client enforced | **強制** | ライフサイクルの自動処理。必ず実行される |
| **rules/** | context injection | 中（読まれるが無視されうる） | 状況依存のガイダンス。条件付きで注入される |
| **CLAUDE.md** | context | 低（読まれるが強制力なし） | プロジェクトの憲法。方針と構造の説明 |
| **skills/** | invocable workflow | なし（呼ばれた時だけ） | 再利用可能な手順のパッケージ |
| **agents/** | subagent definition | 中（権限で制約） | 役割分離と context 保護 |

## 2-2. スコープ優先順位

```
Managed (企業配布) > User (~/.claude/) > Project (.claude/) > Local (.claude/settings.local.json) > Subdirectory (subdir/CLAUDE.md)
```

**重要**: `CLAUDE.md` は context（読まれるが強制力なし）。`settings.json` の permissions / hooks は client enforced（強制力あり）。

## 2-3. 各スコープに持たせるもの

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
- role-detection.md: 役割自動判定ルール
- autonomous-ops.md: 自律運転ルール

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
- dashboard: 人間向け状況サマリー生成
- auto-advance: 自動状態遷移

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

## 2-4. 「短いプロンプトで成立する」メカニズムの全体像

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

## 2-5. root CLAUDE.md の内容

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

## Role Detection
- cwd が _worktrees/ 配下 → Executor
- cwd がメインrepo + review/ にタスクあり → Reviewer
- cwd がメインrepo + それ以外 → Triage
- permission mode が plan → 読み取り中心
- permission mode が acceptEdits → 実装中心

## Default Actions
- Executor: HANDOFF.md から再開
- Triage: queue 全状態スキャン → TRIAGE_REPORT 生成
- Reviewer: review/ の task packet を読み REVIEW.md を書く

## Docs Repo
swe_p1_docs の場所と参照方法

## Output Rules
- PLAN は差分更新
- VERIFY は事実のみ
- RESULT は完了時のみ
- RUNLOG は「何を試し、どうなったか」のみ（「考えた」は書かない）
```

## 2-6. Subdirectory CLAUDE.md

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

会話が飛んでも作業は飛ばない。

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

# 6. ディレクトリ設計

## 6-1. メインrepo構成

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
│   │   ├── docs-output.md            # ドキュメント出力ルール
│   │   ├── role-detection.md          # 役割自動判定ルール
│   │   └── autonomous-ops.md          # 自律運転ルール
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
│   │   ├── task-handoff/
│   │   │   └── SKILL.md
│   │   ├── dashboard/
│   │   │   └── SKILL.md              # 人間向けダッシュボード生成
│   │   └── auto-advance/
│   │       └── SKILL.md              # 自動状態遷移
│   └── agents/
│       ├── triage.md
│       ├── executor.md
│       ├── reviewer.md
│       ├── log-summarizer.md
│       └── research.md
├── .ops/
│   ├── DASHBOARD.md                   # 人間向けダッシュボード
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
│   ├── config/
│   │   ├── scoring-weights.yaml       # triageスコアリング重み設定
│   │   └── parallel.yaml              # 並列実行設定
│   └── hooks/
│       ├── check-task-context.sh
│       ├── pre-tool-safety.sh
│       ├── post-tool-audit.sh
│       ├── session-end-handoff.sh
│       ├── subagent-result-sync.sh
│       └── update-dashboard.sh        # ダッシュボード更新
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

## 6-2. Worktree置き場所

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

## 6-3. User scope

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

## 6-4. parallel.yaml

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

# 7. Queue State Machine

## 7-1. 7状態

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

## 7-2. 遷移ルール（自動進行を含む）

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

### inbox → proposed
- Triage Agentが inbox を読み、構造化されたtask候補に変換
- `task.yaml` を生成し `proposed/` に配置
- スコアリングを実施

### proposed → ready
- **人間のみが実行可能**
- 今日回す本数を決めて昇格

### ready → running
- **AI自動**: 並列度上限に達していなければ自動着手
- worktree 作成
- task packet の `running/` への移動
- Claude session 起動

### running → review
- **AI自動**: review-gate の全チェック項目が pass で自動遷移
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
- **人間判断** でブロック原因が解消された場合
- **AI自動（auto-unblock）**: flaky/environment/implementation_bug の条件付き自動解除

### done → archive
- 一定期間後に `.ops/queue/done/` から削除可能
- runs/ のログは保持

## 7-3. イベント駆動の再トリアージ

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

# 8. Task Packet

## 8-1. 構成

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

## 8-2. task.yaml 詳細仕様

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

## 8-3. 各ドキュメントの詳細仕様

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

## 8-4. task packet が自己完結であるための必須情報

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

# 9. Run Log管理

## 9-1. 1 run = 1 フォルダ (task ID単位)

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

## 9-2. manifest.json

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

## 9-3. summary.md

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

# 10. Settings設計

## 10-1. `.claude/settings.json` (チーム共有)

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

## 10-2. `.claude/settings.local.json` (個人ローカル、gitignore)

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

## 10-3. `~/.claude/settings.json` (個人グローバル)

```json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
```

---

# 11. Hooks設計

Hooks はこのハーネスの **安全装置かつ自動化エンジン** 。`CLAUDE.md` は読まれるが無視されうる。Hooks は client enforced で必ず実行される。

## 11-1. UserPromptSubmit

**目的**: セッション開始時の文脈チェック + 役割判定のための文脈情報注入

```bash
#!/bin/bash
# .ops/hooks/check-task-context.sh

# 役割判定のための文脈情報を注入
CURRENT_DIR=$(pwd)
if [[ "$CURRENT_DIR" == *"_worktrees"* ]]; then
  IS_WORKTREE=true
  ROLE="executor"
  TASK_DIR=$(find .ops/queue/running/ -maxdepth 1 -mindepth 1 -type d 2>/dev/null | head -1)
  if [ -n "$TASK_DIR" ]; then
    echo "CONTEXT: role=executor, task=$(basename $TASK_DIR), handoff=$TASK_DIR/HANDOFF.md"
  fi
else
  IS_WORKTREE=false
  REVIEW_COUNT=$(ls -d .ops/queue/review/*/ 2>/dev/null | wc -l)
  RUNNING_COUNT=$(ls -d .ops/queue/running/*/ 2>/dev/null | wc -l)
  echo "CONTEXT: role=triage_or_review, review_pending=$REVIEW_COUNT, running=$RUNNING_COUNT"
fi

# worktree内にいるのにrunning taskがない場合は警告
if [ "$IS_WORKTREE" = true ] && [ -z "$TASK_DIR" ]; then
  echo "WARNING: worktree内にいますが running task がありません。先にtask packetを確認してください。"
fi

# handoff.md の存在確認
if [ "$IS_WORKTREE" = true ]; then
  if [ -n "$TASK_DIR" ] && [ ! -f "$TASK_DIR/HANDOFF.md" ]; then
    echo "WARNING: HANDOFF.md が存在しません。先に作成してください。"
  fi
fi
```

## 11-2. PreToolUse

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

## 11-3. PostToolUse

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

## 11-4. Stop

**目的**: セッション終了時のhandoff保証 + 自動再triage + ダッシュボード更新

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

# 自動再triage のトリガー
# run 終了時に failure-classifier を呼び、next_action を設定

# DASHBOARD.md を更新
bash .ops/hooks/update-dashboard.sh
```

## 11-5. SubagentStop

**目的**: subagent結果の自動反映

```bash
#!/bin/bash
# .ops/hooks/subagent-result-sync.sh

# triage結果をqueueに反映
# reviewer結果をREVIEW.mdに反映
# (具体的な実装はsubagentの出力形式に依存)
echo "SubagentStop: 結果の同期を実行"
```

## 11-6. update-dashboard.sh

**目的**: ダッシュボードの自動更新

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

# 12. Skills設計

Skills は再利用ワークフローのパッケージ。長いプロンプトを毎回手打ちしない。

## 12-1. triage-from-state

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

## 12-2. make-task-packet

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

## 12-3. failure-classifier

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

## 12-4. impact-analyzer

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

## 12-5. worktree-execution

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

## 12-6. review-gate

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

## 12-7. task-handoff

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

## 12-8. issue-sync

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

## 12-9. dashboard

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

## 12-10. auto-advance

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

---

# 13. Subagents設計

Subagentsは **main sessionを軽く保つため** に使う。常時使うのではなく、context を汚しやすい作業を切り出す。

## 13-1. triage (subagent)

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

## 13-2. executor (subagent)

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

## 13-3. reviewer (subagent)

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

## 13-4. log-summarizer (subagent)

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

## 13-5. research (subagent)

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

# 14. Permissions設計

## 14-1. Agent役割別の権限マトリクス

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

## 14-2. deny-first の原則

permissions は deny → ask → allow の順で評価される。最初にマッチしたルールが勝つ。したがって:

1. 最初に全てのdenyを定義
2. 次にaskのデフォルト
3. 最後に明示的なallow

---

# 15. rules/ 設計

## 15-1. role-detection.md

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

## 15-2. autonomous-ops.md

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

# 16. GitHub Issues / Projects 連携

## 16-1. 役割分担

| 要素 | 役割 |
|---|---|
| **GitHub Project** | 全体管理と優先順位のUI。custom fieldsでメタデータ管理 |
| **GitHub Issue** | 1 taskの外向き管理単位。議論・追跡 |
| **task packet** | Agentが読む実行ドキュメントの真実 |
| **worktree** | 実装の隔離現場 |

**Issue は管理、task packet は実行の真実** 。実行の詳細ログや handoff を Issue に全部書かない。

## 16-2. Issue にする基準

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

## 16-3. Project の custom fields

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

## 16-4. Issue → task packet の接続

```
Issue #42 "Dify compose修正"
  ↕ (双方向リンク)
Q-P1-0042-dify-compose-fix/
  ├── task.yaml (issue_id: "#42")
  └── issue.md (Issue作成用の整形文)
```

PR は Issue にリンクし、merge 時に自動 close させる。

## 16-5. Claude GitHub Actions

local terminal 運用が本命だが、以下の用途でGitHub Actionsを使う:

- Issue で `@claude` に初動整理をさせる
- PR で `@claude review` 的な運用
- CI failure 時の自動分析
- nightly review / triage

`/install-github-app` でセットアップ。必要な権限: Contents, Issues, Pull requests の read/write。

---

# 17. Triage ワークフロー

## 17-1. 定例triage (朝 or 作業開始時)

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

## 17-2. イベント駆動triage

以下のイベントで自動起動:

| イベント | 実装方法 (現在) | 実装方法 (将来) |
|---|---|---|
| run 終了 | Stop hook → triage skill 呼び出し | Agent SDK daemon |
| blocked 発生 | PostToolUse hook → failure-classifier | Agent SDK event handler |
| review 差し戻し | SubagentStop hook → triage update | Agent SDK event handler |
| inbox に仕様変更 | UserPromptSubmit hook → impact-analyzer | Agent SDK watcher |
| stale running 検出 | `/schedule` による定期チェック | Agent SDK cron |

## 17-3. スコアリング重み設定

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

# 18. 実行ワークフロー

## 18-1. 標準成功フロー

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

11. task packet を review/ に移動（自動遷移）
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

## 18-2. 失敗パターンと対応

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
5. 2回失敗 → needs_human_decision に再分類
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

# 19. タスク継続パターン

## 19-1. 同じ日で継続

- session そのまま継続
- 定期的に `/compact`
- HANDOFF.md は区切りごとに更新
- RUNLOG.md に run 単位で記録

## 19-2. 翌日継続

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

## 19-3. Session切り替えでの継続

**継続の基点は会話ではなく handoff** 。`/resume` は便利だが source of truth にはしない。

```
1. 新 session 起動
   claude --permission-mode plan

2. まず HANDOFF.md を読む

3. PLAN.md と VERIFY.md を確認

4. 前回の session がどこで終わったかは HANDOFF.md に書いてある

5. そこから続行
```

## 19-4. Mac 再起動 / tmux 切断からの復帰

致命傷にならない理由:
- worktree は残る
- queue state は残る
- runs/ のログは残る
- HANDOFF.md も残る
- task.yaml も残る

会話が飛んでも作業は飛ばない。新しいセッションが Executor として自動認識し、HANDOFF.md から再開する。

---

# 20. 仕様変更パターン

## 20-1. 仕様変更が来た時

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

## 20-2. ルール

- 走っている task を silent update しない
- 仕様変更が大きい時は task ID を分ける
- 変更理由を必ず decisions.md に残す
- 実装を続ける前に PLAN.md を更新する

---

# 21. ドキュメント管理

## 21-1. 正本ルール

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

## 21-2. 更新順序

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

## 21-3. docs repoに残すもの

- phase設計
- long-lived decisions
- architecture
- schema rationale
- acceptance criteriaの変更
- 重要な失敗学習

## 21-4. main repoに残すもの

- queue
- runs
- task packets
- issue drafts
- operational reports
- templates

---

# 22. ヘルスチェック

## 22-1. 定期チェック項目

Triage Agent がセッション開始時に確認:

- [ ] 00-docs/status.md と queue の実態が一致しているか
- [ ] blocked/ に1週間以上放置されたタスクがないか
- [ ] running/ に24時間以上更新のないタスクがないか
- [ ] review/ にレビュー未着手のタスクがないか
- [ ] HANDOFF.md のない running task がないか
- [ ] 同種の failure が3件以上蓄積していないか
- [ ] proposed/ のスコアが最新状態を反映しているか

## 22-2. 実施タイミング

- Triage session 開始時 (毎日)
- `/schedule` による定期チェック (将来は4時間ごと)
- イベント駆動の再トリアージ時

---

# 23. 安全設計

## 23-1. 重点リスク

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

## 23-2. エスカレーションルール

以下の場合、Execution Agent は即座に blocked にして人間に報告:

- **課金リスク**: 想定外のAPI呼び出し・高コスト処理
- **データ削除リスク**: 取り返しのつかないデータ削除
- **セキュリティ問題**: 秘匿情報の露出・認証情報の漏洩
- **致命的ブロッカー**: 環境が壊れて全体進行が止まる

## 23-3. コスト管理

- 全ツールをサブスクリプション範囲内で使用
- Triage / Review は plan mode (低コスト)
- Execution は task に応じてモデル選択 (sonnet が基本、重い task は opus)
- 不要なリトライでトークンを消費しない (flaky は3回まで)
- `/usage` で定期的にトークン使用量を確認

---

# 24. CLI / コマンド運用

## 24-1. 毎日使う

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

## 24-2. 問題が出た時

| コマンド | 用途 |
|---|---|
| `/debug` | Claude Code自体の不調確認 |
| `/doctor` | 環境診断 |
| `/sandbox` | sandbox状態確認 |
| `/hooks` | hook設定確認・調整 |
| `/rewind` | 会話/コードの巻き戻し |

## 24-3. GitHub連携

| コマンド | 用途 |
|---|---|
| `/install-github-app` | GitHub Actions連携セットアップ |
| `/pr-comments` | PRコメント取得 |
| `/security-review` | セキュリティレビュー |
| `gh issue create/edit/close` | Issue操作 |
| `gh pr create/comment` | PR操作 |
| `gh project item-list/item-edit` | Project操作 |

## 24-4. 自動化

| コマンド | 用途 |
|---|---|
| `/schedule` | 定期タスク (triage, health check) |
| remote triggers | イベント駆動の自動実行 |

## 24-5. 注意

- `/help` を正とする。fixed な一覧を前提にしない
- version と auth によって hidden になる slash commands がある
- `auto` permission mode は成熟するまで主役にしない

---

# 25. 日次ワークフロー

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
2. running のタスクが正常に進行しているか確認
3. blocked が発生していないか確認
4. 問題なければそのまま放置
```

## 起床後の確認（人間: 5分）

```
1. .ops/DASHBOARD.md を開く
2. 夜間の進捗を確認（done に入ったタスク、blocked に入ったタスク）
3. 必要な判断を行う
4. 次のサイクルへ
```

---

# 26. 失敗からの再開設計

## 26-1. このハーネスで一番大事なこと

**会話が飛んでも作業は飛ばない** 構造にする。

| 何が飛んでも | 残るもの | 再開方法 | 人間の介入 |
|---|---|---|---|
| Claude session | worktree, queue, runs, HANDOFF | 新 session が HANDOFF.md から自動再開 | 不要 |
| tmux session | 同上 | 同上 | 不要 |
| Mac 再起動 | 同上 | 同上 | 不要 |
| Git 操作ミス | worktree は独立 | worktree から復帰 | 不要 |
| 全体混乱 | queue structure | 全 running を blocked に、clean triage | **必要** |

## 26-2. blocked が溜まった時

```
1. Triage Agent が blocked/ を定期スキャン
2. 類似原因を束ねる
3. 束ねた原因に対して:
   - 環境系: 高優先度の infra-fix task を proposed に追加
   - 権限系: 人間に permission 変更を提案
   - 仕様系: Issue で議論を開始
4. 3件以上の同種 blocked は自動的に unblock_value +15 で proposed 生成
5. auto-unblock 条件を満たすものは自動で ready に戻す
6. それ以外は DASHBOARD.md の「今すぐ判断が必要なもの」に表示
```

## 26-3. 全体が混乱した時

```
1. 全 running を一旦 blocked に移す
2. Triage session を clean に起動
3. 00-docs/status.md を人間が更新
4. Triage Agent に再スキャンさせる
5. 最小限の ready から再開
```

---

# 27. 導入チェックリスト

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

### Step 2.5: 自律運転の追加設定
- [ ] .claude/rules/role-detection.md
- [ ] .claude/rules/autonomous-ops.md
- [ ] .claude/skills/dashboard/SKILL.md
- [ ] .claude/skills/auto-advance/SKILL.md
- [ ] .ops/config/parallel.yaml
- [ ] .ops/DASHBOARD.md（初期状態: 空テンプレート）
- [ ] .ops/hooks/update-dashboard.sh

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

# 28. 最終判断

## 採用すべきもの (全て初日から)

- root / subdir CLAUDE.md（役割判定・デフォルト行動の記述を含む）
- .claude/settings.json + settings.local.json
- subagents (triage, executor, reviewer, log-summarizer, research)
- skills (10個: 既存8個 + dashboard + auto-advance)
- hooks (5種類 + update-dashboard)
- rules (6個: 既存4個 + role-detection + autonomous-ops)
- queue state machine (7状態、自動進行ルール付き)
- task packet (10ファイル + artifacts、自己完結要件定義付き)
- 1 task = 1 worktree = 1 branch
- 6軸スコアリング
- failure-classifier（自動 retry + auto-unblock 付き）
- impact-analyzer
- イベント駆動の再トリアージ
- run終了時の自動再分類
- Issue / Projects 併用 (issue-sync skill)
- HANDOFF.md を source of truth にする
- manifest.json による構造化ログ
- deny-first permissions
- ヘルスチェック
- decisions.md による判断記録
- DASHBOARD.md による人間向けインターフェース
- parallel.yaml による並列実行制御
- 役割の自動認識（cwd + queue + permission）

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
- 人間ゲートを飛ばして proposed → ready を自動化する運用
- 並列度上限なしで無制限に task を走らせる運用
- DASHBOARD.md なしで queue フォルダを直接見させる運用（非エンジニアには厳しい）
- auto-unblock を全カテゴリに適用する運用（spec_ambiguity の自動解消は危険）
- 失敗時に retry 上限なしで無限リトライする運用

---

この設計の骨格 (queue / task packet / worktree / handoff / failure-classifier / scoring) は **どのAIエージェントプラットフォームでも成立する** 。Claude Code 固有部分 (skills / hooks / subagents / permissions / rules) は、将来ツールが変わっても「同等の概念」が存在する可能性が極めて高く、完全な書き直しにはならない。

骨格は一切壊さず、「人間がプロンプトを書かなくても構造が役割と行動を決定する」層を追加した。Claude Code 固有の構造的決定メカニズム（CLAUDE.md / rules / hooks / skills）を活用して自律性を実現している。

今の Claude Code 運用にも耐え、モデル性能向上・Agent Team・永続エージェント・auto mode の成熟にもそのまま追従できる設計である。
