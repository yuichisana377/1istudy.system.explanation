# Flask API 入門と `/channels`・`/add_schedule`（`bot.py` 2204〜2263行）

対象：`bot.py`の「Flask API — 予定管理」セクション冒頭。ここからいよいよ、`bot.py`の**半分以上を占める本体**であるFlask APIの章に入ります。

## Flask APIの章について

これまでのDiscordコマンドは「Discord上で`/`を打つ」ことで動きましたが、ここから解説するのは、Web側（`bot.1istudy.web`）の各ページがブラウザから`fetch(...)`で呼び出す、普通のHTTP APIです。[../../1I勉強会web解説/01_index_予定管理.md](../../1I勉強会web解説/01_index_予定管理.md)をはじめとする web解説シリーズの各ページで「サーバー側では〇〇というAPIが呼ばれ、△△が行われます」と説明してきた、その**サーバー側の実装そのもの**が、ここから続きます。

すでに解説した`require_login_json`・`reject_if_bug_chars`・`log_event`・`file_diff`・`notify_change`といった共通部品が、この章では絶え間なく登場します。読み方が分からなくなったら、[../02_データ保存基盤](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)・[../05_ユーザーとセッション](../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md)の該当ファイルに戻って確認してください。

## 1. `/channels`：科目チャンネル一覧の取得（2207〜2222行）

```python
@app.route("/channels", methods=["GET"])
def get_channels():
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    if not bot.is_ready():
        return jsonify({"ok": False, "error": "bot_not_ready", "message": "Botが起動中です。数秒後にもう一度お試しください。"})
    guild = bot.get_guild(int(guild_id))
    if not guild:
        return jsonify({"ok": False, "error": "guild not found"})
    channels = [{"id": str(ch.id), "name": ch.name} for ch in get_subject_channels(guild)]
    return jsonify({"ok": True, "channels": channels})
```
- Web側の予定登録フォームで、「科目」を選ぶプルダウンの中身を取得するためのAPIです。[../07_Discordコマンド/00_ユーティリティ関数と_addコマンド.md](../07_Discordコマンド/00_ユーティリティ関数と_addコマンド.md)で見た`get_subject_channels`を、そのままWeb向けにも再利用しています。
- `if not bot.is_ready():`… コメントにある通り、この行が無かった時代には、Botがまだ起動処理中（再接続中や起動直後）だと`bot.get_guild()`が常に`None`を返してしまい、フロント側には本当は一時的なだけなのに「サーバーが見つかりません（guild not found）」という誤解を招くエラーが表示されていました。`bot.is_ready()`で明示的に「準備中」の状態を区別し、`bot_not_ready`という別のエラーコードと、「数秒後にもう一度」という分かりやすいメッセージを返すように改善されています。
- `str(ch.id)`… DiscordのIDは非常に大きな整数（64ビット）です。JavaScriptの数値型は「安全に扱える最大の整数」に上限があり、Discordの大きなIDをそのまま数値としてJSONに含めると、ブラウザ側で最後の数桁が丸められて壊れてしまう可能性があります。これを避けるため、サーバー側であらかじめ文字列に変換してから返しています。

## 2. `/add_schedule`：予定の追加（2224〜2263行）

```python
@app.route("/add_schedule", methods=["POST"])
def add_schedule():
    data     = request.json
    guild_id, _student_id, nickname, err = require_login_json(data)  # ★ 変更にはログイン必須
    if err:
        return err
    date     = data.get("date")
    subject  = data.get("subject")
    category = data.get("category")
    content  = data.get("content")
    points   = data.get("points")

    if not all([date, subject, category, content]):
        return jsonify({"ok": False, "error": "missing fields"})

    err = reject_if_bug_chars({"科目": subject, "カテゴリ": category, "内容": content})
    if err:
        return err

    if points is not None:
        try:
            points = int(points)
        except (TypeError, ValueError):
            return jsonify({"ok": False, "error": "invalid points"})
```
- ここまでの流れは、これまでの回で見た共通処理そのままです。`require_login_json`でログイン確認（本物のニックネームを取得）→`all([...])`で必須項目が全部揃っているか確認（Pythonの`all()`は、渡されたリストの要素が全て真の値であれば`True`）→`reject_if_bug_chars`で不正文字チェック、という順番です。
- `points`（ポイント）は数値として送られてくるべきものですが、万一文字列や不正な形式で送られてきた場合に備えて、`int(points)`への変換を`try`/`except`で囲み、失敗すればエラーを返します。

```python
    guild = bot.get_guild(int(guild_id))
    future = asyncio.run_coroutine_threadsafe(
        add_plan_internal(int(guild_id), subject, date, category, content, points),
        bot.loop
    )
    ok, msg, detail = future.result(timeout=30)
```
- **ここがこの関数で最も重要な部分です**。[../07_Discordコマンド/00_ユーティリティ関数と_addコマンド.md](../07_Discordコマンド/00_ユーティリティ関数と_addコマンド.md)で見た`add_plan_internal`は`async def`で定義された非同期関数ですが、Flask側のこの関数（`add_schedule`）自体は**普通の同期関数**です。[../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md](../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md)で見た通り、Flask（Webサーバー）とDiscord Bot（非同期のイベントループ）は、同じプロセスの中で別々の仕組みとして同時に動いています。同期的なコードから、別のスレッドで動いている非同期のイベントループ上の関数を呼び出すには、特別な橋渡しが必要です。
- `asyncio.run_coroutine_threadsafe(コルーチン, ループ)`… これがその橋渡しです。「`add_plan_internal(...)`という非同期処理（コルーチン）を、`bot.loop`（Discord Bot側が動いているイベントループ）の上で実行してください」と、**スレッドをまたいで安全に**依頼します。戻り値の`future`は、この非同期処理の結果を後で受け取るための「引換券」のようなオブジェクトです。
- `future.result(timeout=30)`… その`future`に対して、実際の処理が終わって結果が返ってくるのを、最大30秒待ちます。Flask側の処理はここで一旦停止（ブロック）し、Discord Bot側での処理が終わるのを待つことになります。もし30秒経っても終わらなければ、タイムアウトの例外が発生します（万一Discord Bot側が固まっていても、Flask側までいつまでも応答不能になり続けるのを防ぐ安全弁です）。
- こうして、Web側から送られてきた予定追加のリクエストが、**Discordコマンドの`/add`と全く同じ`add_plan_internal`を経由して**処理されます。予定の追加ロジックが1箇所にまとまっているため、Discord経由でもWeb経由でも、常に同じルール（過去日付は登録不可、など）が適用されます。

```python
    if ok:
        log_event("schedule", f"予定「{subject}」を追加しました（{date}）。", actor=nickname, detail=detail)
    if ok and guild:
        target_channel = get_subject_channel_by_name(guild, subject)
        if target_channel:
            asyncio.run_coroutine_threadsafe(
                target_channel.send(msg), bot.loop
            ).result(timeout=10)
    return jsonify({"ok": ok, "message": msg})
```
- 成功した場合、[../02_データ保存基盤/01_運用ログとdiff表示.md](../02_データ保存基盤/01_運用ログとdiff表示.md)で見た`log_event`で運用ログに記録します。`detail`（`add_plan_internal`が返した、`file_diff`による差分情報）をそのまま渡しています。`actor=nickname`… ここでも、クライアントの自己申告ではなく、`require_login_json`で取得した本物のニックネームが使われています。
- さらに、`add_plan_internal`自体はDiscordへメッセージを送信する処理を含んでいない（メッセージの文面`msg`を作って返すだけ）ため、Web側から呼ばれた場合はこの`add_schedule`の中で改めて`target_channel.send(msg)`を`run_coroutine_threadsafe`経由で呼び、実際にDiscordチャンネルへの投稿を行っています。これにより、Web側から予定を追加しても、対応する科目のDiscordチャンネルにはこれまで通り通知メッセージが投稿されます（Discordコマンド経由での追加と、見た目上の結果が同じになるようにするためです）。

---

次は、`add_study_log`（学習ログの記録API）を解説します。このAPIは、なりすまし防止・連打対策・記録の水増し防止など、多くの安全対策が詰め込まれた、この章で最も複雑なエンドポイントの1つです。 → [01_学習ログ記録APIのなりすまし対策と連打対策.md](01_学習ログ記録APIのなりすまし対策と連打対策.md)
