# ルーム状態のJSON化・4択の自動生成・過去問アーカイブ（`bot.py` 4877〜5128行）

対象：`bot.py`の`_quiz_room_players_json`・`_quiz_room_snapshot`・`_pick_distractors`・`_build_deck_questions`・`_validate_manual_questions`・`_archive_manual_quiz`・`_archive_room_if_needed`。

## 1. 参加者一覧の整形：`_quiz_room_players_json`（4877〜4896行）

```python
def _quiz_room_players_json(room, include_correct=False):
    players = sorted(room["players"].values(), key=lambda p: -p["score"])
    result = []
    for p in players:
        entry = {"id": p["id"], "nickname": p["nickname"], "color": p["color"], "text_color": p["text_color"], "score": p["score"]}
        if include_correct:
            # ★ 正解発表(reveal)中だけ、他の参加者の今回の問題への正誤も見せる。
            #   出題中(question)にこれを渡すと、まだ発表前の正解がバレてしまうため、
            #   include_correct=True は reveal のときにしか呼ばない。
            answered = p["cur_answer"] is not None
            entry["answered"] = answered
            entry["correct"] = bool(p["cur_correct"]) if answered else None
        result.append(entry)
    return result
```
- `sorted(..., key=lambda p: -p["score"])`… スコアの**降順**（マイナスを付けることで、通常は昇順のソートを降順に反転させています）で並べ、順位表として使えるようにします。
- `include_correct`引数… コメントにある通り、これは`True`にすると各参加者の「今回の問題に正解したか」まで含めます。この情報は、正解発表（`reveal`）の段階になって初めて公開してよい情報なので、呼び出し側は`question`（出題中）の段階では絶対に`True`にしないという規律が求められます。この関数自体は「渡された引数に従って情報を含めるかどうか」を判断するだけで、**いつ呼ぶべきかの判断は呼び出し元の責任**という設計です。

## 2. ルーム状態のスナップショット：`_quiz_room_snapshot`（4898〜4951行）

これは、フロント側がポーリングで受け取る、その時点でのルームの状態全体です。**`state`の値によって、含まれる情報が意図的に変わります**。

```python
def _quiz_room_snapshot(room, student_id):
    snap = {
        "code": room["code"],
        "title": room["title"],
        "state": room["state"],
        "host_nickname": room["host_nickname"],
        "players": _quiz_room_players_json(room),
    }
    if room["state"] == "countdown":
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "countdown_started_at": room["countdown_started_at"],
            "countdown_duration_sec": QUIZ_COUNTDOWN_DURATION_SEC,
        })
    elif room["state"] == "intro":
        # ★ 「第N問」表示中は、まだ問題文・選択肢は渡さない
        #   （question状態になってから渡せば十分で、渡す情報は少ない方がよい）。
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "intro_started_at": room["intro_started_at"],
            "intro_duration_sec": QUIZ_INTRO_DURATION_SEC,
        })
```
- `"intro"`（「第N問」を見せている段階）では、**まだ問題文や選択肢自体を渡していません**。コメントにある通り「question状態になってから渡せば十分で、渡す情報は少ない方がよい」という、最小限の情報しか渡さない方針です。

```python
    elif room["state"] in ("question", "reveal"):
        q = room["questions"][room["current_q"]]
        revealed = room["state"] == "reveal"
        question_payload = {"question": q["question"], "choices": q["choices"]}
        # ★ 正解番号は、発表(reveal)されるまでは誰にも渡さない（レスポンスを
        #   devtools等で覗かれてカンニングされるのを防ぐ）。ホストも今は
        #   1プレイヤーとして参加するため、ホストだけ特別扱いはしない。
        if revealed:
            question_payload["correct_index"] = q["correct_index"]
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "question": question_payload,
            "question_started_at": room["question_started_at"],
            "time_limit_sec": room["time_limit_sec"],
            "answered_count": sum(1 for p in room["players"].values() if p["cur_answer"] is not None),
            "total_players": len(room["players"]),
        })
        if revealed:
            snap["first_correct_nickname"] = room.get("first_correct_nickname")
            snap["reveal_started_at"] = room.get("reveal_started_at")
            snap["reveal_duration_sec"] = QUIZ_REVEAL_DURATION_SEC
            # ★ 発表中だけ、全員分の正誤(◯×)を含めて players を上書きする
            snap["players"] = _quiz_room_players_json(room, include_correct=True)
        player = room["players"].get(student_id)
        if player is not None and player["cur_answer"] is not None:
            snap["your_answer"] = player["cur_answer"]
            if revealed:
                snap["your_correct"] = bool(player["cur_correct"])
    return snap
```
- **これがこの関数で最も重要な安全対策です**：`question_payload`には常に`question`（問題文）と`choices`（選択肢）が含まれますが、`correct_index`（正解の番号）は`revealed`（＝`state == "reveal"`）のときだけ含まれます。もしこのガードが無く、出題中から正解番号がレスポンスに含まれていたら、ブラウザの開発者ツールでネットワーク通信の中身を見るだけで、誰でも正解が分かってしまいます（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)の設計方針にあった「正解番号はホストにも発表されるまでは一切渡さない」の実装がここです）。
- `revealed`のときだけ`_quiz_room_players_json(room, include_correct=True)`で`players`を**上書き**しています。これにより、正解発表の瞬間だけ、他の参加者の正誤（◯×）が一覧に反映されます。
- `player["cur_answer"]`（**自分自身**の回答）は、`revealed`かどうかに関わらず渡されます。自分が何を選んだかは、他の参加者に隠す必要が無い（自分自身の情報だから）ためです。ただし、自分の回答が「正解だったかどうか」（`your_correct`）は、他の参加者の正誤と同様に`revealed`のときだけ渡されます。

## 3. 見分けにくい誤答の生成：`_pick_distractors`（4953〜4976行）

```python
def _pick_distractors(correct: str, pool: list, k: int) -> list:
    """
    ★ 完全ランダムに選ぶと、他のカードの答えと文字数も内容もかけ離れた
      誤答ばかりになりがちで、見た目だけで消去法に正解できてしまっていた。
      正解と文字列として近い（綴り・字面が似ている）ものを優先候補にし、
      その中からランダムに選ぶことで、きちんと覚えていないと迷うような
      4択にする。
    """
    def _score(a):
        seq_ratio = difflib.SequenceMatcher(None, correct, a).ratio()
        longer = max(len(correct), len(a), 1)
        length_ratio = 1 - abs(len(correct) - len(a)) / longer
        return seq_ratio * 0.7 + length_ratio * 0.3

    scored = sorted(pool, key=_score, reverse=True)
    pool_size = max(k, min(len(scored), k * 2))
    return random.sample(scored[:pool_size], k)
```
- これはクイズの「面白さ・妥当性」に関わる、興味深いアルゴリズムです。単に不正解の選択肢をランダムに選ぶだけだと、コメントの通り「正解だけ極端に短い/長い」のような、内容を知らなくても見た目だけで消去法により正解できてしまう、質の低い4択になりがちでした。
- `difflib.SequenceMatcher(None, correct, a).ratio()`… [../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)などで登場した`difflib`ライブラリの、今度は**類似度を測る**機能です。2つの文字列がどれだけ似ているか（0〜1の比率）を計算します。
- `_score`関数は、**文字列としての似ている度合い（70%の重み）**と**文字数の近さ（30%の重み）**を組み合わせたスコアです。コメントの追加説明にある通り、綴りの類似度だけだと「文字数が全然違うせいで一目で誤答と分かる」選択肢が紛れ込みやすかったため、文字数の近さも加味するよう改良されています。
- `scored = sorted(pool, key=_score, reverse=True)`… 候補全体をスコアの高い順（正解に似ている順）に並べます。
- `pool_size = max(k, min(len(scored), k * 2))`… 上位何件から実際に選ぶかを決めます。`k=3`（3つの誤答が必要）なら、`pool_size`は最大6件です。コメントにある通り、以前は「上位9件からランダムに3件」だったものを「上位6件から3件」とタイトに絞ることで、「似ているが選ばれなかった紛らわしい候補が混ざりにくく」し、消去法が効きにくい4択にする改善が行われています。
- `random.sample(scored[:pool_size], k)`… 上位の候補群から、最終的にランダムに`k`個を選びます。**完全にスコア順で機械的に選ぶのではなく、上位群からランダムに選ぶ**ことで、同じデッキから何度クイズを作っても、毎回少しずつ違う顔ぶれの誤答になるようにしています。

## 4. デッキからの自動問題生成：`_build_deck_questions`（4979〜5037行）

```python
def _build_deck_questions(deck_filenames, num_questions):
    """
    ★ deck_filenames は単一のファイル名（文字列）でも、複数のファイル名の
      リストでも受け取れる。複数指定した場合は、それぞれのデッキのカードを
      まとめて1つの問題プールとして扱い、distractorも選んだデッキ全体から選ぶ。
    (questions, error_code) を返す（成功時 error_code は None）。"""
    if isinstance(deck_filenames, str):
        deck_filenames = [deck_filenames]
    if not isinstance(deck_filenames, list) or not deck_filenames:
        return None, "deck_not_found"
    # ★ 重複を除きつつ順序は維持する
    seen_filenames = set()
    deck_filenames = [f for f in deck_filenames if not (f in seen_filenames or seen_filenames.add(f))]
    if len(deck_filenames) > QUIZ_MAX_SOURCE_DECKS:
        return None, "too_many_decks"
```
- `deck_filenames`は文字列でもリストでも受け付ける、柔軟な引数設計です。`isinstance(deck_filenames, str): deck_filenames = [deck_filenames]`で、単一の文字列が来た場合は1要素のリストに正規化してから、以降は常にリストとして扱います。
- `[f for f in deck_filenames if not (f in seen_filenames or seen_filenames.add(f))]`… これは、Pythonでよく使われる「重複除去しつつ順序を保つ」ためのトリックです。`seen_filenames.add(f)`は常に`None`（Pythonの`set.add`は戻り値を返さない）を返すため、`f in seen_filenames`が`False`（まだ見ていない）のときだけ、`or`の右側で`.add(f)`が実行されて`seen_filenames`に追加され、リスト内包表記の条件全体としては`not (False or None)` = `not None` = `True`となり、その要素が残ります。既に見た要素は`f in seen_filenames`が`True`になるため、`.add`は評価されず（`or`の短絡評価）、条件全体が`False`になって除外されます。

```python
    cards = []
    for deck_filename in deck_filenames:
        if not isinstance(deck_filename, str) or "/" in deck_filename or "\\" in deck_filename or ".." in deck_filename:
            return None, "deck_not_found"
        data, _ = get_card_file(deck_filename)
        if data is None:
            return None, "deck_not_found"
        cards.extend(
            c for c in data.get("cards", [])
            if isinstance(c, dict) and str(c.get("question") or "").strip() and str(c.get("answer") or "").strip()
        )
    unique_answers = {c["answer"].strip() for c in cards}
    if len(cards) < 4 or len(unique_answers) < 4:
        return None, "deck_too_small"
```
- ここでも[../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)と同じパストラバーサル対策があります。
- 指定された全デッキから、問題文・解答の両方が空でないカードだけを`cards`に集約します。
- `unique_answers = {c["answer"].strip() for c in cards}`… 集合（set）内包表記で、重複を除いた解答の種類数を数えます。コメントの通り「4択（正解1つ＋不正解3つ）を作るには、答えの異なり（ユニークな値）が最低4つ必要」というルールがここでチェックされます。

```python
    n = min(num_questions, len(cards)) if num_questions else len(cards)
    n = max(1, min(n, QUIZ_MAX_QUESTIONS, len(cards)))

    picked = random.sample(cards, n)
    questions = []
    for card in picked:
        correct = card["answer"].strip()
        pool = list(dict.fromkeys(
            c["answer"].strip() for c in cards if c is not card and c["answer"].strip() != correct
        ))
        if len(pool) < 3:
            return None, "deck_too_small"
        choices = _pick_distractors(correct, pool, 3) + [correct]
        random.shuffle(choices)
        questions.append({
            "question": card["question"].strip(), "choices": choices,
            "correct_index": choices.index(correct),
        })
    return questions, None
```
- `n = max(1, min(n, QUIZ_MAX_QUESTIONS, len(cards)))`… 出題数を「1問以上」「上限30問（`QUIZ_MAX_QUESTIONS`）以内」「実際に使えるカード数以内」の3つの制約すべてに収まるよう調整します。
- `random.sample(cards, n)`で、実際に出題するカードをランダムに選びます。
- 各カードについて、`pool`（そのカード以外の、正解と異なる解答の集合。`dict.fromkeys(...)`は順序を保ったまま重複除去する、リスト内包の`set`版に近いテクニック）を作り、[3. 見分けにくい誤答の生成：`_pick_distractors`（4953〜4976行）](#3-見分けにくい誤答の生成pick_distractors49534976行)の`_pick_distractors`で3つの誤答を選びます。
- `choices = _pick_distractors(...) + [correct]`で4つの選択肢を作り、`random.shuffle(choices)`でシャッフルしてから、`correct_index`（正解が何番目に来たか）を記録します。この時点ではまだ、誰が見ても正解が分からないよう、正解の位置自体もランダムです。

## 5. 手入力問題の検証：`_validate_manual_questions`（5039〜5063行）

```python
def _validate_manual_questions(raw_questions):
    """成功時: ((questions, check_fields), None) / 失敗時: (None, error_code)"""
    if not isinstance(raw_questions, list) or not raw_questions:
        return None, "invalid_questions"
    questions = []
    check_fields = {}
    for i, q in enumerate(raw_questions):
        if not isinstance(q, dict):
            return None, "invalid_questions"
        question_text = str(q.get("question") or "").strip()
        choices = q.get("choices")
        correct_index = q.get("correct_index")
        if not question_text or not isinstance(choices, list) or len(choices) != 4:
            return None, "invalid_questions"
        choices = [str(c or "").strip() for c in choices]
        if any(not c for c in choices):
            return None, "invalid_questions"
        if not isinstance(correct_index, int) or isinstance(correct_index, bool) or not (0 <= correct_index < 4):
            return None, "invalid_questions"
        check_fields[f"問題{i+1}の問題文"] = question_text
        for j, c in enumerate(choices):
            check_fields[f"問題{i+1}の選択肢{j+1}"] = c
        questions.append({"question": question_text, "choices": choices, "correct_index": correct_index})
    return (questions, check_fields), None
```
- ホストが「自分で問題を作る」オリジナル4択モードで入力したデータを検証します。デッキから自動生成する場合と違い、ここでは**利用者が自由に入力したテキスト**を扱うため、非常に丁寧なバリデーションが行われています。
- `len(choices) != 4`… 選択肢は必ずちょうど4つでなければなりません。
- `if not isinstance(correct_index, int) or isinstance(correct_index, bool) or not (0 <= correct_index < 4):`… [../09_FlaskAPI_予定管理/01_学習ログ記録APIのなりすまし対策と連打対策.md](../09_FlaskAPI_予定管理/01_学習ログ記録APIのなりすまし対策と連打対策.md)の`add_study_log`で見たのと同じ、`bool`が`int`のサブクラスであることへの対策が、ここにも登場しています。加えて、正解番号が0〜3の範囲に収まっているかも確認します。
- `check_fields`… 検証したテキストを、後で呼び出し元が[../02_データ保存基盤/02_設定ファイルと不正文字チェック.md](../02_データ保存基盤/02_設定ファイルと不正文字チェック.md)の`reject_if_bug_chars`にまとめて渡すための辞書です。この関数自身は不正文字チェックまでは行わず、その責務は呼び出し元に委ねられています。

## 6. クイズ結果のCardMakerアーカイブ（5065〜5128行）

```python
def _archive_manual_quiz(title, questions, student_id, nickname):
    """
    ★ ホストが「自分で問題を作る」（オリジナル4択）で作成したクイズを、
      CardMakerの「クイズ過去問」フォルダにデッキとして自動保存する。
      いつでも一人用選択式モードで遊べる「過去問」として残すため。
      呼び出し元は _archive_room_if_needed()（クイズが終了した時点で呼ばれる）。
    ・questions は _validate_manual_questions の戻り値そのもの
      （[{"question", "choices"(4件), "correct_index"}, ...]）。
    ・choice_mode/choices/correct_indices は、CardMaker側の選択式デッキ共通
      フォーマット。単一/複数正解はデッキ単位ではなく問題ごとに
      correct_indices の個数で決まる（CardMaker側の仕様）。Quiz.js自体は
      4択・単一正解の固定フォーマットのままで、ここでの変換にしか影響しない。
    ・answer（正解の選択肢文言）も入れておく。これにより単語検索・一覧表示・
      作成済みリストなど、「answerは文字列である」という前提の既存コードを
      一切変更せずに動かせる（choices/correct_indices は選択式UIだけが見る）。
    ・アーカイブに失敗しても、クイズ自体の進行は失敗させない（ベストエフォート）。
    """
    try:
        _ensure_quiz_archive_folder()
        cards = [{
            "id": secrets.token_hex(6),
            "question": q["question"],
            "answer": q["choices"][q["correct_index"]],
            "choices": q["choices"],
            "correct_indices": [q["correct_index"]],
            "explanation": "",
            "imgs_q": [], "imgs_a": [], "imgs_e": [],
        } for q in questions]
        filename = generate_card_filename()
        card_payload = {
            "name": title,
            "cards": cards,
            "subject": None,
            "folder_id": QUIZ_ARCHIVE_FOLDER_ID,
            "published_by": {"id": student_id, "nickname": nickname},
            "incomplete": False,
            "choice_mode": True,  # ★ 選択式デッキであることのマーカー（単一/複数は問題ごとにcorrect_indicesの個数で決まる）
            "quiz_archive": True,  # ★ 追加（2026/08/21）：クイズ過去問デッキであることのマーカー。
            # フォルダ位置に依存させると、フォルダの外へ移動できるようにした際に
            # 判定できなくなる（save_cardsが問題編集を禁止する対象の特定にも使う）ため、
            # デッキ自身に持たせる。
        }
        put_card_file(filename, card_payload)
        index_change = upsert_cards_index_entry(filename, card_payload)
        change = deck_file_diff(f"{CARDS_DIR}/{filename}", None, card_payload)
        detail = [c for c in (change, index_change) if c]
        log_event(
            "card",
            f"みんなでクイズの結果を「{title}」として「クイズ過去問」に保存しました（{len(cards)}問）。",
            actor=nickname,
            detail=detail if detail else None,
        )
    except Exception as e:
        print(f"[WARN] クイズ過去問の保存に失敗しました（クイズの進行自体は続行）: {e}")
```
- **CardMakerとクイズという、2つの独立した機能が繋がる面白い箇所です**。ホストが手作りしたオリジナルのクイズ問題を、クイズが終わった後に**自動的にCardMakerのデッキとして保存**し、いつでも一人用の選択式モードで復習できるようにしています。
- `answer`フィールドにも正解の選択肢文言を入れている理由がコメントに書かれています：「これにより単語検索・一覧表示・作成済みリストなど、『answerは文字列である』という前提の既存コードを一切変更せずに動かせる」。CardMaker側の多くの機能は、通常のフラッシュカードデッキ（`answer`が単純な文字列）を前提に作られています。選択式デッキという新しい概念を追加するにあたり、既存のコードを大きく書き換えるのではなく、**選択式デッキも`answer`フィールドだけは互換性のために律儀に埋めておく**ことで、既存の機能をそのまま動かし続けられるようにする、という実務的な工夫です。
- **`quiz_archive: True`（2026/08/21追加）**… このデッキが「クイズ過去問」由来であることを示す専用フラグです。[../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)・[../18_FlaskAPI_カードフォルダと並び順/00_フォルダのツリー構造と操作API.md](../18_FlaskAPI_カードフォルダと並び順/00_フォルダのツリー構造と操作API.md)で詳しく解説していますが、以前はこの性質を`folder_id`（フォルダの位置）だけで判定していました。ユーザーの要望で「デッキを他のフォルダへ移動できるようにしたいが、問題の編集はできないままにしたい」という仕様に変わったのに伴い、位置に依存しないこの専用フィールドへ切り替わっています。

```python
def _archive_room_if_needed(room):
    """
    ★ QUIZ_ROOMS_LOCK を保持している状態で呼び出すこと。
      ルームが終了(state=="ended")した瞬間に1回だけ呼ばれ、オリジナル4択
      （source=="manual"）だったクイズをCardMakerへアーカイブする。
      room["archived"] で二重登録を防ぐ（自然終了とホストの手動終了の
      両方から呼ばれ得るため）。
    """
    if room.get("source") != "manual" or room.get("archived"):
        return
    room["archived"] = True
    _archive_manual_quiz(room["title"], room["questions"], room["host_id"], room["host_nickname"])
```
- `room.get("source") != "manual"`… デッキから自動生成されたクイズ（`source == "deck"`）は、元々CardMaker上に存在するデッキそのものなので、改めてアーカイブする必要がありません。オリジナル問題（`source == "manual"`）の場合だけアーカイブします。
- `room.get("archived")`（既にアーカイブ済みフラグ）… コメントの通り、この関数は「自然に終了した場合」と「ホストが手動で終了させた場合」の**両方の経路から呼ばれる可能性がある**ため、`room["archived"] = True`を立ててから実際の保存処理を呼ぶことで、同じクイズ結果が誤って2つのデッキとして二重に保存されてしまうのを防いでいます。

## 7. ローカルAIによる誤答の強化（2026/08/26追加、`bot.py` 6058〜6063行・6336〜6510行）

「デッキから自動作成」で作られる4択は、上の`_pick_distractors`が**綴りの類似度だけ**で誤答を選んでいるため、綴りは似ていても意味的には全く見当違いな誤答が混ざることがありました。これをサーバー上に立てたローカルAI（Ollama、`qwen2.5-coder:7b`）に判断させ、「意味的にも紛らわしい」誤答へ差し替える仕組みを追加しています（[[local-coder-ai]]と同じOllamaコンテナを流用）。

### 7-1. 有効/無効の切り替えと設定

```python
OLLAMA_HOST = (os.environ.get("OLLAMA_HOST") or "").strip().rstrip("/")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL") or "qwen2.5-coder:7b"
QUIZ_AI_TIMEOUT_SEC = 45
QUIZ_AI_SHORTLIST_SIZE = 12
```
- `OLLAMA_HOST`が未設定（開発機など）なら、この機能は**一切動かず**、従来通り`_pick_distractors`だけで誤答が決まります。本番サーバーでは`.env`に`OLLAMA_HOST=http://ollama:11434`を設定し、あらかじめ`docker network connect pythonbot1istudy_default ollama`でBotコンテナ（ネットワーク`pythonbot1istudy_default`）とOllamaコンテナ（元々は`bridge`ネットワークのみ、ホストの`127.0.0.1:11434`にしか公開していない）を同じDockerネットワークに繋いでおく必要があります（ホスト側のポート公開範囲は変更していないので、外部への公開状況は変わりません）。

### 7-2. なぜ`quiz_create`をブロックしないのか

Web側の`quiz_create`はクライアントで12秒のタイムアウトが設定されています（`Quiz.js`の`apiPost('quiz_create', body, 12000)`）。一方ローカルAIはCPU動作でモデルも大きめ（7B）のため、応答に数秒〜数十秒かかることがあります（[[local-coder-ai]]のREADMEに実測あり）。そこで、AIへの問い合わせは`quiz_create`の応答後に**バックグラウンドスレッド**で行い、ルームが「ロビー」状態（ホストが「開始」を押す前、参加者を待っている間）の間だけ裏で差し替えます。

```python
if source == "deck" and OLLAMA_HOST:
    Thread(target=_ai_enhance_quiz_room_choices, args=(code, guild_id, deck_filenames), daemon=True).start()
```
- `source == "deck"`のときだけ（ホストが手入力した`manual`クイズは対象外＝人が作った問題にAIが手を入れる必要は無い）。

### 7-3. `_ai_enhance_quiz_room_choices`：安全な差し替えのタイミング制御

```python
def _ai_enhance_quiz_room_choices(code, guild_id, deck_filenames):
    with QUIZ_ROOMS_LOCK:
        room = QUIZ_ROOMS.get(code)
        if room is None or room["state"] != "lobby":
            return
        questions = room["questions"]
    try:
        improved = _ai_pick_quiz_distractors(questions, guild_id, deck_filenames)
    except Exception as e:
        print(f"[WARN] クイズ選択肢のAI強化に失敗しました（既存の選択肢のまま続行）: {e}")
        return
    if not improved:
        return
    with QUIZ_ROOMS_LOCK:
        room = QUIZ_ROOMS.get(code)
        if room is None or room["state"] != "lobby":
            return
        room["questions"] = improved
```
- AIへの問い合わせの**前後2回**、`room["state"] == "lobby"`であることを確認しています。もしホストがAIの応答を待たずに「開始」してしまい、出題（`question`）や発表（`reveal`）が始まっていたら、`questions`を書き換えるのを中断します。出題中に選択肢の中身が裏で変わってしまうと、既にクライアントへ配信済みの選択肢・正解番号と食い違う不整合が起きるためです。
- `try/except`で例外を丸ごと握りつぶし、失敗時はログを出すだけで既存の選択肢のまま進行します（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)で見た「アーカイブ失敗してもクイズ進行は止めない」のと同じ、ベストエフォートの考え方）。

### 7-4. `_ai_pick_quiz_distractors`：候補の絞り込み＋AIへの問い合わせ

```python
def _ai_pick_quiz_distractors(questions, guild_id, deck_filenames):
    answer_pool = _collect_deck_unique_answers(guild_id, deck_filenames)
    if len(answer_pool) < 2:
        return None
    items = []
    for i, q in enumerate(questions):
        correct = q["choices"][q["correct_index"]]
        candidates = _distractor_shortlist(
            correct, [a for a in answer_pool if a != correct], QUIZ_AI_SHORTLIST_SIZE
        )
        items.append({"i": i, "question": q["question"], "correct": correct, "candidates": candidates})
    result = _ollama_generate_json(_build_quiz_distractor_prompt(items))
    ...
```
- `_collect_deck_unique_answers`… `_build_deck_questions`と同じデッキ群から、もう一度「解答（answer）の重複無し一覧」を集め直します（クイズ作成時に選ばれた問題だけでなく、デッキ全体の解答を候補プールにするため）。
- `_distractor_shortlist`… `_pick_distractors`と全く同じ類似度スコア（綴りの類似度70%＋文字数の近さ30%）で並べ替え、**上位12件（`QUIZ_AI_SHORTLIST_SIZE`）だけ**をAIに渡します。全候補をそのまま渡すとプロンプトが膨らみ、CPU動作のAIの応答がさらに遅くなるための絞り込みです。★ 既存の類似度アルゴリズムを「AIに渡す候補の下ごしらえ」として再利用しているのがポイントで、綴りが似た候補の中から**意味的にも紛らわしいものをAIに選ばせる**、という2段構えになっています。
- 全問題ぶんをまとめて**1回のプロンプト**にして`_ollama_generate_json`に渡します（1問ずつ問い合わせると、CPU動作のAIでは往復回数分だけ遅くなるため）。

### 7-5. プロンプトの設計と「併用」方針（`_build_quiz_distractor_prompt`）

AIには「まずcandidates（綴りが似ている実在の解答）の中から意味的に紛らわしいものを3つ選べ、3つ集まらない場合に限り新しく考えて補ってよい」と指示しています。実在するデッキの解答を優先させているのは、AIに完全に自由作文させると事実として不自然・意味不明な誤答が混ざるリスクがあるためで、綴り類似度による絞り込み候補が十分ある通常のデッキでは、実質的に「候補の中から選ぶ」動作になります。出力は`{"questions": [{"i": 0, "distractors": [...]}, ...]}`というJSON1本のみを指示し、Ollama側の`"format": "json"`オプションと組み合わせて壊れた応答を減らしています。

### 7-6. `_sanitize_ai_distractors`：AI応答の検証

```python
def _sanitize_ai_distractors(distractors, correct: str):
    if not isinstance(distractors, list):
        return None
    max_len = max(60, len(correct) * 3)
    cleaned = []
    seen = {correct.strip()}
    for d in distractors:
        text = str(d or "").strip()
        if not text or len(text) > max_len or text in seen or find_bug_chars(text):
            continue
        seen.add(text)
        cleaned.append(text)
        if len(cleaned) == 3:
            break
    return cleaned if len(cleaned) == 3 else None
```
- AIの応答は「JSONとして正しい」ことしか保証されないため（Ollamaの`format: "json"`はJSONとして妥当であることまでしか強制しない）、中身は改めて検証します：文字列であること、空でないこと、正解と異なること（大文字小文字等はそのまま比較）、重複が無いこと、[../02_データ保存基盤/02_設定ファイルと不正文字チェック.md](../02_データ保存基盤/02_設定ファイルと不正文字チェック.md)の`find_bug_chars`で制御文字等を含まないこと、長さが正解の3倍または60文字を超えないこと（AIが長文で暴走するのを防ぐ）。
- **1問につき、有効な誤答がちょうど3件そろわなければその問題ごと`None`**（＝差し替えず、綴り類似度ベースの元の選択肢のまま）にします。「一部だけ差し替える」のような中途半端な救済はせず、AIの応答が信頼できる問題だけを丸ごと採用する設計です。
- `_ai_pick_quiz_distractors`の呼び出し元（`new_questions`構築ループ）でも、`pick`が無い（AIがその問題番号を返さなかった）・`_sanitize_ai_distractors`が`None`を返した、のどちらの場合も**その問題は`questions`の元の値のまま**にして`new_questions`に残します。全30問中1問だけAIの応答がおかしくても、他の29問には影響しません。

### 7-7. まとめ図

```
quiz_create（deck）
  ├─ _build_deck_questions()  … 従来通り即座に4択を生成（この処理自体は変更なし）
  ├─ ルームをstate="lobby"で作成し、即クライアントへ応答
  └─ [OLLAMA_HOST設定時のみ] バックグラウンドスレッドを起動
       └─ _ai_enhance_quiz_room_choices
            ├─ state=="lobby" 確認
            ├─ _ai_pick_quiz_distractors
            │    ├─ _collect_deck_unique_answers … 解答プールを再収集
            │    ├─ _distractor_shortlist × 問題数 … 綴り類似度で候補を12件に絞り込み
            │    ├─ _build_quiz_distractor_prompt → _ollama_generate_json … AIへ一括問い合わせ
            │    └─ _sanitize_ai_distractors × 問題数 … 応答を検証、問題ごとに採否判定
            └─ state=="lobby" を再確認してから room["questions"] を差し替え
```

---

次は、実際にルームを作成・参加・進行させるAPI群（`/quiz_create`から`/quiz_leave`まで）を解説します。 → [02_ルーム操作API.md](02_ルーム作成と過去問ランキング.md)
