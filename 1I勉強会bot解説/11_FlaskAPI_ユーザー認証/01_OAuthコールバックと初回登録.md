# OAuthコールバックと初回登録（`bot.py` 3147〜3421行）

対象：`bot.py`の`_oauth_result_page`・`/discord_oauth_callback`・`_handle_discord_link_callback`・`_guild_membership_status`・`_handle_discord_login_callback`・`/discord_reg_info`・`/discord_complete_registration`。「Discordでログイン」機能の中核部分で、この章で最も複雑な流れです。

## 1. 結果表示用の簡易HTML：`_oauth_result_page`（3147〜3164行）

```python
def _oauth_result_page(success: bool, message: str) -> str:
    """OAuthコールバック後にブラウザへ表示する簡易HTML（StudyLogへ自動で戻る）"""
    color = "#16a34a" if success else "#dc2626"
    icon  = "✓" if success else "✕"
    return f"""<!DOCTYPE html>
<html lang="ja"><head><meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Discord連携</title></head>
<body style="font-family:sans-serif;display:flex;align-items:center;justify-content:center;
             min-height:100vh;margin:0;background:#f1f5f9;">
  <div style="background:#fff;border-radius:16px;padding:32px;max-width:360px;width:90%;
              text-align:center;box-shadow:0 20px 50px rgba(0,0,0,.15);">
    <div style="font-size:40px;color:{color};margin-bottom:12px;">{icon}</div>
    <div style="font-size:15px;color:#334155;line-height:1.6;">{message}</div>
    <div style="font-size:12px;color:#94a3b8;margin-top:16px;">3秒後にStudyLogへ戻ります…</div>
  </div>
  <script>setTimeout(function() {{ location.href = "https://1istudyweb.pages.dev/StudyLog"; }}, 3000);</script>
</body></html>"""
```
- Discordの認可画面から戻ってきた直後、ブラウザに一瞬だけ表示される「連携が完了しました」のような結果画面を、Flaskが**HTML文字列をその場で組み立てて**返しています。これまでのAPIのように`jsonify`でJSONを返すのではなく、Flaskのルート関数はこのように文字列（HTMLそのもの）を返すこともできます。
- `f"""..."""`… 複数行にまたがる**f文字列**（`f"..."`の三重引用符版）です。`{color}`や`{message}`のように、変数の値がそのまま文字列の中に埋め込まれます。
- `<script>setTimeout(...)</script>`… 3秒後に自動的に`StudyLog`ページへ戻るよう、JavaScriptのタイマーが仕込まれています。これは、外部サービス（Discord）を経由するOAuthの流れでは、ページ遷移が何度か発生するため、その途中経過を利用者に見せつつ、最終的には元のアプリ画面へ自然に案内するための、よくあるUXパターンです。
- 変数`message`はサーバー側が用意した固定の文言のみで、利用者の入力が直接埋め込まれることはないため、ここでのHTML文字列の直接組み立てはXSSのリスクにはなりません（[../../1I勉強会web解説/06_Notice/00_HTML構造とその1_一覧と詳細表示.md](../../1I勉強会web解説/06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で見た「利用者が入力したテキストは`innerHTML`で組み立てない」という原則は、あくまで**利用者由来の値**を埋め込む場合の話です）。

## 2. 認可コード → アクセストークンの交換：`/discord_oauth_callback`（3167〜3229行）

```python
@app.route("/discord_oauth_callback", methods=["GET"])
def discord_oauth_callback():
    if not DISCORD_CLIENT_SECRET:
        return _oauth_result_page(False, "サーバー側でDiscord連携が設定されていません。...")
    if request.args.get("error"):
        return _oauth_result_page(False, "連携がキャンセルされました。")
    code  = request.args.get("code")
    state = request.args.get("state")
    if not code or not state:
        return _oauth_result_page(False, "不正なリクエストです。もう一度最初からお試しください。")

    entry = consume_oauth_state(state)
    if not entry:
        return _oauth_result_page(False, "リンクの有効期限が切れました。もう一度最初からお試しください。")

    guild_id = entry["guild_id"]
    purpose  = entry.get("purpose", "link")
```
- これが、Discordの認可画面で利用者が「許可」を押した後にブラウザが戻ってくる先です。DISCORD_OAUTH_REDIRECT_URIとして登録されているURLと一致します。
- `request.args.get("error")`… 利用者が認可画面で「拒否」を選んだ場合、Discordはこのクエリパラメータにエラー内容を入れて戻してきます。その場合はキャンセル扱いにします。
- `consume_oauth_state(state)`… [../06_Discordアカウント連携/00_連携コードとOAuthステート.md](../06_Discordアカウント連携/00_連携コードとOAuthステート.md)で見た、CSRF対策の1回使い切りstateをここで検証・消費します。これが無効（既に使われた、期限切れ、存在しない）であれば、それ以上は絶対に処理を進めません。ここで確認が取れて初めて、`guild_id`と`purpose`（発行時に埋め込んだ`"login"`または`"link"`）を信頼できる情報として取り出せます。

```python
    try:
        token_res = requests.post(
            "https://discord.com/api/oauth2/token",
            data={
                "client_id": DISCORD_CLIENT_ID, "client_secret": DISCORD_CLIENT_SECRET,
                "grant_type": "authorization_code", "code": code,
                "redirect_uri": DISCORD_OAUTH_REDIRECT_URI,
            },
            headers={"Content-Type": "application/x-www-form-urlencoded"},
            timeout=10,
        )
        token_res.raise_for_status()
        access_token = token_res.json()["access_token"]
    except Exception:
        return _oauth_result_page(False, "Discordとの認証に失敗しました。...")
```
- OAuth2の標準的な手順で、Discordから受け取った一時的な「認可コード」（`code`）を、Discord自身のAPIサーバーに送り返して、実際に使える「アクセストークン」に交換してもらいます。この交換には`DISCORD_CLIENT_SECRET`（秘密鍵）が必要で、これを持っているのはこのサーバーだけです。**ブラウザ側は`code`しか受け取らず、`access_token`への交換はサーバー間の通信でのみ行われる**という、OAuth2の安全性の根幹をなす手順です。
- `requests.post(...)`… Pythonの`requests`ライブラリで、外部のAPI（Discord）へHTTPリクエストを送っています。`timeout=10`（10秒でタイムアウト）と`raise_for_status()`（エラーのステータスコードなら例外を発生させる）で、外部サービスとの通信特有の失敗（応答が来ない、エラーが返る）に備えています。

```python
    try:
        me_res = requests.get(
            "https://discord.com/api/users/@me",
            headers={"Authorization": f"Bearer {access_token}"},
            timeout=10,
        )
        me_res.raise_for_status()
        me_json = me_res.json()
        discord_user_id  = int(me_json["id"])
        discord_username = me_json.get("global_name") or me_json.get("username") or ""
    except Exception:
        return _oauth_result_page(False, "Discordのユーザー情報取得に失敗しました。...")

    if purpose == "login":
        return _handle_discord_login_callback(guild_id, discord_user_id, discord_username)
    else:
        return _handle_discord_link_callback(guild_id, entry["student_id"], discord_user_id)
```
- 手に入れたアクセストークンを使って、`/users/@me`（Discord自身のAPI）に問い合わせ、「このアクセストークンの持ち主は誰か」を確認します。ここで初めて、**本当にそのDiscordアカウントの持ち主が今この操作をしている**ことが証明されます。
- `me_json.get("global_name") or me_json.get("username") or ""`… Discordのユーザー名の取得方法です。近年Discordはユーザー名の仕組みを変更しており、`global_name`（表示名）が優先され、無ければ`username`（従来のユーザー名）にフォールバックします。
- 最後に、`purpose`（先ほど`state`から取り出した目的）に応じて、`_handle_discord_login_callback`（新規ログイン）か`_handle_discord_link_callback`（既存アカウントへの追加連携）のどちらかに処理を振り分けます。

## 3. 追加連携の完了：`_handle_discord_link_callback`（3232〜3252行）

```python
def _handle_discord_link_callback(guild_id: int, student_id: str, discord_user_id: int) -> str:
    """既にログイン中の本人が、追加でDiscordを連携する（従来のフロー）。"""
    users = load_users(guild_id)
    matched = next((u for u in users if u["id"] == student_id), None)
    if not matched:
        return _oauth_result_page(False, "生徒データが見つかりませんでした。...")
    nickname = matched.get("nickname", student_id)

    try:
        links = load_discord_links(guild_id)
        links[student_id] = discord_user_id
        save_discord_links(guild_id, links)
    except DataWriteError:
        return _oauth_result_page(False, "連携の保存に失敗しました（サーバーエラー）。...")

    try:
        send_discord_dm(guild_id, student_id, "StudyLog", f"{student_id}の{nickname}さんの通知登録が完了しました。")
    except Exception:
        pass

    return _oauth_result_page(True, "Discordとの連携が完了しました！")
```
- 中身は[../07_Discordコマンド/03_id連携とhelpコマンド.md](../07_Discordコマンド/03_id連携とhelpコマンド.md)の`/id連携`コマンドとほぼ同じです。`discord_links`（DM通知用の対応表）に紐付けを保存し、確認のDMを送ります。異なるのは、こちらはコード入力ではなく、OAuth認可という別の経路で本人確認を済ませている点です。

## 4. メンバーシップ判定：`_guild_membership_status`（3255〜3268行）

```python
def _guild_membership_status(guild_id: int, discord_user_id: int) -> str:
    """
    戻り値: "member" / "not_member" / "unknown"（Bot未準備・サーバー未取得等で
    判定できない場合）。呼び出し側（_session_is_member等）は "unknown" も
    安全側に倒して「メンバーではない」扱いにする。
    """
    if not bot.is_ready():
        return "unknown"
    guild = bot.get_guild(guild_id)
    if not guild:
        return "unknown"
    return "member" if guild.get_member(discord_user_id) is not None else "not_member"
```
- これが、[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md](../05_ユーザーとセッション/01_セッショントークンと権限チェック.md)で見た`_session_is_member`が最終的に呼んでいる、メンバー判定の実体です。`guild.get_member(discord_user_id)`… Discord Botが持つ、そのサーバーのメンバー一覧のキャッシュ（[../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md](../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md)で見た`intents.members = True`によって取得・維持されているキャッシュ）から、該当ユーザーを探します。
- **3値**（`"member"`/`"not_member"`/`"unknown"`）を返す設計になっている点が重要です。単純な`True`/`False`の2値にしてしまうと、「Botがまだ起動中で本当は判定できない」状態と「確実にメンバーではないと分かった」状態を区別できません。コメントの通り、呼び出し側は`"unknown"`も安全側に倒して「メンバーではない」扱いにしますが、この関数自体は判定できない事実をそのまま正直に返すよう設計されています。

## 5. Discordログインの受付：`_handle_discord_login_callback`（3271〜3314行）

コメントに設計上の重要なポイントが3つまとめられています。
- 「全員に登録し直してもらう」方針のため、既存の`discord_links`（`/id連携`のコード方式で作られたDM通知用の紐付け）は**ログイン用途では一切信用しません**。`discord_login_links`（ログイン専用の別ファイル）に登録済みの場合のみ、そのままログインさせます。
- このエンドポイント自体はAPIドメイン（`python-bot-1istudy.onrender.com`のようなバックエンドのドメイン）で動いているため、ここで直接`localStorage`にセッションを書き込んでも、フロントエンド（`1istudyweb.pages.dev`）からは見えません（**ブラウザの`localStorage`はドメインごとに独立しており、異なるドメイン間では共有されない**という仕様のため）。そのため、セッション情報はURLのクエリパラメータとしてフロントエンドへ渡し、フロントエンド自身のJS（`Login.js`）がそちらのドメイン上で`localStorage`に保存します。
- Discordログイン自体は、対象サーバーに参加していないアカウントでも成功させます（完全ブロックはしません）。ただし参加していない場合は「制限付きアカウント」として扱われます（[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md](../05_ユーザーとセッション/01_セッショントークンと権限チェック.md)で見た通りです）。

```python
def _handle_discord_login_callback(guild_id: int, discord_user_id: int, discord_username: str) -> str:
    login_links = load_discord_login_links(guild_id)
    student_id = next((sid for sid, did in login_links.items() if int(did) == discord_user_id), None)

    if student_id:
        user = find_user(guild_id, student_id)
        if user:
            token = create_session(guild_id, student_id)
            nickname = user.get("nickname", student_id)
            _notify_new_login(guild_id, student_id, nickname)
            qs = urlencode({
                "discord_session_token": token,
                "student_id": student_id,
                "nickname": nickname,
            })
            return redirect(f"https://1istudyweb.pages.dev/Login?{qs}")

    reg_token = issue_discord_reg_token(guild_id, discord_user_id, discord_username)
    return redirect(f"https://1istudyweb.pages.dev/Login?discord_reg={reg_token}")
```
- `login_links.items()`（`{学籍番号: DiscordユーザーID}`）を1件ずつ確認し、**逆方向**（DiscordユーザーIDから学籍番号を探す）の検索をしています。`next((sid for sid, did in login_links.items() if int(did) == discord_user_id), None)`は、値の側から一致するキーを探すジェネレータ式です。
- 既に登録済み（`student_id`が見つかり、かつ対応する`user`データも存在する）なら、[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md](../05_ユーザーとセッション/01_セッショントークンと権限チェック.md)の`create_session`でセッショントークンを発行し、`_notify_new_login`（新規ログインの通知。後述）を呼んだ上で、**トークンをURLのクエリパラメータに載せて**フロントエンドへリダイレクトします。コメントの通り、これはドメインをまたいで安全に`localStorage`へ受け渡すための、意図的な設計です。
- 未登録（初回、またはユーザーデータが見つからない不整合）の場合は、[../06_Discordアカウント連携/00_連携コードとOAuthステート.md](../06_Discordアカウント連携/00_連携コードとOAuthステート.md)で見た`issue_discord_reg_token`で登録用トークンを発行し、Login画面の登録ステップへ案内します。

## 6. 登録トークンの事前確認：`/discord_reg_info`（3317〜3329行）

```python
@app.route("/discord_reg_info", methods=["GET"])
def discord_reg_info():
    entry = get_discord_reg_token(request.args.get("dtoken"))
    if not entry:
        return jsonify({"ok": False, "error": "reg_token_invalid"})
    return jsonify({"ok": True, "discord_username": entry.get("discord_username", "")})
```
- Login画面が、URLに付いてきた登録トークンがまだ有効かを確認するためのAPIです。コメントにある通り、生徒IDはまだ分からない段階なので、参考情報としてDiscordの表示名（ニックネーム欄の初期値の候補として使えます）だけを返します。`get_discord_reg_token`は消費しない（取得しても削除しない）バージョンなので、この確認自体はトークンを無駄にしません。

## 7. 初回登録の完了：`/discord_complete_registration`（3332〜3421行）

このAPIのコメントには、**セキュリティ上の既知のトレードオフが率直に書かれています**：「学籍番号を知っているだけで既存アカウントに連携できてしまうため、他人の学籍番号を知っている第三者がなりすましてログインできてしまうリスクがある（本人確認手段はDiscordの認可のみで、学籍番号の所有権は検証していない）。運用上リスクがあると感じた場合は、パスワード確認や通知の追加を検討すること」。これは、11人規模の内輪の運用という前提のもとで、利便性とのバランスを取った意図的な設計判断であり、「知らないうちにリスクを見落としていた」のではなく「リスクを認識した上で許容し、将来見直す余地をコメントとして残している」という、誠実なコードコメントの書き方の一例です。

```python
    data       = request.json or {}
    guild_id   = data.get("guild_id")
    dtoken     = data.get("dtoken")
    student_id = (data.get("student_id") or "").strip().upper()
    nickname   = (data.get("nickname") or "").strip()

    if not guild_id or not dtoken or not student_id:
        return jsonify({"ok": False, "error": "missing fields"})

    guild_id = int(guild_id)
    entry = get_discord_reg_token(dtoken)
    if not entry or entry["guild_id"] != guild_id:
        return jsonify({"ok": False, "error": "reg_token_invalid"})
    discord_user_id = entry["discord_user_id"]
```
- `dtoken`は、コメントにある通り「Discordの認可を実際に済ませていないと手に入らない」ものです。つまり、この時点で「このDiscordアカウントの持ち主である」ことは既に確認済みです。この関数がこれから確認するのは、その持ち主が「どの学籍番号の生徒として登録するか」という部分だけです。

```python
    try:
        users = load_users(guild_id)
        existing = next((u for u in users if u.get("id") == student_id), None)

        if existing:
            final_nickname = existing.get("nickname", student_id)
        else:
            if not nickname:
                return jsonify({"ok": False, "error": "nickname_required"})
            if len(nickname) > 16:
                return jsonify({"ok": False, "error": "nickname too long"})
            err = reject_if_bug_chars({"ニックネーム": nickname})
            if err:
                return err
            old_users_text = _users_text(users)
            users.append({
                "id": student_id, "nickname": nickname,
                "created_at": datetime.now(JST).strftime("%Y-%m-%d"),
            })
            save_users(guild_id, users)
            change = file_diff(f"users_{guild_id}.json", old_users_text, _users_text(users))
            log_event("user", "新しいユーザーが登録されました。", actor=nickname, detail=[change] if change else None)
            final_nickname = nickname
```
- `student_id`が既存の生徒データと一致すれば、コメントの通り「パスワード確認なしでそのまま紐付ける」（既存アカウントへの連携）。一致しなければ「新しい学籍番号」として新規登録し、その場合だけ`nickname`が必須（16文字以内、不正文字チェックあり）になります。
- 新規登録の場合、`users.append({...})`で追加するデータには、コメントの通り**パスワードのフィールドが一切ありません**（`password_hash`無し）。「Discordログイン専用アカウント」として、最初からパスワード方式を経由しない生徒データとして作られます。

```python
        login_links = load_discord_login_links(guild_id)
        login_links[student_id] = discord_user_id
        save_discord_login_links(guild_id, login_links)

        try:
            dm_links = load_discord_links(guild_id)
            dm_links[student_id] = discord_user_id
            save_discord_links(guild_id, dm_links)
        except DataWriteError:
            pass

        discard_discord_reg_token(dtoken)  # ★ 成功時のみ破棄

        token = create_session(guild_id, student_id)
        _notify_new_login(guild_id, student_id, final_nickname)
        return jsonify({
            "ok": True,
            "session_token": token,
            "student": {"id": student_id, "nickname": final_nickname},
        })
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_error: {e}"})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- `discord_login_links`（ログイン専用の紐付け）を保存すると同時に、**DM通知用の`discord_links`も合わせて更新**しています。コメントの通り「失敗してもログイン自体は成功扱い」なので、この部分だけ内側に別の`try`/`except DataWriteError: pass`が入れ子になっています。ログインという主目的が、付随的なDM連携の保存失敗に巻き込まれないようにする設計です。
- `discard_discord_reg_token(dtoken)`… コメントにある通り、**成功時のみ**破棄します。もしこの関数の途中で何かエラーが起きて失敗していたら、`dtoken`はまだ有効なままなので、生徒は同じトークンでもう一度やり直せます（パスワードの誤入力で失敗しても何度でもやり直せる、というのと同じ設計思想です）。
- 最後に、[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md](../05_ユーザーとセッション/01_セッショントークンと権限チェック.md)の`create_session`でセッショントークンを発行し、そのままJSONのレスポンスとして返します（`_handle_discord_login_callback`のように`redirect`のクエリに乗せるのではなく、こちらは元々POSTで呼ばれるJSON APIなので、素直にJSONの中に含めます）。

---

次は、`send_discord_dm`（DM送信の共通処理）と、新規ログイン通知、「問題を報告する」フォームを解説します。 → [02_DM送信と問題報告フォーム.md](02_DM送信と問題報告フォーム.md)
