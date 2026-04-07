# `harness_design-D_unified.md` レビュー

- 対象: `harness_design-D_unified.md`
- 作成日: 2026-04-08
- レビュー方針: 指定URLを実取得し、取得できた本文だけを根拠に、批判よりも「追加すべき設定」「追記すべき設定」「代替案」を優先順位順で整理する
- 重要な前提: OpenAI / Medium / 一部外部サイトは anti-bot / 404 / DNS の影響で本文未取得。レビュー本文は主に本文取得できた URL と、検索結果ページから辿れた補助記事を根拠にしている

## 1. 結論

`harness_design-D_unified.md` は、今回取得できたソース群の中でもかなり強い部類の設計です。特に以下は整合性が高いです。

- 会話ではなく `HANDOFF.md` と task packet を正本にしている点
- `1 task = 1 worktree = 1 branch` で物理隔離している点
- `permissions` / `hooks` / `skills` / `subagents` を、単なるプロンプトではなく実行環境として扱っている点
- 人間の最終意思決定を `ready` 昇格、`merge`、破壊的操作に限定している点
- failure を分類し、queue と runs に落として再利用可能な形で残す点

一方で、取得できた記事を横断すると、この設計をさらに強くするには次の4点を最優先で補うのが有効です。

1. `behavior harness` を明示し、構造品質とユーザー価値の検証を分離すること
2. repo / area ごとの `harnessability` を測り、適用対象を選別すること
3. Review ループに `severity / directive / oscillation detection / root-cause validation` を入れること
4. ローカルハーネスと GitHub / CI の責務境界を明示し、「bashで再実装するだけ」の領域を絞ること

## 2. URL取得結果

### 2-1. 指定URL一覧の取得結果

| No. | タイトル | 状態 | 備考 | URL |
|---|---|---|---|---|
| 1 | Harness engineering: leveraging Codex in an agent-first world | 部分取得 | Cloudflare の challenge ページのみ取得。本文抽出不可。 | https://openai.com/research/harness-engineering-codex |
| 2 | Harness Engineering | 取得成功 | 本文取得可。主要根拠として使用。 | https://martinfowler.com/articles/harness-engineering.html |
| 3 | Harness design for long-running application development | 取得失敗 | 指定URLは 404 Not Found。 | https://www.anthropic.com/engineering/harness-design-long-running-agents |
| 4 | Effective harnesses for long-running agents | 取得失敗 | 指定URLは 404 Not Found。 | https://www.anthropic.com/engineering/effective-harnesses |
| 5 | My AI Adoption Journey | 取得失敗 | 本文ではなくサイトトップ相当のみ。記事本文取得不可。 | https://mitchellh.com/writing/ai-adoption-journey |
| 6 | The Emerging "Harness Engineering" Playbook | 取得失敗 | DNS 解決失敗。 | https://www.artificialignorance.ai/p/harness-engineering-playbook |
| 7 | Improving Deep Agents with harness engineering | 取得失敗 | 404 ページのみ取得。 | https://blog.langchain.dev/improving-deep-agents-harness-engineering |
| 8 | Harness Engineering: The Missing Layer Behind AI Agents | 取得成功 | 本文取得可。主要根拠として使用。 | https://www.louisbouchard.ai/harness-engineering |
| 9 | The Rise of AI Harness Engineering | 部分取得 | Medium の challenge ページのみ取得。 | https://medium.com/@cobusgreyling/the-rise-of-ai-harness-engineering |
| 10 | Skill Issue: Harness Engineering for Coding Agents | 取得失敗 | 本文ではなくサイトトップ/404。記事本文取得不可。 | https://www.humanlayer.dev/blog/harness-engineering |
| 11 | The importance of Agent Harness in 2026 | 取得失敗 | 本文ではなくサイト共通ページのみ。記事本文取得不可。 | https://www.philschmid.de/agent-harness |
| 12 | Harness Engineering vs Context Engineering | 部分取得 | Medium の challenge ページのみ取得。 | https://medium.com/@rickhightower/harness-vs-context-engineering |
| 13 | What Is Harness Engineering? Complete Guide | 取得失敗 | 404 ページのみ取得。 | https://nxcode.io/blog/harness-engineering-guide |
| 14 | How I think about Codex | 取得失敗 | 404 ページのみ取得。 | https://simonwillison.net/2026/Feb/22/codex/ |
| 15 | Beyond the AI Coding Hangover | 部分取得 | Medium の challenge ページのみ取得。 | https://medium.com/spillwave/ai-coding-hangover |
| 16 | Humans and Agents in Software Engineering Loops | 取得失敗 | 404 ページのみ取得。 | https://martinfowler.com/articles/humans-agents-loops.html |
| 17 | Context Engineering for Coding Agents | 取得失敗 | 404 ページのみ取得。 | https://martinfowler.com/articles/context-engineering.html |
| 18 | Unlocking the Codex harness: how we built the App Server | 部分取得 | Cloudflare の challenge ページのみ取得。 | https://openai.com/blog/codex-harness-app-server |
| 19 | Why Harness Engineering Replaced Prompting in 2026 | 取得失敗 | 404 ページのみ取得。 | https://epsilla.com/blog/harness-engineering |
| 20 | ハーネスエンジニアリング入門 ── CLAUDE.mdの次に来るAI駆動開発の基本 | 部分取得 | Qiita 検索結果ページは取得できたが個別本文ではない。 | https://qiita.com/search?q=ハーネスエンジニアリング |
| 21 | 「人間はコードを1行も書かない」という縛りで5ヶ月間プロダクト開発した話 | 部分取得 | Qiita 検索結果ページは取得できたが個別本文ではない。 | https://qiita.com/search?q=ハーネス+コード書かない |
| 22 | Claude Code / Codex ユーザーのためのHarness Engineeringベストプラクティス | 部分取得 | Zenn 検索結果ページのみ。個別本文なし。 | https://zenn.dev/search?q=Harness%20Engineering |
| 23 | ハーネスエンジニアリング × ローカルオーケストレーターでAIエージェントを飼い慣らす | 取得失敗 | Zenn topic 404。 | https://zenn.dev/topics/harness-engineering |
| 24 | OpenAIが実践するAgent-First時代の開発アプローチ | 部分取得 | Zenn 検索結果ページのみ。個別本文なし。 | https://zenn.dev/search?q=OpenAI%20Harness |
| 25 | OpenAIの提唱する「ハーネス・エンジニアリング」とは何か | 部分取得 | ブログトップは取得できたが対象記事本文は直取得できず。 | https://blog.est.co.jp/ |
| 26 | OpenAIが実践する「Harness Engineering」という新しい開発手法 | 部分取得 | note 検索結果ページのみ。個別本文なし。 | https://note.com/search?q=Harness%20Engineering |
| 27 | ハーネスエンジニアリング（Harness Engineering） | 取得失敗 | はてなブログ検索 URL は 404。 | https://hatena.blog/search?q=ハーネスエンジニアリング |
| 28 | ハーネスエンジニアリング - Martin Fowlerの批評から再考する | 部分取得 | Qiita 検索結果ページのみ。個別本文ではない。 | https://qiita.com/search?q=Martin%20Fowler%20ハーネスエンジニアリング |
| 29 | Claude Codeのハーネスをより深く理解するための人間向けガイド | 部分取得 | note 検索結果ページのみ。個別本文なし。 | https://note.com/search?q=Claude%20Code%20ハーネス |
| 30 | 5カ月でコード100万行を生成してソフトウェア構築 | 取得失敗 | atmarkIT 検索URLは 404 かつ本文化不能。 | https://atmarkit.itmedia.co.jp/search?q=ハーネスエンジニアリング |
| 31 | ハーネスエンジニアリングで人間のコードレビューをやめる | 取得成功 | 本文取得可。主要根拠として使用。 | https://zenn.dev/theaktky/articles/1c6c3b9333117c |
| 32 | ハーネスエンジニアリング、それGit Workflowをbashで書き直してるだけでは | 取得成功 | 本文取得可。主要根拠として使用。 | https://zenn.dev/shio_shoppaize/articles/shogun-harness-engineering |
| 33 | コンテキスト・ハーネスエンジニアリングの現在 | 部分取得 | Speaker Deck 検索結果ページのみ。 | https://speakerdeck.com/search?q=harness+engineering |
| 34 | 実践ハーネスエンジニアリング #MOSHTech | 部分取得 | Speaker Deck 検索結果ページのみ。 | https://speakerdeck.com/search?q=MOSH+Harness |
| 35 | ハーネスエンジニアリング | 部分取得 | Speaker Deck 検索結果ページのみ。 | https://speakerdeck.com/search?q=harness+engineering |

### 2-2. 補助取得した直接記事

以下は、指定URLのうち検索結果・一覧ページからリンクを辿って追加取得した補助ソースです。レビュー本文では補助根拠として使っています。

| 出所 | 状態 | 補足 | URL |
|---|---|---|---|
| #20 / #21 の Qiita 検索結果 | 取得成功 | `CLAUDE.md` だけでは足りず、rules / skills / hooks / memory / feedback を束ねる必要がある、という整理を確認 | https://qiita.com/nogataka/items/d1b3fcf355c630cd7fc8 |
| #21 / #28 の Qiita 検索結果 | 取得成功 | internal quality と behavior のズレ、ハーネス過剰構築、legacy との相性を整理した記事として確認 | https://qiita.com/Aochan0604/items/306bde3e138ce071f7b2 |
| #34 の Speaker Deck 検索結果 | 取得成功 | スライド本文相当のテキストを取得し、agent 分離・依存方向制約・ブラウザ検証の観点を確認 | https://speakerdeck.com/kajitack/implementing-herness-engineering |

## 3. 取得できた記事の要点

### 3-1. 一次・重要寄りの要点

- [Martin Fowler: Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)
  - ハーネスは「モデル以外の全部」と広げ過ぎず、コーディングエージェント利用文脈で外側の制御系として捉えるべき
  - `feedforward` と `feedback` の両方が必要
  - `computational` な検査と `inferential` な検査は分けて考えるべき
  - `keep quality left`、つまり後段レビューより前段の制約・検証を厚くする設計が重要
  - regulation の観点は少なくとも `maintainability`、`architecture fitness`、`behaviour` に分けた方がよい
  - repo 側の性質として `harnessability` が重要。どのコードベースでも同じ強度で適用できるわけではない
  - 人間の役割は消えず、どこを人間判断に残すかを設計する必要がある

- [Louis Bouchard: Harness Engineering: The Missing Layer Behind AI Agents](https://www.louisbouchard.ai/harness-engineering)
  - prompt engineering は「何を聞くか」、context engineering は「何を渡すか」、harness engineering は「全体をどう動かすか」の話
  - harness は tools / permissions / state / tests / logs / retries / checkpoints / guardrails / reviews / evals を含む
  - context window が広くなっても、ハーネスなしで agent が安定するわけではない

### 3-2. 日本語の実践記事から取れた要点

- [Qiita: ハーネスエンジニアリング入門 ── CLAUDE.mdの次に来るAIエージェント制御パラダイム](https://qiita.com/nogataka/items/d1b3fcf355c630cd7fc8)
  - `CLAUDE.md` 単体は「お願い」に留まりやすく、強制力・違反検知・更新反映の仕組みが必要
  - ハーネスは rules / skills / hooks / memory / feedback を束ねる層として整理されている
  - session 断絶対策として progress / handoff の外部化が重要

- [Qiita: ハーネスエンジニアリングは本当に「新しい」のか？](https://qiita.com/Aochan0604/items/306bde3e138ce071f7b2)
  - ハーネスの多くは既存の docs / static analysis / CI / DevOps と重なる
  - そのため、internal quality の改善と user-facing behavior の保証を混同しないことが重要
  - ハーネスはそれ自体がソフトウェア化しやすく、導入コストや legacy codebase との相性を見積もる必要がある
  - 作り込み過多は逆効果になり得るため、`minimum viable harness` と `rippable` な姿勢が必要

- [Zenn: ハーネスエンジニアリングで人間のコードレビューをやめる](https://zenn.dev/theaktky/articles/1c6c3b9333117c)
  - review ループは単純に AI に投げるだけでは不安定で、`severity triage`、`oscillation detection`、`directive 固定`、`root-cause validation` が効く
  - 判断基準はプロンプト埋め込みより、プロジェクト内ドキュメント参照に寄せた方が運用しやすい

- [Zenn: ハーネスエンジニアリング、それGit Workflowをbashで書き直してるだけでは](https://zenn.dev/shio_shoppaize/articles/shogun-harness-engineering)
  - ローカルハーネスの一部は、GitHub / PR / CI / branch protection のローカル再配置に過ぎない
  - 本当に agent 固有の設計が要る領域と、既存の開発プロセスで十分な領域を分けて設計した方がよい

- [Speaker Deck: 実践ハーネスエンジニアリング #MOSHTech](https://speakerdeck.com/kajitack/implementing-herness-engineering)
  - コンテキストは散らしたままにせず、GitHub など agent が再利用しやすい場所に集約する
  - agent は実装・品質・性能・仕様・Issue 作成で分け、適切なコンテキスト量を維持する
  - 依存方向制約、型、静的解析、テスト、人間レビューを多層防御として扱う
  - CI 緑だけで完了扱いにせず、ブラウザ操作やスクリーンショットなど user-facing behavior の確認を足す

### 3-3. 検索結果・一覧ページから補助的に見えた傾向

- Qiita / note / Zenn 検索ページでは、`CLAUDE.md の限界`、`context engineering との差分`、`技術的負債制御`、`レビュー自動化` が繰り返し扱われていた
- つまり国内の実践知も、単一プロンプト強化ではなく「構造・状態・評価」の設計に論点が集まっている

## 4. 設計書レビュー

### 4-1. 全体評価

この設計書は、取得できたソース群と照らすと「外部状態」「隔離」「再開性」「権限」「再分類」に強いです。特に、Fowler の `feedforward / feedback` と Louis Bouchard の `tools / permissions / state / tests / logs` という整理にはかなり近いです。

ただし、現状の強みは主に以下へ寄っています。

- maintainability harness
- architecture fitness harness
- operational recovery harness

逆に、今回の取得ソースが一貫して強調していた以下は、まだ設計書では「明文化が薄い」か「運用で吸収する前提」に寄っています。

- behaviour harness
- harnessability の事前判定
- review ループの収束制御
- ローカルハーネスと GitHub/CI の責務境界
- minimum viable / rippable な設計姿勢

以下、優先順位順で整理します。

## 5. 優先度A: 追加すべき設定

### A-1. `behavior verification policy` を追加する

対象章:

- 4-3 `VERIFY.md`
- 9-6 `review-gate`
- 14 `実行ワークフロー`
- 18 `ヘルスチェック`

理由:

- Fowler が `behaviour harness` を独立カテゴリとして挙げている
- Speaker Deck でも「CI が通っただけで完了扱いにしない」「ブラウザで確認する」が強調されている
- 現行設計は lint / typecheck / test / review に強いが、`ユーザー期待通り動くか` を構造的に残す設定が弱い

追加案:

```yaml
# .ops/config/verification-policy.yaml
computational_gates:
  - name: lint
    required: true
  - name: typecheck
    required: true
  - name: unit_test
    required: true
  - name: integration_test
    required: task_based

inferential_gates:
  - name: acceptance_check
    required: true
    artifact: VERIFY.md
  - name: ux_path_check
    required: ui_tasks_only
    artifact: artifacts/screenshots/
  - name: root_cause_validation
    required: when_fixing_review_findings

behavior_checks:
  - name: golden_path
    required: true
  - name: negative_path
    required: task_based
  - name: regression_watchpoints
    required: task_based
```

設計書への追記ポイント:

- `VERIFY.md` を「事実のみ」から一歩進めて、`computational` と `inferential` を分ける
- `done` 判定条件に `acceptance path confirmed` を入れる
- UI / API / workflow 系タスクでは `screenshots` または `curl transcripts` を必須 artifact にする

### A-2. `harnessability` 判定設定を追加する

対象章:

- 0-3 `進化前提`
- 2 `ディレクトリ設計`
- 13 `Triage`
- 23 `導入チェックリスト`
- 24 `最終判断`

理由:

- Fowler が `harnessability` を明示している
- Aochan 記事は legacy codebase との相性と導入コストを問題化している
- 今の設計は「全て初日から採用」の思想が強く、repo / area の成熟度差を吸収しにくい

追加案:

```yaml
# .ops/config/harnessability.yaml
areas:
  - path: 10-apps/dify/
    doc_readiness: high
    test_readiness: medium
    static_analysis: high
    boundary_clarity: medium
    risk_level: medium
    automation_level: assisted
  - path: legacy/
    doc_readiness: low
    test_readiness: low
    static_analysis: low
    boundary_clarity: low
    risk_level: high
    automation_level: observe_only

gates:
  autonomous_execution_requires:
    doc_readiness: medium
    test_readiness: medium
    boundary_clarity: medium
  review_only_mode_if:
    risk_level: high
```

設計書への追記ポイント:

- `ready` に上げる条件へ「対象 area の harnessability が最低基準を満たす」を追加
- `legacy` や `high-risk` area では `Execution Agent` を `edit` ではなく `review-only` で開始できるようにする
- `Step 1` 導入チェックに `area readiness scan` を追加する

### A-3. Review ループ用の `findings policy` を追加する

対象章:

- 9-6 `review-gate`
- 10 `Subagents`
- 14-2 `失敗パターン`
- 22 `失敗からの再開`

理由:

- Zenn の review 実践記事では、`severity triage`、`oscillation detection`、`directive 固定`、`validation` が収束に効いている
- 現行設計の `Review Agent` は存在するが、結果の型と再試行制御が弱い

追加案:

```yaml
# .ops/config/review-loop.yaml
finding_severity:
  allowed: [critical, important, low]
  merge_blockers: [critical, important]

oscillation_control:
  detect_abab: true
  max_fix_loops: 3
  directive_file: DIRECTIVES.md

validation:
  require_root_cause_check: true
  reject_band_aid_fix: true
  ignore_out_of_scope_findings: true

handoff:
  persist_review_directives: true
```

設計書への追記ポイント:

- task packet に `DIRECTIVES.md` を追加する
- `REVIEW.md` の判定を free text だけでなく構造化する
- `review -> blocked` 条件に `oscillation detected` を追加する
- `failure-classifier` に `review_oscillation` を追加する

### A-4. `tool / skill visibility budget` を追加する

対象章:

- 7 `settings`
- 9 `skills`
- 10 `subagents`
- 11 `permissions`

理由:

- Aochan 記事が、ハーネス過剰構築やツールの多さが逆効果になり得ることを整理している
- Speaker Deck でも agent ごとに責務とコンテキストを分けている
- 現行設計は skill 数・agent 数ともに豊富で強いが、見せ過ぎると agent 側の選択コストが上がる

追加案:

```yaml
# .ops/config/tool-budget.yaml
roles:
  triage:
    visible_skills: [triage-from-state, make-task-packet, impact-analyzer, issue-sync]
    visible_tools: [read, grep, gh_issue, gh_project, git_read]
    max_visible_skills: 4
  executor:
    visible_skills: [worktree-execution, task-handoff, failure-classifier]
    visible_tools: [edit, write, test, git_read, git_worktree]
    max_visible_skills: 3
  reviewer:
    visible_skills: [review-gate, issue-sync]
    visible_tools: [read, diff, gh_pr]
    max_visible_skills: 2
```

設計書への追記ポイント:

- `subagent` ごとに「見える skill / 見えない skill」を固定する
- `research` agent には write を与えないだけでなく、review や execution 用 skill も見せない
- `settings.json` の allow/deny と別に「役割別に見せる道具の最小集合」を定義する

## 6. 優先度B: 追記すべき設定

### B-1. `context engineering は harness の一部` と明記する

理由:

- Louis Bouchard と Qiita 記事の両方が、prompt / context / harness の包含関係を整理している
- 現行設計は実質そうなっているが、0章で言い切っておくと後続章の理解が揃う

追記文案:

- `context engineering は harness engineering の一部であり、docs / task packet / skills / handoff は context 供給面を構成する`

### B-2. `human decision taxonomy` を追加する

理由:

- Fowler は人間の役割を設計対象として扱っている
- 現行設計では「人間がやること」は書かれているが、判断の種類が未分類

追記したい分類:

- intent / business trade-off
- spec ambiguity resolution
- destructive / billing / security approval
- merge / reject / hold
- harness change approval

### B-3. `minimum viable harness` と `rippable harness` の両立方針を追加する

理由:

- Aochan 記事は「ハーネスはそれ自体がソフトウェア化する」と整理している
- `24. 採用すべきもの (全て初日から)` は思想として強いが、運用開始条件としては重い

追記文案:

- core: `task packet + handoff + required CI + worktree`
- extended: `failure classifier + automated re-triage + issue sync`
- advanced: `specialized reviewers + browser checks + scheduled daemon`
- 各層に「削除条件」を書く

### B-4. `GitHub native controls との境界表` を追加する

理由:

- Zenn の shio 記事が最も強く指摘している論点
- ローカル hook でやるべきことと GitHub 側で authoritative にすべきことを分けると設計の説明力が増す

追記したい対応表:

| 領域 | ローカルハーネス | GitHub / CI |
|---|---|---|
| 事前安全 | PreToolUse, deny-first | branch protection |
| 品質ゲート | preflight test, review-gate | required checks |
| レビュー記録 | REVIEW.md | PR review / comments |
| 実行状態 | task packet / runs | Issues / Projects |
| 最終承認 | human in queue | merge button / approvals |

### B-5. `evidence registry` を追加する

理由:

- context aggregation の観点では、docs の所在が散るほど agent は弱くなる
- 現行設計は docs repo と main repo に情報が分かれるので、参照インデックスがあると triage と review の安定性が上がる

追記案:

```yaml
# .ops/config/evidence-index.yaml
required_references:
  - requirements
  - decisions
  - status
  - related_issue
  - related_pr
  - last_verify_artifacts
```

## 7. 優先度C: 代替案

### C-1. 代替案: `Thin Local / Strong GitHub`

向いている場面:

- チーム運用
- 既に GitHub Actions / branch protection / CODEOWNERS が強い repo
- ローカル hook を増やし過ぎたくない場合

構成:

- ローカルは `task packet + handoff + worktree + minimal safety hooks` に絞る
- authoritative gate は GitHub の required checks / review approval に寄せる
- `issue-sync` は残すが、レビュー判定の正本は PR に置く

利点:

- 「bash で Git Workflow を書き直すだけ」の面積を減らせる
- チーム共通運用に乗せやすい

### C-2. 代替案: `Area-based rollout`

向いている場面:

- 既存の legacy codebase を含む repo
- area ごとに test / docs / ownership の成熟度が違う場合

構成:

- `automation_level: observe_only / review_only / assisted / autonomous` を area 単位で持つ
- `24. 採用すべきもの` を repo 一律ではなく area 別に適用する

利点:

- harnessability の低い領域へ無理に自動実装を入れずに済む
- 導入失敗率を下げやすい

### C-3. 代替案: `Specialized review mesh`

向いている場面:

- review がボトルネックになっている場合
- 品質観点を明確に分けたい場合

構成:

- `reviewer` を1体にせず、`spec-reviewer` / `arch-reviewer` / `perf-security-reviewer` に分割
- Speaker Deck のように並列化し、最後に human が統合判断する

利点:

- 1つの review prompt に全観点を押し込まずに済む
- `computational -> inferential -> human` の順序が作りやすい

## 8. 章ごとの具体的な追記ポイント

### 0. 設計思想

- `context engineering は harness engineering の一部` を追記
- `minimum viable` と `rippable` を両立する原則を追記

### 3. Queue State Machine

- state は増やさず、`review` 内に `behavior_pending` のサブ状態を持たせる案がよい
- `blocked` の理由に `oscillation` と `harnessability_gap` を追加すると運用が締まる

### 4. Task Packet

- `DIRECTIVES.md` を追加
- `VERIFY.md` を `computational / inferential / behavior evidence` に分ける
- `task.yaml` に `automation_level` と `behavior_risk` を追加する価値が高い

### 7. Settings

- `permissions.defaultMode = plan` は良い
- 追加するなら `role-based visible tools` と `network egress whitelist`
- `model: sonnet` 固定だけでなく、`review` と `behavior check` のモデル選択条件を task 側に持たせるとよい

### 8. Hooks

- `PreToolUse` / `Stop` は良い
- 追加優先度が高いのは `behavior artifact missing` を警告する hook
- 次点で `diff size / file count threshold` を超えたら review 強化へ送る hook

### 9. Skills

- 既存 8 skill は強い
- 追加優先は `review-findings-triage`、`behavior-check`、`directive-lock`

### 11. Permissions

- deny-first は維持
- その上で「許可する」だけではなく「見せる道具を減らす」を併用したい

### 12. GitHub Issues / Projects

- `Issue は管理、task packet は実行の真実` は妥当
- 追記するなら `PR required checks`、`branch protection`、`CODEOWNERS` を authoritative controls として明示

### 13. Triage

- 6軸スコアは実用的
- 追加したい軸は `behavior risk` と `harnessability readiness`
- ただしスコア軸を増やしすぎると複雑になるので、別テーブル化が無難

### 18. ヘルスチェック

- `同種 failure が3件以上` は良い
- 追加したいのは `review loop count`、`oscillation incident count`、`behavior evidence missing count`

### 24. 最終判断

- 「全て初日から」は思想として残してよい
- その直後に `ただし autonomous execution の解禁は harnessability gate を通った area のみ` と追記すると実運用に耐えやすい

## 9. 推奨改訂順

1. `verification-policy.yaml` を追加して behavior harness を明文化する
2. `review-loop.yaml` と `DIRECTIVES.md` を追加してレビュー収束性を上げる
3. `harnessability.yaml` を追加して対象 area を選別する
4. `GitHub native controls` との境界表を 12 章に追加する
5. `tool-budget.yaml` を追加して role ごとの見える道具を減らす
6. 余力があれば `minimum viable / rippable` の段階導入方針を 0 章と 24 章に追加する

## 10. このレビューで主に根拠にしたURL

- https://martinfowler.com/articles/harness-engineering.html
- https://www.louisbouchard.ai/harness-engineering
- https://qiita.com/search?q=ハーネスエンジニアリング
- https://qiita.com/search?q=ハーネス+コード書かない
- https://qiita.com/search?q=Martin%20Fowler%20ハーネスエンジニアリング
- https://zenn.dev/theaktky/articles/1c6c3b9333117c
- https://zenn.dev/shio_shoppaize/articles/shogun-harness-engineering
- https://speakerdeck.com/search?q=MOSH+Harness

補助取得した直接記事:

- https://qiita.com/nogataka/items/d1b3fcf355c630cd7fc8
- https://qiita.com/Aochan0604/items/306bde3e138ce071f7b2
- https://speakerdeck.com/kajitack/implementing-herness-engineering
