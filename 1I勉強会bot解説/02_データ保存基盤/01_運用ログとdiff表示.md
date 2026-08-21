# 運用ログ（システムログ）とdiff表示の仕組み（`bot.py` 343〜493行）

対象：`bot.py`の「システム運用ログ」セクション。これは[[../../1I勉強会web解説/08_ServiceInfo/00_ServiceInfo解説.md]]で紹介した「サービス情報」ページの「運用ログ」タブが表示している、**GitHubのコミット履歴のような差分付きログ**を、サーバー側で実際に記録している部分です。

## 設計方針（コメントより）

コード冒頭の長いコメントに、この機能の設計方針がまとめられています。要点は以下の通りです。
- 以前は`print`でしか見えなかった「何がいつ変更されたか」を、ログインしていれば誰でもWeb上で確認できるようにする（11人規模の運用のため、閲覧に管理者権限のような特別な区分けは設けない）。
- 記録の粒度は「1つのユーザー操作」単位（複数のJSONファイルを書き換える処理でも、ログには1件だけ残す）。
- Discord IDや本名など、ニックネーム以上に個人を特定できる情報は記録しない。
- **「本人にのみ表示される情報」は運用ログに残さない**という基準（運用ログ自体がログイン無しで誰でも見られる実質公開の場のため）。この基準により、課題の達成/取り消しや累計ポイントは運用ログの対象外になっています。

## 1. 保存先と1件追加：`log_event`（369〜428行）

```python
SYSTEM_LOG_FILE = "system_log.json"
SYSTEM_LOG_MAX_ENTRIES = 300
_system_log_lock = Lock()

def log_event(category, summary, level="info", actor=None, detail=None):
    try:
        with _system_log_lock:
            entries, sha = local_get(SYSTEM_LOG_FILE)
            if not isinstance(entries, list):
                entries = []
            entry = {
                "time": datetime.now(JST).strftime("%Y-%m-%d %H:%M:%S"),
                "category": category,
                "summary": summary,
                "level": level,
                "actor": actor,
            }
            files = detail if isinstance(detail, list) else ([{"file": None, "diff": str(detail)}] if detail else [])
            safe_files = []
            for f in files:
                diff_text = str((f or {}).get("diff") or "").strip()
                if not diff_text:
                    continue
                safe_files.append({
                    "file": (f or {}).get("file"),
                    "diff": diff_text[:4000],
                    "status": (f or {}).get("status"),
                })
                if len(safe_files) >= 12:
                    break
            if safe_files:
                entry["detail"] = safe_files
            entries.append(entry)
            entries = entries[-SYSTEM_LOG_MAX_ENTRIES:]
            local_put(SYSTEM_LOG_FILE, entries, sha)
    except Exception as e:
        print(f"[WARN] システムログの記録に失敗しました: {e}")
```
- `SYSTEM_LOG_FILE = "system_log.json"`にログの全件が1つの配列として保存されます。`SYSTEM_LOG_MAX_ENTRIES = 300`で、古いものから自動的に切り捨てて最大300件までしか保持しません（無限に増え続けてファイルが肥大化するのを防ぐ）。
- `_system_log_lock`は、複数のリクエストが同時に「読み込み→末尾に1件追加→書き込み」をしようとしたときに、片方の追加が消えてしまう事故を防ぐための専用ロックです（`local_get`/`local_put`自体のSHA排他制御とは別に、ここでも明示的にロックしています。理由は、「読み込み・配列への追加・書き込み」という3ステップの**間**に他のリクエストが割り込むこと自体を防ぎたいためです。SHA排他制御だけだと、食い違ったときに「無条件で1回だけ上書きリトライ」する仕様（`local_put`参照）なので、タイミングによっては先に追加された1件が消えてしまう可能性が残ります）。
- `entry`… 1件分のログの中身です。`time`（記録時刻、日本時間）・`category`（カテゴリ）・`summary`（短い説明文）・`level`（`"info"`か`"error"`。Web側で色分け表示するため）・`actor`（実行者のニックネーム。サーバー主導の処理では`None`のまま）。
- `detail`パラメータの処理が少し複雑です。コメントによれば、これは「具体的に何が変更されたか」をタップしたときだけ展開表示するための情報で、GitHubのコミット差分の見た目（ファイル名込み、ファイルごとに折りたためる）に近づけるため、`[{"file": ..., "diff": ..., "status": ...}, ...]`という**ファイルごとの差分のリスト**形式になっています。
  - `files = detail if isinstance(detail, list) else (...)`… `detail`がリストならそのまま使い、そうでなければ（後方互換のため）旧形式のプレーン文字列を1件のdiffとして扱います。
  - 各要素について、`diff`が空でなければ`safe_files`に追加します。`diff_text[:4000]`で1ファイル分の差分テキストの長さを4000文字に制限し（`[:4000]`はPythonのスライス構文で「先頭から4000文字だけ取り出す」という意味）、さらに`len(safe_files) >= 12`で1回の操作で記録するファイル数自体も12件までに制限しています。これらは、万一異常に巨大なデータが渡されてきても、ログファイル自体が肥大化しすぎないための安全弁です。
- 最後に`entries.append(entry)`で末尾に追加し、`entries[-SYSTEM_LOG_MAX_ENTRIES:]`（末尾から300件だけを残すスライス）で古いものを切り捨ててから保存します。
- 関数全体が`try`/`except`で囲まれており、**ログの記録に失敗してもBot本体は止まりません**（`print`で警告を出すだけ）。ログ機能はあくまで補助的なものなので、これが原因で本来の機能（予定の追加など）まで失敗させるべきではない、という判断です。

## 2. JSONブロック風の整形：`_json_block`（430〜442行）

```python
def _json_block(fields, label=None):
    open_line = f"{label} {{" if label else "{"
    lines = [open_line]
    lines.extend(f"  {k}: {v}" for k, v in fields)
    lines.append("}")
    return lines
```
- `fields`（`(ラベル, 値)`のタプルのリスト）を受け取り、`{ "フィールド名": 値, ... }`のような、GitHubでJSONファイルの差分を見るときと同じ見た目の複数行のリストに変換します。
- 例えば`_json_block([("date", "2026-08-20"), ("content", "宿題")], label="予定")`なら、`["予定 {", "  date: 2026-08-20", "  content: 宿題", "}"]`という4行のリストが返ります。
- このあと登場する各機能の`_xxx_lines`関数（例えば`_plan_lines`）が、この`_json_block`を使って「予定1件をこの見た目のテキストに変換する」役割を担っています。新規追加・削除された場合は、この`{`から`}`までの全行が丸ごと差分として表示され、実際のJSONファイルのコミット差分にかなり近い見た目になります。1フィールドだけの変更時は、変わった行だけが差分として表示されます（詳しい仕組みは次の`file_diff`で説明します）。

## 3. 差分テキストの生成：`file_diff`（444〜465行）

```python
def file_diff(file, old_text, new_text, max_lines=60):
    diff = _text_diff_lines(old_text, new_text, max_lines=max_lines)
    if not diff:
        return None
    if not (old_text or "").strip():
        status = "added"
    elif not (new_text or "").strip():
        status = "deleted"
    else:
        status = "modified"
    return {"file": file, "diff": diff, "status": status}
```
- `_text_diff_lines`（このあと出てくる、テキスト差分計算を行う内部ヘルパー。Pythonの`difflib`という標準ライブラリで2つのテキストの違いを`+`/`-`付きの行に変換します）を呼び出して、実際の差分行のリストを作ります。
- 差分が無ければ（変更前後で完全に同じなら）`None`を返し、呼び出し側はこの`None`を`detail`に含めないようにします（変化の無いフィールドまでログに残す必要は無いため）。
- `status`の判定がこの関数のポイントです。
  - `old_text`が空（変更前が無かった）→ `"added"`（新規作成）。
  - `new_text`が空（変更後が無い）→ `"deleted"`（削除）。
  - どちらもある → `"modified"`（既存の中身の書き換え）。
- コメントにある通り、カードデッキやお知らせは保存・削除のたびに実際にファイルが1つ作成・削除されますが、予定や時間割などは「共有の1つのJSONファイルの中の、1エントリだけ」を書き換えるため、ファイル自体としては`modified`にしかなりません（`added`/`deleted`は主にCardMaker・お知らせで使われます）。Web側は`added`を「新規作成」、`deleted`を「削除」のバッジとしてファイル名の横に表示します。

## 4. 閲覧API：`/system_log`（467〜493行）

```python
@app.route("/system_log", methods=["GET"])
def system_log():
    guild_id = request.args.get("guild_id")
    token = session_token_from_request()
    if guild_id and token:
        try:
            guild_id_int = int(guild_id)
        except (TypeError, ValueError):
            guild_id_int = None
        if guild_id_int is not None:
            student_id = resolve_session(token, guild_id_int)
            if student_id and not _session_is_member(guild_id_int, student_id):
                return jsonify({"ok": False, "error": "guild_membership_required"})

    entries, _ = local_get(SYSTEM_LOG_FILE)
    if not isinstance(entries, list):
        entries = []
    try:
        limit = max(1, min(200, int(request.args.get("limit", 100))))
    except (TypeError, ValueError):
        limit = 100
    return jsonify({"ok": True, "entries": list(reversed(entries))[:limit]})
```
- このAPIは元々「ログイン不要・誰でも閲覧可」でした（11人規模の運用のため）。ただし2026/08/20の追加で、**「制限付きアカウント」（対象サーバーに参加していないログイン済みDiscordアカウント）には見せない**という条件が加わっています。`guild_id`と`token`（セッショントークン）の両方が送られてきた場合だけメンバーシップを確認し、何も送られてこない場合（未ログインの匿名アクセス）は従来通り誰でも見られます。この「メンバーシップ確認・セッション」まわりの仕組み自体は、後の回（ユーザーとセッションの章）で詳しく解説します。
- `limit`パラメータ… クエリパラメータ`?limit=50`のような形で、何件返すかを指定できます。`max(1, min(200, ...))`で、1〜200件の範囲に強制的に収めています（大きすぎる値を指定されて大量のデータを一度に返してしまうのを防ぐ安全策）。指定が無ければデフォルト100件、数値として不正な値が渡された場合も100件にフォールバックします。
- `list(reversed(entries))[:limit]`… ログは`append`で末尾に追加されていく（＝新しいものほど末尾）ため、`reversed()`で並びを逆転させて「新しいものが先頭に来る」順にしてから、先頭から`limit`件だけを切り出して返します。Web側の一覧は、この「新しい順」のままそのまま上から表示しています。

---

次は、この`log_event`/`file_diff`を実際に使う最初の例として、サーバー設定ファイル（`config_{guild_id}.json`）まわりの読み書きと、投稿内容の不正文字チェックの仕組みを解説します。 → [[02_設定ファイルと不正文字チェック.md]]
