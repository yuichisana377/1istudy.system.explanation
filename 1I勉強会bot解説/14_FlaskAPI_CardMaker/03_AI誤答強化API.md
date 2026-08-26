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
- **`CARDMAKER_AI_SHORTLIST_SIZE`（40、2026/08/26追加、同日16→40へ拡大）**：クライアントが送ってくる`candidates`（各問題の誤答候補）の件数上限。みんなでクイズ側の`QUIZ_AI_SHORTLIST_SIZE`（12、ユーザー希望で現状維持）とは**あえて別定数**にしてある。CardMaker側は「4択にする」の対象デッキを答えの異なり10件超に厳格化した結果、対象デッキの候補プール自体が元々大きくなった。加えて、JS側の綴り類似度による事前絞り込み（bigram＋文字数）だけでは、記述式（文章）の解答で「本当は紛らわしいのに順位が低くて漏れる」候補が出やすいという指摘を受け、AIが「デッキの中から広く探せる」よう候補数をさらに拡大した。ここを共有定数のままにすると、CardMaker側の都合でみんなでクイズのAI強化の挙動まで変わってしまうため、分離が必要だった。
- **`CARDMAKER_AI_MAX_ITEMS`（10問）・`CARDMAKER_AI_MAX_TEXT_LEN`（200文字）**：
  クライアントから直接テキストを受け取る構造上、巨大なペイロードや大量の問題数を
  送りつけられる余地があるため、件数・文字列長の両方に上限を設けている
  （みんなでクイズ側は`QUIZ_MAX_QUESTIONS`等デッキ由来の制約で自然に収まるが、
  こちらはその保護が無いため明示的な上限が必要）。
- **専用プロンプト`_build_cardmaker_distractor_prompt`（2026/08/26追加）**：当初は`_build_quiz_distractor_prompt`（みんなでクイズと共用）を使っていたが、「もっと難しくしてほしい」という要望を受け、CardMaker専用のプロンプトへ切り替えた。考え方（プールから選び、無ければ生成で補う）は共通だが、文言をより踏み込んだもの（「本当に紛らわしいレベルまで作り込む」「妥協せず新しく考えて補う」）にしている。みんなでクイズ側のプロンプトはユーザーの明示的な希望で現状維持と決まっているため、影響を分離する目的で別関数にした。
- **`_ollama_generate_json`（Ollamaへの問い合わせ）・`_sanitize_ai_distractors`**
  （応答の検証：重複なく・正解と異なり・禁止文字を含まない文字列がちょうど3件
  そろうかの確認）はみんなでクイズ側と共用。
- **タイムアウト**：`max(QUIZ_AI_TIMEOUT_SEC, 25 * len(items))`（候補数がクイズ側より多いぶん、1問あたりの見積もりをクイズ側の20秒より少し余裕を持たせて25秒にしている）。`CARDMAKER_AI_MAX_ITEMS=10`が上限なので、最大でも250秒程度に収まる。
- **`keep_alive: "30m"`（`_ollama_generate_json`、2026/08/26追加）**：Ollamaの既定（アイドル5分でモデルをアンロード）のままだと、クイズ/CardMakerの利用間隔が空くたびにモデル再ロード（数秒）が生成時間に上乗せされる。常駐時間を延ばして、体感の「最初の1問が出るまでの時間」を縮めた（この変更はみんなでクイズ側にも効く、共通のヘルパーへの変更）。
- **軽量モデル（`qwen2.5-coder:3b`）への切り替えは検証の結果、見送り**：「もっと早く4択を出したい」という要望を受け、実際に3bモデルで同じプロンプトを試したところ、**7bより遅く（57秒 vs 28.6秒）、かつ候補をほぼ丸ごと返すだけで実質選別していない**（`distractors`に3件どころか渡した候補全部が入っていた）という結果になり、速度・精度の両方で7bに劣ることが実測で確認できたため、モデル変更では解決しないと判断した。

## 呼び出し側の設計（フロント）

フロント（Cardmaker.js）は、このAPIを**学習開始をブロックしない**形で使う：
学習開始時点でまず綴り類似度ベースの4択を即座に組み立てて学習を始められるように
しておき、裏でこのAPIを数問ずつ呼び出して、返ってきたカードから順に選択肢を
差し替えていく。失敗・タイムアウトしても、最初に組み立てた4択のまま問題なく
遊べる（ベストエフォート）設計は、クイズ側の「ロビー中にバックグラウンドスレッドで
強化し、間に合わなければ諦める」という方針と同じ考え方を踏襲している。
**最初の1問だけはバッチサイズ1で送り（2問目以降は3問ずつ）、一番乗りの改善結果が
できるだけ早く届くようにしている**（2026/08/26追加。1問だけの生成は複数問まとめてより
明らかに速いため）。詳細はフロント側の解説
（[07_Cardmaker.js_その7](../../1I勉強会web解説/02_Cardmaker/07_Cardmaker.js_その7_学習モードとクイズ再生.md)）参照。

### 実測でわかった限界

記述式（長文）の解答・10件規模の候補を含む現実的なプロンプトで実測したところ、
1問だけの問い合わせでも約35秒かかった（短い単語同士の問題なら数秒〜十数秒で
済むこともあるが、文章が長くなるほどプロンプトのトークン数が増え、CPU動作の
7Bモデルでは比例して時間がかかる）。これはCPUのみ・GPU無しというハードウェア
制約に起因するもので、バッチサイズや候補数の調整だけでは超えられない実質的な
下限にほぼ到達している。さらに高速化するには、GPUの追加か、3bより高精度な
軽量モデルの登場を待つ必要があると考えられる。
