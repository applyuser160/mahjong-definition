# 要件定義書: rayon による打牌候補評価・鳴き判断・着順EV・牌譜レビューの並列化

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> Issue #88 に基づき、CPUバウンドな計算処理を Rayon でデータ並列化します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | parallel-evaluation-rayon |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-15 |
| 最終更新日 | 2026-09-15 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀AIエンジンにおける打牌候補評価（`evaluate_hand_discards`）、鳴き判断（`advise_call`）、着順期待値算出（`evaluate_hand_discards_with_placement`）、および局後振り返り（`ReviewTracker`）は、独立した候補や局面の組み合わせ探索・シミュレーションを行うためCPU負荷が高い処理です。
本改修では、`rayon` ワークスティーリング型データ並列ライブラリを導入し、候補・分岐・巡目をマルチコアで並列処理することで、計算レイテンシを大幅に短縮し、マルチコアCPUのリソースを最大活用することを目的とします。

### 1.2 スコープ

**スコープ内:**
- `Cargo.toml` への `rayon = "1.10"` の追加
- `evaluate_hand_discards()` における最大14種の打牌候補ループの `rayon::prelude::*` による並列化
- `advise_call()` におけるスルー、ポン、チー各構成候補の評価の並列化
- `evaluate_hand_discards_with_placement()` における素点候補ごとの順位シミュレーションの並列化
- `ReviewTracker::generate_report()` における記録済み巡目データ・悪手診断の並列処理化
- Python API 呼び出し時（GIL解放 `py.allow_threads` との整合性確認）の安全性確保
- ベンチマーク・単体テストによる並列処理後の整合性（確定的なソート結果）の検証

**スコープ外:**
- シャンテン数計算内部（単一スート探索内部）のマルチスレッド化（粒度が細かすぎるためオーバーヘッドが勝る）
- GPUやSIMD等の別アクセラレータ導入

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| オーナー | applyuser160 | 高 |
| 利用者 | UI / Python API / AI開発者 | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| Rayon | Rustで広く使われているワークスティーリング型データ並列ライブラリ |
| 打牌候補評価 | 手牌14枚から切る牌ごとのシャンテン数・受け入れ・和了確率・打点・危険度を算出しEVを求める処理 |
| 鳴き判断 | 捨て牌に対してスルー・ポン・チー（最大3パターン）の各遷移後手牌を評価し最善手を導出する処理 |
| 着順期待値 (Placement EV) | 素点EVに加え、局後着順確率分布シミュレーションを行いルールPtの期待値を算出する処理 |
| 牌譜レビュー | 対局中の全巡目の打牌意思決定とAI推奨手を比較し、悪手・疑問手およびEV損失を診断する処理 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ

- 対局中の思考時間およびUI上の打牌候補・鳴き候補提示の待ち時間を削減したい。
- 局後の全巡目レビューやバッチ牌譜分析を高速に完了させたい。
- マルチコアCPU環境でCPUコアを有効活用したい。
- 並列化しても計算結果やソート順にブレがなく、既存テストと完全に一致させたい。

### 3.2 前提条件・制約

**前提条件:**
- 対象のデータ構造（`TileName`, `Hand`, `AnalysisContext`, `MatchContext` 等）はスレッド間で共有可能（`Send + Sync`）であること。
- Rust 2021 edition。

**制約:**
- 依存関係の追加は `rayon = "1.10"` のみとし、不要なランタイム負荷をかけないこと。
- EV同値時のタイブレーク条件（向聴数、受入枚数、牌インデックス等）を明確にし、並列イテレータの収集後も同一の安定ソート結果を保証すること。

---

## 4. 機能要件

### 4.1 打牌候補評価の並列化 (`expectation.rs`)

#### REQ-001: 打牌候補ループのデータ並列化
**説明:**  
The system shall execute the candidate evaluation loop in `evaluate_hand_discards()` concurrently using `rayon::prelude::IntoParallelIterator`.
手牌に存在する牌種（1..=34で `counts[i] > 0` の最大14種）について、各候補の `calculate_acceptance`, `estimate_win_probability`, `estimate_hand_value`, `evaluate_tile_safety` を独立したタスクとして並列処理すること。

**受入条件:**
- [ ] 候補評価結果のEV、向聴数、受け入れ枚数が従来の逐次実行と完全に一致すること。
- [ ] 期待値降順ソート結果が決定論的であること。

**優先度:** Must

---

### 4.2 鳴き判断の並列化 (`call_advisor.rs`)

#### REQ-002: 鳴き選択肢評価の並列化
**説明:**  
The system shall evaluate standing hand (Pass), Pon, and valid Chii configurations concurrently in `advise_call()`.
各アクション（スルー、ポン、チー最大3種）に対する後続手牌評価（`evaluate_hand_discards` 等）を並列に実行すること。

**受入条件:**
- [ ] スルー、ポン、チーの各候補の評価値が従来の逐次実行と完全に一致すること。
- [ ] 最善鳴きアクションの選定結果（`best_action`, `recommendation`, `rationale`）が同一であること。

**優先度:** Must

---

### 4.3 着順期待値算出の並列化 (`placement_ev.rs`)

#### REQ-003: 候補別着順シミュレーションの並列化
**説明:**  
The system shall compute `PlacementCandidateEvaluation` concurrently for all raw candidate evaluations in `evaluate_hand_discards_with_placement()`.
各候補に対する和了・放銃・流局シナリオの着順確率分布シミュレーション（`estimate_rank_probabilities`）を並列処理すること。

**受入条件:**
- [ ] 順位EV、期待順位、着順確率分布の計算結果が逐次実行と完全に一致すること。
- [ ] 順位EV降順ソートが正しく行われること。

**優先度:** Must

---

### 4.4 牌譜レビュー処理の並列化 (`review.rs`)

#### REQ-004: 局後レビューレポート集計の並列化
**説明:**  
The system shall analyze turn decision records and generate diagnostic explanations concurrently in `ReviewTracker::generate_report()`.
記録された各ターンのEV損失判定および `diagnose_loss_reason` のテキスト生成を並列イテレータで処理すること。

**受入条件:**
- [ ] レポートの精度（`accuracy_rate`）、総EV損失（`total_ev_loss`）、悪手一覧（`blunders`）の件数および内容が一致すること。
- [ ] 悪手リストがEV損失降順に正しく整列されること。

**優先度:** Must

---

## 5. 要件品質チェックリスト

### 5.1 個々の要件の品質（5特性）

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---------|--------|--------|--------|--------|--------|
| REQ-001 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-002 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-003 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-004 | ☑ | ☑ | ☑ | ☑ | ☑ |

---

## 6. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-15 | 初版作成（Issue #88 に基づく） | Antigravity AI |
