# デッキ・フォルダの並び順管理（`bot.py` 6244〜6327行）

対象：`bot.py`の「Flask API — 一覧（デッキ・フォルダ）の並び順（みんなで共有）」セクション。これで「Flask API — 単語カード」に関連する一連の章（データ層・一覧保存・削除・フォルダ・並び順）が全て終わります。

## 設計方針（コメントより）

- フォルダを開いている場所（`"__root__"`またはフォルダid）ごとに、その中でのフォルダ・公開済みデッキの並び順（`data-key`の配列）を保存します。
- 未公開（各自の下書き）デッキは他人からは見えないデータなので、ここには含めません（フロント側でも送らないようにフィルタしています）。

## 1. データの読み書きと後片付け（6251〜6297行）

```python
ORDER_FILE = "list_order.json"

def load_list_order():
    data, sha = local_get(ORDER_FILE)
    return (data or {}), sha

def save_list_order(order_map, sha=None):
    if sha is None:
        _, sha = local_get(ORDER_FILE)
    local_put(ORDER_FILE, order_map, sha)
    notify_change()
```
- `list_order.json`は`{スコープ: [並び順のキーの配列]}`という辞書です。「スコープ」とは「今どのフォルダを開いているか」（ルートなら`"__root__"`、そうでなければフォルダのID）を表し、その中に含まれるフォルダ・デッキが、どの順番で並んでいるかを、`"folder:xxx"`や`"deck:yyy.json"`のような、種類の接頭辞付きキーの配列として保持します。

```python
def cleanup_list_order(remove_keys=None, remove_scopes=None):
    """
    ★ 並び順の掃除は本質的な機能ではない（古い項目が残っていても、フロント側の表示時に
       存在しないものとして自動的に無視されるだけ）ので、失敗しても警告に留め、
       呼び出し元の本来の削除処理自体は失敗させない。
    """
    remove_keys   = set(remove_keys or [])
    remove_scopes = set(remove_scopes or [])
    if not remove_keys and not remove_scopes:
        return
    try:
        order_map, sha = load_list_order()
        changed = False
        for scope in remove_scopes:
            if scope in order_map:
                del order_map[scope]
                changed = True
        if remove_keys:
            for scope, keys in list(order_map.items()):
                new_keys = [k for k in keys if k not in remove_keys]
                if len(new_keys) != len(keys):
                    order_map[scope] = new_keys
                    changed = True
        if changed:
            save_list_order(order_map, sha)
    except DataWriteError as e:
        print(f"[WARN] list_order のクリーンアップに失敗しました: {e}")
    except Exception as e:
        print(f"[WARN] list_order のクリーンアップ中に予期しないエラーが発生しました: {e}")
```
- これが、[00_フォルダのツリー構造と操作API.md](00_フォルダのツリー構造と操作API.md)の`/delete_folder`や[../14_FlaskAPI_CardMaker/02_デッキ削除と作成者確認.md](../14_FlaskAPI_CardMaker/02_デッキ削除と作成者確認.md)の`_delete_card_deck_file`から呼ばれていた、並び順の後片付け役です。2種類の掃除を行います。
  1. `remove_scopes`（丸ごと削除するスコープ自体）… フォルダそのものが削除された場合、そのフォルダの**中**の並び順情報はもう意味を持たないため、スコープごと消します。
  2. `remove_keys`（各スコープの並び順配列から取り除く要素）… 削除されたデッキやサブフォルダへの参照が、**まだ存在している別のスコープの並び順の中**に残っていれば、それも取り除きます。
- コメントで強調されている通り、「並び順の掃除は本質的な機能ではない」という位置づけです。もし古い参照が掃除されずに残っていても、フロント側は表示の際にそのキーに対応する実体（デッキやフォルダ）が見つからなければ、単に無視するだけで、画面が壊れることはありません。そのため、掃除自体が失敗しても、呼び出し元の本来の削除処理（デッキやフォルダの削除そのもの）を失敗させることはなく、警告を出すだけに留めています。**「無くても困らない付随的な処理」を、失敗に対して寛容に扱う**という、これまでの回でも繰り返し見てきた設計方針の、もう1つの実例です。

## 2. `/list_order`・`/save_order`：並び順の取得と保存（6299〜6326行）

```python
@app.route("/list_order", methods=["GET"])
def list_order():
    order_map, _ = load_list_order()
    return jsonify({"ok": True, "order": order_map})

@app.route("/save_order", methods=["POST"])
def save_order():
    """
    body: { scope: "__root__" または フォルダid, keys: ["folder:xxx", "deck:yyy", ...] }
    指定したscope（フォルダの場所）の並び順だけを丸ごと置き換えて保存する。
    """
    data  = request.json or {}
    scope = data.get("scope")
    keys  = data.get("keys")
    if not scope or not isinstance(keys, list):
        return jsonify({"ok": False, "error": "scope と keys は必須です"})
    try:
        order_map, sha = load_list_order()
        order_map[scope] = keys
        save_list_order(order_map, sha)
        return jsonify({"ok": True})
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- `/list_order`… 並び順マップ全体を、ログイン不要でそのまま返します。
- `/save_order`… **注目すべき点は、他のCRUD系APIと違って`require_login_json`によるログイン確認が行われていないことです**。並び順は「デッキ・フォルダそのものの内容」ではなく、単なる表示上の並べ方であり、間違って書き換えられても実害が比較的小さい（デッキやお知らせが消える・改ざんされるのとは性質が異なる）ため、他の変更系APIほど厳格な保護が必要とされていないのだと考えられます。
- `order_map[scope] = keys`… 1つのスコープの並び順を、**丸ごと置き換えます**（部分的な追加・削除ではなく、常に配列全体を新しいものに差し替える方式）。これは、フロント側でドラッグ&ドロップによる並び替えが完了したタイミングで、その場所の並び順全体を1回のAPI呼び出しで送信する、というシンプルな設計に対応しています。
- また、他のAPIで一般的に見られる不正文字チェック（`reject_if_bug_chars`）も、このAPIには存在しません。`scope`と`keys`の中身は、いずれもシステムが発行したID（フォルダID・デッキのファイル名）の組み合わせであり、利用者が自由に入力するテキストが含まれないため、不正文字チェックの対象にする必要が無いという判断です。

---

これで「Flask API — 単語カード」に関わる一連の章（データ層・一覧取得・保存・削除・フォルダ・並び順）が全て終わりました。次は、勉強ログの「わからないマーク」「続きから」「完了記録」といった、生徒ごとの学習進捗データを端末をまたいで共有する仕組みを解説します。 → [../19_FlaskAPI_学習進捗データ/00_わからないマークと進捗の端末間共有.md](../19_FlaskAPI_学習進捗データ/00_わからないマークと進捗の端末間共有.md)
