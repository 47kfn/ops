# harness_design-D_unified.md 批評レビュー

**レビュー日**: 2026-04-08
**対象**: harness_design-D_unified.md (2214行, 2026-04-06作成)
**レビュー手法**: ハーネスエンジニアリングに関する公開記事・一次情報を取得し、それらの知見に照らして設計書を評価

---

# 0. URL取得結果一覧

## 英語一次情報

| URL | 状態 | 備考 |
|-----|------|------|
| Martin Fowler: Harness Engineering | **取得成功** | 最重要ソース。feedforward/feedback、computational/inferential分類、3つのregulation category |
| Anthropic: Harness Design Long-Running Agents | 取得失敗 (404) | URLが無効 |
| Anthropic: Effective Harnesses | 取得失敗 (404) | URLが無効 |
| OpenAI: Harness Engineering Codex | 取得失敗 (403) | アクセス拒否 |
| Mitchell Hashimoto: AI Adoption Journey | 取得失敗 (404) | URLが無効 |
| Artificial Ignorance: Harness Engineering Playbook | 取得失敗 (ECONNREFUSED) | サーバー接続不可 |
| LangChain: Improving Deep Agents | 取得失敗 (リダイレクト先404) | ブログドメイン移行中 |

## 英語補足記事

| URL | 状態 | 備考 |
|-----|------|------|
| Louis Bouchard: Harness Engineering | **取得成功** | 3層区別(prompt/context/harness)、state外部化、harness debt |
| HumanLayer: Skill Issue | 取得失敗 (404) | |
| Phil Schmid: Agent Harness | 取得失敗 (404) | |
| Rick Hightower: Harness vs Context | 取得失敗 (404) | Medium記事削除 |
| nxcode.io: Complete Guide | 取得失敗 (404) | |
| Simon Willison: Codex | 取得失敗 (404) | |
| Medium/Spillwave: AI Coding Hangover | 取得失敗 (404) | Medium記事削除 |
| Martin Fowler: Humans Agents Loops | 取得失敗 (404) | |
| Martin Fowler: Context Engineering | 取得失敗 (404) | |
| OpenAI: Codex Harness App Server | 取得失敗 (403) | |
| Epsilla: Harness Engineering | 取得失敗 (404) | |
| Cobus Greyling: Rise of AI Harness | 取得失敗 (404) | Medium記事削除 |

## 日本語記事

| URL | 状態 | 備考 |
|-----|------|------|
| Qiita検索: ハーネスエンジニアリング | **取得成功** | 104件ヒット、主要20記事リスト取得 |
| Zenn: theaktky (コードレビューをやめる) | **取得成功** | 自動レビューワークフロー、severity分類、オシレーション検出 |
| Zenn: shio_shoppaize (Git Workflowでは) | **取得成功** | 批判的分析。3層構造、Git対応表、未踏領域の指摘 |
| Qiita: nogataka (入門) | **取得成功** | 5要素定義、3回ルール、段階的導入、ECC事例 |
| Qiita: miruky (次のパラダイム) | **取得成功** | 5設計原則、4層フィードバック、技術的負債増幅、JSON管理推奨 |
| Qiita: yurukusa (700時間実践) | **取得成功** | 3層設計、ドリフト防止、コンテキスト監視、641 hooks |
| Qiita: Aochan0604 (新しいのか) | **取得成功** | Fowler批評、過剰設計逆効果、Vercel80%削除、ベンダーロック懸念 |
| Qiita: cvusk (実践ガイド) | **取得成功** | 10トピック網羅、Spotify事例、Vercel事例、エージェントフレンドリー設計 |
| Qiita: isanakamishiro2 (解剖学) | **取得成功** | ハーネス構成要素分類、長期実行パターン |
| Qiita: s-age (3概念比較) | **取得成功** | prompt/context/harness の関係性整理 |
| blog.est.co.jp | **部分取得** | 記事タイトルのみ確認 |
| note.com検索 | 取得失敗 | JS依存で検索結果レンダリングされず |
| Zenn検索 | 取得失敗 | JS依存で検索結果レンダリングされず |
| SpeakerDeck検索 | 未取得 | スライドのテキスト取得困難 |

**取得成功: 11件 / 部分取得: 1件 / 取得失敗: 21件**

---

# 1. 全体評価

## 1-1. 設計書の強み

D案は、A/B/C案の統合として高い完成度を持つ。特に以下の点は取得した記事群の推奨事項と強く合致している:

- **Queue State Machine + Task Packet**: Fowlerの「feedforward guide」に相当。エージェントの行動空間を構造で制約している
- **1 Task = 1 Worktree = 1 Branch**: 隔離原則はOpenAI/Anthropic双方が推奨する「分離」プリミティブの忠実な実装
- **Handoff-first Recovery**: miruky記事が強調する「セッション間一貫性」の中核設計
- **Failure Classification**: Fowlerの「computational sensor」の実装。機械的分類は正しい方向
- **deny-first Permissions**: 全記事が一致して推奨する最小権限原則
- **非ツール依存骨格 + Claude Code全開**: 二層構造は将来対応として理想的

## 1-2. 全体的な懸念

取得した記事群から繰り返し指摘されている問題が、D案にもそのまま当てはまる:

1. **過剰設計のリスク**: Aochan0604記事が引用するFowlerの批評「Vercelはツール80%削除で精度向上」。D案は初日から8 skills + 5 subagents + 5 hooks + 10ドキュメント/タスクを要求する
2. **振る舞い検証の欠落**: Fowlerが最大の弱点として指摘する「Behavior harness」がD案に存在しない
3. **ハーネスドリフト管理の不在**: Bouchard記事が警告する「harness debt」への対策が設計書にない
4. **フィードバックループの弱さ**: cvusk記事が強調する4層フィードバック（構文→ロジック→統合→CI）の設計が薄い

---

# 2. 要らない設定（優先順位順）

## 優先度: 高

### 2-1. DECISIONS.md と RISKS.md の分離（→ 統合すべき）

**根拠**: cvusk記事のSpotify事例「ツールを3つに絞って成功」、Aochan0604記事「Vercelはツール80%削除で精度向上」。

- Task Packetの10ファイル構成は過剰。DECISIONS.mdとRISKS.mdは独立ファイルにする必要がない
- 判断記録はPLAN.mdの変更履歴セクションで十分カバーされる
- リスクはBRIEF.mdの一部として記載すれば足りる
- **ファイル数を減らすことでエージェントの認知負荷を減らす** ことが全記事共通の知見

### 2-2. issue.md（→ 削除すべき）

**根拠**: shio_shoppaize記事「基本設定は誰でも実装可能だが、真のエンジニアリングは足りない部分の設計から始まる」。

- issue.mdはBRIEF.mdとほぼ重複する
- issue-sync skillがtask.yaml + BRIEF.mdからIssueを自動生成すれば済む
- 同じ情報の2箇所管理は同期コストだけ増える

### 2-3. log-summarizer subagent（→ 不要）

**根拠**: nogataka記事「過剰フックは検証の嵐でエージェント応答が極端に遅延」。

- failure-classifier skillが既にログ解析を含む
- research subagentのログ探索機能と責務が重複
- 別subagentにする正当性が薄い。failure-classifierに統合すべき

## 優先度: 中

### 2-4. 4層構造の Layer 0（可視化・追跡層）の独立定義

**根拠**: 実際にはGitHub Projects/Issues は Layer 1〜3 の写像であり、独立した層ではない。

- 「4層」と言いつつ Layer 0 は他層の可視化に過ぎず、設計上の判断を駆動しない
- 3層（計画/運用/実行）+ 外部UI(GitHub) とした方が概念的に正確

### 2-5. `.ops/reports/daily/` ディレクトリ

- 日次サマリーの具体的な生成タイミング・内容・消費者が未定義
- TRIAGE_REPORTが事実上の日次レポートの役割を果たす
- 定義だけあって使い方が書かれていない設計は負債になる

---

# 3. 追記すべき設定（優先順位順）

## 優先度: 最高

### 3-1. フィードバックループの設計（Feedback Loop Architecture）

**根拠**: Martin Fowler「feedforward + feedback の両方が必要」、cvusk「4層フィードバックがハーネスの品質を決定する」、miruky「強い型付け言語はコンパイル時にエージェントのミスを捕捉」。

D案はfeedforward（CLAUDE.md, permissions, hooks, task packet）は充実しているが、**feedback（実行結果からの自動修正ループ）が構造的に設計されていない**。

**追記すべき内容**:

```
## フィードバックループ設計

### 4層フィードバック

| 層 | 対象 | レイテンシ | 実装 |
|----|------|-----------|------|
| 第1層: 構文 | 型エラー・lint | 数秒 | PostToolUse hook でlint自動実行 |
| 第2層: ロジック | ユニットテスト | 数分 | 実装ステップごとにテスト実行 |
| 第3層: 統合 | E2E・compose | 数分〜数十分 | review-gate前の統合テスト |
| 第4層: CI | GitHub Actions | 数十分 | PR作成時の自動検証 |

### ルール
- 層が浅いほど頻繁に回す
- 第1層は毎回のEdit/Write後に自動実行
- 第2層は実装ステップ完了ごと
- 上位層の失敗は下位層のフィードバック不足を示唆
```

### 3-2. 振る舞い検証（Behavior Harness）

**根拠**: Martin Fowler「内部品質（maintainability）は整備されても、ユーザーニーズ充足（behavior）は未検証。これがハーネスの最大の弱点」。

D案のVERIFY.mdはlint/typecheck/unit tests/integration testsの検証を列挙するが、**「完了条件（definition_of_done）が本当にユーザー要求を満たしているか」の検証方法が構造化されていない**。

**追記すべき内容**:

```
## 振る舞い検証

### 問題
- テストがpassしても、仕様の意図と実装が乖離する場合がある
- AI生成テストは「実装を正当化するテスト」になりやすい

### 対策
1. definition_of_doneの各項目に「検証方法」を明示的に紐づける
2. Review Agentに「テストが仕様の意図をカバーしているか」の評価軸を追加
3. 人間がdefinition_of_doneを書く段階で、検証可能な形式にする
   - Bad: "正常に動作する"
   - Good: "curl localhost:3000/health が200を返す"
```

### 3-3. ハーネスドリフト管理（Harness Maintenance）

**根拠**: Louis Bouchard「harness debt becomes real—systems develop their own bugs and drift」、Martin Fowler「Controls become misaligned and contradictory over time. No standard tooling yet to manage harness coherence」、nogataka「ハーネス自体の鮮度管理が必須（90日間隔推奨）」。

D案はハーネスの構築方法を詳細に定義しているが、**ハーネス自体が陳腐化した時の対応**が抜けている。

**追記すべき内容**:

```
## ハーネスドリフト管理

### 問題
- skills / hooks / rules / CLAUDE.md が実態と乖離する
- scoring-weights.yaml が現在の優先度と合わなくなる
- templates が実際の運用パターンと離れる

### 対策
1. 90日ごとにハーネス自体のレビューを実施
2. 以下をチェック:
   - 使われていない skill がないか
   - hook が不要なブロックをしていないか
   - CLAUDE.md の記述が現状と矛盾していないか
   - scoring-weights の配分が適切か
3. 「3回ルール」の適用:
   - 同じルール違反が3回 → ルールをhookに昇格
   - 同じhookが3回誤ブロック → hookを緩和または削除
```

## 優先度: 高

### 3-4. コンテキスト管理戦略

**根拠**: cvusk記事「コンテキストの95%に達したら自動/手動コンパクション実行」、yurukusa記事「コンテキスト監視: 40%警告、20%強制実行」、miruky記事「1セッション1機能が核心ルール」。

D案は `/compact` の使用に言及しているが、**コンテキスト管理の具体的な閾値・手順が設計されていない**。

**追記すべき内容**:

```
## コンテキスト管理

### 閾値
- 40%残: /compact を推奨（HANDOFFを先に更新）
- 20%残: 強制的に /compact + HANDOFF更新
- 10%残: セッション切り替え（新session + HANDOFFから再開）

### 1セッション1機能の原則
- 1つのtaskの中でも、機能単位でセッションを分ける
- セッションの切れ目 = git commit + HANDOFF更新
```

### 3-5. ツール数の最適化指針

**根拠**: cvusk「Spotifyは3つのツール（検証、Git、Bash）で1,500件以上のPRをマージ」「ツール定義だけでコンテキストウィンドウの5-7%を消費」、Aochan0604「Vercelはツール80%削除で精度向上」。

D案は8 skills + 5 subagentsを定義しているが、**全てを同時にロードすべきでない**という指針が欠けている。

**追記すべき内容**:

```
## ツールロード戦略

### 原則
- 全skills/subagentsを同時にロードしない
- 役割に応じて最小限のツールセットを選択

### 役割別ロードセット
| 役割 | ロードする skills | ロードする subagents |
|------|-------------------|---------------------|
| Triage | triage-from-state, impact-analyzer | research |
| Execution | worktree-execution, failure-classifier, task-handoff | (なし - メインで実行) |
| Review | review-gate | reviewer |

### 計測
- /context で定期的にトークン消費量を確認
- skills定義が全体の5%を超えたら分割を検討
```

### 3-6. テスト駆動の実装ガイダンス

**根拠**: cvusk「テストが仕様書。エージェントに『すべてのテストを通す実装を書いて』と指示」「テスト速度はコスト。テスト実行速度がフィードバックループの品質に直結」。

**追記すべき内容**:

```
## テスト駆動実行

### PLAN.mdへの追記項目
- テストの実行コマンド（1行で完結するもの）
- 期待する実行時間
- テストカバレッジの最低ライン（存在する場合）

### Execution Agentの行動ルール
1. 既存テストを先に実行し、現状を把握
2. 実装前にテストを書く（可能な場合）
3. 実装ステップごとにテスト実行
4. 全テストpassを確認してからreview-gateへ
```

## 優先度: 中

### 3-7. エージェントフレンドリーなコードベース設計の推奨事項

**根拠**: cvusk「ORM より素のSQL」「明示的な依存関係管理」「シンプルで直接的なコード」。miruky「DDD のレイヤー分け」「命名規則の統一」。

D案はハーネスの設計書であり、コードベースの設計を規定する立場ではないが、CLAUDE.mdに**エージェントにとって作業しやすいコード設計の推奨事項**を記載する指針を含めるべき。

### 3-8. Progressive Disclosure（段階的情報開示）の設計

**根拠**: cvusk「プログレッシブディスカバリー: 段階的にツール提示でコンテキスト管理」、yurukusa「段階的情報開示で認知負荷軽減」。

CLAUDE.mdの設計で「地図として機能させ、詳細はskillsに分離」の方針を明文化すべき。

### 3-9. コスト監視の具体化

**根拠**: yurukusa「コンテキスト監視」、cvusk「モデルルーティング: トラフィック40%を安価モデルにルーティング→40-60%コスト削減」。

D案の「19-3. コスト管理」は方針のみ。`/usage` のチェック頻度、閾値、対応方法を具体化すべき。

---

# 4. 代替案（優先順位順）

## 優先度: 高

### 4-1. Task Packetの軽量化（10ファイル → 6ファイル）

**根拠**: Spotify「3ツールで成功」、Vercel「80%削減で改善」、Aochan0604「過剰設計の逆効果」。

**現行 (10ファイル + artifacts)**:
```
task.yaml, BRIEF.md, PLAN.md, HANDOFF.md, VERIFY.md,
RESULT.md, REVIEW.md, RUNLOG.md, DECISIONS.md, RISKS.md, issue.md
```

**提案 (6ファイル + artifacts)**:
```
task.yaml       # メタデータ（変更なし）
BRIEF.md        # 背景・目的・完了条件・リスク（RISKS.md統合）
PLAN.md         # 実行計画・判断記録（DECISIONS.md統合、変更履歴で記録）
HANDOFF.md      # 再開点（変更なし、最重要）
VERIFY.md       # 検証結果（変更なし）
LOG.md          # RUNLOG + RESULT + issue化メモを統合（時系列で最終サマリーも含む）
```

- REVIEW.mdは Review Agent の出力として `.ops/runs/` に保存する（task packet内に置かない）
- DECISIONS.mdは PLAN.md の変更履歴セクションに統合
- RISKS.mdは BRIEF.md のリスクセクションに統合
- RESULT.mdは LOG.md の最終エントリとして統合
- issue.mdは issue-sync skill による自動生成に置き換え

### 4-2. Hooks設計の再構成: 計算的(Computational) vs 推論的(Inferential)の分離

**根拠**: Martin Fowler「Computational controls は deterministic で安価、Inferential controls は non-deterministic で高価。この区別がハーネス設計の核」。

D案のhooksは全て bash script（computational）だが、**何がcomputational controlで何がinferential controlかの設計判断が明示されていない**。

**提案**:

```
## Computational Controls (hooks - 必ず実行、高速、決定論的)
- PreToolUse: secrets遮断、worktree外遮断、deploy遮断
- PostToolUse: diff stat記録
- Stop: HANDOFF更新チェック

## Inferential Controls (skills/subagents - 必要時のみ、低速、確率的)
- failure-classifier: エラー分類（LLM判断）
- review-gate: 品質チェック（LLM判断）
- impact-analyzer: 影響分析（LLM判断）

## 設計原則
- Computational controlsは全ての実行で走る（コスト低）
- Inferential controlsは特定イベント時のみ走る（コスト高）
- 同じチェックをhookとskillの両方でやらない
```

### 4-3. subagent構成の見直し: Writer/Reviewer パターンの明確化

**根拠**: cvusk「Writer/Reviewer パターン：実装と新鮮なコンテキストでの検証」「並列スペシャリスト（セキュリティ、スタイル、パフォーマンス同時実行）」。

D案の5 subagents (triage, executor, reviewer, log-summarizer, research) を整理:

**提案 (4 subagents)**:
```
triage      # 変更なし
executor    # 使い方を再定義: subagentではなく独立sessionとして起動
reviewer    # 変更なし
research    # log-summarizer の機能を吸収
```

理由: executorは「subagentとして他のsessionから呼ばれる」使い方より「独立sessionとしてworktreeで直接起動」が実態に合う。subagent定義にexecutorがあるとcontextを無駄に消費する。

## 優先度: 中

### 4-4. task.yamlの priority フィールド: 加算スコアから重み付きスコアへ

**根拠**: cvusk「LangChainはモデル変更なしでハーネス最適化だけで30位→5位に改善」。スコアリングの精度がtriageの品質を決定する。

現行の単純加算 `score = impact + urgency + unblock_value + effort + confidence + leverage` は、全軸が等しい重みを持つ前提。しかし実運用ではフェーズによって重視する軸が変わる。

**提案**: scoring-weights.yaml に軸ごとの重み係数を追加:

```yaml
weights:
  phase-1:  # 立ち上げ期: impact重視
    impact: 1.5
    urgency: 1.0
    unblock_value: 1.2
    effort: 0.8
    confidence: 1.0
    cross_project_leverage: 0.5
  phase-2:  # 安定期: unblock重視
    impact: 1.0
    urgency: 1.0
    unblock_value: 1.5
    effort: 1.0
    confidence: 0.8
    cross_project_leverage: 1.0
```

### 4-5. ヘルスチェックの自動化レベル引き上げ

D案のヘルスチェック（セクション18）は「Triage Agentがセッション開始時に確認」とあるが、これは**手動トリガーの確認リスト**に過ぎない。

**提案**: ヘルスチェックの一部をcomputational control（bash script）として自動化:

```bash
#!/bin/bash
# .ops/hooks/health-check.sh (UserPromptSubmit で実行)

# stale running check
for dir in .ops/queue/running/*/; do
  if [ -d "$dir" ]; then
    handoff="$dir/HANDOFF.md"
    if [ -f "$handoff" ]; then
      last_mod=$(stat -f %m "$handoff" 2>/dev/null)
      now=$(date +%s)
      if [ $((now - last_mod)) -gt 86400 ]; then
        echo "WARNING: $(basename $dir) のHANDOFF.mdが24時間以上更新されていません"
      fi
    fi
  fi
done

# blocked accumulation check
blocked_count=$(ls -d .ops/queue/blocked/*/ 2>/dev/null | wc -l)
if [ "$blocked_count" -gt 5 ]; then
  echo "WARNING: blocked タスクが${blocked_count}件蓄積しています"
fi
```

### 4-6. CLAUDE.md の構造: 「地図」としての設計

**根拠**: yurukusa「CLAUDE.mdを地図として機能させる。参照情報はSkillsに分離」、nogataka「簡潔性の原則: すべての行について『これを削除したらエージェントがミスをするか』と問え」。

D案のCLAUDE.md（セクション6-2）は網羅的だが、**地図としての機能が弱い**。エージェントが「何を見ればいいか」を最速で判断できる構造にすべき。

**提案**: CLAUDE.mdの冒頭に「ルーティングテーブル」を追加:

```markdown
## Quick Reference
| やりたいこと | 見るべき場所 |
|-------------|-------------|
| 次のタスクを決める | /triage-from-state skill |
| タスクを実装する | .ops/queue/ready/{task}/BRIEF.md → PLAN.md |
| 中断したタスクを再開する | .ops/queue/running/{task}/HANDOFF.md |
| 失敗を分類する | /failure-classifier skill |
| レビューする | /review-gate skill |
```

---

# 5. 構造的な批評

## 5-1. 「初日から完成形」vs「段階的導入」の矛盾

**根拠**: nogataka「全部入り導入は応答遅延・管理負荷増加のアンチパターン」、miruky「Day 1: CLAUDE.md作成（30分程度）から始める」、shio_shoppaize「基本設定は誰でも実装可能。真のエンジニアリングは足りない部分の設計」。

D案は「段階的アプローチではなく完成形として定義」（セクション23）と明言しつつ、導入チェックリスト（Step 1-5）で事実上の段階を設けている。この矛盾は意図的なものと理解できるが、**設計書を読む人間にとって「どこから始めるべきか」の判断基準が不足している**。

**推奨**: 「完成形の設計」と「導入順序」を分離して明記する。セクション23の冒頭に以下を追加:

> この設計は完成形として定義されている。ただし、全要素を初日から完全に機能させる必要はない。最低限の動作構成（Minimum Viable Harness）は以下:
> - CLAUDE.md + settings.json + .ops/queue/ + task.yaml + HANDOFF.md
> 他の要素は運用中に「3回ルール」（同じ問題が3回発生したら対応する仕組みを追加）で段階的に有効化する。

## 5-2. 「過渡的性質」への対応不足

**根拠**: Aochan0604記事のFowler引用「Bitter Lesson原則により、モデル進化で不要化される可能性を『剥がせる設計』で想定」。

D案のセクション0-3（進化前提の設計姿勢）は進化への対応を表で示しているが、**「どの要素が剥がせるか」の明示がない**。セクション24の「やるが置き換わるもの」は方向性として正しいが、具体的な「剥がし方」が書かれていない。

**推奨**: 各要素に「剥がし条件」を明記:

```
| 要素 | 剥がし条件 |
|------|-----------|
| hooks による安全装置 | auto permission mode が3ヶ月間誤検知なしで稼働した場合 |
| subagent 手動定義 | Agent Team が公式サポートされ、同等の権限分離が可能な場合 |
| 手動session起動 | 永続エージェントが24時間安定稼働した実績がある場合 |
```

## 5-3. Computational vs Inferential の未分離

**根拠**: Martin Fowler の harness engineering 記事の核心概念。

D案は hooks（computational）と skills/subagents（inferential）を別セクションで定義しているが、**この2つが「ハーネスの2つの制御手段」であるという設計原則が明示されていない**。結果として、何をhookにし何をskillにするかの判断基準が読者に伝わらない。

## 5-4. 「ベンダーロックイン」への言及不足

**根拠**: Aochan0604「Docker調査で76%がAIエージェント実装でのロックイン懸念」。

D案は「非ツール依存の骨格」を強調しているが、**Claude Code固有部分（skills, hooks, subagents, permissions）の「adapter分離」が設計レベルで具体化されていない**。

例: skills の SKILL.md フォーマットは Claude Code 固有。将来 Codex に移行する場合、どのファイルを書き換え、どのファイルはそのまま使えるかが不明。

**推奨**: セクション0-1に「ポータビリティマップ」を追加:

```
| 要素 | ポータビリティ | 移行時の作業 |
|------|-------------|-------------|
| queue/ + task.yaml | 完全ポータブル | なし |
| HANDOFF.md等ドキュメント | 完全ポータブル | なし |
| CLAUDE.md | 書き換え必要 | 対象ツールの設定形式に変換 |
| .claude/skills/ | 書き換え必要 | 対象ツールのプラグイン形式に変換 |
| .claude/agents/ | 書き換え必要 | 対象ツールのagent形式に変換 |
| hooks | 部分ポータブル | bash scriptは再利用可、トリガー定義のみ変換 |
| settings.json | 書き換え必要 | 対象ツールの権限モデルに変換 |
```

## 5-5. 「AIの自己検証問題」への対応

**根拠**: yurukusa「AIの自己評価は過度に肯定的（自己検証不可）」、Martin Fowler「AI-generated tests pass ≠ correctness guarantee」、theaktky「生成者と評価者の分離」。

D案はReview Agentを分離しているが、**Execution Agentが自分で書いたテストを自分で実行して「pass」と報告する構造は自己検証問題を解決していない**。

**推奨**: VERIFY.md に「テスト作成者」フィールドを追加し、AI生成テストと人間作成テストを区別する:

```markdown
| 検証項目 | 結果 | テスト作成者 | 備考 |
|---|---|---|---|
| unit tests (既存) | pass | human | |
| unit tests (新規) | pass | agent | ← Review Agentが重点チェック |
```

---

# 6. 記事横断で見えた設計書に反映すべき知見

## 6-1. 「3回ルール」の組み込み

**出典**: nogataka記事。

ルール違反や問題の再発に対する段階的エスカレーションの仕組み:
- L1（ドキュメント）→ L2（AI検証）→ L3（ツール検証 = hook）→ L4（構造テスト = CI）
- 同じ問題が3回発生したら次のレベルに昇格

D案にはこの概念がない。ハーネスの進化メカニズムとして組み込むべき。

## 6-2. JSON vs Markdown の使い分け

**出典**: miruky記事「MarkdownではなくJSON形式でリストを管理。エージェントによる不適切な変更が起こりにくい」、nogataka「Markdownメモリの無制限編集はエージェント改変リスク」。

D案ではtask.yamlは構造化されているが、**HANDOFF.md等のMarkdownファイルはエージェントによる意図しない編集（セクション削除、構造崩壊）のリスクがある**。

**推奨**: 機械が読み書きするフィールド（status, priority, test results）はYAML/JSON、人間が読むナラティブ（背景、判断理由）はMarkdownという使い分け原則を明記。

## 6-3. オシレーション検出

**出典**: theaktky記事「振り子修正パターンを認識し固定化」。

エージェントが同じ問題に対して A→B→A→B と修正を繰り返す「オシレーション」の検出機構がD案にない。failure-classifierに「同一ファイルへの3回以上の往復修正を検出したらblocked」のルールを追加すべき。

## 6-4. Explore → Plan → Implement パターンの明文化

**出典**: cvusk記事。

D案の実行ワークフロー（セクション14-1）は plan mode → acceptEdits の切り替えに言及しているが、**「Explore（現状把握）→ Plan（計画策定）→ Implement（実装）」の3フェーズを構造として明示していない**。PLAN.mdの「事前確認」セクションがExploreに相当するが、独立フェーズとして認識されていない。

---

# 7. 総合評価

## 設計書のスコア

| 評価軸 | 評価 | コメント |
|--------|------|---------|
| 網羅性 | ★★★★★ | 全要素を詳細に定義。2214行の完成度 |
| 実用性 | ★★★☆☆ | 初日から全部入りは現実的に厳しい。MVH(Minimum Viable Harness)の定義が必要 |
| 安全設計 | ★★★★☆ | deny-first、hooks、permissionsは充実。自己検証問題への対応が弱い |
| 進化対応 | ★★★★☆ | 非ツール依存骨格は正しい。「剥がし方」の具体性が不足 |
| フィードバック設計 | ★★☆☆☆ | feedforwardは強いがfeedbackが構造化されていない。最大の改善点 |
| メンテナンス性 | ★★☆☆☆ | ハーネスドリフト管理、3回ルール、鮮度管理が欠如 |
| コンテキスト効率 | ★★★☆☆ | tools/filesが多すぎる懸念。ロード戦略が未定義 |

## 最優先の改善3項目

1. **フィードバックループの4層設計を追加** — feedforward偏重を是正する最重要項目
2. **Task Packetの軽量化（10→6ファイル）** — エージェントの認知負荷を減らす
3. **ハーネスドリフト管理と3回ルールの追加** — ハーネス自体の持続可能性を担保

---

# 8. 参考文献

取得に成功した記事の一覧（本レビューの根拠）:

1. Martin Fowler, "Harness Engineering", martinfowler.com — feedforward/feedback分類、computational/inferential、3 regulation categories
2. Louis Bouchard, "Harness Engineering", louisbouchard.ai — 3層区別、harness debt、anti-patterns
3. @theaktky, "ハーネスエンジニアリングで人間のコードレビューをやめる", Zenn — 自動レビューWF、オシレーション検出、severity分類
4. @shio_shoppaize, "ハーネスエンジニアリング、それGit Workflowをbashで書き直してるだけでは", Zenn — 批判的分析、3層構造、未踏領域
5. @nogataka, "ハーネスエンジニアリング入門", Qiita — 5要素、3回ルール、段階導入、ECC事例
6. @miruky, "ハーネスエンジニアリングとは何か", Qiita — 5設計原則、4層フィードバック、JSON推奨
7. @yurukusa, "700時間の自律運用で実践している話", Qiita — 3層設計、ドリフト防止、コンテキスト監視
8. @Aochan0604, "ハーネスエンジニアリングは本当に新しいのか", Qiita — Fowler批評、Vercel 80%削除、ベンダーロック
9. @cvusk, "ハーネスエンジニアリング実践ガイド", Qiita — 10トピック、Spotify/Vercel事例、エージェントフレンドリー設計
10. @isanakamishiro2, "エージェント・ハーネスの解剖学", Qiita — 構成要素分類、長期実行パターン
11. @s-age, "プロンプトエンジニアリング vs コンテキストエンジニアリング vs ハーネスエンジニアリング", Qiita — 3概念比較
