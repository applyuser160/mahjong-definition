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
    ├── requirements/          # 要件定義書
    │   └── expectation-learning-tool.md
    ├── design/                # 基本・詳細設計書
    ├── adr/                   # 意思決定記録（ADR）
    └── checklist/             # 非機能要件チェックリスト
```

---

## 運用ルール

1. **ドキュメント先行開発**:
   - コード実装に着手する前に、必ず本リポジトリ配下に要件定義書（`docs/requirements/`）および設計書（`docs/design/`）を作成・合意すること。
2. **参照専用リポジトリの保護**:
   - テンプレートのマスターである `definition/` (`d:\Desktop\definition`) は**完全参照専用（Read-Only）**であり、一切の変更を行わないこと。
3. **コードベースの分離**:
   - 実装コードは `mahjong/` (`d:\Desktop\mahjong`) で行い、仕様・設計・ADRなどのナレッジは本リポジトリ（`mahjong-definition/`）で一元管理すること。
