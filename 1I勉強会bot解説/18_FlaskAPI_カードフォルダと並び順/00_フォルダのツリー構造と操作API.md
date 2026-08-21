# カードのフォルダ機能（`bot.py` 6183〜6421行）

対象：`bot.py`の「Flask API — カードのフォルダ（みんなで共有）」セクションと「『クイズ過去問』フォルダ」セクション。[../../1I勉強会web解説/02_Cardmaker](../../1I勉強会web解説/02_Cardmaker/00_HTML構造とページ全体像.md)で見たCardMakerのフォルダ階層の、サーバー側実装です。

## 1. データの形と運用ログ整形（6183〜6208行）

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

## 2. フォルダ階層のツリー演算（6210〜6243行）

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

## 3. 「クイズ過去問」フォルダという特別な仕組み（6245〜6315行）

コメントの通り、これは[../15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md](../15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md)の`_archive_manual_quiz`が使う、システムが自動管理する固定のフォルダです。

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
- `QUIZ_ARCHIVE_FOLDER_ID`が**固定の文字列**（`generate_folder_id`のようなランダム生成ではない）である理由：「IDを固定にすることで、二重作成を防ぎ、どのルートからでも同じフォルダを指せる」ため。`_ensure_quiz_archive_folder`は、このIDのフォルダが既に存在するかを確認し、無ければ作る、という「無ければ作る」パターンで、何度呼ばれても複数のフォルダが重複して作られることはありません。
- **★ 「このデッキがクイズ過去問デッキかどうか」の判定方法は2026/08/21に変わりました**。以前は専用のフラグを持たせず、`folder_id`がこのフォルダのスコープ内（`_is_in_archive_scope`）かどうかだけで判定していました（『`save_cards`の`card_payload`は固定6キーのため、任意のトップレベルフラグを追加すると通常デッキの保存経路にも影響が及んでしまうのを避けるため』という理由）。この方式は「デッキ自体をこのフォルダの外へは一切移動できない」という制約とセットで初めて成立するもので（位置だけで判定する以上、位置が動くと判定できなくなるため）、ユーザーから「外のフォルダにも移動できるようにしたいが、問題の編集はできないままにしたい」という要望を受けたことで、この前提が崩れました。そこで、[../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)で見た`quiz_archive`という専用フラグをデッキ自身に持たせる方式に切り替え、「位置に依存しない性質」として扱えるようにしました。

### 3-1. 既存データの移行：`_migrate_legacy_quiz_archive_decks`（6283〜6315行）

```python
def _migrate_legacy_quiz_archive_decks():
    try:
        folders, _ = load_card_folders()
        for f in list_card_files():
            filename = f["name"]
            data, sha = get_card_file(filename)
            if not data or data.get("quiz_archive"):
                continue
            if not data.get("choice_mode"):
                continue
            if not _is_in_archive_scope(folders, data.get("folder_id")):
                continue
            data["quiz_archive"] = True
            try:
                put_card_file(filename, data, sha)
                upsert_cards_index_entry(filename, data)
                print(f"[INFO] 「{data.get('name')}」に quiz_archive フラグを補いました（{filename}、既存データの移行）")
            except DataWriteError as e:
                print(f"[WARN] {filename} への quiz_archive フラグ移行に失敗しました: {e}")
    except Exception as e:
        print(f"[WARN] クイズ過去問デッキの移行処理に失敗しました: {e}")
```
- `quiz_archive`フラグを新設しても、**そのフラグが導入される前から既に存在していたデッキ**にはまだ付いていません。そのままだと、既存の（本物の）クイズ過去問デッキが、新しい判定方法では「クイズ過去問デッキではない」と誤判定されてしまいます（編集できてしまう・バッジが消える、など）。これを補うための、一度きりの移行処理です。
- **判定の根拠**がコメントで説明されています：この移行が実行される時点で「クイズ過去問フォルダのスコープ内にあり、かつ選択式デッキ（`choice_mode`あり）」なデッキは、ほぼ間違いなく`_archive_manual_quiz`が自動保存したものだと言えます。なぜなら、以前の制限（このフォルダの外へは絶対に移動できない）が効いていた期間中、この条件を満たすデッキを作るには、わざわざシステムフォルダの中へ自分の選択式デッキを移動する、という現実的にまず起きない操作が必要だったからです。
- `if not data or data.get("quiz_archive"): continue`… 既にフラグが付いているデッキ（この移行が過去に実行済み、またはこのフラグ導入後に新規作成されたデッキ）はスキップします。これにより、この関数は**何度実行しても安全**（冪等）です。1回移行してしまえば、次回以降の起動では対象デッキが見つからず、ほぼ何もせず即座に終わります。
- 呼び出し元は`on_ready`（[../21_スケジューラーとBot起動/00_ジョブ登録とBotの起動リトライ.md](../21_スケジューラーとBot起動/00_ジョブ登録とBotの起動リトライ.md)参照）で、`scheduler.start()`と同じ「プロセス起動後に1回だけ」というガード（`started`フラグ）の中から呼ばれます。

## 4. フォルダ操作のAPI（6317〜6421行）

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
    """
    新規作成: { name, parent_id }
    改名／移動: { id, name, parent_id }
    """
    data      = request.json or {}
    _guild_id, _student_id, nickname, err = require_login_json(data)  # ★ 変更にはログイン必須
    if err:
        return err
    folder_id = data.get("id")
    name      = (data.get("name") or "").strip()
    parent_id = data.get("parent_id")

    if not name:
        return jsonify({"ok": False, "error": "name は必須です"})

    err = reject_if_bug_chars({"フォルダ名": name})
    if err:
        return err

    try:
        folders, sha = load_card_folders()
        old_folders_text = _folders_text(folders)  # ★ 運用ログでファイル全体の差分を見せるため、変更前に控えておく

        if folder_id:
            target = next((f for f in folders if f["id"] == folder_id), None)
            if not target:
                return jsonify({"ok": False, "error": "folder not found"})
            # ★「クイズ過去問」フォルダ自身は改名・移動できないシステムフォルダ
            if folder_id == QUIZ_ARCHIVE_FOLDER_ID and (name != target.get("name") or parent_id != target.get("parent_id")):
                return jsonify({"ok": False, "error": "このフォルダは変更できません"})
            if parent_id != target.get("parent_id"):
                if not _can_move_folder_to(folders, folder_id, parent_id):
                    return jsonify({"ok": False, "error": "移動できません（3階層を超える、または循環参照）"})
                # ★「クイズ過去問」フォルダの中身は、その外へ移動できない
                if _is_in_archive_scope(folders, folder_id) and not _is_in_archive_scope(folders, parent_id):
                    return jsonify({"ok": False, "error": "クイズ過去問フォルダの外には移動できません"})
                target["parent_id"] = parent_id
            target["name"] = name
        else:
            if _folder_level(folders, parent_id) >= MAX_FOLDER_DEPTH:
                return jsonify({"ok": False, "error": f"フォルダは{MAX_FOLDER_DEPTH}階層までしか作成できません"})
            folder_id = generate_folder_id()
            folders.append({"id": folder_id, "name": name, "parent_id": parent_id})

        save_card_folders(folders, sha)
        change = file_diff(FOLDERS_FILE, old_folders_text, _folders_text(folders))
        log_event("card", f"フォルダ「{name}」を保存しました。", actor=nickname, detail=[change] if change else None)
        return jsonify({"ok": True, "id": folder_id})
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- `id`が送られてきたかどうかで、既存フォルダの「改名／移動」か「新規作成」かを1つのAPIで兼ねています（これまでのCRUD系APIと同じパターンです）。
- **改名・移動の分岐**：`QUIZ_ARCHIVE_FOLDER_ID`自身は、名前も親も変更できないようガードされています（`name != target.get("name") or parent_id != target.get("parent_id")`で「何か実際に変えようとしているか」を確認し、変えようとしていなければ通す＝単なる無変化の保存リクエストまでは拒否しません）。
- `parent_id != target.get("parent_id"):`（親が実際に変わる場合だけ）に、`_can_move_folder_to`（前述のツリー演算）と`_is_in_archive_scope`（クイズ過去問フォルダの**中のサブフォルダ**を、その外へ出さないための制約）の両方を確認します。
  ★ 注意（2026/08/21）：この`_is_in_archive_scope`はあくまで**フォルダ自体**（サブフォルダ）の移動制限です。以前は[../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)の`/save_cards`にも、これと対になる**デッキの移動制限**（クイズ過去問デッキ自体をこのフォルダの外へ出せない）がありましたが、そちらはユーザーの要望で撤廃され、代わりに「デッキの中身は編集できない」という別の制限（`quiz_archive`フラグ）に置き換わっています。つまり現在は、**フォルダ（サブフォルダ）の移動は引き続き禁止**・**デッキの移動は許可**という、非対称な扱いになっています。
- **新規作成の分岐**：`_folder_level(folders, parent_id) >= MAX_FOLDER_DEPTH`で、これから作るフォルダの深さが既に上限に達していないかを確認します。

```python
@app.route("/delete_folder", methods=["POST"])
def delete_folder():
    data      = request.json or {}
    _guild_id, _student_id, nickname, err = require_login_json(data)  # ★ 変更にはログイン必須
    if err:
        return err
    folder_id = data.get("id")
    if not folder_id:
        return jsonify({"ok": False, "error": "id は必須です"})
    # ★「クイズ過去問」フォルダ自身は削除できないシステムフォルダ
    if folder_id == QUIZ_ARCHIVE_FOLDER_ID:
        return jsonify({"ok": False, "error": "このフォルダは削除できません"})
    try:
        folders, sha = load_card_folders()
        # ★ 削除前に名前を控えておく（運用ログの詳細表示用。IDだけでは
        #   何のフォルダだったか分からないため）
        deleted_folder = next((f for f in folders if f["id"] == folder_id), None)
        desc_ids   = [f["id"] for f in _folder_descendants(folders, folder_id)]
        remove_ids = set([folder_id] + desc_ids)
        new_folders = [f for f in folders if f["id"] not in remove_ids]
        save_card_folders(new_folders, sha)

        # ★ 並び順（list_order.json）からも、削除したフォルダ自身のスコープと、
        #   他のフォルダ内に残っていた folder: キーの参照を取り除いておく
        cleanup_list_order(
            remove_keys=set(f"folder:{fid}" for fid in remove_ids),
            remove_scopes=remove_ids,
        )

        deleted_folder_name = (deleted_folder or {}).get("name")
        change = file_diff(FOLDERS_FILE, _folders_text(folders), _folders_text(new_folders))
        log_event(
            "card",
            f"フォルダ「{deleted_folder_name}」を削除しました。" if deleted_folder_name else "フォルダを削除しました。",
            actor=nickname,
            detail=[change] if change else None,
        )
        return jsonify({"ok": True, "deleted_ids": list(remove_ids)})
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- `remove_ids = set([folder_id] + desc_ids)`… **フォルダを削除すると、その中の全てのサブフォルダも道連れで削除されます**（カスケード削除）。`_folder_descendants`で子孫を全部洗い出し、自分自身と合わせて一気に取り除きます。
- 削除される全フォルダのIDが、レスポンスの`deleted_ids`として返されます。フロント側はこれを使って、削除されたフォルダの中に入っていたデッキ（フォルダ自体は消えても、中のデッキ自体は別のファイルなので消えません）を、画面上でどう扱うか（例えばルートに戻す、など）を判断できます。
- `cleanup_list_order(...)`（次のファイルで解説）… 削除された全フォルダに関連する並び順情報も、まとめて後片付けします。

---

次は、この`folders.json`を含む、デッキ・フォルダの並び順（`list_order.json`）を管理するAPIを解説します。 → [01_並び順の管理API.md](01_並び順の管理API.md)
