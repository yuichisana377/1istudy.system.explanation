# お知らせのデータ層と一覧・詳細取得・投稿API（`bot.py` 5697〜5921行）

対象：`bot.py`の「Flask API — お知らせ（notices）」セクション前半。[../../1I勉強会web解説/06_Notice](../../1I勉強会web解説/06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で見たお知らせページの、サーバー側の実装です。

## 1. ファイル名の検証と一覧取得（5697〜5722行）

```python
NOTICES_DIR = "notices"
NOTICES_META_FILE = "notices_meta.json"
NOTICE_ALLOWED_EXT = (".md", ".txt")

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
def list_notice_files():
    dir_path = _data_path(NOTICES_DIR)
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
- `notices/`フォルダの中の`.md`/`.txt`ファイルを列挙します。デスクトップの設計資料にある通り、お知らせは（CardMakerの索引のような専用のキャッシュファイルを持たず）**フォルダの中身を毎回直接スキャンする**方式です。お知らせの件数は通常それほど多くならない前提のため、CardMakerほどの最適化は必要とされていません。

## 2. メタ情報の読み書きと運用ログ整形（5725〜5771行）

```python
def load_notices_meta():
    data, sha = local_get(NOTICES_META_FILE)
    return (data or {}), sha

def save_notices_meta(meta, sha=None):
    if sha is None:
        _, sha = local_get(NOTICES_META_FILE)
    local_put(NOTICES_META_FILE, meta, sha)
    notify_change()  # ★ お知らせもguildをまたいで共有されるため全体に通知
```
- `notices_meta.json`は`{ファイル名: {投稿者・投稿日時・実行済みフラグ}}`という辞書です。お知らせ本文そのもの（`.md`/`.txt`ファイル）とは別のファイルに、メタ情報だけを持っています。

```python
def _notice_meta_entry_lines(filename, entry):
    """
    ★ お知らせのfilenameはNotice.js上でそのままタイトルとして全員に
    表示されている情報なので、他のカテゴリの内部ファイル名とは違い、
    フィールドとして表示してよい。"""
    if not entry:
        return []
    fields = [
        ("お知らせ", filename), ("投稿者", entry.get('uploader')),
        ("投稿日時", entry.get('uploaded_at')),
        ("状態", "実行済み" if entry.get("done") else "未実行"),
    ]
    return _json_block(fields)
```
- コメントに興味深い指摘があります。カードデッキの`filename`（`set_20260820_...json`のような、ハッシュのような形式で中身の推測ができない内部識別子）とは違い、お知らせのファイル名は`Notice.js`上で**そのままタイトルとして全員に表示されている**公開情報です。そのため、運用ログにファイル名をそのまま表示しても、新たに何かが漏れるわけではありません。「同じ`filename`というデータでも、機能によって公開情報か内部情報かが異なる」ため、運用ログでの扱い方もそれに応じて変わる、という一貫した考え方の実例です。

```python
def _notice_meta_entry_diff(filename, old_entry, new_entry):
    """
    ★ 追加（2026/08/19）：upload_notice/delete_noticeは、お知らせ本体の
    ファイルだけでなくnotices_meta.json（投稿者・投稿日時）も実際に
    書き換えているのに、これまで運用ログに出ていなかったため対応。"""
    old_text = "\n".join(_notice_meta_entry_lines(filename, old_entry))
    new_text = "\n".join(_notice_meta_entry_lines(filename, new_entry))
    diff = _text_diff_lines(old_text, new_text)
    if not diff:
        return None
    status = "added" if old_entry is None else ("deleted" if new_entry is None else "modified")
    return {"file": NOTICES_META_FILE, "diff": diff, "status": status}
```
- [../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)の`_card_index_diff`と同じパターンの、お知らせメタ情報専用版です。コメントの通り、以前は「お知らせ本体のファイル」の変更しか運用ログに出ておらず、実は同時に書き換わっている`notices_meta.json`側の変更（誰が投稿したか、など）が記録から漏れていた、という抜けを埋めるために追加されています。

## 3. `/list_notices`：一覧取得（5774〜5795行）

```python
@app.route("/list_notices", methods=["GET"])
def list_notices():
    files = list_notice_files()
    meta, _ = load_notices_meta()
    notices = []
    for f in files:
        m = meta.get(f["name"], {})
        notices.append({
            "filename": f["name"], "size": f.get("size"),
            "ext": f["name"].rsplit(".", 1)[-1].lower(),
            "uploader": m.get("uploader"), "uploaded_at": m.get("uploaded_at"),
            "done": bool(m.get("done")),
        })
    notices.sort(key=lambda n: n["filename"], reverse=True)
    return jsonify({"ok": True, "notices": notices})
```
- `list_notice_files()`（実ファイルの一覧）と`load_notices_meta()`（メタ情報）を、`filename`をキーに突き合わせて1つのレスポンスに組み立てます。
- `notices.sort(key=lambda n: n["filename"], reverse=True)`… コメントの通り「ファイル名（先頭に日付を付ける運用を推奨）で新しい順に並べる」という、ファイル名の文字列としての並びをそのまま日付順として利用する、これまでにも何度か登場したテクニックです。

## 4. `/get_notice`：詳細取得（5798〜5822行）

```python
@app.route("/get_notice", methods=["GET"])
def get_notice():
    """お知らせ1件の中身（テキスト本文）と投稿者名を返す"""
    filename = request.args.get("filename", "")
    if not _is_safe_notice_filename(filename):
        return jsonify({"ok": False, "error": "invalid filename"})
    try:
        path = _data_path(f"{NOTICES_DIR}/{filename}")
        if not os.path.isfile(path):
            return jsonify({"ok": False, "error": "not found"})
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        meta, _ = load_notices_meta()
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

## 5. `/upload_notice`：アップロード（5825〜5921行）

```python
@app.route("/upload_notice", methods=["POST"])
def upload_notice():
    data = request.json or {}
    guild_id, _student_id, resolved_nickname, err = require_login_json(data)
    if err:
        return err
    filename = (data.get("filename") or "").strip()
    content = data.get("content")
    uploader = resolved_nickname or "匿名"  # ★ クライアント自己申告ではなく、ログインセッションから引く

    if not _is_safe_notice_filename(filename):
        return jsonify({"ok": False, "error": ".md または .txt ファイルのみアップロードできます"})
    if content is None or not content.strip():
        return jsonify({"ok": False, "error": "内容が空です"})
    err = reject_if_bug_chars({"内容": content, "アップロード者": uploader})
    if err:
        return err
```
- ここまでの流れは、これまでの回で見た「変更」系APIの定番パターンです。

```python
    path = _data_path(f"{NOTICES_DIR}/{filename}")
    dirname = os.path.dirname(path)
    if dirname:
        os.makedirs(dirname, exist_ok=True)

    is_update = os.path.isfile(path)
    old_content = None
    if is_update:
        try:
            with open(path, "r", encoding="utf-8") as f:
                old_content = f.read()
        except OSError:
            old_content = None
    try:
        tmp_path = f"{path}.tmp"
        with open(tmp_path, "w", encoding="utf-8") as f:
            f.write(content)
        os.replace(tmp_path, path)
    except OSError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
```
- お知らせ本文の保存は、これまで見てきたJSONファイルとは違い、`local_put`（JSON形式で保存）ではなく、**プレーンテキストとして直接ファイルに書き込んでいます**（お知らせの中身自体はMarkdown/プレーンテキストなので、JSONに包む必要が無いためです）。ただし、`tmp_path`に書いてから`os.replace`ですり替える、という[../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)で見た**アトミックな書き込み**のテクニックは、ここでも同じように使われています。

```python
    meta_change = None
    try:
        meta, meta_sha = load_notices_meta()
        old_meta_entry = meta.get(filename)
        meta[filename] = {
            "uploader": uploader,
            # ★ 追加：削除の作成者確認機能で使うため、投稿者の学籍番号（student_id）も
            #   保存しておく。uploaderは表示名（ニックネーム）で改名され得るため、
            #   本人特定にはこちらを使う。運用ログの表示には出さない（Discord ID等と
            #   同じ扱いで、ニックネーム以上に個人を特定できる情報を公開の場に出さない方針のため）。
            "uploader_id": _student_id,
            "uploaded_at": datetime.now(JST).strftime("%Y-%m-%d %H:%M"),
        }
        save_notices_meta(meta, meta_sha)
        meta_change = _notice_meta_entry_diff(filename, old_meta_entry, meta[filename])
    except DataWriteError as e:
        print(f"[WARN] notices_meta の更新に失敗しました: {e}")
```
- `uploader`（表示名、後で変更されうる）と`uploader_id`（学籍番号、本人特定の唯一の正しい手がかり）が明確に区別されています。コメントの通り「uploaderは表示名（ニックネーム）で改名され得るため、本人特定にはこちらを使う」という考え方は、[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)の`published_by.id`と全く同じ設計思想です。
- `uploader_id`は`_notice_meta_entry_lines`（運用ログの表示用整形関数）には**含まれていません**。コメントの通り、これはDiscord IDと同じ扱いで、ニックネーム以上に個人を特定できる情報を、誰でも見られる公開の運用ログに出さない、という一貫した方針です。

```python
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
                    asyncio.run_coroutine_threadsafe(channel.send(msg), bot.loop).result(timeout=10)
        except Exception as e:
            print(f"[WARN] upload_notice notify failed: {e}")

    change = file_diff(f"{NOTICES_DIR}/{filename}", old_content, content)
    detail = [c for c in (change, meta_change) if c]
    log_event("notice", f"お知らせ「{filename}」を{'更新' if is_update else '追加'}しました。", actor=uploader, detail=detail if detail else None)
    return jsonify({"ok": True, "filename": filename, "is_update": is_update, "uploader": uploader})
```
- Discordへの通知先は、コメントの通り「お知らせ用」チャンネル（`notice_channel_id`）を優先し、未設定なら通生用チャンネル（`remind_channel_id`）にフォールバックします。
- `detail = [c for c in (change, meta_change) if c]`… [../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)の`/save_cards`と同じ、「本体ファイルの差分」と「メタ情報ファイルの差分」の両方を運用ログにまとめて渡すパターンです。

---

次は、お知らせの削除（作成者確認込み）と「実行済み」フラグの管理を解説します。 → [01_お知らせ削除と実行済みフラグ.md](01_お知らせ削除と実行済みフラグ.md)
