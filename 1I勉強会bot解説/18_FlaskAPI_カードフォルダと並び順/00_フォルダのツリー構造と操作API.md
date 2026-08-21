# カードのフォルダ機能（`bot.py` 6042〜6242行）

対象：`bot.py`の「Flask API — カードのフォルダ（みんなで共有）」セクションと「『クイズ過去問』フォルダ」セクション。[[../../1I勉強会web解説/02_Cardmaker]]で見たCardMakerのフォルダ階層の、サーバー側実装です。

## 1. データの形と運用ログ整形（6045〜6070行）

```python
FOLDERS_FILE = "folders.json"
MAX_FOLDER_DEPTH = 3

def load_card_folders():
    data, sha = local_get(FOLDERS_FILE)
    return (data or []), sha

def save_card_folders(folders, sha=None):
    if sha is None:
        _, sha = local_get(FOLDERS_FILE)
    local_put(FOLDERS_FILE, folders, sha)
    notify_change()
```
- `folders.json`は`[{id, name, parent_id}, ...]`という**フラットな配列**でフォルダの階層構造を表現します。各フォルダは自分の`parent_id`（親フォルダのID。最上位なら`None`）だけを知っており、木構造そのもの（親から子への逆参照リストなど）は持っていません。木構造の解釈は、これから見る関数群が配列全体を毎回走査することで行います。
- `MAX_FOLDER_DEPTH = 3`… フォルダは最大3階層までしか作れません。

```python
def _folders_text(folders):
    """運用ログ用：folders.json（全フォルダ一覧）を { ... } のブロックの並びにする。"""
    folders = folders or []
    by_id = {f.get("id"): f.get("name") for f in folders}
    lines = []
    for f in folders:
        parent_id = f.get("parent_id")
        fields = [
            ("フォルダ名", f.get('name')),
            ("親フォルダ", by_id.get(parent_id, parent_id) if parent_id else "(なし・最上位)"),
        ]
        lines.extend(_json_block(fields))
    return "\n".join(lines)
```
- `by_id = {f.get("id"): f.get("name") for f in folders}`… IDから名前を引く対応表をその場で作り、「親フォルダ」の表示に、無機質なIDではなく人間が読める名前を使えるようにしています。

## 2. フォルダ階層のツリー演算（6072〜6105行）

これが、フラットな配列から「階層」という概念を導き出す、**この機能の数学的な核心**です。この一群の関数は全て汎用的に設計されており、特定の機能に依存していません。

```python
def _folder_level(folders, folder_id):
    lvl = 0
    cur = next((f for f in folders if f["id"] == folder_id), None)
    while cur:
        lvl += 1
        cur = next((f for f in folders if f["id"] == cur.get("parent_id")), None)
    return lvl
```
- 指定したフォルダが、最上位から数えて何階層目にあるかを計算します。`while cur:`で、親を辿れなくなるまで（`parent_id`が`None`になり`cur`が`None`になるまで）1段ずつ遡り、辿った回数が階層の深さになります。`folder_id`が`None`（最上位そのもの）なら、最初の`next(...)`が`None`を返し、ループが1度も回らないため`0`が返ります。

```python
def _folder_descendants(folders, folder_id):
    direct = [f for f in folders if f.get("parent_id") == folder_id]
    all_desc = list(direct)
    for f in direct:
        all_desc += _folder_descendants(folders, f["id"])
    return all_desc
```
- 指定したフォルダの**全ての子孫**（子・孫・ひ孫…）を集めます。`direct`（直接の子）を求めた後、それぞれの子について**自分自身を再帰的に呼び出し**、その子孫たちも合わせて集めています。これは木構造の探索でよく使われる「再帰」というテクニックの典型例です。

```python
def _max_level_in_subtree(folders, folder_id):
    desc = _folder_descendants(folders, folder_id)
    levels = [_folder_level(folders, folder_id)] + [_folder_level(folders, f["id"]) for f in desc]
    return max(levels)
```
- あるフォルダとその子孫全体（サブツリー全体）の中で、最も深い階層がどこまであるかを求めます。自分自身の深さと、全子孫それぞれの深さを集めたリストから、最大値を取ります。

```python
def _can_move_folder_to(folders, folder_id, new_parent_id):
    if folder_id == new_parent_id:
        return False
    desc_ids = [f["id"] for f in _folder_descendants(folders, folder_id)]
    if new_parent_id and new_parent_id in desc_ids:
        return False
    old_level = _folder_level(folders, folder_id)
    new_level = _folder_level(folders, new_parent_id) + 1
    shift = new_level - old_level
    return (_max_level_in_subtree(folders, folder_id) + shift) <= MAX_FOLDER_DEPTH
```
- **これが移動操作の安全性を保証する、最も重要な関数です**。あるフォルダを新しい親（`new_parent_id`）の下に移動できるかどうかを判定します。2つの異なる問題を防いでいます。
  1. **循環参照の防止**：`if new_parent_id and new_parent_id in desc_ids: return False`。もし「自分の子孫」を自分の新しい親にしてしまうと、親子関係がループしてしまい（例えば、Aフォルダの中にBフォルダがあるのに、AをBの中に移動してしまう）、木構造そのものが壊れてしまいます。`_folder_descendants`で調べた自分の子孫一覧に、移動先が含まれていないかを確認しています。`folder_id == new_parent_id`（自分自身を自分の親にしようとする、最も単純な循環）も別途チェックしています。
  2. **深さ制限の超過防止**：`shift = new_level - old_level`（移動によって、階層の深さが何段変化するか）を計算し、`_max_level_in_subtree`（移動するフォルダの、今のサブツリー全体の最大の深さ）に`shift`を足した結果が、`MAX_FOLDER_DEPTH`（3）を超えないかを確認します。**これが重要な理由は、移動するフォルダ自身の深さだけでなく、その中に入っている子孫フォルダの深さも一緒に動く**ためです。もし1階層のフォルダを、既に2階層目にある別のフォルダの下に移動しようとしたら、そのフォルダの中に3階層目の孫フォルダがあった場合、移動後には4階層目になってしまい、制限を超えてしまいます。この関数は、移動対象のサブツリー全体を考慮して、この超過を未然に防ぎます。

```python
def generate_folder_id():
    import string
    return ''.join(random.choices(string.ascii_lowercase + string.digits, k=10))
```
- 新しいフォルダのIDを、10文字のランダムな英数字で発行します。

## 3. 「クイズ過去問」フォルダという特別な仕組み（6107〜6136行）

コメントの通り、これは[[../15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md]]の`_archive_manual_quiz`が使う、システムが自動管理する固定のフォルダです。

```python
QUIZ_ARCHIVE_FOLDER_ID   = "quiz_archive_root"
QUIZ_ARCHIVE_FOLDER_NAME = "クイズ過去問"

def _ensure_quiz_archive_folder():
    """「クイズ過去問」フォルダが無ければ作る（あれば何もしない）。"""
    folders, sha = load_card_folders()
    if not any(f.get("id") == QUIZ_ARCHIVE_FOLDER_ID for f in folders):
        folders.append({"id": QUIZ_ARCHIVE_FOLDER_ID, "name": QUIZ_ARCHIVE_FOLDER_NAME, "parent_id": None})
        save_card_folders(folders, sha)

def _is_in_archive_scope(folders, folder_id):
    """folder_id が「クイズ過去問」フォルダ自身、またはその子孫かどうか。"""
    if folder_id == QUIZ_ARCHIVE_FOLDER_ID:
        return True
    return any(f["id"] == folder_id for f in _folder_descendants(folders, QUIZ_ARCHIVE_FOLDER_ID))
```
- コメントにある設計上の工夫が2つあります。
  1. `QUIZ_ARCHIVE_FOLDER_ID`が**固定の文字列**（`generate_folder_id`のようなランダム生成ではない）である理由：「IDを固定にすることで、二重作成を防ぎ、どのルートからでも同じフォルダを指せる」ため。`_ensure_quiz_archive_folder`は、このIDのフォルダが既に存在するかを確認し、無ければ作る、という「無ければ作る」パターンで、何度呼ばれても複数のフォルダが重複して作られることはありません。
  2. 「このデッキが4択アーカイブかどうか」を専用のフラグとして持たせず、`folder_id`がこのフォルダのスコープ内かどうかだけで判定する理由：「`save_cards`の`card_payload`は固定6キーのため、任意のトップレベルフラグを追加すると通常デッキの保存経路にも影響が及んでしまうのを避けるため」。既存の保存処理のデータ構造を変えずに、**位置（どのフォルダに入っているか）だけで意味を持たせる**ことで、新機能の追加が既存機能に波及するリスクを避けています。

## 4. フォルダ操作のAPI（6138〜6242行）

```python
@app.route("/list_folders", methods=["GET"])
def list_folders():
    folders, _ = load_card_folders()
    return jsonify({"ok": True, "folders": folders})
```
- 単純に`folders.json`の中身をそのまま返します。

```python
@app.route("/save_folder", methods=["POST"])
def save_folder():
    """新規作成: { name, parent_id } / 改名／移動: { id, name, parent_id }"""
    ...
    if folder_id:
        target = next((f for f in folders if f["id"] == folder_id), None)
        if not target:
            return jsonify({"ok": False, "error": "folder not found"})
        if folder_id == QUIZ_ARCHIVE_FOLDER_ID and (name != target.get("name") or parent_id != target.get("parent_id")):
            return jsonify({"ok": False, "error": "このフォルダは変更できません"})
        if parent_id != target.get("parent_id"):
            if not _can_move_folder_to(folders, folder_id, parent_id):
                return jsonify({"ok": False, "error": "移動できません（3階層を超える、または循環参照）"})
            if _is_in_archive_scope(folders, folder_id) and not _is_in_archive_scope(folders, parent_id):
                return jsonify({"ok": False, "error": "クイズ過去問フォルダの外には移動できません"})
            target["parent_id"] = parent_id
        target["name"] = name
    else:
        if _folder_level(folders, parent_id) >= MAX_FOLDER_DEPTH:
            return jsonify({"ok": False, "error": f"フォルダは{MAX_FOLDER_DEPTH}階層までしか作成できません"})
        folder_id = generate_folder_id()
        folders.append({"id": folder_id, "name": name, "parent_id": parent_id})
    ...
```
- `id`が送られてきたかどうかで、既存フォルダの「改名／移動」か「新規作成」かを1つのAPIで兼ねています（これまでのCRUD系APIと同じパターンです）。
- **改名・移動の分岐**：`QUIZ_ARCHIVE_FOLDER_ID`自身は、名前も親も変更できないようガードされています（`name != target.get("name") or parent_id != target.get("parent_id")`で「何か実際に変えようとしているか」を確認し、変えようとしていなければ通す＝単なる無変化の保存リクエストまでは拒否しません）。
- `parent_id != target.get("parent_id"):`（親が実際に変わる場合だけ）に、`_can_move_folder_to`（前述のツリー演算）と`_is_in_archive_scope`（クイズ過去問フォルダの外に出さない制約。[[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md]]の`/save_cards`で見たデッキの移動制限と対になる、フォルダ自体の移動制限です）の両方を確認します。
- **新規作成の分岐**：`_folder_level(folders, parent_id) >= MAX_FOLDER_DEPTH`で、これから作るフォルダの深さが既に上限に達していないかを確認します。

```python
@app.route("/delete_folder", methods=["POST"])
def delete_folder():
    ...
    if folder_id == QUIZ_ARCHIVE_FOLDER_ID:
        return jsonify({"ok": False, "error": "このフォルダは削除できません"})
    try:
        folders, sha = load_card_folders()
        deleted_folder = next((f for f in folders if f["id"] == folder_id), None)
        desc_ids   = [f["id"] for f in _folder_descendants(folders, folder_id)]
        remove_ids = set([folder_id] + desc_ids)
        new_folders = [f for f in folders if f["id"] not in remove_ids]
        save_card_folders(new_folders, sha)

        cleanup_list_order(
            remove_keys=set(f"folder:{fid}" for fid in remove_ids),
            remove_scopes=remove_ids,
        )
        ...
        return jsonify({"ok": True, "deleted_ids": list(remove_ids)})
```
- `remove_ids = set([folder_id] + desc_ids)`… **フォルダを削除すると、その中の全てのサブフォルダも道連れで削除されます**（カスケード削除）。`_folder_descendants`で子孫を全部洗い出し、自分自身と合わせて一気に取り除きます。
- 削除される全フォルダのIDが、レスポンスの`deleted_ids`として返されます。フロント側はこれを使って、削除されたフォルダの中に入っていたデッキ（フォルダ自体は消えても、中のデッキ自体は別のファイルなので消えません）を、画面上でどう扱うか（例えばルートに戻す、など）を判断できます。
- `cleanup_list_order(...)`（次のファイルで解説）… 削除された全フォルダに関連する並び順情報も、まとめて後片付けします。

---

次は、この`folders.json`を含む、デッキ・フォルダの並び順（`list_order.json`）を管理するAPIを解説します。 → [[01_並び順の管理API.md]]
