了解です。
ここでは **Claude だけ** を前提に、あなたがあとでそのまま運用設計に落とせるように、かなり具体的に書きます。
結論から言うと、Claude 前提の最適解は **「AI主導トリアージ + GitHub Projects/Issues による全体管理 + repo内の task pack を実行の真実 + worktree で隔離実装 + hooks/permissions/sandbox で事故防止」** です。Claude Code 自体は、コードベースを読み、編集し、コマンドを実行し、複数ファイルにまたがる開発作業を進められる agentic coding tool で、セッションはディレクトリ単位で独立しており、並列セッションは worktree で運用するのが自然です。 ([Claude API Docs][1])

---

# 0. まずこのハーネスの思想

この設計で一番大事なのは、**役割を会話に持たせない** ことです。
つまり「このセッションは第2層です」と毎回宣言して始めるのではなく、**どの task pack を読んでいるか、どの worktree にいるか、どの permissions mode で走っているか** で、そのセッションの役割を決めます。Claude Code は新しいセッションごとに fresh context で始まり、前の会話履歴は引き継がれません。代わりに `CLAUDE.md` と auto memory が効き、設定は client 側で強制されます。Anthropic 自身も、`CLAUDE.md` は行動指針であり、**強制力があるのは settings / permissions / sandbox** 側だと明言しています。 ([Claude][2])

だから、このハーネスは次の四層で考えます。

1. **GitHub Projects / Issues**
   全体の見える化と優先順位管理。
2. **repo内 task pack**
   実行の真実。Claude が読むべきドキュメント束。
3. **git worktree**
   実際の実装を隔離する現場。
4. **Claude の settings / permissions / hooks / skills / subagents**
   行動の枠と再利用ワークフロー。 ([GitHub Docs][3])

---

# 1. 全体アーキテクチャ

Claude だけで回す場合の全体像はこうです。

```text
GitHub Projects / Issues
    ↓
AI Triage Session (Claude)
    ↓
triage candidates
    ↓
selected tasks -> ready
    ↓
git worktree 作成
    ↓
executor session (Claude)
    ↓
review session (Claude)
    ↓
Issue / Project 更新
    ↓
done / archive
```

つまり、Claude は1種類ではなく、**同じ Claude Code を役割別セッションとして使い分ける** だけです。subagents は必要な時だけ使いますが、基本は「Triage」「Executor」「Reviewer」の3パターンに分ければ十分です。subagents は専用 system prompt、専用 tool access、独立 permissions、独立 context window を持つので、探索やレビューのような高ボリューム作業を main conversation から切り離すのに向いています。 ([Claude API Docs][4])

---

# 2. 推奨ディレクトリ構成

Claude 専用で組むなら、repo 側は次のようにします。

```text
repo-root/
├── CLAUDE.md
├── .claude/
│   ├── settings.json
│   ├── settings.local.json      # gitignore
│   ├── rules/
│   │   ├── repo-core.md
│   │   ├── testing.md
│   │   ├── github-ops.md
│   │   └── docs-output.md
│   ├── skills/
│   │   ├── triage-report/
│   │   │   └── SKILL.md
│   │   ├── task-start/
│   │   │   └── SKILL.md
│   │   ├── task-handoff/
│   │   │   └── SKILL.md
│   │   ├── task-review/
│   │   │   └── SKILL.md
│   │   └── issue-sync/
│   │       └── SKILL.md
│   └── agents/
│       ├── triage.md
│       ├── reviewer.md
│       └── research.md
├── 80-tasks/
│   ├── triage/
│   ├── inbox/
│   ├── ready/
│   ├── running/
│   ├── review/
│   ├── done/
│   └── archive/
├── 85-worktrees/
└── .github/
    └── workflows/   # 必要なら Claude GitHub Actions
```

この構成の理由は、Claude Code の設定スコープが **Managed / User / Project / Local** に分かれていて、repo 共有情報は `.claude/` に置くのが正しいからです。`CLAUDE.md` は毎セッション読み込まれる project memory 用で、skills は `.claude/skills/*/SKILL.md`、subagents は Markdown frontmatter 付きで定義できます。custom commands も skills に統合済みです。 ([Claude][5])

---

# 3. スコープ設計

## 3-1. Managed scope

Managed は企業配布や IT 管理向けで、あなたの現状では通常不要です。
ただし思想としては重要で、**危険操作を本当に止めたいなら settings 側で止める** のが正解です。Anthropic は settings rules は client enforced で、`CLAUDE.md` は hard enforcement ではないと明記しています。 ([Claude API Docs][6])

## 3-2. User scope

`~/.claude/` です。
ここには**個人共通**の設定だけ置きます。

置くもの:

* 個人の一般的な作法
* 汎用 skill
* 汎用 subagent
* 個人用 keybindings
* 全repo共通の `gh` CLI 利用ルール

置かないもの:

* P1/P2/P3 固有ルール
* repo 固有の完了条件
* task 固有の handoff

Claude の設定は `/config` から触れますが、User scope はあなた個人だけに効きます。 ([Claude][5])

## 3-3. Project scope

`.claude/` と `CLAUDE.md` です。
ここが**このハーネスの本体**です。

Project scope に置くべきもの:

* repo 全体の憲法
* テスト方針
* ドキュメント出力ルール
* GitHub 運用ルール
* task pack の標準形式
* hooks
* shared skill
* shared subagents

これは git 管理する前提です。Claude の docs でも Project scope は repository collaborators に共有される層として定義されています。 ([Claude][5])

## 3-4. Local scope

`.claude/settings.local.json` です。
これは gitignore して、あなたのローカル例外だけを入れます。

入れるもの:

* `claudeMdExcludes`
* あなたの実験用 permission 例外
* 一時的な sandbox allow path
* 一時的な model override

Claude 公式でも `claudeMdExcludes` は local に置く使い方を案内しています。 monorepo や複数プロジェクトで不要な親 `CLAUDE.md` を切るのに便利です。 ([Claude API Docs][6])

## 3-5. Task scope

これは Claude のネイティブ scope ではないですが、このハーネスで最重要です。
`80-tasks/task-xxx/` の束がそれです。

Claude はセッションを fresh に始めるので、**「今この task をどう扱うか」は task scope のファイル群が担う** のが安定します。 ([Claude][2])

---

# 4. CLAUDE.md に書くべき内容

`CLAUDE.md` は長くしすぎないほうが良いです。
ここには **憲法だけ** を書き、反復手順は skills に逃がします。

書くべき内容は次です。

```md
# Project Constitution

## Mission
この repo の目的

## Non-negotiables
- main で直接実装しない
- 1 task = 1 worktree
- task 開始前に task pack を読む
- 実装終了時に HANDOFF.md と VERIFY.md を更新
- 破壊的変更・依存追加・本番操作は止まって確認

## Required reads
- 80-tasks/... の BRIEF.md
- PLAN.md
- HANDOFF.md

## Output rules
- PLAN は差分更新
- VERIFY は事実のみ
- RESULT は完了時だけ更新
- RUNLOG は要点のみ

## GitHub rules
- 採用タスクのみ issue 化
- Issue は管理、task pack は実行の真実
```

理由は、Claude Code では `CLAUDE.md` は persistent instructions として毎セッション読まれますが、deterministic に毎回強制したいことは hooks に寄せるべきだからです。Anthropic の best practices でも、**例外ゼロでやらせたいものは hooks**、再利用ワークフローは **skills** に置くべきだと案内しています。 ([Claude API Docs][6])

---

# 5. .claude/settings.json の基本方針

ここが、実際に「Claude をどう動かすか」を決める主設定です。
まずはこの思想で固定するのが良いです。

* 通常は `plan` を初期モード
* 実装時だけ `acceptEdits`
* 危険なパスは deny
* sandbox は有効
* hooks は project 側に置く

Claude の permission mode には `default`, `acceptEdits`, `plan`, `auto`, `dontAsk` があり、`plan` は read-only planning、`acceptEdits` は編集許可をセッション中自動承認、`auto` は background safety checks 付きです。 `auto` は研究プレビューで要件もあるので、今のあなたの本番運用では主役にしないほうが安全です。 ([Claude][7])

おすすめの project 設定イメージはこうです。

```json
{
  "permissions": {
    "defaultMode": "plan"
  },
  "model": "sonnet",
  "sandbox": {
    "enabled": true
  }
}
```

`/config` から調整できますし、`model` は `/model`、`--model`、環境変数、settings の順に設定できます。 ([Claude][5])

---

# 6. permissions と sandbox の具体設計

ここは **prompt より重要** です。

Claude Code の permissions は tiered です。
Read-only は approval 不要、Bash は approval 必要、File modification は approval 必要です。ルールは **deny → ask → allow** の順で評価され、最初にマッチしたルールが勝ちます。 `/permissions` で確認・更新できます。 ([Claude][7])

Claude Code の sandbox は OS-level で filesystem / network isolation をかけます。デフォルトでは current working directory 以下に read/write が許され、外は書けません。必要なら `sandbox.filesystem.allowWrite` で追加パスを許可できます。network も制御でき、サンドボックスの目的は approval fatigue を減らしつつ境界内で Claude を自由に動かすことです。 ([Claude][8])

## 実運用の推奨

* **Triage session**
  `plan` mode、編集なし、Bash も最小
* **Executor session**
  `acceptEdits` か `default`。ただし worktree 内のみ
* **Review session**
  `plan` mode または `dontAsk` + read系 allow のみ
* **危険作業**
  deny rules で完全に止める

### deny すべきもの

* `rm -rf`
* `git push --force`
* 本番 secrets 置き場
* マイグレーション本番ディレクトリ
* repo外の危険パス

### allow しやすいもの

* `npm test`
* `pnpm test`
* `pytest`
* `ruff`
* `eslint`
* `tsc`

この設計にすると、Claude の賢さに依存せず client 側で止まります。 ([Claude][7])

---

# 7. hooks 設計

hooks はこのハーネスの安全装置です。
Anthropic は hooks を「tool 実行前後や compact 前後などの lifecycle で動く shell / HTTP / prompt hook」と定義していて、best practices でも「毎回必ずやらせたいことは hooks」としています。 ([Claude API Docs][9])

## 7-1. 入れるべき hooks

### PreToolUse

目的:

* 危険コマンドの遮断
* `running` task がないのに編集しようとしたら停止
* task pack 未読で実装しようとしたら停止

### PermissionRequest

目的:

* Bash 許可要求に説明をつける
* 本番っぽいネットワークアクセスをブロック

### PostToolUse

目的:

* 編集後に format/lint/typecheck
* `PLAN.md` と無関係な大規模変更の検知
* 変更ファイル一覧を `RUNLOG.md` に要約

### PreCompact / PostCompact

目的:

* compact 前に handoff 候補を書き出す
* compact 後に再開用要約を保存

### Stop / SessionEnd

目的:

* `HANDOFF.md` と `VERIFY.md` 未更新なら警告
* 実行まとめを `RUNLOG.md` に反映

Claude の hook lifecycle には SessionStart, PreToolUse, PermissionRequest, PostToolUse, SubagentStart/Stop, TaskCreated/Completed, PreCompact/PostCompact, SessionEnd などがあり、かなり細かく差し込めます。 ([Claude API Docs][9])

---

# 8. skills 設計

skills は、長いプロンプトを毎回手打ちしないための再利用レイヤーです。
Claude Code では `SKILL.md` を作ると toolkit に追加され、関連時に自動利用されるか、`/skill-name` で明示呼び出しできます。custom commands は skills に統合済みです。Claude Code の skills は Agent Skills open standard ベースです。 ([Claude API Docs][10])

このハーネスでは、最低でも次の skills を作るべきです。

## 8-1. `triage-report`

目的:

* GitHub Project / Issues / 最近の変更 / task pack 状況を読み
* 次候補を優先順位順で出す
* `TRIAGE_REPORT.md` と `CANDIDATES.yaml` を生成

## 8-2. `task-start`

目的:

* `BRIEF.md`, `PLAN.md`, `HANDOFF.md` を読む
* 未整備なら足りないファイルを作る
* 実装前の plan 更新を強制

## 8-3. `task-handoff`

目的:

* 現在地
* 次の1〜3手
* 保留事項
* テスト状況
* issue / project 更新要否
  をまとめる

## 8-4. `task-review`

目的:

* 変更差分をチェック
* リスクを列挙
* merge readiness を判定
* `REVIEW.md` を書く

## 8-5. `issue-sync`

目的:

* task pack の状態を GitHub issue / project に反映
* labels / fields / comments を更新

skills は `.claude/skills/` に置くので、repo 内でチーム共有もできます。 ([Claude API Docs][10])

---

# 9. subagents 設計

subagents は「いつも使う」ものではなく、**main session を軽く保ちたい時だけ使う** のが正解です。
Claude は subagents を自動委譲もでき、各 subagent は独立 context、独立 permissions、独立 tool access を持ちます。 ([Claude API Docs][4])

このハーネスで作るなら3つで十分です。

## 9-1. `triage`

用途:

* 状況整理
* 候補作成
* スコアリング

特徴:

* Read, Grep, LS, `gh` CLI くらい
* 書き込み権限は task pack に限定

## 9-2. `research`

用途:

* ログ探索
* 既存コードの構造把握
* issue / PR コメント読み取り

特徴:

* Read-only に近い
* 高ボリューム探索専用

## 9-3. `reviewer`

用途:

* 差分レビュー
* セキュリティ / 保守性 / テスト漏れの確認

特徴:

* 書き込みは `REVIEW.md` のみでもよい

subagents を main の代わりに常に使うのではなく、「探索」「レビュー」など context を汚しやすい仕事だけ切り出すのが良いです。Anthropic の common workflows でも specialized subagents はそういう使い方を勧めています。 ([Claude API Docs][4])

---

# 10. GitHub Projects / Issues の位置づけ

Claude 限定でも、**Issues は使ったほうがいい** です。
ただし、Issue を「実行の真実」にしないのがコツです。

GitHub Projects は table / board / roadmap で見られ、custom fields を付けられ、built-in automation、API、GitHub Actions で自動化できます。Issues は task / bug / idea / feedback を追跡するための基本単位です。Claude 側も best practices で GitHub を使うなら `gh` CLI を入れることを勧めていて、Claude は `gh` を使って issue 作成、PR 作成、コメント読取をこなせます。 ([GitHub Docs][3])

## 推奨する役割分担

* **GitHub Project** = 全体管理と優先順位 UI
* **GitHub Issue** = 1 task の外向き管理単位
* **task pack** = Claude が読む実行ドキュメントの真実
* **worktree** = 実装の隔離現場

つまり、Issue は必要です。
ただし、**実行の詳細ログや handoff を Issue に全部書かない** ほうがよいです。

---

# 11. GitHub Project の推奨フィールド

Project には最低でも次を持たせます。

* `Project` : P1 / P2 / P3
* `Status` : triage / ready / running / review / done / blocked
* `Priority` : High / Medium / Low
* `Effort` : XS / S / M / L
* `AI Suggested` : yes / no
* `Selected This Cycle` : yes / no
* `Spec Changed` : yes / no
* `Branch`
* `Worktree`
* `Task Pack Path`

GitHub Projects は custom fields をかなり自由につけられるので、この構成で十分管理できます。 built-in automation で item 自動追加や auto archive もできます。 ([GitHub Docs][3])

---

# 12. task pack の標準形

1 task = 1 folder です。
たとえば `80-tasks/running/task-042-auth-refactor/` に次を置きます。

```text
task-042-auth-refactor/
├── BRIEF.md
├── PLAN.md
├── HANDOFF.md
├── VERIFY.md
├── RESULT.md
├── REVIEW.md
├── DECISIONS.md
├── RUNLOG.md
├── RISKS.md
└── artifacts/
    ├── screenshots/
    ├── test-outputs/
    ├── raw/
    └── diffs/
```

Claude は session を fresh に始めるので、**次回の再開点を必ずファイルに残す** 必要があります。`HANDOFF.md` が最重要です。Claude は `/compact` で会話圧縮できますが、外部に handoff を残していないと session 依存になります。 built-in commands として `/compact`, `/rename`, `/resume`, `/rewind`, `/tasks`, `/usage`, `/permissions`, `/model`, `/memory`, `/hooks`, `/skills` などが使えます。 ([Claude][2])

---

# 13. 各ドキュメントの役割

## `BRIEF.md`

この task が何かを短く書く。
背景、目的、完了条件、関連 issue。

## `PLAN.md`

今の計画。
初回だけでなく、途中で更新する。
仕様変更が入ったら最初にここを変える。

## `HANDOFF.md`

再開点。
最低でも

* 現在地
* 次の1〜3手
* 保留事項
* 要承認事項
* 現在の branch/worktree
  を書く。

## `VERIFY.md`

事実だけを書く。
例: lint pass, unit test fail, manual not run。

## `RESULT.md`

完了時の成果要約。
途中では書き込まない。

## `REVIEW.md`

review session が書く。

## `DECISIONS.md`

決まったことだけ。議論の全文は書かない。

## `RUNLOG.md`

構造化ログ。
「考えた」ではなく、「何を試し、どうなったか」。

## `RISKS.md`

未解決リスク、既知制約。

この分離をする理由は、Claude の会話ログと task の監査証跡を分離するためです。Claude 自体も sessions をローカル保存して rewind / resume / fork できますが、運用上の真実は repo 側に出したほうが強いです。 ([Claude][2])

---

# 14. worktree 運用

Claude Code は directory-bound session なので、parallel session は worktree で回すのが正しいです。Anthropic も docs で、parallel Claude sessions は git worktrees による separate directories で走らせる形を案内しています。 ([Claude][2])

## ルール

* main で実装しない
* 1 task = 1 branch = 1 worktree
* review もできれば別 session
* running task 数だけ worktree を持つ

## 命名

* issue: `#42`
* branch: `task/42-auth-refactor`
* worktree: `85-worktrees/task-42-auth-refactor`
* task pack: `80-tasks/running/task-42-auth-refactor`

## 作成

```bash
git switch main
git pull
git worktree add 85-worktrees/task-42-auth-refactor -b task/42-auth-refactor
```

## Claude 起動

```bash
cd 85-worktrees/task-42-auth-refactor
claude --permission-mode plan --model sonnet
```

Triage や Planning はまず plan で開始し、実装許可後に `acceptEdits` に切り替えるのが安全です。 `--model`, `--permission-mode` は startup で使え、モデルは `/model` でも変更できます。 ([Claude][11])

---

# 15. CLI オプションと /コマンドの実務採用方針

ここでは「使うもの」と「今は使わないもの」を分けます。

## 毎日使う

* `claude --permission-mode plan`
* `claude --model sonnet`
* `/plan`
* `/permissions`
* `/status`
* `/model`
* `/rename`
* `/resume`
* `/compact`
* `/tasks`
* `/hooks`
* `/skills`
* `/memory`

`/plan` は plan mode に即入れます。 `/permissions` はルール確認、`/status` は version/model/account/connectivity、`/resume` は再開、`/compact` は会話圧縮、`/tasks` は background tasks、`/rename` は session 名管理です。 ([Claude][12])

## GitHub 連携で使う

* `/install-github-app`
* `/pr-comments`
* `gh` CLI
* 必要なら Claude Code GitHub Actions

`/install-github-app` は GitHub Actions 連携セットアップ用です。Claude GitHub Actions は `@claude` mention で issue/PR を分析し、PR 作成、feature 実装、bug fix までできます。GitHub App には Contents, Issues, Pull requests の read/write 権限が必要です。 ([Claude][12])

## 必要になってから使う

* `/remote-control`
* `/schedule`
* `/sandbox`
* `/rewind`
* `/insights`
* `/security-review`

`/schedule` は cloud scheduled tasks 用、`/rewind` は会話やコードを前のポイントに戻す、`/security-review` は current branch の diff を見てセキュリティレビューします。 ([Claude][12])

## 今は主役にしない

* `auto` permission mode
* `bypassPermissions`

`auto` は background classifier 付きの研究プレビューで、要件もあります。あなたの長時間運用では魅力的ですが、まずは `plan` と `acceptEdits` で枠組みを固めるのが無難です。 ([Claude][13])

---

# 16. AI主導トリアージの具体ワークフロー

ここからが今回の本丸です。
あなたの希望どおり、**タスクの始まりは AI 主導** にします。
ただし、「実行本数を決める」のはあなたです。

## 16-1. Triage セッションの開始

頻度は2種類です。

* **定例 triage**
  朝 or 夜に1回
* **イベント triage**
  仕様変更、失敗、PR 追加、issue 追加時

Claude 起動例:

```bash
claude --permission-mode plan --model sonnet
```

最初の指示:

```text
この repo の次の候補タスクを優先順位順に提案して。
以下を必ず読む:
- CLAUDE.md
- 80-tasks/triage の直近レポート
- 80-tasks/running
- 80-tasks/review
- 80-tasks/done の直近結果
- GitHub Project の現在状態
- Open issue と最近の PR コメント

出力:
- TRIAGE_REPORT.md
- CANDIDATES.yaml
```

Claude は gh CLI を使うのが最も効率的です。Anthropic も best practices で GitHub を使うなら `gh` CLI を入れることを勧めています。 ([Claude][14])

## 16-2. Claude が見る入力

* GitHub Project の項目
* open issue
* review待ち task
* running task
* recent commits
* 直近の失敗ログ
* `DECISIONS.md`
* `RISKS.md`
* 仕様変更メモ

## 16-3. Claude が出す出力

`TRIAGE_REPORT.md`:

* Top 10候補
* 優先順位
* 理由
* 新規 task か継続 task か
* 依存ブロッカー
* 想定作業量
* 今回おすすめ本数

`CANDIDATES.yaml`:

* task_id
* title
* type
* priority_score
* impact
* urgency
* effort
* confidence
* dependency
* source_issue
* source_docs

---

# 17. AI主導トリアージの評価基準

Claude に優先順位を自由文だけで出させるとブレるので、score を持たせます。

おすすめは次の6軸です。

* `impact`
* `urgency`
* `unblock_value`
* `effort`
* `confidence`
* `cross_project_leverage`

例:

```yaml
- task_id: task-042-auth-refactor
  priority_score: 91
  impact: 30
  urgency: 20
  unblock_value: 18
  effort: 8
  confidence: 10
  cross_project_leverage: 5
```

この score は GitHub Project の custom fields と相性が良いです。 Project には priority や effort のような structured metadata を持たせられます。 ([GitHub Docs][3])

---

# 18. Issue 作成ワークフロー

## 原則

* **全部を Issue にしない**
* **採用した task だけ Issue にする**
* local triage 候補の段階では draft 的に扱う

こうすると Issue がノイズだらけになりません。

## 流れ

1. Claude が triage 候補を出す
2. あなたが今回は何本動かすか決める
3. 選ばれたものだけ `gh issue create` で作成
4. Claude が Project に追加
5. task pack を `ready/` に生成

Claude は `gh` を理解できるので、`gh issue create`, `gh issue edit`, `gh pr create` を運用に組み込めます。 ([Claude][14])

---

# 19. 実装ワークフロー: 成功パターン

## 19-1. ready → running

1. Issue を作成
2. task pack を `ready/` に作る
3. worktree を切る
4. `running/` に移す
5. Claude を `plan` で起動
6. `PLAN.md` 更新
7. 実装許可後に `acceptEdits`
8. 変更
9. テスト
10. `VERIFY.md` 更新
11. `HANDOFF.md` 更新
12. `review/` へ

## 19-2. Review

1. 別 Claude session を `plan` で起動
2. `REVIEW.md` を書く
3. 問題なければ merge
4. Issue / Project を更新
5. `done/` に移す
6. worktree cleanup

Claude Code はセッションの rewind / resume / fork ができ、ファイル snapshot も持つので、review と rollback もやりやすいです。 ([Claude][2])

---

# 20. 実装ワークフロー: 失敗パターン

## 20-1. テスト失敗

* `VERIFY.md` に fail を記録
* `HANDOFF.md` に現状と次の1〜3手を書く
* Issue に `blocked` comment
* Project の `Status=blocked`
* task は `running/` のままか `review/blocked` 相当に置く

Claude の common workflows でも「run tests and fix failures」は基本ユースケースです。まずは Claude に復旧を試させ、無理なら blocked にするのがよいです。 ([Claude API Docs][15])

## 20-2. Claude が迷走

兆候:

* plan と違う大規模変更
* unrelated files への編集
* 同じ失敗ループ
* context 膨張

対処:

* `/compact`
* `/rewind`
* `HANDOFF.md` 更新
* いったん session 終了
* 新 session を `plan` で再開

Claude には `/compact`, `/rewind`, `/resume`, `/rename` があるので、会話を無理に引っ張らず切り直せます。 ([Claude][12])

## 20-3. 権限拒否で止まる

* `permissions` を見直す
* deny を緩めるのではなく、まず task を split
* どうしても必要ならその session だけ allow

---

# 21. タスク継続パターン

Claude は session を resume できますが、**permissions は session-scoped なので再承認が必要** です。
つまり、「会話だけ resume すればよい」ではなく、毎回 `HANDOFF.md` を読むのが必要です。 Anthropic は resume で conversation history は戻るが、session-scoped permissions は戻らないと明記しています。 ([Claude][2])

## 再開手順

1. `claude --resume` か `/resume`
2. `HANDOFF.md` を読み直す
3. `PLAN.md` を再確認
4. `VERIFY.md` の最新状態を確認
5. 必要なら permissions 再承認
6. 続行

このパターンを前提にしておくと、tmux 切断や session clear に強くなります。

---

# 22. 仕様変更パターン

これはかなり重要です。
仕様変更時は、**実装を続ける前に plan を更新する** のがルールです。

## 流れ

1. Issue に仕様変更コメントが入る
2. Claude triage session が検出
3. 既存 task 継続か、新規 task 分離かを判断
4. `PLAN.md` を更新
5. `DECISIONS.md` に変更理由
6. Project の `Spec Changed=yes`
7. 必要なら worktree を分岐し直す

### 継続でよいケース

* 変更が小さい
* 既存変更と整合する
* 既存 issue の範囲内

### 分離すべきケース

* 別 feature になった
* scope が大きく増えた
* レビュー観点を分けたい
* rollback 単位を分けたい

この判断を Claude にやらせるには、triage skill に「継続 / 分離判定」を含めます。

---

# 23. 追加機能パターン

追加機能は、今走っている task に無理にねじ込まないほうが良いです。
基本はこうです。

* 今の task の受け入れ基準を変えるか？
* それとも新 issue / 新 task か？

## 原則

* 受け入れ基準が変わるだけなら既存 task 更新
* 別のレビュー単位になるなら新 task
* 既存 worktree に混ぜると危険なら新 worktree

worktree と issue を task 単位に保つのは、この判断をきれいにするためです。 ([Claude][2])

---

# 24. レビューコメント対応パターン

レビューコメントが来たら、3通りに分けます。

1. **今の task の修正**
   そのまま同じ worktree で対応
2. **新 issue 化すべき指摘**
   新 task
3. **仕様論争**
   実装停止、`DECISIONS.md` 更新、triage へ戻す

Claude は `/pr-comments` で PR コメントを引けますし、GitHub Actions を入れておけば `@claude` mention で comment-driven な対応も可能です。 ([Claude][12])

---

# 25. 緊急バグ対応パターン

緊急 bugfix は例外運用に見えますが、構造は同じです。

1. Issue 作成
2. Project で Priority=Critical
3. `ready/` を飛ばして `running/` でもよい
4. 専用 worktree
5. Claude `plan` で root cause 仮説整理
6. `acceptEdits`
7. 最小修正
8. `VERIFY.md`
9. review
10. merge
11. 後続の cleanup issue 作成

重要なのは、**緊急でも main 直叩きしない** ことです。

---

# 26. Claude GitHub Actions を入れるかどうか

Claude 限定運用なら、あとでかなり効いてきます。
Claude Code GitHub Actions は issue / PR で `@claude` とメンションするだけで、コード分析、PR作成、feature 実装、bug fix が可能です。Quick setup は Claude terminal から `/install-github-app` で進められます。必要な権限は Contents, Issues, Pull requests の read/write です。 ([Claude][16])

ただし、今のフェーズでは **本命は local terminal 運用** です。
GitHub Actions は次の用途から入れるのがよいです。

* issue コメントから定型 triage
* PR comment への自動返答
* auto-fix PR
* nightly review

Claude on the web では PR を監視して CI failures や review comments に自動対応する auto-fix もあります。 ([Claude][17])

---

# 27. ログ管理の原則

ここはかなり重要です。
**生ログを全部 git に入れない** のが原則です。
Claude の内部 session はローカルに保存され、resume / rewind 用に使えますが、repo 側には**構造化された運用ログだけ**残します。 ([Claude][2])

## git 管理するもの

* `PLAN.md`
* `HANDOFF.md`
* `VERIFY.md`
* `RESULT.md`
* `REVIEW.md`
* `DECISIONS.md`
* `RUNLOG.md`
* `RISKS.md`

## artifact として保管するもの

* screenshots
* test outputs
* diffs
* raw error logs

## gitignore してよいもの

* 巨大な一時ログ
* tmp export
* Claude の個人 local config
* transient screenshots

---

# 28. RUNLOG と VERIFY の書き方

## RUNLOG

悪い例:

* 13:00 考えた
* 13:05 また考えた

良い例:

* root cause candidate: auth token refresh path mismatch
* attempted fix: updated AuthService retry logic
* result: unit tests pass, integration test fail
* next action: inspect mock server token expiry handling

## VERIFY

良い例:

```md
- lint: pass
- typecheck: pass
- unit tests: pass
- integration tests: fail (token-expiry case)
- manual check: not run
- remaining risk: refresh flow not verified end-to-end
```

Claude の common workflows は tests を run し、失敗を fix する流れを標準ユースケースにしています。だから verify は必ず事実ベースにします。 ([Claude API Docs][15])

---

# 29. 1日の標準運用

## 朝または作業前

1. Claude triage session
2. `TRIAGE_REPORT.md` 更新
3. あなたが今日回す本数を決める
4. 選んだ候補だけ issue 化 / ready 化

## 実装中

1. task ごとに worktree
2. Claude executor
3. 必要に応じて subagent
4. `HANDOFF.md` 更新

## 終了前

1. reviewer session
2. issue / project 更新
3. 次の候補を inbox か triage へ戻す

---

# 30. 成功しやすい運用上のルール

最後に、Claude 限定運用で成功率を上げるルールを絞ります。

* **main で Claude 実装しない**
* **1 task 1 issue 1 worktree 1 handoff**
* **タスク開始前に必ず plan**
* **毎回 role 宣言しない**
* **強制したいことは hooks**
* **再利用したいことは skills**
* **探索やレビューだけ subagent**
* **Issue は管理、task pack は実行の真実**
* **本数は AI が決めず、あなたが決める**
* **仕様変更時は実装前に PLAN を変える**
* **失敗時は無理に続けず compact / rewind / resume**
* **GitHub を使うなら gh CLI を Claude に使わせる**

この設計は、Claude Code の core 機能である session independence、CLAUDE.md memory、permissions modes、sandboxing、hooks、skills、subagents、GitHub Actions と噛み合っています。だから、あとで Codex などに展開する前の「最初の本命ハーネス」としてかなり強いです。 ([Claude][2])

[1]: https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview "Claude Code overview - Claude Code Docs"
[2]: https://code.claude.com/docs/en/how-claude-code-works "How Claude Code works - Claude Code Docs"
[3]: https://docs.github.com/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects "About Projects - GitHub Docs"
[4]: https://docs.anthropic.com/en/docs/claude-code/sub-agents "Create custom subagents - Claude Code Docs"
[5]: https://code.claude.com/docs/en/settings "Claude Code settings - Claude Code Docs"
[6]: https://docs.anthropic.com/en/docs/claude-code/memory "How Claude remembers your project - Claude Code Docs"
[7]: https://code.claude.com/docs/en/permissions "Configure permissions - Claude Code Docs"
[8]: https://code.claude.com/docs/en/sandboxing "Sandboxing - Claude Code Docs"
[9]: https://docs.anthropic.com/en/docs/claude-code/hooks "Hooks reference - Claude Code Docs"
[10]: https://docs.anthropic.com/en/docs/claude-code/skills "Extend Claude with skills - Claude Code Docs"
[11]: https://code.claude.com/docs/en/model-config "Model configuration - Claude Code Docs"
[12]: https://code.claude.com/docs/en/commands "Built-in commands - Claude Code Docs"
[13]: https://code.claude.com/docs/en/permission-modes "Choose a permission mode - Claude Code Docs"
[14]: https://code.claude.com/docs/en/best-practices "Best Practices for Claude Code - Claude Code Docs"
[15]: https://docs.anthropic.com/en/docs/claude-code/common-workflows "Common workflows - Claude Code Docs"
[16]: https://code.claude.com/docs/en/github-actions "Claude Code GitHub Actions - Claude Code Docs"
[17]: https://code.claude.com/docs/en/claude-code-on-the-web?utm_source=chatgpt.com "Claude Code on the web"
