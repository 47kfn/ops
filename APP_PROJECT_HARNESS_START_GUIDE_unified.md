# アプリ開発プロジェクト ハーネス導入スタートガイド(統合版)
## — 草案ありのプロジェクトを Design D ハーネス運用に載せるまで —

**前提設計書**: `harness_design-D_unified.md`(Design D — Unified Architecture)
**運用モード**: current mode(セッション起動は人間、起動後の自律実行はAI)

---

## このドキュメントの目的

すでにアプリの草案・概要メモがある状態から、Design D ハーネス(queue state machine / task packet / worktree / handoff)の運用に載せるまでの **準備手順・作業分割・人間の判断ポイント** を、順番に迷わず実行できる形でまとめる。

目的はいきなり実装に入ることではない。草案を開発用ドキュメントに整理し、`.ops`、`CLAUDE.md`、settings、rules / skills / agents、task packet 雛形を準備し、人間が安全に `proposed → ready → running → review → done / blocked` を判断できる状態を作ることである。

## 対象読者

- 非エンジニア寄りで、Claude Code CLI を主力に使う人(Codex 等の他エージェントを補助的に使う場合も流れは同じ)
- 一度に大量の作業や判断をしたくない人
- 就寝中・仕事中にAIエージェントを安全に回したい人
- 会話ではなくファイルを見れば再開できる運用にしたい人

## このドキュメントで扱う範囲

- 草案 → 00-docs → ハーネス設定 → 初回 triage → 最初の1タスク実行まで
- AIに依頼する作業の分割単位(どこまで一括でよく、どこから分けるか)
- タスク開始前後の人間ワークフロー
- 朝・就寝前・問題発生時・セッション切断時の定型運用

## このドキュメントで扱わない範囲

- アプリの中身の決定(機能仕様・技術スタック・本番構成は決めない)
- Design D 設計書そのものの解説(正本は `harness_design-D_unified.md`)
- secrets・認証情報・本番データをAIに読ませる運用
- 人間ゲートを飛ばして完全自動で merge する運用
- future mode(永続エージェント / Agent SDK daemon)、GitHub Actions / Projects 連携の詳細

## 大原則(常に適用)

- [ ] **current mode 前提**。セッション起動は人間。「AI自動」は起動後の自律実行を指す
- [ ] **1 task = 1 worktree = 1 branch**。main working tree では実装しない
- [ ] **task packet は自己完結**。packet を読めば何をすべきか分かる状態にする
- [ ] **HANDOFF.md が最重要**。継続の基点は会話ではなくファイル
- [ ] **proposed → ready は人間判断**。AI自動にしない
- [ ] **review → done / merge は人間判断**。AI自動にしない
- [ ] **後処理優先**(処理順は §1 参照)
- [ ] **DASHBOARD.md が人間の中心画面**。queue フォルダを直接漁らない
- [ ] **仕様が曖昧なうちは実装タスクに入らない**
- [ ] **破壊的操作・課金・secrets・本番操作は必ず人間確認で止める**

---

# 1. 全体の流れ

草案があるプロジェクトをハーネスに載せるまでの流れは以下の14ステップ。**順番に依存関係がある**ので飛ばさない。

| # | ステップ | なぜこの順番か |
|---:|---|---|
| 0 | 安全な作業場所を決める | 草案・設定・ハーネス関連ファイルを置く repo を決め、secrets や本番設定を最初から混ぜない |
| 1 | 草案を整理する | すべての下流ドキュメントの入力。ここが曖昧だと全部が曖昧になる |
| 2 | 00-docs を作る | 「計画の truth」。CLAUDE.md も triage もここを参照する |
| 3 | .ops を作る | 「実行の truth」。queue / templates の器がないと task packet を置く場所がない |
| 4 | CLAUDE.md を作る | プロジェクトの憲法。00-docs と .ops の構造が決まってから書く |
| 5 | settings.json を作る | 強制力のある安全境界。方針(CLAUDE.md)を物理的に強制する |
| 6 | rules / skills / agents を作る | 定義(CLAUDE.md)と強制(settings)の間を埋める判断・手順・役割 |
| 7 | task packet 雛形を作る | triage の出力フォーマット。triage より先に必要 |
| 7.5 | スモークテストをする | ダミータスクで queue を1周させ、配管を本物のタスクより先に確認する |
| 8 | 初回 triage をする | 00-docs と queue を読んで候補を作る。準備が揃って初めて意味を持つ |
| 9 | proposed を作る | triage の出力。AIが構造化・スコアリングする |
| 10 | 人間が ready に昇格する | **人間ゲート**。「今日何を回すか」は人間の判断 |
| 11 | worktree で実行する | 1 task = 1 worktree。人間が起動し、AIが自律実行する |
| 12 | review する | Review Agent が REVIEW.md を書く(別セッション・plan mode) |
| 13 | 人間が done / merge / blocked を判断する | **人間ゲート**。merge は最終承認 |

## ステップ0: 安全な作業場所を決める(見落としがちな準備)

- **git リポジトリの初期化と初回コミット**: 草案を含めて git 管理下に置く。worktree 運用は git が前提
- **.gitignore の整備**: `settings.local.json`、secrets 系、`.env*` を最初から除外する
- **worktree 置き場の確保**: メインrepoの**外**に `_worktrees/` を作る(例: `~/GitHub/_worktrees/{project}/`)
- **secrets の置き場所を決めて隔離**: 「AIに読ませないディレクトリ」を最初に1つ決める(例: `20-configs/secrets/`)。中身がまだ無くても deny 設定だけ先に入れる
- **`.ops/config/parallel.yaml` の初期値**: 最初は `max_concurrent_running: 1` にする

## 後処理優先の原則

セッション開始時や朝の確認では、新規タスクより先に後処理を片付ける。処理順は固定する。

1. stale running
2. blocked
3. review
4. running
5. proposed
6. ready
7. inbox

**新規着手に進んでよい条件**:

- [ ] stale running がない、または救済方針が決まっている
- [ ] blocked のうち、今すぐ人間判断できるものを処理済み
- [ ] review が溜まりすぎていない(3件以上なら review 消化を優先)
- [ ] 今日動かす ready が絞れている(初期は1つ)
- [ ] そのタスクの afk_safe 判断が済んでいる

---

# 2. 一括作成による品質低下を防ぐための作業分割

## 分割の基本ルール

- **「読む量が多い作業」と「書く量が多い作業」を同じセッションに入れない**
- **人間ゲートを挟むべき場所でセッションを切る**。AIの出力を人間が確認してから次に進む
- **性質の違う成果物(方針文書 / 強制設定 / 手順)を混ぜない**。混ぜると後半の品質が落ちる
- 1セッションの成果物は **「人間が15〜30分で確認しきれる量」** に収める
- 最初の実行は**1タスクだけ**にする

## セッションA: 草案 → 00-docs 整理

| 項目 | 内容 |
|---|---|
| 読むファイル | 草案・概要メモ(持っているもの全部) |
| 作るファイル | `00-docs/project_overview.md`, `requirements.md`, `roadmap.md`, `decisions.md`, `status.md` |
| 作らせてはいけないもの | CLAUDE.md、設定ファイル、タスク、技術スタックの決定、実装コード |
| 完了条件 | 5ファイルが揃い、**決定 / 仮決定 / 未決定** が分類され、人間が読んで「自分の構想と合っている」と確認できた |
| 分割する理由 | 00-docs は全下流の入力。他の作業を混ぜると、草案の解釈ミスが設定ファイルまで波及する |

## セッションB: .ops / queue / templates 作成

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書の §6(ディレクトリ設計)・§8(Task Packet)、`00-docs/project_overview.md`(概要把握用のみ) |
| 作るファイル | `.ops/` 一式(queue 7状態 / runs / reports / templates / config)、`.ops/DASHBOARD.md`(空テンプレート)、`.ops/config/parallel.yaml`、`.ops/templates/task-packet/` 雛形一式 |
| 作らせてはいけないもの | 実際のタスク(queue は空のまま)、CLAUDE.md、settings.json、hooks の作り込み |
| 完了条件 | ディレクトリ構造が設計書 §6-1 と一致、task-packet 雛形に必須ファイルが揃い、parallel.yaml が `max_concurrent_running: 1` |
| 分割する理由 | 機械的な構造作業で内容判断をほぼ含まない。内容理解を要する作業と混ぜると、どちらも雑になる |

## セッションB2: スモークテスト(セッションFの前に必ず挟む)

| 項目 | 内容 |
|---|---|
| 読むファイル | `.ops/` 一式、CLAUDE.md |
| 作るファイル | ダミータスク1件(例: 「`.ops/` 配下に README を1行追加する」)と、その1周分の packet 更新 |
| 作らせてはいけないもの | 本物の実装タスク、複数のダミータスク |
| 完了条件 | ダミータスクが proposed → ready → running → review → done を1周し、packet 移動・HANDOFF 更新・DASHBOARD 反映が確認できた |
| 分割する理由 | **本物のタスクで初めて queue を動かすと、ハーネスの不備とタスクの問題が混ざる**。無害なタスクで配管を先に確認する |

## セッションC: CLAUDE.md 作成

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書の §2(スコープ設計)・§7(Queue State Machine)、00-docs 全5ファイル、`.ops/` の実際の構造 |
| 作るファイル | ルート `CLAUDE.md` のみ |
| 作らせてはいけないもの | settings.json、rules、skills、00-docs の書き換え |
| 完了条件 | §5-1 の必須要素が揃い、**実在するパスだけを参照している** |
| 分割する理由 | CLAUDE.md は全セッションが毎回読む「憲法」。品質がそのまま全運用の品質になる。人間が全文を読んで確認する時間を取る |

## セッションD: settings.json 作成

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書の §10(Settings)・§14(Permissions)、CLAUDE.md、Claude Code 公式ドキュメント(settings の現行仕様) |
| 作るファイル | `.claude/settings.json`、`.claude/settings.local.json`(gitignore 対象)、`.gitignore` への追記 |
| 作らせてはいけないもの | hooks スクリプトの本格実装(最初は空 or 最小)、permissions の緩和、スキーマ未確認の断定的設定 |
| 完了条件 | secrets / 破壊的操作 / 本番系の deny が入り、**実際に Claude Code が起動してエラーが出ない**ことを確認した |
| 分割する理由 | settings.json は**唯一の強制力レイヤー**。ミス=安全装置の欠陥。スキーマはバージョンで変わりうるため、実仕様を確認しながら単独で作る |

## セッションE: rules / skills / agents 作成

量が多いので **E1 / E2 / E3 に分ける**。1回でやる場合も rules → agents → skills の順で、途中で人間が確認する。

### E1: rules

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書 §15、CLAUDE.md |
| 作るファイル | `.claude/rules/safety.md`, `role-detection.md`, `autonomous-ops.md`, `docs-output.md`, `testing.md` |
| 完了条件 | 「条件 → 行動」形式で書かれ、CLAUDE.md の定義と矛盾しない |

### E2: agents

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書 §13、CLAUDE.md、rules |
| 作るファイル | `.claude/agents/triage.md`, `executor.md`, `reviewer.md`(最初はこの3つ) |
| 完了条件 | 各 agent の権限制約(permissionMode / tools)が §14 の権限マトリクスと一致する |

### E3: skills

| 項目 | 内容 |
|---|---|
| 読むファイル | 設計書 §12、CLAUDE.md、rules、`.ops/templates/` |
| 作るファイル | `.claude/skills/triage-from-state/`, `make-task-packet/`, `task-handoff/`, `dashboard/`(実用レベル)+ `review-gate/`, `failure-classifier/`(雛形のみ) |
| 完了条件 | 各 skill の「読むもの/出力」が実在パスと一致する |

| 項目 | 内容 |
|---|---|
| 作らせてはいけないもの(E共通) | settings.json の変更、auto-advance / auto-unblock の作り込み、00-docs の変更、人間ゲートの自動化 |
| 分割する理由 | rules(判断基準)→ agents(権限)→ skills(手順)は参照関係がある。一括で作ると後半の skills が rules と整合しなくなる。3種で「書き方の型」が違うため、混ぜると型が崩れる |

## セッションF: 初回 triage

| 項目 | 内容 |
|---|---|
| 読むファイル | CLAUDE.md、00-docs 全部、`.ops/queue/`、`.ops/templates/task-packet/` |
| 作るファイル | `.ops/reports/triage/TRIAGE_REPORT_{date}.md`、`CANDIDATES_{date}.yaml`、`proposed/` への task packet(**3〜5件まで**)、DASHBOARD.md の更新 |
| 作らせてはいけないもの | ready への昇格(人間ゲート)、実装、worktree 作成、6件以上の大量候補生成 |
| 完了条件 | proposed に 3〜5件の自己完結した task packet があり、各 BRIEF.md を人間が読んで意味が分かる。仕様未決定の項目は調査タスクか spec-question になっている |
| 分割する理由 | triage は「読む量が最大」のセッション。準備作業と混ぜると context が膨張して候補の質が落ちる。初回の出力は人間が全件確認するので件数を絞る |

## セッションG: 最初の ready タスクを1つだけ実行

| 項目 | 内容 |
|---|---|
| 読むファイル | 対象 task packet 一式(task.yaml / BRIEF / PLAN / HANDOFF) |
| 作るファイル | worktree 内の成果物、VERIFY.md / HANDOFF.md / RESULT.md / RUNLOG.md の更新 |
| 作らせてはいけないもの | 他タスクへの着手、target_paths 外の編集、queue の他 packet の変更、merge |
| 完了条件 | task packet が review/ に移動し、人間が REVIEW.md と diff を確認できた。止まった場合は blocked_reason.md か spec-question.md がある |
| 分割する理由 | 初回実行は「ハーネス自体の動作確認」を兼ねる。並列にすると、問題の原因(タスクかハーネスか)を切り分けられない。**必ず1本だけ** |

## 後から分けて行うセッション(初期には作らない)

| セッション | 分ける理由 |
|---|---|
| hooks 実装 | 安全装置だが、誤実装すると作業全体を止める。運用して「ここで止めたい」が見えてから |
| secrets / 本番 / 課金の棚卸し | AIに読ませず、人間が管理方針だけを渡す |
| GitHub Issues / Projects 連携 | 外部サービス権限が絡む。実行サイクルが安定してから |

---

# 3. ドキュメント作成の順番

作る順番(上から):

1. `00-docs/project_overview.md`
2. `00-docs/requirements.md`
3. `00-docs/roadmap.md`
4. `00-docs/decisions.md`
5. `00-docs/status.md`
6. `.ops/` ディレクトリ構造(`queue/` 7状態, `runs/`, `reports/`, `config/`)
7. `.ops/DASHBOARD.md`(空テンプレート)
8. `.ops/config/parallel.yaml`
9. `.ops/templates/task-packet/`
10. `CLAUDE.md`
11. `.claude/settings.json`(+ `settings.local.json`)
12. `.claude/rules/`
13. `.claude/agents/`
14. `.claude/skills/`

## 分類

### 最初に一括作成してよいもの(機械的・内容判断が少ない)

- `.ops/` のディレクトリ構造一式(queue 7状態 / runs / reports / templates / config)
- `.ops/config/parallel.yaml`(設計書の値をほぼ流用、ただし `max_concurrent_running: 1`)
- `.gitignore` への追記

### 1つずつ丁寧に作るべきもの(品質がそのまま運用品質になる)

- `00-docs/project_overview.md`, `requirements.md`, `roadmap.md`(草案の解釈を含むため人間の確認必須)
- `CLAUDE.md`(全セッションが読む憲法)
- `.claude/settings.json`(唯一の強制力。ミス=安全装置の欠陥)
- `.claude/rules/safety.md` と `role-detection.md`

### 雛形だけでよいもの(運用が始まってから中身が決まる)

- `.ops/templates/task-packet/` の各ファイル(空雛形)
- `.ops/DASHBOARD.md`(初期状態は「タスクなし」)
- `00-docs/status.md`(最初は「phase-0: 準備中」だけ)
- `00-docs/decisions.md`(最初は記録フォーマットだけ)
- `.claude/skills/review-gate/`, `failure-classifier/`, `impact-analyzer/`(ヘッダと目的だけ)

### 実運用後に改善すればよいもの

- `.ops/hooks/` の各スクリプト(運用して「ここで止めたい/記録したい」が見えてから強化)
- `.ops/config/scoring-weights.yaml`(初回 triage は設計書のデフォルト値でよい)
- `.claude/skills/dashboard/` の出力フォーマット
- permissions の allow リスト(不便を感じた箇所だけ1つずつ追加)
- GitHub Issues / Projects 連携、hooks の高度な自動化

### 最初は作らなくてもよいもの

- `.github/`(ISSUE_TEMPLATE / PR テンプレート / Actions)
- `.mcp.json`
- `.claude/agents/log-summarizer.md`, `research.md` — 長いログや大規模調査が発生してから
- `00-docs/architecture.md` — **技術スタック未決定のうちは作らない**(「決め打ち」を誘発する)
- subdirectory CLAUDE.md — アプリのディレクトリ構造が生まれてから
- 本番デプロイ自動化、課金APIの自動実行、secrets を読む前提の設定、`/schedule` による無人定期実行、複数タスクの並列実行

---

# 4. 00-docs の作り方

## 共通の注意点(必ず守る)

- [ ] **技術スタックを勝手に決めない**。草案にない技術名は「未決定」または「候補」として書く
- [ ] **決定 / 仮決定 / 未決定 を分ける**。すべての記述にこの3分類のどれかが付く状態にする
- [ ] **草案に書かれていないことは「未決定」として扱う**。推測で補完しない
- [ ] **初期版でやること / やらないことを分ける**。スコープの境界が triage の判断材料になる
- [ ] **仕様が曖昧なものは実装タスクにしない**。requirements.md の「未決定」欄に残し、調査タスクか人間判断の対象にする
- [ ] **secrets・認証情報・本番接続情報は書かない**

## project_overview.md

| | |
|---|---|
| 書くこと | このアプリは何か(1〜3行)、誰のためか、解決する問題、初期版のゴール、初期版でやらないこと、重要な制約 |
| 書かないこと | 機能の詳細(requirements へ)、実装方法、技術スタック、スケジュール(roadmap へ) |

```text
作成プロンプト例:
添付の草案を読み、00-docs/project_overview.md を作成してください。
内容は「このアプリは何か」「誰のためか」「解決する問題」「初期版のゴール」
「初期版でやらないこと」の5節のみ。
技術スタックや実装方法は書かないでください。
草案に書かれていない内容を推測で補わないでください。
草案から読み取れないことは「未確認: 人間に確認」と明記してください。
```

## requirements.md

| | |
|---|---|
| 書くこと | 機能要求の一覧(各項目に **[決定] / [仮決定] / [未決定]** のタグ)、利用シーン・ユーザー操作、初期版スコープ内/外の区別、非機能要求(あれば)、実装前に質問が必要な項目 |
| 書かないこと | 実装方法、画面の細かいデザイン、技術選定。**草案にない要求の創作** |

```text
作成プロンプト例:
00-docs/project_overview.md と草案を読み、00-docs/requirements.md を作成してください。
各要求に [決定] [仮決定] [未決定] のいずれかのタグを付けてください。
草案から確定と読み取れないものはすべて [未決定] にしてください。
最後に「未決定一覧」をまとめ、それぞれ何を人間が決めれば確定するかを
1行ずつ書いてください。新しい機能を発明しないでください。
```

## roadmap.md

| | |
|---|---|
| 書くこと | フェーズ分割(phase-0: 準備 / phase-1: 最小動作 …)、各フェーズのゴールと完了条件、フェーズ間の依存、実装より前に必要な調査、優先順位の理由 |
| 書かないこと | 日付の確約、個別タスク(queue の仕事)、技術選定 |

```text
作成プロンプト例:
project_overview.md と requirements.md を読み、00-docs/roadmap.md を作成してください。
フェーズは3〜5個。各フェーズに「ゴール」「完了条件」「前提となる未決定事項」を書いてください。
phase-0 は「ハーネス準備と仕様確定」としてください。
[未決定] の要求に依存するフェーズには、その旨を明記してください。
日付やタスクの詳細は書かないでください。
```

## decisions.md

| | |
|---|---|
| 書くこと | 決定の記録(DEC-001 形式: 番号 / 日付 / 決定内容 / 理由 / 影響範囲)。**最初はフォーマットと、草案時点で既に決まっていることだけ** |
| 書かないこと | まだ決めていないことの先取り。AIによる勝手な決定。secrets |

```text
作成プロンプト例:
00-docs/decisions.md を作成してください。
冒頭に記録フォーマット(DEC-{番号} / 日付 / 決定 / 理由 / 影響)を定義し、
草案と project_overview.md から「人間が既に決定済みと明確に読み取れること」
だけを DEC として記録してください。判断に迷うものは記録せず、
「decisions候補(人間の確認待ち)」として末尾に列挙してください。
```

## status.md

| | |
|---|---|
| 書くこと | 現在フェーズ、直近の完了事項、進行中のこと、次の予定の4節。**最初は「phase-0: ハーネス準備中」だけでよい** |
| 書かないこと | 詳細ログ(runs/ の仕事)、タスク一覧(queue の仕事)、AIとの会話履歴 |

```text
作成プロンプト例:
00-docs/status.md を作成してください。
内容は「現在フェーズ」「完了したこと」「進行中のこと」「次にやること」の4節。
現時点では phase-0(ハーネス準備)である事実だけを書いてください。
今後この4節構造を維持して更新する旨を冒頭に注記してください。
```

---

# 5. CLAUDE.md / settings.json / rules / skills / agents の作り方

## 役割分担の全体像

| ファイル | 答える問い | 強制力 |
|---|---|---|
| CLAUDE.md | 「Xとは何か」(定義・構造)= **憲法** | なし(context) |
| 00-docs | 「このアプリは何を作るか」(仕様の正本) | なし |
| rules/ | 「Xの時どうするか」(条件→行動) | 中(条件付き注入) |
| skills/ | 「Xを実行する手順」(再利用手順) | なし(呼ばれた時だけ) |
| agents/ | 「誰がどの権限で動くか」(役割分離) | 中(permissionMode / tools) |
| settings.json | 「何が物理的に可能/不可能か」 | **強制** |

## 5-1. CLAUDE.md

**役割**: プロジェクトの憲法。全セッションが毎回読む「定義と構造の宣言場所」。強制力はない(読まれるが無視されうる)。

**書くべきこと**(設計書 §2-5 準拠):

- Mission(このrepoの目的と現在フェーズ)
- Source of Truth(計画=00-docs、実行=.ops/queue と .ops/runs、人間の画面=DASHBOARD.md)
- Non-negotiables(main working tree で実装しない / 1 task = 1 worktree / 開始前に BRIEF+PLAN+HANDOFF を読む / 終了時に VERIFY+HANDOFF を更新 / 破壊的・課金・本番は止まる / secrets は読まない)
- Queue State Machine の7状態の定義
- Task Packet の必須ファイル定義
- Done / Review の定義
- 役割判定ルール(cwd + queue + permission mode → Triage / Executor / Reviewer)
- 各役割のデフォルト行動
- 状態処理の優先順位(stale running → blocked → review → running → proposed → ready → inbox)
- Output Rules

**書かないこと**: 長いワークフロー手順(skills へ)/ 条件分岐の判断ロジック(rules へ)/ 設定値(settings.json へ)/ アプリの仕様(00-docs へ)/ タスク固有の手順(task packet へ)

```text
作成プロンプト例:
harness_design-D_unified.md の §2-5(root CLAUDE.md の内容)と §7(Queue State Machine)、
および 00-docs/ の5ファイルを読み、このプロジェクトのルート CLAUDE.md を作成してください。
含める節: Mission / Source of Truth / Non-negotiables / Queue State Machine /
Task Packet Required Files / Done Definition / Review Definition /
Role Detection / Default Actions / 状態処理優先順位 / Output Rules。
実在するパスだけを参照してください。アプリの機能仕様は書かず、00-docs への参照にしてください。
長い手順は書かず「skills/ 参照」としてください。
```

**人間が確認するポイント**:

- [ ] Non-negotiables に「secrets 禁止」「main で実装しない」「破壊的・課金・本番で止まる」が入っているか
- [ ] 参照しているパスがすべて実在するか
- [ ] proposed → ready と review → done が「人間ゲート」と明記されているか
- [ ] 自分(人間)が読んで全文理解できるか(理解できない憲法は運用できない)

## 5-2. settings.json

**役割**: 唯一の強制力レイヤー。permissions(許可/拒否の物理境界)と hooks(必ず実行されるライフサイクル処理)を持つ。CLAUDE.md が「方針」なら settings.json は「鍵」。

**方針**:

- **secrets を読ませない**: secrets ディレクトリと `.env*` への Read / Edit を deny。「読む必要が出たら人間が値を渡す」運用にする
- **破壊的操作を止める**: `rm -rf`、`git push --force`、`git reset --hard`、volume 削除系、DB破壊操作を deny(または ask)
- **本番操作を止める**: deploy / production migration / 本番接続を deny。本番系の認証情報はそもそも repo に置かない
- **worktree 外書き込みを止める**: Executor は自分の worktree 内だけ編集可。hooks(PreToolUse)で遮断する方向で設計する
- **最初は厳しめにする理由**: deny の漏れは「事故」になるが、allow の不足は「確認が1回増える」だけ。**緩める調整は安全、締める調整は手遅れになりうる**。blocked で止まるのは失敗ではなく、人間が判断できる形で止まるための機能

> **重要**: Claude Code の settings スキーマ(permissions の記法、hooks のイベント名や引数)はバージョンで変わりうる。設計書 §10 の JSON は参考形であり、**実際の仕様を公式ドキュメントと `/permissions`・`/hooks` で確認しながら作成する**。不明な項目は断定せず、作成後に必ず起動テストをする。

```text
作成プロンプト例:
harness_design-D_unified.md の §10(Settings設計)と §14(Permissions設計)を参考に、
.claude/settings.json を作成してください。
ただし設計書の JSON をそのままコピーせず、まず Claude Code の現行の settings 仕様
(permissions / hooks の正しい記法)を公式ドキュメントで確認し、それに合わせてください。
必須の deny: secrets ディレクトリの Read/Edit、.env* の Edit、rm -rf、
git push --force、docker volume 削除系、deploy/本番系コマンド。
allow は read 系 git 操作・テスト系コマンドなど最小限に。迷ったら allow に入れない。
hooks は最初は設定しない(後で追加する)でください。
作成後、確認方法(起動して /permissions で確認する手順)を提示してください。
```

**人間が確認するポイント**:

- [ ] deny に secrets / `.env*` / 破壊的 git / volume 削除 / 本番系が入っているか
- [ ] `settings.local.json` が `.gitignore` に入っているか
- [ ] 実際に `claude` を起動してエラーが出ないか
- [ ] 試しに secrets 配下のファイルを読ませようとして、拒否されるか(**deny の実地テスト**)

## 5-3. rules

**役割**: 「この状況ではこう振る舞え」の条件付きガイダンス。「条件 → 行動」形式で書く。

**最初に必要な rules(5つ)**:

| rule | 内容 |
|---|---|
| `safety.md` | 停止条件(破壊的操作・課金・secrets・本番・仕様曖昧・**3回連続同一エラー**・**target_paths 外の編集が必要になった場合** → 即 blocked)、エスカレーション基準 |
| `role-detection.md` | cwd + queue + permission mode から役割を判定するロジックと、各役割のデフォルト行動 |
| `autonomous-ops.md` | 自動進行の範囲、人間ゲートを飛ばさないこと、retry ルール、HANDOFF.md の更新頻度 |
| `docs-output.md` | RUNLOG / HANDOFF / VERIFY 等の書き方の型(事実のみ・思考ログを書かない・更新タイミング) |
| `testing.md` | 検証の最低ライン(完了条件に対応する検証を VERIFY.md に事実で残す。not-run には理由を書く) |

**後回しでよい rules**: `github-ops.md` / `release-ops.md` / `cost-management.md` / `incident-response.md`(該当する運用を始めてから)

```text
作成プロンプト例:
harness_design-D_unified.md の §15(rules設計)と §23(安全設計)、および CLAUDE.md を読み、
.claude/rules/ に safety.md / role-detection.md / autonomous-ops.md /
docs-output.md / testing.md の5つを作成してください。
すべて「条件 → 行動」の形式で書いてください。
safety.md の停止条件には、破壊的操作・課金リスク・secrets・本番操作・仕様曖昧・
3回連続同一エラー・target_paths外の編集が必要になった場合、を必ず含めてください。
CLAUDE.md と重複する「定義の説明」は書かず、CLAUDE.md への参照にしてください。
```

## 5-4. skills

**役割**: 再利用する複数ステップ手順のパッケージ。呼ばれた時だけ実行される。

**最初に作る skills(6つ)**:

| skill | 初期レベル | 内容 |
|---|---|---|
| `triage-from-state` | 実用 | 状態スキャンと候補生成(初回 triage で使う) |
| `make-task-packet` | 実用 | task packet 雛形生成(命名規則・必須ファイルの一貫性を保つ) |
| `task-handoff` | 実用 | HANDOFF.md 更新(最重要ファイルの品質を型で保証する) |
| `dashboard` | 実用 | DASHBOARD.md 生成(人間の中心画面) |
| `review-gate` | **雛形のみ** | review 移行前のチェック。中身は運用後に充実させる |
| `failure-classifier` | **雛形のみ** | 失敗の分類。**最初は「分類して報告するだけ」に留める** |

**雛形すら後でよい skills**: `impact-analyzer` / `worktree-execution` / `issue-sync`

**作り込みすぎると危険な skills**:

- `auto-advance`(自動状態遷移)— 動作実績のない段階で作り込むと、**人間ゲートを意図せず飛ばす**事故につながる。queue を手で1周させて挙動を理解してから作る
- `issue-sync` — 外部(GitHub)への書き込みを伴う。運用初期は人間が手で Issue を作る方が安全
- `failure-classifier` の auto-unblock 部分 — 自動で blocked → ready に戻す処理は、誤分類があると危険な再実行になる

```text
作成プロンプト例:
harness_design-D_unified.md の §12(Skills設計)と CLAUDE.md、.claude/rules/ を読み、
.claude/skills/ に SKILL.md を作成してください。
実用レベルで作るもの: triage-from-state / make-task-packet / task-handoff / dashboard
雛形(ヘッダと目的のみ)で作るもの: review-gate / failure-classifier
各 skill の「読むもの」「出力」はこの repo の実在パスに合わせてください。
判断基準は rules/ を参照する形にし、skill 内に独自の判断基準を書かないでください。
auto-advance と issue-sync は作らないでください(運用安定後に作ります)。
failure-classifier に auto-unblock(自動で ready に戻す処理)を入れないでください。
```

## 5-5. agents

**役割**: subagent の定義。「この役割にはこの権限とツールだけ」を宣言し、役割分離と context 保護を実現する。

**最初に作るのは triage / executor / reviewer の3つ**。`log-summarizer.md` と `research.md` は、長いログや大規模調査が発生してから追加する。

| agent | mode | 書ける場所 | やること | やらないこと |
|---|---|---|---|---|
| triage | plan | `.ops/reports/`, `.ops/queue/inbox・proposed/` | 状態スキャン、候補生成、スコアリング | コード編集、ready 以降への移動 |
| executor | acceptEdits | 自分の worktree 内 + 自 task packet | 実装、テスト、HANDOFF/VERIFY 更新 | 他 worktree・メインrepo・secrets |
| reviewer | plan | REVIEW.md のみ | diff と VERIFY を読み判定を書く | コード編集、merge の決定 |

**context を分離する理由**: triage は「広く浅く読む」、executor は「狭く深く書く」、reviewer は「疑って読む」と、必要な情報も思考の向きも違う。1つのセッションに混ぜると、(1) context が膨張して後半の品質が落ちる、(2) 実装した本人がレビューする「自己レビュー」になり甘くなる、(3) 権限を最大公約数で持つことになり安全境界が崩れる。

```text
作成プロンプト例:
harness_design-D_unified.md の §13(Subagents設計)と §14(権限マトリクス)を読み、
.claude/agents/ に triage.md / executor.md / reviewer.md を作成してください。
各 agent に役割・制約・permissionMode・利用可能ツールを書き、
§14 の権限マトリクスと矛盾しないようにしてください。
reviewer には「判定は出すが最終決定は人間」を必ず明記してください。
log-summarizer と research は作らないでください(必要になってから追加します)。
具体的なタスク指示や手順は書かないでください(それは task packet と skills の仕事です)。
```

---

# 6. .ops と task packet

## 6-1. .ops の各構成要素

| パス | 役割 | 主な読者 |
|---|---|---|
| `.ops/DASHBOARD.md` | 人間の中心画面。「今すぐ判断が必要なもの」「進捗サマリー」「警告」を5分で読める量に集約。AIが状態変化のたびに更新する | **人間** |
| `.ops/queue/` | 実行状態の正本。7ディレクトリで、task packet の物理的な置き場所=状態 | AI(人間はゲート操作のみ) |
| `.ops/runs/` | 実行ログの保管庫。task ID 単位 → run 単位で記録を残す。事後追跡用 | AIが書き、人間は問題時に読む |
| `.ops/reports/` | triage の出力置き場(TRIAGE_REPORT / CANDIDATES)と日次サマリー | 人間とAI両方 |
| `.ops/templates/task-packet/` | task packet の雛形。**雛形の品質=全タスクの品質の下限** | AI |
| `.ops/config/parallel.yaml` | 並列実行数などの運用設定。**初期値は `max_concurrent_running: 1`** | 人間が変更、AIが参照 |

## 6-2. queue の7状態

| 状態 | 意味 | 人間ゲート |
|---|---|---|
| `inbox` | 未整理のメモ、追加要求、仕様変更 | なし |
| `proposed` | AIが候補化したタスク | **ready 昇格は人間** |
| `ready` | 実行開始してよいタスク | 人間が選んだ後 |
| `running` | worktree で実行中 | AIが進める |
| `review` | 実装完了、レビュー待ち | **done / merge は人間** |
| `blocked` | 人間判断・権限・環境・仕様曖昧で停止 | 人間が見る |
| `done` | 完了 | 人間判断後 |

## 6-3. task packet の必須ファイル

| ファイル | 何のためか | 主な更新者 | いつ更新されるか |
|---|---|---|---|
| `task.yaml` | 機械可読の正本。ID / 状態 / スコープ(target_paths, forbidden_paths)/ 完了条件 / worktree情報 / 優先度 / afk_safe | AI(triage が生成、executor が状態更新) | 状態遷移のたび、run 終了のたび |
| `BRIEF.md` | 「なぜ」「何を」。背景・目的・完了条件・触る範囲・触らない範囲 | AIが下書き、**人間が ready 昇格前に確認** | タスク生成時。以後は原則変えない(変えるなら仕様変更フロー) |
| `PLAN.md` | 「どうやって」。事前確認・実装ステップ・テスト方針・リスク | AI(executor が差分更新) | 実行開始時と方針変更時 |
| `HANDOFF.md` | **最重要**。現在地・次の1〜3手・保留事項・環境状態。セッションが死んでもここから再開できる | AI | 完了条件を1つ潰すごと / 30分ごと / セッション終了時 / compact 前 |
| `VERIFY.md` | 検証結果の**事実のみ**(lint / test / manual の pass・fail) | AI | テスト実行のたび |
| `RESULT.md` | 完了時の最終サマリー。人間が「何が達成されたか」を読む | AI | 完了時のみ |
| `RUNLOG.md` | 「何を試し、どうなったか」の構造化ログ。「考えた」は書かない | AI | run のたびに追記(最新が上) |
| `REVIEW.md` | Review Agent の判定(approve / fix / blocked / reject)+ 差分サマリー + チェック表。**人間の merge 判断の材料** | AI(reviewer)が書き、**人間が読んで判断** | review 時 |
| `blocked_reason.md` | blocked の理由を非エンジニア向けに説明: 何が止まっているか(1行)/ なぜか(1〜3行)/ **人間に何をしてほしいか(選択肢)** | AI | blocked 遷移時 |
| `spec-question.md` | 仕様曖昧の場合の質問書。複数の解釈を並べ、人間がどれか選べば再開できる形にする | AIが書き、**人間が回答** | spec_ambiguity で blocked になった時 |

**人間が日常的に開くのは DASHBOARD.md → (必要に応じて) BRIEF / REVIEW / blocked_reason / spec-question の4つだけ**。それ以外はAIのための作業ファイル。

## 6-4. task packet の最低条件

- [ ] `BRIEF.md` だけ読めば目的が分かる
- [ ] `PLAN.md` だけ読めば手順が分かる
- [ ] `HANDOFF.md` だけ読めば再開できる
- [ ] `task.yaml` に `target_paths` と `forbidden_paths` がある
- [ ] `definition_of_done` が判断可能
- [ ] secrets・本番・課金・破壊的操作が必要なら ready にしない

---

# 7. 最初のタスクを作ってよい条件

## task packet にしてはいけない状態

1つでも当てはまるなら、まだタスク化しない(00-docs の「未決定」か inbox に戻すか、調査タスクに変換する):

- 何を作るか曖昧(BRIEF の「目的」が1文で書けない)
- 完了条件が曖昧(definition_of_done を客観的な文にできない)
- target_paths / forbidden_paths が決められない
- 技術選定が未決定なのに、その技術を前提とする
- 人間がまだ仕様判断していない
- secrets / 認証情報が必要
- 本番環境に触る
- 課金が発生しうる
- 失敗時に元に戻せない操作を含む
- 影響範囲が広すぎる(3ディレクトリ以上にまたがる、完了条件7個以上)

## proposed には入れてよい状態

実装でなくても、以下はタスクとして成立する:

- 調査タスク(「Xの選択肢を3つ比較して表にする」)
- ドキュメント整理タスク(「requirements.md の未決定一覧を選択肢付きの質問リストにする」)
- 設計比較タスク(「データ保存方式の候補とトレードオフをまとめる」)
- 小さい検証タスク(「雛形が design D の必須ファイル定義を満たすか検査する」)
- 調査・整理タスクであれば、未決定が残っていてもよい(ready にするかは人間が判断)

## ready に昇格してよい状態(全部 Yes が条件)

- [ ] 作業範囲が明確(target_paths が書ける)
- [ ] 完了条件が明確(definition_of_done が3〜5個で、判定可能)
- [ ] forbidden_paths が明確(secrets / 設定の正本など)
- [ ] secrets 不要・本番に触らない・課金なし・破壊的操作なし
- [ ] 1〜2時間以内で終わる見込み
- [ ] afk_safe かどうか判断済み(task.yaml に true/false が入っている)
- [ ] 依存する running タスクがない

## 初回に向いているタスク

- ディレクトリ雛形の作成・検査
- 00-docs の整理(未決定一覧の質問リスト化など)
- CLAUDE.md の整合チェック
- task packet 雛形や `blocked_reason.md` / `REVIEW.md` テンプレートの改善
- DASHBOARD.md の初期化・生成確認
- 小さい調査タスク(読み取りのみで完結するもの)

## 初回に向いていないタスク

- 本格実装(複数機能・複数ディレクトリにまたがる変更)
- 技術選定を伴う実装
- 認証情報が必要な実装
- 外部サービス連携
- 課金が絡む操作
- 本番環境操作
- データ削除や移行
- 大規模リファクタリング

---

# 8. タスク開始までの人間ワークフロー

各ステップは §2 のセッションA〜Fに対応する。**1日に全部やろうとしない**(目安は §12 の1週間プラン)。

### Step 0: 安全な作業場所を決める

§1「ステップ0」の通り。git 初期化・.gitignore・worktree 置き場・secrets 隔離を済ませる。

### Step 1: 草案を概要ドキュメントにまとめる

| | |
|---|---|
| 人間が見るファイル | 自分の草案・メモ |
| AIに頼むこと | (まだ頼まない)草案を1ファイルに集めて、自分の言葉で「決まっていること/決まっていないこと」に印を付ける |
| 次に進む条件 | 草案を読み返して「これが今の構想のすべて」と言える |
| やってはいけないこと | この段階でAIに「いい感じに補完して」と頼む(構想がAIの創作で汚染される) |

### Step 2: 00-docs を作る(セッションA)

| | |
|---|---|
| 人間が見るファイル | 出力された 00-docs 5ファイル |
| AIに頼むこと | §4 のプロンプトで草案 → 00-docs 変換 |
| 次に進む条件 | 5ファイルを**全文読んで**、草案との食い違い・勝手な補完がないと確認した |
| やってはいけないこと | 未決定をAIに決めさせる。確認せずに次へ進む |

### Step 3: .ops を作る(セッションB)

| | |
|---|---|
| 人間が見るファイル | 出来上がったディレクトリ構造(`ls -R .ops` 程度でよい) |
| AIに頼むこと | §11 のプロンプトで .ops 一式 + parallel.yaml + task packet 雛形を生成 |
| 次に進む条件 | queue の7状態ディレクトリと templates が存在し、parallel.yaml が `max_concurrent_running: 1` |
| やってはいけないこと | この時点でタスクを入れ始める |

### Step 4: CLAUDE.md を作る(セッションC)

| | |
|---|---|
| 人間が見るファイル | CLAUDE.md 全文 |
| AIに頼むこと | §5-1 のプロンプト |
| 次に進む条件 | §5-1 のチェックリストを通過し、自分が全文理解できた |
| やってはいけないこと | 理解できない記述を「AIが書いたから正しいだろう」で残す |

### Step 5: settings.json を作る(セッションD)

| | |
|---|---|
| 人間が見るファイル | settings.json、.gitignore |
| AIに頼むこと | §5-2 のプロンプト(現行仕様の確認込み) |
| 次に進む条件 | `claude` 起動でエラーなし + secrets 読み取りの deny を実地テストで確認 |
| やってはいけないこと | 不便だからと deny を外す。テストせずに次へ進む |

### Step 6: rules / agents / skills を作る(セッションE1〜E3)

| | |
|---|---|
| 人間が見るファイル | safety.md と role-detection.md は全文。他はざっと |
| AIに頼むこと | §5-3〜5-5 のプロンプトを E1→E2→E3 の順に |
| 次に進む条件 | safety.md の停止条件に破壊・課金・secrets・本番が入っている。triage / executor / reviewer が分かれている |
| やってはいけないこと | auto-advance / issue-sync / auto-unblock をこの段階で作る。人間ゲートを自動化する |

### Step 6.5: ダミータスクで queue を1周させる(セッションB2)

無害なダミータスクで proposed → done まで動かし、配管(packet 移動、HANDOFF 更新、DASHBOARD 反映)を確認する。**ここを飛ばして本物のタスクに進まない**。

### Step 7: 初回 triage をする(セッションF)

| | |
|---|---|
| 人間が見るファイル | TRIAGE_REPORT、DASHBOARD.md |
| AIに頼むこと | 「triage して」(§11 のプロンプト) |
| 次に進む条件 | proposed の各 BRIEF.md を読んで意味が分かる |
| やってはいけないこと | AIに ready 昇格までやらせる |

### Step 8: proposed を確認する

| | |
|---|---|
| 人間が見るファイル | 各 proposed の BRIEF.md と task.yaml(完了条件・target_paths・afk_safe) |
| AIに頼むこと | 不明点の説明、BRIEF の修正 |
| 次に進む条件 | §7 の「ready 昇格条件」を満たすタスクが少なくとも1つある |
| やってはいけないこと | 曖昧なまま「とりあえず実行して様子を見る」 |

### Step 9: ready を1つだけ選ぶ

| | |
|---|---|
| 人間が見るファイル | 選んだタスクの packet 一式 |
| AIに頼むこと | 「Q-xxx を ready に昇格して」(packet の移動と task.yaml 更新) |
| 次に進む条件 | ready が**1件だけ**であること |
| やってはいけないこと | 初回から複数 ready にする |

### Step 10: 最初のタスクを実行する(セッションG)

§9 の「ready → running」へ。

---

# 9. タスク開始後の人間ワークフロー

## ready → running

**人間が確認すること**:

- [ ] DASHBOARD.md に異常(stale / blocked / review 滞留)がない(後処理優先)
- [ ] BRIEF.md の目的が分かり、PLAN.md が無理なく小さく、HANDOFF.md の初期状態がある
- [ ] task.yaml の target_paths / forbidden_paths / definition_of_done を最終確認
- [ ] afk_safe 判断済みで、依存する running タスクがない

**Claude Code をどこで起動するか**: メインrepoで起動し、AIに worktree 作成と packet 移動をさせてから、提示された worktree で実行させる。worktree が既にあるなら worktree 内で起動する。

```text
実行開始プロンプト例(メインrepoで):
Q-{ID} を実行してください。
worktree を作成し、task packet を running/ に移動し、
BRIEF.md → PLAN.md → HANDOFF.md を読んでから実装を開始してください。
target_paths の外は編集せず、forbidden_paths に触らないでください。
完了条件を1つ達成するごとに HANDOFF.md を更新してください。
破壊的操作・課金・secrets・本番操作が必要になったら、即座に止まって
blocked_reason.md を書いてください。
```

## running 中

**見るファイル**: `DASHBOARD.md`(基本これだけ)→ 気になったら該当タスクの `HANDOFF.md` と `RUNLOG.md`

**RUNLOG.md の見方**: 「Action(何を試した)→ Result(どうなった)」の積み上げ。思考ログではなく実行事実のログ。**同じ Action が3回以上続いていたらループの兆候**なので止める。

**HANDOFF.md の見方**: 「現在地」と「次の1〜3手」を見る。次の手が具体的(ファイル名・操作が書ける)なら健全。「引き続き実装する」のような曖昧な記述は品質低下のサイン。

**放置してよい条件**: afk_safe: true / HANDOFF.md が更新され続けている / RUNLOG に同一エラーの繰り返しがない / 触っているのが target_paths 内だけ。

**止める条件**: secrets・課金・本番に近づいている / target_paths 外を編集しようとしている / 同一エラー3回 / HANDOFF.md が2時間以上未更新(stale)/ 仕様判断が必要になった / 自分が意図と違う方向だと感じた。

## セッションが切れた時

**慌てない。会話が飛んでも作業は飛ばない**(worktree / queue / HANDOFF.md は全部残っている)。

1. `.ops/queue/running/` で対象タスクを確認する
2. 該当タスクの `HANDOFF.md` を読む(これが source of truth。`/resume` は使ってもよいが正本にしない)
3. worktree に cd して `claude` を起動する

```text
再開プロンプト例:
Q-{ID} の続きです。
HANDOFF.md を読み、「現在地」と「次の1〜3手」を私に要約してから、
次の1手から再開してください。
HANDOFF.md と実際のファイル状態に食い違いがあれば、作業を始める前に報告してください。
HANDOFF.md が古い・不十分な場合は、実装を進めずに現在状態を確認して
HANDOFF.md を更新してください。
```

## review

**REVIEW.md の見方**(merge 判断の最小確認セット):

1. **判定欄**: approve candidate / fix before merge / blocked / reject の4択と理由
2. **差分サマリー**: 変更ファイル数と、非エンジニア向け1〜3行の説明
3. **チェック表**: 全項目が ok/ng で埋まっているか
4. VERIFY.md へのリンク

**VERIFY.md の見方**: 全項目が pass か、fail/not-run に正当な理由(justified skip)が書かれているか。**「not-run が理由なしに残っている」なら merge しない**。

**merge してよい条件**: 判定が approve candidate **かつ** チェック表が全 ok **かつ** VERIFY が全 pass(または justified skip)。

**merge してはいけない条件**: 1つでも ng がある / 判定が blocked・reject / diff に target_paths 外の変更が混ざっている / 変更範囲がタスクの目的を超えている / 自分が説明を読んで理解できない変更がある(理解できないものは保留してAIに説明させる)。

## blocked

**blocked_reason.md の見方**: 「何が止まっているか(1行)」「なぜ(1〜3行)」「**人間に何をしてほしいか(選択肢)**」の3点を読む。選択肢が書かれていなければ、まずAIに選択肢を書かせる。

**spec-question.md の見方**: 複数の解釈が並んでいるはず。どれかを選んで回答を書き込む。**選んだ内容は 00-docs/decisions.md にも DEC として記録させる**。

**ready に戻す条件**: 原因が解消した(仕様を決めた / 環境を直した / 権限を変えた)+ BRIEF/PLAN が新しい決定と整合している。

**保留する条件**: 判断材料が足りない / 急がない / 他のタスクを先に進めるべき。保留も「判断済み」として DASHBOARD に残す。

```text
blocked 再開プロンプト例:
Q-{ID} の blocked を解除します。
spec-question.md の質問には「{あなたの決定}」と回答します。
この決定を 00-docs/decisions.md に DEC として記録し、
BRIEF.md と PLAN.md を決定に合わせて更新したうえで、
task packet を ready/ に戻してください。実装はまだ始めないでください。
```

---

# 10. 朝・就寝前・問題発生時の運用

## 朝または作業開始前(目安: 10〜15分)

**開くファイルと見る順番**:

1. `.ops/DASHBOARD.md` — これだけで全体把握
2. 「今すぐ判断が必要なもの」欄 → 該当する `REVIEW.md` / `blocked_reason.md`
3. 「警告」欄(stale running の有無)

**判断すること(後処理優先の順)**:

1. stale running → 救済するか blocked に落とすか
2. blocked → 判断する(保留も判断のうち)
3. review → merge / 差し戻し / 保留
4. proposed → 今日回す本数を決めて ready 昇格(0本でもよい)

```text
プロンプト例(メインrepoで起動して):
状況を教えてください。DASHBOARD.md と queue の状態を読み、
stale running / blocked / review / running の順で、
私が今判断すべきことを優先度順に列挙してください。
新規タスクを ready に昇格しないでください。
```

**やってはいけないこと**: review を飛ばして新規着手する / DASHBOARD を見ずに queue フォルダを直接漁る / 朝の判断時間に実装の細部に潜る / 朝の短時間で大きな仕様判断をAIに任せる。

## 就寝前・放置前(目安: 15分)

1. **DASHBOARD.md の確認**: 警告ゼロか。stale や blocked を放置したまま夜間運転しない
2. **review の処理**: merge できるものは merge して running slot を空ける
3. **blocked の処理**: 「翌朝まで放置してよいか」だけ判定する(深追いしない)
4. **AFK safe タスクの選び方**: task.yaml で `afk_safe: true` のものだけ。破壊なし・課金なし・secrets 不要・2時間以内・依存なし・失敗しても blocked で安全に止まれる。**迷ったら起動しない**
5. **セッション起動の判断**: 起動するなら1〜2本まで。実行開始プロンプト(§9)で起動し、最初の数分の動き(HANDOFF 更新・正しい worktree)だけ見届ける
6. **翌朝見る場所**: DASHBOARD.md(夜間の結果はすべてここに集約されている)

```text
就寝前プロンプト例:
就寝前の確認です。ready にあるタスクのうち afk_safe: true のものを列挙し、
それぞれについて「失敗しても安全な理由」を1行ずつ説明してください。
afk_safe の判断に不安があるタスクは正直にそう書いてください。
起動はまだしないでください。
```

## 問題発生時

1. **DASHBOARD.md の警告を見る** — 何がどの状態で止まっているか(個別ログから読み始めない)
2. **blocked_reason.md を見る** — 「人間に何をしてほしいか」の選択肢
3. **HANDOFF.md を見る** — どこまで進んでいたか、環境状態(branch / compose / テスト)
4. **判断する選択肢**:
   - 原因を解消して ready に戻す
   - タスクをキャンセルして done(不要判定)に移す
   - 仕様を確定して(decisions.md に記録して)再開させる
   - scope を小さくしてタスクを分割する
   - 保留して他を進める
   - 全体が混乱している場合: **全 running を blocked に移し、clean triage からやり直す**(設計書 §26-3)

```text
問題発生時プロンプト例:
Q-{ID} が止まっています。
blocked_reason.md と HANDOFF.md と RUNLOG.md の直近部分を読み、
(1) 何が起きたか (2) 原因の仮説 (3) 私が取れる選択肢とそれぞれのリスク
を非エンジニア向けに説明してください。まだ何も実行しないでください。
```

---

# 11. コピペ用プロンプト集

> 共通の注意: どのプロンプトも **1セッション1目的**。終わったらセッションを切り替える。出力は必ず人間が確認してから次の工程へ。

### 11-1. 草案を 00-docs に整理する(セッションA)

- 使う場面: 最初の1回 / 読ませるファイル: 草案一式 / 期待する出力: 00-docs 5ファイル
- 注意点: 未決定を決めさせない。出力後に全文確認

```text
添付の草案を読み、00-docs/ に project_overview.md, requirements.md,
roadmap.md, decisions.md, status.md の5ファイルを作成してください。
ルール:
- 草案に書かれていないことを推測で補わない
- すべての要求に [決定] [仮決定] [未決定] のタグを付ける
- 技術スタック・本番構成は決めない(草案に明記がある場合のみ [決定] とする)
- 初期版でやること/やらないことを分ける
- secrets、認証情報、本番接続情報は書かない
- 各ファイルの最後に「人間に確認したいこと」を箇条書きにする
完成したら、5ファイルの要約と確認事項の一覧を提示してください。
```

### 11-2. CLAUDE.md を作る(セッションC)

→ §5-1 のプロンプト例を使用。

### 11-3. settings.json を作る(セッションD)

→ §5-2 のプロンプト例を使用。

### 11-4. rules を作る(セッションE1)

→ §5-3 のプロンプト例を使用。

### 11-5. agents を作る(セッションE2)

→ §5-5 のプロンプト例を使用。

### 11-6. skills を作る(セッションE3)

→ §5-4 のプロンプト例を使用。

### 11-7. .ops を作る(セッションB)

- 使う場面: 00-docs 完成後 / 読ませるファイル: 設計書 §6・§8 / 期待する出力: .ops 一式
- 注意点: タスクは入れない。parallel は 1

```text
harness_design-D_unified.md の §6(ディレクトリ設計)と §8(Task Packet)を読み、
.ops/ のディレクトリ構造一式を作成してください。
- queue/ に inbox, proposed, ready, running, review, blocked, done の7ディレクトリ
- runs/, reports/triage/, reports/daily/, templates/task-packet/, config/
- DASHBOARD.md は「タスクなし」の初期テンプレート
- config/parallel.yaml は設計書 §6-4 を基に、ただし max_concurrent_running: 1
- 空ディレクトリには .gitkeep を置く
タスクの中身はまだ作らないでください。完成したら構造をツリー表示してください。
```

### 11-8. task packet 雛形を作る(セッションB内)

- 使う場面: .ops 作成直後 / 読ませるファイル: 設計書 §8 / 期待する出力: templates/task-packet/ 一式
- 注意点: 雛形に具体的なアプリ内容を書かせない

```text
harness_design-D_unified.md の §8(Task Packet)を読み、
.ops/templates/task-packet/ に雛形一式を作成してください。
ファイル: task.yaml, BRIEF.md, PLAN.md, HANDOFF.md, VERIFY.md,
RESULT.md, RUNLOG.md, REVIEW.md, blocked_reason.md, spec-question.md
- task.yaml には §8-2 の必須フィールド(target_paths / forbidden_paths /
  definition_of_done / afk_safe を含む)をプレースホルダで
- HANDOFF.md の初期状態は「未開始」。「現在地」「次の1〜3手」「環境状態」
  「要承認事項」の節を持たせる
- 各ファイル冒頭に「このファイルの目的と更新タイミング」を1〜2行で注記
命名規則 Q-{project}-{4桁連番}-{slug} も雛形内に記載してください。
```

### 11-9. 初回 triage をする(セッションF)

- 使う場面: 準備完了後の最初の候補出し / 読ませるファイル: CLAUDE.md, 00-docs, queue / 期待する出力: TRIAGE_REPORT, CANDIDATES, proposed 3〜5件
- 注意点: ready 昇格はさせない

```text
triage してください。
00-docs と .ops/queue/ の状態を読み、TRIAGE_REPORT と CANDIDATES を生成し、
proposed/ に task packet を作成してください。
ルール:
- 候補は最大5件まで
- [未決定] の仕様に依存する実装タスクは作らない(代わりに調査タスクか
  spec-question 化を提案する)
- 各候補に afk_safe の判定と理由を付ける
- ready への昇格はしない(私が判断します)
最後に、各候補の「目的・完了条件・リスク」を1件3行で要約してください。
```

### 11-10. proposed を確認する

- 使う場面: triage 後、ready 昇格前 / 読ませるファイル: proposed の packet / 期待する出力: 比較説明
- 注意点: AIの推薦を鵜呑みにしない(判断は人間)

```text
proposed/ にあるタスクを1件ずつ、
「目的 / 完了条件 / 触る範囲 / 失敗時のリスク / afk_safe とその理由」
の形式で説明してください。
そのうえで「最初の1本」として適切な順に並べ、理由を付けてください。
昇格はまだしないでください。
```

### 11-11. ready に昇格する

- 使う場面: 人間が1本選んだ後 / 期待する出力: packet の移動と task.yaml 更新
- 注意点: 昇格は人間の決定。複数同時に昇格させない(初期)

```text
Q-{ID} を ready に昇格すると決めました。
task packet を proposed/ から ready/ に移動し、task.yaml の status を更新してください。
他のタスクは動かさないでください。
移動後、このタスクの完了条件と forbidden_paths を再表示してください。
```

### 11-12. ready タスクを1つだけ実行する(セッションG)

→ §9「ready → running」のプロンプト例を使用。

### 11-13. 状況確認する

- 使う場面: いつでも / 読ませるファイル: DASHBOARD, queue / 期待する出力: 優先度順の判断リスト
- 注意点: このセッションで実装させない

```text
状況を教えてください。
DASHBOARD.md と queue の各状態を読み、
stale running → blocked → review → running → proposed → ready → inbox の優先順で、
私が判断すべきこと・放置してよいことを分けて報告してください。
```

### 11-14. HANDOFF を更新させる

- 使う場面: 作業の区切り / セッションを切る前 / compact 前 / 期待する出力: 更新された HANDOFF.md
- 注意点: 「次の1〜3手」が具体的かを確認する

```text
ここで一区切りにします。HANDOFF.md を更新してください。
- 現在地(何がどこまで終わったか、事実のみ)
- 次の1〜3手(ファイル名と操作が分かる具体度で)
- 保留事項と要承認事項
- 環境状態(branch / worktree / テスト状態)
更新後、HANDOFF.md の内容をそのまま表示してください。
```

### 11-15. review する

- 使う場面: review/ にタスクがある時(**実装とは別セッション、plan mode**)/ 期待する出力: REVIEW.md
- 注意点: AIの判定は材料。merge の決定は人間

```text
review/ にあるタスクをレビューしてください。
diff、PLAN.md、VERIFY.md、RUNLOG.md を読み、REVIEW.md を書いてください。
含めるもの: 差分サマリー(非エンジニア向け1〜3行の説明付き)、
チェック項目表(全項目 ok/ng)、判定(approve/fix/blocked/reject)と理由。
コードの編集はしないでください。最終判断は私が行います。
```

### 11-16. blocked を確認する

- 使う場面: blocked 発生時 / 期待する出力: 選択肢付きの説明
- 注意点: このセッションで解決まで進めさせない

```text
blocked/ にあるタスクを1件ずつ、blocked_reason.md(あれば spec-question.md)を基に
「何が止まっているか / なぜか / 私の選択肢と各選択肢の影響」を説明してください。
即決できるものと、調べてから決めるべきものを分けてください。
解除作業はまだしないでください。
```

### 11-17. stale running を救済する

- 使う場面: HANDOFF.md が2時間以上未更新の running がある時 / 期待する出力: 救済判定と処理
- 注意点: 機械的に blocked に落とさず、HANDOFF の中身で判断させる

```text
running/ の Q-{ID} が stale です。
HANDOFF.md と RUNLOG.md と worktree の実際の状態を確認し、
a) HANDOFF に明確な次の手があり再開可能 → そのまま継続再開
b) HANDOFF が古い・不明確 → blocked に移動して blocked_reason.md を作成
c) 環境障害が原因 → blocked に移動し、infra-fix 候補を proposed に提案
のどれに該当するか判定し、判定理由を示してから処理してください。
```

### 11-18. 寝る前に AFK safe タスクを選ぶ

→ §10「就寝前」のプロンプト例を使用。選んだ後の起動は §9 の実行開始プロンプト。

### 11-19. 翌朝に状況確認する

- 使う場面: 夜間運転の翌朝 / 期待する出力: 夜間の結果サマリー

```text
おはようございます。夜間の結果を確認します。
DASHBOARD.md と runs/ の直近ログを読み、
(1) 夜間に完了したこと (2) 止まったこととその理由 (3) 今朝私が判断すべきこと
を後処理優先の順(stale → blocked → review)で報告してください。
新規タスクの提案はその後にしてください。
```

### 11-20. 仕様変更を投入する

- 使う場面: 構想が変わった時 / 注意点: 走っているタスクを直接書き換えさせない

```text
仕様変更があります: {変更内容を1〜3行で}
これを .ops/queue/inbox/ に change-request として投入し、
既存の ready / running / review への影響を分析して impact.md を作成してください。
影響のある task は continue / suspend / discard_and_replan に分類してください。
00-docs の更新案と decisions.md への記録案も提示してください。
私の承認なしに、既存タスクの移動や 00-docs の書き換えはしないでください。
```

### 11-21. done 後の片付けをする

- 使う場面: merge 判断の直後 / 注意点: worktree / branch の削除は確認してから

```text
Q-{ID} の merge を承認します。
merge 手順を提示し、私の確認後に実行してください。
merge 完了後: task packet を done/ へ移動、worktree の削除と branch の削除を
実行前に対象パスを表示して確認を取ってから行い、
DASHBOARD.md と 00-docs/status.md を更新してください。
```

---

# 12. 最初の1週間のモデルプラン

> 1日あたりの人間の作業時間は 30〜60分 を想定。**実装を急がない**。この1週間の成果物は「アプリのコード」ではなく「安全に回る運用」である。

### Day 1: 草案整理(人間中心の日)

| | |
|---|---|
| 目的 | 構想の正本を1ファイルに固める |
| 人間が見るファイル | 自分の草案・メモ全部 |
| AIに頼むこと | 原則なし(頼むとしても「メモの重複を整理して」程度) |
| 完了条件 | 草案が1ファイルにまとまり、「決まっている/いない」の印が付いている |
| やってはいけないこと | AIに構想を補完させる。この日に 00-docs まで進める |

### Day 2: 00-docs 作成(セッションA)

| | |
|---|---|
| 目的 | 計画の truth を確定する |
| 人間が見るファイル | 出力された 00-docs 5ファイル(全文) |
| AIに頼むこと | プロンプト 11-1 |
| 完了条件 | 5ファイル確認済み。未決定一覧が手元にある |
| やってはいけないこと | 未決定をその場のノリで決定扱いにする |

### Day 3: .ops 作成(セッションB)+ CLAUDE.md 作成(セッションC)

| | |
|---|---|
| 目的 | 実行の器と憲法を作る |
| 人間が見るファイル | .ops のツリー、CLAUDE.md 全文 |
| AIに頼むこと | プロンプト 11-7 → 11-8 → 別セッションで 11-2 |
| 完了条件 | .ops 構造が設計書と一致、CLAUDE.md のチェックリスト通過 |
| やってはいけないこと | 両方を1セッションでやる(構造作業と憲法執筆は分ける) |

> ※ B(機械的)が軽いので C と同日に置くが、**セッションは必ず分ける**。

### Day 4: settings / rules 初期作成(セッションD → E1)

| | |
|---|---|
| 目的 | 強制力のある安全境界と行動ルールを張る |
| 人間が見るファイル | settings.json、safety.md |
| AIに頼むこと | プロンプト 11-3 → 別セッションで 11-4 |
| 完了条件 | 起動テスト通過、secrets deny の実地テスト通過 |
| やってはいけないこと | deny テストを省略する。便利さのために allow を増やす。スキーマ未確認の項目を断定する |

### Day 5: agents / skills 作成 + スモークテスト(セッションE2 → E3 → B2)

| | |
|---|---|
| 目的 | 役割と手順を整え、ダミータスクで配管を確認する |
| 人間が見るファイル | agents 3ファイル、ダミータスクの動き(DASHBOARD) |
| AIに頼むこと | プロンプト 11-5 → 11-6 → ダミータスク1周 |
| 完了条件 | ダミータスクが proposed → done まで1周し、HANDOFF / DASHBOARD が更新された |
| やってはいけないこと | スモークテストを飛ばして本物のタスクに進む |

### Day 6: 初回 triage(セッションF)

| | |
|---|---|
| 目的 | 本物の候補を作り、人間が中身を確認する |
| 人間が見るファイル | TRIAGE_REPORT、proposed 各件の BRIEF.md |
| AIに頼むこと | プロンプト 11-9 → 11-10 |
| 完了条件 | proposed 3〜5件を全件確認し、ready 候補を1本決めた |
| やってはいけないこと | 候補を確認せずに昇格する。実装まで進める |

### Day 7: 最小の ready タスクを1つだけ実行(セッションG)

| | |
|---|---|
| 目的 | 本番の運用サイクル(ready → running → review → 人間判断)を初めて完走する |
| 人間が見るファイル | DASHBOARD.md → REVIEW.md → diff |
| AIに頼むこと | プロンプト 11-11 → 11-12 →(別セッションで)11-15 |
| 完了条件 | review まで到達し、人間が merge / 差し戻し / 保留のいずれかを**判断した**(merge できなくてもサイクル完走が成果) |
| やってはいけないこと | 並列で2本目を始める。review を自分で読まずに merge する |

## 1週間で終わらない場合の延長方針

- **順番を守ったまま日数を伸ばす**。工程を飛ばして縮めない
- 詰まりやすいのは Day 2(未決定が多い)と Day 4(settings の実仕様確認)。ここは2日かけてよい
- 未決定が多すぎて triage で実装候補がゼロになる場合、それは失敗ではない。**「未決定を潰す調査タスク」を最初の数本にする**のが正しい進行
- 延長中も Day 5 のスモークテストだけは省略しない(飛ばすと、最初の本番タスクで配管の不具合とタスクの問題が混ざる)
- 2週目以降: 1日1〜2タスクのサイクルを2〜3日回して安定したら、`parallel.yaml` の `max_concurrent_running` を 2 に上げる → さらに安定したら 3。**並列度を上げるのは「review が滞留していない」ことが条件**

---

# 13. 最小チェックリスト

## タスク開始前

- [ ] `00-docs/` の5ファイル(project_overview / requirements / roadmap / decisions / status)がある
- [ ] `CLAUDE.md` がある
- [ ] `.claude/settings.json` の安全方針があり、起動テスト・deny 実地テストを通過した
- [ ] `.claude/rules/` `.claude/skills/` `.claude/agents/` がある
- [ ] `.ops/DASHBOARD.md` と `.ops/queue/` の7状態がある
- [ ] `.ops/templates/task-packet/` がある
- [ ] `parallel.yaml` が `max_concurrent_running: 1`
- [ ] ダミータスクで queue を1周させた(スモークテスト済み)
- [ ] 初回 triage が `proposed` までで止まっている

## ready 昇格前

- [ ] 人間が選んだ1件だけである
- [ ] 作業範囲(target_paths)が明確
- [ ] 完了条件が明確で判定可能
- [ ] forbidden_paths が明確
- [ ] secrets 不要・本番操作不要・課金不要・破壊的操作不要
- [ ] afk_safe か判断済み
- [ ] 依存する running タスクがない

## セッション終了前

- [ ] `HANDOFF.md` が更新され、次の1〜3手が具体的に書かれている
- [ ] `RUNLOG.md` が更新されている
- [ ] `VERIFY.md` に事実が書かれている
- [ ] blocked なら `blocked_reason.md` がある
- [ ] 仕様曖昧なら `spec-question.md` がある
- [ ] `DASHBOARD.md` が現状と大きくズレていない

## 絶対に人間確認するもの

- [ ] `proposed → ready`
- [ ] `review → done / merge`
- [ ] secrets の扱い
- [ ] 本番操作
- [ ] 課金操作
- [ ] データ削除・破壊的 git 操作
- [ ] worktree / branch の削除
- [ ] 技術スタックや仕様の確定

---

# 付録: 困った時の早見表

| 症状 | 見るファイル | 取る行動 |
|---|---|---|
| 何が起きてるか分からない | DASHBOARD.md | プロンプト 11-13 |
| セッションが切れた | running/ の HANDOFF.md | §9「セッションが切れた時」 |
| タスクが止まった | blocked_reason.md / spec-question.md | プロンプト 11-16 |
| HANDOFF が2時間以上未更新 | HANDOFF.md + RUNLOG.md | プロンプト 11-17 |
| AIが同じ失敗を繰り返す | RUNLOG.md | セッションを止め、blocked に落として人間が原因を読む |
| review が溜まった(3件以上) | DASHBOARD.md | 新規着手を止めて review 消化に集中 |
| 全体が混乱した | queue 全体 | 全 running を blocked へ → status.md を人間が更新 → clean triage |
| AIが仕様を勝手に決めようとする | spec-question.md | 実装を止めて人間が決定 → decisions.md に記録 |

---

*このガイドの正本となる設計思想は `harness_design-D_unified.md` を参照。本ガイドと設計書が食い違う場合は設計書を優先し、本ガイドを更新すること。*
