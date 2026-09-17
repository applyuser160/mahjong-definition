# 要件定義書：向聴数計算へのスーツ別ルックアップテーブル導入

> ISO/IEC/IEEE 29148 準拠（SRS簡易版）  
> 本要件定義書は、参照専用の `definition/templates/requirements.md` のテンプレートに基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-core-engine |
| モジュール名 | shanten & acceptance (向聴数・受入牌計算コアエンジン) |
| Issue | [#86 perf: 向聴数計算にスーツ別ルックアップテーブルを導入する](https://github.com/applyuser160/mahjong/issues/86) |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-15 |
| 最終更新日 | 2026-09-15 |
| 作成者 | Syun & AI Assistant |
| ステータス | Approved |

---

## 1. 目的・スコープ

### 1.1 目的
麻雀エンジンにおいて向聴数（シャンテン数）計算は、有効牌受入計算（`calculate_acceptance`）、14枚手牌の全打牌候補評価（`analyze_all_discards`）、および期待値（EV）評価（`evaluate_hand_discards`）の最深ループ内で頻繁に呼び出されるクリティカルパスです。

従来の `search_normal()` は再帰的な深さ優先探索（DFS）で刻子・順子・対子・搭子の全組み合わせをバックトラッキング探索しており、特に清一色や複雑な多面張手牌では分岐数が爆発し、1回のシャンテン数計算に 50µs 以上の時間を要していました。

本改善では、数牌の各スーツ（萬子・筒子・索子の各9種、枚数0〜4）の配置パターン数が高々 $5^9 = 1,953,125$ 通り（手牌枚数合計 $\le 14$ の実質パターン数は 405,350 通り）である特性を活かし、**事前計算済みスーツ別ルックアップテーブル（LUT）** を導入します。これにより、数牌の再帰探索を $O(1)$ の配列参照3回と字牌の簡易走査、および雀頭パターンのループに置き換え、シャンテン数計算を 10〜100 倍高速化することを目的とします。

### 1.2 スコープ

**スコープ内:**
- **スーツ別ルックアップテーブル（LUT）のデータ構造・事前生成設計**:
  - 各スーツ（9牌、各0〜4枚）の牌姿インデックス化（5進数エンコード: $key = \sum_{i=0}^8 c_i \cdot 5^i$）。
  - 各牌姿における「雀頭なし」「雀頭あり」それぞれでの面子数 $m \in \{0..4\}$ に対する最大搭子数 $t(m)$ の事前計算と管理。
- **面子手シャンテン数計算ロジックのLUT統合**:
  - 萬子・筒子・索子の3スーツに対する LUT 参照。
  - 字牌（28..=34）の $O(7)$ 簡易走査（刻子数・対子数の抽出）。
  - 雀頭候補（萬子・筒子・索子・字牌・雀頭なしの最大5候補）と目標面子数（$4 - \text{open\_melds}$）に応じた最適向聴数の算出。
- **既存の七対子・国士無双および既存テストとの完全互換性保証**:
  - 既存のすべての単体テスト・結合テスト（境界値・副露手・多面張・九蓮宝燈等）が一切の振る舞いを変えずにパスすること。
  - 特殊手（七対子・国士無双）との最小値統合が既存仕様どおり機能すること。
- **性能ベンチマークによる効果測定**:
  - `benches/mahjong_benchmark.rs` を用いて、通常手・清一色・受入計算・全打牌評価における実行時間の短縮を実測・検証。

**スコープ外:**
- 七対子・国士無双のアルゴリズム変更（これらは既に $O(1) \sim O(13)$ で極めて軽量なため現状維持）。
- 点数計算や役判定ロジック自体の変更。

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| プロジェクトオーナー | ユーザー | 高 |
| 設計・実装 | AI Assistant | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| スーツ (Suit) | 麻雀の牌種カテゴリ。萬子（m）、筒子（p）、索子（s）の数牌3種と、字牌（z）1種。 |
| 面子 (Meld) | 3枚組の刻子（AAA）または順子（ABC）。 |
| 搭子 (Taatsu / Partial) | あと1枚で面子になる2枚組。対子（AA）、両面/辺張（AB）、嵌張（AC）。 |
| 雀頭 (Head) | 和了形を構成する1組の対子（2枚組）。 |
| ルックアップテーブル (LUT) | 事前計算された結果をメモリ上に保持し、キーから $O(1)$ で値を取得する配列・テーブル構造。 |
| 5進数エンコード (Base-5 Indexing) | 9種類の各牌の枚数（0〜4）を5進数の一桁として表現するインデックス計算方式。 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ
- CPU戦やドリル機能、打牌期待値（EV）計算において、ミリ秒単位で多数の手牌候補を評価する処理をストレスなく一瞬で完了させたい。
- 探索アルゴリズムの高速化に伴い、判定結果の正当性（既存の牌理・向聴数ロジック）が一切損なわれないことを保証したい。
- メモリ消費量が過大にならず、起動速度やバイナリフットプリントとのバランスが取れていること。

### 3.2 前提条件・制約
- **言語・環境**: Rust 2021 Edition。Windows / Linux / macOS クロスプラットフォーム。
- **正当性保証**: 既存の `test_shanten.rs` や `acceptance.rs` 等の全テストに完全合致すること。
- **メモリ制約**: テーブルのメモリフットプリントは 20MB 以内に抑えること。

---

## 4. 機能要件

### 4.1 テーブル構造およびインデックス化

#### REQ-001: 5進数インデックス化と異常入力防衛
**説明:**  
The system shall calculate the suit index from a 9-element tile count slice using base-5 encoding $\sum_{i=0}^8 \min(c_i, 4) \cdot 5^i$, producing an integer key in the range $[0, 5^9 - 1]$ ($0 \le \text{key} < 1,953,125$). Furthermore, input validation shall prevent out-of-bounds access or panics when invalid tile counts (e.g., 5 or more identical tiles) are supplied via Rust or Python APIs.

**受入条件:**
- [ ] 各要素が $0 \le c_i \le 4$ の牌配列について一意の整数キーが算出されること。
- [ ] $c_i \ge 5$ の異常入力が与えられた場合でも、安全に飽和処理（$\min(c_i, 4)$）され、パニック（配列外参照）が発生しないこと。
- [ ] Python API (`py_calculate_shanten`) において同一牌5枚以上の異常入力を安全に防衛・検証すること。

#### REQ-002: スーツ別牌姿情報の保持
**説明:**  
The system shall provide a lookup table storing the maximum partial melds (taatsu) achievable for each meld count $m \in \{0, 1, 2, 3, 4\}$, distinguishing between the presence and absence of a pair designated as the head (雀頭).

**受入条件:**
- [ ] 雀頭なし (`no_head`) と雀頭あり (`with_head`) の各ケースについて、面子数 $m \in 0..=4$ ごとに最大搭子数 $t$（または作成不可を示す無効値）を取得できること。
- [ ] 手牌合計枚数 $\le 14$ の全パターンに対して、現行のバックトラッキング探索と完全に同一の最大搭子数が記録されていること。

### 4.2 面子手向聴数計算（`calculate_normal_shanten`）の再実装

#### REQ-003: 字牌の高速走査
**説明:**  
The system shall evaluate honor tiles (字牌: 28..=34) in $O(7)$ operations without backtracking, counting triplets ($c \ge 3$) as melds and remaining pairs ($c = 2$) as partial melds / head candidates.

**受入条件:**
- [ ] 字牌の枚数から刻子数と対子数が一意に定まり、バックトラッキングなしで正確に集計されること。

#### REQ-004: スーツ合成と雀頭ループによる向聴数判定
**説明:**  
The system shall combine the lookup table entries for the three suits (萬子, 筒子, 索子) and the honor tile statistics across all possible head placements (each suit, honors, or headless), determining the minimum normal shanten for any given `open_melds_count` ($0 \le \text{open\_melds\_count} \le 4$).

**受入条件:**
- [ ] 雀頭候補（萬子・筒子・索子・字牌・雀頭なし）のすべてを網羅し、目標面子数 $4 - \text{open\_melds\_count}$ に適合する最小シャンテン数を算出すること。
- [ ] 和了形（-1）、テンパイ（0）、一向聴（1）から最大（8）まで、現行の計算結果と 100% 一致すること。

### 4.3 性能要件・非機能要件

#### REQ-005: 性能向上（10倍以上の高速化）
**説明:**  
The system shall execute `calculate_shanten` at least 10 times faster on complex hand patterns (e.g., Chinitsu / Chuuren Poutou) compared to the baseline DFS implementation.

**受入条件:**
- [ ] `benches/mahjong_benchmark.rs` の `Complex Chinitsu` において、実行時間が 50µs 超から 5µs 未満（10倍以上）に短縮されること。
- [ ] `calculate_acceptance` および `analyze_all_discards` が大幅に高速化されること。

#### REQ-006: ビルド時事前生成と冷間時ゼロレイテンシ
**説明:**  
The system shall generate the suit lookup table at build time via `build.rs` or statically initialize it so that cold-start shanten calculation incurs zero perceptible latency (no 2-second freeze on first request).

**受入条件:**
- [ ] ビルド時にテーブルバイナリを事前生成し、実行時の初回向聴数計算における遅延（2秒超の同期生成）を完全に排除（0.00ms）すること。
- [ ] 冷間時（初回呼び出し時）の向聴数計算テストが即座にパスすること。
- [ ] テーブル全体のメモリ使用量が 20 MiB 以下であること。
