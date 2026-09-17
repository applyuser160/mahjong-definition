# 設計書: rayon による打牌候補評価・鳴き判断・着順EV・牌譜レビューの並列化

> C4モデル（Component Diagram）およびデータフローに準拠した設計書です。  
> Mermaid 記法を用いて並列実行フローとデータ依存性を可視化します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 設計対象 | parallel-evaluation-rayon |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-15 |
| 最終更新日 | 2026-09-15 |
| 作成者 | Antigravity AI |

---

## 1. システム構成・コンポーネント図（C4 Level 3）

```mermaid
graph TB
    subgraph Engine ["mahjong Core Engine (Rust)"]
        subgraph Expectation ["expectation.rs"]
            EHD["evaluate_hand_discards()<br/>(打牌候補並列評価)"]
            ShantenAcc["calculate_acceptance()"]
            SpeedEst["estimate_win_probability()"]
            ValueEst["estimate_hand_value()"]
            SafetyEst["evaluate_tile_safety()"]
        end

        subgraph CallAdvisor ["call_advisor.rs"]
            AC["advise_call()<br/>(鳴き候補並列評価)"]
            StandingEval["evaluate_standing_hand()"]
        end

        subgraph PlacementEV ["placement_ev.rs"]
            EHDW["evaluate_hand_discards_with_placement()<br/>(順位シミュレーション並列化)"]
            RankProb["estimate_rank_probabilities()"]
        end

        subgraph Review ["review.rs"]
            RT["ReviewTracker::generate_report()<br/>(牌譜レポート集計並列化)"]
            Diagnose["diagnose_loss_reason()"]
        end

        RayonPool["⚡ Rayon Global ThreadPool<br/>(Work-Stealing Scheduler)"]
    end

    EHD -.->|"par_iter()"| RayonPool
    AC -.->|"par_iter()"| RayonPool
    EHDW -.->|"par_iter()"| RayonPool
    RT -.->|"par_iter()"| RayonPool

    RayonPool --> ShantenAcc
    RayonPool --> SpeedEst
    RayonPool --> ValueEst
    RayonPool --> SafetyEst
    RayonPool --> StandingEval
    RayonPool --> RankProb
    RayonPool --> Diagnose
```

---

## 2. 各対象機能の並列処理アーキテクチャ

### 2.1 打牌候補評価 (`expectation.rs`)

手牌（14枚）に存在する牌種（最大14種）について、打牌候補ごとに独立した手牌カウンタ `working` のコピーを作成し、並列に評価します。

```mermaid
sequenceDiagram
    autonumber
    participant Caller as 呼出元
    participant EHD as evaluate_hand_discards
    participant Pool as Rayon Work-Stealing Pool
    participant Worker1 as Worker Thread A
    participant Worker2 as Worker Thread B

    Caller->>EHD: hand, visible_counts, ctx
    EHD->>Pool: (1..=34).into_par_iter().filter(count > 0)
    par 候補1の評価
        Pool->>Worker1: 候補 tile A 評価 (向聴数・打点・安全度)
    and 候補2の評価
        Pool->>Worker2: 候補 tile B 評価 (向聴数・打点・安全度)
    end
    Worker1-->>Pool: CandidateEvaluation A
    Worker2-->>Pool: CandidateEvaluation B
    Pool-->>EHD: Vec<CandidateEvaluation>
    EHD->>EHD: 決定論的ソート (EV降順 -> 向聴数昇順 -> 受入降順 -> 牌ID昇順)
    EHD-->>Caller: 評価結果リスト
```

#### スレッドセーフティとデータ独立性
- `hand.counts` は `[u8; 35]`（35バイトのスタック値）。各ワーカーが手牌カウンタをローカルコピーして1枚減算するため、ミュータブルな共有状態は存在しません。
- `base_visible` および `ctx` (`AnalysisContext`) は不変参照（`&base_visible`, `&AnalysisContext`）としてワーカー間で共有されます。これらはすべて `Sync` を満たします。

---

### 2.2 鳴き判断 (`call_advisor.rs`)

`advise_call()` では、スルー（門前維持）、ポン、チー（最大3パターン）の評価タスクを列挙し、`into_par_iter()` で並列評価します。

```mermaid
graph TD
    AC["advise_call()"] --> Tasks["CallTask リスト生成"]
    Tasks --> T1["Task: Pass (スルー)"]
    Tasks --> T2["Task: Pon (ポン)"]
    Tasks --> T3["Task: Chii A (チーパターンA)"]
    Tasks --> T4["Task: Chii B (チーパターンB)"]
    Tasks --> T5["Task: Chii C (チーパターンC)"]

    T1 -.->|"par_iter()"| W1["Worker 1: evaluate_standing_hand()"]
    T2 -.->|"par_iter()"| W2["Worker 2: evaluate_hand_discards(post_hand)"]
    T3 -.->|"par_iter()"| W3["Worker 3: eval_chii_choice()"]
    T4 -.->|"par_iter()"| W4["Worker 4: eval_chii_choice()"]
    T5 -.->|"par_iter()"| W1

    W1 --> Result["CallChoice リストの集約"]
    W2 --> Result
    W3 --> Result
    W4 --> Result
    Result --> Best["最善鳴きアクション選定・アドバイス生成"]
```

---

### 2.3 着順期待値評価 (`placement_ev.rs`)

素点評価済みの打牌候補リスト `raw_evaluations`（最大14件）に対し、各候補の「和了・放銃・流局シナリオ着順確率分布」のシミュレーション計算を並列化します。

```rust
let placement_evals: Vec<PlacementCandidateEvaluation> = raw_evaluations
    .into_par_iter()
    .map(|ev| {
        // 各シナリオの局後スコア・着順確率シミュレーション
        let probs_win = estimate_rank_probabilities(match_ctx, player_idx, score_on_win);
        let probs_deal = estimate_rank_probabilities(match_ctx, player_idx, score_on_deal);
        let probs_other = estimate_rank_probabilities(match_ctx, player_idx, score_on_other);
        ...
    })
    .collect();
```

---

### 2.4 牌譜レビュー処理 (`review.rs`)

`ReviewTracker::generate_report()` において、蓄積された `records: Vec<TurnDecisionRecord>` の集計と悪手診断文生成（`diagnose_loss_reason`）を並列化します。
Rayon の `par_iter()` を用いて、各レコードの `is_optimal`、`ev_loss` の加算、および `blunder` の判定・理由診断を並列に実行します。

```rust
let (optimal_picks_count, total_ev_loss, mut blunders) = self.records
    .par_iter()
    .fold(
        || (0usize, 0.0f64, Vec::new()),
        |(mut opt, mut loss, mut blunders), rec| {
            if rec.is_optimal { opt += 1; }
            loss += rec.ev_loss;
            if rec.ev_loss >= 150.0 {
                blunders.push(Self::build_blunder_record(rec));
            }
            (opt, loss, blunders)
        },
    )
    .reduce(
        || (0usize, 0.0f64, Vec::new()),
        |(opt1, loss1, mut blunders1), (opt2, loss2, blunders2)| {
            blunders1.extend(blunders2);
            (opt1 + opt2, loss1 + loss2, blunders1)
        },
    );
```

---

## 3. 型安全性・スレッドセーフティの検証

| 型 | `Send` | `Sync` | 備考 |
|---|:---:|:---:|---|
| `TileName` | ✅ | ✅ | `#[repr(u8)]`, `Copy`, `Clone` |
| `Hand` | ✅ | ✅ | `counts: [u8; 35]`, `open_melds: Vec<Meld>` |
| `AnalysisContext<'_>` | ✅ | ✅ | 不変参照のみを保持 |
| `MatchContext` | ✅ | ✅ | スコア・順位・ルール定義 |
| `CandidateEvaluation` | ✅ | ✅ | EV・向聴数・受入牌 |
| `TurnDecisionRecord` | ✅ | ✅ | 巡目打牌記録 |

---

## 4. Python API と GIL の取り扱い

Python API（`python_api.rs`）における `py_evaluate_hand_discards` などの関数では、すでに `py.allow_threads(...)` が導入されています。
Rayon によるマルチスレッドプール実行時にも Python GIL が解放されているため、GIL競合によるデッドロックや並列性阻害は発生せず、真のCPUマルチコア並列実行が保証されます。

---

## 5. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-15 | 初版作成（Issue #88 に基づく） | Antigravity AI |
