# ハーネス設計案 A / B / C 比較評価レポート

**作成日**: 2026-04-02
**目的**: 3つのハーネス設計案を、自律運用・タスク管理・将来性の観点から評価・比較する

---

## 1. 各設計案の概要

### 設計案A: 伝統的レイヤー構造

- **構造**: ユーザー + Layer1（全体管理AI）+ Layer2（実行管理AI）+ Layer3（実行AI）の4層
- **思想**: 「会話」ではなく「状態ファイル」で継続する。正本を少なくする
- **タスク管理**: ローカルIssueファイル（open/closed/runs）+ `_index.md` で一覧管理
- **Layer間通信**: Issueファイルへの追記（`## 判定案` `## 実行結果` セクション）
- **ツール**: Claude Code一本化。安定後に拡張
- **セッション管理**: tmux + `/clear` + 状態ファイルからの復帰

### 設計案B: Claude Code機能フル活用型

- **構造**: Triage / Executor / Reviewer の3役割セッション
- **思想**: 「役割を会話に持たせない」。ファイル配置・権限・queue状態・worktreeで役割を決定
- **タスク管理**: task pack（BRIEF/PLAN/HANDOFF/VERIFY/RESULT/REVIEW/DECISIONS/RUNLOG/RISKS）
- **queue**: inbox → proposed → ready → running → review → done の6状態
- **Claude Code活用**: skills, hooks, subagents, permissions, sandbox を全面活用
- **GitHub連携**: Projects + Issues + `gh` CLI。AI主導トリアージ with スコアリング

### 設計案C: プロジェクト内運用統合型

- **構造**: Triage Claude / Execution Claude / Review Claude の3役割（B案と同系統）
- **思想**: `.ops/` ディレクトリでqueue・runs・reports・templatesをプロジェクト内に統合
- **タスク管理**: task packet（task.yaml + brief/plan/review/handoff/result/issue/links）
- **queue**: inbox → proposed → ready → running → review → blocked → done の7状態（blocked追加）
- **特徴**: manifest.json による構造化ログ、failure-classifier skill、Local-first → GitHub-integrated の段階的移行

---

## 2. 評価軸と各案の評価

### 2-1. 不在時の自律実行性（仕事中・就寝中にタスクが前進するか）

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| タスク自動開始 | **不可**。Layer2/3の起動はユーザーまたはLayer1が手動で行う | **半自動**。AI主導トリアージ → ユーザーが本数決定 → 実行 | **半自動**。B案と同様だが `proposed` → `ready` の昇格にユーザー介入が必要 |
| セッション継続性 | tmux依存。切断時は状態ファイルから復帰 | worktree + handoff で会話非依存。resumeも可 | 同上。さらにmanifest.jsonで実行メタデータを構造化 |
| 失敗時の自己回復 | Layer3は失敗仮説を残すのみ。再試行はLayer2判断 | hooks（PostToolUse）で自動検知。skills（failure-classifier相当）で分類可 | **最も充実**。failure-classifier skill で環境/権限/flaky/仕様曖昧/実装バグ/要人間判断に自動分類 |
| 並列実行 | Issue単位で並列可。ただし手動起動 | worktree単位で並列可。task pack + worktreeで自然に隔離 | 同上 |

**評価**: A案は手動起動が前提のため不在時の自律性が最も低い。B案・C案は「AIがタスク候補を出し、ユーザーが承認した分だけ自律実行」という構造で、不在時運用に適している。C案はfailure-classifierにより失敗パターンの自動分類ができる点で若干優位。

---

### 2-2. 完了/失敗タスクの整理しやすさ（起床後・仕事終わり）

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| 完了確認 | `_index.md` + issueファイルの `## 判定案` を読む | `TRIAGE_REPORT.md` + `RESULT.md` + `VERIFY.md` | `summary.md` + `manifest.json` + queue状態 |
| 失敗確認 | `runs/` ログ + issueの `## 実行結果` | `RUNLOG.md` + `HANDOFF.md` | `summary.md` + `blocked_reason.md` + failure分類結果 |
| 一覧性 | `_index.md` が一覧だが手動更新 | `TRIAGE_REPORT.md` が自動生成される前提 | queue のフォルダ構造（proposed/ready/running/review/blocked/done）で物理的に分類 |
| 次アクション判断 | Layer2の判定案（ready_for_close/continue/needs_followup/blocked）を読む | CANDIDATES.yamlのスコアリングで優先度が明確 | 同上 + blocked再スキャン + 類似原因の束ね機能 |

**評価**: C案が最も「朝起きて状況を把握する」ユースケースに強い。queueのフォルダ状態がそのまま進捗を示し、manifest.jsonで各runのメタデータが構造化されている。A案は`_index.md`の手動更新負荷が高い。B案はTRIAGE_REPORTの自動生成で中間的。

---

### 2-3. やり直し/継続/新規の判断支援

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| 判定フレームワーク | 4択（ready_for_close / continue / needs_followup / blocked） | AI主導トリアージが自動で「継続/分離判定」を含む | 同上 + failure-classifier による失敗原因の自動分類 |
| 新規タスク提案 | Layer1が手動で判断 | Triage sessionが6軸スコアリングで候補を自動生成 | 同上。Triage Claudeが毎朝blocked再スキャンも実施 |
| 仕様変更対応 | 上位文書から下位へ更新する手順あり。ただし検知は手動 | 「仕様変更パターン」として継続/分離の判断基準を明記 | `inbox` に変更要求 → `impact.md` 作成 → 影響先を自動分類 |

**評価**: B案・C案のAI主導トリアージが圧倒的に優位。特にC案は仕様変更時に `impact.md` を作成して影響範囲を自動分析する仕組みがある。A案はLayer1が人間+AIの半自動で、判断プロセスが最も手動寄り。

---

### 2-4. リアルタイム状況把握とタスク提案の精度

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| 状況の把握方法 | `status.md` + `_index.md` + issueファイル | GitHub Project + queue状態 + recent commits + 失敗ログ + DECISIONS + RISKS | 同上 + docs repo + `.ops/runs/` の構造化ログ |
| 提案の構造化 | なし（Layer1の自由記述） | CANDIDATES.yaml（6軸スコアリング: impact/urgency/unblock_value/effort/confidence/cross_project_leverage） | 同上（B案のスコアリングをそのまま採用可能） |
| ヘルスチェック | Layer1の定期チェック項目（5項目）あり | 明示的なヘルスチェック機構なし（hooksで部分的にカバー） | Triage Claudeが毎朝blocked再スキャン + stale running検知 |

**評価**: B案のスコアリング体系が最も体系的。C案はB案の体系をベースにしつつ、毎朝の自動再スキャンを追加。A案のヘルスチェック項目は運用として堅実だが自動化度が低い。

---

### 2-5. 実装の複雑さと導入コスト

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| 初期セットアップ | **最小限**。ファイル構造のみ。CLAUDE.mdと状態ファイルがあれば開始可能 | **中程度**。CLAUDE.md + settings.json + skills + hooks + subagents + task pack雛形 | **最大**。B案の全要素 + .ops/ディレクトリ構造 + manifest.json + templates + reports |
| 学習コスト | 低。Layer構造は直感的 | 中〜高。Claude Codeの機能（skills/hooks/subagents/permissions/sandbox）の理解が必要 | 高。B案の全知識 + queue state machineの設計 + ログ構造の理解 |
| ファイル数 | 少（status.md, _index.md, issueファイル, runs/） | 中（task pack 9ファイル + skills + subagents） | 多（task packet 7ファイル + .ops/下の多数のディレクトリ + manifest.json） |
| 段階的導入 | 容易。最小運用セットが明記されている | 可能。ただし全機能を使わないと設計の真価が出にくい | Phase A〜Dの段階的導入が明記されている |

**評価**: A案が最も軽量に始められる。C案は最も包括的だが初期コストが高い。B案は中間だが、Claude Code機能への依存度が高く、機能理解が前提。

---

## 3. 将来性の分析（AI性能向上・ツール機能追加への耐性）

### 3-1. モデル性能向上への耐性

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| モデル非依存性 | **高**。設計がモデル機能に依存しない。どのモデルでも動く | **中**。Claude Code固有機能（skills/hooks/subagents）に強く依存 | **中〜高**。Claude Code機能を使うが、queue/task packet/worktreeの骨格はツール非依存 |
| 性能向上の恩恵 | **受けにくい**。手動運用部分がボトルネック。モデルが賢くなってもLayer1の手動判断は変わらない | **受けやすい**。AIトリアージの精度向上がそのままスコアリング精度に直結 | **最も受けやすい**。failure-classifierや仕様変更impact分析など、モデル性能が直接品質に影響する機能が多い |
| コンテキスト拡大の恩恵 | 限定的。設計が「小さく分割」前提 | 恩恵あり。だがhandoff/task packで分割する設計は維持すべき | 同上 |

### 3-2. Claude Code機能追加への耐性

| 想定される機能追加 | A案への影響 | B案への影響 | C案への影響 |
|---|---|---|---|
| **Agent Team / 並列実行の公式サポート** | Layer2→Layer3の手動起動が自動化される可能性。設計変更が必要 | subagentsベースなので自然に統合可能 | 同上 |
| **auto permissionモードの成熟** | 手動承認フローの簡素化のみ | permissions設計全体が簡素化。hooks/settingsの安全装置は残る | 同上 |
| **永続セッション / 常駐エージェント** | tmux依存が不要に。ただしLayer構造は維持 | Triage sessionの常駐化が自然に実現 | Triage Claudeの常駐化 + 毎朝再スキャンの自動化 |
| **GitHub Actions連携の強化** | 設計に含まれていないため、後付けで統合が必要 | `/install-github-app` から段階的に導入する設計あり | Local-first → GitHub-integrated の移行パスが明記 |
| **MCP / 外部ツール統合の拡大** | 設計に含まれていない | `.mcp.json`を想定しているが第1版では不要と明記 | 同上 |
| **schedule / remote trigger 機能** | 対応なし。手動起動前提 | `/schedule`を「必要になってから使う」に分類。導入パスあり | 同上。CLI commands節で整理済み |
| **skills / hooks の高度化** | 設計に含まれていない | **最も恩恵を受ける**。skills/hooksを中心に設計されているため | 同様に恩恵大 |

### 3-3. 他ツール（Codex等）への拡張性

| 観点 | A案 | B案 | C案 |
|---|---|---|---|
| ツール切替の容易さ | **高**。ファイルベースの状態管理はツール非依存 | **中**。queue/task pack/worktree/issueの骨格は流用可能だが、skills/hooks/subagentsはClaude Code固有 | **中〜高**。C案は「queue/task packet/worktree/handoff の骨格はそのまま流用できる」と明記 |
| マルチツール運用 | 想定していないが、Layer別にツールを変えることは原理的に可能 | 「Codex等への展開前の最初の本命ハーネス」として設計 | 同上 |

### 3-4. 将来性の総合判断

**短期（〜6ヶ月）**: B案・C案が優位。Claude Codeの既存機能を最大活用でき、AI主導トリアージにより不在時運用が現実的。

**中期（6ヶ月〜1年）**: C案が最も有利。段階的導入パス（Phase A〜D）が明記されており、GitHub Actions連携やAgent SDK常駐化への移行が計画されている。Claude Code機能が成熟するほど恩恵を受ける。

**長期（1年〜）**: A案の「ファイルベース・ツール非依存」の思想は、AIツールの世代交代に最も強い。ただし手動運用部分がボトルネックとして残り続ける。B案・C案のqueue/task pack/worktreeの骨格部分はツール非依存なので、Claude Code固有部分（skills/hooks/subagents）だけを差し替えれば移行可能。

---

## 4. ユースケース別適合度

### 主要ユースケース: 「仕事中・就寝中に実装を回し、仕事終わり・起床後にタスク整理」

| 要件 | A案 | B案 | C案 |
|---|---|---|---|
| 不在時にタスクが前進する | △ 手動起動が必要 | ○ AI主導トリアージ + ユーザー承認で実行 | ○ 同上 + blocked自動再スキャン |
| 起床後に完了/失敗を一覧できる | △ _index.md手動更新 | ○ TRIAGE_REPORT自動生成 | ◎ queueフォルダ状態 + manifest.json + summary.md |
| やり直し/継続/新規を判断できる | △ 4択判定案のみ | ○ 6軸スコアリング + 継続/分離判定 | ◎ failure-classifier + impact分析 + 毎朝再スキャン |
| 毎回の状況把握が素早い | △ 複数ファイル参照が必要 | ○ CANDIDATES.yamlで構造化 | ○ 同上 |
| スムーズに次の実装サイクルを開始できる | △ Layer起動が手動 | ◎ 「proposed見る → 本数決める → ready昇格」の3ステップ | ◎ 同上 |

---

## 5. 総合評価

| 評価軸 | A案 | B案 | C案 |
|---|---|---|---|
| 自律実行性 | ★★☆☆☆ | ★★★★☆ | ★★★★☆ |
| タスク整理のしやすさ | ★★★☆☆ | ★★★★☆ | ★★★★★ |
| 判断支援の質 | ★★☆☆☆ | ★★★★★ | ★★★★★ |
| 導入の容易さ | ★★★★★ | ★★★☆☆ | ★★☆☆☆ |
| 将来の拡張性 | ★★★☆☆ | ★★★★☆ | ★★★★★ |
| ツール非依存性 | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| 不在時運用への適合 | ★★☆☆☆ | ★★★★☆ | ★★★★★ |

---

## 6. 推奨と提案

### 6-1. 推奨案

**C案をベースに、A案の軽量さを初期導入に取り入れる段階的アプローチ** を推奨する。

理由:
- C案はユースケース（不在時運用 → 起床後整理 → 次サイクル開始）に最も適合している
- C案のPhase A〜D段階導入は、A案の「最小運用セット」思想と両立可能
- C案のqueue/task packet/worktree/handoffの骨格は、AIツールの進化に対してもロバスト
- failure-classifierやimpact分析は、モデル性能向上の恩恵を最も受けやすい

### 6-2. 具体的な導入ステップ

**Phase 0（今すぐ）**: A案の最小運用セットで開始
- CLAUDE.md
- status.md
- 基本的なissue管理

**Phase 1（1〜2週間後）**: C案のqueue + task packetを導入
- `.ops/queue/` ディレクトリ構造
- task packet雛形
- handoff.mdの運用開始

**Phase 2（1ヶ月後）**: C案のClaude Code機能を順次有効化
- subagents（triage-manager, executor-general, reviewer）
- skills（triage-from-state, make-task-packet, failure-classifier）
- hooks（PreToolUse, PostToolUse, Stop）

**Phase 3（2〜3ヶ月後）**: GitHub連携の本格化
- GitHub Projects + Issues
- AI主導トリアージの自動化
- `/schedule` やGitHub Actionsの検討

### 6-3. 3案から採用すべき要素の取捨選択

| 採用元 | 要素 | 理由 |
|---|---|---|
| A案 | ヘルスチェック5項目 | 定期的な整合性確認は全設計に必要 |
| A案 | ドキュメント更新順序（上位→下位） | 正本管理の原則として堅実 |
| A案 | decisions.mdの記録基準 | 判断理由の残し方が最も明確 |
| B案 | 6軸スコアリング（CANDIDATES.yaml） | タスク提案の質を定量化できる |
| B案 | 「毎回role宣言しない」原則 | ファイル/権限/worktreeで役割を決める思想 |
| B案 | CLI/slashコマンドの使い分け整理 | 日常/問題時/導入時の分類が実用的 |
| C案 | queue 7状態（blocked追加） | blockedの明示的管理が不在時運用に必須 |
| C案 | failure-classifier skill | 失敗の自動分類が再試行判断の精度を上げる |
| C案 | manifest.json構造化ログ | 起床後の一括状況把握に最適 |
| C案 | Phase A〜D段階導入 | 現実的な移行パス |

### 6-4. 各案から不採用とすべき要素

| 不採用元 | 要素 | 理由 |
|---|---|---|
| A案 | Layer1〜3の3層構造 | セッション起動が手動で不在時運用に不向き。Triage/Executor/Reviewer の3役割の方がClaude Codeのsubagent/sessionモデルと整合する |
| A案 | `_index.md`手動更新 | 更新漏れのリスクが高い。queueフォルダ状態の方が信頼性が高い |
| B案 | task pack 9ファイル構成 | RISKS.mdやDECISIONS.mdはtask単位では過剰。プロジェクト単位で十分 |
| C案 | `.ops/runs/` の日付ベースディレクトリ | task ID ベースの方が追跡しやすい（A案のruns/ISS-001/形式が優れる） |

---

## 7. AI性能向上・機能追加に対するロバスト性の結論

### 継続して使える設計要素（ツール非依存の骨格）

以下の要素は、どのAIツールが主役になっても変わらない普遍的な設計原則であり、全案に共通して長期的に有効:

1. **queue state machine**（inbox → proposed → ready → running → review → blocked → done）
2. **1 task = 1 worktree = 1 branch**（隔離原則）
3. **handoff.md を再開の正本にする**（会話依存の排除）
4. **ファイルベースの状態管理**（AIツール非依存の真実）
5. **AI主導トリアージ + 人間が本数を決める**（自律性と安全性のバランス）
6. **failure分類の自動化**（モデル性能向上で精度が上がる設計）

### AIの進化で陳腐化しうる要素

1. **手動セッション起動**（永続エージェント/schedule機能で不要になる）
2. **tmux依存**（同上）
3. **毎回のpermission承認**（auto modeの成熟で簡素化）
4. **手動のissue同期**（GitHub Actions連携の強化で自動化）
5. **subagent定義の手動管理**（Agent Team機能で自動委譲が進む）

### 結論

**C案の骨格（queue/task packet/worktree/handoff/failure-classifier）は、AIの進化に対して最もロバスト**。Claude Code固有機能（skills/hooks/subagents）への依存部分は、将来的にツールが変わっても「同等の概念」が存在する可能性が高く（既にCodexやCursorにも類似機能がある）、完全な書き直しにはならない。

一方、A案の「ツール非依存」思想は保険として価値があるが、**現時点でAIの能力を活用しきれない設計は、結果的に「手動作業のボトルネック」として残り続け、AIの進化の恩恵を受けにくい**。

したがって、**C案をベースに段階的に導入し、骨格部分（queue/task packet/worktree/handoff）をツール非依存に保ちつつ、Claude Code固有機能は積極的に活用する**のが最適解である。
