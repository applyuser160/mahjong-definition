# 基本・詳細設計書: 数牌スーツLUT（suit_table）のキャッシュ局所性改善 & パッキング圧縮

> 本設計書は、Issue #107 の解決に向けた `suit_table.rs` および `build.rs` のデータ構造・ビットレイアウト・パッキング設計です。

---

## メタ情報

| 項目 | 値 |
|------|-----|
| プロジェクト名 | mahjong |
| 機能名 | 数牌スーツLUTキャッシュ局所性改善 & パッキング圧縮（Issue #107） |
| バージョン | 1.0.0 |
| 作成日 | 2026-09-17 |
| 最終更新日 | 2026-09-17 |
| 作成者 | Antigravity AI |
| ステータス | Review |

---

## 1. ビットパッキング設計

### 1.1 丸めの正当性証明
シャンテン数計算では、面子合計 $M$ と搭子合計 $T$ から以下のように向聴数を算出します：
$$S = 8 - 2M - \min(T, 4 - M) - \text{雀頭ボーナス}$$
各スーツ単体で見ると、ある面子数 $m \in \{0..4\}$ に対して、そのスーツ内で作れる搭子数が $4 - m$ を超えていたとしても、手牌全体で使える搭子は最大でも $4 - M \le 4 - m$ 個です。
したがって、各面子数 $m$ ごとの搭子数 $t$ を $\min(t, 4 - m)$ で切り詰めても、手牌全体での和 $\sum \min(t_i, 4 - m_i)$ は常に必要な搭子数を満たし、評価結果は厳密に不変です。

### 1.2 30bit `u32` ビットレイアウト
搭子数 $t$ の丸め後、値の取りうる範囲は以下の 6 値です：
- $-1$: 構成不可
- $0$: 0個
- $1$: 1個
- $2$: 2個
- $3$: 3個
- $4$: 4個

これを 3 bit（$0..=5$）にマッピング：
$$\text{code} = \begin{cases} 0 & (t = -1) \\ t + 1 & (t \ge 0) \end{cases}$$
復元時は：
$$t = \begin{cases} -1 & (\text{code} = 0) \\ (\text{code} - 1) \text{ as i8} & (\text{code} > 0) \end{cases}$$

10 セルの配置（各 3 bit、計 30 bit）：
- bit 0..2: `no_head[0]`
- bit 3..5: `no_head[1]`
- bit 6..8: `no_head[2]`
- bit 9..11: `no_head[3]`
- bit 12..14: `no_head[4]`
- bit 15..17: `with_head[0]`
- bit 18..20: `with_head[1]`
- bit 21..23: `with_head[2]`
- bit 24..26: `with_head[3]`
- bit 27..29: `with_head[4]`
- bit 30..31: 未使用（0）

### 1.3 データ構造の定義

```rust
#[repr(transparent)]
#[derive(Clone, Copy, Debug, PartialEq, Eq, Default)]
pub struct PackedSuitEntry(pub u32);

impl PackedSuitEntry {
    #[inline(always)]
    pub fn unpack(self) -> SuitEntry {
        let val = self.0;
        let decode = |shift: u32| -> i8 {
            let code = (val >> shift) & 0b111;
            if code == 0 { -1 } else { (code - 1) as i8 }
        };

        SuitEntry {
            no_head: [
                decode(0),
                decode(3),
                decode(6),
                decode(9),
                decode(12),
            ],
            with_head: [
                decode(15),
                decode(18),
                decode(21),
                decode(24),
                decode(27),
            ],
        }
    }

    #[inline(always)]
    pub fn pack(entry: &SuitEntry) -> Self {
        let encode = |val: i8, m: usize| -> u32 {
            if val < 0 {
                0
            } else {
                let clamped = (val as usize).min(4 - m) as u32;
                clamped + 1
            }
        };

        let mut packed = 0u32;
        for m in 0..5 {
            packed |= encode(entry.no_head[m], m) << (m * 3);
            packed |= encode(entry.with_head[m], m) << (15 + m * 3);
        }
        Self(packed)
    }
}
```

---

## 2. 方式比較とテーブル構成

| 項目 | 既存 (10 bytes) | 方式 A: Base-5 直接インデックス (4 bytes) | 方式 B: 有効パターン圧縮 (4 bytes) |
|---|---|---|---|
| エントリ数 | 1,953,125 | 1,953,125 | 405,350 |
| エントリサイズ | 10 bytes | 4 bytes (`u32`) | 4 bytes (`u32`) |
| テーブル容量 | 18.63 MiB | 7.45 MiB (-58%) | 1.55 MiB (-91.7%) |
| インデックス変換 | $O(1)$ (乗加算のみ) | $O(1)$ (乗加算のみ、変換不要) | 順位計算・DP探索が必要 |
| キャッシュ適合性 | L3 からも溢れやすい | 大幅改善 | L2/L3 にほぼ完全に収まる |

方式 A は、インデックス変換ロジックを一切追加せずにテーブルサイズを 18.6MB から 7.45MB に即座に半減させ、一切の CPU オーバーヘッドなしに L3 キャッシュヒット率を向上させます。
方式 B は 1.55MB まで縮小できますが、変換テーブルまたは計算コストが発生します。

本改修では、まず **直接インデックス 4 バイト化**（7.45MB）を確実に実装して完全な互換性と安全性を担保した上で、ベンチマーク測定を実施します。

---

## 3. テスト設計

1. **パッキング・アンパック可逆性・丸めテスト**:
   - ランダム・全境界値での `PackedSuitEntry::pack` と `unpack` の値が、$\min(\text{taatsu}, 4 - m)$ の期待値と完全一致すること。
2. **シャンテン数同値性テスト**:
   - 既存の全シャンテンテスト（チートイツ・国士・通常手・刻子・順子・対子組み合わせ）がすべて完全一致すること。
3. **ベンチマーク**:
   - シャンテン数計算スループットおよび冷間起動時間の測定。
