# 設計書：麻雀AI学習プラットフォーム Web UI アプリケーション (`mahjong-ui`)

> C4モデル（Context / Container / Component）準拠  
> 本設計書は、参照専用の `definition/templates/design.md` のテンプレートに基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-ui |
| モジュール名 | mahjong-ui (麻雀AI学習 Web/GUI アプリケーション) |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-14 |
| 最終更新日 | 2026-09-14 |
| 作成者 | Syun & AI Assistant |
| ステータス | Review |

---

## 1. Level 1: システムコンテキスト図（Context Diagram）

### 1.1 説明
`mahjong-ui` は、プレイヤーがブラウザ上で麻雀対局、AI HUD によるリアルタイム指導、何切るドリル学習を行うためのフルスタック Web アプリケーションです。  
基盤となる麻雀数理・ルール判定・AI評価・シリアライズには、別リポジトリの Python ライブラリ `rust-mahjong` を使用します。

### 1.2 コンテキスト図

```mermaid
graph TB
    User["👤 プレイヤー（ユーザー）<br/>[Person]<br/>ブラウザで対局・HUD・ドリルを操作"]

    subgraph MahjongUISystem ["mahjong-ui (Web アプリケーション)"]
        App["🖥️ mahjong-ui<br/>[Software System]<br/>卓ビュー、HUD描画、WebSocket通信、ドリル提供"]
    end

    subgraph MahjongCoreLib ["rust-mahjong (別リポジトリ)"]
        CoreLib["📦 rust-mahjong (Python Library)<br/>[Library]<br/>卓状態、順位EV、何切る、シリアライズ"]
    end

    User -->|"操作 (打牌, 鳴き, 回答)"| App
    App -->|"卓面・HUD・解説文表示"| User
    App -->|"import mahjong"| CoreLib
```

---

## 2. Level 2: コンテナ図（Container Diagram）

### 2.1 コンテナ図

```mermaid
graph TB
    User["👤 プレイヤー<br/>[Person]"]

    subgraph MahjongUI ["mahjong-ui システム"]
        Frontend["🌐 フロントエンド<br/>[Container: React / TypeScript / Vite / Tailwind CSS]<br/>卓描画, 牌操作, AI HUD, ドリル画面, 悪手タイムライン"]
        Backend["⚙️ バックエンドサーバー<br/>[Container: Python / FastAPI / Uvicorn]<br/>REST API, WebSocket セッション管理, 対局進行ロジック"]
    end

    subgraph CoreLib ["外部依存ライブラリ"]
        RustMahjong["📦 rust-mahjong<br/>[Python Package: PyO3 + Rust]<br/>TableState, PlacementEvaluation, DrillProblem, Explainer"]
    end

    User -->|"ブラウザ操作 (HTTPS / WS)"| Frontend
    Frontend <== "WebSocket (打牌・手番通知)" ==> Backend
    Frontend -. "REST API (ドリル・ヘルスチェック)" .-> Backend
    Backend -->|"import mahjong"| RustMahjong
```

### 2.2 コンテナ一覧

| コンテナ名 | 技術スタック | 責務 | 通信プロトコル |
|-----------|------------|------|--------------|
| フロントエンド | React 18, TypeScript, Vite, Tailwind CSS, Lucide Icons | 4人卓描画、牌操作、AI HUDパネル、ドリルUI、悪手レビュー | HTTP / WebSocket |
| バックエンド | Python 3.10+, FastAPI, Uvicorn, WebSockets | 対局セッション制御、CPU思考委譲、REST/WSエンドポイント | In-process (Python) |
| コアライブラリ | `rust-mahjong` (PyO3) | 牌理、点数計算、順位期待値、ドリル問題生成、辞書シリアライズ | C-API / PyO3 |

---

## 3. Level 3: コンポーネント図（Component Diagram）

### 3.1 バックエンド構成 (`backend/`)

```mermaid
graph TB
    subgraph FastAPIApp ["FastAPI アプリケーション"]
        Router["main.py (FastAPI App & CORS)"]
        WsManager["match_session.py (WebSocket 対局セッション管理)"]
        DrillRouter["drill_router.py (何切るドリル REST エンドポイント)"]
    end

    subgraph CoreIntegration ["rust-mahjong 連携"]
        TableWrapper["TableState (卓スナップショット)"]
        HudGenerator["get_ai_hud_data (HUDデータ一括生成)"]
        DrillWrapper["DrillProblem / CallAdvice"]
    end

    Router --> WsManager
    Router --> DrillRouter
    WsManager --> TableWrapper
    WsManager --> HudGenerator
    DrillRouter --> DrillWrapper
```

### 3.2 フロントエンド構成 (`frontend/src/`)

```mermaid
graph TB
    subgraph AppRoot ["App Root"]
        App["App.tsx (タブナビゲーション: 対局 / ドリル / レビュー)"]
    end

    subgraph Views ["画面ビュー"]
        TableView["TableView.tsx (4人卓 & AI HUD 統合画面)"]
        DrillView["DrillView.tsx (何切るドリル画面)"]
        ReviewView["ReviewView.tsx (悪手タイムライン画面)"]
    end

    subgraph Components ["主要コンポーネント"]
        TableLayout["TableLayout.tsx (四角形麻雀卓・4家配置)"]
        TileComponent["Tile.tsx (萬筒索字牌のビジュアル表示)"]
        ActionModal["ActionPrompt.tsx (鳴き・リーチ・ロン選択)"]
        HudPanel["AiHudPanel.tsx (打牌EVランキング・要因分解・解説)"]
    end

    App --> TableView
    App --> DrillView
    App --> ReviewView

    TableView --> TableLayout
    TableView --> HudPanel
    TableLayout --> TileComponent
    TableLayout --> ActionModal
```

---

## 4. ディレクトリ構成設計

```
mahjong-ui/
├── backend/
│   ├── pyproject.toml         # FastAPI, uvicorn, websockets, rust-mahjong
│   ├── main.py                # FastAPI エントリーポイント
│   ├── match_manager.py       # 4人CPU対局進行・状態マシン
│   └── routers/
│       ├── match.py           # WebSocket /ws/match
│       └── drill.py           # REST /api/drill/*
├── frontend/
│   ├── package.json           # React, Vite, Tailwind CSS, TypeScript
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── App.tsx            # メイン画面（タブ切替）
│       ├── components/
│       │   ├── Table/         # 麻雀卓・手牌・河・アクション
│       │   ├── HUD/           # AI HUD パネル・要因分解カード
│       │   ├── Drill/         # 何切るドリル
│       │   └── Common/        # 牌描画 (Tile.tsx)
│       ├── services/          # API & WebSocket クライアント
│       └── types/             # TypeScript 型定義 (TableState, HudData等)
├── README.md
└── .gitignore
```

---

## 5. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | 新設リポジトリ `mahjong-ui` の基本設計書として初版作成 | Syun & AI Assistant |
