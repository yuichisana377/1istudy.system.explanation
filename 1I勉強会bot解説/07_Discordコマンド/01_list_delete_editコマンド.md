# `/list`・`/delete`・`/edit` コマンド（`bot.py` 1608〜1772行）

対象：`bot.py`の「/list」「/delete」「/edit」セクション。[00_ユーティリティ関数と_addコマンド.md](00_ユーティリティ関数と_addコマンド.md)で見た`/add`と同じ構造の繰り返しなので、今回は差分（新しく登場する考え方）を中心に説明します。

## 1. `/list`：予定一覧の表示（1611〜1640行）

```python
@bot.tree.command(name="list", description="予定一覧を表示する")
@app_commands.describe(date="all または 日付（例: 6/15, 2026-06-15）")
async def list_plans(interaction: discord.Interaction, date: str):
    await interaction.response.defer(ephemeral=True)
    guild_id = interaction.guild.id
    plans = await async_load_plans(guild_id)
    if date.lower() == "all":
        if not plans:
            await interaction.followup.send("予定はありません。", ephemeral=True)
            return
        sorted_plans = sorted(plans, key=lambda p: p["date"])
        msg = "📘 **すべての予定一覧**\n"
        for p in sorted_plans:
            pts_str = f" ⭐{p['points']}pt" if "points" in p else ""
            msg += f"- {p['date']}：{p['subject']} {p['content']}{pts_str}\n"
        await interaction.followup.send(msg, ephemeral=True)
        return
    date_str = parse_date(date)
    if not date_str:
        await interaction.followup.send("日付の形式が正しくありません！", ephemeral=True)
        return
    selected = [p for p in plans if p["date"] == date_str]
    if not selected:
        await interaction.followup.send(f"{date} の予定はありません。", ephemeral=True)
        return
    msg = f"📘 **{date_str} の予定**\n"
    for p in selected:
        pts_str = f" ⭐{p['points']}pt" if "points" in p else ""
        msg += f"- {p['subject']} {p['content']}{pts_str}\n"
    await interaction.followup.send(msg, ephemeral=True)
```
- `date`引数に`"all"`（大文字小文字を区別しないよう`.lower()`で統一してから比較）を指定すると、全ての予定を日付順（`sorted(plans, key=lambda p: p["date"])`。`key=lambda p: p["date"]`は「並び替えの基準として、各予定の`date`フィールドを使う」という指定）に並べて一覧表示します。
- それ以外の場合は`parse_date`で日付として解析し、その日付に一致する予定だけを`selected`として絞り込みます（リスト内包表記）。
- どちらの場合も、`interaction.followup.send(msg, ephemeral=True)`で、コマンド実行者だけに見える形でメッセージ本文として一覧を返します（`/add`と違い、他のメンバーに公開のチャンネル投稿はしません。「一覧を確認したいだけ」の操作なので、公開のノイズを増やさない設計です）。

## 2. `/delete`：予定の削除（1645〜1681行）

```python
@bot.tree.command(name="delete", description="予定を削除する")
@app_commands.describe(target="削除したい予定")
async def delete_plan(interaction: discord.Interaction, target: str):
    await interaction.response.defer(ephemeral=True)
    guild = interaction.guild
    plans = await async_load_plans(guild.id)
    deleted = None
    new_plans = []
    for p in plans:
        label = f"{p['date']}/{p['subject']}{p['content']}"
        if label == target:
            deleted = p
        else:
            new_plans.append(p)
    if not deleted:
        await interaction.followup.send("その予定は見つかりませんでした。", ephemeral=True)
        return
    try:
        save_plans(guild.id, new_plans)
    except DataWriteError as e:
        await interaction.followup.send(f"保存に失敗しました（データ保存エラー）。もう一度お試しください。\n{e}", ephemeral=True)
        return
    write_log(guild.id, "delete", detail=f"{deleted['date']} / {deleted['subject']} / {deleted['content']}")
    msg = f"削除しました！\n{target}"
    target_channel = get_subject_channel_by_name(guild, deleted["subject"])
    await (target_channel or interaction.channel).send(msg)
    await interaction.followup.send("完了しました！", ephemeral=True)
```
- **予定には専用のIDが無い**ため、`f"{p['date']}/{p['subject']}{p['content']}"`という「日付/科目 内容」を組み合わせた文字列（`label`）を、その予定を一意に指し示すための識別子として使っています。これは[../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md](../05_ユーザーとセッション/00_ポイント課題達成ユーザーデータとレート制限.md)の`_task_id_of_plan`と似た考え方ですが、区切り文字やフィールドの組み合わせ方が微妙に異なる点に注意してください（別々の目的のために、別々の場所で個別に定義されている識別子です）。
- `deleted`（削除対象。1つだけ見つかる想定）と`new_plans`（それ以外の残す予定）の2つに、ループを回しながら振り分けています。`target`と一致する`label`を持つ予定だけを除外し、残りをまるごと`new_plans`として保存し直す、という「配列を丸ごと入れ替える」削除の仕方です（個別の要素を`remove`するのではなく、新しい配列を作って上書きする方式です）。
- 一致するものが無ければ（`deleted`が`None`のまま）、見つからなかった旨を伝えて処理を打ち切ります。

```python
@delete_plan.autocomplete("target")
async def delete_autocomplete(interaction: discord.Interaction, current: str):
    plans = load_plans(interaction.guild.id)
    choices = []
    for p in plans:
        label = f"{p['date']}/{p['subject']}{p['content']}"
        if current in label:
            choices.append(app_commands.Choice(name=label, value=label))
    return choices[:25]
```
- `target`引数のオートコンプリートです。今存在する全ての予定の`label`を候補として表示し、入力中の文字列を含むものだけに絞り込みます。これにより、生徒は予定を1文字1文字正確に打ち込む必要がなく、候補一覧から選ぶだけで`target`を指定できます。

## 3. `/edit`：予定の編集（1686〜1771行）

```python
@bot.tree.command(name="edit", description="予定を編集する")
@app_commands.describe(
    target="編集したい予定",
    date="新しい日付",
    subject="新しい科目",
    category="新しい分類",
    content="新しい内容",
    points="新しいポイント（提出・宿題のみ有効）"
)
async def edit_plan(interaction: discord.Interaction, target: str, date: str = None, subject: str = None, category: str = None, content: str = None, points: int = None):
    await interaction.response.defer(ephemeral=True)
    guild = interaction.guild
    plans = await async_load_plans(guild.id)
    found = None
    for p in plans:
        label = f"{p['date']}/{p['subject']}{p['content']}"
        if label == target:
            found = p
            break
    if not found:
        await interaction.followup.send("その予定が見つかりませんでした。", ephemeral=True)
        return
```
- `date`・`subject`・`category`・`content`・`points`は全て**省略可能**（デフォルト値`None`）な引数です。「変更したい項目だけを指定し、それ以外は元のまま残す」という部分更新の形になっています。
- `for ... break`… `target`と一致する予定を見つけたら`break`でループを打ち切ります（`/delete`のようにループを最後まで回して振り分ける必要はなく、1件見つければそれで十分だからです）。

```python
    before_str = f"{found['date']} / {found['subject']} / {found['content']}"
    if date:
        date_str = parse_date(date)
        if not date_str:
            await interaction.followup.send("日付の形式が正しくありません！", ephemeral=True)
            return
        found["date"] = date_str
    if subject:
        found["subject"] = subject
    if category and content:
        found["content"] = f"【{category}】{content}"
    elif category:
        body = found["content"].split("】", 1)[1] if "】" in found["content"] else found["content"]
        found["content"] = f"【{category}】{body}"
    elif content:
        tag = found["content"].split("】", 1)[0] + "】" if "】" in found["content"] else ""
        found["content"] = f"{tag}{content}"
```
- `before_str`… 変更を加える**前**の状態を、あとで運用ログに「編集前 → 編集後」として表示するために控えておきます。
- `content`の書き換えロジックが少し込み入っています。[00_ユーティリティ関数と_addコマンド.md](00_ユーティリティ関数と_addコマンド.md)の`add_plan_internal`で見た通り、実際の内容は`【カテゴリ】本文`という1本の文字列として保存されているため、「カテゴリだけ変えたい」「本文だけ変えたい」「両方変えたい」の3パターンをそれぞれ個別に組み立て直す必要があります。
  - 両方指定 → シンプルに`f"【{category}】{content}"`で丸ごと作り直す。
  - カテゴリだけ指定 → `found["content"].split("】", 1)[1]`（`】`で1回だけ分割し、その後ろの部分＝元の本文）を取り出し、新しいカテゴリと組み合わせる。`if "】" in found["content"] else found["content"]`は、万一元のデータに`】`が含まれていない異常なケースへの保険です。
  - 本文だけ指定 → `found["content"].split("】", 1)[0] + "】"`（`】`より前の部分＋`】`自体＝元のカテゴリタグ）を取り出し、新しい本文と組み合わせる。`】`が無ければタグ無し（空文字列）として扱う。

```python
    current_category = found["content"].split("】", 1)[0].lstrip("【") if "】" in found["content"] else ""
    if points is not None:
        found["points"] = points
    if current_category not in POINT_CATEGORIES and "points" in found:
        del found["points"]
    elif current_category in POINT_CATEGORIES and "points" not in found:
        found["points"] = DEFAULT_TASK_POINTS
```
- 編集後の`content`から、改めて今のカテゴリ（`current_category`）を取り出します。`.lstrip("【")`は文字列の先頭にある`【`を取り除く処理です。
- `points`が明示的に指定されていれば、それを反映します。
- 続く2つの条件分岐が、**カテゴリの変更に連動してポイントの有無を自動的に整合させる**ための処理です。
  - もし今のカテゴリが「提出」「宿題」以外に変わったのに、まだ`points`フィールドが残っていれば、`del found["points"]`で削除します（例えば「宿題」から「持ち物」にカテゴリを変更したら、ポイントの概念が無くなるべきです）。
  - 逆に、今のカテゴリが「提出」「宿題」になったのに`points`フィールドがまだ無ければ（例えば元々「持ち物」だった予定を「宿題」に変更した場合）、デフォルトのポイント数を自動的に付与します。
- こうして、`category`と`points`という2つのフィールドの状態が、常に矛盾なく保たれるようになっています。

```python
    try:
        await async_save_plans(guild.id, plans)
    except DataWriteError as e:
        await interaction.followup.send(f"保存に失敗しました（データ保存エラー）。もう一度お試しください。\n{e}", ephemeral=True)
        return
    after_str = f"{found['date']} / {found['subject']} / {found['content']}"
    await async_write_log(guild.id, "edit", detail=f"{before_str} → {after_str}")
    msg = f"編集しました！\n\n【編集前】\n{before_str}\n\n【編集後】\n{after_str}"
    if "points" in found:
        msg += f"\n⭐ {found['points']}pt"
    target_channel = get_subject_channel_by_name(guild, found["subject"])
    await (target_channel or interaction.channel).send(msg)
    await interaction.followup.send("完了しました！", ephemeral=True)
```
- `found`はここまでの処理で`plans`（読み込んだ配列全体）の中の1要素そのものを直接書き換えてきているので（Pythonでは辞書やリストの要素は参照渡しされるため、`found`を書き換えると`plans`の中身も一緒に変わります）、最後に`async_save_plans(guild.id, plans)`で`plans`全体を保存すれば、変更が反映されます。
- `before_str`と`after_str`を組み合わせたメッセージで、「何がどう変わったか」が一目で分かる編集完了メッセージを組み立てています。

```python
@edit_plan.autocomplete("target")
async def edit_target_autocomplete(interaction: discord.Interaction, current: str):
    plans = load_plans(interaction.guild.id)
    choices = []
    for p in plans:
        label = f"{p['date']}/{p['subject']}{p['content']}"
        if current in label:
            choices.append(app_commands.Choice(name=label, value=label))
    return choices[:25]

@edit_plan.autocomplete("subject")
async def edit_subject_autocomplete(interaction: discord.Interaction, current: str):
    channels = get_subject_channels(interaction.guild)
    return [
        app_commands.Choice(name=ch.name, value=ch.name)
        for ch in channels if current.lower() in ch.name.lower()
    ][:25]

@edit_plan.autocomplete("category")
async def edit_category_autocomplete(interaction: discord.Interaction, current: str):
    candidates = ["宿題", "提出", "持ち物", "テスト", "その他"]
    return [app_commands.Choice(name=c, value=c) for c in candidates if current in c][:25]
```
- `target`・`subject`・`category`のそれぞれに、[00_ユーティリティ関数と_addコマンド.md](00_ユーティリティ関数と_addコマンド.md)で見たものと**全く同じロジック**のオートコンプリートが定義されています。`/add`・`/delete`・`/edit`で同じ種類の候補（科目一覧、カテゴリ候補）を使い回しているにも関わらず、共通関数として切り出さずにそれぞれのコマンドごとに個別定義されている点は、やや重複が見られる箇所です（動作上の問題はありませんが、保守性の観点では1つの共通関数にまとめる余地があります）。

---

次は、`/setchannel`（通知チャンネルの設定）と`/setup_roles`（通生/寮生の振り分けパネル）を解説します。 → [02_setchannel_setup_rolesコマンド.md](02_setchannel_setup_rolesコマンド.md)
