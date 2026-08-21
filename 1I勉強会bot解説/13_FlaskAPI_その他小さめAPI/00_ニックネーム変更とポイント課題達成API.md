# ニックネーム変更・ポイント・課題達成API（`bot.py` 3893〜4090行）

対象：`bot.py`の`/change_nickname`・`/list_study_logs`・「Flask API — ポイント」「Flask API — 課題達成」セクション。

## 1. `/change_nickname`：ニックネーム変更（3893〜3927行）

コメントにある通り、ログインをDiscordのみに一本化したため、パスワード関連の機能（変更・確認コード送付）は廃止されており、ここに残っているのはニックネーム変更だけです。

```python
@app.route("/change_nickname", methods=["POST"])
def change_nickname():
    data     = request.json or {}
    guild_id = data.get("guild_id")
    nickname = (data.get("nickname") or "").strip()
    if not guild_id or not nickname:
        return jsonify({"ok": False, "error": "missing fields"})
    if len(nickname) > 16:
        return jsonify({"ok": False, "error": "nickname too long"})
    err = reject_if_bug_chars({"ニックネーム": nickname})
    if err:
        return err

    guild_id = int(guild_id)
    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    try:
        users  = load_users(guild_id)
        target = next((u for u in users if u.get("id") == student_id), None)
        if not target:
            return jsonify({"ok": False, "error": "user_not_found"})
        target["nickname"] = nickname
        save_users(guild_id, users)
        return jsonify({"ok": True, "nickname": nickname})
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
```
- これまでに見てきたパターンの組み合わせです。文字数制限（16文字）と不正文字チェックを行った後、`require_member_session`で本人確認をし、`users`（ユーザー一覧）の中から自分自身のエントリを探して`nickname`フィールドだけを書き換えます。
- `target["nickname"] = nickname`… `target`は`users`（読み込んだリストそのもの）の中の1要素への参照なので、これを書き換えれば`users`全体の中身も一緒に変わります。[../07_Discordコマンド/01_list_delete_editコマンド.md](../07_Discordコマンド/01_list_delete_editコマンド.md)の`/edit`コマンドで見た`found`と同じ、Pythonの参照渡しの性質を利用したパターンです。

## 2. `/list_study_logs`：勉強ログ一覧（3929〜3935行）

```python
@app.route("/list_study_logs", methods=["GET"])
def list_study_logs():
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    logs = load_study_logs(int(guild_id))
    return jsonify({"ok": True, "logs": logs})
```
- `study_logs_{guild_id}.json`の中身を、ログイン不要でそのまま全件返すだけのシンプルなAPIです。「みんなの記録」表示のために使われます。

## 3. `/get_points`：ポイント一覧（3940〜3947行）

```python
@app.route("/get_points", methods=["GET"])
def get_points():
    """全ユーザーのポイント合計を返す"""
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    pts = load_points(int(guild_id))
    return jsonify({"ok": True, "points": pts})
```
- `points_{guild_id}.json`（全生徒の累計ポイント）を、ログイン不要でそのまま返します。デスクトップの設計資料では「本人にしか表示しない設計」とありますが、これはあくまで**フロント側の画面上の見せ方の話**（自分の累計だけをヘッダーバッジに表示する）であり、API自体は全員分のデータをまとめて返している点に注意してください。実際のアクセス制御はフロント側の表示ロジックに委ねられています。

## 4. `/get_completed_tasks`：達成済み課題一覧（3949〜3976行）

```python
@app.route("/get_completed_tasks", methods=["GET"])
def get_completed_tasks():
    """
    student_id を指定: そのユーザーの達成済み課題リストを返す
    student_id を省略: 全ユーザー分を { student_id: [...] } の形でまとめて返す
                        （週間ランキングで全員の課題達成ポイントを集計するために使用）
    """
    guild_id   = request.args.get("guild_id")
    student_id = request.args.get("student_id")  # 省略可
    if not guild_id:
        return jsonify({"ok": False, "error": "missing params"})

    tasks = load_completed_tasks(int(guild_id))

    if student_id:
        raw = tasks.get(student_id, [])
        normalized = [_normalize_task_entry(e) for e in raw]
        return jsonify({"ok": True, "done": normalized})

    # student_id 省略 → 全員分をまとめて返す
    all_normalized = {
        sid: [_normalize_task_entry(e) for e in raw]
        for sid, raw in tasks.items()
    }
    return jsonify({"ok": True, "done": all_normalized})
```
- `student_id`パラメータの有無で、返す範囲が変わる、これまでにも登場した「同じAPIで2つの用途を兼ねる」パターンです。1人分のリストが欲しい場合と、全員分をまとめて集計したい場合（週間ランキング用）の両方に、この1つのAPIで対応しています。
- どちらの場合も、[../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md](../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md)で見た`_normalize_task_entry`を通して、新旧さまざまな形式のデータを統一してから返します。

## 5. `/complete_task`：課題の達成（3979〜4038行）

```python
@app.route("/complete_task", methods=["POST"])
def complete_task():
    data     = request.json or {}
    guild_id = int(data.get("guild_id"))
    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    task_id = data.get("task_id")
    if not task_id:
        return jsonify({"ok": False, "error": "missing fields"})

    user     = find_user(guild_id, student_id)
    nickname = user["nickname"] if user else None

    # --- ★ points はクライアントから受け取らず、サーバー側の予定データから引き直す ---
    #     （クライアントが任意の points を送っても無視される）
    points = find_task_points(guild_id, task_id)
    if points is None:
        return jsonify({"ok": False, "error": "task not found"})
```
- `points = find_task_points(guild_id, task_id)`… [../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md](../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md)で解説した通り、**ポイント数はクライアントから受け取らず、必ずサーバー側の予定データから引き直します**。コメントにある通り「クライアントが任意の`points`を送っても無視される」設計です。もし`task_id`に対応する予定が見つからなければ`None`が返り、そのまま「そんな課題は無い」というエラーになります。

```python
    done = load_completed_tasks(guild_id)
    if student_id not in done:
        done[student_id] = []

    normalized = [_normalize_task_entry(e) for e in done[student_id]]
    existing_ids = [e["id"] for e in normalized]

    if task_id not in existing_ids:
        normalized.append({
            "id":       task_id,
            "date":     datetime.now(JST).strftime("%Y-%m-%d"),
            "points":   points,
            "nickname": nickname,  # ★ ニックネームを保存
        })

    done[student_id] = normalized
    try:
        save_completed_tasks(guild_id, done)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})

    # --- ポイント加算 ---
    pts = load_points(guild_id)
    pts[student_id] = pts.get(student_id, 0) + points
    try:
        save_points(guild_id, pts)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})

    # ★ 2026/08/19：課題の達成状況はStudyLog.js上「自分のみ」にしか表示され
    #   ない個人的な記録（他の生徒には見えない）なので、運用ログには残さない
    #   （運用ログはログインなしでも閲覧できる＝実質公開の場のため）。
    return jsonify({"ok": True, "total": pts[student_id]})
```
- `if task_id not in existing_ids:`… **既に達成済みの課題を、もう一度重複して追加しない**ためのガードです。もし何らかの理由でこのAPIが同じ`task_id`で2回呼ばれても（例えば通信の不具合でリクエストが2回送られてしまった場合など）、記録もポイント加算も1回分しか反映されません。
- コメントにある通り、この処理の結果は`log_event`（運用ログ）には記録されません。理由は「課題の達成状況はStudyLog.js上『自分のみ』にしか表示されない個人的な記録（他の生徒には見えない）」であり、[../02_データ保存基盤/01_運用ログとdiff表示.md](../02_データ保存基盤/01_運用ログとdiff表示.md)で解説した「本人にのみ表示される情報は運用ログに残さない」という基準（運用ログ自体はログイン無しで誰でも見られる実質公開の場のため）に従っているからです。

## 6. `/uncomplete_task`：課題達成の取り消し（4041〜4089行）

```python
@app.route("/uncomplete_task", methods=["POST"])
def uncomplete_task():
    """
    /complete_task の逆操作。
    指定 student_id の達成済みリストから task_id を取り除き、
    そのタスクに付与されていたポイント分を累計ポイントから減算する。
    （ポイントが0未満にならないようガードする）
    """
    data     = request.json or {}
    guild_id = int(data.get("guild_id"))

    # --- ★ 本人確認：session_token から student_id を特定する（なりすまし防止）。
    #     課題の達成/取り消しも編集の一種なので、対象サーバーのメンバーであることも必須。 ---
    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    task_id = data.get("task_id")
    if not task_id:
        return jsonify({"ok": False, "error": "missing fields"})

    done = load_completed_tasks(guild_id)
    if student_id not in done:
        return jsonify({"ok": False, "error": "not completed"})

    normalized = [_normalize_task_entry(e) for e in done[student_id]]
    target = next((e for e in normalized if e["id"] == task_id), None)
    if target is None:
        return jsonify({"ok": False, "error": "task not found in completed list"})

    normalized = [e for e in normalized if e["id"] != task_id]
    done[student_id] = normalized
    try:
        save_completed_tasks(guild_id, done)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})

    # --- ポイント減算（0未満にはしない） ---
    removed_points = target.get("points") or 0
    pts = load_points(guild_id)
    pts[student_id] = max(0, pts.get(student_id, 0) - removed_points)
    try:
        save_points(guild_id, pts)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
```
- `/complete_task`の完全な逆操作です。重要なのは、**取り消すポイント数を、`find_task_points`で予定データから再取得するのではなく、達成済みリストに実際に保存されていた`target.get("points")`を使っている**点です。これには理由があります。もし予定自体が編集されてポイント数が変わっていたら（例えば5ptの課題が後で10ptに変更されていたら）、`find_task_points`は新しい10ptを返してしまいますが、実際にこの生徒に加算されていたのは元々の5ptです。**加算した時の実績値（当時の`points`）を使って減算する**ことで、加算・減算のつじつまが必ず合うようになっています。
- `max(0, ...)`… [../09_FlaskAPI_予定管理/02_学習ログ削除と予定の一覧編集削除.md](../09_FlaskAPI_予定管理/02_学習ログ削除と予定の一覧編集削除.md)の`/delete_study_log`と同じく、ポイントがマイナスにならないようにするガードです。
- こちらも`complete_task`と同じ理由で、運用ログには記録されません。

---

次は、CardMakerの大きなセクション（単語カードのデータ層と一覧取得・保存・削除API）に入ります。ここは`bot.py`全体の中でも最大級のボリュームです。 → [../14_FlaskAPI_CardMaker/00_カードデータ層とindex管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)
