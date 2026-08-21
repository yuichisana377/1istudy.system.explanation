# Flask API — ユーザー認証（前半）（`bot.py` 3035〜3144行）

対象：`bot.py`の「Flask API — ユーザー認証」セクション冒頭。ここから500行超にわたって、ログイン・Discord連携まわりのAPIが続きます。これまでの回で解説した[[../06_Discordアカウント連携/00_連携コードとOAuthステート.md]]の各種一時トークンを、実際にどう使っているかがここで分かります。

## 1. `/get_users`：ユーザー一覧の公開情報（3035〜3049行）

```python
@app.route("/get_users", methods=["GET"])
def get_users():
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    try:
        users = load_users(int(guild_id))
        public_users = [
            {"id": u.get("id"), "nickname": u.get("nickname"), "created_at": u.get("created_at")}
            for u in users
        ]
        return jsonify({"ok": True, "users": public_users})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- コメントにある通り、このAPIはログイン不要で誰でも呼べます。そのため`load_users`が返す生のデータをそのまま返さず、**公開してよい項目だけに絞ったリスト**（`id`・`nickname`・`created_at`）を新しく組み立てて返しています。デスクトップの設計資料に書かれている通り、`users_{guild_id}.json`には「旧パスワード方式の名残でハッシュ化されたパスワードのフィールドが残っている生徒データもある」ため、もし生のデータをそのまま返してしまうと、パスワードのハッシュ値のような本来公開すべきでない情報まで漏れてしまいます。**サーバー側で明示的にフィールドを絞り込んで新しい辞書を組み立てる**というこのやり方は、うっかり秘密の情報を漏らさないための、堅実な防御方法です。

## 2. `/generate_link_code`：連携コードの発行（3051〜3080行）

```python
@app.route("/generate_link_code", methods=["POST"])
def generate_link_code():
    data     = request.json or {}
    guild_id = data.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing fields"})

    guild_id   = int(guild_id)
    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    try:
        result = issue_link_code(guild_id, student_id)
        return jsonify({"ok": True, **result})
    except ValueError as e:
        msg = str(e)
        if msg.startswith("too_soon:"):
            return jsonify({"ok": False, "error": "too_soon", "retry_after_sec": int(msg.split(":", 1)[1])})
        return jsonify({"ok": False, "error": msg})
```
- [[../06_Discordアカウント連携/00_連携コードとOAuthステート.md]]で見た`issue_link_code`を呼び出すAPIです。ここが実際にDiscord連携コードを発行するエンドポイントで、[[../07_Discordコマンド/03_id連携とhelpコマンド.md]]の`/id連携`コマンドと対になっています。
- `student_id, err = require_member_session(...)`… ログイン確認は必須です。コメントにある通り「StudyLogにログイン済み（session_token検証済み）の本人だけが」呼べるようになっており、生徒IDだけを知っている第三者はこのAPIを呼べません（`session_token`が無いため）。
- `issue_link_code`が発生させる`ValueError`（30秒以内の連続発行）を`except ValueError as e:`で捕まえ、`msg.startswith("too_soon:")`で判定して、フロント側が「あと何秒待てばよいか」を表示できるよう`retry_after_sec`付きの専用エラーに変換しています。

## 3. `/discord_login_start`：Discordログインの開始（3083〜3109行）

```python
@app.route("/discord_login_start", methods=["GET"])
def discord_login_start():
    """
    ブラウザが直接GETするエンドポイント（ログイン前・session_token不要）。
    Discordの認可画面へリダイレクトする。
    """
    if not DISCORD_CLIENT_SECRET:
        return _oauth_result_page(False, "現在Discordログインは準備中です。時間をおいてもう一度お試しください。")

    guild_id = request.args.get("guild_id")
    if not guild_id:
        return _oauth_result_page(False, "不正なリクエストです。")

    guild_id = int(guild_id)
    state = issue_oauth_state(guild_id, None, purpose="login")
    authorize_url = "https://discord.com/oauth2/authorize?" + urlencode({
        "client_id":     DISCORD_CLIENT_ID,
        "redirect_uri":  DISCORD_OAUTH_REDIRECT_URI,
        "response_type": "code",
        "scope":         "identify",
        "state":         state,
        "prompt":        "consent",
    })
    return redirect(authorize_url)
```
- ここまでのAPIと違い、これは`fetch()`で呼ばれる純粋なJSON APIではなく、**ブラウザが直接アドレスバー的にアクセスする（ページ遷移として開く）**エンドポイントです。コメントの通り、ログイン前なので当然`session_token`は持っておらず不要です。
- `DISCORD_CLIENT_SECRET`が設定されていなければ（[[../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md]]で見た通り、これが無いとOAuth機能全体が無効化されます）、エラーページを返します。
- コメントにある通り、これは学籍番号+パスワードでのログインとは完全に別経路です。`issue_oauth_state(guild_id, None, purpose="login")`… `student_id`に`None`を渡していることに注目してください。この時点ではまだ「誰がログインしようとしているのか」は分かりません（それこそがこれから確認しようとしている当のことです）。`purpose="login"`は、後でコールバックが返ってきた際に「これは新規ログインの試みだった」と区別するための印です。
- `urlencode({...})`… Pythonの標準ライブラリで、辞書をURLのクエリ文字列（`key1=value1&key2=value2&...`の形式）に変換します。組み立てているのは、Discord公式のOAuth2認可エンドポイントのURLです。`response_type: "code"`（認可コードを受け取る方式）、`scope: "identify"`（[[../../1I勉強会webの裏側（設計の話）.md|デスクトップの設計資料]]で見た通り、最低限のプロフィール情報だけを要求するスコープ）、`prompt: "consent"`（毎回確認画面を出す設定）を指定しています。
- `return redirect(authorize_url)`… Flaskの`redirect`関数で、ブラウザに「このURL（Discordの認可画面）に移動してください」という指示（HTTPのリダイレクトレスポンス）を返します。

## 4. `/discord_oauth_start`：追加連携の開始（3112〜3144行）

```python
@app.route("/discord_oauth_start", methods=["POST"])
def discord_oauth_start():
    """
    ★ StudyLogにログイン済み（session_token検証済み）の本人だけが呼べる。
      返ってきた authorize_url にブラウザを移動させると、Discordの認可画面が
      表示され、許可すると /discord_oauth_callback に戻ってくる。
    """
    if not DISCORD_CLIENT_SECRET:
        return jsonify({"ok": False, "error": "oauth_not_configured"})

    data     = request.json or {}
    guild_id = data.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing fields"})

    guild_id   = int(guild_id)
    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    state = issue_oauth_state(guild_id, student_id, purpose="link")
    authorize_url = "https://discord.com/oauth2/authorize?" + urlencode({...})
    return jsonify({"ok": True, "authorize_url": authorize_url})
```
- `discord_login_start`と非常によく似ていますが、決定的な違いが2つあります。
  1. こちらは**普通のJSON API**（POST）です。`jsonify({"ok": True, "authorize_url": ...})`でURLを**文字列として返すだけ**で、`redirect`はしません。フロント側のJavaScriptが、このURLを受け取ってから`location.href = authorize_url`のような形で、あらためて自分でページ遷移させる想定です（GETで直接ブラウザに開かれる`discord_login_start`とは、呼ばれ方の前提が違います）。
  2. **既にログイン済みであることが前提**です（`require_member_session`で確認）。これは「まだログインしていない人がDiscordでログインする」ための`discord_login_start`とは違い、「学籍番号+パスワードなどで既にログイン済みの本人が、追加でDiscordアカウントを紐付ける（DM通知を受け取れるようにする）」ための入り口だからです。`issue_oauth_state(guild_id, student_id, purpose="link")`で、今度は`student_id`（既に分かっている本人の学籍番号）を`state`に埋め込んでおきます。この`purpose="link"`が、[[../06_Discordアカウント連携/00_連携コードとOAuthステート.md]]で見た`issue_oauth_state`の`purpose`引数の実際の使われ方です。

---

次は、OAuthコールバック（Discordの認可画面から戻ってきたときの処理）と、初回登録の完了処理を解説します。ここがユーザー認証の章で最も複雑な部分です。 → [[01_OAuthコールバックと初回登録.md]]
