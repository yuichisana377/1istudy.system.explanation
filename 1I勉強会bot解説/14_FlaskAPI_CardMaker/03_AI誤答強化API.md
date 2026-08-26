# CardMaker学習モード用のAI誤答強化API（`bot.py` `/cardmaker_ai_distractors`）（2026/08/26追加）

対象：`bot.py`の`/cardmaker_ai_distractors`。

## 背景：みんなでクイズのAI強化をCardMakerでも使いたい

[../15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md](../15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md)の
「7. ローカルAIによる誤答の強化」で、「みんなでクイズ」の自動生成4択がローカルAI
（Ollama、`qwen2.5-coder:7b`）で誤答を強化するようになったのに続き、CardMakerの
通常デッキ学習（フラッシュカード）にも「自動採点する」＋「4択にする」を選ぶと
同じ形式で遊べる機能が追加された（フロント側は
[../../1I勉強会web解説/02_Cardmaker/07_Cardmaker.js_その7_学習モードとクイズ再生.md](../../1I勉強会web解説/02_Cardmaker/07_Cardmaker.js_その7_学習モードとクイズ再生.md)参照）。

このAPIは、その裏側でローカルAIに問い合わせるための**汎用エンドポイント**。

## みんなでクイズ版との違い：ファイルを読まない

みんなでクイズの`_ai_pick_quiz_distractors`は、サーバー上のデッキファイル
（`words/*.json`）を読み込んで問題プールを作っていた。CardMaker側は事情が違う：
学習しているデッキのカードデータは、学習開始時点で**既にクライアント（ブラウザ）側に
全部揃っている**（`ensureDeckCardsLoaded`で読み込み済み）。そのため、このAPIは
ファイルを一切読まず、クライアントが直接渡してきた「問題文・正解・候補」をそのまま
ローカルAIに問い合わせ、応答を検証して返すだけの薄いラッパーになっている。

```python
CARDMAKER_AI_MAX_ITEMS = 10
CARDMAKER_AI_MAX_TEXT_LEN = 200

@app.route("/cardmaker_ai_distractors", methods=["POST"])
def cardmaker_ai_distractors():
    data = request.get_json(silent=True) or {}
    guild_id, student_id, nickname, err = require_login_json(data)
    if err:
        return err
    if not OLLAMA_HOST:
        return jsonify({"ok": False, "error": "ai_unavailable"})

    raw_items = data.get("items")
    if not isinstance(raw_items, list) or not raw_items or len(raw_items) > CARDMAKER_AI_MAX_ITEMS:
        return jsonify({"ok": False, "error": "invalid_items"})

    items = []
    for it in raw_items:
        ...
        items.append({"i": it["i"], "question": question, "correct": correct, "candidates": candidates})

    timeout = max(QUIZ_AI_TIMEOUT_SEC, 20 * len(items))
    result = _ollama_generate_json(_build_quiz_distractor_prompt(items), timeout=timeout)
    ...
    return jsonify({"ok": True, "questions": out})
```

- **`require_login_json`必須**：他のCardMaker系「変更」APIと違い、このAPIはデータを
  何も書き換えない（読み取り専用に見える）が、ローカルAIへの問い合わせという
  実際に計算資源（CPU）を消費する処理を起動してしまうため、匿名（未ログイン）の
  第三者がインターネット経由で叩いて資源を消費させることを防ぐ目的でログインを
  必須にしている（クイズ側の`_ai_enhance_quiz_room_choices`はルーム作成に紐づく
  ため元々この心配が薄いが、こちらは単体の汎用APIなので明示的にガードした）。
- **`CARDMAKER_AI_MAX_ITEMS`（10問）・`CARDMAKER_AI_MAX_TEXT_LEN`（200文字）**：
  クライアントから直接テキストを受け取る構造上、巨大なペイロードや大量の問題数を
  送りつけられる余地があるため、件数・文字列長の両方に上限を設けている
  （みんなでクイズ側は`QUIZ_MAX_QUESTIONS`等デッキ由来の制約で自然に収まるが、
  こちらはその保護が無いため明示的な上限が必要）。
- **既存ヘルパーの再利用**：`_build_quiz_distractor_prompt`（プロンプト組み立て）・
  `_ollama_generate_json`（Ollamaへの問い合わせ）・`_sanitize_ai_distractors`
  （応答の検証：重複なく・正解と異なり・禁止文字を含まない文字列がちょうど3件
  そろうかの確認）は、みんなでクイズ側で定義したものをそのまま流用している。
  「AIに何を聞き、どう検証するか」のロジックは完全に共通で、「候補をどこから
  集めてくるか（ファイルから読むか、クライアントから受け取るか）」だけが違う、
  という切り分けになっている。
- **タイムアウトも同じ考え方**：`max(QUIZ_AI_TIMEOUT_SEC, 20 * len(items))`で
  問題数に応じて動的に伸ばす（クイズ側と全く同じ式）。ただしこちらは
  `CARDMAKER_AI_MAX_ITEMS=10`が上限なので、最大でも200秒程度に収まる。

## 呼び出し側の設計（フロント）

フロント（Cardmaker.js）は、このAPIを**学習開始をブロックしない**形で使う：
学習開始時点でまず綴り類似度ベースの4択を即座に組み立てて学習を始められるように
しておき、裏でこのAPIを数問ずつ（5問ずつ）呼び出して、返ってきたカードから順に
選択肢を差し替えていく。失敗・タイムアウトしても、最初に組み立てた4択のまま
問題なく遊べる（ベストエフォート）設計は、クイズ側の
「ロビー中にバックグラウンドスレッドで強化し、間に合わなければ諦める」という
方針と同じ考え方を踏襲している。詳細はフロント側の解説
（[07_Cardmaker.js_その7](../../1I勉強会web解説/02_Cardmaker/07_Cardmaker.js_その7_学習モードとクイズ再生.md)）参照。
