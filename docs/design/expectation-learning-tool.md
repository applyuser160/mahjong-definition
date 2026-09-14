# 基本・詳細設計書：リアルタイム期待値ヒント＆意思決定モデル学習ツール

> C4モデル（Context / Container / Component）準拠  
> 本設計書は、参照専用の `definition/templates/design.md` に基づき作成されています。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong-learning-assistant |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-13 |
| 最終更新日 | 2026-09-13 |
| 作成者 | Syun & AI Assistant |

---

## 1. Level 1: システムコンテキスト図（Context Diagram）

### 1.1 説明
プレイヤーが麻雀の対局（一人麻雀・練習対戦）をプレイする中で、プレイヤーの手番（ツモ直後）にリアルタイムで打牌期待値（EV）ランキングおよび要因分解モデル（速度・打点・守備）を提示する学習システムです。

### 1.2 コンテキスト図

```mermaid
graph TB
    Player["👤 プレイヤー<br/>[Person]<br/>麻雀をプレイし期待値判断を学習するユーザー"]

    System["🀄 麻雀学習支援システム<br/>[Software System: mahjong]<br/>局進行・期待値計算・要因分析ヒントHUDを提供"]

    Player -->|"1. ツモ/打牌入力"| System
    System -->|"2. リアルタイム期待値ヒント & 要因分析"| Player
    System -->|"3. 局面状態・局結果のフィードバック"| Player
```

---

## 2. Level 2: コンテナ図（Container Diagram）

### 2.1 コンテナ図

```mermaid
graph TB
    Player["👤 プレイヤー<br/>[Person]"]

    subgraph System ["麻雀学習支援システム (mahjong)"]
        CLI["💻 対局 CLI & 学習 HUD<br/>[Container: Rust CLI / Python Runner]<br/>対局進行・ヒントのリアルタイム描画・ユーザー入力"]
        PyBinding["🐍 Python Binding<br/>[Container: PyO3 / maturin]<br/>RustコアとPythonスクリプトの連携インターフェース"]
        RustCore["🦀 麻雀コア & 期待値計算エンジン<br/>[Container: Rust Library (mahjong)]<br/>局進行・向聴数・受け入れ・符計算・期待値要因モデル"]
    end

    Player -->|"打牌選択 / コマンド入力"| CLI
    CLI -->|"手番ごとのヒント要求"| RustCore
    CLI -.->|"スクリプト実行時"| PyBinding
    PyBinding -->|"ネイティブAPI呼出"| RustCore
    RustCore -->|"EVスコア・要因内訳・解説文"| CLI
    CLI -->|"ANSIカラー表示 / 牌姿HUD"| Player
```

### 2.2 コンテナ一覧

| コンテナ名 | 技術スタック | 責務 | 通信プロトコル |
|-----------|------------|------|--------------|
| **Rust Core** | Rust 2021 (cdylib/rlib) | 牌・手牌・局進行・向聴数・符計算・期待値・要因分析の超高速計算 | インメモリ関数呼出 |
| **Python Binding** | PyO3, maturin | Python 側から Rust 計算エンジンを呼び出すためのバインディング層 | C-FFI / Python C-API |
| **対局 CLI & HUD** | Rust / Python | 局面のターミナル表示、手番時ヒントHUD、打牌受付 | 標準入出力 (stdin/stdout) |

---

## 3. Level 3: コンポーネント図（Component Diagram: Rust Core）

### 3.1 コンポーネント図

```mermaid
graph TB
    subgraph RustCore ["Rust Core Engine (mahjong)"]
        GameLoop["🔄 Round / GameLoop<br/>局進行・配牌・ツモ・打牌制御"]
        
        ShantenEngine["📐 Shanten Engine<br/>向聴数計算 (一般手/七対子/国士)"]
        AcceptanceEngine["🎯 Acceptance Engine<br/>有効牌・残り枚数・待ち形判定"]
        
        ScoreEngine["💰 Score Engine<br/>符計算 (20〜110符)・翻符得点算出"]
        DoraEngine["🌟 Dora Engine<br/>表ドラ・裏ドラ・赤ドラ加算"]
        YakuEngine["🀄 Yaku Engine<br/>役判定 (1翻〜役満)"]
        
        EVEngine["🧠 Expectation Engine<br/>打牌候補別 EV スコアリング"]
        ExplanationModel["📊 Explanation Model<br/>速度・打点・リスク分解 & 決定理由文生成"]
    end

    GameLoop -->|"手番時の手牌(14枚)"| EVEngine
    EVEngine -->|"各打牌後の向聴数"| ShantenEngine
    EVEngine -->|"有効牌種・残り枚数"| AcceptanceEngine
    EVEngine -->|"和了想定役"| YakuEngine
    EVEngine -->|"和了想定ドラ"| DoraEngine
    EVEngine -->|"和了想定得点"| ScoreEngine
    EVEngine -->|"計算結果データ"| ExplanationModel
    ExplanationModel -->|"ヒントHUDデータ"| GameLoop
```

### 3.2 主要コンポーネント詳細

| コンポーネント | ソースファイル | 主な責務とアルゴリズム |
| :--- | :--- | :--- |
| **向聴数エンジン** | `src/mahjong/shanten.rs` | 面子手（バックトラック/深さ優先探索）、七対子（対子数と種類）、国士無双（19字牌の種類と対子）の最小向聴数を計算。 |
| **有効牌エンジン** | `src/mahjong/acceptance.rs` | 手牌から各牌を切った後、全34種の牌を1枚加えたときの向聴数を走査。向聴数が下がる牌を「有効牌」としてリストアップし、見えている牌を引いて残り枚数を集計。 |
| **符・得点エンジン** | `src/mahjong/score.rs` | 和了時の手牌構成（面子の明暗・公九、雀頭役牌、待ち形符）から符を10符単位で切り上げ計算。翻数と組み合わせて子・親のツモ/ロン得点を算出。 |
| **ドラエンジン** | `src/mahjong/dora.rs` | ドラ表示牌の次の牌（ネクスト牌）判定、赤ドラ判定（赤5m/5p/5s）と翻数合算。 |
| **期待値エンジン** | `src/mahjong/expectation.rs` | 各打牌候補の $EV = P(\text{和了}) \times \text{平均和了打点} - P(\text{放銃}) \times \text{平均放銃失点}$ を算出。 |
| **要因分解モデル** | `src/mahjong/explanation.rs` | EV上位候補同士の速度（枚数差）・打点（満貫変化等）・安全度を比較し、「なぜ1位が最善か」の日本語解説テキストを生成。 |

---

## 4. データ構造・モデル設計

### 4.1 期待値ヒント＆要因モデル構造体

```rust
/// 各打牌候補の評価結果
pub struct DiscardCandidateEvaluation {
    pub discard_tile: TileName,        // 切る牌
    pub shanten_after: i8,             // 打牌後の向聴数 (0: テンパイ, 1: 一向聴...)
    pub ev: f64,                       // 総合期待値スコア
    
    // 要因分解モデル (Explanatory Breakdown)
    pub speed: SpeedMetric,            // 速度指標
    pub value: ValueMetric,            // 打点指標
    pub safety: SafetyMetric,          // 安全度指標
}

/// 速度指標
pub struct SpeedMetric {
    pub accepted_tiles: Vec<TileName>, // 受け入れ牌一覧
    pub remaining_count: usize,        // 残り受け入れ合計枚数
    pub win_probability: f64,          // 想定和了確率 (0.0 ~ 1.0)
}

/// 打点指標
pub struct ValueMetric {
    pub expected_score: f64,           // 平均和了打点
    pub expected_han: f64,             // 平均翻数
    pub primary_yaku: Vec<&'static str>,// 想定される主要役 (タンヤオ, 平和, 三色等)
    pub has_high_value_potential: bool,// 打点アップ変化があるか
}

/// 安全度指標
pub struct SafetyMetric {
    pub risk_score: f64,               // 危険度スコア (0.0: 現物安全 ~ 1.0: 超危険)
    pub is_genbutsu: bool,             // 他家への現物か
}

/// ヒントHUD全体出力
pub struct TurnHint {
    pub candidates: Vec<DiscardCandidateEvaluation>, // 期待値順ランキング
    pub best_discard: TileName,                      // 推奨打牌
    pub rationale: String,                           // なぜその打牌なのかの解説文
}

/// 毎巡の手番打牌決定記録
pub struct TurnDecisionRecord {
    pub turn: usize,                                 // 巡目
    pub chosen_tile: TileName,                       // ユーザーが選択した打牌
    pub chosen_ev: f64,                              // 選択打牌のEV
    pub best_tile: TileName,                         // AI推奨最善打牌
    pub best_ev: f64,                                // 最善打牌のEV
    pub ev_loss: f64,                                // EV損失 (best_ev - chosen_ev)
    pub candidates: Vec<DiscardCandidateEvaluation>, // 当該巡の全候補評価
}

/// 悪手・疑問手記録
pub struct BlunderRecord {
    pub turn: usize,
    pub chosen_tile: TileName,
    pub best_tile: TileName,
    pub ev_loss: f64,
    pub severity: BlunderSeverity,                   // Inaccuracy (疑問手) / Mistake (悪手) / Blunder (大悪手)
    pub explanation: String,                         // なぜ悪手なのかの解説
}

pub enum BlunderSeverity {
    Inaccuracy, // 損失 150〜400点
    Mistake,    // 損失 400〜1000点
    Blunder,    // 損失 1000点以上
}

/// 対局全体の学習振り返りレポート
pub struct MatchReviewReport {
    pub total_turns: usize,                          // 総打牌数
    pub optimal_picks_count: usize,                  // AI最善手と一致した回数
    pub accuracy_rate: f64,                          // 打牌精度 (一致率 0.0 ~ 1.0)
    pub total_ev_loss: f64,                          // 累計EV損失
    pub blunders: Vec<BlunderRecord>,                // 悪手一覧（EV損失順）
}

/// 何切るドリル問題
pub struct DrillProblem {
    pub hand_tiles: Vec<TileName>,                   // 14枚の手牌
    pub dora_indicator: TileName,                    // ドラ表示牌
    pub turn_number: usize,                          // 想定巡目
    pub candidates: Vec<DiscardCandidateEvaluation>, // 打牌候補別EV評価
    pub best_tile: TileName,                         // 最善打牌
    pub rationale: String,                           // なぜ最善かの要因解説
}

/// 何切るセッション成績
pub struct DrillSessionReport {
    pub total_problems: usize,                       // 総出題数
    pub correct_count: usize,                        // 正解数（最善手選択）
    pub accuracy_rate: f64,                          // 正答率
    pub total_ev_loss: f64,                          // 累計EV損失
    pub average_ev_loss: f64,                        // 1問平均EV損失
}

/// 副露（鳴き）判断推奨
pub struct CallAdvice {
    pub target_tile: TileName,                       // 鳴きの対象牌（他家打牌）
    pub choices: Vec<CallChoice>,                    // 選択肢一覧（チー/ポン/スルー）
    pub best_action: CallAction,                     // 推奨アクション
    pub rationale: String,                           // なぜその判断かの解説
}

pub struct CallChoice {
    pub action: CallAction,
    pub post_shanten: i8,                            // 鳴き後の向聴数
    pub post_acceptance: usize,                      // 鳴き後の有効牌枚数
    pub estimated_score: f64,                        // 鳴き後の想定打点
    pub ev: f64,                                     // アクション総合EV
}

pub enum CallAction {
    Pass,                                            // スルー
    Chii,                                            // チー
    Pon,                                             // ポン
    Kan,                                             // カン
}
```

---

## 5. 主要シーケンス図

### 5.1 手番時のヒント生成フロー
```mermaid
sequenceDiagram
    autonumber
    actor Player as プレイヤー
    participant GL as GameLoop / CLI
    participant EV as ExpectationEngine
    participant Shanten as Shanten / Acceptance
    participant Score as Score / Yaku
    participant Exp as ExplanationModel

    GL->>GL: ツモ牌を手牌に追加 (14枚)
    GL->>EV: analyze_turn(hand, visible_tiles, round_ctx)
    
    loop 手牌のユニークな打牌候補ごと
        EV->>Shanten: calculate_acceptance(hand_without_tile)
        Shanten-->>EV: 有効牌リスト & 残り枚数
        EV->>Score: estimate_expected_value(hand, accepted_tiles, dora)
        Score-->>EV: 想定平均打点 & 主要役
        EV->>EV: EV = P(win) * Score - Risk
    end

    EV->>Exp: generate_rationale(ranked_candidates)
    Exp-->>EV: 要因比較解説テキスト (Rationale)
    EV-->>GL: TurnHint (ランキング + 要因内訳 + 解説)

    GL->>Player: 💡 ヒントHUD描画 (EV, 受入, 想定打点, なぜそうなのか)
    Player->>GL: 打牌選択入力 (例: "9p")
    GL->>GL: 打牌実行・河へ追加・ターン交代
```

### 5.2 局終了時の学習振り返りレポート生成フロー
```mermaid
sequenceDiagram
    autonumber
    actor Player as プレイヤー
    participant GL as GameLoop / CLI
    participant RT as ReviewTracker
    participant Exp as ExplanationModel

    Note over GL,RT: 対局中、毎巡の打牌を記録
    GL->>RT: record_decision(turn, chosen_tile, candidates)
    RT->>RT: EV損失 = best_ev - chosen_ev を算出・蓄積

    Note over GL,RT: 和了または流局（終局時）
    GL->>RT: generate_report()
    RT->>Exp: 悪手打牌に対する要因解説を要求
    Exp-->>RT: 「受入低下」「打点放棄」「危険度無視」の理由文
    RT-->>GL: MatchReviewReport (総損失、一致率、悪手ランキング)
    GL->>Player: 📊 終局振り返りHUD表示 (悪手診断・学習アドバイス)
```

### 5.3 何切るドリル問題出題・採点フロー
```mermaid
sequenceDiagram
    autonumber
    actor Learner as 学習者
    participant Drill as DrillEngine
    participant EV as ExpectationEngine
    participant Exp as ExplanationModel

    Drill->>EV: generate_problem(shanten_filter)
    EV-->>Drill: DrillProblem (牌姿14枚 + 候補ランキング + 最善手 + 解説)
    Drill->>Learner: 🀄 問題提示 (手牌・ドラ・巡目)
    Learner->>Drill: 回答入力 (例: "3m")
    Drill->>Drill: 正誤判定 & EV損失算出
    Drill->>Learner: 💡 即時フィードバック (正誤・EV差・要因解説)
```

---

## 6. 実装タスク計画（マイルストーン）

```mermaid
gantt
    title 実装マイルストーン
    dateFormat  YYYY-MM-DD
    section フェーズ1: 計算基盤 & EVモデル (完了)
    向聴数・受け入れエンジン実装 (shanten/acceptance) :done, 2026-09-14, 2d
    符計算・得点・ドラエンジン実装 (score/dora)      :done, 2026-09-16, 2d
    期待値スコアリングエンジン (expectation)        :done, 2026-09-18, 2d
    要因分解 & 自然言語解説生成 (explanation)       :done, 2026-09-20, 2d
    section フェーズ2: 振り返り診断 & 4人対局 (完了)
    振り返り診断エンジン (review.rs)                :done, 2026-09-21, 2d
    一人麻雀CLIへの振り返りレポート統合             :done, 2026-09-22, 1d
    4人CPU対局ループ & 守備・押し引き学習 (play_with_cpu) :done, 2026-09-23, 2d
    section フェーズ3: 何切るドリル & 鳴き判断 (現在)
    何切るドリルエンジン & CLI (drill.rs / play_drill) :active, 2026-09-24, 2d
    副露（鳴き）判断アドバイザー (call_advisor.rs)  :active, 2026-09-25, 2d
    Pythonバインディング拡充 & 統合テスト           :2026-09-26, 1d
```


