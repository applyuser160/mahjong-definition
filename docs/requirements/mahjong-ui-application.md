# 要件定義書：麻雀AI学習プラットフォーム Web UI アプリケーション (`mahjong-ui`)

> ISO/IEC/IEEE 29148 準拠（SRS簡易版）  
> 本要件定義書は、参照専用の `definition/templates/requirements.md` のテンプレートに基づき作成されています。

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

## 1. 目的・スコープ

### 1.1 目的
`rust-mahjong`（高性能麻雀エンジン・順位EV・AI評価ライブラリ）を活用し、プレイヤーが直感的かつグラフィカルに4人CPU対局、リアルタイムAI HUD学習、何切るドリル特訓、および終局後の悪手検討を行えるリッチな Web UI アプリケーション（`mahjong-ui`）を新設・提供します。

### 1.2 スコープ

**スコープ内:**
- **リポジトリ新設 (`mahjong-ui`)**:
  - GitHub リポジトリ `applyuser160/mahjong-ui` の新規開設とローカル環境セットアップ。
- **バックエンド (`backend/`)**:
  - Python FastAPI による REST API & WebSocket サーバー。
  - `rust-mahjong` ライブラリのインポートによる卓状態管理（`TableState`）、AI HUD データ生成（`get_ai_hud_data`）、順位期待値（`PlacementEvaluation`）、何切るドリル（`DrillProblem`）、悪手ログ（`ReviewTracker`）。
  - 対局セッション管理（プレイヤー打牌、CPU自動打牌、鳴き・ロン判定、局進行）。
- **フロントエンド (`frontend/`)**:
  - React + TypeScript + Vite + Tailwind CSS によるモダンなSPA。
  - **4人卓ビュー**:
    - 自手牌・ツモ牌（クリックで打牌）、他家手牌、副露面子、河、点棒、ドラ表示牌、風表示。
    - 副露アクションポップアップ（チー、ポン、カン、リーチ、ロン、ツモ、パス）。
  - **リアルタイム AI HUD パネル**:
    - 推奨打牌ランキング（素点EV、順位Pt EV、受入枚数、向聴数）。
    - 局面要因分解カード（速度・打点・放銃安全度）。
    - 日本語解説文およびオーラス逆転条件。
  - **何切るドリル特訓画面**:
    - ランダム出題、回答打牌選択、即時正誤・EV損失・要因解説レビュー。
  - **悪手タイムライン検討画面**:
    - 終局後、EV損失の大きかった巡目をリスト化し、盤面再現とともに振り返る機能。

**スコープ外:**
- インターネット経由の不特定多数とのマルチプレイヤーオンラインマッチング（ローカルまたはLAN内でのシングルプレイヤー vs CPUを対象）。

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| プロジェクトオーナー | ユーザー | 高 |
| 設計・実装 | AI Assistant | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| `mahjong-ui` | 本アプリケーションのリポジトリ名。 |
| `rust-mahjong` | コア数理・判定・EV計算を提供する基盤 Python ライブラリ。 |
| MPSZコード | 萬子(1m-9m)、筒子(1p-9p)、索子(1s-9s)、字牌(1z-7z)を表す標準英数字表記。 |
| 卓ビュー | 麻雀卓の四方（東南西北）の牌・河・点棒を描画するメイン盤面。 |
| AI HUD | 卓の横に常時表示され、リアルタイムに打牌推奨理由や期待値を提示する情報画面。 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ
- ブラウザを開くだけで、グラフィカルな美しい麻雀卓でAIと対局したい。
- 1巡ごとにAIが何を考え、どの牌をなぜ推奨しているのか（要因分解・順位期待値）をリアルタイムに学びたい。
- 何切るドリルを手軽に解いて牌効率と点況判断の勘を養いたい。
- 局が終わった後、「どこで悪手を打ったか」を盤面再現つきで復習したい。

### 3.2 前提条件・制約
- **動作環境**: モダンブラウザ（Chrome, Edge, Safari, Firefox）。
- **Python環境**: Python >= 3.10, FastAPI, Uvicorn, `rust-mahjong`。
- **Node.js環境**: Node >= 18, Vite, React 18, TypeScript。

---

## 4. 機能要件

### 4.1 バックエンド (FastAPI)

#### REQ-APP-001: 対局セッションとWebSocket制御
**説明:**  
The backend shall manage match state using `rust-mahjong` and broadcast table updates and AI hints to the frontend via WebSocket.

**受入条件:**
- [ ] WebSocket接続時に新規対局を開始し、初期配牌と卓状態（`TableState.to_dict()`）を返却する。
- [ ] プレイヤーの打牌または鳴きアクションを受信した際、合法手を検証し、CPUの反応を含めて盤面を進行・配信する。

**優先度:** Must  

---

#### REQ-APP-002: 何切るドリル REST API
**説明:**  
The backend shall provide REST endpoints `GET /api/drill/question` and `POST /api/drill/answer`.

**受入条件:**
- [ ] `GET /api/drill/question` で何切る問題データを返却する。
- [ ] `POST /api/drill/answer` で正誤判定、最善手、EV差、要因解説を返却する。

**優先度:** Must  

---

### 4.2 フロントエンド (React UI)

#### REQ-APP-003: 卓ビューと手牌操作
**説明:**  
The frontend shall render the 4-player mahjong table and support responsive tile interactions.

**受入条件:**
- [ ] 牌の画像またはSVG/CSS表現がMPSZコードに基づいて美しく描画される。
- [ ] 自手牌の牌をクリックすることで打牌が送信される。
- [ ] 鳴き・リーチ・ロンが可能な場合、アクションボタンがポップアップする。

**優先度:** Must  

---

#### REQ-APP-004: AI アシスト HUD パネル
**説明:**  
The frontend shall display the real-time AI advice HUD adjacent to the mahjong table.

**受入条件:**
- [ ] 推奨打牌ランキング（上位候補、素点EV、順位Pt EV、受入枚数）を表示する。
- [ ] 局面要因分解（速度、打点、放銃安全度）および日本語解説テキストを表示する。
- [ ] オーラス時には逆転条件（満貫ツモ、直撃等）を自動表示する。

**優先度:** Must  

---

#### REQ-APP-005: 何切るドリル & 悪手検討モード
**説明:**  
The frontend shall provide a dedicated drill tab and post-match blunder review timeline.

**受入条件:**
- [ ] ナビゲーションバーから「対局」「何切るドリル」「悪手検討」を切り替えられる。
- [ ] ドリル画面でテンポよく牌を選択し、解説を確認できる。

**優先度:** Must  

---

## 5. 要件品質チェックリスト

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---------|:------:|:------:|:------:|:------:|:------:|
| REQ-APP-001 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-APP-002 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-APP-003 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-APP-004 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-APP-005 | ✓ | ✓ | ✓ | ✓ | ✓ |

---

## 6. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | 新設リポジトリ `mahjong-ui` の要件定義書として初版作成 | Syun & AI Assistant |
