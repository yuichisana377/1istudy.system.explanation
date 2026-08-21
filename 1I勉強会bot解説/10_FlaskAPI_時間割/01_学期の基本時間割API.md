# Flask API — 学期ごとの基本時間割（`bot.py` 2893〜3031行）

対象：`bot.py`の「Flask API — 学期ごとの基本時間割（前期・後期など）」セクション。前回の「上書き」データとは別の、**曜日×時限そのものの基本時間割**を扱います。

## データの形（コメントより）

- 「前期」「後期」のように、期間ごとにまるごと違う基本時間割（曜日×時限の科目・持ち物）を切り替えられるようにするための機能です。
- 1件のデータ構造は`{ id, name, start_date, end_date, timetable: {mon:[...], ...} }`。
- `start_date`〜`end_date`の範囲に対象日が入っていれば、その学期の時間割がベースとして使われます（この判定自体はフロント側`Timetable.js`の`getTimetableForDate`が行います）。
- 前回見た`change`/`holiday`/`period_holiday`のオーバーライドは、**この学期のベース時間割の上にそのまま重ねて適用される**ので、前期のデータをいじらずに後期分を新規に追加・編集できます。

```python
def load_terms(guild_id: int):
    data, _ = local_get(f"terms_{guild_id}.json")
    return data or {}

def save_terms(guild_id: int, terms: dict):
    _, sha = local_get(f"terms_{guild_id}.json")
    local_put(f"terms_{guild_id}.json", terms, sha)
    notify_change(guild_id)
```
- `terms_{guild_id}.json`に、`{学期ID: 学期データ}`という辞書で保存されます。

## 運用ログ用の整形（2913〜2941行）

```python
_TERM_DAY_LABELS = {"mon": "月", "tue": "火", "wed": "水", "thu": "木", "fri": "金"}

def _term_lines(t):
    """
    ★ 2026/08/19、以前はコマ数の合計だけを出していて、曜日・時限ごとの
    科目を入れ替えてもdiffに出ない（コマ数が変わらなければ検知できない）
    抜けがあったため、曜日ごとに実際の科目を1フィールドで並べるよう修正。"""
    fields = [
        ("学期名", t.get('name')),
        ("開始日", t.get('start_date')),
        ("終了日", t.get('end_date')),
    ]
    tt = t.get("timetable") or {}
    for day_key, label in _TERM_DAY_LABELS.items():
        periods = tt.get(day_key) or []
        if not periods:
            continue
        subjects = "、".join(
            f"{i+1}限:{(p.get('subject') if isinstance(p, dict) else None) or '(空きコマ)'}"
            for i, p in enumerate(periods)
        )
        fields.append((f"{label}曜", subjects))
    return _json_block(fields)
```
- コメントに、これまでと同じく実際にあった不具合の修正経緯が残っています。以前の実装は「コマ数の合計」のような、ざっくりした要約しか運用ログに出していなかったため、**「月曜1限の科目を数学から英語に変える」のような、コマ数自体は変わらない編集**をしても、差分（diff）には何も現れないという抜けがありました。
- 修正後は、`for day_key, label in _TERM_DAY_LABELS.items():`で曜日ごとにループし、`enumerate(periods)`（各時限の番号`i`と中身`p`を同時に取り出す）で、その曜日の「1限:数学、2限:英語、3限:(空きコマ)」のような**具体的な科目名を並べた1行**を作ります。これにより、科目の入れ替えも確実に差分として検知できるようになっています。
- `(p.get('subject') if isinstance(p, dict) else None) or '(空きコマ)'`… その時限のデータが辞書でない（壊れている）場合や、科目名が設定されていない場合は、`'(空きコマ)'`と表示するフォールバックです。

## エンドポイント

```python
@app.route("/list_terms", methods=["GET"])
def list_terms():
    ...
    terms = load_terms(int(guild_id))
    return jsonify({"ok": True, "terms": list(terms.values())})
```
- `/list_terms`… 閲覧専用のGET APIです。`list(terms.values())`で、辞書の値の部分（学期データそのもの）だけを配列にして返します（キーである`term_id`は、各学期データ自身の`id`フィールドとして既に含まれているため、辞書のキーとしては返す必要がありません）。

```python
@app.route("/save_term", methods=["POST"])
def save_term():
    ...
    name       = data.get("name")
    start_date = data.get("start_date")
    end_date   = data.get("end_date")
    timetable  = data.get("timetable")
    if not all([name, start_date, end_date]) or not isinstance(timetable, dict):
        return jsonify({"ok": False, "error": "missing fields"})
    if end_date < start_date:
        return jsonify({"ok": False, "error": "終了日は開始日以降にしてください"})
    ...
    terms = load_terms(guild_id)
    term_id = data.get("id") or f"term_{time.time_ns()}"

    for tid, t in terms.items():
        if tid == term_id:
            continue
        if start_date <= t.get("end_date", "") and t.get("start_date", "") <= end_date:
            return jsonify({"ok": False, "error": f"「{t.get('name')}」（{t.get('start_date')}〜{t.get('end_date')}）と期間が重なっています"})
```
- `/save_term`… 新規作成と編集の両方を1つのエンドポイントで担う、よくあるパターンです。`data.get("id")`が送られてくれば既存の学期の編集、無ければ`f"term_{time.time_ns()}"`（現在時刻のナノ秒単位の値。ほぼ確実に他と被らない値になります）で新しいIDを発行します。
- `end_date < start_date`… 日付文字列（`YYYY-MM-DD`形式）は文字列としての比較がそのまま日付の前後関係と一致するため、`<`演算子でそのまま比較できます（[../09_FlaskAPI_予定管理/00_チャンネル一覧と予定の追加API.md](../09_FlaskAPI_予定管理/00_チャンネル一覧と予定の追加API.md)の`sorted(plans_by_date.keys())`と同じ性質を利用しています）。終了日が開始日より前という矛盾した入力を弾いています。
- **期間の重複チェック**が、この関数で最も注目すべき部分です。「前期」と「後期」の期間が重なってしまうと、ある1つの日付に対してどちらの学期の時間割を使うべきかが曖昧になってしまいます。`for tid, t in terms.items():`で既存の全学期を確認し、`if tid == term_id: continue`（今まさに編集しようとしている学期自身は、比較対象から除外する）とした上で、`start_date <= t.get("end_date") and t.get("start_date") <= end_date`という条件で期間が重なっているかを判定します。この条件式は、「2つの期間が重ならない」ことの否定（＝ド・モルガンの法則的な考え方）から導かれる、期間の重なり判定の定石です。重なっていれば、具体的にどの学期と重なっているかを名前・期間付きのエラーメッセージで伝え、保存を拒否します。

```python
    old_terms_text = _terms_text(terms)
    terms[term_id] = {
        "id": term_id, "name": name, "start_date": start_date,
        "end_date": end_date, "timetable": timetable,
    }
    ...
    return jsonify({"ok": True, "id": term_id})
```
- 重複が無ければ、`terms[term_id] = {...}`で保存し、（新規作成の場合はフロント側が使えるように）発行された`id`をレスポンスに含めて返します。

```python
@app.route("/delete_term", methods=["POST"])
def delete_term():
    ...
    terms = load_terms(guild_id)
    if term_id in terms:
        ...
        del terms[term_id]
        ...
    return jsonify({"ok": True})
```
- `/delete_term`… これも[00_時間割の上書きデータとCRUD_API.md](00_時間割の上書きデータとCRUD_API.md)の`/delete_timetable`と同じく、存在しないIDが指定されても`if term_id in terms:`のガードにより静かに成功扱いで終わる、冪等な削除です。

---

これで「Flask API — 時間割」の章が終わりました。次は「Flask API — ユーザー認証」の章に入ります。ここは`bot.py`の中でもとりわけボリュームが大きく（500行超）、セッション・Discordログイン・OAuth連携・削除依頼の仕組みなど、重要な内容が詰まっています。 → [../11_FlaskAPI_ユーザー認証/00_ユーザー一覧と連携コード発行.md](../11_FlaskAPI_ユーザー認証/00_ユーザー一覧と連携コード発行とOAuth開始.md)
