# 要件定義書：UI連携用 Python API & シリアライズ基盤

> ISO/IEC/IEEE 29148 準拠（SRS簡易版）  
> 本要件定義書は、参照専用の `definition/templates/requirements.md` のテンプレートに基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-learning-assistant |
| モジュール名 | ui-integration-api (UI連携用 Python API & シリアライズ基盤) |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-14 |
| 最終更新日 | 2026-09-14 |
| 作成者 | Syun & AI Assistant |
| ステータス | Review |

---

## 1. 目的・スコープ

### 1.1 目的
本プロジェクトは、Rustによる高速な麻雀数理・判定・AI評価エンジンを `PyO3` / `maturin` を介して Python ライブラリ（`rust-mahjong`）として提供しています。  
UI（Web/GUIフロントエンド）は**別リポジトリで独立して開発・保守**される設計方針に基づき、UIリポジトリ側が `pip install rust-mahjong` して容易かつ低遅延に麻雀卓のグラフィカル描画、リアルタイムAI HUD、何切るドリル、終局後の悪手検討を実装できるよう、必要な **Python API、完全な局面データモデル、および JSON/dict シリアライズ基盤** を提供します。

### 1.2 スコープ

**スコープ内:**
- **完全な局面データモデルとシリアライズ (`to_dict()` / JSON)**:
  - 4人の手牌・ツモ牌、副露面子（チー・ポン・カン）、河（捨て牌・リーチ宣言牌）、山牌残数、ドラ表示牌、点棒、局・本場・供託棒、親番、風（場風・自風）。
  - UIフロントエンド（React/Vue/Next.js等）へそのまま渡せる辞書（`dict`）または JSON 文字列形式の出力。
- **順位期待値（Placement EV）およびオーラス逆転条件の Python バインディング**:
  - `MatchContext`（点況・ルール・局進行）、`PlacementEvaluation`（素点EV、順位EV、1位〜4位確率分布、オーラス逆転条件）の Python クラス化および `to_dict()`。
- **リアルタイム AI アシスト情報の統合取得 API**:
  - 現在の手番における推奨打牌ランキング（上位候補一覧、素点EV、順位EV、向聴数、受入枚数、要因分解、日本語解説文）を1回の呼び出しで取得できる Python API。
- **対局進行・イベント駆動 API**:
  - プレイヤーの打牌・リーチ・副露（チー・ポン・カン・ロン・パス）を反映し、局面を次の状態へ進めるゲームマネージャーの Python API。
- **何切るドリル & 悪手レビューの Python API 強化**:
  - 問題出題・回答判定・要因解説の dict 変換、および `ReviewTracker` のタイムラインデータの dict 変換。

**スコープ外:**
- HTML/CSS/React コンポーネントそのものの実装（別UIリポジトリの管轄）。
- HTTP / WebSocket サーバーの実装（UIリポジトリ側のバックエンドまたは専用サーバーの管轄）。

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| プロジェクトオーナー | ユーザー | 高 |
| 設計・実装 | AI Assistant | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| `rust-mahjong` | 本リポジトリからビルドされる Python パッケージ名。 |
| UIリポジトリ | 本ライブラリをインポートしてGUI画面・Webアプリを構築する別リポジトリ。 |
| `to_dict()` | 各種ゲームオブジェクト・AI評価結果を Python の標準辞書形式へシリアライズするメソッド。 |
| Placement EV | 順位点（ウマ・オカ）を考慮した順位Pt期待値。 |
| 卓状態 (TableState) | UI側が麻雀卓全体を描画するために必要な4家の情報・ドラ・点棒の完全なスナップショット。 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ
- UIリポジトリ側で `import mahjong` するだけで、複雑な麻雀ルールやAI評価を意識せず、即座にUI描画用のデータを取得したい。
- フロントエンドとの通信（REST/WebSocket）で使えるように、全データが直感的な JSON/dict 形式で取得できるようにしたい。
- 順位期待値やオーラス逆転条件も Python からワンストップで利用できるようにしたい。

### 3.2 前提条件・制約
- **Python環境**: Python >= 3.10 対応。
- **型ヒント**: UI開発側が補完を効かせられるよう、`_core.pyi` に完全な型定義を提供する。
- **パフォーマンス**: シリアライズ処理がPythonの対局ループのボトルネックにならないよう、Rust側で高速にデータ生成を行う。

---

## 4. 機能要件

### 4.1 局面データモデル & シリアライズ

#### REQ-UI-001: 卓状態スナップショットのシリアライズ
**説明:**  
The system shall provide a `TableState` (or `GameState`) Python class with a `to_dict()` method returning complete board state including hands, calls, discards, scores, winds, doras, and round counters.

**受入条件:**
- [ ] `state.to_dict()` が Python 標準の `dict`（キー名: `round_wind`, `dealer`, `honba`, `riichi_sticks`, `dora_indicators`, `players` 等）を返却する。
- [ ] 各プレイヤーの `hand`（非公開/公開フラグ対応）、`melds`、`discards`、`score`、`seat_wind` が含まれる。

**優先度:** Must  
**備考:** UI側が JSON 変換してフロントエンドへそのまま渡せる構造とする。

---

### 4.2 順位期待値・AIアシスト統合 API

#### REQ-UI-002: 順位期待値（Placement EV）Python バインディング
**説明:**  
The system shall expose `MatchContext` and `PlacementEvaluation` to Python, allowing calculation of placement probabilities, ranking EV, and final-round win conditions.

**受入条件:**
- [ ] Pythonから持ち点、ウマ・オカ設定、局（東1局〜南4局）を指定して `MatchContext` を作成できる。
- [ ] `evaluate_placement_discards(hand, context, ...)` を呼び出すと、各打牌の素点EV、順位EV、着順確率分布（1位率〜4位率）、解説文を含むオブジェクト（`to_dict()` 対応）が返却される。
- [ ] オーラス時における各プレイヤーへの逆転条件（満貫ツモ、ハネマン直撃等）が取得できる。

**優先度:** Must  

---

#### REQ-UI-003: AI HUD 用推奨打牌・要因分解の一括取得
**説明:**  
The system shall provide a unified function `get_ai_hud_data(hand, context, ...)` that returns ranked discard recommendations, factor breakdowns (Speed, Value, Safety), and placement context in a single dictionary.

**受入条件:**
- [ ] 単一の呼び出しで、上位推奨打牌ランキング、各候補の要因分解、日本語解説文、現在順位EVが辞書形式で取得できる。

**優先度:** Must  

---

### 4.3 何切るドリル & レビュー機能の dict 変換

#### REQ-UI-004: 何切るドリル問題と判定の辞書化
**説明:**  
The system shall provide `.to_dict()` on `DrillProblem` and `CallAdvice` for frictionless API response building in UI backends.

**受入条件:**
- [ ] `DrillProblem.to_dict()` が問題の牌姿、ドラ、巡目、最善手、要因解説の辞書を返却する。
- [ ] `CallAdvice.to_dict()` が副露判断の各選択肢と解説を返却する。

**優先度:** Must  

---

#### REQ-UI-005: 悪手レビューログのシリアライズ
**説明:**  
The system shall allow `ReviewTracker` to export all turn decisions, EV losses, and blunders as a list of dictionaries suitable for timeline UI replay.

**受入条件:**
- [ ] `review_tracker.get_blunders_dict()` で EV 損失が一定以上の局面一覧を辞書配列で取得できる。

**優先度:** Should  

---

## 5. 要件品質チェックリスト

### 5.1 個々の要件の品質（5特性）

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---------|:------:|:------:|:------:|:------:|:------:|
| REQ-UI-001 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-UI-002 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-UI-003 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-UI-004 | ✓ | ✓ | ✓ | ✓ | ✓ |
| REQ-UI-005 | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.2 要件集合の品質

| チェック項目 | OK | 備考 |
|-------------|:---:|------|
| **完全性:** 必要な機能が漏れなく記述されているか | ✓ | 卓状態、順位EV、AI HUD、ドリル、悪手レビューの連携を網羅 |
| **一貫性:** 要件間に矛盾がないか | ✓ | Pythonライブラリとしての独立性とUIリポジトリ側の利便性を両立 |

---

## 6. 曖昧語禁止リスト

曖昧語（best, easy, scalable, etc.）を排除し、具体的なクラス名、メソッド名、返却データ構造で定義されています。

---

## 7. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | UIリポジトリ分離方針（パターン1）に基づき、UI連携用Python API・シリアライズ要件として初版作成 | Syun & AI Assistant |
