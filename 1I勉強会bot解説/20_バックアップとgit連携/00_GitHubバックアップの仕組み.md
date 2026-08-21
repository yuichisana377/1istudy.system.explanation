# 毎晩のGitHubバックアップ（`bot.py` 6565〜6777行）

対象：`bot.py`の「データの自動バックアップ（毎日0:00・GitHubのプライベートリポジトリへ）」セクション。デスクトップの設計資料「1.5 バックアップ」で説明されている仕組みの、実際のコードです。

## 設計方針（コメントより）

`DATA_DIR`（実行時データ・生徒の個人情報を含む）はサーバーのローカルディスクにしか置かれておらず、これまで自動バックアップが存在しませんでした（以前はGitHub Contents APIで直接読み書きしていたため、その経路自体がバックアップも兼ねていましたが、ローカルディスク方式への移行でその経路ごと無くなっていました）。ここで追加されているのは、「毎晩、`DATA_DIR`の中身をまるごと専用のプライベートリポジトリへ`git push`する」というシンプルな仕組みです。

サーバー側で別途用意する必要があるものとして、①`git`コマンドが使えること、②環境変数`BACKUP_GITHUB_TOKEN`に、対象リポジトリへpushできるGitHubのPersonal Access Token（対象リポジトリの Contents: Read and write 権限）を設定しておくこと、が挙げられています。これらが揃っていない場合、バックアップはBot本体を止めずにスキップされ、理由だけがログに出力されます。

## 1. 設定とトークンの安全な扱い（6585〜6614行）

```python
BACKUP_GITHUB_TOKEN = os.getenv("BACKUP_GITHUB_TOKEN")
BACKUP_REPO_URL = os.getenv("BACKUP_REPO_URL", "https://github.com/yuichisana377/python.bot.1istudy-backup.git")
BACKUP_REPO_DIR = os.getenv("BACKUP_REPO_DIR", "/tmp/1istudy-backup-repo")
```
- `BACKUP_REPO_DIR`のデフォルト値についてもコメントに修正の経緯があります。「以前はDATA_DIRの1つ上の階層を既定値にしていたが、DATA_DIRがコンテナのアプリルート直下（例: `/app`）に設定されている環境だと、その1つ上＝ファイルシステムのルート直下に作ろうとしてしまい、権限エラー等で失敗する」。実行環境（Dockerコンテナなど）によって`DATA_DIR`の実際の位置が変わりうる中、「その相対位置」に依存した既定値は環境によって壊れる可能性があるため、`DATA_DIR`の場所に依存しない固定のパス（`/tmp`配下）に変更されています。

```python
def _run_git(args, cwd, use_auth=False):
    cmd = ["git"]
    if use_auth:
        # ★ 修正：GitHubのgit HTTP認証は "bearer" スキームを受け付けず、
        #   Basic認証（x-access-token:<token> をbase64化）である必要がある。
        #   以前はbearerを使っていたため401→ユーザー名入力待ちで失敗していた。
        basic = base64.b64encode(f"x-access-token:{BACKUP_GITHUB_TOKEN}".encode()).decode()
        cmd += ["-c", f"http.extraheader=AUTHORIZATION: basic {basic}"]
    cmd += args
    return subprocess.run(cmd, cwd=cwd, check=True, capture_output=True, text=True, timeout=120)
```
- `subprocess.run(...)`… Pythonから、OSの外部コマンド（ここでは`git`）を実行する標準的な方法です。`check=True`は「終了コードが失敗を表す場合、自動的に例外（`CalledProcessError`）を発生させる」指定、`capture_output=True`は標準出力・標準エラー出力を後から参照できるように取得する指定、`timeout=120`は120秒でタイムアウトさせる指定です。
- コメントには、認証方式に関する**実際の試行錯誤**が記録されています。GitHubのgit HTTP認証は、Bearer方式のトークンではなく、Basic認証（ユーザー名とパスワードを`:`で繋いでBase64エンコードする、古くからあるHTTP認証方式）である必要があり、以前はBearer方式を使っていたために認証が失敗し、`git`コマンドが対話的なユーザー名入力待ちの状態でハングしてしまっていた、という経緯です。修正後は、`x-access-token:<token>`という決まった形式の文字列をBase64エンコードして、正しいBasic認証ヘッダーとして渡しています。
- コメントの通り、このヘッダーは**トークンをファイル（`.git/config`など）に一切残さないよう**、`git`コマンド実行時の一時的な引数（`-c http.extraheader=...`）としてだけ渡されます。もし通常のやり方（リモートURLに`https://<token>@github.com/...`のようにトークンを埋め込む）をすると、そのURLが`.git/config`ファイルに保存されてしまい、サーバーの他のプロセスやバックアップ自体の巻き添えでトークンが漏れるリスクがあります。この方法なら、トークンはそのコマンド実行の一瞬だけメモリ上に存在し、ディスクには一切書き残されません。

```python
def _redact_token(text):
    """ログにトークンを絶対に出さないための保険。万一 use_auth 付きコマンドが
    失敗して stderr や cmd の中に -c http.extraheader=... が含まれていても、
    トークン本体は必ず伏せ字にする。"""
    if not BACKUP_GITHUB_TOKEN:
        return text
    return text.replace(BACKUP_GITHUB_TOKEN, "***REDACTED***")
```
- 上記の`-c http.extraheader=...`の仕組みは、コマンドの実行**自体**にトークンをファイルに残さない工夫ですが、もしそのコマンドが**失敗**した場合、エラーメッセージ（`stderr`）やコマンド自体の記録（`e.cmd`）に、実行したコマンド全体（トークンを含む文字列）がそのまま含まれてしまう可能性があります。`_redact_token`は、この失敗時のログ出力に対する**最後の保険**として、`BACKUP_GITHUB_TOKEN`という文字列がテキストのどこかに含まれていれば、それを`"***REDACTED***"`（編集済み、伏せ字の意味）に置き換えます。**トークンを扱う処理では、成功時の経路だけでなく、失敗時のエラーログという盲点にも同じ注意を払う必要がある**、という良い実例です。

## 2. バックアップ対象の匿名化：`_backup_category`（6616〜6649行）

デスクトップの設計資料にある通り、バックアップ対象は「生徒個人が特定できるデータファイル」を含みます。運用ログには、これをそのままの実ファイル名で出すことはできません（[../02_データ保存基盤/01_運用ログとdiff表示.md](../02_データ保存基盤/01_運用ログとdiff表示.md)で見た運用ログは、ログイン不要で誰でも見られる実質公開の場だからです）。

```python
_BACKUP_CATEGORY_PREFIXES = [
    ("study_data_", "学習データ"), ("study_logs_", "学習ログ"), ("study_timers_", "学習タイマー"),
    ("points_", "ポイント"), ("users_", "ユーザー情報"), ("completed_tasks_", "達成課題"),
    ("config_", "設定"), ("discord_login_links_", "Discordログイン連携"), ("discord_links_", "Discordリンク"),
    ("timetable_", "時間割"), ("cards_index", "カード"), ("folders", "フォルダ"), ("list_order", "表示順"),
    ("notices_meta", "お知らせ"), ("in_progress_decks", "学習中デッキ"), ("plans", "予定"),
    ("logs_", "ログ"), ("system_log", "システムログ"),
]

def _backup_category(rel_path):
    """変更されたファイルのパスから、個人を特定できるID等を含まない
    大まかな種別ラベルへ変換する（生徒個人が特定できるファイル名を
    そのままログに出さないため）。"""
    name = os.path.basename(rel_path)
    if rel_path.startswith("notices/") or rel_path.startswith("notices\\"):
        return "お知らせ（ファイル）"
    if rel_path.startswith("words/") or rel_path.startswith("words\\"):
        return "単語セット"
    for prefix, label in _BACKUP_CATEGORY_PREFIXES:
        if name.startswith(prefix):
            return label
    return "その他データ"
```
- `_BACKUP_CATEGORY_PREFIXES`… ファイル名の接頭辞と、対応する日本語ラベルの対応表です。例えば`study_data_1234_5678_abcd1234.json`（[../19_FlaskAPI_学習進捗データ/00_わからないマークと進捗の端末間共有.md](../19_FlaskAPI_学習進捗データ/00_わからないマークと進捗の端末間共有.md)で見た、学籍番号を含んだハッシュ化済みのファイル名）のようなファイルは、この対応表を通すと単に「学習データ」という大まかなラベルに変換され、実際にどの生徒のファイルだったかは一切分かりません。
- `for prefix, label in _BACKUP_CATEGORY_PREFIXES: if name.startswith(prefix): return label`… リストを順番に調べ、最初に一致した接頭辞のラベルを返します。どれにも一致しなければ`"その他データ"`という、これも中身が特定できない汎用ラベルになります。**「何が変わったか」という有用な情報は残しつつ、「誰の」データかという個人特定に繋がる情報は徹底して排除する**、匿名化の実例です。

## 3. 変更内容のサマリー化（6651〜6691行）

```python
def _backup_status_files(porcelain_output, max_files=30):
    """
    ★ 注意：他のカテゴリ（カードデッキ・お知らせ等）とは異なり、ここでは
    実際のファイル名の代わりに `_backup_category()` の大まかな種別ラベルを
    使う。"""
    files = []
    for line in (porcelain_output or "").splitlines():
        if len(line) < 4:
            continue
        code = line[:2].strip()
        rel = line[3:].strip().split(" -> ")[-1]
        if rel.startswith("data/"):
            rel = rel[len("data/"):]
        label = _backup_category(rel)
        action = "削除" if code.upper() == "D" else ("新規追加" if code in ("??", "A") else "更新")
        sign = "-" if action == "削除" else "+"
        status = {"削除": "deleted", "新規追加": "added", "更新": "modified"}[action]
        files.append({"file": label, "diff": f"{sign} {action}", "status": status})
    if len(files) > max_files:
        files = files[:max_files] + [{"file": None, "diff": f"…ほか{len(files) - max_files}件"}]
    return files
```
- `git status --porcelain`… `git status`コマンドに`--porcelain`オプションを付けると、人間向けの装飾された出力ではなく、プログラムで解析しやすい、機械可読な決まった形式の出力が得られます（各行が`XY パス`のような形式で、`X`/`Y`が変更の種類を表す2文字のコードです）。
- `code[:2]`… 行頭2文字の変更コードを取り出します。`"D"`（削除）・`"??"`または`"A"`（新規）・それ以外（更新）で日本語のアクション名に変換します。
- 前の章で見てきた`file_diff`（[../02_データ保存基盤/01_運用ログとdiff表示.md](../02_データ保存基盤/01_運用ログとdiff表示.md)）が返す`{"file", "diff", "status"}`という形式に、コメントの通り「実際のファイル名の代わりに`_backup_category()`の大まかな種別ラベル」を使って合わせています。これにより、バックアップの結果も、これまで見てきた運用ログの「変更されたファイル一覧」の表示形式にそのまま乗せることができます。
- `max_files=30`で上限を設け、それを超える件数の変更があれば、残りを「…ほか〇件」という1行にまとめます（バックアップは、通常のAPI呼び出しと違って一度に非常に多くのファイルが変わりうる操作なので、この上限は他の箇所より重要です）。

```python
def _summarize_backup_changes(porcelain_output):
    """`git status --porcelain`の出力から、種別ごとの変更件数を
    「学習データ2件、ポイント1件」のような短い日本語にまとめる。"""
    counts = {}
    for line in porcelain_output.splitlines():
        if len(line) < 4:
            continue
        rel = line[3:].strip().split(" -> ")[-1]
        if rel.startswith("data/"):
            rel = rel[len("data/"):]
        label = _backup_category(rel)
        counts[label] = counts.get(label, 0) + 1
    if not counts:
        return ""
    return "、".join(f"{label}{n}件" for label, n in counts.items())
```
- こちらは`_backup_status_files`（詳細な一覧）とは別に、「学習データ2件、ポイント1件」のような、運用ログの**要約行**（タップして展開する前の、一目で分かる短い文）を作るための集計です。

## 4. バックアップの実行本体：`backup_data_to_github`（6693〜6768行）

```python
def backup_data_to_github():
    """失敗してもBot本体を止めないよう、例外は外に投げずログだけ出す。"""
    if not BACKUP_GITHUB_TOKEN:
        print("[backup] 環境変数 BACKUP_GITHUB_TOKEN が未設定のため、自動バックアップをスキップしました。")
        return
    if shutil.which("git") is None:
        print("[backup] git コマンドが見つからないため、自動バックアップをスキップしました。")
        return
```
- コメントにあった「必要なものが揃っていなければスキップする」の実装です。`shutil.which("git")`は、OSの`PATH`環境変数を実際に探索して`git`コマンドが見つかるかを確認する、標準ライブラリの関数です。

```python
    try:
        if not os.path.isdir(os.path.join(BACKUP_REPO_DIR, ".git")):
            parent = os.path.dirname(BACKUP_REPO_DIR)
            if parent:
                os.makedirs(parent, exist_ok=True)
            _run_git(["clone", BACKUP_REPO_URL, BACKUP_REPO_DIR], cwd=".", use_auth=True)
        else:
            _run_git(["fetch", "origin", "main"], cwd=BACKUP_REPO_DIR, use_auth=True)
            _run_git(["reset", "--hard", "origin/main"], cwd=BACKUP_REPO_DIR)
```
- バックアップ用のリポジトリが、ローカルにまだクローンされていなければ（`.git`フォルダが無ければ）新規にクローンし、既にあれば`fetch`＋`reset --hard`で強制的に最新の状態に揃えます。`reset --hard`は、ローカルの変更を全て捨てて、リモートの状態に完全に合わせる強力なコマンドです。バックアップ専用のリポジトリなので、ローカルに何らかの中途半端な変更が残っていても気にせず、常にクリーンな状態から始めることを優先しています。

```python
        # ★ 修正：環境によってはDATA_DIRがコード本体と同じディレクトリになっている
        #   場合があり、DATA_DIR全体を丸ごとコピーするとbot.py・.git・.env（秘密情報）
        #   まで巻き込んでしまう。このリポジトリの .gitignore が「データ」とみなしている
        #   範囲（直下の*.jsonファイル、notices/・words/サブディレクトリ）だけを選んでコピーする。
        dest = os.path.join(BACKUP_REPO_DIR, "data")
        if os.path.isdir(dest):
            shutil.rmtree(dest)
        os.makedirs(dest, exist_ok=True)
        for name in os.listdir(DATA_DIR):
            src = os.path.join(DATA_DIR, name)
            if os.path.isfile(src) and name.endswith(".json"):
                shutil.copy2(src, os.path.join(dest, name))
        for sub in ("notices", "words"):
            src_sub = os.path.join(DATA_DIR, sub)
            if os.path.isdir(src_sub):
                shutil.copytree(src_sub, os.path.join(dest, sub))
```
- **これも実際の運用環境の落とし穴から学んだ修正です**。もし`DATA_DIR`が、Bot本体のコード（`bot.py`）と同じディレクトリに設定されている環境があった場合、単純に「`DATA_DIR`を丸ごとコピー」してしまうと、`bot.py`のソースコードや、`.env`ファイル（環境変数、`TOKEN`や`SESSION_SECRET`のような秘密情報が書かれているファイル）まで、バックアップ用のリポジトリにコピーされてしまう危険があります。デスクトップの設計資料にも「Botのコード本体（`bot.py`）や`.env`のような秘密情報はバックアップの対象から意図的に外している」とある通り、この修正では、「データとみなす範囲」（直下の`.json`ファイルと、`notices/`・`words/`サブディレクトリだけ）を**明示的に選んでコピー**する方式に変更されています。ホワイトリスト方式（許可するものだけを明示的に指定する）を徹底することで、意図しないファイルの混入を防いでいます。
- `shutil.rmtree(dest)`→`os.makedirs(dest)`（一旦丸ごと削除してから作り直す）… コメントにある通り、これにより「削除されたファイル」もバックアップに正しく反映されます。もし既存の`dest`フォルダに単純に上書きコピーするだけだと、元のデータ側で既に削除されたファイルが、バックアップ側にはいつまでも残り続けてしまいます（バックアップリポジトリの`data/`フォルダの中身を、毎回`DATA_DIR`の現状と完全に一致させる、という発想です）。

```python
        status = _run_git(["status", "--porcelain"], cwd=BACKUP_REPO_DIR)
        if not status.stdout.strip():
            print("[backup] 前回から変更が無いため、コミットはスキップしました。")
            return

        change_summary = _summarize_backup_changes(status.stdout)
        timestamp = datetime.now(timezone("UTC")).strftime("%Y-%m-%dT%H:%M:%SZ")
        _run_git(["add", "-A"], cwd=BACKUP_REPO_DIR)
        _run_git(
            ["-c", "user.email=backup@1istudy.local", "-c", "user.name=1istudy-backup",
             "commit", "-m", f"backup {timestamp}"],
            cwd=BACKUP_REPO_DIR,
        )
        _run_git(["push", "origin", "HEAD:main"], cwd=BACKUP_REPO_DIR, use_auth=True)
        print(f"[backup] {timestamp} のバックアップをpushしました。")
        log_event("backup", f"データをバックアップしました（{change_summary}）" if change_summary else "データをバックアップしました。", detail=_backup_status_files(status.stdout))
```
- `if not status.stdout.strip(): return`… コメントの通り「前回から変化が無ければコミットしない（空コミットの量産を防ぐ）」という配慮です。データが1件も変わっていなければ、そもそもコミット自体を作らずに終わります。
- `["-c", "user.email=...", "-c", "user.name=...", "commit", ...]`… gitのコミットには、通常グローバル設定やリポジトリ設定でユーザー名・メールアドレスがあらかじめ設定されている必要がありますが、`-c`オプションでその場限りの設定値を指定することで、サーバー環境に依存せず、確実に固定のbot用の名前でコミットできるようにしています。
- `_run_git(["push", "origin", "HEAD:main"], ..., use_auth=True)`… ここで初めて、認証付きの`push`が行われます。ここでも`use_auth=True`により、トークンは一時的なヘッダーとしてのみ使われます。
- 成功すれば`log_event`で運用ログにも記録されます（このカテゴリは`"backup"`で、他の機能とは別枠として扱われます）。デスクトップの設計資料にある通り、バックアップは「サーバー主導の処理」なので、`actor`（実行者）は指定されず`None`のままです。

```python
    except subprocess.CalledProcessError as e:
        # ★ 修正：e.cmd には use_auth=True 時の -c http.extraheader=...（トークン本体）
        #   がそのまま含まれるため、ログに出す前に必ず伏せ字にする。
        safe_cmd = _redact_token(" ".join(e.cmd))
        safe_stderr = _redact_token(e.stderr or "")
        print(f"[backup] 失敗しました（{safe_cmd}）: {safe_stderr}")
        log_event("backup", f"バックアップに失敗しました: {safe_stderr[:200]}", level="error", detail=f"{safe_cmd}\n{safe_stderr}")
    except Exception as e:
        safe_msg = _redact_token(str(e))
        print(f"[backup] 失敗しました: {safe_msg}")
        log_event("backup", f"バックアップに失敗しました: {safe_msg[:200]}", level="error", detail=safe_msg)
```
- 失敗時の処理でも、前述の`_redact_token`が徹底されています。`subprocess.CalledProcessError`（`git`コマンド自体が失敗した場合）と、それ以外の予期しない例外の両方を捕まえ、**どちらの経路でも**トークンを伏せ字にしてからログに出力しています。
- 関数全体がこの2つの`except`で覆われており、コメントの通り「失敗してもBot本体を止めない」ようになっています。バックアップの失敗は、`log_event`によって`level="error"`として運用ログに記録されるため、後から気づいて対応できます。

## 5. 非同期化：`scheduled_backup_data_to_github`（6770〜6776行）

```python
async def scheduled_backup_data_to_github():
    """★ backup_data_to_github() はブロッキングI/O（subprocess・ファイルコピー）
    を含む同期関数なので、asyncioのイベントループ（Discordの通信もここで
    動いている）を止めないよう、別スレッドで実行する。"""
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, backup_data_to_github)
```
- [../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)の`async_local_get`と全く同じパターンです。`git`コマンドの実行やファイルコピーは、完了まで数秒〜数十秒かかることもある「重い」処理です。これをそのままDiscord Botの非同期イベントループの中で実行してしまうと、その間Bot全体が他のイベント（メッセージへの応答など）に反応できなくなってしまうため、別スレッドに逃がしています。

---

次は、これらのスケジュールされたジョブの実際の登録と、Botの起動・再接続処理を解説します。 → [../21_スケジューラーとBot起動/00_ジョブ登録とBotの起動リトライ.md](../21_スケジューラーとBot起動/00_ジョブ登録とBotの起動リトライ.md)
