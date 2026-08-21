# スケジューラーの登録とBotの起動・再接続処理（`bot.py` 6778〜6885行）

対象：`bot.py`の「スケジューラー & 起動」セクション、そしてファイルの末尾。**このシリーズの最終回**です。ここまで定義してきた全ての定期処理が実際に登録され、最後にBot自体が起動します。

## 1. 定期ジョブの登録（6781〜6786行）

```python
scheduler.add_job(scheduled_backup_data_to_github, "cron", hour=0, minute=0)  # 毎日0:00（JST）にデータを自動バックアップ
scheduler.add_job(send_tomorrow_plans,     "cron", hour=20, minute=0)
scheduler.add_job(send_today_plans_commute, "cron", hour=5,  minute=30)  # 通生（現行時間）
scheduler.add_job(send_today_plans_dorm,    "cron", hour=7,  minute=20)  # 寮生
scheduler.add_job(send_weekly_plans,        "cron", day_of_week="sun", hour=14, minute=0)  # 毎週日曜14:00に今週の予定
scheduler.add_job(check_study_timers,       "interval", minutes=1)  # 勉強タイマーの3時間ごとの自動休憩チェック（最大1分遅れで検知）
```
- ここでようやく、[../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md](../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md)の冒頭で作られた`scheduler`（`AsyncIOScheduler`）に、実際のジョブが登録されます。このシリーズを通して個別に見てきた関数（[../08_自動通知とスケジューラー監視/00_定期通知の仕組み.md](../08_自動通知とスケジューラー監視/00_定期通知の仕組み.md)で解説した`send_tomorrow_plans`など、[../20_バックアップとgit連携/00_GitHubバックアップの仕組み.md](../20_バックアップとgit連携/00_GitHubバックアップの仕組み.md)の`scheduled_backup_data_to_github`）が、ここで初めて「実際にいつ実行されるか」と結びつけられます。
- `"cron"`ジョブ… `hour`/`minute`（さらに`day_of_week`）で指定した**特定の時刻**に実行されます。カレンダーの「毎日〇時」「毎週日曜〇時」のような予定に近い動き方です。
- `"interval"`ジョブ（`check_study_timers`のみ）… 一定の**間隔**（ここでは1分ごと）で繰り返し実行されます。コメントの通り「最大1分遅れで検知」となるのは、この1分間隔が理由です（3時間ちょうどに検知できるわけではなく、最大でも1分の誤差に収まる、という意味です。より即座の検知は、各APIアクセス時にその場評価される[../04_勉強タイマー/00_勉強タイマーの状態管理.md](../04_勉強タイマー/00_勉強タイマーの状態管理.md)の`_sync_timer_entry`が別途カバーしています）。
- コードの並びを見ると、`scheduler.add_job(...)`という**関数呼び出しの文が、`def`のようなブロックの外に、モジュールの最上位レベルにそのまま書かれています**。これは、Pythonのスクリプトが上から下に「実行」されていく性質を利用したものです。この`bot.py`ファイルが最初に読み込まれる（インポートされる）タイミングで、これらの行が実際に実行され、スケジューラーへのジョブ登録が完了します。ただし、`scheduler.start()`（実際にスケジューラーを動かし始める処理）自体は、まだこの時点では呼ばれていません（次項で見る`on_ready`の中で呼ばれます）。

## 2. Bot起動完了時の処理：`on_ready`（6788〜6811行）

```python
started = False
synced_once = False

@bot.event
async def on_ready():
    global started, synced_once
    print(f"Bot is ready! {bot.user}")

    # ★ 429対策①：コマンド同期は起動後1回だけ行う。
    #    再接続（resume失敗などで on_ready が複数回呼ばれるケース）のたびに
    #    tree.sync() を叩くと、それ自体がAPI呼び出しの積み重ねになり
    #    レート制限を誘発しやすくなるため、初回のみに限定する。
    if not synced_once:
        try:
            synced = await bot.tree.sync()
            print(f"Synced {len(synced)} commands")
        except discord.HTTPException as e:
            print(f"[WARN] tree.sync failed (will not retry until next process start): {e}")
        synced_once = True

    if not started:
        scheduler.start()
        started = True
        print("Scheduler started!")
```
- `on_ready`は、Discordとの接続が確立し、Botが実際に使える状態になったときに、discord.pyのライブラリから自動的に呼ばれるイベントハンドラです。
- `global started, synced_once`… これらの変数は関数の外（モジュールの最上位）で定義されているため、関数の中でこれらの値を**書き換える**には`global`宣言が必要です（Pythonでは、関数内で変数に代入すると、デフォルトではその関数だけのローカル変数として扱われてしまうため、外側の変数を変更したい場合は明示的に`global`と宣言する必要があります）。
- `bot.tree.sync()`… ここまで`@bot.tree.command(...)`で定義してきた全てのスラッシュコマンド（`/add`、`/list`など）の情報を、Discord側に登録（同期）する処理です。これを行って初めて、Discordのユーザーが実際に`/`と打ったときにコマンドの候補が表示されるようになります。
- **`synced_once`フラグで1回限りに制限している理由**がコメントに書かれています。「再接続（`resume`失敗などで`on_ready`が複数回呼ばれるケース）のたびに`tree.sync()`を叩くと、それ自体がAPI呼び出しの積み重ねになりレート制限を誘発しやすくなる」。`on_ready`は、初回の起動時だけでなく、ネットワークの瞬断などで接続が一度切れて再接続した際にも、再び呼ばれる可能性があります。もし毎回律儀に`tree.sync()`を呼んでいると、頻繁な再接続が起きた場合にDiscordのAPIへの呼び出し回数が積み重なり、[../11_FlaskAPI_ユーザー認証](../11_FlaskAPI_ユーザー認証/00_ユーザー一覧と連携コード発行とOAuth開始.md)などで見た「レート制限」と同じ種類の問題（今度はBotとDiscordサーバーとの間で）を引き起こしかねません。実際にはコマンドの一覧は起動中に変わることはまず無いため、**プロセスが生きている間に1回だけ同期すれば十分**という判断です。
- `scheduler.start()`も同様に`started`フラグで1回限りに制限されています。もし再接続のたびに`scheduler.start()`を呼んでしまうと、同じジョブが何重にも登録されて、例えば「毎日0時のバックアップ」が複数回同時に走ってしまう、といった不具合につながりかねません。

## 3. 切断イベント（6814〜6816行）

```python
@bot.event
async def on_disconnect():
    print("[WARN] Discord からの接続が切断されました。discord.py 内部で自動再接続を試みます。")
```
- 接続が切れたことを検知するだけの、ログ出力用のハンドラです。コメントにある通り、実際の再接続処理自体はdiscord.pyライブラリの内部が自動的に行うため、ここでは何もせず、状況をログに残すだけです。

## 4. 全体の起動処理（6819〜6823行）

```python
keep_alive()
start_quiz_scheduler()

print(f"[INFO] TOKEN set: {bool(TOKEN)}, length: {len(TOKEN) if TOKEN else 0}")
print(f"[INFO] Starting bot.run()...")
```
- ここで、[../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md](../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md)の`keep_alive()`（Flaskサーバーを別スレッドで起動）と、[../15_FlaskAPI_クイズ/00_クイズルームの設計とヘルパー関数.md](../15_FlaskAPI_クイズ/00_クイズルームの設計とヘルパー関数.md)の`start_quiz_scheduler()`（クイズの自動進行を監視する専用スレッドを起動）が、実際に呼び出されます。これで、**Flask（Web API）・Discord Bot・クイズ監視スレッドという3つの独立した処理の流れ**が、全てこの1つのプロセスの中で動き出す準備が整います。
- `print(f"[INFO] TOKEN set: {bool(TOKEN)}, length: {len(TOKEN) if TOKEN else 0}")`… トークンの**中身**は絶対にログに出しませんが、「設定されているかどうか」と「その文字数」だけは出力しています。これはデバッグ用の便利な情報（トークンが正しく環境変数から読み込めているかを確認できる）でありながら、トークンの中身自体を漏らさない、バランスの取れたログ出力です。

## 5. レート制限を考慮した再接続ループ：`run_bot_forever`（6825〜6885行）

### 設計方針（コメントより）

discord.pyは、ゲートウェイ（Discordとのリアルタイム通信路）が一時的に切断された場合の再接続（resume/reconnect）自体は、それ自体で自動的にバックオフ（再試行の間隔を徐々に伸ばしていくこと）しながら処理してくれます。問題になりやすいのは、**プロセスごと落ちて、ホスティング側（Renderなど）がすぐ再起動 → 起動のたびに新規IDENTIFY（Discordへの新規ログイン扱いの接続）を繰り返すケース**で、これがCloudflare側の429（1015 Too Many Requests、レート制限）を招きやすくなります。

そのため、`bot.run()`が例外で終了しても**プロセスを終了させず**、同じプロセスの中で指数バックオフ（失敗するたびに待ち時間を倍々に伸ばしていく方式）しながら`bot.run()`をやり直します。429（レート制限）の場合は、レスポンスの`retry_after`（あと何秒待てばよいか、という具体的な指示）秒だけ確実に待ってから再試行します。

```python
MAX_BACKOFF = 300  # 最大5分待機

def run_bot_forever():
    backoff = 5
    while True:
        try:
            bot.run(TOKEN)
            print("[INFO] bot.run() が正常終了しました。5秒後に再起動します。")
            time.sleep(5)
            backoff = 5
            continue
```
- `while True:`… 無限ループで、`bot.run(TOKEN)`（Botの接続・実行を開始する、通常は「戻ってこない」呼び出し）を何度でもやり直せるようにしています。
- コメントの通り「`bot.run()`は内部で`asyncio.run()`を呼ぶため、ここが正常終了/例外終了するたびにイベントループは閉じられる。discord.pyの`Client`は`close`後も再度`run()`できる設計になっている」ため、`bot.run()`が終わるたびに同じ`bot`オブジェクトで再び`run()`を呼び直すことができます。
- `bot.run()`が例外を投げずに戻ってきた場合（`bot.close()`などによる正常終了）は、5秒待ってから`backoff`（バックオフの待ち時間）を初期値の5秒にリセットして、ループを継続します。

```python
        except discord.HTTPException as e:
            if e.status == 429:
                retry_after = None
                try:
                    retry_after = float(e.response.headers.get("Retry-After"))
                except Exception:
                    pass
                if not retry_after:
                    retry_after = backoff
                print(f"[WARN] Discordからレート制限(429)を受けました。{retry_after:.1f}秒待機して再接続します。")
                time.sleep(retry_after + 1)
                backoff = min(backoff * 2, MAX_BACKOFF)
            else:
                print(f"[ERROR] discord.HTTPException: {e}. {backoff}秒後に再試行します。")
                time.sleep(backoff)
                backoff = min(backoff * 2, MAX_BACKOFF)
```
- `e.status == 429`… HTTPステータスコード429（Too Many Requests）を特別扱いします。`e.response.headers.get("Retry-After")`… Discordからのレスポンス自体に含まれる「あと何秒待てばよいか」という具体的な指示を読み取り、それを最優先で使います。もしこのヘッダーが無ければ（`retry_after`が取得できなければ）、代わりに現在の`backoff`の値を使います。`time.sleep(retry_after + 1)`… 指示された秒数に、念のため1秒の余裕を追加してから待機します。
- それ以外の`HTTPException`（429以外のHTTPエラー）は、単純に現在の`backoff`秒だけ待ってから再試行します。
- `backoff = min(backoff * 2, MAX_BACKOFF)`… **指数バックオフ**の実装です。失敗するたびに待ち時間を2倍にしていきますが、`MAX_BACKOFF`（300秒＝5分）を超えては伸ばさないよう、`min`で上限を設けています。これにより、一時的な障害が続く間は「5秒→10秒→20秒→…→最大5分」と、徐々に間隔を空けながら気長に再試行し続けます。もし上限を設けなければ、待ち時間がどんどん伸び続け、最終的には非現実的に長い時間になってしまいます。

```python
        except discord.LoginFailure as e:
            # トークンが無効な場合はリトライしても無駄なので停止する
            print(f"[FATAL] ログインに失敗しました。TOKENを確認してください: {e}")
            break

        except Exception as e:
            print(f"[ERROR] 予期しないエラーが発生しました: {e}. {backoff}秒後に再試行します。")
            time.sleep(backoff)
            backoff = min(backoff * 2, MAX_BACKOFF)
```
- `discord.LoginFailure`（トークン自体が無効という、根本的な設定ミス）だけは特別扱いされ、`break`でループ自体を抜けて処理を停止します。コメントにある通り「トークンが無効な場合はリトライしても無駄」だからです。何度再試行しても直らない種類の失敗を、無限にリトライし続けてログを埋め尽くすことのないよう、明確に区別されています。
- それ以外の予期しない例外（ネットワークの瞬断など）は、同じ指数バックオフの仕組みでキャッチし、プロセス自体は落とさずに再試行を続けます。

## ファイルの末尾

```python
run_bot_forever()
```
- 全ての準備（Flask・スケジューラー登録・各種イベントハンドラの定義）が整った後、この1行が実行されることで、Botが実際に動き出します。この呼び出しは（`LoginFailure`で`break`する場合を除いて）**通常は戻ってきません**。`while True:`の無限ループの中に留まり続け、プロセスが生きている限り、Botとしての機能を提供し続けます。

---

これで、`bot.py`全体（約6900行）の解説シリーズが完了しました。設定・起動まわりから始まり、データ保存の共通基盤、各機能のデータ層、Discordコマンド、自動通知、そして`bot.py`の半分以上を占めるFlask API群（予定管理・時間割・ユーザー認証・削除依頼システム・お知らせ・CardMaker・クイズ・学習進捗データ）を経て、最後にバックアップとBotの起動処理まで、上から下へ順番にたどってきました。

[../00_はじめに_この解説シリーズについて.md](../00_はじめに_この解説シリーズについて.md)で紹介した通り、これは[../../1I勉強会webの裏側（設計の話）.md](../../1I勉強会webの裏側（設計の話）.md)（設計の考え方）や[../../1I勉強会web解説](../../1I勉強会web解説/01_index_予定管理.md)（フロント側の実装）と組み合わせて読むことで、このサイトの裏側（サーバー側の実装の細部）と表側（画面・操作）の両方を一通りカバーできる資料になっています。
