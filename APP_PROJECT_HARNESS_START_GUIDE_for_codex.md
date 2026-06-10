# 草案ありのアプリ開発プロジェクトをハーネス運用に載せる開始ガイド

## このドキュメントの目的

このドキュメントは、すでにアプリの草案・概要メモがある状態から、AIエージェント運用ハーネスに載せるための手順書です。

目的は、いきなり実装に入ることではありません。草案を開発用ドキュメントに整理し、`.ops`、`CLAUDE.md`、Claude Code設定、rules / skills / agents、task packet 雛形を準備し、人間が安全に `proposed → ready → running → review → done / blocked` を判断できる状態を作ることです。

## 対象読者

- 非エンジニア寄りだが、AIエージェントでアプリ開発を進めたい人
- Claude Code CLI を主力に使い、Codex / Antigravity / Cursor も補助的に使う可能性がある人
- 就寝中・仕事中にAIエージェントを安全に動かしたい人
- 会話履歴ではなく、ファイルを見れば再開できる運用にしたい人

## このドキュメントで扱う範囲

- 草案から `00-docs/` を作る手順
- `.ops/`、queue、task packet 雛形の作り方
- `CLAUDE.md`、`.claude/settings.json`、`.claude/rules/`、`.claude/skills/`、`.claude/agents/` の役割分担
- 最初の triage と、最初の `ready` タスクを1つだけ実行するまでの流れ
- タスク開始後に人間が見るべきファイル
- 朝、就寝前、問題発生時、セッション切断時の再開手順

## このドキュメントで扱わない範囲

- 具体的なアプリ名、機能、画面、業務内容の決定
- 技術スタック、フレームワーク、データベース、クラウド構成の決定
- 本番環境構成、デプロイ方式、課金サービス利用方針の決定
- secrets、認証情報、本番データをAIに読ませる運用
- 人間ゲートを飛ばして完全自動で merge する運用

## 最初に守るルール

- [ ] current mode 前提にする。セッション起動は人間が行う。
- [ ] 「AI自動」は「セッション起動後にAIが自律実行する」という意味で使う。
- [ ] `1 task = 1 worktree = 1 branch` を守る。
- [ ] task packet は自己完結させる。
- [ ] `HANDOFF.md` を再開の正本にする。
- [ ] `DASHBOARD.md` を人間の中心画面にする。
- [ ] `proposed → ready` は人間が判断する。
- [ ] `review → done / merge` は人間が判断する。
- [ ] 仕様が曖昧なうちは実装タスクにしない。
- [ ] 破壊的操作、課金、secrets、本番操作は必ず停止して人間確認にする。

---

# 1. 全体の流れ

草案があるアプリ開発プロジェクトをハーネス運用に載せる流れは、次の順番にします。

この順番の理由は、上位の正本を先に作り、その後に運用構造を作り、最後に小さい task packet へ落とすためです。先に実装タスクを作ると、仕様未決定、完了条件不足、権限不足、secrets問題が後から出て止まりやすくなります。

| 順番 | 作業 | 何をするか | なぜこの順番か |
|---:|---|---|---|
| 0 | 安全な作業場所を決める | 草案、設定、ハーネス関連ファイルを置く repo を決める | secrets や本番設定を混ぜないため |
| 1 | 草案を整理する | 既存メモから目的、ユーザー、機能、未決定を抜き出す | 生メモのままだとAIが勝手に補完しやすい |
| 2 | `00-docs` を作る | 目的、要件、ロードマップ、決定事項、現在地を分ける | 以後の全タスクの正本になる |
| 3 | `.ops` を作る | queue、runs、reports、templates、config を作る | タスク状態を会話ではなくファイルで管理するため |
| 4 | `CLAUDE.md` を作る | プロジェクト憲法、正本、状態定義、役割判定を書く | Claude Codeセッションの共通前提になる |
| 5 | `settings.json` を作る | deny-first の権限、安全境界、hook方針を置く | 書いてあるだけのルールではなく、物理的に止めるため |
| 6 | `rules / skills / agents` を作る | 状況依存ルール、再利用手順、役割分離を定義する | 短いプロンプトでも構造で動けるようにするため |
| 7 | task packet 雛形を作る | `task.yaml`、`BRIEF.md`、`PLAN.md`、`HANDOFF.md` などを準備する | 1タスクを自己完結させるため |
| 8 | 初回 triage をする | `00-docs` と `.ops` を読み、候補を整理する | いきなり実装せず、候補を `proposed` に留めるため |
| 9 | `proposed` を作る | 調査、整理、小さい準備タスクを候補化する | AI提案と人間承認を分離するため |
| 10 | 人間が `ready` に昇格する | 今日回すタスクを人間が1つ選ぶ | 優先順位とリスクは人間判断にするため |
| 11 | worktree で実行する | `1 task = 1 worktree = 1 branch` で実行する | main working tree を汚さず並列可能にするため |
| 12 | review する | Review Agent が `REVIEW.md` を書き、人間が見る | 実装者とレビューを分け、判断材料を整えるため |
| 13 | 人間が `done / merge / blocked` を判断する | merge、差し戻し、保留、blocked解除を決める | 最終責任がある操作は人間が判断するため |

## 後処理優先の原則

セッション開始時や朝の確認では、新規タスクより先に後処理を片付けます。

処理順は固定します。

1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

新規着手に進んでよい条件:

- [ ] stale running がない、または救済方針が決まっている
- [ ] blocked のうち、今すぐ人間判断できるものを処理済み
- [ ] review が溜まりすぎていない
- [ ] 今日動かす `ready` が1つに絞れている
- [ ] `afk_safe` かどうか判断済み

---

# 2. 一括作成による品質低下を防ぐための作業分割

タスク開始前の準備を全部まとめてAIに作らせると、後半の品質が落ちやすくなります。特に `CLAUDE.md`、`settings.json`、rules、skills、agents は責務が違うため、別セッションで作る方が安全です。

## 分割の基本方針

- 一括で作ってよいもの: 雛形、空ディレクトリ、初期テンプレート
- 分けるべきもの: 仕様の正本、安全設定、役割定義、実行手順、初回 triage
- 人間確認が必要なもの: `proposed → ready`、権限変更、課金、本番、secrets、merge
- 最初の実行は1タスクだけにする

## セッションA: 草案 → `00-docs` 整理

| 項目 | 内容 |
|---|---|
| 読ませるファイル | 草案メモ、既存の概要資料、`harness_design-D_unified.md` の基本原則 |
| 作らせるファイル | `00-docs/project_overview.md`、`00-docs/requirements.md`、`00-docs/roadmap.md`、`00-docs/decisions.md`、`00-docs/status.md` |
| 作らせてはいけないもの | 実装コード、技術スタック決定、`.claude/settings.json`、task packet、`ready` タスク |
| 完了条件 | 決定事項、仮決定、未決定が分かれている。初期版でやること / やらないことが分かれている。仕様曖昧な項目が明示されている |
| 分割する理由 | ここが正本になるため。後続の設定やタスク化の品質が `00-docs` の品質に依存する |

## セッションB: `.ops / queue / templates` 作成

| 項目 | 内容 |
|---|---|
| 読ませるファイル | `00-docs/status.md`、`00-docs/roadmap.md`、`harness_design-D_unified.md` の queue / task packet 部分 |
| 作らせるファイル | `.ops/DASHBOARD.md`、`.ops/queue/{inbox,proposed,ready,running,review,blocked,done}/`、`.ops/runs/`、`.ops/reports/`、`.ops/config/parallel.yaml`、`.ops/templates/task-packet/` |
| 作らせてはいけないもの | 具体タスクの `ready` 昇格、worktree作成、実装、merge |
| 完了条件 | queue状態のディレクトリが揃っている。task packet雛形がある。`DASHBOARD.md` が人間向けの空テンプレートとして読める |
| 分割する理由 | `.ops` は運用機械なので、仕様整理と混ぜると構造が雑になりやすい |

## セッションC: `CLAUDE.md` 作成

| 項目 | 内容 |
|---|---|
| 読ませるファイル | `00-docs/project_overview.md`、`00-docs/status.md`、`.ops` 構成、`harness_design-D_unified.md` の CLAUDE.md / role detection 部分 |
| 作らせるファイル | `CLAUDE.md` |
| 作らせてはいけないもの | permissionsの詳細、hooks実装、具体タスク、アプリ仕様の新規決定 |
| 完了条件 | Source of Truth、Non-negotiables、Queue State Machine、Task Packet Required Files、Role Detection、Default Actions が書かれている |
| 分割する理由 | `CLAUDE.md` は憲法であり、settingsやskillsと責務が違う。長い実行手順を書きすぎると保守しにくい |

## セッションD: `settings.json` 作成

| 項目 | 内容 |
|---|---|
| 読ませるファイル | `CLAUDE.md`、`.ops/config/parallel.yaml`、`harness_design-D_unified.md` の permissions / hooks / safety 部分、Claude Codeの実際のsettings仕様 |
| 作らせるファイル | `.claude/settings.json` の初期案、必要なら `.claude/settings.local.json.example` |
| 作らせてはいけないもの | secretsの読み取り、本番接続、危険コマンドのallow、スキーマ未確認の断定的設定 |
| 完了条件 | secrets、破壊的操作、本番操作、worktree外書き込みが止まる方針になっている。Claude Codeの現在の仕様確認ポイントが明記されている |
| 分割する理由 | 安全設定は間違えると危険。Claude Codeのsettingsスキーマが変わる可能性があるため、別セッションで慎重に確認する |

## セッションE: `rules / skills / agents` 作成

| 項目 | 内容 |
|---|---|
| 読ませるファイル | `CLAUDE.md`、`.ops/templates/task-packet/`、`.claude/settings.json`、`harness_design-D_unified.md` の rules / skills / agents 部分 |
| 作らせるファイル | `.claude/rules/`、`.claude/skills/`、`.claude/agents/` の初期ファイル |
| 作らせてはいけないもの | 実装コード、権限の緩和、`proposed → ready` の自動化、`review → done` の自動化 |
| 完了条件 | role-detection、autonomous-ops、safety、docs-output の初期rulesがある。triage、executor、reviewer のagent定義が分かれている |
| 分割する理由 | rulesは判断基準、skillsは手順、agentsは役割分離。混ぜるとAIが迷いやすい |

## セッションF: 初回 triage

| 項目 | 内容 |
|---|---|
| 読ませるファイル | `00-docs/`、`CLAUDE.md`、`.ops/DASHBOARD.md`、`.ops/queue/`、`.ops/templates/task-packet/` |
| 作らせるファイル | `.ops/reports/triage/TRIAGE_REPORT_*.md`、`.ops/reports/triage/CANDIDATES_*.yaml`、必要な `proposed` task packet |
| 作らせてはいけないもの | `ready` への昇格、worktree作成、実装、外部課金、secrets参照 |
| 完了条件 | 候補が `proposed` に留まっている。各候補に完了条件、対象範囲、禁止範囲、AFK可否、リスクがある |
| 分割する理由 | triageは候補作成であり、実行判断ではない。人間ゲートを保つため |

## セッションG: 最初の `ready` タスクを1つだけ実行

| 項目 | 内容 |
|---|---|
| 読ませるファイル | 選んだtask packet一式、`CLAUDE.md`、関連する `00-docs` |
| 作らせるファイル | worktree、branch、更新された `HANDOFF.md`、`RUNLOG.md`、`VERIFY.md`、`RESULT.md`、必要なら `REVIEW.md` の準備 |
| 作らせてはいけないもの | 2個目の同時タスク、main working tree直接編集、merge、本番操作、課金、secrets読み取り |
| 完了条件 | `HANDOFF.md` が更新されている。成功なら `review` に移る。止まるなら `blocked_reason.md` または `spec-question.md` がある |
| 分割する理由 | 初回は運用確認が目的。並列や本格実装に入ると、問題発生時の原因が分からなくなる |

## 追加で分けた方がよいセッション

必要に応じて、以下は別セッションにします。

| セッション | 分ける理由 |
|---|---|
| GitHub Issues / Projects連携 | 外部サービス権限が絡むため |
| hooks実装 | 安全装置だが、誤実装すると作業全体を止めるため |
| 初回review | 実装者とreviewerのcontextを分けるため |
| secrets / 本番 / 課金の棚卸し | AIに読ませず、人間が管理方針だけを渡すため |

---

# 3. ドキュメント作成の順番

## 作る順番

| 順番 | ファイル / ディレクトリ | 分類 | 作り方 |
|---:|---|---|---|
| 1 | `00-docs/project_overview.md` | 1つずつ丁寧に作るべきもの | 草案から目的と範囲を整理する |
| 2 | `00-docs/requirements.md` | 1つずつ丁寧に作るべきもの | 機能要件、非機能要件、未決定を分ける |
| 3 | `00-docs/roadmap.md` | 1つずつ丁寧に作るべきもの | 初期版、後回し、やらないことを分ける |
| 4 | `00-docs/decisions.md` | 雛形だけでよいもの | 決定事項 / 仮決定 / 未決定を記録する器を作る |
| 5 | `00-docs/status.md` | 最初に一括作成してよいもの | 現在地、次の確認、未解決を短く書く |
| 6 | `.ops/queue/` | 最初に一括作成してよいもの | 7状態の空ディレクトリを作る |
| 7 | `.ops/DASHBOARD.md` | 最初に一括作成してよいもの | 人間向けの中心画面テンプレートを作る |
| 8 | `.ops/config/parallel.yaml` | 雛形だけでよいもの | 最初は同時実行1または少なめにする |
| 9 | `.ops/templates/task-packet/` | 最初に一括作成してよいもの | 必須ファイルの雛形を作る |
| 10 | `CLAUDE.md` | 1つずつ丁寧に作るべきもの | 正本、状態、役割、非交渉ルールを書く |
| 11 | `.claude/settings.json` | 1つずつ丁寧に作るべきもの | 実際のClaude Code仕様を確認しながら厳しめに作る |
| 12 | `.claude/rules/` | 1つずつ丁寧に作るべきもの | 条件と行動のルールを書く |
| 13 | `.claude/skills/` | 雛形だけでよいもの | 最初は重要skillsの `SKILL.md` 雛形でよい |
| 14 | `.claude/agents/` | 雛形だけでよいもの | triage / executor / reviewer を分ける |

## 分類別の考え方

### 最初に一括作成してよいもの

- `.ops/queue/{inbox,proposed,ready,running,review,blocked,done}/`
- `.ops/runs/`
- `.ops/reports/triage/`
- `.ops/reports/daily/`
- `.ops/templates/task-packet/`
- `.ops/DASHBOARD.md` の空テンプレート
- `00-docs/status.md` の初期テンプレート

### 1つずつ丁寧に作るべきもの

- `00-docs/project_overview.md`
- `00-docs/requirements.md`
- `00-docs/roadmap.md`
- `CLAUDE.md`
- `.claude/settings.json`
- `.claude/rules/safety.md`
- `.claude/rules/autonomous-ops.md`
- `.claude/rules/role-detection.md`

### 雛形だけでよいもの

- `00-docs/decisions.md`
- `.ops/config/parallel.yaml`
- `.claude/skills/triage-from-state/SKILL.md`
- `.claude/skills/make-task-packet/SKILL.md`
- `.claude/skills/task-handoff/SKILL.md`
- `.claude/skills/review-gate/SKILL.md`
- `.claude/agents/triage.md`
- `.claude/agents/executor.md`
- `.claude/agents/reviewer.md`

### 実運用後に改善すればよいもの

- `.claude/skills/issue-sync/SKILL.md`
- `.claude/skills/impact-analyzer/SKILL.md`
- `.claude/skills/auto-advance/SKILL.md`
- `.claude/skills/dashboard/SKILL.md`
- `.ops/config/scoring-weights.yaml`
- GitHub Issues / Projects連携
- hooksの高度な自動化

### 最初は作らなくてもよいもの

- 本番デプロイ自動化
- 課金APIの自動実行
- secretsを読む前提の設定
- `/schedule` による無人定期実行
- GitHub Actionsからの自動実装
- 複数タスクの並列実行

---

# 4. `00-docs` の作り方

`00-docs` は計画と仕様の正本です。ここで曖昧なものを曖昧なまま実装タスクに落とさないことが重要です。

作成時の共通ルール:

- [ ] 技術スタックを勝手に決めない
- [ ] 決定事項 / 仮決定 / 未決定を分ける
- [ ] 初期版でやること / やらないことを分ける
- [ ] 仕様が曖昧なものは実装タスクにしない
- [ ] 草案に書かれていないことは「未決定」として扱う
- [ ] secrets、認証情報、本番接続情報は書かない

## `00-docs/project_overview.md`

### 書くこと

- アプリ開発プロジェクトの目的
- 誰のためのものか
- どんな課題を解くのか
- 初期版の範囲
- 明確にやらないこと
- 重要な制約
- 未決定事項の一覧

### 書かないこと

- 技術スタックの断定
- DBやクラウド構成の決定
- APIキー、認証情報、環境変数の中身
- 実装手順
- task packetの詳細

### 作成プロンプト例

```text
草案メモを読み、アプリ開発プロジェクトの 00-docs/project_overview.md を作成してください。

要件:
- 具体的なアプリ名や技術スタックを新しく決めない
- 草案に書かれている事実、推測、未決定を分ける
- 初期版でやること / やらないことを分ける
- 非エンジニアが読める日本語にする
- secrets、認証情報、本番情報は含めない

出力:
- 00-docs/project_overview.md
```

## `00-docs/requirements.md`

### 書くこと

- 機能要件
- 非機能要件
- 受け入れ条件
- ユーザー操作や利用シーン
- 未確定要件
- 実装前に質問が必要な項目

### 書かないこと

- フレームワーク名の決定
- DB製品名の決定
- 本番運用方式
- 「たぶん必要」だけの機能を確定要件として書くこと

### 作成プロンプト例

```text
草案メモと 00-docs/project_overview.md を読み、00-docs/requirements.md を作成してください。

要件:
- 機能要件、非機能要件、未決定要件を分ける
- 初期版の受け入れ条件を、判断可能な形で書く
- 仕様が曖昧な項目は「実装タスク化禁止」と明記する
- 技術スタックや本番構成は勝手に決めない

出力:
- 00-docs/requirements.md
```

## `00-docs/roadmap.md`

### 書くこと

- 初期版でやること
- 次フェーズ以降でやること
- 当面やらないこと
- 実装より前に必要な調査や設計
- 優先順位の理由

### 書かないこと

- 詳細なスプリント計画
- 大量の実装タスク
- 期限の断定
- 仕様未確定の機能を実装予定として固定すること

### 作成プロンプト例

```text
00-docs/project_overview.md と 00-docs/requirements.md を読み、00-docs/roadmap.md を作成してください。

要件:
- 初期版、次フェーズ、当面やらないことを分ける
- 実装に入る前に必要な調査・設計を明示する
- 仕様未確定のものは「要確認」として扱う
- 具体的な期限や技術スタックを勝手に決めない

出力:
- 00-docs/roadmap.md
```

## `00-docs/decisions.md`

### 書くこと

- 決定事項
- 仮決定
- 未決定
- 決定理由
- 変更履歴
- 人間が判断したこと

### 書かないこと

- AIが勝手に決めた内容
- secrets
- 本番アクセス情報
- 実装ログの細部

### 作成プロンプト例

```text
草案メモと 00-docs/ を読み、00-docs/decisions.md の初期版を作成してください。

要件:
- 決定事項 / 仮決定 / 未決定を分ける
- 草案から判断できないものは未決定にする
- AIが推測で決めたことを決定事項にしない
- 今後、人間が追記しやすいテンプレートにする

出力:
- 00-docs/decisions.md
```

## `00-docs/status.md`

### 書くこと

- 現在地
- 直近で完了したこと
- 次に確認すること
- blocked / 未決定
- `DASHBOARD.md` との関係

### 書かないこと

- 長い作業ログ
- 実装の詳細diff
- AIとの会話履歴
- task packetに書くべき実行情報

### 作成プロンプト例

```text
00-docs/project_overview.md、requirements.md、roadmap.md、decisions.md を読み、00-docs/status.md を作成してください。

要件:
- 現在地が5分で分かるように短くする
- 次に人間が判断することを明示する
- 仕様未確定、blocked、準備不足を隠さない
- 実装ログではなく、プロジェクト状態を書く

出力:
- 00-docs/status.md
```

---

# 5. `CLAUDE.md / settings.json / rules / skills / agents` の作り方

## `CLAUDE.md`

### 役割

`CLAUDE.md` はプロジェクトの憲法です。Claude Codeセッションが最初に読む共通前提として、何が正本か、どの状態が何を意味するか、どの役割で動くかを定義します。

### 書くべきこと

- Mission
- Source of Truth
- Non-negotiables
- Queue State Machine
- Task Packet Required Files
- Done Definition
- Review Definition
- Role Detection
- Default Actions
- Output Rules

### 書かないこと

- 長い実行手順
- permissionsの詳細
- hooksのスクリプト本体
- task固有の実装手順
- 技術スタックの未承認決定

### `00-docs / rules / skills / settings.json` との違い

| 場所 | 役割 |
|---|---|
| `00-docs/` | 目的、仕様、計画、判断の正本 |
| `CLAUDE.md` | プロジェクト憲法、構造、状態、役割の定義 |
| `.claude/rules/` | 条件に応じた行動ルール |
| `.claude/skills/` | 繰り返し使う実行手順 |
| `.claude/settings.json` | 権限と安全境界の強制 |

### 作成プロンプト例

```text
00-docs/ と .ops/ の初期構成を読み、Claude Code用の root CLAUDE.md を作成してください。

要件:
- このプロジェクトの憲法として書く
- Source of Truth を明確にする
- 1 task = 1 worktree = 1 branch を明記する
- task開始前に BRIEF.md / PLAN.md / HANDOFF.md を読むことを明記する
- HANDOFF.md を再開の正本にする
- proposed → ready と review → done は人間判断と明記する
- secrets、破壊的操作、課金、本番操作は必ず停止と明記する
- 技術スタックやアプリ内容を勝手に決めない

出力:
- CLAUDE.md
```

### 人間が確認するポイント

- [ ] `00-docs/` が計画と仕様の正本になっている
- [ ] `.ops/DASHBOARD.md` が人間の中心画面になっている
- [ ] `HANDOFF.md` が最重要ファイルとして扱われている
- [ ] `proposed → ready` がAI自動になっていない
- [ ] `review → done / merge` がAI自動になっていない
- [ ] secrets、本番、課金、破壊的操作が停止条件になっている

## `settings.json`

### 役割

`.claude/settings.json` は、Claude Codeの権限と安全境界を強制する設定です。`CLAUDE.md` は読まれるだけですが、settings の permissions / hooks はより強い安全装置として働きます。

Claude Codeのsettingsスキーマは変更される可能性があります。実ファイルを作るときは、必ずその時点のClaude Code公式仕様や `/help`、`/permissions`、`/hooks` などで確認しながら作成してください。

### secretsを読ませない方針

- secretsディレクトリをdenyする
- `.env.production` など本番相当のファイルをdenyする
- AIに認証情報の中身を貼らない
- 必要な場合は「キーが必要」という事実だけを task packet に書く

### 破壊的操作を止める方針

- `rm -rf`
- `git reset --hard`
- `git push --force`
- volume削除
- DB破壊操作
- 大量削除や移動

これらは原則denyまたはaskにします。

### 本番操作を止める方針

- deploy
- production migration
- production DB接続
- production環境変数の読み取り
- 本番リソースの作成・削除

これらはAIが勝手に行わないようにします。

### worktree外書き込みを止める方針

Executorは自分のworktree内だけ編集可にします。メインrepo、他worktree、secrets、本番設定への書き込みは禁止します。

### 最初は厳しめにする理由

最初は何が安全か分からないため、広く許可するより、止まって `blocked_reason.md` を出させる方が安全です。blockedは失敗ではなく、人間が判断できる形で止まるための機能です。

### 作成プロンプト例

```text
CLAUDE.md と harness_design-D_unified.md の safety / permissions 方針を読み、.claude/settings.json の初期案を作成してください。

重要:
- Claude Codeの現在のsettingsスキーマが不明な部分は断定しない
- 実際の仕様を確認しながら作成する前提で、確認ポイントをコメントまたは別セクションに書く
- secrets、本番、課金、破壊的操作、worktree外書き込みを止める方針にする
- 最初は deny-first で厳しめにする
- 便利さより安全を優先する

出力:
- .claude/settings.json
- 必要なら .claude/settings.local.json.example
```

### 人間が確認するポイント

- [ ] secretsを読める設定になっていない
- [ ] 本番操作をallowしていない
- [ ] 破壊的操作をallowしていない
- [ ] `git push --force` が止まる
- [ ] Executorがworktree外を書けない
- [ ] schema未確認の項目を断定していない
- [ ] local個人設定と共有設定が分かれている

## `rules`

### 役割

`rules/` は、状況に応じた行動ルールを書く場所です。書き方は「条件 → 行動」にします。

### 最初に必要なrules

- `.claude/rules/safety.md`
- `.claude/rules/role-detection.md`
- `.claude/rules/autonomous-ops.md`
- `.claude/rules/docs-output.md`
- `.claude/rules/testing.md`

### 後回しでよいrules

- `.claude/rules/github-ops.md`
- `.claude/rules/release-ops.md`
- `.claude/rules/cost-management.md`
- `.claude/rules/incident-response.md`

### 作成プロンプト例

```text
CLAUDE.md と .ops の構成を読み、最初に必要な .claude/rules/ を作成してください。

作るもの:
- safety.md
- role-detection.md
- autonomous-ops.md
- docs-output.md
- testing.md

要件:
- rules は「条件 → 行動」で書く
- 仕様やプロジェクト目的は 00-docs に任せる
- 長い実行手順は skills に任せる
- proposed → ready と review → done は人間ゲートとして明記する
- 不確実な場合は blocked にして止まる方針にする

出力:
- .claude/rules/*.md
```

## `skills`

### 役割

`skills/` は、繰り返し使う複数ステップの手順をパッケージ化する場所です。毎回長いプロンプトを書かずに、triage、handoff、review-gateなどを実行できるようにします。

### 最初に必要なskills

- `triage-from-state`
- `make-task-packet`
- `task-handoff`
- `review-gate`
- `failure-classifier`
- `dashboard`

### 雛形だけでよいskills

- `impact-analyzer`
- `issue-sync`
- `auto-advance`
- `worktree-execution`

### 作り込みすぎると危険なskills

- `auto-advance`: 人間ゲートを飛ばす実装にしない
- `issue-sync`: GitHub権限や外部更新が絡むため最初は慎重にする
- `worktree-execution`: 初回は人間がコマンドを確認してから実行する
- `failure-classifier`: 仕様曖昧を自動解決しない

### 作成プロンプト例

```text
CLAUDE.md、.claude/rules/、.ops/templates/task-packet/ を読み、.claude/skills/ の初期雛形を作成してください。

作るもの:
- triage-from-state/SKILL.md
- make-task-packet/SKILL.md
- task-handoff/SKILL.md
- review-gate/SKILL.md
- failure-classifier/SKILL.md
- dashboard/SKILL.md

要件:
- 最初は雛形でよい
- 人間ゲートを飛ばさない
- secrets、本番、課金、破壊的操作が出たら blocked にする
- HANDOFF.md の更新を最重要にする
- 技術スタックやアプリ内容を勝手に決めない

出力:
- .claude/skills/*/SKILL.md
```

## `agents`

### 役割

`agents/` は、AIの役割とcontextを分けるための定義です。Triage、Executor、Reviewerを同じ文脈で動かすと、候補作成、実装、レビューの責務が混ざります。

### triage / executor / reviewer の違い

| Agent | 主な役割 | 権限方針 |
|---|---|---|
| triage | 状況整理、候補作成、`proposed` 作成 | コード編集しない。`ready` 以降へ勝手に進めない |
| executor | 1タスクをworktreeで実行 | 自分のworktree内だけ編集可 |
| reviewer | diff、VERIFY、RESULTを見て `REVIEW.md` を書く | コード編集しない。最終merge判断はしない |

### contextを分離する理由

- Triageは広く読むが、実装しない
- Executorは狭く深く作業する
- Reviewerは実装者と別視点で確認する
- 長いログや調査でmain sessionを汚さない

### 作成プロンプト例

```text
CLAUDE.md、.claude/rules/、.claude/skills/ を読み、.claude/agents/ の初期定義を作成してください。

作るもの:
- triage.md
- executor.md
- reviewer.md
- log-summarizer.md
- research.md

要件:
- triage は proposed 作成まで。ready昇格はしない
- executor は自分のworktree内だけ編集
- reviewer は REVIEW.md を書くが merge 判断は人間
- secrets、本番、課金、破壊的操作は全agentで停止

出力:
- .claude/agents/*.md
```

---

# 6. `.ops` と task packet の作り方

## `.ops` の役割

| パス | 役割 |
|---|---|
| `.ops/DASHBOARD.md` | 人間の中心画面。今すぐ判断が必要なもの、進捗、警告を見る |
| `.ops/queue/` | task packetの状態管理。`inbox / proposed / ready / running / review / blocked / done` を持つ |
| `.ops/runs/` | 実行ログの保管場所。task IDごとにrun結果を残す |
| `.ops/reports/` | triage report、daily reportなどの集約結果を置く |
| `.ops/templates/task-packet/` | task packetの雛形を置く |

## queue状態

| 状態 | 意味 | 人間ゲート |
|---|---|---|
| `inbox` | 未整理のメモ、追加要求、仕様変更 | なし |
| `proposed` | AIが候補化したタスク | `ready` 昇格は人間 |
| `ready` | 実行開始してよいタスク | 人間が選んだ後 |
| `running` | worktreeで実行中 | AIが進める |
| `review` | 実装完了、レビュー待ち | `done / merge` は人間 |
| `blocked` | 人間判断、権限、環境、仕様曖昧で停止 | 人間が見る |
| `done` | 完了 | 人間判断後 |

## task packet 必須ファイル

| ファイル | 何のためのファイルか | 主に見る / 更新する人 | いつ更新されるか |
|---|---|---|---|
| `task.yaml` | 機械可読の正本。id、状態、範囲、禁止範囲、完了条件、worktree、branch、優先度を書く | AIが更新、人間も確認 | 作成時、状態遷移時、失敗分類時 |
| `BRIEF.md` | 背景、目的、完了条件、触る範囲、触らない範囲を書く | 人間とAI | task作成時。仕様変更時 |
| `PLAN.md` | 実行計画、確認手順、テスト方針、リスクを書く | AIが更新、人間も確認 | 実行前、計画変更時 |
| `HANDOFF.md` | 再開点。現在地、次の1〜3手、環境状態、要承認事項を書く | 人間とAI。最重要 | 実行開始時、区切りごと、セッション終了前、stale救済時 |
| `VERIFY.md` | 検証結果の事実を書く | AIが更新、人間とReviewerが確認 | テスト実行後、review前 |
| `RESULT.md` | 最終結果の短いサマリーを書く | AIが更新、人間が確認 | review前、done前 |
| `RUNLOG.md` | 何を試し、どうなったかを事実で積む | AIが更新 | 実行中、失敗時 |
| `REVIEW.md` | Review Agentの判定、差分サマリー、チェック結果を書く | Reviewerが更新、人間が確認 | review状態で作成 |
| `blocked_reason.md` | blockedの理由と人間への選択肢を書く | AIが作成、人間が確認 | blocked遷移時 |
| `spec-question.md` | 仕様曖昧の質問と選択肢を書く | AIが作成、人間が回答 | spec_ambiguityでblocked時 |

任意で追加してよいファイル:

- `DECISIONS.md`: task内での判断記録
- `RISKS.md`: 未解決リスク
- `issue.md`: GitHub Issue化用文章
- `artifacts/`: スクリーンショット、テスト出力、diffなど

## task packet の最低条件

- [ ] `BRIEF.md` だけ読めば目的が分かる
- [ ] `PLAN.md` だけ読めば手順が分かる
- [ ] `HANDOFF.md` だけ読めば再開できる
- [ ] `task.yaml` に `target_paths` と `forbidden_paths` がある
- [ ] `definition_of_done` が判断可能
- [ ] secrets、本番、課金、破壊的操作が必要なら `ready` にしない

---

# 7. 最初のタスクを作ってよい条件

## task packet にしてはいけない状態

次の状態では、まだtask packetにしません。まず `00-docs` か `inbox` に置いて整理します。

- 何を作るか曖昧
- 完了条件が曖昧
- 技術選定が未決定で、それが作業範囲に直結する
- secretsが必要
- 本番環境に触る
- 課金操作が必要
- 影響範囲が広すぎる
- target paths が決まっていない
- forbidden paths が決まっていない
- 成功 / 失敗の判定方法がない
- 人間がまだ仕様判断していない

## `proposed` には入れてよい状態

`proposed` は「AIが候補化したが、人間がまだ実行承認していない」状態です。

- 調査タスク
- ドキュメント整理タスク
- 設計比較タスク
- 小さい検証タスク
- 雛形作成タスク
- 未決定事項を整理するタスク
- 実装に入らない準備タスク

`proposed` に入れる条件:

- [ ] 目的がある
- [ ] なぜ必要かが書かれている
- [ ] 実装ではなく調査・整理であれば、未決定が残っていてもよい
- [ ] `ready` にしてよいかは人間が判断する

## `ready` に昇格してよい状態

- 作業範囲が明確
- 完了条件が明確
- `target_paths` が明確
- `forbidden_paths` が明確
- 失敗しても安全に止まれる
- 1〜2時間以内で終わる見込み
- AFK safe か判断済み
- secretsを読まない
- 本番に触らない
- 課金が発生しない
- 破壊的操作がない

## 初回に向いているタスク

- ディレクトリ雛形作成
- `00-docs` 整理
- `CLAUDE.md` 作成
- task packet雛形作成
- `DASHBOARD.md` 初期化
- 小さい調査タスク
- `blocked_reason.md` や `REVIEW.md` のテンプレート整備

## 初回に向いていないタスク

- 本格実装
- 認証情報が必要な実装
- 外部サービス連携
- 課金が絡む操作
- 本番環境操作
- 大規模リファクタリング
- データ削除や移行
- 複数ディレクトリにまたがる変更
- 技術選定を伴う実装

---

# 8. タスク開始までの人間ワークフロー

## 1. 草案を概要ドキュメントにまとめる

- 人間が見るファイル: 草案メモ
- AIに頼むこと: 草案から目的、範囲、未決定を抽出
- 出力されるもの: 整理済みメモ、または `00-docs/project_overview.md` の下書き
- 次に進む条件: 草案にあること / ないことが分かれている
- やってはいけないこと: AIにアプリ内容や技術スタックを決めさせる

## 2. `00-docs` を作る

- 人間が見るファイル: `00-docs/project_overview.md`、`requirements.md`、`roadmap.md`、`decisions.md`、`status.md`
- AIに頼むこと: 草案を正本ドキュメントに分解
- 出力されるもの: `00-docs/` 一式
- 次に進む条件: 未決定事項が隠れていない
- やってはいけないこと: 仕様が曖昧なまま実装タスクにする

## 3. `.ops` を作る

- 人間が見るファイル: `.ops/DASHBOARD.md`、`.ops/queue/`
- AIに頼むこと: queue、runs、reports、templates、configの雛形作成
- 出力されるもの: `.ops/` 一式
- 次に進む条件: 7状態のqueueがある
- やってはいけないこと: `ready` タスクを勝手に作らせる

## 4. `CLAUDE.md` を作る

- 人間が見るファイル: `CLAUDE.md`
- AIに頼むこと: プロジェクト憲法を作る
- 出力されるもの: `CLAUDE.md`
- 次に進む条件: 正本、役割、状態、停止条件が分かる
- やってはいけないこと: 実装手順を長く詰め込む

## 5. `settings.json` を作る

- 人間が見るファイル: `.claude/settings.json`
- AIに頼むこと: deny-firstの初期案を作る
- 出力されるもの: `.claude/settings.json`
- 次に進む条件: secrets、本番、課金、破壊的操作が止まる
- やってはいけないこと: Claude Codeの現在仕様を確認せず断定する

## 6. `rules / skills / agents` を作る

- 人間が見るファイル: `.claude/rules/`、`.claude/skills/`、`.claude/agents/`
- AIに頼むこと: 役割判定、停止条件、handoff、triage、reviewの雛形を作る
- 出力されるもの: 初期rules、skills、agents
- 次に進む条件: triage / executor / reviewer が分かれている
- やってはいけないこと: 人間ゲートを自動化する

## 7. 初回triageをする

- 人間が見るファイル: `.ops/reports/triage/TRIAGE_REPORT_*.md`、`.ops/reports/triage/CANDIDATES_*.yaml`、`.ops/queue/proposed/`
- AIに頼むこと: 候補を `proposed` に作る
- 出力されるもの: proposed task packet
- 次に進む条件: 候補ごとのリスクとAFK可否が分かる
- やってはいけないこと: `ready` に自動昇格させる

## 8. `proposed` を確認する

- 人間が見るファイル: `.ops/DASHBOARD.md`、`.ops/queue/proposed/*/BRIEF.md`
- AIに頼むこと: 候補の比較サマリーを出す
- 出力されるもの: 人間向け比較
- 次に進む条件: 最初に回す1件を選べる
- やってはいけないこと: 複数を一気に選ぶ

## 9. `ready` を1つだけ選ぶ

- 人間が見るファイル: 選んだtask packet
- AIに頼むこと: 選んだ1件だけ `ready` に移す準備
- 出力されるもの: `.ops/queue/ready/{task}/`
- 次に進む条件: 完了条件と禁止範囲が明確
- やってはいけないこと: AFK unsafeを就寝前に回す

## 10. 最初のタスクを実行する

- 人間が見るファイル: `task.yaml`、`BRIEF.md`、`PLAN.md`、`HANDOFF.md`
- AIに頼むこと: worktree作成、branch作成、実行開始
- 出力されるもの: worktree、branch、更新されたtask packet
- 次に進む条件: `HANDOFF.md` が更新され続ける
- やってはいけないこと: main working treeで直接実装する

---

# 9. タスク開始後の人間ワークフロー

## `ready → running`

### 人間が確認すること

- [ ] `BRIEF.md` の目的が分かる
- [ ] `PLAN.md` が無理なく小さい
- [ ] `HANDOFF.md` の初期状態がある
- [ ] `task.yaml` の `target_paths` と `forbidden_paths` が明確
- [ ] secrets、本番、課金、破壊的操作がない
- [ ] AFK safe か判断済み

### Claude Codeをどこで起動するか

current modeでは、人間がセッションを起動します。基本は、AIにworktree作成コマンドを提示させ、確認してからworktreeに移動してClaude Codeを起動します。

```bash
cd {worktree_path}
claude --permission-mode plan
```

事前確認後、必要ならClaude Code内で権限を確認してから実装モードに進みます。

### 実行開始プロンプト例

```text
このworktreeで、対応する running task packet を読んで実行を開始してください。

最初に読むもの:
- task.yaml
- BRIEF.md
- PLAN.md
- HANDOFF.md

制約:
- HANDOFF.md を再開の正本にする
- target_paths の外を編集しない
- forbidden_paths に触らない
- secrets、本番、課金、破壊的操作が必要なら即 blocked にする
- 区切りごとに HANDOFF.md と RUNLOG.md を更新する
```

## running中

### 見るファイル

- `.ops/DASHBOARD.md`
- `.ops/queue/running/{task}/HANDOFF.md`
- `.ops/queue/running/{task}/RUNLOG.md`
- `.ops/queue/running/{task}/VERIFY.md`

### `RUNLOG.md` の見方

見るべき点:

- 何を試したか
- どうなったか
- 同じ失敗を繰り返していないか
- 次に何を試す予定か

`RUNLOG.md` は思考ログではなく、実行事実のログです。

### `HANDOFF.md` の見方

見るべき点:

- 現在地
- 次の1〜3手
- 要承認事項
- 環境状態
- 警告

`HANDOFF.md` が古い、曖昧、空の場合は放置しません。

### 放置してよい条件

- `HANDOFF.md` が30分以内に更新されている
- 次の1〜3手が明確
- `forbidden_paths` に触っていない
- secrets、本番、課金、破壊的操作が出ていない
- 同じ失敗を無限に繰り返していない

### 止める条件

- secretsを読もうとしている
- 本番操作が必要になった
- 課金APIや高コスト処理が必要になった
- データ削除、volume削除、破壊的git操作が必要になった
- 仕様判断が必要になった
- `HANDOFF.md` が更新されない
- `target_paths` 外の編集が必要になった

## セッションが切れた時

### どのファイルを見るか

1. `.ops/DASHBOARD.md`
2. `.ops/queue/running/{task}/HANDOFF.md`
3. `.ops/queue/running/{task}/RUNLOG.md`
4. `.ops/queue/running/{task}/task.yaml`

### どう再開するか

会話履歴ではなく `HANDOFF.md` から再開します。

```bash
cd {worktree_path}
claude --permission-mode plan
```

### 再開プロンプト例

```text
前回セッションは切れました。
会話履歴ではなく、task packetを正として再開してください。

最初に読むもの:
- HANDOFF.md
- task.yaml
- PLAN.md
- VERIFY.md
- RUNLOG.md

HANDOFF.md の「次の1〜3手」から再開してください。
HANDOFF.md が古い、矛盾している、または不十分な場合は、実装を進めずに現在状態を確認して HANDOFF.md を更新してください。
```

## review

### `REVIEW.md` の見方

見るべき点:

- 判定: approve candidate / fix before merge / blocked / reject
- 差分サマリー
- チェック項目
- テスト不足
- 破壊的変更の有無
- 仕様逸脱の有無

### `VERIFY.md` の見方

見るべき点:

- lint / typecheck / unit / integration / manual check の結果
- not-run の理由
- 残存リスク
- 完了条件に対応する検証があるか

### mergeしてよい条件

- `REVIEW.md` が approve candidate
- `VERIFY.md` の必要項目が pass、または justified skip
- `RESULT.md` が書かれている
- `HANDOFF.md` が最終状態に更新されている
- 破壊的変更がない
- secrets、本番、課金の問題がない
- 人間がdiffまたはレビューサマリーに納得している

### mergeしてはいけない条件

- `REVIEW.md` に ng がある
- `VERIFY.md` が空、または重要テストが未実施で理由がない
- 仕様逸脱がある
- `forbidden_paths` に触っている
- 本番・secrets・課金・破壊的操作が含まれる
- 変更範囲がtaskの目的を超えている

## blocked

### `blocked_reason.md` の見方

見るべき点:

- 何が止まっているか
- なぜ止まっているか
- 人間に何をしてほしいか
- 選択肢と影響
- 再開条件

### `spec-question.md` の見方

見るべき点:

- 仕様のどこが曖昧か
- 解釈A / B / C
- それぞれの影響
- 推奨ではなく、判断材料が書かれているか

### `ready` に戻す条件

- 人間が仕様を決めた
- 必要な権限変更を人間が行った
- 環境問題が解消した
- secretsや本番に触らず再開できる
- `HANDOFF.md` の次の1〜3手が明確

### 保留する条件

- 仕様判断に時間がかかる
- 本番操作が必要
- 課金判断が必要
- secretsの扱いが決まっていない
- 影響範囲が広すぎる

### 再開プロンプト例

```text
blocked_reason.md と spec-question.md を確認し、人間判断は以下の通りです。

判断:
- {人間の決定を書く}

この判断を 00-docs/decisions.md と該当task packetに反映してください。
そのうえで、再開可能なら ready に戻すための条件を確認してください。
まだ不明点がある場合は、実装せず blocked のまま追加質問を書いてください。
```

---

# 10. 朝・就寝前・問題発生時の運用

## 朝または作業開始前

### 開くファイル

1. `.ops/DASHBOARD.md`
2. `.ops/queue/blocked/*/blocked_reason.md`
3. `.ops/queue/review/*/REVIEW.md`
4. `.ops/reports/triage/TRIAGE_REPORT_*.md`

### 見る順番

1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

### 判断すること

- stale running を再開するか、blockedにするか
- blockedを解除するか、保留するか
- reviewをmergeするか、差し戻すか、保留するか
- 今日 `ready` に昇格するタスクを選ぶか
- 今から回すタスクを1つに絞るか

### 打つプロンプト例

```text
朝の状況確認をしてください。

見る順番:
1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

.ops/DASHBOARD.md を更新し、人間が判断すべき項目だけを短くまとめてください。
新規タスクを ready に昇格しないでください。
```

### やってはいけないこと

- reviewを飛ばして新規着手する
- blockedを読まずにreadyを増やす
- stale runningを放置する
- 朝の短時間で大きな仕様判断をAIに任せる

## 就寝前・放置前

### `DASHBOARD.md` の確認

- `review` が残っていないか
- `blocked` が判断待ちで残っていないか
- `running` の `HANDOFF.md` は新しいか
- `ready` にAFK safeなタスクがあるか

### reviewの処理

mergeできるものは人間が判断して処理します。reviewが3件以上ある場合は、新規着手よりreview消化を優先します。

### blockedの処理

翌朝まで放置してよいかを判断します。仕様判断や本番操作が絡むものは、就寝前に無理に進めません。

### AFK safeタスクの選び方

AFK safe の条件:

- [ ] `afk_safe: true`
- [ ] 作業時間が1〜2時間以内
- [ ] 完了条件が少ない
- [ ] secretsを読まない
- [ ] 本番に触らない
- [ ] 課金が発生しない
- [ ] 破壊的操作がない
- [ ] 失敗してもblockedで安全に止まれる

### セッション起動の判断

就寝前に起動してよいのは、AFK safeで小さいタスクだけです。迷ったら起動しません。

### 翌朝見る場所

- `.ops/DASHBOARD.md`
- `.ops/queue/review/`
- `.ops/queue/blocked/`
- `.ops/queue/running/*/HANDOFF.md`

## 問題発生時

### `DASHBOARD.md` の警告を見る

まず警告欄を見ます。個別のログから読み始めると迷いやすいため、中心画面から入ります。

### `blocked_reason.md` を見る

blockedになっている場合は、最初に `blocked_reason.md` を読みます。エラーログ全文ではなく、人間が判断できる要約を確認します。

### `HANDOFF.md` を見る

runningやstale runningの場合は、`HANDOFF.md` を読みます。会話履歴ではなく、ここを正本にします。

### 判断する選択肢

- 再開する
- blockedのまま保留する
- 仕様を決めてreadyに戻す
- scopeを小さくしてtaskを分割する
- 全runningを一旦blockedにしてclean triageする

### 再開プロンプト例

```text
問題発生後の復旧をしてください。

最初に .ops/DASHBOARD.md を読み、次に該当taskの HANDOFF.md / blocked_reason.md を読んでください。

実装はまだ進めず、以下だけ出してください:
- 何が起きているか
- 人間が選べる選択肢
- 推奨する次の一手
- その理由
- readyに戻してよいか、blockedのままにすべきか
```

---

# 11. コピペ用プロンプト集

## 草案を`00-docs`に整理する

使う場面: 草案メモを開発用ドキュメントに変換するとき  
読ませるファイル: 草案メモ、必要なら `harness_design-D_unified.md`  
期待する出力: `00-docs/project_overview.md`、`requirements.md`、`roadmap.md`、`decisions.md`、`status.md`  
注意点: 技術スタックや仕様を勝手に決めさせない

```text
草案メモを読み、アプリ開発プロジェクトをハーネス運用に載せるための 00-docs を作成してください。

作るファイル:
- 00-docs/project_overview.md
- 00-docs/requirements.md
- 00-docs/roadmap.md
- 00-docs/decisions.md
- 00-docs/status.md

ルール:
- 具体的なアプリ名や技術スタックを新しく決めない
- 決定事項 / 仮決定 / 未決定を分ける
- 初期版でやること / やらないことを分ける
- 仕様が曖昧なものは実装タスクにしない
- secrets、本番情報、認証情報は含めない
```

## `CLAUDE.md` を作る

使う場面: `00-docs` 作成後  
読ませるファイル: `00-docs/`、`.ops/` 構成、`harness_design-D_unified.md`  
期待する出力: `CLAUDE.md`  
注意点: settingsやskillsの詳細を書きすぎない

```text
00-docs と .ops 構成を読み、root CLAUDE.md を作成してください。

含めるもの:
- Mission
- Source of Truth
- Non-negotiables
- Queue State Machine
- Task Packet Required Files
- Done Definition
- Review Definition
- Role Detection
- Default Actions
- Output Rules

必ず守ること:
- 1 task = 1 worktree = 1 branch
- HANDOFF.md を再開の正本にする
- proposed → ready は人間判断
- review → done / merge は人間判断
- secrets、本番、課金、破壊的操作は即停止
```

## `settings.json` を作る

使う場面: `CLAUDE.md` 作成後  
読ませるファイル: `CLAUDE.md`、Claude Codeの現在仕様、`harness_design-D_unified.md`  
期待する出力: `.claude/settings.json`  
注意点: スキーマ不明時は断定しない

```text
CLAUDE.md とハーネス設計の safety / permissions 方針を読み、.claude/settings.json の初期案を作成してください。

条件:
- Claude Codeの現在のsettingsスキーマを確認しながら作る前提にする
- スキーマ不明な項目は断定しない
- secrets、本番、課金、破壊的操作、worktree外書き込みを止める
- 最初は deny-first で厳しめにする
- 便利さより安全を優先する
```

## `rules` を作る

使う場面: `CLAUDE.md` と `settings.json` の初期案があるとき  
読ませるファイル: `CLAUDE.md`、`.ops`、`harness_design-D_unified.md`  
期待する出力: `.claude/rules/*.md`  
注意点: rulesには「条件 → 行動」を書く

```text
CLAUDE.md と .ops 構成を読み、.claude/rules/ の初期ファイルを作成してください。

作るもの:
- safety.md
- role-detection.md
- autonomous-ops.md
- docs-output.md
- testing.md

ルール:
- 「条件 → 行動」の形式で書く
- 人間ゲートを飛ばさない
- 不確実なら blocked にする
- 仕様は 00-docs、手順は skills、強制は settings に任せる
```

## `skills` を作る

使う場面: 繰り返し手順をパッケージ化するとき  
読ませるファイル: `CLAUDE.md`、`.claude/rules/`、`.ops/templates/task-packet/`  
期待する出力: `.claude/skills/*/SKILL.md`  
注意点: 最初は雛形でよい

```text
CLAUDE.md、rules、task packet雛形を読み、.claude/skills/ の初期雛形を作成してください。

作るもの:
- triage-from-state
- make-task-packet
- task-handoff
- review-gate
- failure-classifier
- dashboard

条件:
- 最初は雛形でよい
- 人間ゲートを飛ばさない
- HANDOFF.md 更新を最重要にする
- secrets、本番、課金、破壊的操作は blocked にする
```

## `agents` を作る

使う場面: triage / executor / reviewerを分けたいとき  
読ませるファイル: `CLAUDE.md`、rules、skills  
期待する出力: `.claude/agents/*.md`  
注意点: reviewerにmergeさせない

```text
CLAUDE.md、rules、skillsを読み、.claude/agents/ の初期定義を作成してください。

作るもの:
- triage.md
- executor.md
- reviewer.md
- log-summarizer.md
- research.md

条件:
- triage は proposed 作成まで
- executor は自分のworktree内だけ編集
- reviewer は REVIEW.md を書くが merge 判断は人間
- 全agentで secrets、本番、課金、破壊的操作を停止条件にする
```

## `.ops` を作る

使う場面: 運用管理ディレクトリを初期化するとき  
読ませるファイル: `harness_design-D_unified.md`、`00-docs/status.md`  
期待する出力: `.ops/` 一式  
注意点: 実タスクを勝手にreadyにしない

```text
ハーネス運用用の .ops ディレクトリを初期化してください。

作るもの:
- .ops/DASHBOARD.md
- .ops/queue/inbox/
- .ops/queue/proposed/
- .ops/queue/ready/
- .ops/queue/running/
- .ops/queue/review/
- .ops/queue/blocked/
- .ops/queue/done/
- .ops/runs/
- .ops/reports/triage/
- .ops/reports/daily/
- .ops/config/parallel.yaml
- .ops/templates/task-packet/

条件:
- DASHBOARD.md は人間が見る中心画面にする
- queue状態の意味をREADMEまたはDASHBOARDに短く書く
- 具体タスクを ready に入れない
```

## task packet雛形を作る

使う場面: `.ops/templates/task-packet/` を整備するとき  
読ませるファイル: `CLAUDE.md`、`harness_design-D_unified.md`  
期待する出力: task packet雛形  
注意点: `HANDOFF.md` を最重要にする

```text
.ops/templates/task-packet/ に task packet 雛形を作成してください。

必須ファイル:
- task.yaml
- BRIEF.md
- PLAN.md
- HANDOFF.md
- VERIFY.md
- RESULT.md
- RUNLOG.md
- REVIEW.md
- blocked_reason.md
- spec-question.md

条件:
- task packet単体で再開できるようにする
- task.yaml に target_paths / forbidden_paths / definition_of_done を含める
- HANDOFF.md に「現在地」「次の1〜3手」「環境状態」「要承認事項」を含める
```

## 初回triageをする

使う場面: 準備ドキュメントと `.ops` が揃った後  
読ませるファイル: `00-docs/`、`CLAUDE.md`、`.ops/`  
期待する出力: triage report、candidates、`proposed`  
注意点: `ready` に昇格しない

```text
初回triageをしてください。

読むもの:
- 00-docs/
- CLAUDE.md
- .ops/DASHBOARD.md
- .ops/queue/
- .ops/templates/task-packet/

出力:
- .ops/reports/triage/TRIAGE_REPORT_{date}.md
- .ops/reports/triage/CANDIDATES_{date}.yaml
- 必要な proposed task packet

条件:
- 後処理優先の順番で確認する
- proposed までに留める
- ready 昇格は人間判断にする
- 仕様が曖昧なものは実装タスク化しない
```

## `proposed` を確認する

使う場面: AIが候補を作った後  
読ませるファイル: `.ops/queue/proposed/`、`.ops/reports/triage/`  
期待する出力: 人間向け比較表  
注意点: 候補確認と承認を混ぜない

```text
.ops/queue/proposed/ の候補を人間向けに比較してください。

出力:
- 候補一覧
- 目的
- 想定時間
- リスク
- AFK safe か
- ready にしてよい条件

注意:
- まだ ready に移動しない
- 実装を開始しない
```

## `ready` に昇格する

使う場面: 人間が1件選んだ後  
読ませるファイル: 選んだ `proposed` task packet  
期待する出力: `.ops/queue/ready/{task}/`  
注意点: 1件だけ

```text
人間判断として、次の1件だけ ready に昇格します。

対象:
- {task_id}

実行前に確認してください:
- 作業範囲が明確
- 完了条件が明確
- forbidden_paths が明確
- secrets、本番、課金、破壊的操作がない
- AFK safe 判定がある

問題なければ、この1件だけ proposed から ready に移してください。
他の候補は触らないでください。
```

## readyタスクを1つだけ実行する

使う場面: 初回実行または小さい実行  
読ませるファイル: 選んだready task packet  
期待する出力: worktree、branch、running task、更新ログ  
注意点: main working treeで実装しない

```text
ready の {task_id} を1つだけ実行してください。

条件:
- 1 task = 1 worktree = 1 branch
- main working tree で実装しない
- task.yaml / BRIEF.md / PLAN.md / HANDOFF.md を読んでから始める
- target_paths 外を編集しない
- forbidden_paths に触らない
- secrets、本番、課金、破壊的操作が必要なら即 blocked
- 区切りごとに HANDOFF.md と RUNLOG.md を更新
```

## 状況確認する

使う場面: 朝、作業中、帰宅前  
読ませるファイル: `.ops/DASHBOARD.md`、`.ops/queue/`  
期待する出力: 人間向け短い状況サマリー  
注意点: 後処理優先

```text
現在の状況を確認してください。

見る順番:
1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

.ops/DASHBOARD.md を更新し、人間が判断すべきことを短くまとめてください。
新規実装は開始しないでください。
```

## `HANDOFF` を更新させる

使う場面: セッション終了前、長時間作業中、context圧縮前  
読ませるファイル: 該当task packet、現在diff、実行ログ  
期待する出力: 更新された `HANDOFF.md`  
注意点: 次の1〜3手を必ず書かせる

```text
セッション終了または中断に備えて HANDOFF.md を更新してください。

必ず書くもの:
- 現在地
- 次の1〜3手
- 完了したこと
- 未完了のこと
- 要承認事項
- 環境状態
- 警告

会話履歴が消えても、HANDOFF.md だけで再開できる内容にしてください。
```

## reviewする

使う場面: `review/` にtaskがあるとき  
読ませるファイル: review task packet、diff、`VERIFY.md`、`RESULT.md`  
期待する出力: `REVIEW.md`  
注意点: Reviewerはmergeしない

```text
.ops/queue/review/ にある最も古いtaskをレビューしてください。

読むもの:
- task.yaml
- BRIEF.md
- PLAN.md
- VERIFY.md
- RESULT.md
- RUNLOG.md
- diff

出力:
- REVIEW.md

判定:
- approve candidate
- fix before merge
- blocked
- reject

注意:
- コード編集はしない
- merge判断は人間に残す
```

## blockedを確認する

使う場面: `blocked/` があるとき  
読ませるファイル: `blocked_reason.md`、`spec-question.md`、`HANDOFF.md`  
期待する出力: 人間向け選択肢  
注意点: 自動解除しない

```text
.ops/queue/blocked/ の内容を確認してください。

出力:
- 何が止まっているか
- なぜ止まっているか
- 人間が選べる選択肢
- readyに戻してよい条件
- 保留すべき理由があればその理由

注意:
- 仕様判断をAIだけで決めない
- secrets、本番、課金、破壊的操作が絡む場合は止めたままにする
```

## stale runningを救済する

使う場面: `HANDOFF.md` が長時間更新されていないとき  
読ませるファイル: `.ops/DASHBOARD.md`、running taskの `HANDOFF.md`、`RUNLOG.md`  
期待する出力: 再開またはblocked判断  
注意点: 新規着手より優先

```text
stale running を救済してください。

見るもの:
- .ops/DASHBOARD.md
- .ops/queue/running/*/HANDOFF.md
- .ops/queue/running/*/RUNLOG.md
- task.yaml

判定:
- HANDOFF.md の次の1〜3手が明確なら、runningのまま再開案を出す
- HANDOFF.md が古い、不明確、または危険なら blocked に移す案を出す
- 環境障害なら blocked_reason.md を作る

新規タスクには着手しないでください。
```

## 寝る前にAFK safeタスクを選ぶ

使う場面: 就寝前、仕事中に放置したいとき  
読ませるファイル: `.ops/DASHBOARD.md`、`.ops/queue/ready/`、`.ops/queue/proposed/`  
期待する出力: 起動してよい1件、または起動しない判断  
注意点: 迷ったら起動しない

```text
就寝前にAFK safeなタスクがあるか確認してください。

条件:
- afk_safe: true
- 1〜2時間以内
- secretsを読まない
- 本番に触らない
- 課金がない
- 破壊的操作がない
- 失敗してもblockedで安全に止まれる

出力:
- 起動してよいタスクが1件あるか
- 起動する場合の理由
- 起動しない場合の理由

ready昇格や実行開始は、人間確認後にしてください。
```

## 翌朝に状況確認する

使う場面: 夜間実行後  
読ませるファイル: `.ops/DASHBOARD.md`、`review/`、`blocked/`、`running/`  
期待する出力: 朝の判断リスト  
注意点: reviewとblockedを先に見る

```text
翌朝の状況確認をしてください。

見る順番:
1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

出力:
- 夜間に完了したこと
- 人間がmerge判断すべきreview
- 人間が判断すべきblocked
- stale runningの有無
- 今日readyにしてよい候補

新規実装は開始しないでください。
```

## 仕様変更を入れる

使う場面: 草案や要件が変わったとき  
読ませるファイル: 変更内容、`00-docs/`、`.ops/queue/`  
期待する出力: inbox change request、impact確認  
注意点: running taskをsilent updateしない

```text
仕様変更をハーネスに投入してください。

やること:
- .ops/queue/inbox/ に change-request を作る
- 00-docs/decisions.md に判断待ちとして記録する
- 既存 proposed / ready / running / review への影響を調査する
- 影響が大きいものは実装を進めず blocked 候補にする

注意:
- running taskを黙って書き換えない
- 実装を開始しない
```

---

# 12. 最初の1週間のモデルプラン

実装開始を急がず、最初の1週間はハーネス運用の土台を作ります。1日で終わらなくても問題ありません。品質が落ちるくらいなら、延長します。

## Day 1: 草案整理

- 目的: 草案を読み、事実、仮説、未決定を分ける
- 人間が見るファイル: 草案メモ
- AIに頼むこと: 目的、ユーザー、初期版の範囲、未決定事項の抽出
- 完了条件: AIが勝手に補完した内容が決定事項に混ざっていない
- やってはいけないこと: 技術スタックや本番構成を決める

## Day 2: `00-docs` 作成

- 目的: 草案を開発用の正本にする
- 人間が見るファイル: `00-docs/project_overview.md`、`requirements.md`、`roadmap.md`、`decisions.md`、`status.md`
- AIに頼むこと: 5ファイルの初期版作成
- 完了条件: 初期版でやること / やらないこと、決定 / 仮決定 / 未決定が分かれている
- やってはいけないこと: 仕様曖昧なものを実装タスクにする

## Day 3: `CLAUDE.md` 作成

- 目的: Claude Codeセッションの共通前提を作る
- 人間が見るファイル: `CLAUDE.md`
- AIに頼むこと: Source of Truth、Queue State、Role Detection、Default Actionsを書く
- 完了条件: `HANDOFF.md`、人間ゲート、後処理優先、安全停止条件が明記されている
- やってはいけないこと: settingsやskillsの詳細を全部詰め込む

## Day 4: settings / rules 初期作成

- 目的: 安全境界と行動ルールを作る
- 人間が見るファイル: `.claude/settings.json`、`.claude/rules/`
- AIに頼むこと: deny-first設定案、safety、role-detection、autonomous-opsを作る
- 完了条件: secrets、本番、課金、破壊的操作、worktree外書き込みが止まる方針になっている
- やってはいけないこと: Claude Codeの現在スキーマを確認せず断定する

## Day 5: `.ops / task packet` 雛形作成

- 目的: queueとtask packetをファイルで管理できるようにする
- 人間が見るファイル: `.ops/DASHBOARD.md`、`.ops/queue/`、`.ops/templates/task-packet/`
- AIに頼むこと: `.ops` 構造、`parallel.yaml`、task packet雛形を作る
- 完了条件: 7状態のqueueと必須ファイル雛形がある
- やってはいけないこと: 実装タスクをreadyに入れる

## Day 6: 初回triage

- 目的: 実装ではなく、候補を `proposed` に整理する
- 人間が見るファイル: `.ops/reports/triage/`、`.ops/queue/proposed/`、`.ops/DASHBOARD.md`
- AIに頼むこと: 後処理優先の順でスキャンし、候補を作る
- 完了条件: 候補が `proposed` にあり、AFK可否、リスク、完了条件が書かれている
- やってはいけないこと: `proposed → ready` をAIだけで進める

## Day 7: 最小の`ready`タスクを1つだけ実行

- 目的: ハーネスの1サイクルを小さく検証する
- 人間が見るファイル: 選んだtask packet、`HANDOFF.md`、`RUNLOG.md`、`VERIFY.md`
- AIに頼むこと: 1件だけworktreeで実行する
- 完了条件: 成功なら `review` へ、失敗なら `blocked_reason.md` または `spec-question.md` がある
- やってはいけないこと: 複数タスク並列、本格実装、外部連携、本番操作

## 1週間で終わらない場合の延長方針

延長する場合も、順番は崩しません。

- `00-docs` が曖昧なら Day 2 を延長する
- `settings.json` が不安なら Day 4 を延長する
- `.ops` が読みにくいなら Day 5 を延長する
- triage候補が大きすぎるなら Day 6 を延長して分割する
- 最初の `ready` に迷うなら Day 7 は実行せず、調査タスクだけにする

---

# 13. 最小チェックリスト

## タスク開始前

- [ ] `00-docs/project_overview.md` がある
- [ ] `00-docs/requirements.md` がある
- [ ] `00-docs/roadmap.md` がある
- [ ] `00-docs/decisions.md` がある
- [ ] `00-docs/status.md` がある
- [ ] `CLAUDE.md` がある
- [ ] `.claude/settings.json` の安全方針がある
- [ ] `.claude/rules/` がある
- [ ] `.claude/skills/` がある
- [ ] `.claude/agents/` がある
- [ ] `.ops/DASHBOARD.md` がある
- [ ] `.ops/queue/` の7状態がある
- [ ] `.ops/templates/task-packet/` がある
- [ ] 初回triageが `proposed` までで止まっている

## `ready` 昇格前

- [ ] 人間が選んだ1件だけである
- [ ] 作業範囲が明確
- [ ] 完了条件が明確
- [ ] forbidden paths が明確
- [ ] secretsが不要
- [ ] 本番操作が不要
- [ ] 課金が不要
- [ ] 破壊的操作が不要
- [ ] AFK safe か判断済み

## セッション終了前

- [ ] `HANDOFF.md` が更新されている
- [ ] 次の1〜3手が書かれている
- [ ] `RUNLOG.md` が更新されている
- [ ] `VERIFY.md` に事実が書かれている
- [ ] blockedなら `blocked_reason.md` がある
- [ ] 仕様曖昧なら `spec-question.md` がある
- [ ] `.ops/DASHBOARD.md` が現状と大きくズレていない

## 絶対に人間確認するもの

- [ ] `proposed → ready`
- [ ] `review → done / merge`
- [ ] secretsの扱い
- [ ] 本番操作
- [ ] 課金操作
- [ ] データ削除
- [ ] 破壊的git操作
- [ ] 技術スタックや仕様の確定
