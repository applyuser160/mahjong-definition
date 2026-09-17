# 要件定義書: 役判定結果の u64 ビットマスク化 (YakuSet) による高速化

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> GitHub Issue: #89「perf: yaku.rs の HashSet<YakuId> を u64 ビットマスクに置き換える」に対応します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | YakuSet による役判定結果の u64 ビットマスク化 |
| バージョン | 1.1.0 |
| 作成日 | 2026-09-15 |
| 最終更新日 | 2026-09-15 |
| 作成者 | Antigravity AI |
| ステータス | Review (PR #101 指摘反映) |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀AIエンジン（`mahjong`）における役判定処理は、打牌候補評価（`expectation.rs`）や着順期待値シミュレーションにおいて極めて高頻度で呼び出されます。
従来の `HashSet<YakuId>` はハッシュ計算、ヒープ割り当て、キャッシュ非局所性によるオーバーヘッドを伴うため、手牌評価のホットパスで深刻な性能ボトルネックとなっていました。

全41種類の `YakuId` は 64-bit 整数（`u64`）のビットマスクで余剰なく表現可能です。
本機能の目的は、スタック完結型の `u64` ビットマスク構造体 `YakuSet` およびホットパス用判定API `judge_yaku_set()` を提供し、役判定およびその判定結果の参照におけるヒープ割り当てを完全排除することです。
同時に、既存の公開API `judge_yaku()` を互換ラッパーとして維持し、外部利用者の後方互換性を100%保証します。

### 1.2 スコープ

**スコープ内:**
- `YakuSet` 構造体のカプセル化（内部値非公開）と安全なビット操作の実装
- 有効な41ビットのマスク定数（`VALID_YAKU_MASK`）による不変条件保護
- `YakuSet` に対する `IntoIterator` / `ExactSizeIterator` の整合性保証
- ホットパス向け `judge_yaku_set()` の新設
- 既存公開API `judge_yaku()` の互換ラッパー（`HashSet<YakuId>` 返却）の維持
- 役満判定・フィルタリング（`retain_yakuman_only`）のビット演算化
- `expectation.rs` および Python API での `judge_yaku_set()` 利用によるホットパス最適化

**スコープ外:**
- 麻雀役の判定ルールや成立条件自体の変更
- 新規役（ローカル役等）の追加
- シャンテン数計算アルゴリズムの変更

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| `YakuId` | 全41種類の麻雀役を一意に識別する列挙型（`Riichi`, `Tanyao`, `Daisangen` 等）。 |
| `YakuSet` | `u64` の各ビットを `YakuId` の各バリアントに対応させたビットマスク構造体。内部値は非公開でスタック割り当て・`Copy` 可能。 |
| `VALID_YAKU_MASK` | 全41種類の役に該当する下位41ビット（`0..=40`）のみを 1 にした有効マスク定数（`(1u64 << 41) - 1`）。 |
| `judge_yaku_set` | ホットパス向けに `YakuSet` を返却するゼロアロケーション役判定API。 |
| `judge_yaku` | 既存の後方互換性を維持するために `HashSet<YakuId>` を返却する公開APIラッパー。 |

---

## 3. 機能要件

### 3.1 YakuSet のデータ構造とカプセル化

#### REQ-001: YakuSet 構造体のカプセル化と不変条件保護
**説明:**  
The system shall define `pub struct YakuSet(u64)` with a private internal field, ensuring all contained bits strictly satisfy `bits & !VALID_YAKU_MASK == 0`.

**受入条件:**
- [ ] 内部フィールドは非公開（private）であること。
- [ ] `from_raw(bits: u64) -> Self` は、有効な41ビット外のビットを自動的にマスク（`bits & VALID_YAKU_MASK`）すること。
- [ ] `from_raw_checked(bits: u64) -> Option<Self>` は、無効ビットが含まれる場合に `None` を返却すること。
- [ ] `as_raw(&self) -> u64` により内部の有効ビット表現を取得できること。

#### REQ-002: YakuSet の基本操作メソッド
**説明:**  
The system shall provide `insert`, `remove`, `contains`, `is_empty`, and `len` methods on `YakuSet`.

**受入条件:**
- [ ] `insert(id: YakuId)` により、対応するビットが 1 に設定されること。
- [ ] `remove(id: YakuId)` により、対応するビットが 0 に設定されること。
- [ ] `contains(&self, id: &YakuId) -> bool` により、対応ビットが立っているかを O(1) 分岐なしで判定できること。
- [ ] `is_empty(&self) -> bool` により、保持する役が0個であるかを `self.0 == 0` で判定できること。
- [ ] `len(&self) -> usize` により、立っているビット数を返却すること。
- [ ] `ExactSizeIterator` の返す要素数と `len()` の値、および `Debug` 出力の要素数が完全に一致すること。

### 3.2 役判定ロジックと後方互換性

#### REQ-003: judge_yaku_set の提供
**説明:**  
The system shall provide `judge_yaku_set(...) -> YakuSet` as a zero-allocation public API for performance-critical callers.

**受入条件:**
- [ ] `judge_yaku_set()` は内部で一切のヒープ割り当てを行わず `YakuSet` を返却すること。
- [ ] `expectation.rs` および `python_api.rs` では `judge_yaku_set()` を使用してホットパスを高速化すること。

#### REQ-004: judge_yaku の後方互換性維持
**説明:**  
The system shall retain `judge_yaku(...) -> HashSet<YakuId>` as a backward-compatible public API.

**受入条件:**
- [ ] `judge_yaku()` の戻り値型は `HashSet<YakuId>` を維持すること。
- [ ] 内部で `judge_yaku_set()` を呼び出し、その結果を `HashSet` に変換して返却すること。
- [ ] 既存の外部利用者のコード・型注釈が修正なしでコンパイル・動作可能であること。
