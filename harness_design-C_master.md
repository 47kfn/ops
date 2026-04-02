---

# 0. この設計の思想

このハーネスでは、Claude を3種類の役に分けます。

* **Triage Claude**
  計画書、repo 状態、進捗、失敗ログ、仕様変更を見て、次タスクを提案・優先順位付けする
* **Execution Claude**
  `ready` にある task を 1 task = 1 worktree で実行する
* **Review Claude**
  実行結果を読み、差し戻し・完了判定の材料を作る

重要なのは、Claude Code の各セッションは毎回 fresh context で始まり、`CLAUDE.md` と auto memory は読み込まれるものの、どちらも「強制設定」ではなく context 扱いだという点です。だから、役割を毎回プロンプトで思い出させるより、**ファイル配置・権限・queue 状態・worktree 境界** で役割を決めるほうが安定します。 ([Claude API Docs][1])

---

# 1. 全体アーキテクチャ

## 1-1. 目指す形

全体は次の4層です。

1. **長期計画層**
   `swe_p1_docs` にある計画書・フェーズ設計・仕様・判断メモ
2. **運用管理層**
   `SWE_P1/.ops/` にある queue、runs、handoff、reports
3. **実行層**
   各 task ごとの git worktree と Claude セッション
4. **可視化層**
   GitHub Issues / Projects / PR

このうち **Claude が直接回す中心** は 2 と 3 です。
1 は読みに行く。
4 は追跡と意思決定に使う。

---

# 2. Claude-only ハーネスの完成形

## 2-1. 結論

この設計では、**タスクの始まりは AI 主導** にできます。
ただし「AI が勝手に全部実行開始」ではなく、**AI が次タスク候補を作る → 人間が今日回す本数だけ決める → 実行 Claude が回す** という流れにします。

つまり、あなたの役割はこうなります。

* `proposed` を見る
* 今日何本回すか決める
* `ready` に昇格させる
* `review` を見て merge / close / hold を決める

それ以外はかなり Claude に寄せられます。

---

# 3. ディレクトリ設計

メイン repo 側に、Claude ハーネス用の運用ディレクトリを追加します。

```text
SWE_P1/
├── .ops/
│   ├── queue/
│   │   ├── inbox/
│   │   ├── proposed/
│   │   ├── ready/
│   │   ├── running/
│   │   ├── review/
│   │   ├── blocked/
│   │   └── done/
│   ├── runs/
│   ├── reports/
│   ├── templates/
│   ├── issues/
│   │   ├── drafts/
│   │   └── synced/
│   └── worktrees/
├── .claude/
│   ├── CLAUDE.md
│   ├── settings.json
│   ├── settings.local.json     # gitignore
│   ├── agents/
│   ├── skills/
│   └── rules/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/
├── 00-docs/
├── 10-apps/
├── 20-configs/
├── 30-scripts/
└── 90-git/
```

この形にしておくと、メイン repo 側で「運用に必要な可変データ」を管理しつつ、長期設計は `swe_p1_docs` 側に残せます。今の repo 分離方針とも矛盾しません。 

---

# 4. Claude のスコープ設計

Claude Code では、持続的な指示は `CLAUDE.md`、設定は settings、再利用手順は skills、役割分担は subagents、イベント自動化は hooks で構成するのが基本です。`/init` は `CLAUDE.md`、skills、hooks の初期セットアップを対話的に提案できます。 ([Claude API Docs][1])

## 4-1. スコープの優先順位

Claude 側のスコープは、実務上こう考えるのが分かりやすいです。

### A. Managed scope

* 組織全体の制約
* 会社ルール、セキュリティ、禁止事項
* ローカルで上書きされない

Anthropic は managed permissions や managed policies をローカル設定で上書きできない形で配布できると案内しています。 ([Claude API Docs][2])

### B. User scope

* `~/.claude/CLAUDE.md`
* `~/.claude/settings.json`
* `~/.claude.json`

個人の普遍的な好みを書く場所です。Claude Code は `~/.claude/settings.json` をユーザー設定、`~/.claude.json` をグローバル状態として使います。 ([Claude API Docs][3])

### C. Project scope

* `./CLAUDE.md` または `./.claude/CLAUDE.md`
* `.claude/settings.json`
* `.mcp.json`

プロジェクト共通ルールを書く場所です。repo root の `CLAUDE.md` は起動時に読み込まれ、プロジェクト標準を共有できます。 ([Claude API Docs][1])

### D. Local project scope

* `.claude/settings.local.json`

あなた個人のこの repo 専用のローカル設定です。git に入れません。Anthropic のトラブルシューティングでも、`.claude/settings.local.json` はコミットされないローカルプロジェクト設定として扱われています。 ([Claude API Docs][3])

### E. Subdirectory scope

* `subdir/CLAUDE.md`
* `.claude/rules/`

Claude は working directory より上位の `CLAUDE.md` を起動時に読み、サブディレクトリ内の `CLAUDE.md` はその配下のファイルを読む時に on-demand で読み込みます。大きい repo では topic-specific に分けるのが推奨です。 ([Claude API Docs][4])

---

# 5. このハーネスでの各スコープの役割

## 5-1. User CLAUDE.md

ここには **あなた個人の一貫した好み** だけを書きます。

例:

* 破壊的変更は必ず確認
* 実装より前に現状把握を優先
* ログと handoff を必ず残す
* diff は小さくする
* 説明は初心者向けにする

ここには repo 固有の Dify や n8n の話は書きません。

## 5-2. Project CLAUDE.md

ここには **SWE_P1 の憲法** を書きます。

例:

* この repo の目的
* 現在フェーズは何か
* docs repo がどこか
* Secrets の禁止領域
* queue の状態遷移
* task から始めること
* 1 task = 1 worktree
* `result.md` と `handoff.md` を必ず更新すること

プロジェクト CLAUDE.md は root に置いて全員で共有するのが基本です。Anthropic も repo root にプロジェクトアーキテクチャ、build commands、contribution guidelines を置くのを強く勧めています。 ([Claude API Docs][2])

## 5-3. Subdirectory CLAUDE.md

たとえば `10-apps/dify/CLAUDE.md` や `20-configs/CLAUDE.md` を置きます。

例:

* `10-apps/dify/CLAUDE.md`

  * このディレクトリでは compose / env / Dockerfile を中心に扱う
  * volume 破壊は禁止
  * 起動確認コマンド一覧
* `20-configs/CLAUDE.md`

  * secrets を直接編集しない
  * example と local を分ける
  * schema 変更時は docs 更新

---

# 6. settings ファイルの役割分担

Claude Code は複数の設定ファイルを持ちます。
少なくとも `~/.claude/settings.json`、`.claude/settings.json`、`.claude/settings.local.json`、`~/.claude.json`、`.mcp.json` を意識して設計する必要があります。 ([Claude API Docs][3])

## 6-1. `~/.claude/settings.json`

個人設定。
ここには次を書くのが基本です。

* デフォルトの permission 方針
* ユーザー共通 hooks
* グローバルな deny
* MCP の個人設定
* モデルの既定

## 6-2. `.claude/settings.json`

チーム共有設定。
ここには次を書きます。

* プロジェクトの `permissions.allow/deny`
* プロジェクト共通 hooks
* shared subagent 制約
* log 保存先
* 安全ルール

## 6-3. `.claude/settings.local.json`

この repo だけであなたが一時的に変える場所です。

* 一時的に network を許す
* 一時的に 특정 tool を許す
* 個人用の debug hook
* その日の運用でだけ必要な override

## 6-4. `.mcp.json`

MCP を使う時だけ。
ただしこのハーネス第1版では、MCP は必須にしません。

---

# 7. Permissions 設計

Claude Code は permission-first で始めるのが自然です。Quickstart でも、Claude Code はファイル変更前に permission を求めると案内されています。 ([Claude API Docs][5])

## 7-1. このハーネスの初期値

### Triage Claude

* 基本 read-only
* コード編集禁止
* 許可対象は `.ops/queue/**`, `.ops/reports/**`, docs repo の読み取り
* network off

### Execution Claude

* workspace-write
* 自分の worktree 配下のみ編集可
* 親 repo と他 worktree は deny
* `20-configs/secrets/**` は deny
* network はデフォルト off、必要 task だけ local settings で on

### Review Claude

* read-only
* diff, logs, result, review だけ読む
* 編集は `review.md` と `handoff.md` 程度に限定

## 7-2. Subagent の permissionMode

公式の subagents 設定には `permissionMode` があり、少なくとも `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` が documented です。 subagent は description を見て委譲され、手動追加した場合はセッション再起動か `/agents` で即時読み込みできます。 ([Claude API Docs][6])

このハーネスではこう使います。

* `triage-manager` : `plan`
* `executor-general` : `default`
* `reviewer` : `plan` か `dontAsk` だが read-only
* `log-summarizer` : `dontAsk` + read-only

---

# 8. Subagents 設計

Claude は subagents を session start 時に読み込み、description をもとに delegation します。subagent は親会話の履歴を全部持たず、局所的な役割に閉じられるので、ハーネスとの相性がかなり良いです。 ([Claude API Docs][6])

## 8-1. 必須 subagents

### `triage-manager`

役割:

* docs repo と `.ops/queue/` を見て proposed task を作る
* 優先度を付ける
* Issue 化すべきものを判断する

tools:

* Read, Glob, Grep
* 必要なら Write は `.ops/queue/**` のみ

### `executor-general`

役割:

* 1 task を 1 worktree で実装する
* plan に従う
* result と handoff を更新する

tools:

* Read, Edit, Write, Bash
* ただし worktree 配下のみ

### `reviewer`

役割:

* 差分レビュー
* 仕様逸脱、破壊的変更、ログ不足を指摘
* ready 戻し or done 候補判断

tools:

* Read, Glob, Grep
* 必要なら Write は `review.md` だけ

### `log-summarizer`

役割:

* 長い stdout/stderr を短くまとめる
* blocked 原因を抽出する

tools:

* Read, Grep

## 8-2. 将来追加して良い subagents

* `infra-safety-checker`
* `docs-syncer`
* `issue-drafter`
* `spec-diff-analyzer`

---

# 9. Skills 設計

Claude Code の skills は `SKILL.md` を核にした再利用パッケージで、必要時にロードされ、直接 `/skill-name` で呼ぶこともできます。大きい reference を毎回全部読み込まないよう supporting files を含められます。 ([Claude API Docs][7])

## 9-1. このハーネスに必要な skills

### `triage-from-state`

やること:

* docs repo
* README
* queue 状態
* recent logs
* recent spec changes
  から proposed task を生成

### `make-task-packet`

やること:

* `task.yaml`
* `brief.md`
* `plan.md`
* `review.md`
  の初期雛形を作る

### `worktree-execution`

やること:

* worktree 生成
* branch 命名
* 実行前チェック
* result / handoff 更新ルール

### `failure-classifier`

やること:

* 実行失敗を

  * environment
  * permission
  * flaky
  * spec ambiguity
  * implementation bug
  * needs human decision
    に分類

### `issue-draft-generator`

やること:

* issue title
* background
* acceptance criteria
* linked docs
* linked task
  を整えて draft 作成

### `review-gate`

やること:

* review 前に最低限見る項目を機械的に揃える

## 9-2. 配置

* 共通 skill は `.claude/skills/`
* 個人 skill は `~/.claude/skills/`

---

# 10. Hooks 設計

Hooks は Claude Code lifecycle の特定地点で自動実行される shell command / HTTP / prompt hook です。hooks は JSON settings に定義し、主なイベントは `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop` です。subagent frontmatter にも hooks を持てます。 ([Claude API Docs][8])

## 10-1. 必須 hooks

### `UserPromptSubmit`

使い道:

* 今の session が queue item を指しているか確認
* queue item 不在なら「先に task packet を開け」と注意
* handoff の存在確認

### `PreToolUse`

使い道:

* `20-configs/secrets/**` アクセス遮断
* worktree 外書き込み遮断
* `git push` 遮断
* `rm -rf` 遮断
* deploy 系コマンド遮断

### `PostToolUse`

使い道:

* 編集後に `git diff --stat`
* changed files を run summary に追記
* formatter / linter 実行
* 失敗したら blocked 候補に記録

### `Stop`

使い道:

* `handoff.md` 更新
* `result.md` 下書き
* current state を `.ops/runs/.../summary.md` に保存

### `SubagentStop`

使い道:

* triage 結果を queue に反映
* reviewer コメントを `review.md` へ統合

---

# 11. CLAUDE.md の実際の分割方針

## 11-1. root CLAUDE.md に書くもの

* この repo の目的
* 現在フェーズ
* docs repo の場所
* queue state machine
* 1 task = 1 worktree
* task packet の必須ファイル
* Secrets の絶対禁止
* done の定義
* review の定義

## 11-2. 書かないもの

* 長い API リファレンス
* 1回限りの指示
* 細かいテンプレ本文
* 具体的な失敗ログの山

これらは skills / references / runs に逃がします。

---

# 12. Queue 設計

## 12-1. 状態

このハーネスでは queue 状態を7つにします。

* `inbox`
  雑多な観測、メモ、追加要求
* `proposed`
  Triage Claude が「次これ」と出した候補
* `ready`
  いつでも実行開始できる
* `running`
  Claude が実行中
* `review`
  実装完了、レビュー待ち
* `blocked`
  人間判断、権限、環境、不確実性で停止
* `done`
  完了、必要なら docs に昇格

## 12-2. 1件の task packet

各 task はフォルダで持ちます。

```text
.ops/queue/proposed/Q-P1-0042-dify-compose-fix/
├── task.yaml
├── brief.md
├── plan.md
├── review.md
├── handoff.md
├── result.md
├── issue.md
└── links.md
```

### `task.yaml`

最低限:

* id
* phase
* title
* priority
* reason_now
* target_paths
* forbidden_paths
* definition_of_done
* related_docs
* issue_id
* worktree_path
* status

### `brief.md`

* 背景
* 今困っていること
* 触る範囲
* 触らない範囲
* 最低限のゴール

### `plan.md`

* 実行前の計画
* 何を確認してから編集するか
* テスト方針

### `review.md`

* レビュー観点
* 成功基準
* 差し戻し条件

### `handoff.md`

* いま何が終わったか
* どこで止まったか
* 次の1手

### `result.md`

* 最終サマリー

### `issue.md`

* Issue 化用の整形済み文章

---

# 13. Issues / Projects の位置づけ

GitHub Issues は work の議論・追跡単位、Projects は issue / PR に custom fields を付けて横断可視化する単位です。Projects は issue と PR の変更を双方向に同期でき、custom fields や built-in automation、Actions 連携もあります。PR は issue にリンクでき、merge 時に自動 close もできます。 ([GitHub Docs][9])

## 13-1. この設計での立ち位置

### source of truth

* **実行の truth**: `.ops/queue/` と `.ops/runs/`
* **管理の dashboard**: GitHub Projects
* **議論と中粒度 work**: GitHub Issues

つまり、Issue は使います。
ただし **micro task まで全部 issue にしません。**

## 13-2. Issue にする基準

### Issue にする

* 数時間〜数日かかる
* PR 単位になる
* 他 project と比較対象になる
* 仕様変更の議論がある
* 後で検索したい

### queue だけで済ませる

* 設定名1個修正
* path typo 修正
* flaky 再試行
* 軽微な cleanup

## 13-3. Project の custom fields

* `Project` : P1 / P2 / P3
* `Phase`
* `Priority`
* `State`
* `Area`
* `Executor`
* `Needs Human`
* `Issue Type`
* `Ready Score`
* `Linked Task ID`

---

# 14. Claude と Issue 管理の接続

Claude-only で Issue 管理まで寄せるなら、2パターンあります。

## 14-1. Local-first

ローカル Claude が `issue.md` を作る。
あなたがそれを GitHub に投げる。

これは最初の導入に向いています。

## 14-2. GitHub-integrated

Claude Code GitHub Actions を使う。
Anthropic の公式では、issue や PR に `@claude` とメンションするだけで Claude が分析、実装、PR 作成まででき、`/install-github-app` で初期セットアップを支援します。Claude Agent SDK 上に構築されていて、安全面でも GitHub runner 内に留まる前提です。 ([Claude API Docs][10])

この設計では、最初は Local-first で始め、ハーネスが安定したら GitHub-integrated に寄せるのがいいです。

---

# 15. Worktree 設計

Claude 固有機能ではありませんが、このハーネスでは必須です。
**1 task = 1 worktree** を絶対ルールにします。

## 15-1. 置き場所

```text
/Users/miurakousuke/GitHub/_worktrees/
└── swe_p1/
    ├── Q-P1-0042-dify-compose-fix/
    ├── Q-P1-0043-n8n-min-flow/
    └── Q-P1-0044-pgvector-ddl/
```

## 15-2. ルール

* main working tree では実装しない
* `ready` から worktree を切る
* `running` 中はその worktree の中で Claude を起動
* `review` までは worktree を残す
* `done` 後に削除
* `blocked` は凍結して残す

## 15-3. Claude の session 運用

Claude の session は project directory 単位で保存され、`/resume` picker は同じ git repository 内の session を見ます。公式 docs では same repository の worktrees も resume 対象になります。 ([Claude API Docs][11])

---

# 16. CLI 起動パターン

Claude Code は terminal, IDE, desktop, browser で使えますが、このハーネスは terminal 中心で組みます。Claude Code は codebase を読み、編集し、command を実行し、開発ツールと統合できます。 ([Claude API Docs][12])

## 16-1. 基本

```bash
cd /path/to/worktree
claude
```

起動後は `/help` と `/resume` が基本です。 ([Claude API Docs][11])

## 16-2. 初期セットアップ

```bash
claude
/init
```

`/init` は既存 codebase を見て `CLAUDE.md` を作るか改善提案し、`CLAUDE_CODE_NEW_INIT=true` なら CLAUDE.md / skills / hooks まで含む multi-phase flow を提案します。 ([Claude API Docs][1])

## 16-3. セッション整理

* `/clear` : 履歴をクリア
* `/compact` : コンテキスト圧縮
* `/resume` : 過去会話へ切替
* `/rename` : セッション名変更

`/compact` と `manage sessions` は公式の common workflows / memory docs でも重要な日常運用として扱われています。 ([Claude API Docs][13])

## 16-4. 構成変更

* `/config`
* `/permissions`
* `/sandbox`
* `/hooks`
* `/mcp`
* `/model`
* `/agents`

## 16-5. デバッグ / 診断

* `/doctor`
* `/debug`

## 16-6. Commands の注意

release notes にある通り、Claude Code は version と auth setup によって hidden になる slash commands があります。古い一覧を固定前提にせず、**毎回 `/help` を正** にするのが安全です。 ([Claude API Docs][14])

---

# 17. このハーネスで実際に使う /commands

## Triage 用

* `/init`
  初期骨格生成
* `/agents`
  triage-manager / reviewer など読み込み
* `/compact`
  長い triage session の整理
* `/resume`
  前日の triage session 継続
* `/hooks`
  triage guardrail 調整

## Execution 用

* `/permissions`
  今の task 用に allow/deny 微調整
* `/sandbox`
  実行環境確認
* `/model`
  重い task で model 切り替え
* `/compact`
  長い debugging 途中で圧縮
* `/debug`
  Claude Code 自体の不調確認
* `/doctor`
  環境診断

## GitHub 連携用

* `/install-github-app`
  GitHub Actions 側へ寄せる段階で使用可能 ([Claude API Docs][10])

---

# 18. 日次ワークフロー

## 18-1. 朝 or 作業開始時: Triage フロー

1. `swe_p1_docs` の relevant doc を読む
2. `README`, `CLAUDE.md`, `.ops/queue/`, `.ops/runs/`, recent diffs を読む
3. `blocked`, `review`, stale `running` を確認
4. 仕様変更や追加要求が `inbox` にあるか確認
5. `proposed` を再生成または更新
6. priority 順に並べる
7. 必要なものだけ Issue draft 作成

このとき Triage Claude は、**現在 phase に直結するもの、依存を解放するもの、blocked を解消するものを優先** します。
SWE_P1 は現フェーズが Phase1 ローカル基盤構築なので、まず Dify / n8n / DB 接続・CRUD が最優先です。 

## 18-2. あなたの介入

* `proposed` 上位を見る
* 今日回す本数を決める
* それだけ `ready` に移す

---

# 19. 成功パターン

## 19-1. 標準成功フロー

1. `proposed` → `ready`
2. worktree 作成
3. Claude 起動
4. `task.yaml`, `brief.md`, `plan.md` を読む
5. 事前確認
6. 実装
7. テスト
8. `result.md`, `handoff.md`, `review.md` 更新
9. `running` → `review`
10. Review Claude が確認
11. あなたが PR / merge 判断
12. `done`

Claude の common workflows は、新しい codebase 理解、bug fix、refactor、test、PR 作成、session 管理まで日常フローをカバーしています。 ([Claude API Docs][13])

---

# 20. 失敗パターン

## 20-1. 環境失敗

例:

* compose 起動失敗
* 依存不足
* path 不整合
* auth 問題

対応:

* `failure-classifier` skill で分類
* `blocked` に移す
* `blocked_reason.md` に原因を書く
* 同種失敗が続くなら high priority の infra-fix task を `proposed` に追加

## 20-2. 権限失敗

例:

* permission deny
* sandbox 制限
* hook block

対応:

* 原則その場で bypass しない
* `needs-human-decision: true`
* 人間が local settings で一時開放するか判断

## 20-3. 仕様曖昧

例:

* どちらのデータ構造が正なのか不明
* acceptance criteria が曖昧

対応:

* `blocked`
* `spec-question.md` を作る
* Issue に落として議論

## 20-4. Claude 自体の運用失敗

例:

* command discovery が不安定
* session が肥大化
* context が崩れた

対応:

* `/compact`
* `/resume`
* `/debug`
* 必要なら session 切り替え

---

# 21. タスク継続パターン

## 21-1. 同じ日で継続

* そのまま session 継続
* 定期的に `/compact`
* `handoff.md` は区切りごとに更新

## 21-2. 翌日継続

* 同じ worktree に入る
* `claude`
* `/resume`
* まず `handoff.md` を読む
* その後で続ける

## 21-3. session を切り替えて継続

Claude は fresh context 前提なので、**継続の基点は会話ではなく handoff** に置きます。
`/resume` は便利ですが、source of truth にはしません。 ([Claude API Docs][15])

---

# 22. 仕様変更パターン

これはかなり大事です。

## 22-1. 仕様変更が来た時

1. 変更要求を `inbox` に置く
2. Triage Claude が既存 task / issue / PR への影響を洗う
3. `impact.md` を作る
4. 影響先を

   * 継続可能
   * 途中中断
   * 破棄して再計画
     に分類
5. 必要なら既存 `ready/running` を `blocked` に移す
6. 新しい `proposed` を生成

## 22-2. ルール

* 走っている task を silent update しない
* 仕様変更が大きい時は task ID を分ける
* 変更理由を docs repo に残す

---

# 23. 追加機能パターン

## 23-1. 小さい追加

* 既存 issue の subtask
* queue item 追加のみ
* same epic 内で処理

## 23-2. 中〜大きい追加

* 新規 issue
* Project item 追加
* 既存 phase との整合確認
* `proposed` に入れて優先度再計算

GitHub Issues はアイデア、バグ、新機能の計画・議論・追跡に向いており、sub-issues で階層化もできます。Projects は issue / PR のメタデータ管理と横断可視化に強いです。 ([GitHub Docs][9])

---

# 24. Review パターン

## 24-1. Review Claude

見るもの:

* diff
* `plan.md`
* `review.md`
* `result.md`
* test output
* changed files list

判定:

* approve candidate
* fix before review
* blocked by ambiguity
* unsafe change

## 24-2. あなた

* review を読む
* PR を見る
* merge / hold / reject を決める

PR は issue にリンクでき、merge で自動 close させられます。 ([GitHub Docs][16])

---

# 25. Issue / PR / Project のワークフロー

## 25-1. 標準

1. Triage Claude が `issue.md` draft を作る
2. issue 作成
3. Project に追加
4. `proposed`
5. `ready`
6. `running`
7. PR 作成
8. PR に issue をリンク
9. review
10. merge
11. issue close
12. `done`

Projects は issue と PR の変更が自動同期され、Actions や built-in automation でも自動化できます。 ([GitHub Docs][17])

## 25-2. Claude GitHub Actions を使う場合

* issue で `@claude` に初動整理をさせる
* PR で `@claude review` 的な運用に寄せる
* main harness は local queue のまま
* GitHub 側は tracking と review 拡張に使う

---

# 26. ログ管理

## 26-1. 原則

**1 run = 1フォルダ**

```text
.ops/runs/
└── 2026-03-31/
    └── Q-P1-0042/
        └── run-001/
            ├── manifest.json
            ├── stdout.log
            ├── stderr.log
            ├── summary.md
            ├── review-input.md
            ├── changed-files.txt
            └── artifacts/
```

## 26-2. 各ファイル

### `manifest.json`

* task id
* worktree path
* branch
* model
* started_at
* ended_at
* result
* linked issue/pr

### `summary.md`

* 何をしたか
* 何が終わったか
* 何が残ったか
* 次の1手

### `stdout.log` / `stderr.log`

生ログ

### `review-input.md`

Review Claude にそのまま渡す要約

### `artifacts/`

* screenshots
* compose config output
* test report
* diff snapshots

---

# 27. docs 管理

## 27-1. docs repo に残すもの

* phase 設計
* long-lived decisions
* architecture
* schema rationale
* acceptance criteria の変更
* 重要な失敗学習

## 27-2. main repo に残すもの

* queue
* runs
* task packets
* issue drafts
* operational reports

---

# 28. 失敗からの再開設計

このハーネスで一番大事なのはここです。

## 28-1. Claude が途中で変になった

* `/compact`
* `/resume`
* だめなら session 切替
* `handoff.md` を正として継続

## 28-2. Mac 再起動 / tmux 切断

* worktree は残る
* queue state は残る
* logs は残る
* `handoff.md` も残る

だから致命傷になりません。
会話が飛んでも作業は飛ばない構造にします。

## 28-3. blocked が溜まった

* Triage Claude が毎朝 `blocked` を再スキャン
* 類似原因を束ねる
* 高優先度の unblock task を `proposed` に作る

---

# 29. CLI / slash command の現実的セット

最初の1か月は、実際にはこれだけ使えれば十分です。

## 毎日使う

* `claude`
* `/help`
* `/resume`
* `/compact`
* `/agents`
* `/permissions`
* `/model`

## 問題が出た時

* `/debug`
* `/doctor`
* `/sandbox`
* `/hooks`

## 導入時だけ

* `/init`
* `/install-github-app`

---

# 30. 初期導入手順

## Phase A

* root `CLAUDE.md`
* `.claude/settings.json`
* `.ops/queue/`
* `.ops/runs/`
* task packet 雛形
* `triage-manager`, `executor-general`, `reviewer`

## Phase B

* hooks
* skills
* subdirectory `CLAUDE.md`
* Project fields

## Phase C

* GitHub Issues / Projects 完全連携
* Claude Code GitHub Actions
* stale item 自動再トリアージ

## Phase D

* 必要なら Claude Agent SDK で triage の一部を daemon 化
  ただし第1版は CLI だけで十分です。Agent SDK は Claude Code と同じ tools / agent loop / context management をプログラムから使える拡張先ですが、今はまだ不要です。 ([Claude API Docs][18])

---

# 31. 忖度なしの最終判断

あなたの条件だと、Claude-only 第1版としてはこの構成が一番強いです。

**採用すべきもの**

* root / subdir `CLAUDE.md`
* `.claude/settings.json` / `.claude/settings.local.json`
* subagents
* skills
* hooks
* queue state machine
* 1 task = 1 worktree
* Issue / Projects 併用
* `handoff.md` と `result.md` を source of truth にする

**まだ採用しなくていいもの**

* いきなり完全自動 merge
* Claude 任せの勝手な実行開始
* API ベースの自前 Agent SDK 常駐化
* MCP 盛り盛り構成

**一番危ないもの**

* 毎回プロンプトで「あなたは第2層です」と宣言するだけの運用
* 会話履歴を記憶の本体にする運用
* micro task まで全部 issue 化する運用
* main working tree で直接実装する運用

この設計なら、まず Claude だけで十分に回せます。
その後に Codex や他ツールへ広げても、**queue / task packet / worktree / issue-project / handoff** の骨格はそのまま流用できます。

次にやるべきなのは、この設計を前提に
**実際の `CLAUDE.md`、`settings.json`、subagent frontmatter、task.yaml、handoff.md の雛形** をコピペ可能な形で作ることです。

[1]: https://docs.anthropic.com/ja/docs/claude-code/memory?utm_source=chatgpt.com "Claude があなたのプロジェクトを記憶する方法"
[2]: https://docs.anthropic.com/en/docs/claude-code/third-party-integrations?utm_source=chatgpt.com "Enterprise deployment overview - Claude Code Docs"
[3]: https://docs.anthropic.com/ja/docs/claude-code/troubleshooting?utm_source=chatgpt.com "トラブルシューティング - Claude Code Docs"
[4]: https://docs.anthropic.com/en/docs/claude-code/memory?utm_source=chatgpt.com "How Claude remembers your project - Claude Code Docs"
[5]: https://docs.anthropic.com/en/docs/claude-code/quickstart?utm_source=chatgpt.com "Quickstart - Claude Code Docs"
[6]: https://docs.anthropic.com/en/docs/claude-code/sub-agents?utm_source=chatgpt.com "Create custom subagents - Claude Code Docs"
[7]: https://docs.anthropic.com/en/docs/claude-code/skills?utm_source=chatgpt.com "Extend Claude with skills - Claude Code Docs"
[8]: https://docs.anthropic.com/en/docs/claude-code/hooks?utm_source=chatgpt.com "Hooks reference - Claude Code Docs"
[9]: https://docs.github.com/articles/about-issues?utm_source=chatgpt.com "About issues"
[10]: https://docs.anthropic.com/ja/docs/claude-code/github-actions?utm_source=chatgpt.com "Claude Code GitHub Actions"
[11]: https://docs.anthropic.com/en/docs/claude-code/quickstart "https://docs.anthropic.com/en/docs/claude-code/quickstart"
[12]: https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview?utm_source=chatgpt.com "Claude Code overview - Claude Code Docs"
[13]: https://docs.anthropic.com/en/docs/claude-code/common-workflows?utm_source=chatgpt.com "Common workflows - Claude Code Docs"
[14]: https://docs.anthropic.com/en/release-notes/claude-code?utm_source=chatgpt.com "claude-code/CHANGELOG.md at main"
[15]: https://docs.anthropic.com/en/docs/claude-code/common-workflows "https://docs.anthropic.com/en/docs/claude-code/common-workflows"
[16]: https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue?utm_source=chatgpt.com "Linking a pull request to an issue"
[17]: https://docs.github.com/ja/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects?utm_source=chatgpt.com "Projects の について - GitHub ドキュメント"
[18]: https://docs.anthropic.com/en/docs/claude-code/sdk?utm_source=chatgpt.com "Agent SDK overview - Claude API Docs"
