# ADR-0005: `mahjong-ui` アプリケーションの技術スタック選定

> **ADR番号:** 0005  
> **作成日:** 2026-09-14  
> **更新日:** 2026-09-14  
> **作成者:** Syun & AI Assistant

---

## ステータス

`Accepted`

---

## コンテキスト（Context）

麻雀AI学習プラットフォームのGUIとして、新設リポジトリ `mahjong-ui` を作成することとなった。  
基盤ライブラリ `rust-mahjong` の Python API / 辞書シリアライズ基盤を最大限に活かし、低遅延なリアルタイム対局通信とリッチでレスポンシブなグラフィックスUI（麻雀卓・AI HUD・何切るドリル）を実現するための技術スタック選定が必要となった。

---

## 意思決定の要因（Decision Drivers）

- **`rust-mahjong` との親和性**: Python ライブラリとして開発されたコア資産をそのまま `import mahjong` して利用できること。
- **リアルタイム性と開発速度**: WebSocket による低遅延な局進行イベント配信と、Vite/React による高速なUIコンポーネント開発・ホットリロード。
- **UI表現力と保守性**: 麻雀牌の視覚的配置、レスポンシブな四角形卓レイアウト、HUDの要因分解カード等を美しくスタイリングできること。

---

## 検討した選択肢（Considered Options）

- **選択肢 A (採用): Python (FastAPI / Uvicorn) バックエンド ＋ React (TypeScript / Vite / Tailwind CSS) フロントエンド**
- **選択肢 B: Python デスクトップGUI (CustomTkinter / PyQt)**
- **選択肢 C: Next.js (Node.js) ＋ Python マイクロサービス**

---

## 決定結果（Decision Outcome）

**選択した選択肢:** **選択肢 A (FastAPI + React/Vite/Tailwind CSS)** を採用。

**理由:**
1. **バックエンド (FastAPI)**:
   - `rust-mahjong` をインプロセスで直接インポートして `to_dict()` されたデータをそのまま WebSocket / REST でクライアントへ送信可能。
   - 非同期WebSocketハンドラーが標準で備わっており、手番同期やCPU打牌のディレイアニメーション配信が容易。
2. **フロントエンド (React + TypeScript + Vite + Tailwind CSS)**:
   - コンポーネント指向により、卓（手牌、河、副露、ドラ）、HUD（EVランキング、レーダー、解説）、ドリルを疎結合に保守可能。
   - Tailwind CSS による直感的な麻雀卓スタイリングとアニメーション。
   - 将来的に Tauri を被せてデスクトップ exe アプリとして配布することも容易。

### 期待される効果（Positive Consequences）
- バックエンド・フロントエンドが明確に分離され、型安全かつ迅速な開発が可能。
- Webブラウザさえあればプラットフォーム（Windows, Mac, モバイル）を問わずプレイ・学習可能。

---

## 参考資料（References）

- 要件定義書: `docs/requirements/mahjong-ui-application.md`
- 基本設計書: `docs/design/mahjong-ui-application.md`
