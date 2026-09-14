# ADR-0006: 対局コンテキスト（巡目・風・可視牌）の動的連携方式と後方互換性維持

> **ADR番号:** 0006  
> **作成日:** 2026-09-14  
> **更新日:** 2026-09-14  
> **作成者:** AI Assistant  
> **ステータス:** Accepted  

---

## コンテキスト（Context）

麻雀AIエンジン（`rust-mahjong`）における打牌評価（`evaluate_hand_discards`）および順位期待値評価（`evaluate_placement_discards`）は、これまで `turn_number=6`, `remaining_wall_tiles=50`, `seat_wind=East`, `round_wind=East`, `visible_counts=None` をハードコードして評価を行っていた。  
`mahjong-ui` の `MatchManager` から呼び出す際、実際の巡目、残り山枚数、自風・場風、および河・副露・手牌・ドラ表示牌によって確定している可視牌情報を渡し、実戦に即した精度の高い評価・悪手レビューを行う必要が生じた。

---

## 意思決定の要因（Decision Drivers）

1. **完全な後方互換性**: 既存の単体テストや簡易呼び出しスクリプト（手牌のみ渡す形式）を破損させないこと。
2. **パフォーマンス（低オーバーヘッド）**: 打牌ごとに可視牌を集約・受け渡す際のメモリコピーや処理時間を最小限（ミリ秒未満）に抑えること。
3. **境界責務の明確化**: 可視牌の枚数カウント配列（`[u8; 35]`）への変換処理を Rust 側で安全に行うか、Python 側で行うか。

---

## 検討した選択肢（Considered Options）

- **選択肢 A (採用): Python API のシグネチャにオプショナル引数を追加し、可視牌リストを Rust 側で `[u8; 35]` カウント配列へ変換する**
  - Python API の既存の引数（`tiles`, `is_dealer`, `dora_indicators`）に加え、`turn_number`, `remaining_wall_tiles`, `seat_wind`, `round_wind`, `visible_tiles` を `Option`（Python では `None` デフォルト）として追加する。
  - `visible_tiles`（`Vec<PyTileName>`）を Rust 側で受け取り、Rust 内で `[u8; 35]` の枚数カウント配列へ一括集約する。

- **選択肢 B: 局面コンテキストを表す専用の Python クラス（`AnalysisContext`）を新設して渡す**
  - クラスの初期化と受け渡しが必要となり、Python 呼び出し側のコード量が増加する。

- **選択肢 C: 可視牌の 35 要素配列を Python 側で生成して渡す**
  - Python 側で牌インデックスへのマッピングやカウント処理を行う必要があり、Rust の型安全性が活かせない。

---

## 決定結果（Decision Outcome）

**選択肢 A** を採用。

### 採用理由:
1. Python の呼び出し側は `visible_tiles: List[TileName]` をそのまま渡すだけで済み、Python 側での複雑な配列計算が不要。
2. 省略時は従来のデフォルト値（6巡目、50枚、東風、可視牌=手牌のみ）が自動適用されるため、既存テストやベンチマークがそのまま動作する。
3. Rust 側での `Vec<PyTileName>` から `[u8; 35]` への変換は数マイクロ秒以下で完了するため、パフォーマンスへの影響が極めて軽微。

---

## 関連ドキュメント

- 要件定義書: `docs/requirements/match-context-integration.md`
- 設計書: `docs/design/match-context-integration.md`
- 関連Issue: `mahjong-ui#1`, `mahjong#80`
