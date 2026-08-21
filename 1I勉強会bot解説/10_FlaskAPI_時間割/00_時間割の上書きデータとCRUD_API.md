# Flask API — 時間割（`bot.py` 2703〜2892行）

対象：`bot.py`の「Flask API — 時間割」セクション前半（時間割の「上書き」データ）。[../../1I勉強会web解説/03_Timetable](../../1I勉強会web解説/03_Timetable/00_HTML構造とページ全体像.md)で解説した時間割ページの、サーバー側の実装です。

## データの形

```python
def load_timetable(guild_id: int):
    data, _ = local_get(f"timetable_{guild_id}.json")
    return data or {}

def save_timetable(guild_id: int, data: dict):
    _, sha = local_get(f"timetable_{guild_id}.json")
    local_put(f"timetable_{guild_id}.json", data, sha)
    notify_change(guild_id)
```
- `timetable_{guild_id}.json`は、デスクトップの設計資料にある通り「通常時間割からの**上書き**」だけを持つファイルです。曜日ごとの基本の時間割そのもの（後述の`terms_{guild_id}.json`）とは別で、「この日のこの時限だけ休講になった」「この日のこの時限だけ別の科目に変わった」といった**例外**だけを、`{キー: 内容}`という辞書の形で持っています。
- `_TIMETABLE_TYPE_LABELS = {"change": "授業変更", "holiday": "休校", "period_holiday": "1コマ休み"}`… 上書きには3種類あります。「授業変更」（ある1コマの科目・持ち物が変わる）、「休校」（その日全体が休み）、「1コマ休み」（特定の1コマだけが休み）です。

```python
def _timetable_entry_lines(e):
    t = e.get("type")
    fields = [
        ("種別", _TIMETABLE_TYPE_LABELS.get(t, t)),
        ("日付", e.get("date")),
    ]
    if t in ("change", "period_holiday"):
        fields.append(("時限", f"{e.get('period')}限"))
    if t == "change":
        fields.append(("科目", e.get("subject")))
        fields.append(("持ち物", "、".join(e.get("items") or []) or "(なし)"))
    else:
        fields.append(("理由", e.get("reason")))
    fields.append(("備考", e.get("note") or "(なし)"))
    return _json_block(fields)

def _timetable_text(tt):
    lines = []
    for e in (tt or {}).values():
        lines.extend(_timetable_entry_lines(e))
    return "\n".join(lines)
```
- 運用ログ用の整形関数です。種類（`type`）によって表示するフィールドを出し分けています。`"、".join(e.get("items") or []) or "(なし)"`… 持ち物のリストを日本語の読点`、`で連結し、もし空リストなら`"(なし)"`と表示します。

## エンドポイント

```python
@app.route("/list_timetable", methods=["GET"])
def list_timetable():
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    data = load_timetable(int(guild_id))
    overrides = [{"key": k, **v} for k, v in data.items()]
    return jsonify({"ok": True, "overrides": overrides})
```
- `/list_timetable`… ログイン確認が無い、閲覧専用のGET APIです。`{"key": k, **v}`という書き方は、辞書のキー`k`を`"key"`フィールドとして明示的に加えつつ、元の値`v`（辞書）の中身を`**v`で展開して1つの辞書にまとめる、という書き方です（`**`は辞書の中身を展開して合成するPythonの構文です）。これにより、フロント側は「キーが何だったか」を各エントリの中身から直接読み取れます。

```python
@app.route("/update_timetable", methods=["POST"])
def update_timetable():
    ...
    tt = load_timetable(guild_id)
    old_tt_text = _timetable_text(tt)
    tt[key] = {
        "key": key, "type": "change", "date": data.get("date"),
        "period": data.get("period"), "subject": data.get("subject"),
        "items": data.get("items", []), "note": data.get("note", ""),
    }
    ...
```
- `/update_timetable`（授業変更）・`/set_holiday`（休校）・`/set_period_holiday`（1コマ休み）・`/delete_timetable`（削除）の4つは、いずれも同じパターンの繰り返しです。
  1. `require_login_json`でログイン確認。
  2. `reject_if_bug_chars`で不正文字チェック（`update_timetable`のみ。他の3つは自由記述のテキスト項目が少ないため省略されています）。
  3. `_timetable_text`で変更前の状態を控える。
  4. `tt[key] = {...}`で、指定された`key`（時間割上のどのコマかを表す識別子。フロント側が組み立てて送ってくる文字列）に対応するエントリを、新しい内容で**丸ごと上書き**する（部分更新ではなく全部差し替え）。
  5. `save_timetable`で保存。
  6. `write_log`（簡易ログ）と`log_event`（運用ログ、`file_diff`による差分付き）の両方に記録。
- `/set_period_holiday`のコメントには、実際にあった不具合の経緯が残っています：「これまでこのエンドポイントが未実装だったため、フロント側（Timetable.js）が保存に失敗してもエラーを握りつぶしてしまい、`localStorage`にしか残らず『他の端末では反映されない／たまに消える』原因になっていた」。つまり、以前はこのAPI自体が存在せず、フロント側がその保存失敗を静かに無視してブラウザだけのローカル保存で誤魔化していたため、見た目上は保存できたように見えても、実際には他の端末や他の生徒には全く共有されていなかった、という不具合です。この行が追加されたことで、`/set_holiday`と同じ要領でサーバー側にきちんと保存されるようになり、問題が解消されています。

```python
@app.route("/delete_timetable", methods=["POST"])
def delete_timetable():
    ...
    tt = load_timetable(guild_id)
    if key in tt:
        old_entry = tt[key]
        old_tt_text = _timetable_text(tt)
        del tt[key]
        try:
            save_timetable(guild_id, tt)
        except DataWriteError as e:
            return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
        ...
    return jsonify({"ok": True})
```
- `/delete_timetable`… `if key in tt:`で、そもそもそのキーが存在する場合だけ削除処理を行います。存在しない場合でも`return jsonify({"ok": True})`（成功扱い）で終わる点に注目してください。これは「削除しようとしたときには既に他の誰かが削除済みだった」というケースを、エラーではなく**結果として同じ状態（そのキーが存在しない）に既になっている**ため、素直に成功として扱う、という寛容な設計です（冪等性：同じ削除操作を何度呼んでも、最終的な状態は変わらない）。

---

次は、学期ごとの基本時間割（曜日×時限の科目表そのもの）を管理するAPIを解説します。 → [01_学期の基本時間割API.md](01_学期の基本時間割API.md)
