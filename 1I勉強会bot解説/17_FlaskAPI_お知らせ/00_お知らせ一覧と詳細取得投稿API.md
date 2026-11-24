# お知らせのデータ層と一覧・詳細取得・投稿API（`bot.py` 8648〜8927行）

対象：`bot.py`の「Flask API — お知らせ（notices）」セクション前半。[../../1I勉強会web解説/06_Notice](../../1I勉強会web解説/06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で見たお知らせページの、サーバー側の実装です。

## 1. 置き場所（サーバーごと）・ファイル名の検証・一覧取得（8652〜8702行）

```python
NOTICE_ALLOWED_EXT = (".md", ".txt")
```

```python
# ★ 複数サーバー対応：以前はnotices/・notices_meta.jsonが全サーバー共通の
#   単一の置き場だったため、新サーバーのお知らせ一覧に1Iのお知らせが
#   全部混ざって見えてしまっていた。お知らせはCardMakerのデッキと違い
#   生のMarkdown/テキストファイル（JSON構造を持たない）なので、ファイル内に
#   guild_idを埋め込む方式が使えない。かつファイル名は利用者が自由に
#   付けられる（CardMakerのようなランダムファイル名ではない）ため、
#   共有ディレクトリのままだと他サーバーとファイル名が衝突するリスクもある。
#   そのため、ディレクトリ自体をサーバーごとに分ける方式にした。
def notices_dir(guild_id: int) -> str:
    return f"notices_{guild_id}"

def notices_meta_file(guild_id: int) -> str:
    return f"notices_meta_{guild_id}.json"
```
- コメントの通り、お知らせの置き場所は**サーバー（guild）ごとに別ディレクトリ**です（`notices_{guild_id}/`と`notices_meta_{guild_id}.json`）。CardMakerのデッキのように「JSONの中に`guild_id`を書いて区別する」方式はお知らせには使えません。お知らせは生のMarkdown/テキストファイルでJSONの構造を持たず、ファイル名も利用者が自由に付けるので、同じディレクトリに置くと他のサーバーのお知らせとファイル名がぶつかる恐れもあるためです。

```python
def _is_safe_notice_filename(filename: str) -> bool:
    """パストラバーサル対策・拡張子チェック"""
    if not filename:
        return False
    if "/" in filename or "\\" in filename or ".." in filename:
        return False
    return filename.lower().endswith(NOTICE_ALLOWED_EXT)
```
- これまでの各所で個別に書かれていたパストラバーサル対策が、[../15_FlaskAPI_クイズ/02_ルーム作成と過去問ランキング.md](../15_FlaskAPI_クイズ/02_ルーム作成と過去問ランキング.md)の`_is_safe_deck_filename`と同じように、お知らせ専用の1つの関数にまとめられています。加えて拡張子（`.md`または`.txt`のみ）のチェックも兼ねています。

```python
def list_notice_files(guild_id: int):
    dir_path = _data_path(notices_dir(guild_id))
    if not os.path.isdir(dir_path):
        return []
    results = []
    for name in sorted(os.listdir(dir_path)):
        if not name.lower().endswith(NOTICE_ALLOWED_EXT):
            continue
        full_path = os.path.join(dir_path, name)
        if os.path.isfile(full_path):
            results.append({"name": name, "size": os.path.getsize(full_path)})
    return results
```
- `notices_{guild_id}/`フォルダの中の`.md`/`.txt`ファイルを列挙します。デスクトップの設計資料にある通り、お知らせは（CardMakerの索引のような専用のキャッシュファイルを持たず）**フォルダの中身を毎回直接スキャンする**方式です。お知らせの件数は通常それほど多くならない前提のため、CardMakerほどの最適化は必要とされていません。

## 2. メタ情報の読み書きと運用ログ整形（8705〜8751行）

```python
def load_notices_meta(guild_id: int):
    data, sha = local_get(notices_meta_file(guild_id))
    return (data or {}), sha


def save_notices_meta(guild_id: int, meta, sha=None):
    if sha is None:
        _, sha = local_get(notices_meta_file(guild_id))
    local_put(notices_meta_file(guild_id), meta, sha)
    notify_change(guild_id)  # ★ 複数サーバー対応：guildごとに分離したので、そのguildだけに通知する
```
- `notices_meta_{guild_id}.json`は`{ファイル名: {投稿者・投稿日時・実行済みフラグ}}`という辞書です。お知らせ本文そのもの（`.md`/`.txt`ファイル）とは別のファイルに、メタ情報だけを持っています。
- `notify_change(guild_id)`… 以前はお知らせが全サーバー共通だったため全体に通知していましたが、サーバーごとに分けたので、そのサーバーを見ているタブだけに「データが変わった」と知らせます。

```python
def _notice_meta_entry_lines(filename, entry):
    """notices_meta.json内の1エントリを { ... } のブロックにする。
    ★ お知らせのfilenameはNotice.js上でそのままタイトルとして全員に
    表示されている情報なので、他のカテゴリの内部ファイル名とは違い、
    フィールドとして表示してよい。"""
    if not entry:
        return []
    fields = [
        ("お知らせ", filename),
        ("投稿者", entry.get('uploader')),
        ("投稿日時", entry.get('uploaded_at')),
        ("状態", "実行済み" if entry.get("done") else "未実行"),
    ]
    return _json_block(fields)
```
- コメントに興味深い指摘があります。カードデッキの`filename`（`set_20260820_...json`のような、ハッシュのような形式で中身の推測ができない内部識別子）とは違い、お知らせのファイル名は`Notice.js`上で**そのままタイトルとして全員に表示されている**公開情報です。そのため、運用ログにファイル名をそのまま表示しても、新たに何かが漏れるわけではありません。「同じ`filename`というデータでも、機能によって公開情報か内部情報かが異なる」ため、運用ログでの扱い方もそれに応じて変わる、という一貫した考え方の実例です。

```python
def _notice_meta_entry_diff(guild_id, filename, old_entry, new_entry):
    """notices_meta_{guild_id}.json内の1エントリの変更を、log_event の
    detail に渡す {"file","diff","status"} の形にする（無ければNone）。
    ★ 追加（2026/08/19）：upload_notice/delete_noticeは、お知らせ本体の
    ファイルだけでなくnotices_meta.json（投稿者・投稿日時）も実際に
    書き換えているのに、これまで運用ログに出ていなかったため対応。"""
    old_text = "\n".join(_notice_meta_entry_lines(filename, old_entry))
    new_text = "\n".join(_notice_meta_entry_lines(filename, new_entry))
    diff = _text_diff_lines(old_text, new_text)
    if not diff:
        return None
    status = "added" if old_entry is None else ("deleted" if new_entry is None else "modified")
    return {"file": notices_meta_file(guild_id), "diff": diff, "status": status}
```
- [../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)の`_card_index_diff`と同じパターンの、お知らせメタ情報専用版です。コメントの通り、以前は「お知らせ本体のファイル」の変更しか運用ログに出ておらず、実は同時に書き換わっている`notices_meta.json`側の変更（誰が投稿したか、など）が記録から漏れていた、という抜けを埋めるために追加されています。

## 3. `/list_notices`：一覧取得（8754〜8780行）

```python
@app.route("/list_notices", methods=["GET"])
def list_notices():
    """お知らせファイルの一覧を返す（中身は含まない、投稿者名つき）"""
    # ★ 追加（2026/08/24）：予定一覧以外の閲覧系APIをサーバー参加済み
    #   ログインユーザー限定にする方針により追加。
    guild_id, _student_id, err = require_member_session_get()
    if err:
        return err
    try:
        files = list_notice_files(guild_id)
        meta, _ = load_notices_meta(guild_id)
        notices = []
        for f in files:
            m = meta.get(f["name"], {})
            notices.append({
                "filename": f["name"],
                "size": f.get("size"),
                "ext": f["name"].rsplit(".", 1)[-1].lower(),
                "uploader": m.get("uploader"),
                "uploaded_at": m.get("uploaded_at"),
                "done": bool(m.get("done")),  # ★ 追加：実行済み（全員共有）
            })
        # ファイル名（先頭に日付を付ける運用を推奨）で新しい順に並べる
        notices.sort(key=lambda n: n["filename"], reverse=True)
        return jsonify({"ok": True, "notices": notices})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- `require_member_session_get()`… コメントの通り、2026/08/24から一覧の閲覧も「サーバーに参加しているログイン中のユーザー」に限定されています。戻り値の`guild_id`で、どのサーバーのお知らせを返すかが決まります。
- `list_notice_files()`（実ファイルの一覧）と`load_notices_meta()`（メタ情報）を、`filename`をキーに突き合わせて1つのレスポンスに組み立てます。
- `notices.sort(key=lambda n: n["filename"], reverse=True)`… コメントの通り「ファイル名（先頭に日付を付ける運用を推奨）で新しい順に並べる」という、ファイル名の文字列としての並びをそのまま日付順として利用する、これまでにも何度か登場したテクニックです。

## 4. `/get_notice`：詳細取得（8783〜8810行）

```python
@app.route("/get_notice", methods=["GET"])
def get_notice():
    """お知らせ1件の中身（テキスト本文）と投稿者名を返す"""
    guild_id, _student_id, err = require_member_session_get()
    if err:
        return err
    filename = request.args.get("filename", "")
    if not _is_safe_notice_filename(filename):
        return jsonify({"ok": False, "error": "invalid filename"})
    try:
        path = _data_path(f"{notices_dir(guild_id)}/{filename}")
        if not os.path.isfile(path):
            return jsonify({"ok": False, "error": "not found"})
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        meta, _ = load_notices_meta(guild_id)
        m = meta.get(filename, {})

        return jsonify({
            "ok": True,
            "filename": filename,
            "content": content,
            "uploader": m.get("uploader"),
            "uploaded_at": m.get("uploaded_at"),
        })
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- ファイル本文をそのままテキストとして読み込み返します。[Web解説シリーズのNotice.js解説](../../1I勉強会web解説/06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で見た通り、この本文はフロント側でMarkdownとしてレンダリングされる際に、必ず`textContent`/DOM APIを使った安全な組み立て方がされます（サーバー側は特に無害化処理をせず、生のテキストをそのまま返しています）。
- 本文に画像が入っている場合、その画像は data URI（`data:image/jpeg;base64,...`）として本文の中にそのまま書かれています。サーバーは画像を別ファイルとして持たず、ここでも本文を丸ごと返すだけです（6章参照）。

## 5. `/upload_notice`：アップロード（8813〜8927行）

```python
@app.route("/upload_notice", methods=["POST"])
def upload_notice():
    """お知らせファイル（.md / .txt）をアップロード（新規 or 上書き）する"""
    data = request.json or {}
    guild_id, _student_id, resolved_nickname, err = require_login_json(data)  # ★ 変更にはログイン必須
    if err:
        return err
    filename = (data.get("filename") or "").strip()
    content = data.get("content")
    uploader = resolved_nickname or "匿名"  # ★ クライアント自己申告ではなく、ログインセッションから引く

    if not _is_safe_notice_filename(filename):
        return jsonify({"ok": False, "error": ".md または .txt ファイルのみアップロードできます"})
    if content is None or not content.strip():
        return jsonify({"ok": False, "error": "内容が空です"})
    # 画像は data URI としてMarkdown本文に埋め込まれるため、極端に大きい投稿は弾く
    if len(content) > NOTICE_MAX_CONTENT_CHARS:
        return jsonify({"ok": False, "error": "内容（画像を含む）が大きすぎます。画像の枚数を減らしてください"})

    # 画像(base64)部分は文字種が限られ禁止文字を含み得ない。1文字ずつの判定は重いので除いて調べる
    err = reject_if_bug_chars({"内容": _NOTICE_IMG_DATA_RE.sub("()", content), "アップロード者": uploader})
    if err:
        return err
```
- ここまでの流れは、これまでの回で見た「変更」系APIの定番パターンです。
- 画像添付に合わせて、`NOTICE_MAX_CONTENT_CHARS`による本文の長さ制限と、禁止文字チェック（`reject_if_bug_chars`）から画像部分を除く処理が加わっています。詳しくは6章で説明します。

```python
    path = _data_path(f"{notices_dir(guild_id)}/{filename}")
    dirname = os.path.dirname(path)
    if dirname:
        os.makedirs(dirname, exist_ok=True)

    # 既存ファイルなら上書き
    is_update = os.path.isfile(path)
    # ★ 運用ログで内容の差分を見せるため、上書き前に旧内容を読んでおく。
    #   読めなくても（＝新規作成、または読み取り失敗）アップロード自体は続行する。
    old_content = None
    if is_update:
        try:
            with open(path, "r", encoding="utf-8") as f:
                old_content = f.read()
        except OSError:
            old_content = None
    # ★ 修正（不具合修正、2026/08/26）：_local_write_once と全く同じ理由で、
    #   固定名の一時ファイル（f"{path}.tmp"）だと、同じお知らせへほぼ同時に
    #   届いた2つの投稿リクエストが一時ファイルを共有し、内容が混ざった
    #   壊れたファイルになりうる。呼び出しごとに一意な一時ファイル名にする。
    tmp_path = f"{path}.{os.getpid()}.{secrets.token_hex(8)}.tmp"
    try:
        with open(tmp_path, "w", encoding="utf-8") as f:
            f.write(content)
        os.replace(tmp_path, path)
    except OSError as e:
        return jsonify({
            "ok": False,
            "error": f"local_write_failed: {e}"
        })
    finally:
        try:
            os.remove(tmp_path)
        except OSError:
            pass
```
- お知らせ本文の保存は、これまで見てきたJSONファイルとは違い、`local_put`（JSON形式で保存）ではなく、**プレーンテキストとして直接ファイルに書き込んでいます**（お知らせの中身自体はMarkdown/プレーンテキストなので、JSONに包む必要が無いためです）。ただし、`tmp_path`に書いてから`os.replace`ですり替える、という[../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)で見た**アトミックな書き込み**のテクニックは、ここでも同じように使われています。
- **（2026/08/26追記）`tmp_path`を一意な名前に修正**：以前は`f"{path}.tmp"`という固定名だったため、`_local_write_once`（[00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)参照）で実際に見つかったのと全く同じ理由（Flaskの`threaded=True`で、同じお知らせへほぼ同時に届いた2つの投稿リクエストが一時ファイルを共有し、内容が混ざって壊れる）で破損する可能性があった。`upload_notice`はこの一時ファイル書き込みを`_local_write_once`とは独立に自前で実装していたため、あちらを直しても連動して直らない別コードだった。プロセスID＋ランダム値で呼び出しごとに一意な一時ファイル名にし、例外時の後始末（`finally`での`os.remove`）も揃えた。

```python
    # --- 投稿者メタ情報を notices_meta_{guild_id}.json に保存 ---
    meta_change = None
    try:
        meta, meta_sha = load_notices_meta(guild_id)
        old_meta_entry = meta.get(filename)
        meta[filename] = {
            "uploader": uploader,
            # ★ 追加：削除の作成者確認機能で使うため、投稿者の学籍番号（student_id）も
            #   保存しておく。uploaderは表示名（ニックネーム）で改名され得るため、
            #   本人特定にはこちらを使う。運用ログの表示（_notice_meta_entry_lines）
            #   には出さない（Discord ID等と同じ扱いで、ニックネーム以上に個人を
            #   特定できる情報を公開の場に出さない方針のため）。
            "uploader_id": _student_id,
            "uploaded_at": datetime.now(JST).strftime("%Y-%m-%d %H:%M"),
        }
        save_notices_meta(guild_id, meta, meta_sha)
        meta_change = _notice_meta_entry_diff(guild_id, filename, old_meta_entry, meta[filename])
    except DataWriteError as e:
        # 本体の保存自体は成功しているので、メタ情報の失敗は警告に留める
        print(f"[WARN] notices_meta の更新に失敗しました: {e}")
```
- `uploader`（表示名、後で変更されうる）と`uploader_id`（学籍番号、本人特定の唯一の正しい手がかり）が明確に区別されています。コメントの通り「uploaderは表示名（ニックネーム）で改名され得るため、本人特定にはこちらを使う」という考え方は、[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)の`published_by.id`と全く同じ設計思想です。
- `uploader_id`は`_notice_meta_entry_lines`（運用ログの表示用整形関数）には**含まれていません**。コメントの通り、これはDiscord IDと同じ扱いで、ニックネーム以上に個人を特定できる情報を、誰でも見られる公開の運用ログに出さない、という一貫した方針です。

```python
    # --- 任意：Discordの通知チャンネルに投稿 ---
    #     /setchannel main で設定した「お知らせ用」チャンネルを優先し、
    #     未設定の場合は通生用チャンネル（remind_channel_id）にフォールバックする
    if guild_id:
        try:
            guild_id_int = int(guild_id)
            guild = bot.get_guild(guild_id_int)
            if guild:
                config = load_config(guild_id_int)
                channel_id = config.get("notice_channel_id") or config.get("remind_channel_id")
                channel = bot.get_channel(channel_id) if channel_id else None
                if channel:
                    action = "更新" if is_update else "公開"
                    msg = f"📢 お知らせ「{filename}」が{uploader}さんによって{action}されました！"
                    asyncio.run_coroutine_threadsafe(
                        channel.send(msg), bot.loop
                    ).result(timeout=10)
        except Exception as e:
            print(f"[WARN] upload_notice notify failed: {e}")

    change = file_diff(
        f"{notices_dir(guild_id)}/{filename}",
        _mask_notice_images(old_content),
        _mask_notice_images(content),
    )
    detail = [c for c in (change, meta_change) if c]
    log_event(
        guild_id,
        "notice",
        f"お知らせ「{filename}」を{'更新' if is_update else '追加'}しました。",
        actor=uploader,
        detail=detail if detail else None,
    )
    return jsonify({"ok": True, "filename": filename, "is_update": is_update, "uploader": uploader})
```
- Discordへの通知先は、コメントの通り「お知らせ用」チャンネル（`notice_channel_id`）を優先し、未設定なら通生用チャンネル（`remind_channel_id`）にフォールバックします。
- `detail = [c for c in (change, meta_change) if c]`… [../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)の`/save_cards`と同じ、「本体ファイルの差分」と「メタ情報ファイルの差分」の両方を運用ログにまとめて渡すパターンです。
- 運用ログに渡す前後の本文は`_mask_notice_images()`を通しています（6章参照）。

## 6. 画像添付への対応：サイズ上限とマスク（8652〜8665行）

お知らせには画像を添付できます。Web側（[../../1I勉強会web解説/06_Notice/01_Notice.js_その2_削除依頼とアップロード編集.md](../../1I勉強会web解説/06_Notice/01_Notice.js_その2_削除依頼とアップロード編集.md)の「画像の添付」）が画像を縮小してから、Markdownの画像記法`![名前](data:image/jpeg;base64,...)`として本文の中に埋め込んで送ってきます。サーバーには画像専用の保存先は無く、本文の一部として普通のお知らせと同じように保存されます。そのためサーバー側で追加されたのは、次の3つの対策だけです。

```python
NOTICE_ALLOWED_EXT = (".md", ".txt")
NOTICE_MAX_CONTENT_CHARS = 6_000_000  # 画像(data URI)込みの本文の上限。Web側(Notice.js)と同値

_NOTICE_IMG_DATA_RE = re.compile(r"\(data:image/[a-z+]+;base64,[A-Za-z0-9+/=]+\)")


def _mask_notice_images(text):
    """運用ログ用：本文に埋め込まれた画像のbase64を「(画像データ)」に置き換える
    （巨大な文字列がログの差分に出ないようにする）。"""
    if not text:
        return text
    # 画像だけ差し替えた変更も差分に出るよう、中身の短いハッシュを付ける
    return _NOTICE_IMG_DATA_RE.sub(
        lambda m: "(画像データ " + hashlib.sha1(m.group(0).encode("utf-8")).hexdigest()[:8] + ")", text)
```
- **長さの上限**：`NOTICE_MAX_CONTENT_CHARS`（600万文字）は、コメントの通りWeb側（`Notice.js`の`NOTICE_IMG_MAX_TOTAL`）と同じ値です。画像込みで極端に大きい投稿は、`upload_notice`の入口（8829行）で弾きます。
- **禁止文字チェックの軽量化**：`reject_if_bug_chars`は1文字ずつ調べるため、数MBのbase64があると重くなります。base64に使われる文字（英数字と`+/=`）は禁止文字になり得ないので、8833行では`_NOTICE_IMG_DATA_RE.sub("()", content)`で画像部分を取り除いた本文だけを調べています（保存されるのは元の`content`のままです）。
- **運用ログのマスク**：`_mask_notice_images()`は画像部分を`(画像データ 1a2b3c4d)`のような短い目印に置き換えます。後ろの8文字は画像データのsha1ハッシュの先頭です。コメントの通り、文章は同じで画像だけ差し替えた場合にも、目印が変わるので差分に出ます。このマスクは3か所で使われています。投稿時の運用ログ（`upload_notice`、8914〜8918行）、削除時の運用ログ（`_delete_notice_file`、[01_お知らせ削除と実行済みフラグ.md](01_お知らせ削除と実行済みフラグ.md)参照）、削除依頼の承認ページに出す本文（`_delete_target_summary`、[../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)参照）です。

---

次は、お知らせの削除（作成者確認込み）と「実行済み」フラグの管理を解説します。 → [01_お知らせ削除と実行済みフラグ.md](01_お知らせ削除と実行済みフラグ.md)
