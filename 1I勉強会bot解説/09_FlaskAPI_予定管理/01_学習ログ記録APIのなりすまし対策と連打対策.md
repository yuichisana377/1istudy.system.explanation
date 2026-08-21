# `/add_study_log`：学習ログ記録APIの不正対策（`bot.py` 2265〜2438行）

対象：`bot.py`の`/add_study_log`エンドポイント。このAPIには、**過去に実際に起きた不正利用の経験を踏まえた対策**が何重にも重ねられており、`bot.py`全体の中でも特にセキュリティ的な作り込みが濃い場所です。

## 準備：定数（2265〜2267行）

```python
BASE_MAX_LOG_MINUTES = 180  # ★ タイマーを使っていない場合（手入力等）の上限。タイマー使用時は自動休憩を挟むたびに+3時間される
MANUAL_COOLDOWN_SEC = 20  # ★ 手入力：連続記録は前回から20秒あける（連打対策）
MANUAL_DAILY_MAX_MINUTES = 960  # ★ 1日の記録合計の上限（16時間）。短時間の連投による水増し防止
```
- これから登場する不正対策のしきい値が、ここにまとめて定義されています。

## 1. 補助関数：ログの時刻取得（2269〜2286行）

```python
def _parse_log_time(log):
    """ログの正確な時刻（"time"）をdatetimeに変換する。無ければNone。"""
    t = log.get("time")
    if not t:
        return None
    try:
        return datetime.strptime(t, "%Y-%m-%d %H:%M:%S").replace(tzinfo=JST)
    except Exception:
        return None

def _latest_log_time(candidates):
    """candidates（ログのリスト）の中で最も新しい時刻を返す。無ければNone。"""
    latest = None
    for l in candidates:
        t = _parse_log_time(l)
        if t and (latest is None or t > latest):
            latest = t
    return latest
```
- `_latest_log_time`は、渡されたログのリストの中から一番新しい記録時刻を1つだけ取り出します。このあと「前回の記録からどれくらい時間が経過したか」を判定するために繰り返し使われます。

## 2. 本人確認とニックネームの取得（2288〜2304行）

```python
@app.route("/add_study_log", methods=["POST"])
def add_study_log():
    data = request.json or {}
    guild_id = int(data.get("guild_id"))

    student_id, err = require_member_session(data.get("session_token"), guild_id)
    if err:
        return err

    user = find_user(guild_id, student_id)
    nickname = user["nickname"] if user else data.get("nickname")
```
- ここでは`require_login_json`ではなく`require_member_session`を直接呼んでいます（`guild_id`のバリデーション部分だけ自前で行い、残りは共通処理を使う形です）。コメントの通り、クライアントが自己申告する`student_id`は一切信用せず、セッショントークンから本人を特定します。
- `nickname`も同様に、`find_user`でサーバー側のデータから引き直しています。もし何らかの理由で`user`が見つからなければ（通常は起こらないはずですが）、フォールバックとしてクライアントの自己申告（`data.get("nickname")`）を使います。

## 3. 記録できる分数の上限チェック（2306〜2322行）

```python
    minutes = data.get("minutes")
    if not isinstance(minutes, int) or isinstance(minutes, bool):
        return jsonify({"ok": False, "error": "invalid minutes"})

    timer_entry = load_study_timers(guild_id).get(student_id)
    if timer_entry and timer_entry.get("next_checkpoint_sec"):
        max_log_minutes = int(timer_entry["next_checkpoint_sec"] // 60)
    else:
        max_log_minutes = BASE_MAX_LOG_MINUTES

    if minutes < 1 or minutes > max_log_minutes:
        return jsonify({"ok": False, "error": f"minutes must be between 1 and {max_log_minutes}"})
```
- `isinstance(minutes, int) or isinstance(minutes, bool)`… `minutes`が本当に整数かどうかをチェックしています。`isinstance(minutes, bool)`を別途チェックしているのは、Pythonでは`bool`（`True`/`False`）が`int`のサブクラスであるため、`isinstance(True, int)`が`True`になってしまう（＝`True`が整数として通ってしまう）という言語仕様上の落とし穴を避けるためです。ブール値が数値として紛れ込むのを、明示的に弾いています。
- コメントにある通り、記録できる上限は固定の180分ではなく、**実際にタイマーで自動休憩を何回挟んだか**に応じて自動的に引き上がる仕組みです。[../04_勉強タイマー/00_勉強タイマーの状態管理.md](../04_勉強タイマー/00_勉強タイマーの状態管理.md)で見た`next_checkpoint_sec`（次の3時間区切りのライン）を分に変換した値が、そのまま今回記録できる上限になります。タイマーを使わない手入力の場合は、基本の180分（3時間）が上限のままです。この仕組みにより、長時間勉強を続けて何度も自動休憩を挟んだ人ほど、後でまとめて記録する際により長い時間を記録できるようになっています。

## 4. 前回の記録からの経過時間チェック（2338〜2394行）

コメントにある通り、これはクライアント側（`StudyLog.js`）でも行われているチェックですが、devtools等で直接APIを叩けば素通りしてしまうため、**サーバー側でも独立して**判定する必要がある、という一連の防御です。

```python
    my_logs = [l for l in logs if l.get("student_id") == student_id]

    if method == "timer":
        last_time = _latest_log_time(my_logs)
        if last_time:
            elapsed_sec  = (now_jst - last_time).total_seconds()
            required_sec = minutes * 60
            if elapsed_sec < required_sec:
                remain_min = int((required_sec - elapsed_sec) // 60) + 1
                return jsonify({
                    "ok": False,
                    "error": f"前回の記録からまだ十分な時間が経過していません（あと約{remain_min}分待つ必要があります）"
                })
```
- `method == "timer"`（タイマー機能を使って計測した記録）の場合：本人の（教科を問わない）前回の記録時刻から、**今回記録しようとしている分数と同じだけの実時間が、実際に経過していなければ拒否**します。これは、タイマーの経過時間そのものをブラウザ側で改ざんし、「本当は1分しか計測していないのに180分計測したことにして送信する」といった不正を防ぐためです。実際の時計の進み方はサーバー側で管理しているため、この時間の経過だけは偽ることができません。

```python
    elif method == "manual":
        last_time_any = _latest_log_time(my_logs)
        if last_time_any:
            elapsed_sec_any = (now_jst - last_time_any).total_seconds()
            if elapsed_sec_any < MANUAL_COOLDOWN_SEC:
                remain_sec = int(MANUAL_COOLDOWN_SEC - elapsed_sec_any) + 1
                return jsonify({...})

        same_subject_logs = [l for l in my_logs if l.get("subject") == subject]
        last_time = _latest_log_time(same_subject_logs)
        if last_time:
            elapsed_sec = (now_jst - last_time).total_seconds()
            if elapsed_sec < MANUAL_COOLDOWN_SEC:
                remain_sec = int(MANUAL_COOLDOWN_SEC - elapsed_sec) + 1
                return jsonify({...})

        today_str = now_jst.strftime("%Y-%m-%d")
        today_total = sum(l.get("minutes", 0) for l in my_logs if l.get("date") == today_str)
        if today_total + minutes > MANUAL_DAILY_MAX_MINUTES:
            return jsonify({...})
```
- `method == "manual"`（手入力での記録）の場合は3段階のチェックがあります。
  1. **教科を問わない、前回の記録からのクールダウン**（20秒）。
  2. **同じ教科での、前回の記録からのクールダウン**（20秒）。
  3. **1日の記録合計の上限**（960分＝16時間）。
- 特に1番目のチェックには、コメントに書かれている**実際に起きた被害の記録**が残っています。「2026-08-17、教科名を毎回変えながら連投することで『同じ教科』判定の1分クールダウンを回避され、10分間に34件・180分ずつの水増し記録を入れられる被害が発生した」とあります。つまり、以前は「同じ教科」でのクールダウンしかチェックしておらず、記録するたびに教科名を変えれば連続投稿ができてしまう抜け穴があり、実際にこれが悪用されました。
- この修正で「本人の直近ログ（教科不問）」からの経過時間もチェックするようになり、**教科を変えるだけの連投そのもの**を防いでいます。さらに3番目の「1日の合計上限」も、コメントの通り「1分間隔さえ守れば教科を変えて延々と積み上げられてしまう」ことへの、もう一段階の保険として追加されています。
- **これは実際の攻撃と、それに対する具体的な修正が両方コードのコメントとして残っている、生きた事例**です。セキュリティ対策は「こういう攻撃があるかもしれない」という抽象的な想像だけでなく、実際に起きた被害から学んで一つ一つ穴を塞いでいく、という積み重ねであることがよく分かる箇所です。

## 5. 記録の保存（2396〜2420行）

```python
    entry = {
        "date": now_jst.strftime("%Y-%m-%d"),
        "time": now_jst.strftime("%Y-%m-%d %H:%M:%S"),
        "subject": subject,
        "minutes": minutes,
        "memo": memo,
        "student_id": student_id,
        "nickname": nickname
    }

    now = now_jst.date()
    logs = [
        l for l in logs
        if (now - datetime.strptime(l["date"], "%Y-%m-%d").date()).days <= 30
    ]
    old_logs = list(logs)
    logs.append(entry)
    try:
        save_study_logs(guild_id, logs)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})
```
- コメントにある通り、`date`/`time`は**クライアントから送られてきた値を一切使わず、サーバー自身の時計（`now_jst`）から作っています**。もしクライアントの自己申告する日時をそのまま信用してしまうと、生徒が自分のPCの時計を進めたり戻したりするだけで、記録される日時を自由に偽装できてしまいます。サーバー側の時刻だけを正としているため、この手の改ざんは効きません。
- `logs = [... if ... <= 30]`… [../03_予定と時間割データ層/00_予定データとログ記録.md](../03_予定と時間割データ層/00_予定データとログ記録.md)の`write_log`と同じ「30日より古い記録の自動間引き」がここでも行われています。
- `old_logs = list(logs)`… ここで控えているのは、**間引いた後**の状態です。コメントの通り、これは「今回追加する1件による変化だけ」が運用ログの差分として浮かび上がるようにするためです。もし間引く前の状態を`old_logs`にしてしまうと、「30日前のログが消えたこと」まで今回の変更の差分に混ざって表示されてしまいます。

## 6. ポイント加算と運用ログ記録（2422〜2438行）

```python
    earned = entry["minutes"] // 5
    pts = load_points(guild_id)
    pts[entry["student_id"]] = pts.get(entry["student_id"], 0) + earned
    try:
        save_points(guild_id, pts)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})

    change = _diff_study_logs(f"study_logs_{guild_id}.json", old_logs, logs)
    log_event(
        "study",
        f"学習ログ「{subject}」を記録しました（{entry['minutes']}分・{earned}pt加算）。",
        actor=nickname,
        detail=[change] if change else None,
    )
    return jsonify({"ok": True, "earned": earned, "total": pts[entry["student_id"]]})
```
- `earned = entry["minutes"] // 5`… 5分ごとに1ポイントというルールで、今回の記録で獲得したポイントを計算します（`//`は整数の切り捨て除算。7分なら`7 // 5 = 1`ポイント）。
- `pts.get(entry["student_id"], 0) + earned`… その生徒の累計ポイントに、今回獲得した分を足します。`.get(..., 0)`により、まだ一度もポイントが無い（初めての記録）生徒でも、`0`から始められます。
- 最後に、[../03_予定と時間割データ層/00_予定データとログ記録.md](../03_予定と時間割データ層/00_予定データとログ記録.md)で見た`_diff_study_logs`（テキストではなくキー突き合わせで正しく差分を取る、修正済みの版）で運用ログ用の差分を作り、`log_event`で記録します。
- レスポンスには`earned`（今回もらったポイント）と`total`（累計）の両方を返し、フロント側が「+1pt！」のようなその場のフィードバックと、最新の累計表示の両方をすぐに更新できるようにしています。

---

学習ログの削除・予定の一覧取得/編集/削除については、次のファイルでまとめて解説します。 → [02_学習ログ削除と予定の一覧編集削除.md](02_学習ログ削除と予定の一覧編集削除.md)
