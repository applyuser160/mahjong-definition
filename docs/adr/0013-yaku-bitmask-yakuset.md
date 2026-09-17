# ADR-0013: YakuSet (u64 ビットマスク) による役判定結果の高速化と後方互換性維持

> **ADR番号:** 0013  
> **作成日:** 2026-09-15  
> **更新日:** 2026-09-15  
> **作成者:** Antigravity AI  

---

## ステータス

`Accepted` (PR #101 レビュー指摘反映)

---

## コンテキスト（Context）

麻雀AIエンジン（`mahjong`）における `judge_yaku()` は、手牌の成立役を判定して `HashSet<YakuId>` で返却していました。
`YakuId` は41個のバリアントを持つ列挙型です。打牌候補評価（`expectation.rs`）のループ（`estimate_hand_value`）内で高頻度に呼び出されるため、`HashSet` のヒープ割り当てとハッシュ走査がボトルネックとなっていました。

一方、既存の公開APIとしての `judge_yaku` の戻り値型を変更すると、外部クレートや利用者の型注釈・関数境界でコンパイルエラー（破壊的変更）が発生します。
また、`u64` ビットマスク構造体において未定義の上位23ビット（41〜63ビット）が混入すると、`len()` とイテレータの要素数が乖離し `ExactSizeIterator` の契約違反が生じる課題がありました。

---

## 意思決定の要因（Decision Drivers）

- **ゼロ・アロケーション:** ホットパスにおいてヒープ割り当てを完全に排除すること。
- **後方互換性の完全維持:** 既存の `judge_yaku(...) -> HashSet<YakuId>` を破壊せず、利用者の移行コストをゼロにすること。
- **型安全性・カプセル化:** `YakuSet` の内部ビットをカプセル化し、不正な範囲外ビットの混入を防止してイテレータ等の不変条件を厳格に保証すること。

---

## 決定結果（Decision Outcome）

1. **`judge_yaku_set` の新設と `judge_yaku` の互換ラッパー化:**
   - ホットパス向けにゼロアロケーションの `judge_yaku_set(...) -> YakuSet` を新設。
   - 既存の公開関数 `judge_yaku(...) -> HashSet<YakuId>` は内部で `judge_yaku_set` を呼び出して `HashSet` へ変換する互換ラッパーとして残す。
   - `expectation.rs` および `python_api.rs` では `judge_yaku_set` を直接使用する。
2. **`YakuSet` のカプセル化と有効ビットマスク (`VALID_YAKU_MASK`) の適用:**
   - 内部フィールドを private（`pub struct YakuSet(u64)`）にする。
   - 有効な41ビットのマスク定数 `VALID_YAKU_MASK = (1u64 << 41) - 1` を定義。
   - `from_raw` は有効ビットでマスク（truncate）し、`from_raw_checked` は無効ビット混入時に `None` を返却する。
