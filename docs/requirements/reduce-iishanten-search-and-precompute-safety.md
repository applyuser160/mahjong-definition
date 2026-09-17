# 要件定義書: 一向聴打点推定の探索削減 & 安全度事前計算

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> GitHub Issue: #104「perf(expectation): 一向聴打点推定における不要な受け入れ全探索ループの削減」に対応します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | 一向聴打点推定の探索削減 & 安全度事前計算（Issue #104） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀エンジンの打牌候補評価（`evaluate_hand_discards`）および一向聴（`shanten == 1`）打点推定（`estimate_hand_value`）は、何切る解析やCPU思考、牌譜レビューにおいて頻繁に呼び出される最重要ホットパスです。

現状の実装では以下の2つの重大な計算冗長性が存在しています：
1. **一向聴時の不要な受け入れ全探索**: テンパイ打牌が存在するかを調べる目的で、手牌の打牌候補（最大14種）ごとに `calculate_acceptance`（内部で全34種の仮ツモとシャンテン計算）を無条件に呼び出しており、最大50,000回以上のシャンテン数計算が発生している。
2. **打牌候補ごとの河スキャン・スジビットマスク再構築**: `evaluate_hand_discards` において他家河や可視牌状況は全候補で共通であるにもかかわらず、各候補の `evaluate_tile_safety` でリーチ者の河（`contains(tile)`）を毎回線形探索し、スジ判定用 `u16` ビットマスクも毎打牌ごとに再構築している。

本改修の目的は、一向聴テンパイ判定の Fast Reject 化および `SafetyFeatures` による安全度事前計算を導入し、アルゴリズムの評価結果を100%同一に保ったまま、打牌評価ホットパスを劇的に高速化することです。

### 1.2 スコープ

**スコープ内:**
- `estimate_hand_value()` における一向聴テンパイ探索の軽量化（`calculate_shanten_from_counts` による向聴数直接判定と Fast Reject）
- `SafetyFeatures` 構造体の設計・導入（全リーチ者の現物ビットマスク `u64`、スジ用ランクマスク `[u16; 3]`、河欠損フラグの事前集計）
- `evaluate_tile_safety_with_features()` の追加および `evaluate_tile_safety()` の互換性維持
- `evaluate_hand_discards()` における `SafetyFeatures` の一括事前計算と再利用
- リーチなし、単独リーチ、複数リーチ、河欠損を含む全34牌での安全度同値性検証テスト
- 一向聴および複数リーチ局面における `evaluate_hand_discards` ベンチマークの追加

**スコープ外:**
- 打点・和了確率・安全度の計算式および評価ロジックの変更（結果は既存と完全一致）
- `yaku.rs` や `suit_table.rs` 等、他のモジュールの最適化（Issue #105〜#109 で対応）

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| `SafetyFeatures` | 対局コンテキスト（リーチ状態・各プレイヤーの河）から事前計算された、全打牌候補で再利用可能な安全度特徴量。 |
| `Fast Reject` | テンパイ不成立の打牌候補を、34回の有効牌全探索を行わずに1回の向聴数計算で即座に除外する最適化手法。 |
| 現物 (Genbutsu) | リーチ者がすでに河に捨てている牌（フリテンルールによりロン和了されない安全牌）。 |
| スジ (Suji) | 両面待ちに対する安全牌理論（例: 4が捨てられている場合の1および7）。 |

---

## 3. 機能要件

### REQ-001: 一向聴テンパイ探索の Fast Reject 化
**説明:**  
The system shall verify whether a discarded tile achieves tenpai (`shanten == 0`) using `calculate_shanten_from_counts(&working, open_melds.len()).min_shanten == 0` before invoking acceptance calculation.

**受入条件:**
- [ ] テンパイ不成立の打牌候補に対して `calculate_acceptance` が呼び出されないこと。
- [ ] テンパイが確認された打牌候補に対してのみ和了牌（待ち牌）が取得されること。
- [ ] 打点推定結果（`expected_score`, `expected_han`, `primary_yaku`）が従来実装と完全一致すること。

### REQ-002: SafetyFeatures の事前集計と O(1) 判定
**説明:**  
The system shall precompute `SafetyFeatures` once per `evaluate_hand_discards` call and evaluate tile safety in O(1) time without rescanning player rivers.

**受入条件:**
- [ ] 全リーチ者に対する現物ビットマスク（積集合 `all` / 和集合 `any`）およびスジ用ランクマスク `[u16; 3]` が事前に1回だけ構築されること。
- [ ] リーチ者の河が欠落している（スライスの範囲外）場合に、現物全安全を false にする既存の挙動が忠実に保持されること。
- [ ] 既存の `evaluate_tile_safety` と `SafetyFeatures` 経由の判定結果が、リーチなし・単独/複数リーチ・河欠損を含むすべての牌種（1..=34）で完全に一致すること。

---

## 4. 非機能要件

### NFR-001: 性能向上
- `evaluate_hand_discards` の実行速度が、一向聴局面および複数リーチ局面において大幅に向上すること（ベンチマークで測定）。

### NFR-002: 後方互換性
- 既存の `evaluate_tile_safety(tile, ctx, visible_counts)` のシグネチャを維持し、外部呼出元を破壊しないこと。
