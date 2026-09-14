# ADR-0004: UIリポジトリの分離と本ライブラリにおけるUI連携用Python API・シリアライズ基盤の提供

> **ADR番号:** 0004  
> **作成日:** 2026-09-14  
> **更新日:** 2026-09-14  
> **作成者:** Syun & AI Assistant

---

## ステータス

`Accepted`

---

## コンテキスト（Context）

本プロジェクト（`rust-mahjong`）は、Rustで記述された超高速な麻雀数理・判定・AI評価エンジンを `maturin` + `PyO3` で Python 向けライブラリとして提供している。  
Web / GUI インターフェース（麻雀卓描画、AI HUD、何切るドリル、悪手レビュー）の構築にあたり、UI を本リポジトリ内に内包するのか、別リポジトリとして分離するのか、また本ライブラリがどのようなインターフェース（責務境界）を提供すべきかの決定が必要となった。

---

## 意思決定の要因（Decision Drivers）

- **Python ライブラリとしての責務の純粋性**: 本リポジトリは高速な麻雀数理・AIエンジンを提供するコアライブラリであり、WebフレームワークやUIアセットの混入による肥大化を防ぐこと。
- **UI 開発の独立性と柔軟性**: UIリポジトリ側で React, Next.js, Electron, Tauri, FastAPI 等、最適なフロントエンド/フルスタック構成を自由に選定・更新できること。
- **データ連携の容易性**: UI開発側が `import mahjong` するだけで、複雑な麻雀ルールや盤面状態、AI評価値をそのまま JSON / dict 形式で取得して即座に画面へ渡せること。

---

## 検討した選択肢（Considered Options）

- **選択肢 A (パターン1 / 採用): UIは別リポジトリとし、本リポジトリは純粋なPythonライブラリとしてUI連携用API & シリアライズ基盤を提供する**
- **選択肢 B (パターン2): 本リポジトリが FastAPI / axum による API/WebSocket サーバーを提供し、UIリポジトリは純粋なフロントエンドのみとする**
- **選択肢 C: 本リポジトリ内に Web フロントエンドおよびサーバーを一体内包する（モノレポ）**

---

## 決定結果（Decision Outcome）

**選択した選択肢:** **選択肢 A (パターン1)** を採用。

**理由:**
1. 本リポジトリを純粋な Python ライブラリとして維持できるため、`pip install rust-mahjong` の配布形態や機械学習・データ分析用途との一貫性が完全に保たれる。
2. UIリポジトリ側は `rust-mahjong` を通常通りインポートし、自らのアーキテクチャ（FastAPI + React や Next.js 等）でバックエンドとフロントエンドを構築できる。
3. 本リポジトリ側で「卓状態（`TableState`）」「順位EV・逆転条件（`PlacementEvaluation`）」「AI HUD（`get_ai_hud_data`）」「何切る（`DrillProblem`）」の `to_dict()` メソッドおよび完全な Python 型定義（`_core.pyi`）を整備することで、UIリポジトリ側の開発負荷を最小化できる。

### 期待される効果（Positive Consequences）
- 本リポジトリの依存関係がシンプルに保たれ、CI/CD・PyPIビルドが高速・堅牢になる。
- UIリポジトリ側でPythonの機械学習モデルやカスタムAIルールを容易に組み込める。
- 型定義（`_core.pyi`）の充実により、UI開発時のIDE入力補完が完全に機能する。

### 想定されるリスク（Negative Consequences）
- リポジトリが分かれるため、データ構造変更時のバージョン連携（`rust-mahjong` のバージョニング）を適切に管理する必要がある。

---

## 参考資料（References）

- 要件定義書: `docs/requirements/ui-integration-api.md`
- 基本設計書: `docs/design/ui-integration-api.md`
- [PyO3 公式ドキュメント](https://pyo3.rs/)
