# 作成中デッキの一覧共有API（`bot.py` 5554〜5692行）

対象：`bot.py`の「Flask API — 作成中デッキ（公開予定だがまだ未公開のもの）をみんなで共有表示する」セクション。

## 設計方針（コメントより）

- カード名だけ入力して「作成」を押した時点で登録し、他の人の一覧にも「🟠 作成中（〇〇さん）」として表示できるようにします。
- カード本体（問題・解答）はここには一切含めません（軽量なメタ情報のみ）。まだ他人に見せるべきでない中身までは共有せず、「誰が何というデッキを作っている最中か」という存在だけをみんなに知らせる仕組みです。
- 実際に公開（[[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md]]の`/save_cards`）されたら、対応するエントリはここから取り除かれます。

`in_progress_decks.json`（`IN_PROGRESS_FILE`）に、作成中デッキのメタ情報だけを配列で保持します。

## 1. 読み書きと期限切れの間引き（5566〜5589行）

```python
def load_in_progress():
    data, sha = local_get(IN_PROGRESS_FILE)
    return (data or []), sha

def save_in_progress(items, sha=None):
    if sha is None:
        _, sha = local_get(IN_PROGRESS_FILE)
    local_put(IN_PROGRESS_FILE, items, sha)

def _prune_stale_in_progress(items):
    """登録から IN_PROGRESS_STALE_DAYS 日以上経過したエントリを取り除いた新しいリストを返す。
    （壊れた/古い形式の created_at は安全側に倒して除外しない）"""
    now_jst = datetime.now(JST)
    kept = []
    for it in items:
        created_at = it.get("created_at")
        try:
            created_dt = datetime.strptime(created_at, "%Y-%m-%d %H:%M:%S").replace(tzinfo=JST)
            if (now_jst - created_dt).days > IN_PROGRESS_STALE_DAYS:
                continue
        except Exception:
            pass
        kept.append(it)
    return kept
```
- `_prune_stale_in_progress`… 「作成中」のまま長期間放置されたデッキ（作りかけで途中で辞めてしまったものなど）を、一定日数を過ぎたら自動的に一覧から除きます。
- `try: ... except Exception: pass`（`kept.append(it)`より前）… `created_at`が壊れている・想定外の形式の場合、日数の計算自体ができません。コメントにある通り「安全側に倒して除外しない」、つまり判定に失敗したエントリは**除外せずそのまま残す**という判断です。もし逆に「よく分からないものは消してしまう」設計にしていたら、何らかのバグでデータ形式が想定外になった際に、まだ有効なはずの作成中デッキの表示が意図せず一斉に消えてしまう、というより大きな実害につながりかねません。**判定できない場合は、より安全な側（何もしない）に倒す**という考え方は、[[../11_FlaskAPI_ユーザー認証/01_OAuthコールバックと初回登録.md]]の`_guild_membership_status`（`"unknown"`を安全側でメンバーでない扱いにする）とも共通する、このファイル全体を貫く方針です。

## 2. 一覧取得：`/list_in_progress`（5591〜5603行）

```python
@app.route("/list_in_progress", methods=["GET"])
def list_in_progress():
    try:
        items, sha = load_in_progress()
        pruned = _prune_stale_in_progress(items)
        if len(pruned) != len(items):
            try:
                save_in_progress(pruned, sha)
            except DataWriteError as e:
                print(f"[WARN] in_progress の自動間引き保存に失敗しました: {e}")
        return jsonify({"ok": True, "items": pruned})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- **一覧取得（GET、本来「読むだけ」のはずの操作）のたびに、間引きが実際に行われた場合は書き込みも発生する**、という珍しい構造です。もし間引きの結果、件数が実際に減っていれば（`len(pruned) != len(items)`）、その場で保存し直します。保存に失敗しても、`print`で警告するだけで、レスポンス自体は`pruned`（間引き後の一覧）をそのまま返します。次に誰かがこのAPIを呼んだときに、また同じ間引きが試みられるため、多少の失敗があっても実害はほとんどありません。

## 3. 登録・更新・削除（5605〜5692行）

```python
@app.route("/register_in_progress", methods=["POST"])
def register_in_progress():
    """
    body: { id, name, subject, folder_id, creator_id, creator_nickname }
    ・id はフロント側で生成しているデッキのローカルID（他人と衝突しない前提）。
    ・同じ id で既にエントリがある場合は上書きする（念のため）。
    """
    ...
    entry = {
        "id": draft_id, "name": name, "subject": data.get("subject"),
        "folder_id": data.get("folder_id"), "creator_id": data.get("creator_id"),
        "creator_nickname": creator_nickname,
        "created_at": datetime.now(JST).strftime("%Y-%m-%d %H:%M:%S"),
    }
    try:
        items, sha = load_in_progress()
        items = [it for it in items if it.get("id") != draft_id]
        items.append(entry)
        save_in_progress(items, sha)
        return jsonify({"ok": True})
    ...
```
- 注目すべき点は、**このAPIには`require_login_json`によるログイン確認がありません**。`creator_id`/`creator_nickname`はクライアントからの自己申告のまま使われています。これは、[[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md]]の`/save_cards`と同様に、「作成中」という段階の性質上、厳密な本人確認よりも気軽に使えることを優先した設計だと考えられます（実際に保存される内容も、カードの中身を含まない軽量なメタ情報だけです）。
- `items = [it for it in items if it.get("id") != draft_id]; items.append(entry)`… 同じ`id`の既存エントリを取り除いてから新しいものを追加する、という「削除してから追加」のやり方で upsert（無ければ挿入・あれば更新）を実現しています。

```python
@app.route("/update_in_progress", methods=["POST"])
def update_in_progress():
    """
    該当エントリが無ければ（既に公開済み・削除済みなど）何もせず ok:true を返す。
    """
    ...
    for it in items:
        if it.get("id") == draft_id:
            if "name" in data:      it["name"]      = data["name"]
            if "subject" in data:   it["subject"]   = data["subject"]
            if "folder_id" in data: it["folder_id"] = data["folder_id"]
            found = True
            break
    if found:
        save_in_progress(items, sha)
    return jsonify({"ok": True, "found": found})
```
- `if "name" in data:`のような書き方で、**送られてきたフィールドだけを部分的に更新**します（`data.get("name")`ではなく`"name" in data`を使っているのは、「そもそも送られてこなかった」のか「明示的に空文字列を送ってきた」のかを区別するためです）。
- コメントにある通り、該当エントリが見つからなくても（既に公開済み・削除済みなどで、そもそも「作成中」ではなくなっている場合）エラーにはせず、`found: false`と共に成功扱いで返します。フロント側の楽観的な更新処理（サーバーの状態を細かく気にせず、とりあえず更新を試みる）が、タイミングの問題でエラー表示になってしまわないようにする配慮です。

```python
@app.route("/remove_in_progress", methods=["POST"])
def remove_in_progress():
    """body: { id } — 公開された・削除された・非公開のまま維持することにした等で不要になったエントリを消す。"""
    ...
    new_items = [it for it in items if it.get("id") != draft_id]
    if len(new_items) != len(items):
        save_in_progress(new_items, sha)
    return jsonify({"ok": True})
```
- これも、[[../10_FlaskAPI_時間割/00_時間割の上書きデータとCRUD_API.md]]の`/delete_timetable`などと同じ、存在しなくても静かに成功扱いになる**冪等な削除**です。実際に何か削除された場合だけ保存し直す、という最適化も同じパターンです。

---

次は「お知らせ（notices）」のAPIを解説します。 → [[../17_FlaskAPI_お知らせ/00_お知らせ一覧と詳細取得投稿API.md]]
