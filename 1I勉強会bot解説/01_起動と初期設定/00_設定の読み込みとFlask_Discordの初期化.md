# 設定の読み込みとFlask・Discordの初期化（`bot.py` 1〜268行）

対象：`bot.py`の先頭部分。用語は[[../../1I勉強会web解説/01_index_予定管理.md]]の「0. ミニ用語辞典」も参照してください。

## 1. importと環境変数（1〜51行）

```python
import discord
from discord import app_commands
from discord.ext import commands
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from flask import Flask, request, jsonify, make_response, redirect, Response
from flask_cors import CORS
...
```
- 冒頭で使うライブラリをまとめて読み込んでいます。大きく分けると、①`discord`系（Discord Botを作るためのライブラリ）、②`flask`系（Web APIサーバーを作るためのライブラリ）、③`apscheduler`（「毎日この時刻に実行」のような定期処理を仕込むライブラリ）、④`hashlib`・`hmac`・`secrets`（暗号学的な処理。パスワードやトークンの安全な取り扱いに使う）などです。**1つのPythonプロセスの中で、Discord BotとWebサーバーの両方が同時に動いている**のがこのファイルの特徴です。

```python
DATA_DIR = os.getenv("DATA_DIR", os.path.join(os.path.dirname(os.path.abspath(__file__)), "data"))
os.makedirs(DATA_DIR, exist_ok=True)
```
- `os.getenv("DATA_DIR", ...)`は「環境変数`DATA_DIR`があればその値を、無ければ`,`の後の値（このスクリプトと同じ場所の`data`フォルダ）を使う」という意味です。環境変数とは、プログラムの外側（サーバーのOS側）から設定できる値のことで、コードを書き換えずに動作を変えられるようにするための仕組みです。全てのデータ（予定・カード・お知らせなど）は、このディレクトリの下にJSONファイルとして置かれます。
- コメントにある通り、以前はGitHub上にデータを保存する方式でしたが、今はUbuntuサーバーの常時稼働に合わせて、サーバー自身のディスクに直接保存する方式に変わっています。

```python
TOKEN               = os.getenv("TOKEN")
SUBJECT_CATEGORY_ID = os.getenv("SUBJECT_CATEGORY_ID")
SUBJECT_CATEGORY    = os.getenv("SUBJECT_CATEGORY")
JST = timezone("Asia/Tokyo")
```
- `TOKEN`はDiscord Botとしてログインするための秘密のトークン（合言葉）です。これも環境変数から読み込むことで、コードそのものやGitHubには一切書かれないようになっています（漏れると誰でもこのBotになりすませてしまう非常に重要な情報です）。
- `JST`は日本時間のタイムゾーンオブジェクトです。サーバーが海外のデータセンターで動いていても、日付や時刻の計算がすべて日本時間基準になるように、随所でこの`JST`を使って現在時刻を取得しています。

## 2. Discord OAuth2の設定（53〜82行）

```python
DISCORD_CLIENT_ID     = os.getenv("DISCORD_CLIENT_ID", "1515358957542047975")
DISCORD_CLIENT_SECRET = os.getenv("DISCORD_CLIENT_SECRET")
DISCORD_OAUTH_REDIRECT_URI = os.getenv(
    "DISCORD_OAUTH_REDIRECT_URI",
    "https://chiro-ubuntuserver.tail1130ba.ts.net/discord_oauth_callback"
)
if not DISCORD_CLIENT_SECRET:
    print("[WARN] 環境変数 DISCORD_CLIENT_SECRET が未設定です。...")
```
- 「Discordでログイン」ボタンを実現するための3点セットです。OAuth2とは、生徒が自分のDiscordのパスワードをこのサイトに直接教えることなく、「私はこのDiscordアカウントの持ち主です」とDiscord自身に証明してもらう仕組みです。
- `DISCORD_CLIENT_ID`はこのアプリを識別するID（公開情報。ログイン画面のURLを見れば誰でも分かる）。`DISCORD_CLIENT_SECRET`は逆にこのアプリだけが知っている秘密のパスワードのようなもので、環境変数に無ければ警告を出し、OAuthログイン機能そのものを無効化します（コードにハードコード＝直接書き込む、をしないことで、GitHub等に漏れるリスクを避けています）。
- `DISCORD_OAUTH_REDIRECT_URI`は、Discordでの認可が終わったあとにブラウザが戻ってくる先のURLです。Discord Developer Portal（Discord公式の開発者用管理画面）に事前登録したURLと1文字でも違うと拒否される、という制約があります。

```python
EMOJI_COMMUTER = "🚃"
EMOJI_DORM     = "🏠"
REPORT_PROBLEM_ADMIN_STUDENT_ID = os.getenv("REPORT_PROBLEM_ADMIN_STUDENT_ID", "32618")
scheduler = AsyncIOScheduler(timezone=JST)
```
- `EMOJI_COMMUTER`/`EMOJI_DORM`は、後述する「通生/寮生振り分けパネル」でリアクションとして使う絵文字です。
- `REPORT_PROBLEM_ADMIN_STUDENT_ID`は、サイトのJSが全く動かない緊急時用の「問題を報告する」フォームの送り先（学籍番号）です。
- `scheduler`は、「毎朝7時に予定を通知する」のような定期実行を管理するオブジェクトです（実際の登録は別の場所で行われます）。

## 3. Flaskアプリの初期化とヘッダー処理（97〜153行）

```python
app = Flask("")
CORS(app, resources={r"/*": {"origins": "*"}}, supports_credentials=False)
```
- `app`が、これ以降`@app.route(...)`で定義していくWeb APIサーバー本体です。
- `CORS(...)`は「オリジンをまたいだリクエスト」を許可する設定です。このBotのAPIサーバーと、Web側（`bot.1istudy.web`、Cloudflare Pages上で動く別ドメイン）はドメインが違うため、ブラウザは標準では「他ドメインへのfetchはブロックする」という安全策（同一オリジンポリシー）を取ります。`origins: "*"`で「どのドメインからのアクセスも許可する」と明示的に許可しています。

```python
NO_CACHE_PATHS = {
    "/list_cards", "/get_card_set", "/list_folders", "/list_order",
    "/channels", "/list_in_progress", "/timer_state", "/get_study_data",
    "/deck_understanding", "/list_notices", "/quiz_state",
}

@app.after_request
def add_no_cache_headers(response):
    if request.method == "GET" and request.path in NO_CACHE_PATHS:
        response.headers["Cache-Control"] = "no-store, no-cache, must-revalidate, max-age=0"
        response.headers["Pragma"] = "no-cache"
        response.headers["Expires"] = "0"
    return response
```
- `@app.after_request`は「どのAPIのレスポンスを返す直前にも、必ずこの関数を通す」という指定です（Flaskのフック機能）。
- `NO_CACHE_PATHS`に列挙されたGET系APIのレスポンスには、明示的に「キャッシュしないで」というヘッダーを付けます。コメントにある通り、これはブラウザ（特にChrome）がAPIの応答を勝手にディスクキャッシュしてしまい、新しく作ったデッキやフォルダが一覧にすぐ反映されないという不具合への対策です。フロント側（`Cardmaker.js`等）でも`fetch`に`cache: 'no-store'`を付けていますが、サーバー側でも二重に対策することで、途中の中継サーバー（プロキシ）等でキャッシュされるケースも防いでいます。

```python
@app.after_request
def add_security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    return response
```
- 一般的なWebアプリの基本的なセキュリティヘッダーです。
- `X-Content-Type-Options: nosniff`… ブラウザが「このファイルは本当はJavaScriptっぽいから実行しちゃおう」のように、サーバーが指定したContent-Type（ファイルの種類の申告）を無視して中身を勝手に推測（スニッフィング）するのを禁止します。
- `X-Frame-Options: DENY`… 他サイトが`<iframe>`でこのAPIの応答を埋め込むことを禁止します（クリックジャッキングという攻撃手法への対策。JSON APIなので実害は薄いですが、定番の対策として入れてあります）。
- `Referrer-Policy: strict-origin-when-cross-origin`… 別サイトへリンクで移動する際、移動元のURL全体（トークンなどが含まれている可能性がある）をそのままreferrer情報として渡さないようにする設定です。

```python
@app.before_request
def handle_preflight():
    if request.method == "OPTIONS":
        res = make_response()
        res.headers["Access-Control-Allow-Origin"]  = "*"
        res.headers["Access-Control-Allow-Methods"] = "GET, POST, OPTIONS"
        res.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
        return res, 200
```
- ブラウザは、`Content-Type: application/json`のようなヘッダー付きでPOSTする前に、「このリクエストを送っても大丈夫か」を`OPTIONS`メソッドで事前確認すること（プリフライトリクエスト）があります。ここではその確認に対して「大丈夫です、許可します」と即座に200 OKを返しています。

## 4. リアルタイム更新通知（SSE）の仕組み（154〜229行）

これは[[../../1I勉強会web解説/02_Cardmaker/00_HTML構造とページ全体像.md]]など、フロント側の各解説で何度も登場した「リアルタイム更新」の**サーバー側の実装**です。

```python
EVENT_SUBSCRIBERS = {}  # guild_id(int) or None(全guild共有分) -> set[queue.Queue]
EVENT_SUBSCRIBERS_LOCK = Lock()
EVENT_KEEPALIVE_SEC = 20
```
- `EVENT_SUBSCRIBERS`は「今つながっているブラウザタブ」の一覧です。キーがguild_id（DiscordサーバーのID。CardMakerのようにguildをまたいで共有されるデータは`None`キーに入る）、値がそのguildを見ているタブそれぞれに対応する`Queue`（順番に物を出し入れできる箱）の集合です。
- `Lock()`は排他制御用のロックです。複数のリクエストが同時に`EVENT_SUBSCRIBERS`を読み書きしようとして中身が壊れることを防ぎます（Pythonの辞書やセットへの操作自体は基本的にスレッドセーフではないため、複数スレッドから同時に書き換える可能性がある場面では明示的にロックする必要があります）。

```python
def notify_change(guild_id=None):
    with EVENT_SUBSCRIBERS_LOCK:
        targets = list(EVENT_SUBSCRIBERS.get(guild_id, ()))
    for q in targets:
        try:
            q.put_nowait(1)
        except Exception:
            pass
```
- 「データが変わったよ」を今つながっている全タブに知らせる関数です。データを保存する処理（`upload_notice`や`save_cards`など）の最後で、このシリーズを通して何度も`notify_change(...)`が呼ばれることになります。
- `q.put_nowait(1)`… 各タブに対応するQueueに、`1`という値（中身自体に意味は無く「何か入った」という合図）を入れます。`put_nowait`は「入れられなければ（満杯なら）例外を出してすぐ諦める」バージョンで、通知が多少詰まっても処理全体が止まらないようにしています。

```python
@app.route("/events", methods=["GET"])
def sse_events():
    guild_id = request.args.get("guild_id")
    guild_id = int(guild_id) if guild_id else None

    q = queue.Queue()
    with EVENT_SUBSCRIBERS_LOCK:
        EVENT_SUBSCRIBERS.setdefault(guild_id, set()).add(q)
        if guild_id is not None:
            EVENT_SUBSCRIBERS.setdefault(None, set()).add(q)

    def gen():
        try:
            while True:
                try:
                    q.get(timeout=EVENT_KEEPALIVE_SEC)
                    yield "data: changed\n\n"
                except queue.Empty:
                    yield ": keep-alive\n\n"
        finally:
            with EVENT_SUBSCRIBERS_LOCK:
                EVENT_SUBSCRIBERS.get(guild_id, set()).discard(q)
                if guild_id is not None:
                    EVENT_SUBSCRIBERS.get(None, set()).discard(q)

    return Response(gen(), mimetype="text/event-stream", headers={
        "Cache-Control": "no-cache",
        "X-Accel-Buffering": "no",
    })
```
- ここが**SSE（Server-Sent Events）**の本体です。SSEとは、普通のHTTPリクエストのように1回で終わらず、接続を張りっぱなしにして、サーバー側から一方的にどんどんデータを送り続けられる仕組みです（LINEやDiscordのように、サーバーから「新着通知」が届く仕組みの基礎技術の1つです）。
- 自分専用のQueue（`q`）を1つ作り、`EVENT_SUBSCRIBERS`に登録します。guild_id付きで接続した場合は、そのguild専用のキー**と**共有分（`None`）の両方に同じQueueを登録します（コメントにある通り、CardMakerのように guild をまたいで共有されるデータの更新も、guild_id付きで繋いだページで拾えるようにするためです）。
- `gen()`はPythonの**ジェネレータ関数**です。`yield`で値を返すたびに一旦停止し、次に呼ばれたときに続きから再開する、という特殊な関数で、「終わりのない無限のストリーム」を表現するのに向いています。
  - `q.get(timeout=EVENT_KEEPALIVE_SEC)`… 自分のQueueに何か入るのを、最大20秒待ちます。`notify_change()`が呼ばれて何か入れば`"data: changed\n\n"`という行を送り出します（これがSSEの1メッセージのフォーマットです）。中身を受け取ったブラウザ側は「何か変わった」とだけ知り、実際のデータは改めて普段通りのGET APIで取りに行きます。
  - 20秒間何も無ければ`queue.Empty`例外が発生し、代わりに`": keep-alive\n\n"`（`:`で始まる行はSSEの仕様上「コメント行」＝データ更新扱いにはならない）を送ります。これは、間に入っているプロキシ（Tailscale等）が「20秒以上通信が無い接続」を勝手に切ってしまうのを防ぐためのダミー送信です。
  - `finally`節… タブが閉じられたりページがリロードされたりして接続が切れると、このジェネレータは終了し、`finally`で自動的に`EVENT_SUBSCRIBERS`から自分のQueueを取り除きます（購読解除）。これにより、閉じたタブの分がいつまでも登録されっぱなしになる「メモリリーク」を防いでいます。
- コメントにある通り、この仕組みが万一切れていても（サーバーのスリープ復帰直後など）画面が永久に古いままにならないよう、フロント側では60秒おきの緩いポーリングも保険として残されています（結果整合性：多少通知が遅れても、最終的には必ず正しい状態に追いつく設計）。

## 5. トップページとFlaskスレッドの起動（230〜241行）

```python
@app.route("/")
def home():
    return "I'm alive"

def run_flask():
    port = int(os.environ.get("PORT", 10000))
    app.run(host="0.0.0.0", port=port, use_reloader=False, threaded=True)

def keep_alive():
    t = Thread(target=run_flask, daemon=True)
    t.start()
    print("[INFO] Flask thread started")
```
- `home()`は、ルートURL（`/`）にアクセスしたときに単に`"I'm alive"`という文字列を返すだけの、生存確認用のエンドポイントです。
- `run_flask()`がFlaskサーバーを実際に起動する関数です。`threaded=True`は「複数のリクエストを同時並行で処理できるようにする」設定で、これが無いと、あるリクエストの処理中は他のリクエストが待たされてしまいます。
- `keep_alive()`は、Flaskサーバーを**別スレッド**（`Thread`）で起動する関数です。`daemon=True`は「メインの処理（Discord Bot側）が終了したら、このスレッドも道連れで終了してよい」という指定です。この設計により、**同じプロセスの中でDiscord Bot（メインスレッド）とFlask API（別スレッド）が同時に動く**ことになります。

## 6. Discord Botの初期化（243〜251行）

```python
intents = discord.Intents.default()
intents.message_content = True
intents.presences = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)
```
- `Intents`（インテント）とは、Discord Botがどの種類のイベント情報を受け取りたいかをDiscord側に申告するための仕組みです。デフォルトでは無効になっている、より踏み込んだ情報（後述の「Privileged Gateway Intents」）を明示的に有効化しないと、Discord側にブロックされます。
  - `message_content`… メッセージの本文そのものを読み取れるようにする。
  - `presences`… メンバーのオンライン状態を取得できるようにする。
  - `members`… サーバーの全メンバー一覧を取得・キャッシュできるようにする（後述の「制限付きアカウント」判定＝対象サーバーのメンバーかどうかの確認に必須）。
- `bot = commands.Bot(...)`で、Discord Bot本体のオブジェクトを作ります。`command_prefix="!"`は昔ながらの「`!`で始まるメッセージをコマンドとして扱う」方式の名残りの設定ですが、実際にこのBotで使われているコマンドは、後述する`/`から始まる**スラッシュコマンド**（`@bot.tree.command`で定義されるもの）が中心です。

---

次回は、この後に続く「ローカルファイルの読み書きとSHA排他制御」の部分を解説します。 → [[../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md]]
