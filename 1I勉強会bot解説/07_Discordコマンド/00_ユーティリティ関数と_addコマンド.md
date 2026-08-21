# 細々としたユーティリティと `/add` コマンド（`bot.py` 1491〜1607行）

対象：`bot.py`の「科目チャンネルユーティリティ」「日付パース」「ポイントを付与すべきカテゴリかどうか」「add 内部関数」「/add」セクション。ここからいよいよ、Discord上で実際に使われる`/`コマンドの実装に入ります。

## 用語：スラッシュコマンドとは

Discordで`/`を打つと出てくる、あらかじめ登録されたコマンドの一覧から選んで実行する機能です。`bot.py`では`@bot.tree.command(...)`という書き方で、1つ1つのコマンドを定義しています。コマンドの引数（例えば`/add`の`date`や`content`）も、Discordの入力補完（オートコンプリート）付きで表示されるように、Python側で型や説明文を指定しておく仕組みになっています。

## 1. 科目チャンネルユーティリティ（1491〜1509行）

```python
def get_subject_channels(guild: discord.Guild) -> list:
    if SUBJECT_CATEGORY_ID:
        for cat in guild.categories:
            if cat.id == int(SUBJECT_CATEGORY_ID):
                return list(cat.text_channels)
    if SUBJECT_CATEGORY:
        for cat in guild.categories:
            if cat.name == SUBJECT_CATEGORY:
                return list(cat.text_channels)
    return list(guild.text_channels)
```
- Discordサーバーには「カテゴリ」というチャンネルのグループ分けの仕組みがあります（例えば「授業科目」というカテゴリの下に、数学・英語・国語…というテキストチャンネルがまとまっている、というような構成）。この関数は、「科目ごとのチャンネル一覧」を取得します。
- [../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md](../01_起動と初期設定/00_設定の読み込みとFlask_Discordの初期化.md)で見た環境変数`SUBJECT_CATEGORY_ID`（カテゴリIDでの指定、優先）または`SUBJECT_CATEGORY`（カテゴリ名での指定、フォールバック）のどちらかで指定されたカテゴリの中のテキストチャンネル一覧を返します。どちらも設定されていない場合は、サーバー内の全テキストチャンネルを返します（`guild.text_channels`）。
- IDでの指定を優先しているのは、チャンネル名やカテゴリ名は後から変更される可能性がありますが、IDは一度作られたら変わらないため、より確実な指定方法だからです。

```python
def get_subject_channel_by_name(guild: discord.Guild, name: str):
    for ch in get_subject_channels(guild):
        if ch.name == name:
            return ch
    return None
```
- 科目名（チャンネル名と一致する文字列）から、実際のチャンネルオブジェクトを1つ探します。見つからなければ`None`です。`/add`コマンドで「この科目の予定を、対応するチャンネルに通知する」という処理で使われます。

## 2. 日付パース（1511〜1525行）

```python
def parse_date(date: str):
    try:
        if "-" in date and len(date.split("-")[0]) == 4:
            parsed = datetime.strptime(date, "%Y-%m-%d")
        else:
            date = date.replace("/", "-")
            m, d = date.split("-")
            y = datetime.now().year
            parsed = datetime.strptime(f"{y}-{int(m):02d}-{int(d):02d}", "%Y-%m-%d")
        return parsed.strftime("%Y-%m-%d")
    except Exception:
        return None
```
- Discordコマンドの`date`引数として、生徒が`2026-06-20`のようなフル表記でも`6-20`や`6/20`のような省略表記でも入力できるようにするための、柔軟な日付解析関数です。
- `"-" in date and len(date.split("-")[0]) == 4`… `-`が含まれていて、かつ最初の部分（`-`で区切った1つ目）が4文字（＝西暦4桁）なら、フル表記（`YYYY-MM-DD`）とみなしてそのまま解析します。
- そうでなければ省略表記とみなし、まず`/`を`-`に統一してから`月-日`の2つに分割し、今年の年（`datetime.now().year`）を補って`YYYY-MM-DD`の形に組み立て直します。`f"{y}-{int(m):02d}-{int(d):02d}"`の`:02d`は「2桁になるよう0埋めする」という書式指定です（例えば`6`は`06`になります）。
- どのパターンにも当てはまらず解析に失敗した場合（`try`ブロック内で例外が発生した場合）は、`except Exception:`で捕まえて`None`を返します。呼び出し側はこの`None`を「日付として不正な入力だった」というエラー扱いにします。

## 3. ポイント付与の対象カテゴリ（1527〜1531行）

```python
POINT_CATEGORIES = ("提出", "宿題")
DEFAULT_TASK_POINTS = 5
```
- 予定の`category`（分類）が「提出」または「宿題」の場合だけ、ポイントが付与される課題として扱われます（例えば「持ち物」や「テスト」といったカテゴリの予定にはポイントは付きません）。
- `DEFAULT_TASK_POINTS = 5`… ポイント数が省略された場合のデフォルト値（5ポイント）です。[../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md](../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md)で見た`find_task_points`でも、この定数が「本来のポイント数」の初期値として使われていました。

## 4. `/add`の内部処理：`add_plan_internal`（1533〜1568行）

```python
async def add_plan_internal(guild_id: int, subject: str, date: str, category: str, content: str, points=None):
    date_str = parse_date(date)
    if not date_str:
        return False, "日付の形式が正しくありません！", None
    today = datetime.now(JST).date()
    if datetime.strptime(date_str, "%Y-%m-%d").date() < today:
        return False, "過去の日付は登録できません！", None
    tagged_content = f"【{category}】{content}"

    plan = {"date": date_str, "subject": subject, "content": tagged_content}
    if category in POINT_CATEGORIES:
        plan["points"] = points if points is not None else DEFAULT_TASK_POINTS

    plans = load_plans(guild_id)
    old_plans_text = _plans_text(plans)  # ★ 運用ログでファイル全体の差分を見せるため、追加前に控えておく
    plans.append(plan)
    try:
        save_plans(guild_id, plans)
    except DataWriteError as e:
        return False, f"保存に失敗しました（データ保存エラー）。もう一度お試しください。\n{e}", None

    detail = f"{date_str} / {subject} / {tagged_content}"
    if "points" in plan:
        detail += f" ({plan['points']}pt)"
    write_log(guild_id, "add", detail=detail)

    msg = f"登録しました！\n{date_str} / {subject} / {tagged_content}"
    if "points" in plan:
        msg += f"\n⭐ {plan['points']}pt"
    change = file_diff(f"plans_{guild_id}.json", old_plans_text, _plans_text(plans))
    return True, msg, [change] if change else None
```
- 「予定を1件追加する」という実際の処理本体です。関数名の末尾が`_internal`となっている通り、Discordコマンド版の`/add`とWeb版の`/add_schedule`（後のFlask API章で解説）の**両方から共通で呼ばれる**、実装の核心部分です。同じ処理を2箇所に書かずに済むよう、こうして共通関数として切り出されています。
- 戻り値は常に`(成功したか, メッセージ, 運用ログ用の変更差分のリスト or None)`という3つ組です。
- `parse_date`で日付を検証し、不正なら失敗として返します。さらに、`today`（今日の日付、日本時間）より過去の日付は登録を拒否します（過去の予定を追加できても意味が無いための制限）。
- `tagged_content = f"【{category}】{content}"`… 予定の本文の先頭に、カテゴリ名を`【】`で囲んで埋め込みます。これにより、`content`自体には「カテゴリ」という独立したフィールドを持たず、テキストの中に埋め込む形で管理されています（フロント側の`Plan.js`が、この`【〇〇】`という接頭辞を解析してカテゴリバッジとして表示しています）。
- `if category in POINT_CATEGORIES:`… カテゴリが「提出」または「宿題」の場合だけ、`points`フィールドを追加します。`points if points is not None else DEFAULT_TASK_POINTS`は、引数で明示的にポイント数が指定されていればそれを、無ければデフォルトの5ポイントを使う、という意味です。
- `old_plans_text = _plans_text(plans)`… 予定を追加する**前**の状態を、[../03_予定と時間割データ層/00_予定データとログ記録.md](../03_予定と時間割データ層/00_予定データとログ記録.md)で見た`_plans_text`でテキスト化して控えておきます。コメントの通り、これは運用ログでファイル全体の差分を表示するために必要です。
- `save_plans`が`DataWriteError`を投げた場合（保存失敗）は、その場でエラーメッセージを返して処理を打ち切ります。
- `write_log(guild_id, "add", detail=detail)`… [../03_予定と時間割データ層/00_予定データとログ記録.md](../03_予定と時間割データ層/00_予定データとログ記録.md)で見た、古い方の簡易ログにも記録します。
- 最後に、追加後の状態を改めて`_plans_text(plans)`でテキスト化し、`file_diff`で追加前後の差分を計算します。これが運用ログ（`system_log.json`）に載る「＋（追加された行）」の中身になります。`[change] if change else None`… 差分が実際にあれば1件のリストにして返し、無ければ（通常は追加なので必ず差分があるはずですが）`None`を返します。

## 5. Discordコマンド本体：`/add`（1570〜1606行）

```python
@bot.tree.command(name="add", description="予定を追加する")
@app_commands.describe(
    date="日付（例: 6-20, 2026-06-20）",
    subject="科目（省略するとこのチャンネル名を使用）",
    category="分類（宿題・提出・持ち物など）",
    content="内容",
    points="ポイント（提出・宿題のみ有効。省略時は5pt）"
)
async def add_plan(interaction: discord.Interaction, date: str, category: str, content: str, subject: str = None, points: int = None):
    await interaction.response.defer(ephemeral=True)
    guild = interaction.guild
    if not subject:
        subject = interaction.channel.name
    ok, msg, _detail = await add_plan_internal(guild.id, subject, date, category, content, points)
    if ok:
        target_channel = get_subject_channel_by_name(guild, subject)
        await (target_channel or interaction.channel).send(msg)
    else:
        await interaction.followup.send(msg, ephemeral=True)
        return
    await interaction.followup.send("完了しました！", ephemeral=True)
```
- `@bot.tree.command(name="add", ...)`… これが`/add`というスラッシュコマンドを定義する部分です。`@app_commands.describe(...)`で、各引数にDiscord上で表示される説明文を付けています。
- `interaction: discord.Interaction`… Discordのコマンド実行という「イベント」そのものを表すオブジェクトです。「誰が」「どのサーバーの」「どのチャンネルで」実行したか、といった情報がここに詰まっています。
- `await interaction.response.defer(ephemeral=True)`… コマンドの処理に時間がかかる可能性があるため、まず「今処理中です」という一時的な応答をDiscordに送っておきます（`defer`は「先延ばしにする」の意味）。`ephemeral=True`は、この応答をコマンドを実行した本人にしか見えない一時的なメッセージにする指定です。Discordのスラッシュコマンドは、一定時間内（3秒）に何かしら応答しないとタイムアウトしてしまうため、この`defer`が実質的な時間稼ぎの役割を果たします。
- `if not subject: subject = interaction.channel.name`… 科目が省略された場合は、コマンドを実行した**そのチャンネルの名前**を科目名として自動的に使います（各科目ごとにチャンネルが分かれている運用を前提にした、便利な省略機能です）。
- `add_plan_internal`を呼び出し、結果に応じて処理を分けます。成功した場合は、`get_subject_channel_by_name`でその科目に対応するチャンネルを探し、そこに（見つからなければコマンドを実行したチャンネルに）登録完了メッセージを送信します。これは、コマンドをどのチャンネルで実行しても、実際の通知メッセージは正しい科目のチャンネルに投稿される、という便利な仕組みです。
- 失敗した場合は、`interaction.followup.send(msg, ephemeral=True)`で、コマンド実行者本人にだけ見えるエラーメッセージを返し、`return`で処理を打ち切ります（`followup.send`は、最初の`defer`の続きとして実際のメッセージを送るためのメソッドです）。
- 成功した場合は最後に、実行者本人にだけ見える「完了しました！」を追加で送ります（チャンネルへの投稿は全員に見える一方、コマンド実行者への「ちゃんと成功しましたよ」という個別のフィードバックも別途行っている、という2段構えです）。

```python
@add_plan.autocomplete("subject")
async def add_subject_autocomplete(interaction: discord.Interaction, current: str):
    channels = get_subject_channels(interaction.guild)
    return [
        app_commands.Choice(name=ch.name, value=ch.name)
        for ch in channels if current.lower() in ch.name.lower()
    ][:25]

@add_plan.autocomplete("category")
async def add_category_autocomplete(interaction: discord.Interaction, current: str):
    candidates = ["宿題", "提出", "持ち物", "テスト", "その他"]
    return [app_commands.Choice(name=c, value=c) for c in candidates if current in c][:25]
```
- `@add_plan.autocomplete("subject")`… `subject`引数を入力しているときに、Discordの入力欄にリアルタイムで候補を表示する「オートコンプリート」機能を定義しています。
- `add_subject_autocomplete`は、`get_subject_channels`で取得した科目チャンネルの中から、今まさに入力中の文字列`current`を含むものだけを候補として返します（`current.lower() in ch.name.lower()`… 大文字小文字を無視した部分一致検索）。
- `add_category_autocomplete`は、あらかじめ決められた候補（`"宿題", "提出", "持ち物", "テスト", "その他"`）の中から、入力中の文字列を含むものを返します。
- どちらも`[:25]`で最大25件までに絞っています。これはDiscord側の仕様上の制限（オートコンプリートの候補は最大25件までしか表示できない）に合わせたものです。

---

次は、同じような構造を持つ`/list`・`/delete`・`/edit`コマンド（予定の一覧表示・削除・編集）を解説します。 → [01_list_delete_editコマンド.md](01_list_delete_editコマンド.md)
