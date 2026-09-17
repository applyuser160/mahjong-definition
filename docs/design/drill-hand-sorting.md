# 設計書：何切るドリル手牌の理牌（ソート）機能

> C4モデル（Context / Container / Component）準拠  
> 本設計書は、参照専用の definition/templates/design.md に基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-drill-hand-sorting |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-14 |
| 最終更新日 | 2026-09-14 |
| 作成者 | Syun & AI Assistant |

---

## 1. Level 1: システムコンテキスト図（Context Diagram）

### 1.1 説明
何切るドリルを利用するユーザー（CLIプレイヤーおよびWebブラウザ利用者）に対し、理牌（ソート）された手牌情報を提供します。

### 1.2 コンテキスト図

`mermaid
graph TB
    User["👤 プレイヤー<br/>[Person]<br/>何切るドリルで牌効率を学習"]

    System["🀄 麻雀分析・ドリルシステム<br/>[Software System]<br/>手牌評価・何切る問題生成・理牌表示"]

    User -->|"問題取得・打牌選択"| System
    System -->|"ソート済み手牌・解説返却"| User
`

---

## 2. Level 2: コンテナ図（Container Diagram）

### 2.1 コンテナ図

`mermaid
graph TB
    User["👤 プレイヤー<br/>[Person]"]

    subgraph System ["何切るドリルシステム"]
        WebUI["🌐 フロントエンド<br/>[Container: React / Vite]<br/>DrillView.tsx で手牌を視覚表示"]
        Backend["⚙️ APIサーバー<br/>[Container: FastAPI / Python]<br/>/api/drill/problem エンドポイント"]
        CoreLib["🦀 麻雀コアライブラリ<br/>[Container: Rust / PyO3]<br/>Hand::sort / DrillEngine"]
        CLI["💻 CLIドリル<br/>[Container: Rust Example]<br/>play_drill.rs"]
    end

    User -->|"Webブラウザ操作"| WebUI
    User -->|"ターミナル入力"| CLI
    WebUI -->|"HTTP GET /api/drill/problem"| Backend
    Backend -->|"PyO3呼び出し"| CoreLib
    CLI -->|"Rust直接呼び出し"| CoreLib
`

### 2.2 コンテナ一覧

| コンテナ名 | 技術スタック | 責務 | 通信プロトコル |
|-----------|------------|------|--------------|
| フロントエンド | React / TypeScript / Vite | 手牌の視覚的描画・選択UI | HTTP / REST |
| APIサーバー | Python / FastAPI | ドリル問題の配信・回答判定 | PyO3 / In-process |
| 麻雀コア | Rust | 高速牌効率計算・手牌生成・ソート | In-process |
| CLIドリル | Rust | ターミナルでの学習ドリル実行 | In-process |

---

## 3. Level 3: コンポーネント図（Component Diagram）

### 3.1 麻雀コアおよびドリルエンジンのコンポーネント図

`mermaid
graph TB
    subgraph DrillModule ["何切るドリルモジュール (drill.rs)"]
        DrillEngine["⚙️ DrillEngine<br/>[Component]<br/>generate_problem()"]
    end

    subgraph HandModule ["手牌モジュール (hand.rs)"]
        Hand["🃏 Hand<br/>[Component]<br/>tiles: [TileName; 14]<br/>sort()"]
    end

    subgraph TileModule ["牌種モジュール (tile.rs)"]
        TileName["🀄 TileName<br/>[Component]<br/>Ord / PartialOrd 実装"]
    end

    subgraph PythonApiModule ["Python APIモジュール (python_api.rs)"]
        PyDrill["🐍 PyDrillProblem<br/>[Component]<br/>to_dict() -> mpsz[]"]
    end

    DrillEngine -->|"1. 14枚の手牌生成"| Hand
    DrillEngine -->|"2. hand.sort() 実行"| Hand
    Hand -->|"要素比較 (Ord)"| TileName
    PyDrill -->|"ソート済み hand.tiles() を変換"| Hand
`

### 3.2 詳細設計

#### Hand::sort
Hand 構造体に以下のメソッドを追加します。
`ust
impl Hand {
    /// 手牌（有効な len 枚）を TileName の昇順（萬子・筒子・索子・字牌）にソートします。
    pub fn sort(&mut self) {
        self.tiles[..self.len].sort_unstable();
    }
}
`
- self.counts は各牌種の所持枚数（35要素配列）であり、手牌内の並び順が変化しても値は不変です。
- 	iles[..len] の範囲のみを対象とするため、未使用のスロットは影響を受けません。
- sort_unstable() はアロケーションを行わず最速で動作します。

#### DrillEngine::generate_problem
手牌生成後、打牌候補の評価（valuate_hand_discards）の前または直後に hand.sort() を適用します。
valuate_hand_discards は手牌配列を巡回して打牌を試行するため、hand がソートされていても全く同一の評価結果が得られます。
ドリル問題として返却される DrillProblem.hand は常にソートされた状態となります。

#### Python API
py_generate_drill_problem 内で problem.hand.tiles() をイテレートして 	iles および 	o_dict()["mpsz"] を生成するため、Rust側でソートされていれば自動的にソート済みのリストが返却されます。

---

## 4. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | 初版作成 | Syun & AI Assistant |
