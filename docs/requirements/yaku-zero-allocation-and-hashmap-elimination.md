# 要件定義書: 役判定におけるヒープアロケーション & HashMap の排除（ゼロアロケーション化）

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> GitHub Issue: #105「perf(yaku): 役判定におけるヒープアロケーション & HashMap の排除（ゼロアロケーション化）」に対応します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | 役判定ゼロアロケーション化 & HashMap 排除（Issue #105） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀エンジンの役判定（`judge_yaku_set`）は、何切る評価や対局シミュレーション、期待値計算（`estimate_hand_value`）の最深部で高頻度に呼び出されるホットパスです。
現状の実装では、面子判定関数において以下のヒープ確保・ハッシュ計算オーバーヘッドが発生しています：

1. **一盃口・二盃口における HashMap 確保**: `has_ipeiko` および `has_ryanpeiko` において、パターンごとに `HashMap<TileName, usize>` が生成・破棄され、アロケータ負荷とハッシュ計算が発生している。
2. **三色同順・三色同刻における二重コレクション確保**: `has_sanshoku_doujun` および `has_sanshoku_doukou` において、`HashMap<usize, HashSet<usize>>` が動的生成され、ヒープ割り当てが連鎖している。
3. **面子パターンの動的確保**: `HandPattern` の `melds: Vec<MeldKind>`, `open_melds: Vec<MeldKind>` および `generate_patterns` の `patterns` で `Vec` のヒープ確保が発生している。

本改修の目的は、固定長配列・ビットマスクおよびスタックバッファ（`ArrayVec` / `SmallVec`）を導入し、役判定ホットパス（`judge_yaku_set`）の内部メモリアロケーションを完全にゼロ（または上限超過時のみ安全退避）とし、判定スループットを向上させることです。

### 1.2 スコープ

**スコープ内:**
- `has_ipeiko` / `has_ryanpeiko` の `HashMap` をスタック上の固定長配列 `[u8; 35]` に置換
- `has_sanshoku_doujun` / `has_sanshoku_doukou` の `HashMap<usize, HashSet<usize>>` をランク別 `u8` ビットマスク（順子 `[u8; 8]`、刻子 `[u8; 10]`）に置換
- `HandPattern.melds` および `open_melds` の `ArrayVec<MeldKind, 4>` 化（面子数は最大4組）
- パターン配列の `SmallVec<[HandPattern; 8]>` 化による安全なスタックインライン化
- `judge_yaku_set` を直接計測するベンチマークの追加（互換ラッパー `judge_yaku` との分離）
- 役判定結果（三色同順・三色同刻・一盃口・二盃口など全役）の同値性検証テスト

**スコープ外:**
- `judge_yaku` の戻り値シグネチャ（`HashSet<YakuId>`）の変更（後方互換性維持）
- 役の成立条件・ルールの変更

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| ゼロアロケーション (Zero-allocation) | 処理実行中にヒープメモリアロケータ（`malloc`/`free`）の呼び出しを一切行わず、スタックまたは静的メモリのみで完結させる設計。 |
| スーツビットマスク | 萬子(bit 0)、筒子(bit 1)、索子(bit 2) の存在フラグを表す `u8` 値。`0b111` で3スーツすべて揃った三色判定。 |
| `ArrayVec<T, CAP>` | 固定容量 `CAP` をスタック上に保持する可変長配列コンテナ。 |
| `SmallVec<[T; N]>` | $N$ 要素まではスタック上にインライン保持し、超過時のみ安全にヒープへ退避する動的配列コンテナ。 |

---

## 3. 機能要件

### REQ-001: 順子・刻子出現集計のビットマスク・固定配列化
**説明:**  
The system shall count sequences using `[u8; 35]` stack array for Ipeiko/Ryanpeiko, and check suits using `u8` bitmask array for Sanshoku, without allocating `HashMap` or `HashSet`.

**受入条件:**
- [ ] `has_ipeiko` および `has_ryanpeiko` から `HashMap` が完全に排除されること。
- [ ] `has_sanshoku_doujun` および `has_sanshoku_doukou` から `HashMap` および `HashSet` が完全に排除されること。
- [ ] 判定結果が既存実装と完全一致すること。

### REQ-002: HandPattern のスタック完結化
**説明:**  
The system shall store melds and open melds in `HandPattern` using `ArrayVec<MeldKind, 4>`, eliminating nested heap allocations.

**受入条件:**
- [ ] `HandPattern.melds` および `open_melds` が `ArrayVec<MeldKind, 4>` となり、ヒープ割り当てが発生しないこと。
- [ ] 面子分解パターン生成がスタック上（`SmallVec<[HandPattern; 8]>`）で安全に動作し、8パターン以下でヒープ割り当てがゼロであること。
- [ ] 九蓮宝燈等の多数パターン分岐でもあふれエラーなく安全に動作すること。

---

## 4. 非機能要件

### NFR-001: ベンチマーク分離とスループット向上
- `benches/mahjong_benchmark.rs` において、`judge_yaku`（`HashSet` 返却）と `judge_yaku_set`（ビットマスク返却）の双方を測定可能とすること。
- `judge_yaku_set` のレイテンシが改善すること。
