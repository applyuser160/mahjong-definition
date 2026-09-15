# 基本設計書: 大規模 match の算術演算・配列ルックアップへの置換最適化

> 本設計書は `docs/requirements/match-optimization-arithmetic-lookup.md` に基づき、大規模 match 式の算術演算・配列ルックアップへの置換アーキテクチャおよび詳細アルゴリズムを定義します。

---

## 1. システム構成・アーキテクチャ概要

`TileName` 列挙型はメモリレイアウト `#[repr(u8)]` で定義されており、各バリアントが 0 から 34 の連続した整数値にマップされています。

```
0: None
1..=9:   OneM .. NineM    (萬子: スーツ0, ランク1..=9)
10..=18: OneP .. NineP    (筒子: スーツ1, ランク1..=9)
19..=27: OneS .. NineS    (索子: スーツ2, ランク1..=9)
28..=31: East .. North    (風牌: 字牌)
32..=34: Red .. White     (三元牌: 字牌)
```

この連続したメモリ表現を活用し、従来の 27〜35 分岐 `match` によるジャンプテーブル／分岐連鎖を排除し、O(1) の定数時間処理に移行します。

```mermaid
flowchart TD
    subgraph TileName Transformations
        A["Tile Index (0..=34)"] -->|Bounds check: n <= 34| B["std::mem::transmute(n as u8)"]
        A -->|n > 34| C["TileName::None"]
        D["TileName instance"] -->|Discriminant as u8| E["Numeric Range Match (tile_type)"]
        D -->|Index into [&str; 35]| F["O(1) String Literal (as_str)"]
        D -->|Index into [TileName; 35]| G["O(1) Dora Indicator (indicator_to_dora)"]
    end

    subgraph Yaku & Property Helpers
        H["TileName"] -->|idx in 1..=27| I["Suit = (idx - 1) / 9\nRank = (idx - 1) % 9 + 1"]
        H -->|idx in 28..=34| J["None (is_number_tile)"]
        H -->|idx in 1,9,10,18,19,27| K["is_terminal = true"]
        H -->|idx in 28..=34| L["is_honor = true"]
        H -->|idx in 2..=8,11..=17,20..=26| M["is_simple = true"]
    end
```

---

## 2. 詳細設計

### 2.1 `TileName::from_usize` (`src/mahjong/tile.rs`)

従来の 35 分岐 `match` を境界チェックと `std::mem::transmute` に置換します。

```rust
#[inline(always)]
#[allow(dead_code)]
pub const fn from_usize(n: usize) -> TileName {
    if n <= 34 {
        // SAFETY: TileName は #[repr(u8)] であり、0..=34 のすべての値に対して
        // 有効な enum バリアント (None = 0, OneM..=White = 1..=34) が定義されているため健全。
        unsafe { std::mem::transmute(n as u8) }
    } else {
        TileName::None
    }
}
```

- **定数化 (`const fn`)**: コンパイル時評価が可能。
- **インライン展開**: `#[inline(always)]` により呼び出し元に関数ボディがインライン展開され、余計なコールスタックやジャンプを排除。

### 2.2 `TileName::tile_type` & `TileName::category` (`src/mahjong/tile.rs`)

34 分岐のパターンマッチを、単一の `u8` キャストと数値範囲マッチングに集約します。

```rust
#[inline]
pub const fn tile_type(&self) -> TileType {
    let idx = *self as u8;
    match idx {
        1..=9 => TileType::Characters,
        10..=18 => TileType::Circles,
        19..=27 => TileType::Bamboos,
        28..=31 => TileType::Winds,
        32..=34 => TileType::Dragons,
        _ => TileType::None,
    }
}

#[inline]
pub const fn category(&self) -> TileCategory {
    let idx = *self as u8;
    match idx {
        1..=27 => TileCategory::Simples,
        28..=34 => TileCategory::Honors,
        _ => TileCategory::None,
    }
}
```

### 2.3 `TileName::as_str` (`src/mahjong/tile.rs`)

35 分岐の match を静的配列ルックアップに置換します。

```rust
const TILE_STRS: [&str; 35] = [
    " ",
    "1m", "2m", "3m", "4m", "5m", "6m", "7m", "8m", "9m",
    "1p", "2p", "3p", "4p", "5p", "6p", "7p", "8p", "9p",
    "1s", "2s", "3s", "4s", "5s", "6s", "7s", "8s", "9s",
    "東", "南", "西", "北",
    "中", "発", "白",
];

#[inline]
#[allow(dead_code)]
pub fn as_str(&self) -> &'static str {
    let idx = *self as usize;
    if idx < TILE_STRS.len() {
        TILE_STRS[idx]
    } else {
        " "
    }
}
```

### 2.4 `indicator_to_dora` (`src/mahjong/dora.rs`)

ドラ表示牌の変換（万・筒・索の 1→2..9→1、風の 東→南→西→北→東、三元の 白→發→中→白）を静的配列ルックアップに置換します。

```rust
const DORA_INDICATOR_TABLE: [TileName; 35] = [
    TileName::None,
    // 萬子 (1..9) -> 2..9, 1
    TileName::TwoM, TileName::ThreeM, TileName::FourM, TileName::FiveM,
    TileName::SixM, TileName::SevenM, TileName::EightM, TileName::NineM, TileName::OneM,
    // 筒子 (10..18) -> 11..18, 10
    TileName::TwoP, TileName::ThreeP, TileName::FourP, TileName::FiveP,
    TileName::SixP, TileName::SevenP, TileName::EightP, TileName::NineP, TileName::OneP,
    // 索子 (19..27) -> 20..27, 19
    TileName::TwoS, TileName::ThreeS, TileName::FourS, TileName::FiveS,
    TileName::SixS, TileName::SevenS, TileName::EightS, TileName::NineS, TileName::OneS,
    // 風牌 (28..31) -> 29, 30, 31, 28 (東->南->西->北->東)
    TileName::South, TileName::West, TileName::North, TileName::East,
    // 三元牌 (32..34) -> Red(32), Green(33), White(34)
    // 白(34) -> 發(33) -> 中(32) -> 白(34)
    TileName::White, // 32 (Red: 中) -> White (白)
    TileName::Red,   // 33 (Green: 發) -> Red (中)
    TileName::Green, // 34 (White: 白) -> Green (發)
];

#[inline]
pub fn indicator_to_dora(indicator: TileName) -> TileName {
    let idx = indicator as usize;
    if idx < DORA_INDICATOR_TABLE.len() {
        DORA_INDICATOR_TABLE[idx]
    } else {
        TileName::None
    }
}
```

### 2.5 役判定ヘルパー群 (`src/mahjong/yaku.rs`)

#### 2.5.1 `is_number_tile`
数牌（1..=27）に対して、スーツ（萬子:0, 筒子:1, 索子:2）とランク（1..=9）を算術演算で算出します。

```rust
#[inline]
pub fn is_number_tile(tile: TileName) -> Option<(usize, usize)> {
    let idx = tile as usize;
    if (1..=27).contains(&idx) {
        let zero_based = idx - 1;
        Some((zero_based / 9, zero_based % 9 + 1))
    } else {
        None
    }
}
```

#### 2.5.2 `is_terminal`, `is_honor`, `is_simple`, `is_terminal_or_honor`
ビットマスクや簡潔な数値範囲判定により分岐予測と I-Cache を効率化します。

```rust
#[inline]
fn is_terminal(tile: TileName) -> bool {
    let idx = tile as u8;
    matches!(idx, 1 | 9 | 10 | 18 | 19 | 27)
}

#[inline]
fn is_honor(tile: TileName) -> bool {
    let idx = tile as u8;
    (28..=34).contains(&idx)
}

#[inline]
fn is_terminal_or_honor(tile: TileName) -> bool {
    let idx = tile as u8;
    matches!(idx, 1 | 9 | 10 | 18 | 19 | 27 | 28..=34)
}

#[inline]
fn is_simple(tile: TileName) -> bool {
    let idx = tile as u8;
    matches!(idx, 2..=8 | 11..=17 | 20..=26)
}
```

#### 2.5.3 `search_melds` の順子探索最適化
`is_number_tile(tile)` で `rank <= 7` が保証されている場合、`next1 = i + 1`, `next2 = i + 2` は自明に同一スーツかつ `rank + 1`, `rank + 2` となるため、余計な `TileName::from_usize` および `is_number_tile` の呼び出しをスキップして牌カウントのみを判定します。

---

## 3. テスト・検証方針

1. **既存テストの全件通過**:
   - `cargo test` による 63 件のユニットテスト、および全結合テストのパス
2. **境界値テスト・網羅性検証**:
   - `TileName::from_usize(0)` 〜 `TileName::from_usize(35)`（境界外含む）
   - `indicator_to_dora` の 0..=34 全牌についての正当性確認
   - `is_number_tile` の全 35 牌に対する戻り値検証
   - `is_terminal`, `is_honor`, `is_simple` の全 35 牌に対する真偽値整合性検証
