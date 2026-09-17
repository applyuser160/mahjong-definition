# ADR 0010: ロン和了処理のライフサイクル整合性と通信堅牢化

## ステータス
承認済 (Accepted)

## コンテキスト
CPU対局中に人間プレイヤーがロン和了した際、`POST /api/match/call` が 400 Bad Request となり画面がクラッシュする不具合が発生した。
調査の結果、副露選択肢（`pending_call_options`）が和了後に消去されず UI に残り続けて二重送信されること、和了後手牌（14枚）や局終了状態で AI HUD の打牌候補評価が誤って実行されること、およびフロントエンドの WebSocket 再接続タイマー多重化が複合して起因していた。

## 決定事項
1. **状態クリアの強制化**: `_handle_ron`, `_handle_ryuukyoku`, `start_round` で `pending_call_options` を必ず `None` に初期化する。
2. **手牌枚数チェックによる多牌防止**: 和了牌追加時に `len(hand) % 3 == 1` を検証し、14枚以上の状態で重複追加されることを防止する。
3. **局終了時のAI HUD計算抑止**: `self.status == "waiting_user_discard" and self.current_turn == 0` の場合のみ AI HUD（打牌評価）を算出し、局終了時（`round_end`）は `{}` を返却する。
4. **ActionPrompt の状態バインドと連打防止**: `matchStatus === "waiting_user_call"` を表示条件に含め、クリック後の多重送信を抑止する。
5. **WebSocket ライフサイクルの適正管理**: タイマークリーンアップと `readyState === WebSocket.OPEN` の直接確認を行う。

## 影響・効果
- ロン和了後にボタンが正常に消去され、リザルト画面へスムーズに遷移する。
- 局終了時やCPU思考中のバックエンドCPU負荷が大幅に軽減される。
- WebSocket接続が安定し、開発環境（React StrictMode等）でも不要な多重接続や HTTP フォールバックエラーが発生しなくなる。
