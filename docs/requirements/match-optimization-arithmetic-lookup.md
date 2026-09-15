# 要件定義書: 大規模 match の算術演算・配列ルックアップへの置換最適化

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> GitHub Issue: #92「perf: tile.rs / yaku.rs の大規模 match を算術演算・配列ルックアップに置き換える」に対応します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | 大規模 match の算術演算・配列ルックアップ化最適化 |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-15 |
| 最終更新日 | 2026-09-15 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀AIエンジン（`mahjong`）における打牌評価シミュレーション、シャンテン数・受入計算、役判定（`yaku.rs`）は毎秒数十万〜数百万回実行される極めてクリティカルなパスです。
その基盤となる `TileName` 型（`#[repr(u8)]`）や牌種判別処理において、以下のような30〜35分岐の大規模な `match` 式が多数存在しています：

1. `TileName::from_usize(n)`: 0〜34のインデックスから `TileName` への35分岐 `match`
2. `TileName::tile_type(&self)`: 牌種（萬子・筒子・索子・風牌・三元牌）判定の34分岐 `match`
3. `TileName::as_str(&self)`: 文字列リテラル取得の35分岐 `match`
4. `is_number_tile(tile)`: 数牌判定および（スーツ番号・ランク番号）取得の27分岐 `match`
5. `is_terminal`, `is_honor`, `is_simple`: 么九牌・字牌・中張牌判定の大規模 `matches!`
6. `search_melds` 内での同一スーツ連続牌の冗長な牌情報問い合わせ

これらはバイナリ内でジャンプテーブルや分岐チェーンとして展開され、命令キャッシュ（I-Cache）の浪費やCPU分岐予測ミスの原因となります。
`TileName` が `#[repr(u8)]` で 0〜34 の連続値として定義されている特性を活かし、安全な境界チェック付きキャスト（transmute）や算術演算、固定ルックアップテーブルに置き換えることで、分岐数を激減させCPUパイプラインとキャッシュ効率を最大化します。

### 1.2 スコープ

**スコープ内:**
- `TileName::from_usize`: 境界チェック (`n <= 34`) のもと `unsafe { std::mem::transmute(n as u8) }` による O(1) 変換
- `TileName::tile_type`: `*self as u8` による数値範囲マッチング（5区間）への短縮
- `TileName::category`: `*self as u8` による数値範囲比較への短縮
- `TileName::as_str`: 静的配列 `[&'static str; 35]` による O(1) 配列ルックアップ化
- `indicator_to_dora`: 静的配列 `[TileName; 35]` による O(1) 配列ルックアップ化
- `is_number_tile`: `idx in 1..=27` に対する算術演算 `((idx - 1) / 9, (idx - 1) % 9 + 1)` への置換
- `is_terminal`, `is_honor`, `is_simple`, `is_terminal_or_honor`: 数値範囲およびビットマスクによる即時判定化
- `search_melds`: `rank <= 7` 判定後の自明な連続牌に対する冗長な関数呼び出しの排除

**スコープ外:**
- 麻雀ルールの判定結果・仕様の変更（既存テストの完全互換を維持）
- 公開 API のシグネチャ変更（互換性を維持）

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| `TileName` | 萬子・筒子・索子・字牌を表す `#[repr(u8)]` の列挙型（値域: 0..=34）。 |
| `#[repr(u8)]` | 各列挙バリアントがメモリ上で 1 バイトの符号なし整数として連続配置されることを保証する Rust アトリビュート。 |
| `I-Cache` | CPU 命令キャッシュ。ジャンプテーブルやコードサイズ肥大化を抑えることでヒット率が向上する。 |
| 分岐予測 | CPU パイプラインが条件分岐の成否を事前予測して投機実行する機構。分岐が排除されるとミスによるストールがゼロになる。 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ

- 大量シミュレーション時における命令実行数の削減およびスループットの向上
- 既存のテストスイートおよび機能・動作の完全な互換性維持

### 3.2 前提条件・制約

**前提条件:**
- `TileName` のバリアント定義順（0: None, 1..=9: 萬子, 10..=18: 筒子, 19..=27: 索子, 28..=31: 風牌, 32..=34: 三元牌）は固定であること。

**制約:**
- Rust の未定義動作（UB）を絶対に発生させないこと（境界外値に対する安全ガードの徹底）。
- 既存のすべての単体テスト・結合テストを一切壊さずパスすること。

---

## 4. 機能要件

### 4.1 `TileName` の O(1) 変換および判定最適化

#### REQ-001: `TileName::from_usize` の定数時間変換
The system shall convert `usize` index (0..=34) to `TileName` using bounds checking and safe `transmute` in O(1) constant time without 35-branch match statements, returning `TileName::None` for out-of-bounds inputs.

**受入条件:**
- [ ] 0..=34 のすべての入力に対して従来の `from_usize` と完全に同一の `TileName` を返すこと。
- [ ] 35 以上の入力に対して安全に `TileName::None` を返すこと。
- [ ] パニックや未定義動作が発生しないこと。

#### REQ-002: `TileName::tile_type` の数値範囲判定
The system shall determine `TileType` using integer range matching on `u8` discriminant instead of 34 individual variant matching arms.

**受入条件:**
- [ ] すべての `TileName` バリアントに対して従来の判定結果と一致すること。

#### REQ-003: `TileName::as_str` の配列ルックアップ
The system shall return string representation of `TileName` via static array indexing in O(1) time without branch instructions.

**受入条件:**
- [ ] すべての `TileName` バリアントに対して従来の文字列表現と完全に一致すること。

---

### 4.2 役判定および牌性質判定の算術演算化

#### REQ-004: `is_number_tile` の算術演算化
The system shall compute `(suit, rank)` for any `TileName` using `((idx - 1) / 9, (idx - 1) % 9 + 1)` for `idx in 1..=27` in O(1) time without match branching.

**受入条件:**
- [ ] 1m〜9s の全27種に対して正しいスーツ（0:萬子, 1:筒子, 2:索子）およびランク（1..=9）を返すこと。
- [ ] 字牌および `TileName::None` に対しては `None` を返すこと。

#### REQ-005: `is_terminal`, `is_honor`, `is_simple` の高速判定化
The system shall determine terminal, honor, and simple tile properties using bitwise masks or concise integer range expressions.

**受入条件:**
- [ ] 全35種の牌に対して従来の真偽判定結果と完全一致すること。

#### REQ-006: `search_melds` における順子探索の冗長チェック排除
The system shall streamline sequential meld search by eliminating redundant `is_number_tile` calls for `next1` and `next2` when the base tile is already validated as a number tile with `rank <= 7`.

**受入条件:**
- [ ] 面子分解および役判定テストが全件正常にパスすること。

---

## 5. 要件品質チェックリスト

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---------|--------|--------|--------|--------|--------|
| REQ-001 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-002 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-003 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-004 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-005 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-006 | ☑ | ☑ | ☑ | ☑ | ☑ |

---

## 6. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-15 | 初版作成（Issue #92 対応） | Antigravity AI |
