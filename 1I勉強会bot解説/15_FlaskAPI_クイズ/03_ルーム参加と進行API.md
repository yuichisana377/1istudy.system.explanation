# ルーム一覧・参加・進行・回答・終了・退出API（`bot.py` 5323〜5552行）

対象：`bot.py`の`/quiz_list_rooms`〜`/quiz_leave`。これで「Flask API — みんなでクイズ」の章が終わります。

## 1. `/quiz_list_rooms`：ルーム一覧（5323〜5370行）

```python
@app.route("/quiz_list_rooms", methods=["GET"])
def quiz_list_rooms():
    """
    ★ 参加者向け：コード入力の代わりに、クイズルーム一覧をタイトルで選べるように
      するためのAPI。
    ・"lobby"だけでなく、"question"/"reveal"（進行中）のルームも「プレイ中」として
      一覧に出しっぱなしにする（終了するまで一覧から消えない）。
    ・"ended"（終了）になったルームだけ一覧から外す。
    """
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    guild_id = int(guild_id)
    student_id, err = require_member_session(session_token_from_request(), guild_id)
    if err:
        return err
    now = time.time()
    with QUIZ_ROOMS_LOCK:
        _quiz_gc_locked(now)
        rooms = []
        for room in QUIZ_ROOMS.values():
            if room["guild_id"] != guild_id or room["state"] == "ended":
                continue
            _quiz_autoadvance_locked(room, now)  # 一覧表示中もstateを最新化しておく
            if room["state"] == "ended":
                continue
            rooms.append({
                "code": room["code"], "title": room["title"], "host_nickname": room["host_nickname"],
                "player_count": len(room["players"]), "question_count": len(room["questions"]),
                "state": room["state"],
                "current_q": room["current_q"] if room["state"] != "lobby" else None,
                "allow_late_join": bool(room.get("allow_late_join")),
                "created_at": room["created_at"],
            })
    rooms.sort(key=lambda r: r["created_at"], reverse=True)
    for r in rooms:
        del r["created_at"]
    return jsonify({"ok": True, "rooms": rooms})
```
- コメントの通り、これは招待コードの手入力に頼らず、開催中のルームをタイトルで選んで参加できるようにするためのAPIです。「終了していない」全ルームを対象に、`_quiz_autoadvance_locked`で最新の状態に更新しながら一覧を組み立てます。**一覧表示のためにアクセスするだけで、内部的には状態遷移のチェックも行われる**という点が興味深いところです（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)の設計方針にあった「各APIリクエストのたびにその場評価する」の一例です）。
- `if room["state"] == "ended": continue`が2回登場します。1回目はループの入り口での早期チェック、2回目は`_quiz_autoadvance_locked`を呼んだ**後**の再チェックです。これは、`_quiz_autoadvance_locked`の呼び出しによって、それまで進行中だったルームがちょうどこの瞬間に`"ended"`へ遷移した可能性があるためです。一覧を取得しようとした瞬間にクイズが終わっていたら、それを一覧に含めない、という一貫性が保たれています。
- `created_at`でソートしてから、レスポンスからは`del r["created_at"]`で削除しています。ソートの基準としては必要でも、フロント側の表示自体には使わない内部情報なので、最終的なレスポンスには含めない、という配慮です。

## 2. `/quiz_join`：ルームへの参加（5372〜5409行）

```python
@app.route("/quiz_join", methods=["POST"])
def quiz_join():
    data, guild_id, student_id, nickname, err = _quiz_auth_from_json()
    if err:
        return err
    code = (data.get("code") or "").strip().upper()

    now = time.time()
    with QUIZ_ROOMS_LOCK:
        _quiz_gc_locked(now)
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        is_host = (student_id == room["host_id"])
        if not is_host and student_id not in room["players"]:
            if room["state"] == "ended":
                return jsonify({"ok": False, "error": "quiz_already_started"})
            if room["state"] != "lobby" and not room.get("allow_late_join"):
                return jsonify({"ok": False, "error": "quiz_already_started"})
            color, text_color = _quiz_palette_for(student_id)
            room["players"][student_id] = {
                "id": student_id, "nickname": nickname, "color": color, "text_color": text_color,
                "score": 0, "cur_answer": None, "cur_correct": None,
            }
        room["last_activity"] = now
        _quiz_autoadvance_locked(room, now)
        snap = _quiz_room_snapshot(room, student_id)
    notify_change(guild_id)
    return jsonify({"ok": True, "is_host": is_host, "room": snap, "server_now": int(now * 1000)})
```
- `is_host`（ホスト自身か）や、`student_id in room["players"]`（既に参加済みか）であれば、新しく追加する処理はスキップされます（同じ人が何度参加操作を行っても、二重に登録されることはありません）。
- コメントにある通り、新規参加を許可する条件は「`lobby`中は誰でも参加可能」「開始後（`question`/`reveal`）は、ホストが`allow_late_join`を許可していた場合のみ」「終了後（`ended`）はどちらの場合も参加不可」です。この3条件が、上のコードの2つの`if`文にそのまま対応しています。
- `notify_change(guild_id)`… **参加者が増えたことを、他の端末が次にポーリングするのを待たずに即座に通知します**。「遅延低減」というコメントが、この関数を含む複数のクイズAPIに繰り返し登場します。クイズはリアルタイム性が特に重視される機能なので、通常のポーリングに加えて、状態が変わるたびに積極的にSSE通知を送る設計になっています。

## 3. `/quiz_state`：状態のポーリング（5411〜5435行）

```python
@app.route("/quiz_state", methods=["GET"])
def quiz_state():
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    guild_id = int(guild_id)
    student_id, err = require_member_session(session_token_from_request(), guild_id)
    if err:
        return err
    code = (request.args.get("code") or "").strip().upper()

    now = time.time()
    with QUIZ_ROOMS_LOCK:
        _quiz_gc_locked(now)
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        is_host = (student_id == room["host_id"])
        if not is_host and student_id not in room["players"]:
            # 参加したことのない部屋の状態は覗けないようにする（未参加なら「見つからない」扱い）
            return jsonify({"ok": False, "error": "room_not_found"})
        room["last_activity"] = now
        _quiz_autoadvance_locked(room, now)
        snap = _quiz_room_snapshot(room, student_id)
    return jsonify({"ok": True, "is_host": is_host, "room": snap, "server_now": int(now * 1000)})
```
- これがフロント側から1秒ごとにポーリングされる、クイズの心臓部となるAPIです。
- **参加していないルームの状態を覗けないようにする**アクセス制御が興味深い点です。単に「権限がありません」ではなく、`"room_not_found"`（そもそも見つからない）という、参加済みルームと未参加ルームを区別できないエラーコードを返しています。これは、他人のルームコードを推測して覗き見しようとする行為に対し、「存在しない」のか「存在するが見られない」のかという情報すら与えない、という慎重な設計です（セキュリティの世界で「情報の漏洩を最小限にする」考え方の一例です）。

## 4. `/quiz_start`：クイズの開始（5437〜5472行）

```python
@app.route("/quiz_start", methods=["POST"])
def quiz_start():
    data, guild_id, student_id, nickname, err = _quiz_auth_from_json()
    if err:
        return err
    code = (data.get("code") or "").strip().upper()

    now = time.time()
    with QUIZ_ROOMS_LOCK:
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        if student_id != room["host_id"]:
            return jsonify({"ok": False, "error": "not_host"})
        if room["state"] != "lobby":
            return jsonify({"ok": False, "error": "quiz_already_started"})
        # ★ 変更：いきなり出題(question)にはせず、まず"countdown"
        #   （5,4,3,2,1のカウントダウン）→"intro"（「第1問」表示）を挟む。
        #   実際の制限時間のカウントは、intro表示が終わってから始まる
        #   （_quiz_autoadvance_locked参照）。
        room["state"] = "countdown"
        room["current_q"] = 0
        room["countdown_started_at"] = now
        room["intro_started_at"] = None
        room["question_started_at"] = None
        room["reveal_started_at"] = None
        room["first_correct_id"] = None
        room["first_correct_nickname"] = None
        for p in room["players"].values():
            p["cur_answer"] = None
            p["cur_correct"] = None
        room["last_activity"] = now
        snap = _quiz_room_snapshot(room, student_id)
    # ★ 遅延低減：カウントダウン開始を、他の端末のポーリング待ちなしで即座に知らせる
    notify_change(guild_id)
    return jsonify({"ok": True, "room": snap, "server_now": int(now * 1000)})
```
- `if student_id != room["host_id"]: return jsonify({"ok": False, "error": "not_host"})`… クイズを開始できるのはホストだけです。
- `if room["state"] != "lobby":`… 既に開始済みのルームを、二重に開始させないためのガードです。
- コメントの通り、いきなり出題（`"question"`）にはせず、まず`"countdown"`（カウントダウン）を経由します。実際の制限時間のカウントは、イントロ表示が終わってから始まる（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)の`_quiz_autoadvance_locked`で見た通り）ため、カウントダウンや「第N問」の表示時間は、回答時間には一切影響しません。

## 5. `/quiz_answer`：回答の送信（5474〜5515行）

```python
@app.route("/quiz_answer", methods=["POST"])
def quiz_answer():
    data, guild_id, student_id, nickname, err = _quiz_auth_from_json()
    if err:
        return err
    code = (data.get("code") or "").strip().upper()
    choice_index = data.get("choice_index")
    if not isinstance(choice_index, int) or isinstance(choice_index, bool) or not (0 <= choice_index < 4):
        return jsonify({"ok": False, "error": "invalid_choice"})

    now = time.time()
    with QUIZ_ROOMS_LOCK:
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        if room["state"] != "question":
            return jsonify({"ok": False, "error": "not_answerable"})
        player = room["players"].get(student_id)
        if player is None:
            return jsonify({"ok": False, "error": "not_in_room"})
        if player["cur_answer"] is not None:
            return jsonify({"ok": False, "error": "already_answered"})

        q = room["questions"][room["current_q"]]
        correct = (choice_index == q["correct_index"])
        player["cur_answer"] = choice_index
        player["cur_correct"] = correct
        if correct:
            points = QUIZ_ANSWER_BASE_POINTS
            if room["first_correct_id"] is None:
                room["first_correct_id"] = student_id
                room["first_correct_nickname"] = player["nickname"]
                points += QUIZ_FIRST_CORRECT_BONUS
            player["score"] += points
        room["last_activity"] = now
        _quiz_autoadvance_locked(room, now)
    notify_change(guild_id)
    return jsonify({"ok": True})
```
- `if room["state"] != "question":`… 出題中（`"question"`）以外の状態での回答は受け付けません。
- `if player["cur_answer"] is not None:`… **既に回答済みの場合、二重に回答を送れないようにするガード**です。これにより、同じ問題に対して1人が複数回答を送りつけて得点を不正に稼ぐことはできません。
- `if room["first_correct_id"] is None:`… その問題での「一番最初の正解者」だけにボーナス（`QUIZ_FIRST_CORRECT_BONUS`）が付きます。`first_correct_id`が既に埋まっていれば（誰か他の人が先に正解していれば）、このボーナス判定はスキップされます。
- `_quiz_autoadvance_locked(room, now)`… コメントの通り、この回答によって「全員が回答し終わった」場合、次のポーリングを待たずに、その場で正解発表（`reveal`）へ進めます。これも「体感の速さ」のための工夫で、[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)で見た`_quiz_scheduler_loop`（0.15秒ごとの自動チェック）と合わせて、**あらゆる場面でできるだけ素早く状態を進行させる**という一貫した設計方針が表れています。

## 6. `/quiz_end`：クイズの手動終了（5517〜5536行）

```python
@app.route("/quiz_end", methods=["POST"])
def quiz_end():
    data, guild_id, student_id, nickname, err = _quiz_auth_from_json()
    if err:
        return err
    code = (data.get("code") or "").strip().upper()

    now = time.time()
    with QUIZ_ROOMS_LOCK:
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        if student_id != room["host_id"]:
            return jsonify({"ok": False, "error": "not_host"})
        room["state"] = "ended"
        room["ended_at"] = now
        room["last_activity"] = now
        _archive_room_if_needed(room)
    notify_change(guild_id)
    return jsonify({"ok": True})
```
- ホストが途中でクイズを打ち切るためのAPIです。`_archive_room_if_needed`が呼ばれている点に注目してください。これは[01_ルーム状態のJSON化と問題の自動生成.md](01_ルーム状態のJSON化と問題の自動生成.md)で見た通り、自然に最後の問題まで終わった場合（`_quiz_autoadvance_locked`内から呼ばれる）と、このように途中でホストが終了させた場合の**両方の経路**から呼ばれる想定で、`room["archived"]`フラグによって二重アーカイブが防がれています。

## 7. `/quiz_leave`：ルームからの退出（5538〜5552行）

```python
@app.route("/quiz_leave", methods=["POST"])
def quiz_leave():
    data, guild_id, student_id, nickname, err = _quiz_auth_from_json()
    if err:
        return err
    code = (data.get("code") or "").strip().upper()

    with QUIZ_ROOMS_LOCK:
        room, err = _quiz_get_room_or_error(code)
        if err:
            return err
        room["players"].pop(student_id, None)
        room["last_activity"] = time.time()
    notify_change(guild_id)
    return jsonify({"ok": True})
```
- `room["players"].pop(student_id, None)`… 自分自身をプレイヤー一覧から取り除きます。`.pop(..., None)`により、既に何らかの理由で一覧に無くても（もう抜けていても）エラーにはなりません。
- ホスト自身がこのAPIを呼んでも、特別扱いはされずルームの`host_id`自体は変わらない点に注意してください（ホストが退出してもルーム自体は残り続けます。クイズを完全に終わらせたい場合は`/quiz_end`を使う必要があります）。

---

これで「Flask API — みんなでクイズ」の章が全て終わりました（約880行、`bot.py`最大のセクションでした）。次は、CardMakerで「作成中デッキ（まだ未公開）をみんなで共有表示する」ための仕組みを解説します。 → [../16_FlaskAPI_作成中デッキ共有/00_作成中デッキの一覧共有API.md](../16_FlaskAPI_作成中デッキ共有/00_作成中デッキの一覧共有API.md)
