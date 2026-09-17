# ADR 0016: Python API (PyO3) 境界での中間 PyClass 排除および Placement EV 他家スコア事前ソート化

- **ステータス**: Accepted
- **日付**: 2026-09-17
- **対象**: `src/mahjong/placement_ev.rs`, `src/python_api.rs`
- **関連 Issue**: #106

---

## コンテキストと課題

Python 連携 API において、`py_get_ai_hud_data` や打牌順位期待値計算（Placement EV）の呼び出し頻度が増加した際、以下のオーバーヘッドがボトルネックとなっていました：

1. `evaluate_hand_discards_with_placement` は候補牌（最大14枚）ごとに着順確率計算（`estimate_rank_probabilities`）を3シナリオで実行するが、内部で毎回 `Vec` を生成し他家3名のスコアをソートしていた（計最大42回のヒープアロケーションとソート）。
2. `py_get_ai_hud_data` は HUD 表示用 dict を返す API であるにもかかわらず、内部で `py_evaluate_placement_discards` を経由して中間 PyClass（`Vec<PyPlacementEvaluation>`）を生成し、その直後にフィールド再読み出しを行って `PyDict` に詰め直していた。

また、当初懸念されていた文字列アロケーション（`TileName::as_str()` や `mpsz()`）については、実態が `&'static str` であり Rust 側ヒープ確保は発生していないことが判明したため、無用な文字列キャッシュ等の過剰設計は避ける必要がありました。

---

## 決定事項

1. **`placement_ev.rs` における事前ソート配列 `[i32; 3]` の導入**:
   - `evaluate_hand_discards_with_placement` の外側で他家3名のスコアを固定長配列 `[i32; 3]` に降順ソートして保持する。
   - `estimate_rank_probabilities` にはその参照 `&[i32; 3]` を渡し、候補・シナリオごとの `Vec` 動的確保およびソートを完全ゼロ化する。
2. **`python_api.rs` における中間 PyClass の排除**:
   - 入力検証・コンテキスト構築・Rust評価を行う内部共通関数 `evaluate_placement_discards_internal` を抽出する。
   - `py_evaluate_placement_discards`（公開API）は従来通り `Vec<PyPlacementEvaluation>` を返し互換性を維持する。
   - `py_get_ai_hud_data` は内部関数の Rust 結果（`PlacementCandidateEvaluation`）から直接 1 パスで `PyDict` / `PyList` を構築し、中間 PyClass の生成を完全排除する。
   - オーラス条件（`orasu_conditions`）も同様に `calculate_orasu_conditions` から直接 `PyDict` を構築する。

---

## メリットと影響

- **性能向上**: 候補牌ごとの最大42回の動的確保・ソートが完全排除され、さらに PyO3 境界での不要な Python オブジェクトインスタンス化・解放オーバーヘッドが削減される。
- **100% 後方互換性**: 公開 Python API のシグネチャ、戻り値の型・キー構成、および例外動作は完全に維持される。
- **保守性**: 入力検証・コンテキスト構築コードが共通化され、二重管理が解消される。
