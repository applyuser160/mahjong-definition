# 要件定義書: 分析エンジン高度化（和了確率・打点計算・危険度・順位失点モデル）

> ISO/IEC/IEEE 29148 をベースにした簡易版要件定義書です。  
> GitHub Issue #76, #77, #78, #79 を集約し、分析エンジンの高度化要件を定義します。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong (分析エンジン高度化) |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-14 |
| 最終更新日 | 2026-09-14 |
| 作成者 | Antigravity |
| ステータス | Approved |

---

## 1. 目的・スコープ

### 1.1 目的

麻雀分析エンジン（`expectation.rs` / `placement_ev.rs`）における計算モデルは、従来ヒューリスティクス固定係数（一律30符、門前即リーチ前提、牌種別固定危険度、一律8,000点放銃失点）で構成されており、実戦対局データ（天鳳・雀魂統計）や河・点況情報との乖離が存在していた。  
本改修では、これらを待ち形・符計算・河とスジ現物・相手状況依存モデルへと高度化し、AI打牌推奨および順位期待値の判定精度を大幅に向上させる。

### 1.2 スコープ

**スコープ内:**
- 和了確率推定の動的減衰モデル（待ちの好形/愚形、巡目減衰、他家のリーチ・副露脅威度）（Issue #76）
- 和了形に応じた正確な符計算の連動（ピンフ20符、七対子25符、暗刻40〜70符）、門前役あり時のダマテン考慮、一向聴探索の拡張（Issue #77）
- 他家リーチ・河・スジ・カベ・生牌を反映した動的危険度判定および押し引き（ベタオリ判断）モデルの実装（Issue #78）
- 順位期待値（Placement EV）における放銃失点モデルの動的補正（親12,000点 vs 子8,000点、副露打点推定、見えドラ数連動）（Issue #79）
- 分析コンテキスト（`AnalysisContext`）の拡張と後方互換性維持

**スコープ外:**
- 機械学習モデル（ディープラーニング重みファイル）の直接推論エンジンの組み込み（本改修は統計・ルールベースの数理モデル高度化を対象）
- 3人麻雀（サンマ）ルール固有の特例計算

### 1.3 ステークホルダー

| 役割 | 担当者 | 関与度 |
|------|--------|--------|
| オーナー | applyuser160 | 高 |
| 開発者 / AI | Antigravity | 高 |
| 利用者 | 麻雀学習・検討ツール利用者 | 高 |

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| 好形待ち | 両面待ち、多面張など、出和了・ツモの受入種・枚数が多い待ち形 |
| 愚形待ち | 嵌張（カンチャン）、辺張（ペンチャン）、単騎など、受入が1種（最大4枚）に限定される待ち形 |
| 現物（Genbutsu） | 対象プレイヤー（特にリーチ者）が既に捨てている牌、およびリーチ後の他家の捨て牌。振聴規定によりロン和了不可（危険度0%） |
| スジ（Suji） | 両面待ちで対になる牌の関係（例: 4が捨てられている場合の 1 と 7）。表スジ、片スジ、両スジ等がある |
| カベ（Wall） | 4枚見えている牌（No Chance）または3枚見えている牌（One Chance）の外側にある牌の安全度理論 |
| 符（Fu） | 麻雀の得点計算における基本単位。副底20符に加え、門前ロン（10符）、ツモ（2符）、待ち形、面子の種類（暗刻・明刻・槓子）、雀頭役牌等で加算 |
| 順位期待値 (Placement EV) | 素点収支だけでなく、各着順（1〜4位）の確率分布にウマ・オカ順位点を掛け合わせた総合評価値 |

---

## 3. 全体要件

### 3.1 ユーザーニーズ

- 愚形残りや序盤・終盤の巡目差、他家のリーチに対して、AIが実戦のプロや高段位プレイヤーに近い押し引き判断を下せるようにしたい。
- ピンフや七対子、高符の手牌で正確な打点期待値が算出されるようにしたい。
- オーラス等で、親リーチに対する放銃リスクが正しく評価され、無理な攻めを回避できるようにしたい。

### 3.2 前提条件・制約

**前提条件:**
- 既存のRust単体テストおよびPython API呼び出しとの後方互換性を完全に保つ。
- 計算処理はミリ秒単位（リアルタイムHUDおよび局面検討で遅延が生じない速度）で完了すること。

**制約:**
- コマンド実行時は1行ずつの独立実行を行うこと。
- `definition/` は参照専用とし、成果物は `mahjong-definition/` および `mahjong/` 配下に格納すること。

---

## 4. 機能要件

### 4.1 和了確率推定モデルの改善 (Issue #76)

#### REQ-001: 待ち形・巡目・他家状況に応じた和了確率の動的算出

**説明:**  
The system shall calculate the win probability `win_probability` dynamically by adjusting advance probabilities based on:
1. Wait shape (favorable wait vs unfavorable wait): Unfavorable waits shall receive a dynamic decay factor (approx. 0.60–0.70x) compared to favorable multi-sided waits.
2. Turn decay: Progressing turns shall apply non-linear probability decay reflecting remaining wall depletion and defense tendencies.
3. Opponent threat decay: The existence of opponent riichi declarations or multi-melded hands shall scale down the win probability.

**受入条件:**
- [ ] テンパイ時、好形待ち（両面等）と愚形待ち（嵌張・辺張・単騎）で和了確率に適切な格差が生じること。
- [ ] 他家にリーチ者が存在する場合、非リーチ時と比較して和了確率が減衰すること。
- [ ] 一向聴・二向聴においても、受入枚数と好形度に応じた確率が算出されること。

**優先度:** Must  

---

### 4.2 想定和了打点算出の高度化 (Issue #77)

#### REQ-002: 面子分解（HandPattern）に基づく正確な符計算と高点法・門前ダマテン・探索拡張・暗槓門前維持

**説明:**  
The system shall calculate the expected winning score by:
1. Decomposing winning hand into valid meld patterns (`HandPattern`), evaluating triplets, sequences, quads, pair, and wait shape (ryanmen, shanpon, kanchan, penchan, tanki), and choosing the decomposition that yields the highest score (高点法の適用).
2. Distinguishing sequences crossing overlapping tiles (e.g. 123m 234m 345m) from triplets to eliminate false triplet fu additions.
3. Evaluating damaten suitability when the hand possesses legitimate yaku without riichi, preventing score inflation.
4. Expanding one-shanten branch simulation up to 8 candidate accepted tiles.
5. Correctly recognizing hands with only concealed quads (`Meld::Ankan`) as fully concealed (門前清 / `is_closed == true`), preserving the 10-fu closed ron bonus (門前ロン加符 10符) and closed-hand yaku (such as Riichi, Menzen Tsumo) across scoring and expectation evaluation.

**受入条件:**
- [ ] 順子が重なった手牌（例: 123m 234m 345m）において、同一牌が3枚あっても暗刻符が誤加算されないこと。
- [ ] 辺張待ち・嵌張待ち・単騎待ちの2符が正しく加算されること（例: 111m 234p 456p 78s 99s の6sツモが40符となること）。
- [ ] ピンフツモ（20符）および七対子（25符）で正確な打点が算出されること。
- [ ] 複数の分解が存在する場合、最高得点となる分解が採用されること。
- [ ] 門前役あり手でダマテン想定打点が算出され、不要なリーチ依存の打点過大評価が抑制されること。
- [ ] 暗槓（`Meld::Ankan`）のみの手牌において、副露手として扱われず門前（`is_closed = true`）が維持され、門前ロン加符10符および立直等の門前役が正しく評価されること。

**優先度:** Must  

---

### 4.3 危険度判定および押し引き評価の高度化 (Issue #78)

#### REQ-003: 河・現物・スジ・カベ・生牌を考慮した動的危険度評価および分析対象座席の整合

**説明:**  
The system shall evaluate tile safety and risk score based on contextual river discards:
1. Genbutsu (safe against declarer): Risk score shall be strictly `0.0`.
2. Suji (no-riichi wait on two-sided): Risk score shall be reduced based on double-suji (`~0.15`), half-suji terminal (`~0.08`), half-suji simple (`~0.18-0.25`).
3. Honor tiles: Shall be evaluated dynamically based on visible copies (4 visible: `0.0`, 3 visible: `0.05`, unrevealed/fresh: `0.35`).
4. Wall (No Chance / One Chance): Outer tiles shall have reduced risk scores when blocker counts reach 3 or 4.
5. In the absence of opponent riichi/threat in early turns, penalty for discarding middle tiles shall be minimized.
6. Public Python API (`py_evaluate_hand_discards`, `py_evaluate_placement_discards`, `py_get_ai_hud_data`) and CPU game loops shall accept and bind all opponent context (riichi, rivers, melds, dealer).
7. Contextual seat alignment: `AnalysisContext` shall identify the target evaluation seat (`target_player: usize`). All opponent threat and defense evaluations (`estimate_win_probability`, `evaluate_tile_safety`, deal loss) shall consistently evaluate players where `p != target_player`, properly treating opponent 0 as a threat when evaluating player indices 1, 2, or 3, and never treating self as an opponent threat.

**受入条件:**
- [ ] リーチ者の河にある現物牌の危険度が `0.0` と判定されること。
- [ ] スジ牌の危険度が無筋の中張牌よりも大幅に低減されること。
- [ ] リーチが入った局面において、現物・安全牌のEVが無筋危険牌を上回り、ベタオリ選択が正しく機能すること。
- [ ] Python API および CLI サンプルにおいて対局状態から安全度・順位EVが一元的に計算されること。
- [ ] `target_player != 0`（例: CPUや他家 player_idx）の分析時において、プレイヤー自身のリーチは他家脅威とならず、プレイヤー0のリーチが正しく他家脅威として和了率減衰・危険度判定されること。

**優先度:** Must  

---

### 4.4 順位期待値（Placement EV）の失点モデル改善 (Issue #79)

#### REQ-004: 放銃相手と局面に応じた動的失点期待値モデル

**説明:**  
The system shall calculate the deal-in point loss dynamically in Placement EV evaluation:
1. Dealer deal-in shall assume a higher base loss (12,000 points / mangan) compared to non-dealer deal-in (8,000 points).
2. When multiple opponents have declared riichi, the system shall evaluate the maximum risk among all active riichi callers (prioritizing dealer riichi over non-dealer riichi).
3. Dora indicator counts shall not artificially reduce deal-in loss; base benchmark losses (dealer 12,000 pt, non-dealer 8,000 pt, non-riichi open meld 5,200/9,600 pt) plus honba bonuses shall be applied reliably.
4. Post-round scores and rank probability distributions shall reflect this differentiated deal-in loss.

**受入条件:**
- [ ] 親リーチに対する放銃シナリオで 12,000 点失点として着順確率が試算されること。
- [ ] 子と親が共にリーチしている複数リーチ局面で、親リーチ（12,000点）を優先して失点見積もりがなされること。
- [ ] 子に対する放銃シナリオで 8,000 点（または副露に応じた適正打点）として試算されること。
- [ ] オーラス等で親リーチに対する守備判断がより厳密に下されること。

**優先度:** Must  

---

## 5. 要件品質チェックリスト

| REQ番号 | 必要性 | 明白性 | 単一性 | 検証性 | 正確性 |
|---|:---:|:---:|:---:|:---:|:---:|
| REQ-001 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-002 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-003 | ☑ | ☑ | ☑ | ☑ | ☑ |
| REQ-004 | ☑ | ☑ | ☑ | ☑ | ☑ |

---

## 6. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-09-14 | 初版作成（Issue #76, #77, #78, #79 対応） | Antigravity |
