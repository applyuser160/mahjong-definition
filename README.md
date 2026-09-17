# mahjong-definition

麻雀プロジェクト（`mahjong`）における要件定義書・設計書・意思決定記録（ADR）を管理・蓄積するドキュメント専用リポジトリです。  
参照専用の汎用定義テンプレート（`definition/`）の規約（ISO/IEC/IEEE 29148、C4モデル、MADR形式）に準拠しています。

---

## ディレクトリ構成

```
mahjong-definition/
├── README.md                  # 本ドキュメント
├── AGENTS.md / GEMINI.md      # AIエージェント向けガイドライン
├── templates/                 # ドキュメント用テンプレート
│   ├── requirements.md        # 要件定義テンプレート（SRS簡易版）
│   ├── design.md              # 設計書テンプレート（C4モデル準拠）
│   └── adr.md                 # ADRテンプレート（MADR形式）
└── docs/
    ├── requirements/          # 要件定義書 (SRS)
    ├── design/                # 基本・詳細設計書 (Architecture & Design)
    ├── adr/                   # 意思決定記録 (ADR)
    └── checklist/             # 非機能要件チェックリスト
```

---

## 蓄積ドキュメント一覧

### 1. 要件定義書 & 設計書 (`docs/requirements/` & `docs/design/`)
- **向聴数・有効牌・高速化**:
  - `suit-lookup-table-shanten.md`: 色別ルックアップテーブルによる向聴数判定
  - `simd-vectorization-issue-109.md`: AVX2/SWAR ベクトル化によるキー生成
  - `acceptance-pruning-issue-108.md`: 有効牌探索の枝刈り最適化
  - `suit-table-compression-issue-107.md`: スーツテーブル圧縮とパッキング
- **役判定・点数計算**:
  - `yaku-bitmask-yakuset.md`: ビットマスクによる YakuSet 最適化
  - `yaku-zero-allocation-and-hashmap-elimination.md`: 役判定ホットパスのゼロアロケーション
- **期待値探索 & AI評価**:
  - `parallel-evaluation-rayon.md`: Rayon による局収支 EV 並列探索
  - `placement-ev-model.md`: 着順分布モデルと順位点 EV 評価
  - `analysis-engine-enhancements.md`: 統計的・防御的判断モデルの統合
  - `reduce-iishanten-search-and-precompute-safety.md`: 一向聴探索削減と安全度事前計算
- **対局進行 & UI連携**:
  - `cpu-match-hand-sorting.md`: 4人CPU対局における理牌・手牌表示
  - `drill-hand-sorting.md`: 何切るドリル出題時の理牌
  - `ron-call-handling-and-stability.md`: ロン・鳴き判定の安定化
  - `ui-integration-api.md`: UI との Python API 連携仕様
  - `mahjong-ui-application.md`: Web アプリケーション全体要件

### 2. 意思決定記録 (`docs/adr/`)
- `0003-placement-ev-calculation.md`: 順位点 EV 計算モデルの採択
- `0004-ui-separation-and-python-api.md`: UI 分離と PyO3 インターフェース
- `0005-mahjong-ui-tech-stack.md`: Web フロントエンド・バックエンド技術選定
- `0006-match-context-integration.md`: 半荘コンテキストの局進行連携
- `0007-analysis-engine-statistical-and-defensive-model.md`: 統計的防御モデル
- `0008-drill-hand-sorting.md`: ドリル手牌ソート仕様
- `0009-cpu-match-hand-sorting.md`: CPU対局理牌仕様
- `0010-ron-call-handling-and-stability.md`: 鳴き・ロン判定安定化
- `0011-suit-lookup-table-for-shanten.md`: 向聴数 Suit LUT 採択
- `0012-parallel-evaluation-rayon.md`: Rayon マルチスレッド並列化
- `0013-yaku-bitmask-yakuset.md`: YakuSet ビットマスク設計
- `0014-eliminate-hotpath-heap-allocations.md`: ホットパス ヒープ確保排除
- `0014-match-optimization-arithmetic-lookup.md`: 算術テーブル化
- `0014-reduce-iishanten-search-and-precompute-safety.md`: 安全度事前計算
- `0015-yaku-zero-allocation-and-hashmap-elimination.md`: HashMap 排除
- `0016-api-serialization-and-intermediate-pyclass-elimination.md`: API 高速化
- `0017-suit-table-compression-and-entry-packing.md`: エントリパッキング検証
- `0018-acceptance-pruning-and-incremental-shanten.md`: 受入れ枝刈り採択
- `0019-simd-vectorization-for-shanten-and-suit-key.md`: SIMD ベクトル化
- `0020-codebase-refactoring-and-dry-cleanup.md`: リファクタリング方針

---

## 運用ルール

1. **ドキュメント先行開発**:
   - コード実装に着手する前に、必ず本リポジトリ配下に要件定義書（`docs/requirements/`）および設計書（`docs/design/`）を作成・合意すること。
2. **参照専用リポジトリの保護**:
   - テンプレートのマスターである `definition/` (`d:\Desktop\definition`) は**完全参照専用（Read-Only）**であり、一切の変更を行わないこと。
3. **コードベースの分離**:
   - 実装コードは `mahjong/` (`d:\Desktop\mahjong`) で行い、仕様・設計・ADRなどのナレッジは本リポジトリ（`mahjong-definition/`）で一元管理すること。
