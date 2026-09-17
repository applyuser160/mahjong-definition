# 要件定義書: コードベースのリファクタリング及びコード品質改善

> ISO/IEC/IEEE 29148 をベースにした軽量版要件定義書です。  
> パフォーマンス最適化PR群（#110〜#115）マージ後のコードベース健全化および保守性向上を目的とします。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | コードベースのリファクタリング（Clippy警告解消・重複排除・DRY化） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的

Issue #104〜#109 のパフォーマンス改善PR（#110, #111, #112, #114, #115）が `main` にマージされた結果、多数の最適化機能（`ShantenState` 差分更新、`SafetyFeatures` 事前集計、SIMD ベクトル化等）が追加されました。
しかし、これに伴い以下の保守性・品質上の課題が生じています：
1. **Clippy 警告による CI 失敗**:
   - `acceptance.rs` および `expectation.rs` にて `clippy::needless_range_loop` が検知され、`-D warnings` でビルドがブロックされている。
2. **コードの重複（DRY違反）**:
   - `acceptance.rs` 内で萬子・筒子・索子の $\pm 2$ ビットマスク生成ロジックが 3 箇所に重複記述されている。
   - `shanten.rs` の `ShantenState::new` 内で七対子・国士無双の集計ループがインライン記述されており、通常計算との整合・再利用性が低下している。
3. **不要な中間バッファ**:
   - `expectation.rs` の `SafetyFeatures::from_context` で固定長の一時配列 `riichi_opponents` を確保してから再走査している。

本改修の目的は、外部仕様・計算結果・実行性能を 100% 維持したまま、コードベースをクリーン化し、Clippy 警告をゼロにすることです。

### 1.2 スコープ

**スコープ内:**
- `acceptance.rs`: `needless_range_loop` の解消、および数牌 $\pm 2$ ビットマスク算出の共通化（`suit_neighbor_mask`）
- `expectation.rs`: `needless_range_loop` の解消、および `SafetyFeatures::from_context` での中間配列の排除
- `shanten.rs`: `ShantenState::new` 内の特殊形集計ヘルパーの整理
- 全単体テスト・プロパティテスト・Pythonテストの合格確認
- `cargo clippy --all-targets -- -D warnings` の完全通過
- `cargo fmt --all -- --check` の完全通過

**スコープ外:**
- 公開 API のシグネチャ変更
- 新機能の追加

---

## 2. 機能要件 (FR)

### FR-1: Clippy 警告の解消
- `cargo clippy --all-targets -- -D warnings` が一切のエラーおよび警告なしで正常終了すること。

### FR-2: 枝刈りマスク生成の共通化
- `acceptance.rs` において、萬子・筒子・索子（1..=27）に対する周辺牌マスク生成ロジックを共通化し、可読性と保守性を向上させること。

### FR-3: 安全度特徴量抽出の効率化
- `expectation.rs` において、`ctx.riichi_status` を直接走査して `riichi_opponents` 中間配列を不要とすること。

---

## 3. 非機能要件 (NFR)

### NFR-1: 100% の後方互換性と同値性
- 全 150 以上の Rust テストおよび 19 件の Python テストがすべて PASS すること。

### NFR-2: 性能の維持
- ベンチマークにおいて性能低下（リグレッション）が発生しないこと。