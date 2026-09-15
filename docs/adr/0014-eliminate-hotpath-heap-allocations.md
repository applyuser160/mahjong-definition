# ADR 0014: ホットパスのヒープ割り当て排除（ArrayVec / ビットマスクの採用）

## メタ情報

| 項目 | 値 |
|------|-----|
| ステータス | 承認済 (Accepted) |
| 日付 | 2026-09-15 |
| 意思決定者 | applyuser160, Antigravity AI |
| 関連Issue | #91 |
| 関連ADR | ADR 0011 (シャンテン数計算ルックアップテーブル), ADR 0013 (YakuSet ビットマスク化) |

---

## コンテキスト (Context)

麻雀AIエンジン（`mahjong`）では、打牌候補評価（`expectation.rs`）、有効牌計算（`acceptance.rs`）、手牌管理（`hand.rs`）、および副露アドバイザー（`call_advisor.rs`）がミリ秒単位で数万回呼び出されるコアホットパスとなっています。

これまで以下の箇所で動的配列 `Vec` が使われていました：
1. `evaluate_tile_safety()` におけるリーチ者の河の同色牌ランク収集（打牌候補ごとに毎回 `Vec::new()`）
2. `evaluate_tile_safety()` におけるリーチ対局者のフィルタリング（`Vec<usize>` への collect）
3. `AcceptanceResult.waits`（最大34種類の待ち牌情報）
4. `SpeedMetric.accepted_tiles`（最大34種類）および `ValueMetric.primary_yaku`（最大10種類程度）
5. `Hand.open_melds`（最大4個の副露情報）

これらは要素数の上限がドメインルール（牌種34種、対局者4人、副露4回）により確定しているにもかかわらず、ヒープ割り当てが発生し、アロケータのロック競合（並列時）やキャッシュミス、クローンオーバーヘッドの原因となっていました。

---

## 検討した選択肢 (Options Considered)

### 選択肢 1: 固定長配列（`[Option<T>; N]` や `([T; N], usize)`）を自作する
- **長所:** 外部依存クレートが不要。
- **短所:** イテレータ対応、スライスコピー、`push` / `pop` などのボイラープレートコードが各所で肥大化し保守性が下がる。

### 選択肢 2: `tinyvec` クレートを採用する
- **長所:** `no_std` や `100% safe code` 志向。
- **短所:** `ArrayVec` に格納する要素に `Default` トレイトの実装が要求される場合があり、API制約がやや強い。

### 選択肢 3: `arrayvec` クレート（`ArrayVec<T, CAP>`）および整数ビットマスク（`u16`）を採用する（推奨）
- **長所:**
  - Rustエコシステムでデファクトスタンダードの実績があり、`Vec` ライクなAPI（`Deref<Target=[T]>`、`FromIterator` 等）をスタック上に提供。
  - `Default` 制約なしに任意の型を格納可能。
  - スジ判定は `u16` ビットマスク（bit 1..=9）にすることで、割り当てだけでなく比較処理も O(1) に高速化。
- **短所:**
  - 外部クレート依存（`arrayvec = "0.7"`）が1つ増える（ただし依存が極めて軽量でピュアRust）。

---

## 決定事項 (Decision)

**選択肢 3 を採用する。**

1. `Cargo.toml` に `arrayvec = "0.7"` を追加する。
2. `AcceptanceResult.waits` を `ArrayVec<WaitTile, 34>` に変更。
3. `Hand.open_melds` を `ArrayVec<Meld, 4>` に変更し、`Hand` 構造体を完全スタック型にする。
4. `SpeedMetric.accepted_tiles` を `ArrayVec<TileName, 34>` に、`ValueMetric.primary_yaku` を `ArrayVec<&'static str, 10>` に変更。
5. `evaluate_tile_safety()` 内のリーチ者河収集を `u16` ビットマスクに、リーチ者フィルタをインラインループに変更。

---

## 帰結 (Consequences)

### ポジティブな影響
- **ゼロアロケーション**: 打牌候補評価ループおよび有効牌計算のホットパスからヒープ割り当てが完全に排除される。
- **手牌クローンの超高速化**: `Hand` が固定サイズになるため、`call_advisor.rs` での副露シミュレーション等でのクローンがポインタ追従不要のメモリコピー（memcpy）となる。
- **スジ判定の O(1) 化**: `contains(&rank)` の線形探索からビット演算（`&`）になり、CPUサイクルが削減される。
- **後方互換性**: Python バインディング層で透過的に変換されるため、Python 側クライアントへの破壊的変更はない。

### ネガティブな影響・トレードオフ
- `Hand` 構造体のスタックサイズがわずかに増加する（`ArrayVec<Meld, 4>` によるスタック確保）。ただし `Meld` は数バイト〜十数バイトのため、キャッシュライン内に十分収まり、むしろキャッシュ局所性が劇的に向上する。
