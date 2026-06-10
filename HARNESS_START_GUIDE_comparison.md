# ハーネス スタートガイド 徹底比較

## Codex版 vs Claude版

**比較対象**:
- [Codex版](file:///Users/miurakousuke/GitHub/ops/APP_PROJECT_HARNESS_START_GUIDE_for_codex.md) (1927行 / 67,674 bytes)
- [Claude版](file:///Users/miurakousuke/GitHub/ops/APP_PROJECT_HARNESS_START_GUIDE_for_claude.md) (1179行 / 68,133 bytes)

**比較日**: 2026-06-10

---

## 総評

両文書は同じ設計書（`harness_design-D_unified.md`）を基にしており、**13ステップの全体フロー・セッションA〜Gの分割・queue 7状態・task packet必須ファイル・後処理優先の原則** といった根幹構造は一致している。ただし「書き方のスタイル」と「何をどこまで踏み込むか」に明確な差がある。

| 観点 | Codex版 | Claude版 |
|---|---|---|
| 全体の長さ | 1927行（より長い） | 1179行（より短い） |
| バイト数 | 67,674 bytes | 68,133 bytes（日本語密度が高い） |
| 文体 | 箇条書き・リスト主体。各項目を独立した小見出しで列挙 | 散文＋テーブル＋引用ブロックを混合。段落内で理由を説明しがち |
| 情報密度 | 低め（1項目1見出しで冗長だが検索しやすい） | 高め（段落内に複数の判断基準が埋め込まれる） |
| 構成思想 | 「全部書き出す」辞書型。コピペで使いやすさを重視 | 「理由を語る」ガイド型。なぜそうするかの説明を重視 |

---

## セクション別 詳細比較

### 1. 冒頭（目的・対象読者・範囲）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| タイトル | 「草案ありの〜開始ガイド」 | 「ハーネス導入スタートガイド — Design D ハーネス運用に載せるまで —」 | Claude版はサブタイトルで Design D を明示 |
| 前提設計書の明示 | 本文中で触れる | 冒頭にメタデータとして `harness_design-D_unified.md` と `current mode` を明記 | **Claude版のみ** |
| 対象読者 | 4項目（Claude/Codex/Antigravity/Cursor に言及） | 4項目（Claude Code CLI のみ言及） | Codex版は他エージェントの補助利用に言及 |
| 扱わない範囲 | 5項目 | 4項目 | Codex版は `secrets を読ませる運用`・`人間ゲートを飛ばす merge` を明示的に除外 |
| 大原則 | 10項目のチェックリスト | 9項目のチェックリスト | ほぼ同一。Codex版は `current mode 前提にする` を独立項目化。Claude版は `後処理優先の処理順` を本文に埋め込む |

### 2. 全体の流れ（§1）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| ステップ数 | 14ステップ（0〜13） | 13ステップ（1〜13） | **Codex版にのみステップ0「安全な作業場所を決める」がある**。Claude版はこれを「追加で必要な準備」として別枠で扱う |
| テーブル列 | 順番 / 作業 / 何をするか / なぜこの順番か | # / ステップ / なぜこの順番か | Codex版は「何をするか」列が追加で冗長 |
| 追加準備の明示 | なし（各セッション内に散在） | **専用セクション「追加で必要な準備・確認」** として git初期化、.gitignore、worktree置き場、secrets隔離、parallel.yaml初期値、スモークテストの6項目を列挙 | **Claude版のみ**。実務的に重要な抜け漏れ防止項目 |
| 後処理優先 | 独立セクション「後処理優先の原則」として7段階の処理順と新規着手条件5項目を明記 | 大原則の1項目として処理順を1行で記載 | **Codex版が詳しい** |

### 3. 作業分割（§2）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 分割の基本方針 | 4項目の箇条書き | 4項目＋「1セッション=人間が15〜30分で確認しきれる量」 | **Claude版のみ「確認時間の目安」**を明示 |
| セッションEの分割 | 1セッションとして記載 | **E1(rules) / E2(agents) / E3(skills) に分割を推奨** | **Claude版が細かく分割**。rules→agents→skills の参照関係を理由に挙げる |
| セッションB2 | 追加セッションの表に「hooks実装」として記載 | **独立セクション「セッションB2: スモークテスト」** として詳述 | **Claude版のみ**ダミータスクでの配管確認を強く推奨 |
| 追加セッション | hooks実装 / 初回review / secrets棚卸し / GitHub連携 の4つ | なし（スモークテストのみ追加） | **Codex版が多い**（hooks実装を分離すべきと明言） |
| セッションBの読ませるファイル | `status.md`、`roadmap.md`、設計書の queue/task packet 部分 | 設計書 §6・§8、**project_overview.md のみ**（概要把握用） | Claude版はセッションBで読む量を意図的に絞っている |
| 分割する理由の説明 | 1行の理由 | **太字＋補足説明**で理由を詳述 | Claude版は「なぜ分けるか」の説明が厚い |

### 4. ドキュメント作成順（§3）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 順番テーブル | 14項目テーブルに「分類」「作り方」列あり | 14項目の番号付きリスト | 形式の違い。内容は同一 |
| 分類の数 | 5分類（一括 / 丁寧 / 雛形 / 実運用後 / 最初は不要） | 5分類（同じ） | 同一 |
| 丁寧に作るべきリスト | `rules/safety.md`、`role-detection.md` を含む | `rules/safety.md`、`role-detection.md` を含む | 同一 |
| 実運用後リスト | Codex版: `issue-sync` / `impact-analyzer` / `auto-advance` / `dashboard` / `scoring-weights` / GitHub連携 / hooks高度化 | Claude版: **hooks** / `scoring-weights` / `dashboard` 出力 / permissions allow追加 | **Codex版の方が多い**。Claude版は hooks をここで初めて言及 |
| 最初は不要リスト | 本番デプロイ / 課金API / secrets前提 / `/schedule` / GitHub Actions / 並列実行 | `.github/` / `.mcp.json` / `log-summarizer` / `research` agent / `architecture.md` / subdirectory CLAUDE.md | **ほぼ重複なし**。視点が違う（Codex版=運用の高度化、Claude版=ファイル構成） |

### 5. 00-docs の作り方（§4）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 共通ルール | 6項目チェックリスト（`secrets を書かない` を含む） | 4項目チェックリスト | **Codex版に `secrets 書かない`・`草案にないことは未決定`** の2項目が追加 |
| 各ドキュメントの説明形式 | 「書くこと / 書かないこと」を箇条書き＋プロンプト例 | テーブル（1行「書くこと」「書かないこと」）＋プロンプト例 | Claude版はテーブルでコンパクト |
| project_overview.md | 書くこと7項目、書かないこと5項目 | 書くこと・書かないことを各1セル | **Codex版が詳細** |
| requirements.md | 書くこと6項目（ユーザー操作・利用シーン・実装前質問を含む） | 書くこと＝要求一覧＋タグ付け＋スコープ＋非機能 | Codex版に「ユーザー操作や利用シーン」「実装前に質問が必要な項目」の2点が追加 |
| roadmap.md | 書くこと5項目（優先順位の理由を含む） | フェーズ分割＋ゴール＋完了条件＋依存 | Codex版に「優先順位の理由」「実装より前に必要な調査」が追加 |
| decisions.md | 書くこと6項目 / 書かないこと4項目 | DEC-001形式を明記。**「迷うものは記録せず候補に」** | **Claude版のみ** DEC 形式（番号/日付/決定/理由/影響）を指定 |
| status.md | 書くこと5項目（`DASHBOARD.md` との関係を含む） | 4節構造（現在フェーズ/完了/進行中/次の予定）を明記 | **Claude版のみ** 4節構造を指定 |
| プロンプト例の詳しさ | 要件・出力を分けた構造化プロンプト | 自然文に近いプロンプト＋制約 | Codex版は機械的、Claude版は説明的 |

### 6. CLAUDE.md / settings.json / rules / skills / agents（§5）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| **CLAUDE.md** | | | |
| 他ファイルとの違い | テーブル形式で5つの場所と役割を比較 | テーブルで「答える問い」と**「強制力」**の2列で比較 | **Claude版のみ「強制力」概念**を導入（context / 条件付き注入 / 強制） |
| 書くべきこと | 10項目リスト | 10項目＋階層構造で具体値例あり（例: Non-negotiables の中身を列挙） | **Claude版の方が具体的** |
| 確認ポイント | 6項目チェックリスト | 4項目チェックリスト（「自分が全文理解できるか」を含む） | Claude版に **「理解できない憲法は運用できない」** という独自の観点 |
| **settings.json** | | | |
| 構成 | 方針別（secrets / 破壊的 / 本番 / worktree外 / 理由）で分割 | 同様の分割＋引用ブロックで**バージョン変更リスク**を強調 | Claude版は settings スキーマ変更リスクをより強く警告 |
| 確認ポイント | 7項目 | 4項目（**`deny の実地テスト`** を含む） | **Claude版のみ「secrets 読み取りの deny を実地テストで確認」**を要求 |
| `settings.local.json` | 必要なら作成と記載 | `.gitignore` への追記を明示的に成果物に含む | Claude版がより実務的 |
| **rules** | | | |
| 最初に必要な rules | 5つ（safety / role-detection / autonomous-ops / **docs-output** / **testing**） | 3つ（safety / role-detection / autonomous-ops） | **Codex版に docs-output と testing が追加** |
| 後回しの rules | 4つ（github-ops / release-ops / cost-management / incident-response） | 3つ（**testing** / github-ops / docs-output） | **testing と docs-output の優先度が逆** |
| safety.md の停止条件 | 一般的な記述 | **`3回連続同一エラー`・`target_paths外の編集が必要になった場合`** を明記 | **Claude版のみ**具体的な停止トリガーを列挙 |
| **skills** | | | |
| 最初に必要 | 6つ（triage / make-task-packet / task-handoff / review-gate / **failure-classifier** / dashboard） | 4つ（triage / make-task-packet / task-handoff / dashboard） | **Codex版に failure-classifier と review-gate が追加** |
| 作り込み危険な skills | 4つ（auto-advance / issue-sync / worktree-execution / failure-classifier） | 3つ（auto-advance / issue-sync / **failure-classifier の auto-unblock 部分**） | Claude版は failure-classifier の危険部分を**auto-unblock に限定** |
| **agents** | | | |
| 作成するagent | 5つ（triage / executor / reviewer / **log-summarizer** / **research**） | 3つ（triage / executor / reviewer） | **Codex版に log-summarizer と research が追加** |
| 権限テーブル | 3列（主な役割 / 権限方針 / ─） | 5列（mode / 書ける場所 / やること / やらないこと） | **Claude版の方が詳細**（mode=plan/acceptEdits、書ける場所を具体的に） |
| context分離の理由 | 4項目箇条書き | 散文で3つの理由＋「混ぜると起きる問題」3点 | Claude版は問題を具体的に説明 |

### 7. .ops と task packet（§6）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| .ops テーブル | 2列（パス / 役割） | 3列（パス / 役割 / **主な読者**） | **Claude版のみ「主な読者」**列を追加 |
| task packet テーブル | 4列（ファイル / 何のため / 見る人 / いつ） | 4列（同じ）＋より詳しい説明 | Claude版は各ファイルの説明が**2〜3倍長い** |
| task packet 最低条件 | 6項目チェックリスト | なし（本文中に散在） | **Codex版のみ**独立したチェックリストとして集約 |
| DASHBOARD.md の位置付け | 「人間の中心画面」 | 「人間の中心画面」＋**「人間が日常的に開くのは DASHBOARD → BRIEF / REVIEW / blocked_reason / spec-question の4つだけ」** | **Claude版のみ**人間が読むファイルを限定的に明示 |
| HANDOFF.md の更新頻度 | 「区切りごと」 | **「完了条件を1つ潰すごと / 30分ごと / セッション終了時 / compact 前」** | **Claude版の方が具体的** |

### 8. 最初のタスク条件（§7）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| task化禁止条件 | 10項目 | 8項目 | Codex版に `target paths未決定`・`forbidden paths未決定`・`成功/失敗の判定方法がない`・`人間が仕様判断していない` の4つ追加。Claude版に `影響範囲が広すぎる（3ディレクトリ以上、完了条件7個以上）`・`失敗時に元に戻せない操作` の具体的閾値あり |
| proposed 条件 | 4項目チェックリスト＋7種類のタスク例 | 4種類のタスク例（具体的な依頼文付き） | Codex版はタスク種類が多い。Claude版は依頼文例が具体的 |
| ready 昇格条件 | 11項目（secrets/本番/課金/破壊的を個別列挙） | 7項目チェックリスト（afk_safe判定・依存タスクなしを含む） | **Codex版が細かく列挙**。Claude版は `afk_safe` と `依存する running タスクがない` を追加 |
| 初回向きタスク | 7項目（`blocked_reason.md テンプレート整備` を含む） | 6項目 | ほぼ同一 |
| 初回不向きタスク | 9項目（`技術選定を伴う実装`・`データ削除や移行` を含む） | 6項目 | **Codex版が多い** |

### 9. タスク開始までの人間ワークフロー（§8）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 全体形式 | 10ステップ。各ステップを小見出し＋箇条書き | 10ステップ。各ステップをテーブル形式 | Claude版はテーブルで統一されていて見やすい |
| Step 1 草案整理 | AIに「目的、範囲、未決定の抽出」を頼む | **「まだ頼まない」**。人間が自力で印を付ける | **大きな差異**。Claude版は「AIに構想を補完させない」ことを強調 |
| Step 2 完了条件 | 「未決定事項が隠れていない」 | 「5ファイルを**全文読んで**、草案との食い違い・勝手な補完がないと確認した」 | Claude版は**全文確認**を要求 |
| Step 4 CLAUDE.md | 「正本、役割、状態、停止条件が分かる」 | 「チェックリスト通過＋**自分が全文理解できた**」 | Claude版に**人間の理解**条件あり |
| Step 5 settings.json | 「Claude Codeの現在仕様を確認せず断定しない」 | 「起動でエラーなし＋**secrets 読み取りの deny を実地テストで確認**」 | **Claude版のみ実地テスト** |
| Step 6.5 スモークテスト | なし | **「ダミータスクで queue を1周させる」** | **Claude版のみ** |
| プロンプト参照 | セクション内にプロンプト例を直接記載 | 「§5-1のプロンプト例を使用」と参照 | Codex版はスタンドアロン、Claude版は DRY（重複排除） |

### 10. タスク開始後の人間ワークフロー（§9）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| ready→running 確認項目 | 6項目（BRIEF/PLAN/HANDOFF初期状態/target-forbidden/安全/AFK） | 3項目（DASHBOARD異常なし/target-forbidden-DoD最終確認/依存タスクなし） | **視点が違う**: Codex版=packet内確認、Claude版=全体状態確認 |
| Claude Code 起動場所 | worktree に cd して起動（`cd {worktree_path}`） | メインrepoで起動し、AIに worktree 作成させてから実行 | **Claude版はメインrepo起動を推奨**（worktree作成もAIに任せる） |
| 実行開始プロンプト | worktree内で起動。task packet を読んで実行 | メインrepoで起動。**worktree作成→packet移動→読み→実行** まで1プロンプト | Claude版の方が包括的 |
| RUNLOG の見方 | 4項目箇条書き＋「思考ログではなく実行事実のログ」 | 「Action→Result の積み上げ。**同一Action3回以上はループの兆候**」 | **Claude版のみ**具体的な異常判定基準 |
| HANDOFF の見方 | 5項目箇条書き＋「古い/曖昧/空の場合は放置しない」 | 「次の手が具体的なら健全。**『引き続き実装する』のような曖昧な記述は品質低下のサイン**」 | **Claude版のみ**品質低下の具体例 |
| 放置条件 | 5項目 | 4条件（`afk_safe: true` を条件に含む） | ほぼ同一 |
| 止める条件 | 7項目 | 5条件（**`同一エラー3回`・`HANDOFF 2時間未更新`** を含む） | Claude版は**時間・回数の具体閾値** |
| セッション切れ対応 | 4ファイル参照→再開プロンプト | 3ステップ＋**「慌てない。会話が飛んでも作業は飛ばない」** | Claude版に心理的安心の配慮 |
| 再開プロンプト | 詳細（HANDOFF古い場合の指示あり） | 簡潔（次の1手から再開＋食い違いは報告） | Codex版が詳細 |
| review の VERIFY 見方 | 「not-run の理由」「残存リスク」を含む | 「**not-run が理由なしに残っているなら merge しない**」 | **Claude版のみ**明確な merge 拒否基準 |
| merge 条件 | 7項目 | 3条件（approve＋全ok＋全pass/justified skip） | **Codex版が詳細**、Claude版は簡潔で明確 |
| merge拒否条件 | 6項目 | 4条件＋**「自分が説明を読んで理解できない変更は保留」** | **Claude版のみ** |
| blocked ready戻し | 5条件 | 3条件＋**「BRIEF/PLANが新しい決定と整合している」** | Claude版は整合確認を要求 |
| blocked 再開プロンプト | 詳細構造化 | 簡潔＋**「実装はまだ始めないでください」** | Claude版は安全の念押し |

### 11. 朝・就寝前・問題発生時（§10）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 朝の目安時間 | なし | **「目安: 10〜15分」** | **Claude版のみ** |
| 朝の開くファイル | 4つ列挙 | 3つ＋「DASHBOARDだけで全体把握」 | Claude版は DASHBOARD の集約力を強調 |
| 朝のやってはいけないこと | 4項目 | 3項目＋**「朝の判断時間に実装の細部に潜る」** | **Claude版のみ**の注意 |
| 就寝前の目安時間 | なし | **「目安: 15分」** | **Claude版のみ** |
| 就寝前の構成 | DASHBOARD確認→review→blocked→AFK safe→セッション起動→翌朝 の6セクション | 6段階の番号付きリスト | Claude版はコンパクト |
| AFK safe 条件 | 8項目チェックリスト | 1行で条件列挙＋**「迷ったら起動しない」** | 同等だが形式が違う |
| 就寝前プロンプト | AFK safe 確認プロンプト | **「afk_safe の判断に不安があるタスクは正直にそう書いてください」** | **Claude版のみ**AI に正直さを要求 |
| 問題発生時 | DASHBOARD→blocked_reason→HANDOFF→選択肢→再開プロンプト | DASHBOARD→blocked_reason→HANDOFF→選択肢＋**「全 running を blocked に移し、clean triage からやり直す」** | **Claude版のみ** clean triage オプション |
| 問題発生時プロンプト | 「人間が選べる選択肢」「推奨する次の一手」 | 「**私が取れる選択肢とそれぞれのリスク**」「**まだ何も実行しないでください**」 | Claude版はリスク説明を要求し安全重視 |

### 12. コピペ用プロンプト集（§11）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| プロンプト数 | **18個**（草案整理〜仕様変更） | **21個**（11-1〜11-21） | **Claude版が3個多い** |
| 各プロンプトの形式 | 使う場面 / 読ませるファイル / 期待する出力 / 注意点＋プロンプト本体 | 使う場面 / 読ませるファイル / 期待する出力 / 注意点＋プロンプト本体＋前セクション参照 | Claude版は前セクションへの参照で DRY |
| Claude版にのみある | ─ | **11-14 HANDOFF更新** / **11-17 stale running救済** / **11-18 AFK safe選択** / **11-19 翌朝確認** / **11-20 仕様変更投入** / **11-21 done後片付け** | Claude版は運用フェーズのプロンプトが充実 |
| Codex版にのみある | `.ops` 初期化プロンプト / task packet雛形プロンプト（独立セクション） | ─ | Codex版は構造作成プロンプトが独立 |
| HANDOFF更新プロンプト | あり（7項目指定） | あり（6項目指定＋**「更新後、内容をそのまま表示してください」**） | Claude版は出力確認を要求 |
| review プロンプト | 詳細（読むもの7つ列挙） | 簡潔＋**「非エンジニア向け1〜3行の説明付き」** | Claude版は非エンジニア配慮 |
| 仕様変更プロンプト | あり（inbox投入＋impact調査） | あり＋**「continue / suspend / discard_and_replan に分類」**＋**「00-docs更新案と decisions.md記録案も提示」** | **Claude版が詳細** |
| done後の片付け | なし | **11-21 として独立**（merge手順提示→確認→worktree/branch削除→DASHBOARD更新） | **Claude版のみ** |
| 共通注意書き | なし | **「どのプロンプトも1セッション1目的。終わったらセッション切り替え」** | **Claude版のみ** |

### 13. 最初の1週間モデルプラン（§12）

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 1日あたりの想定 | 明示なし | **「人間の作業時間は30〜60分」** | **Claude版のみ** |
| 成果の定義 | 「ハーネス運用の土台」 | **「アプリのコード」ではなく「安全に回る運用」** | Claude版は対比で明確化 |
| Day 1 | 草案整理。AIに抽出を頼む | 草案整理。**AIに原則頼まない**（「メモの重複整理」程度） | **大きな差異**。Claude版はDay1をほぼ人間作業に |
| Day 3 | CLAUDE.md のみ | **CLAUDE.md ＋ .ops 作成**（セッションB+C を同日、セッション分離） | Claude版は Day3 に .ops を組み込み、Day5 を空ける |
| Day 5 | `.ops / task packet 雛形` | **agents / skills / task packet雛形 ＋ ダミータスクで配管確認** | Claude版はスモークテストを Day5 に組み込み |
| Day 3-5 の順番 | Day3=CLAUDE.md → Day4=settings/rules → Day5=.ops | Day3=.ops+CLAUDE.md → Day4=settings/rules → Day5=agents/skills+smoke | **順番が異なる**。Codex版は .ops を Day5 に後回し |
| Day の形式 | 5項目箇条書き | テーブル（目的/見るファイル/AIに頼む/完了条件/やってはいけないこと） | Claude版はテーブルで統一 |
| 延長方針 | 5項目箇条書き | 5項目＋**並列度の段階的引き上げ条件**（1→2→3、review滞留なしが条件） | **Claude版のみ**並列度の拡大基準 |
| 延長の具体ポイント | 「settings.json が不安なら Day4 延長」 | 「詰まりやすいのは Day2（未決定が多い）と Day4（settings実仕様確認）。**ここは2日かけてよい**」 | Claude版は詰まりやすい箇所を予測 |

### 14. チェックリスト / 付録

| 項目 | Codex版 | Claude版 | 差異 |
|---|---|---|---|
| 最小チェックリスト | **§13 として独立セクション**。タスク開始前14項目 / ready昇格前9項目 / セッション終了前7項目 / 絶対人間確認8項目 | なし（各セクション内に分散） | **Codex版のみ**チェックリストを集約 |
| 困った時の早見表 | なし | **付録として8症状×3列の早見表**（症状/見るファイル/取る行動） | **Claude版のみ** |
| 正本への言及 | なし | **末尾に「設計書を優先し、本ガイドを更新する」旨を明記** | **Claude版のみ** |

---

## まとめ: 主要な差異一覧

### Claude版にあって Codex版にないもの

1. **スモークテスト（セッションB2）** — ダミータスクで配管を先に確認
2. **セッションEの3分割推奨**（E1 rules → E2 agents → E3 skills）
3. **「追加で必要な準備」セクション** — git初期化、worktree置き場、secrets隔離
4. **done後の片付けプロンプト**（merge→worktree/branch削除→DASHBOARD更新）
5. **強制力の概念**（context / 条件付き注入 / 強制）
6. **deny の実地テスト**（settings.json確認時）
7. **時間・回数の具体閾値**（HANDOFF 2時間未更新=stale、同一エラー3回=ループ、朝10-15分、就寝前15分）
8. **困った時の早見表**（付録）
9. **並列度の段階的引き上げ基準**
10. **Day1 を「AIに頼まない」人間中心の日に設定**
11. **1セッション=15〜30分で確認しきれる量** の目安
12. **仕様変更のimpact分類**（continue / suspend / discard_and_replan）
13. **「理解できない憲法は運用できない」** という人間理解の要求
14. **正本（設計書）との優先関係の明記**

### Codex版にあって Claude版にないもの

1. **ステップ0「安全な作業場所を決める」**
2. **後処理優先の独立セクション**（処理順7段階＋新規着手条件5項目）
3. **最小チェックリスト集約セクション**（§13）— 4場面38項目
4. **初期 rules に docs-output.md と testing.md を含む**
5. **初期 skills に failure-classifier と review-gate を含む**
6. **初期 agents に log-summarizer と research を含む**
7. **追加セッションとして hooks実装 / 初回review / secrets棚卸し / GitHub連携 を明示**
8. **task packet 最低条件の6項目チェックリスト**
9. **proposed の条件チェックリスト**（4項目）
10. **ready 昇格条件に secrets/本番/課金/破壊的を個別列挙**（11項目）
11. **初回に向いていないタスクが多い**（9項目 vs 6項目）
12. **対象読者に他エージェント（Codex/Antigravity/Cursor）への言及**
13. **`.ops` 初期化と task packet 雛形のプロンプトが独立**

---

## どちらを使うべきか

| 用途 | 推奨 | 理由 |
|---|---|---|
| 初めてハーネスを導入する | **Claude版** | 理由の説明が厚く、判断に迷いにくい。スモークテスト推奨が安全。時間目安がある |
| チェックリストとして手元に置く | **Codex版** | §13 の集約チェックリストが実務で便利。箇条書き主体で検索しやすい |
| コピペ用プロンプト集として使う | **Claude版** | プロンプト数が多く（21個）、運用フェーズ（done後片付け・仕様変更・翌朝確認）までカバー |
| 細かい初期構成を漏れなく作る | **Codex版** | 初期 rules/skills/agents の候補が多い。後処理優先の条件が独立セクションで参照しやすい |
| 統合版を作る | **両方の強みを組み合わせ** | Claude版をベースにしつつ、Codex版の §13 チェックリスト・後処理優先セクション・追加 skills/agents を補完する |
