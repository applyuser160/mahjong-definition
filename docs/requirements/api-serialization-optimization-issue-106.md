# 要件定義書: Python API (PyO3) 境界でのシリアライズ・中間オブジェクト削減 & 順位点計算最適化

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> GitHub Issue: #106「perf(api): Python API (PyO3) 境界でのシリアライズ・文字列アロケーションの削減」に対応します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | Python API シリアライズ最適化 & Placement EV 最適化（Issue #106） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

Python API（PyO3）経由で打牌評価や AI HUD データを取得する処理（`py_get_ai_hud_data`）および順位期待値計算（`evaluate_hand_discards_with_placement`）において、以下の計算・オブジェクト生成オーバーヘッドが存在しています：

1. **`placement_ev.rs` における候補ごとの重複ソートと動的メモリ確保**:
   - `evaluate_hand_discards_with_placement` は打牌候補（最大14）ごとに `estimate_rank_probabilities` を3シナリオ（和了・放銃・その他）で呼び出す（計最大42回）。
   - 各呼び出しにおいて、変化しない他家3人の持ち点を毎回 `Vec` へ push して降順ソートしており、計最大42回のヒープアロケーションと3要素ソートが無駄に反復実行されている。
2. **`py_get_ai_hud_data` における中間 PyClass 生成とフィールド再読み出し**:
   - `py_get_ai_hud_data` は内部で `py_evaluate_placement_discards` を呼び出し、まず `Vec<PyPlacementEvaluation>`（PyClass）を生成している。
   - その直後に各 PyClass インスタンスから `.to_dict(py)` を呼び出して Python 辞書を構築しているが、HUD API 自体は PyClass の配列を返さないため、この中間 PyClass 群の生成と解放が純粋なオーバーヘッドとなっている。
   - 同様にオーラス条件判定（`py_calculate_orasu_conditions`）も `Vec<PyWinCondition>` を経由して辞書化されている。

本改修の目的は：
- `placement_ev.rs` 側で他家スコアのソートを評価開始時に1度だけ固定長配列 `[i32; 3]` に事前集計し、確率推定関数への引数として引き渡すことで、ヒープアロケーションとソートのオーバーヘッドを完全排除すること。
- Python API 境界において、内部の入力検証・手牌/コンテキスト構築・Rust評価処理を共有ヘルパー関数へ抽出し、`py_get_ai_hud_data` は Rust の評価結果構造体から直接 `PyDict` / `PyList` を 1 パスで構築するようにして中間 PyClass 生成を排除すること。
- 公開 Python API（`get_ai_hud_data`, `evaluate_placement_discards`, `calculate_orasu_conditions`）のシグネチャ・レスポンス構造・例外挙動の後方互換性を 100% 維持すること。

### 1.2 スコープ

**スコープ内:**
- `placement_ev.rs`: 他家スコアの事前ソート配列 `[i32; 3]` の導入と `estimate_rank_probabilities` のゼロアロケーション化
- `placement_ev.rs`: 旧実装と新実装の順位確率・Placement EV 同値性検証テストの追加
- `python_api.rs`: `evaluate_placement_discards_internal` の抽出と、`py_get_ai_hud_data` からの中間 `PyPlacementEvaluation` / `PyWinCondition` 排除
- `python_api.rs`: 既存 HUD 出力との完全なキー・値一致、不正入力時の例外一致の検証
- ベンチマーク `benches/python_api_benchmark.rs`: オーラス局面・複数候補条件における `py_get_ai_hud_data` のエンドツーエンドベンチマークの追加と測定

**スコープ外:**
- `TileName::as_str()` / `mpsz()` は既に `&'static str` であり Rust 文字列の動的アロケーションは発生していないため変更しない
- Python 側に返す辞書・リストのスキーマ変更（GUI/CLI への影響を防ぐため互換性維持）

---

## 2. 機能要件 (FR)

### FR-1: 他家持ち点配列の事前ソート化
- `evaluate_hand_discards_with_placement` のループ外（先頭）で、他家3名の持ち点を固定長スタック配列 `[i32; 3]` に抽出し、降順ソートして保持すること。
- `estimate_rank_probabilities` は `ctx` 全体と `player_idx` ではなく、事前ソート済み `&[i32; 3]` と予測スコア `i32` を受け取るように変更し、内部での `Vec` 生成・ソートを一切行わないこと。

### FR-2: 内部評価ヘルパー関数の抽出
- `python_api.rs` 内に、入力引数の検証（`player_idx < 4`, `dealer_idx < 4`）、手牌/牌面/分析コンテキストの構築、および `evaluate_hand_discards_with_placement` の呼び出しを行うプライベート関数 `evaluate_placement_discards_internal` を定義すること。
- 戻り値として `(MatchContext, Vec<PlacementCandidateEvaluation>)` を返すこと。

### FR-3: HUD 辞書の直接生成（中間 PyClass 排除）
- `py_get_ai_hud_data` は上記ヘルパーから得られた `Vec<PlacementCandidateEvaluation>` より直接 `PyDict` / `PyList` を構築すること。
- `candidates` リスト内の各要素、`best_*` フィールド、およびオーラス時の `orasu_conditions` を中間 PyClass を介さずに直接構築すること。

### FR-4: 後方互換性と例外の完全一致
- `py_evaluate_placement_discards` は従来通り `Vec<PyPlacementEvaluation>` を返し、公開インターフェースを変更しないこと。
- `py_get_ai_hud_data` の返す辞書の全キー・値・型・リスト順序が従来と完全に一致すること。
- 範囲外の `player_idx` や `dealer_idx` 等の入力に対する `PyValueError` の送出条件が一致すること。

---

## 3. 非機能要件 (NFR)

### NFR-1: 性能向上
- `evaluate_hand_discards_with_placement` 内でのアロケーション回数を候補数 × 3回分削減（0回に）。
- `py_get_ai_hud_data` 呼び出しにおける中間 Python オブジェクト生成を削減し、オーラス局面を含むエンドツーエンド実行時間を短縮すること。

### NFR-2: 決定論的安定性 & 正確性
- 新旧実装で順位確率分布および Placement EV の計算結果が浮動小数点誤差の範囲内で完全に一致すること。

---

## 4. 検証・完了基準 (DoD)

- [ ] `estimate_rank_probabilities` がヒープ確保なし（ゼロアロケーション）で動作すること。
- [ ] 既存の Rust 単体テスト（`cargo test`）がすべてパスすること。
- [ ] 既存の Python テスト（`pytest`）がすべてパスすること。
- [ ] HUD 出力のキー・値・例外の完全一致テストが追加され、パスすること。
- [ ] `benches/python_api_benchmark.rs` に `py_get_ai_hud_data`（オーラス条件・複数打牌候補）ベンチマークが追加され、性能向上が確認できること。
