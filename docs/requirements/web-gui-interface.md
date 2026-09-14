# 要件定義書：Web / GUI インターフェース

> ISO/IEC/IEEE 29148 準拠（SRS簡易版）  
> 本要件定義書は、参照専用の `definition/templates/requirements.md` のテンプレートに基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-learning-assistant |
| モジュール名 | web-gui-interface (麻雀AI Web/GUI インターフェース) |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-14 |
| 最終更新日 | 2026-09-14 |
| 作成者 | Syun & AI Assistant |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的
これまでCLI（コマンドライン）上で実装・検証してきた「麻雀コアエンジン」「4人CPU対局」「何切るドリル」「副露判断アドバイザー」「リアルタイムEVヒントHUD」「点況判断・順位期待値（Placement EV）エンジン」を、直感的かつグラフィカルに操作・学習できるWeb/GUIインターフェースとして提供します。
ユーザーが牌の視覚的配置や点況レーダー、AI思考プロセス（要因分解・推奨打牌・日本語解説文）をストレスなく閲覧・操作できるリッチな学習環境を構築します。

### 1.2 スコープ

**スコープ内:**
- **バックエンド API / WebSocket サーバー (`crates/mahjong_server`)**:
  - Rust製Webフレームワーク（`axum`）によるHTTP REST APIおよびWebSocket双方向通信基盤の実装。
  - セッション管理（対局セッション、何切るドリルセッション）。
  - 局面状態（手牌・ツモ牌・河・副露・山牌・点棒・風・ドラ）とAI計算結果（打牌EVランキング・要因分解・解説文・オーラス逆転条件）のJSONシリアライズ配信。
- **フロントエンド Web UI (`web/`)**:
  - React + TypeScript + Vite によるSPA（Single Page Application）の構築。
  - **対局画面（卓ビュー）**:
    - 4人の手牌・河・副露面子・点棒・供託・本場・親マーク・ドラ表示牌のリアルタイム描画。
    - ユーザーの手番時の直感的な打牌選択（クリック／タップ）および副露アクションボタン（チー、ポン、カン、リーチ、ロン、ツモ、パス）。
  - **リアルタイム AI アシスト HUD パネル**:
    - 推奨打牌ランキング（上位候補の打牌、素点EV、順位EV、受入枚数、向聴数）。
    - 局面要因分解カード（速度・打点・放銃安全度・着順確率分布）。
    - 日本語解説文およびオーラス逆転条件の自動表示。
  - **何切るドリル & 悪手検討画面**:
    - ランダム／指定ドリル問題の出題・回答・正誤判定UI。
    - 対局終了後の全巡目タイムライン再生および悪手（EV損失牌）の振り返りビューア。

**スコープ外:**
- インターネット経由での見知らぬ他者とのオンライン対戦（本フェーズではローカル環境または同一LAN内でのシングルプレイヤー vs CPU学習用）。
- 課金・ユーザー認証管理基盤（スタンドアロン学習ツールとしての利用を前提とする）。

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| プロジェクトオーナー | ユーザー | 高 |
| 設計・実装 | AI Assistant | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| SPA | Single Page Application。ブラウザ側で画面描画と状態管理を行うWebアプリケーション形態。 |
| WebSocket | 双方向の低遅延通信プロトコル。手番通知や打牌イベントのリアルタイム配信に使用。 |
| HUD | Heads-Up Display。卓面と並行して常時表示されるAIアシスタント情報ウィンドウ。 |
| EV損失 | 最善手とユーザーの選択打牌との期待値差（悪手判定基準）。 |
| 卓ビュー | 麻雀牌、河、点棒、東南西北の風、ドラ表示牌が配置された全景画面。 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ
- CLIの文字列表示ではなく、美しい麻雀牌画像や直感的なレイアウトで対局を楽しみたい。
- 手番ごとにAIの打牌推奨理由や要因分解、オーラス逆転条件をリアルタイムに確認しながら打ちたい。
- 何切るドリルを隙間時間でテンポよく解き、牌効率と点況判断の理解度を高めたい。
- 局が終了した後に「どの巡目でどんな悪手を打ったか」を盤面再現とともに振り返りたい。

### 3.2 前提条件・制約
- **動作環境**: モダンブラウザ（Chrome, Edge, Firefox, Safari等）。
- **言語・ツール**: バックエンドは Rust（既存の `mahjong_core` / `mahjong_engine` を活用）、フロントエンドは React / TypeScript / Vite。
- **通信レイテンシ**: ローカル実行のため、AI計算および画面反映の遅延は 100ms 以内とする。

---

## 4. 機能要件

### 4.1 バックエンド通信・セッション基盤

#### REQ-WEB-001: 対局セッションのライフサイクル管理
**説明:**  
The system shall provide WebSocket and REST endpoints to initialize, advance, and terminate 4-player mahjong match sessions against CPUs.

**受入条件:**
- [ ] クライアントから対局開始リクエストを受信した際、新規セッションIDを発行し初期局状態（配牌・点棒・親番）を返却する。
- [ ] プレイヤーの打牌・副露アクションを受信した際、局状態を進めて全家の反応（CPUの打牌・副露含む）をクライアントへ逐次配信する。

**優先度:** Must  
**備考:** `axum` の WebSocket ハンドラーにより実装。

---

#### REQ-WEB-002: AIアシスト情報のリアルタイム配信
**説明:**  
The system shall calculate and broadcast placement EV, raw EV, factor decomposition, candidate ranking, and natural language explanations for the active human player on every turn.

**受入条件:**
- [ ] プレイヤーの手番が回ってきた際、打牌候補ランキング（上位3手以上）のEV、受入枚数、向聴数を即座に送信する。
- [ ] 点況データ（持ち点、本場、供託）に基づくオーラス逆転条件および着順予測確率を含める。

**優先度:** Must  
**備考:** `PlacementEvaluation`, `Explainer`, `MatchContext` と直接連携。

---

#### REQ-WEB-003: 何切るドリルAPI
**説明:**  
The system shall expose REST endpoints to fetch drill questions and verify player answers with comprehensive factor explanations.

**受入条件:**
- [ ] `GET /api/drill/question` でランダムまたは指定IDの何切る問題（手牌、ツモ、ドラ、点況）を取得できる。
- [ ] `POST /api/drill/answer` で選択打牌を送信すると、正誤、最善打牌、EV差、要因解説が返却される。

**優先度:** Must  
**備考:** `DrillEngine` を直接呼び出す。

---

### 4.2 フロントエンド UI / 卓ビュー

#### REQ-WEB-004: 卓景・牌姿のグラフィカル描画
**説明:**  
The system shall render the 4-player mahjong table including player hands, opponent concealed/exposed hands, discards (kawa), doras, point sticks, and wind indicators.

**受入条件:**
- [ ] 自手牌およびツモ牌が牌画像またはSVG/Unicode等で見やすく整列表示される。
- [ ] 副露面子（チー、ポン、暗槓、明槓）が正しく晒されて右端または所定位置に描画される。
- [ ] リーチ宣言牌が横向きに配置され、リーチ棒が供託エリアに表示される。

**優先度:** Must  
**備考:** 画面リサイズに対応するレスポンシブデザインとする。

---

#### REQ-WEB-005: プレイヤーアクションの直感的UI
**説明:**  
The system shall allow the user to select tiles for discard and prompt interactive action buttons when calls (Chii, Pon, Kan, Riichi, Ron, Tsumo, Pass) are legally available.

**受入条件:**
- [ ] 自手牌の牌をクリック／タップすることで打牌が決定・送信される。
- [ ] 他家の打牌に対して鳴き可能な場合、「チー」「ポン」「カン」「ロン」「パス」のモーダル／ボタンがポップアップ表示される。

**優先度:** Must  
**備考:** 操作ミス防止のための打牌確認オプションやホバーハイライトを設ける。

---

### 4.3 AIアシスタント HUD & 悪手診断ビューア

#### REQ-WEB-006: AIアシストHUD表示
**説明:**  
The system shall display an integrated HUD panel showing recommended discards, EV radar, factor cards (Speed, Value, Safety), and contextual explanations.

**受入条件:**
- [ ] 推奨打牌ランキングカードが表示され、クリックでその打牌を選択できる。
- [ ] 点況に応じた日本語解説文（「トップ目のためスピード重視」「オーラス満貫ツモ条件」等）が表示される。

**優先度:** Must  
**備考:** HUDの表示/非表示（ブラインド特訓モード）をワンクリックで切り替え可能にする。

---

#### REQ-WEB-007: 終局後の悪手タイムラインレビュー
**説明:**  
The system shall display a post-match review timeline showing turns where significant EV loss occurred, allowing replay and analysis.

**受入条件:**
- [ ] 対局終了後、プレイヤーの全打牌のうちEV損失が閾値（例: 0.5pt以上）を超えた巡目がリストアップされる。
- [ ] 各悪手項目を選択すると、その時点の盤面・手牌・推奨打牌・要因解説が再現表示される。

**優先度:** Should  
**備考:** `ReviewTracker` の履歴データを活用。

---

## 5. 要件品質チェックリスト

### 5.1 個々の要件の品質（5特性）

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---------|:------:|:------:|:------:|:------:|:------:|
| REQ-WEB-001 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-002 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-003 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-004 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-005 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-006 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-WEB-007 | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.2 要件集合の品質

| チェック項目 | OK | 備考 |
|-------------|:---:|------|
| **完全性:** 必要な機能が漏れなく記述されているか | ✓ | バックエンド通信、卓UI、AI HUD、ドリル、悪手レビューを網羅 |
| **一貫性:** 要件間に矛盾がないか | ✓ | 既存のRustエンジン資産との互換性を確保 |

---

## 6. 曖昧語禁止リスト

要件定義文において、以下の曖昧語は使用せず具体的数値・プロトコルで定義されています。
- best / most $\to$ 「上位3手以上」「EV損失0.5pt以上」
- user friendly / easy to use $\to$ 「クリック／タップで送信」「モーダル／ボタンがポップアップ」
- flex / scalable $\to$ 「レスポンシブデザイン」
- etc. $\to$ 網羅的に列挙（チー、ポン、カン、リーチ、ロン、ツモ、パス）

---

## 7. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | 初版作成（Web/GUI インターフェース要件定義） | Syun & AI Assistant |
