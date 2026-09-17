# 基本・詳細設計書: 有効牌探索（calculate_acceptance）における差分シャンテン計算 & 枝刈り導入

> 本設計書は、Issue #108 の解決に向けた `ShantenState` の差分更新アルゴリズムおよび有効牌枝刈りマスクの設計書です。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | 有効牌探索の差分シャンテン計算 & 枝刈り（Issue #108） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 差分シャンテン計算アーキテクチャ (`ShantenState`)

### 1.1 状態構造体の定義

```rust
pub struct ShantenState<'a> {
    counts: &'a [u8; 35],
    open_melds_count: usize,
    is_closed: bool,

    // 通常形用事前集計
    suit_keys: [usize; 3],          // [萬子キー, 筒子キー, 索子キー]
    suit_entries: [SuitEntry; 3],   // [萬子LUT, 筒子LUT, 索子LUT]
    z_melds: i8,                    // 字牌刻子数
    z_pairs: i8,                    // 字牌対子数

    // 七対子用事前集計 (門前時のみ)
    chitoitsu_pairs: i8,
    chitoitsu_kinds: i8,

    // 国士無双用事前集計 (門前時のみ)
    kokushi_kinds: i8,
    kokushi_has_pair: bool,

    // 現在の向聴数
    pub current_result: ShantenResult,
}
```

### 1.2 仮ツモ `after_draw(tile_idx)` の差分更新ロジック

```rust
impl<'a> ShantenState<'a> {
    pub fn new(counts: &'a [u8; 35], open_melds_count: usize) -> Self {
        ...
    }

    pub fn after_draw(&self, tile_idx: usize) -> ShantenResult {
        let c = self.counts[tile_idx];
        debug_assert!(c < 4);

        // 1. 通常形の差分更新
        let normal = if tile_idx <= 27 {
            // 数牌
            let suit = (tile_idx - 1) / 9;
            let pos = (tile_idx - 1) % 9;
            let pow5 = POW5[pos];
            let new_key = self.suit_keys[suit] + pow5;
            let new_entry = get_suit_entry(new_key);

            let mut suits = self.suit_entries;
            suits[suit] = new_entry;
            eval_normal_shanten(&suits, self.z_melds, self.z_pairs, self.open_melds_count)
        } else {
            // 字牌: 3スーツのLUTは再利用
            let (new_z_melds, new_z_pairs) = match c {
                1 => (self.z_melds, self.z_pairs + 1),         // 1->2 (対子化)
                2 => (self.z_melds + 1, self.z_pairs - 1),     // 2->3 (刻子化)
                _ => (self.z_melds, self.z_pairs),             // 0->1 または 3->4
            };
            eval_normal_shanten(&self.suit_entries, new_z_melds, new_z_pairs, self.open_melds_count)
        };

        // 2. 七対子の差分更新
        let chitoitsu = if self.is_closed {
            let (new_pairs, new_kinds) = match c {
                0 => (self.chitoitsu_pairs, self.chitoitsu_kinds + 1),
                1 => (self.chitoitsu_pairs + 1, self.chitoitsu_kinds),
                _ => (self.chitoitsu_pairs, self.chitoitsu_kinds),
            };
            let mut s = 6 - new_pairs;
            if new_kinds < 7 {
                s += 7 - new_kinds;
            }
            s
        } else {
            99
        };

        // 3. 国士無双の差分更新
        let kokushi = if self.is_closed && is_terminal_or_honor(tile_idx) {
            let (new_kinds, new_pair) = match c {
                0 => (self.kokushi_kinds + 1, self.kokushi_has_pair),
                1 => (self.kokushi_kinds, true),
                _ => (self.kokushi_kinds, self.kokushi_has_pair),
            };
            13 - new_kinds - if new_pair { 1 } else { 0 }
        } else if self.is_closed {
            self.current_result.kokushi
        } else {
            99
        };

        let min_shanten = normal.min(chitoitsu).min(kokushi);
        ShantenResult {
            min_shanten,
            normal,
            chitoitsu,
            kokushi,
        }
    }
}
```

---

## 2. 枝刈りマスク（Pruning Mask）設計

手牌に牌が存在する位置から、シャンテン数を進めうる有効牌のマスク（`u64` ビットフラグ）を $O(1)$ で構築：

1. **通常形の候補**:
   - 数牌（萬子: 1..=9, 筒子: 10..=18, 索子: 19..=27）:
     - `counts[i] > 0` ならば、同一スート内の `max(start, i - 2)..=min(end, i + 2)` のビットを立てる。
   - 字牌（28..=34）:
     - `counts[i] > 0` ならば、ビット `i` を立てる。
2. **特殊形の候補（条件付き追加）**:
   - `is_closed` かつ `current_result.chitoitsu <= current_shanten`:
     - 七対子が進む可能性があるため、全対子候補（`counts[i] == 1`）はもちろん、種類数が 7 未満なら全牌（1..=34）を探索対象に含める。
   - `is_closed` かつ `current_result.kokushi <= current_shanten`:
     - 国士無双が進む可能性があるため、13 種の么九牌すべてをビットマスクに含める。

マスク外の牌は、仮ツモ・向聴数計算を一切スキップします。

---

## 3. テスト設計

1. **同値性プロパティテスト (`test_shanten_state_draw_equivalence`)**:
   - 多様な手牌（一向聴、二向聴、テンパイ、七対子、国士無双）に対し、全 34 牌の `after_draw(i)` の結果が、従来の `calculate_shanten_from_counts` の結果と厳密に一致することをテスト。
2. **受け入れ牌計算テスト (`test_acceptance_with_pruning`)**:
   - 既存の `test_shanten` および受け入れ牌テストが 100% 同値であることを確認。
