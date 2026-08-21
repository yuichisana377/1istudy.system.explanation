# Discordアカウント連携の一時トークン群（`bot.py` 1310〜1490行）

対象：`bot.py`の「Discordアカウント連携」「アカウント連携コード」「Discord OAuth2の一時state」「Discordログイン専用の紐付け」「初回登録用の一時トークン」セクション。ここは**短時間だけ有効な使い捨てのコード・トークンを発行して、なりすましを防ぐ**という同じ考え方が、4つの異なる場面で繰り返し登場します。パターンを覚えると読みやすくなります。

## 1. DM通知用の紐付け：`discord_links`（1310〜1322行）

```python
def load_discord_links(guild_id: int) -> dict:
    data, _ = local_get(f"discord_links_{guild_id}.json")
    return data or {}

def save_discord_links(guild_id: int, links: dict, sha=None):
    if sha is None:
        _, sha = local_get(f"discord_links_{guild_id}.json")
    local_put(f"discord_links_{guild_id}.json", links, sha)
```
- `discord_links_{guild_id}.json`は、コメントにある通り`{"1I001": 123456789012345678, ...}`のような、**学籍番号 → DiscordユーザーIDの対応表**です。個別のDM通知（勉強タイマーの自動休憩通知や、新規ログイン通知など）を送る先を決めるために使われます。
- [[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md]]で見た「ログイン専用」の`discord_login_links_{guild_id}.json`とは**意図的に別ファイル**になっている点に注意してください（後述）。

## 2. アカウント連携コード：`/id連携`のなりすまし対策（1324〜1389行）

### 背景（コメントより）

以前は、Discord上で`/id連携 <生徒ID>`というコマンドを実行するだけで、誰でも好きな生徒IDを自分のDiscordアカウントに紐付けられてしまっていました。生徒IDは`1I001`のような推測しやすい形式である上、StudyLog自体にログインしていなくてもこのコマンドは実行できたため、他人になりすまして通知を横取りしたり、パスワード再設定の確認コードを自分宛に届かせて本人のアカウントごと乗っ取ることが理論上可能でした。

対策として、連携には**「StudyLog側で既にログイン（パスワード認証）済みであること」を証明する、短時間だけ有効なワンタイムコード**を要求するようになっています。流れは以下の通りです。
1. 生徒がStudyLogにログインした状態で連携コードを発行する（`/generate_link_code`）→ その生徒の`student_id`に紐付いたランダムな8桁コードが発行される（5分間だけ有効・1回使い切り）。
2. 生徒はDiscord上で`/id連携 <発行されたコード>`を実行する → コードが有効なら、そのコードに紐付いていた`student_id`と「今コマンドを実行したDiscordユーザー」を連携する。

コードを知らない第三者は、たとえ他人の生徒IDを知っていても連携できません（生徒IDそのものではなく、ログイン済みの本人しか手に入れられないコードが鍵になっているため）。

```python
LINK_CODE_TTL_SEC      = 5 * 60   # コードの有効期限：5分
LINK_CODE_COOLDOWN_SEC = 30       # 連続発行の連打防止（クールダウン）

LINK_CODES = {}             # code(str) -> {"guild_id", "student_id", "expires"}
LINK_CODE_LAST_ISSUED = {}  # f"{guild_id}:{student_id}" -> 直近の発行時刻（連打防止用）
```
- `LINK_CODES`と`LINK_CODE_LAST_ISSUED`は、[[../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md]]で見たレート制限と同じく、**プロセスのメモリ上だけ**に持たれるデータです（ファイルには保存されません）。5分やそこらしか使わない使い捨てのデータなので、ディスクに保存する必要が無く、プロセスが再起動されればどのみち全部消えて構わない、という性質のデータだからです。

```python
def _generate_link_code() -> str:
    alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"
    while True:
        code = "".join(secrets.choice(alphabet) for _ in range(8))
        if code not in LINK_CODES:
            return code
```
- `alphabet`… コメントの通り、Discord上で打ち込みやすいように、`0`と`O`、`1`と`I`のような**見分けにくい文字を意図的に除いた**、大文字と数字の組み合わせです。
- `secrets.choice(...)`… 暗号学的に安全な乱数生成器から1文字選びます（`random.choice`ではなく`secrets`モジュールを使っているのは、予測されにくい真にランダムな値が必要なためです）。これを8文字分繰り返して1つのコードを作ります。
- コメントの通り、この文字数・組み合わせなら、5分の有効期限内に総当たりで的中させるのは現実的ではありません（33種類の文字の8乗 ≒ 1.1兆通り）。
- `while True: ... if code not in LINK_CODES: return code`… 万が一、生成したコードが偶然すでに使われている別のコードと衝突していたら、重複しないコードが出るまでループし直します。

```python
def issue_link_code(guild_id: int, student_id: str) -> dict:
    key = f"{guild_id}:{student_id}"
    now = time.time()
    last = LINK_CODE_LAST_ISSUED.get(key)
    if last and (now - last < LINK_CODE_COOLDOWN_SEC):
        remain = int(LINK_CODE_COOLDOWN_SEC - (now - last)) + 1
        raise ValueError(f"too_soon:{remain}")

    for c in [c for c, v in LINK_CODES.items()
              if v["guild_id"] == guild_id and v["student_id"] == student_id]:
        del LINK_CODES[c]

    code = _generate_link_code()
    LINK_CODES[code] = {"guild_id": guild_id, "student_id": student_id, "expires": now + LINK_CODE_TTL_SEC}
    LINK_CODE_LAST_ISSUED[key] = now
    return {"code": code, "expires_in_sec": LINK_CODE_TTL_SEC}
```
- コメントの通り、この関数は「StudyLogにログイン済み（session_token検証済み）の本人だけが呼び出す想定」で、呼び出し元のFlask APIで先に`require_login_json`などによるログイン確認が済んでいることが前提になっています。
- 30秒（`LINK_CODE_COOLDOWN_SEC`）以内に連続で発行しようとすると、`ValueError`を`raise`（送出）して拒否します。これは「発行ボタンの連打」対策です。`f"too_soon:{remain}"`という文字列に、あと何秒待てばよいかの情報も含めています。
- 同じ生徒に対して、未使用の古いコードがまだ残っていれば、新しいコードを発行する前に無効化（削除）しておきます。これにより、1人につき常に最新の1つのコードしか有効にならず、混乱を防いでいます。
- `raise ValueError(...)`は例外を発生させる文なので、呼び出し元のFlask APIでは`try`/`except ValueError`のような形でこれを捕まえてエラーレスポンスに変換することになります（実際の呼び出し箇所は、後のFlask API章の`/generate_link_code`で解説します）。

```python
def consume_link_code(guild_id: int, code: str):
    entry = LINK_CODES.get(code)
    if not entry:
        return None
    if entry["guild_id"] != guild_id:
        return None
    if time.time() > entry["expires"]:
        del LINK_CODES[code]
        return None
    del LINK_CODES[code]  # ★ 1回使い切り（使い回し防止）
    return entry["student_id"]
```
- コードを検証し、有効なら紐付いていた`student_id`を返します。
- 注目すべきは、**成功する場合も失敗する場合（期限切れの場合）も、最終的にコードが`LINK_CODES`から削除される**という点です。有効期限内であっても成功したら即座に`del`しているのは、「1回使い切り」を保証するためです。1度連携に成功したコードを、誰かがもう一度使い回して別のDiscordアカウントを連携する、といった不正を防ぎます。

## 3. Discord OAuth2の一時state（1392〜1431行）

これは「Discordでログイン」ボタンから直接Discordの認可画面に飛ぶ方式のための、別の仕組みです。

```python
OAUTH_STATE_TTL_SEC = 5 * 60
OAUTH_STATES = {}  # state(str) -> {"guild_id", "student_id", "purpose", "expires"}

def issue_oauth_state(guild_id: int, student_id, purpose: str) -> str:
    state = secrets.token_urlsafe(24)
    OAUTH_STATES[state] = {
        "guild_id": guild_id,
        "student_id": student_id,
        "purpose": purpose,
        "expires": time.time() + OAUTH_STATE_TTL_SEC,
    }
    return state

def consume_oauth_state(state: str):
    entry = OAUTH_STATES.get(state)
    if not entry:
        return None
    if time.time() > entry["expires"]:
        del OAUTH_STATES[state]
        return None
    del OAUTH_STATES[state]
    return entry
```
- コメントにある通り、これは**CSRF（Cross-Site Request Forgery）対策**です。CSRFとは、悪意のある第三者が用意した罠のURLをユーザーに踏ませることで、ユーザー本人が意図していない操作をこっそり実行させてしまう攻撃手法です。
- 生徒がボタンを押してDiscordの認可画面に飛ぶ**直前**に、ランダムな`state`（状態を表す使い捨ての値）を発行し、`guild_id`・`student_id`（既にログイン中の場合）・`purpose`（後述）と紐付けて`OAUTH_STATES`に保存しておきます。
- Discordの認可画面から戻ってくる際（コールバック）、URLにこの`state`が一緒に返ってきます。サーバー側は`consume_oauth_state`でこの`state`を検証し、確かに自分が直前に発行したものであることを確認してから処理を進めます。これにより、「他人が用意した、Discordの認可を経ていない偽のコールバックURLを踏まされて、意図しない連携をさせられる」というリスクを防いでいます。
- `purpose`は`"link"`（既にログイン中の本人が、追加でDiscordを連携する）と`"login"`（まだログインしていない状態で、Discordそのものでログインしようとしている）の2種類です。`"login"`の場合は、この時点ではまだ誰なのか分からないため`student_id`は`None`のまま発行されます。
- `secrets.token_urlsafe(24)`… URLの中に安全に含められる、ランダムな文字列を生成します（Base64のURLセーフ版）。
- こちらも`consume_link_code`と同様、成功・失敗を問わず必ず`del`で1回使い切りにしています。

## 4. Discordログイン専用の紐付け（1434〜1451行）

```python
def load_discord_login_links(guild_id: int) -> dict:
    data, _ = local_get(f"discord_login_links_{guild_id}.json")
    return data or {}

def save_discord_login_links(guild_id: int, links: dict, sha=None):
    if sha is None:
        _, sha = local_get(f"discord_login_links_{guild_id}.json")
    local_put(f"discord_login_links_{guild_id}.json", links, sha)
```
- 冒頭で見た`discord_links`（DM通知用）とは**意図的に別ファイル**にしてある理由が、コメントに明記されています。「学籍番号+パスワードで既に登録している生徒」や「すでに`/id連携`（DM用）だけを済ませている生徒」であっても、**Discordログイン機能そのものは全員一度、新しい登録ステップ（既存アカウントならパスワードで本人確認）を通ってもらう**という運用方針です。そのため、`discord_login_links`は意図的に空の状態から始まり、既存の`discord_links`の中身を自動的には引き継がないようになっています。これは「DM通知だけ受け取りたいが学籍番号ログインのままにしたい」といった過去の運用との互換性を保つための設計判断です。

## 5. 初回登録用の一時トークン（1454〜1489行）

```python
DISCORD_REG_TOKEN_TTL_SEC = 10 * 60
DISCORD_REG_TOKENS = {}  # token(str) -> {"guild_id","discord_user_id","discord_username","expires"}

def issue_discord_reg_token(guild_id: int, discord_user_id: int, discord_username: str = "") -> str:
    token = secrets.token_urlsafe(24)
    DISCORD_REG_TOKENS[token] = {
        "guild_id": guild_id,
        "discord_user_id": discord_user_id,
        "discord_username": discord_username,
        "expires": time.time() + DISCORD_REG_TOKEN_TTL_SEC,
    }
    return token

def get_discord_reg_token(token):
    """有効なら中身のdictを返す（消費しない＝再試行可能）。無効・期限切れならNone。"""
    entry = DISCORD_REG_TOKENS.get(token)
    if not entry:
        return None
    if time.time() > entry["expires"]:
        del DISCORD_REG_TOKENS[token]
        return None
    return entry

def discard_discord_reg_token(token):
    DISCORD_REG_TOKENS.pop(token, None)
```
- コメントにある通り、「Discordでログイン」を初めて使う生徒は、OAuth認可の直後に一度だけ、学籍番号（＋既存アカウントの場合はパスワード）を入力する登録ステップを通ります。この`token`は「Discordの認可自体は既に済んでいる」ことの証明として、`Login.html`側の登録フォームからサーバーに送られてきます。
- ここまでの`consume_link_code`/`consume_oauth_state`と違い、`get_discord_reg_token`は**取得しただけでは消費（削除）しません**。コメントの通り、これはパスワードの誤入力などで登録が失敗しても、有効期限（10分）内であれば何度でもやり直せるようにするための設計です。
- 明示的に成功した場合だけ、呼び出し元が`discard_discord_reg_token(token)`を呼んでトークンを破棄します。`.pop(token, None)`は、辞書からキーを取り除く際に「そのキーが無くてもエラーにしない」（デフォルト値`None`を指定）書き方です。

---

これで、Discordアカウント連携まわりの4つの使い捨てトークンの仕組み（連携コード・OAuth state・ログイン専用紐付け・登録用トークン）を一通り見ました。実際にこれらを使う具体的なFlask API（`/generate_link_code`、`/discord_oauth_start`、`/discord_oauth_callback`など）は、後の「Flask API — ユーザー認証」の章でまとめて解説します。次は、Discordコマンドで使われる細々としたユーティリティ関数を見ていきます。 → [[../07_ユーティリティ関数群/00_チャンネル取得と日付パース.md]]
