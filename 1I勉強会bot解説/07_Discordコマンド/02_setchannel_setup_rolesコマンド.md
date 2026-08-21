# `/setchannel`・`/setup_roles`・リアクション振り分け（`bot.py` 1773〜1929行）

対象：`bot.py`の「/setchannel」「/setup_roles（通生/寮生 振り分けパネル）」セクション。

## 1. `/setchannel`：通知チャンネルの設定（1777〜1807行）

```python
@bot.tree.command(name="setchannel", description="通知チャンネルを設定する")
@app_commands.describe(type="どの通知に使うチャンネルか（省略時は通生）")
@app_commands.choices(type=[
    app_commands.Choice(name="通生（朝5:30 / 夜20:00）", value="commute"),
    app_commands.Choice(name="寮生（朝7:20 / 夜20:00）", value="dorm"),
    app_commands.Choice(name="お知らせ用", value="main"),
])
async def setchannel(interaction: discord.Interaction, type: app_commands.Choice[str] = None):
    ...
    kind = type.value if type else "commute"
    if kind == "dorm":
        config["remind_channel_id_dorm"] = interaction.channel.id
        label = "寮生（朝7:20）"
    elif kind == "main":
        config["notice_channel_id"] = interaction.channel.id
        label = "お知らせ用"
    else:
        config["remind_channel_id"] = interaction.channel.id
        label = "通生（朝5:30・夜20:00）"
    ...
```
- `@app_commands.choices(type=[...])`… `type`引数を自由入力ではなく、あらかじめ決められた3つの選択肢（`Choice`）からしか選べないようにする指定です。[00_ユーティリティ関数と_addコマンド.md](00_ユーティリティ関数と_addコマンド.md)のオートコンプリート（自由入力＋候補表示）とは違い、こちらは**候補以外を入力できない**という、より厳格な制約です。通知チャンネルの種類は決め打ちで数が少ないため、こちらの方式が使われています。
- `type: app_commands.Choice[str] = None`… デフォルト値が`None`なので、省略も可能です。省略時は`kind = type.value if type else "commute"`により`"commute"`（通生）扱いになります。
- 選ばれた種類に応じて、`config`（このguildの設定）の対応するフィールドに、**今このコマンドを実行したチャンネルのID**（`interaction.channel.id`）を保存します。「このコマンドを実行したチャンネル自体が、その種類の通知先になる」という直感的な設定方法です。
- 保存された`config`は、後述する自動通知の章（`send_tomorrow_plans`など）で、「どのチャンネルに通知を送るか」を決めるために読み込まれることになります。

## 2. `/setup_roles`：通生/寮生の振り分けパネル（1811〜1867行）

これは、生徒が絵文字のリアクションを押すだけで「通生」または「寮生」のロール（Discordのメンバーに付けられる役職・グループのようなもの）を自動的に付け外しできるようにする機能です。

```python
@bot.tree.command(name="setup_roles", description="通生/寮生 振り分けパネルを投稿します")
@app_commands.describe(通生ロール="通生に付与するロール", 寮生ロール="寮生に付与するロール")
@app_commands.checks.has_permissions(manage_roles=True)
async def setup_roles(
    interaction: discord.Interaction,
    通生ロール: discord.Role,
    寮生ロール: discord.Role,
):
```
- `@app_commands.checks.has_permissions(manage_roles=True)`… このコマンドは「ロールの管理」権限を持つメンバー（先生や運営メンバーなど）しか実行できないように制限されています。一般の生徒が誤って（あるいは悪意を持って）ロール振り分けパネルを何度も投稿できてしまわないようにするための制約です。
- 引数の名前が`通生ロール`・`寮生ロール`と日本語になっている点にも注目してください。Pythonの変数名・関数の引数名は日本語（Unicode識別子）も使うことができ、ここではDiscordの入力画面にそのまま表示される引数名を分かりやすくするために、あえて日本語名にしてあります。型は`discord.Role`で、Discord側のロール選択UIから直接ロールを指定できるようになっています。

```python
    if 通生ロール >= guild.me.top_role or 寮生ロール >= guild.me.top_role:
        await interaction.followup.send(
            "ロールの順序を確認してください。Botの役職を、通生・寮生ロールより上に配置する必要があります。",
            ephemeral=True,
        )
        return
```
- Discordのロールには「上下関係（階層）」があり、**Botは自分より上位のロールを他人に付け外しすることができません**（Discordの仕様上の制約）。ここで事前にその条件をチェックし、もし設定ミス（Botの役職が通生・寮生ロールより下に置かれている）があれば、実際にパネルを投稿する前にエラーとして伝えています。`guild.me`は「このサーバーにおけるBot自身のメンバー情報」、`top_role`は「そのメンバーが持つロールの中で最も高いもの」です。

```python
    embed = discord.Embed(
        title="通生 / 寮生 登録",
        description=(
            f"{EMOJI_COMMUTER} → 通生\n"
            f"{EMOJI_DORM} → 寮生\n\n"
            "どちらか当てはまる方にリアクションしてください。"
        ),
        color=discord.Color.teal(),
    )
    msg = await interaction.channel.send(embed=embed)
    await msg.add_reaction(EMOJI_COMMUTER)
    await msg.add_reaction(EMOJI_DORM)
```
- `discord.Embed`… Discordの「埋め込みメッセージ」（枠線付きで、タイトルや色を付けて装飾できる特別なメッセージ形式）を作ります。
- メッセージを送信した後、Bot自身が`add_reaction`で🚃と🏠の絵文字を最初にリアクションとして付けておきます。これにより、生徒はそのリアクションをクリックするだけで、自分も同じ絵文字でリアクションできるようになります（絵文字を1から自分で選ぶ必要がありません）。

```python
    config = await async_load_config(guild.id)
    config["role_panel_message_id"] = msg.id
    config["role_panel_channel_id"] = msg.channel.id
    config["commuter_role_id"] = 通生ロール.id
    config["dorm_role_id"] = 寮生ロール.id
    try:
        await async_save_config(guild.id, config)
    except DataWriteError as e:
        ...
```
- 投稿したパネルのメッセージID・チャンネルID、そして指定された2つのロールのIDを`config`に保存します。これにより、後で誰かがこのパネルにリアクションしたときに、「これはどのメッセージへのリアクションで、どのロールを付け外しすればよいか」を照合できるようになります。

```python
@setup_roles.error
async def setup_roles_error(interaction: discord.Interaction, error: app_commands.AppCommandError):
    if isinstance(error, app_commands.MissingPermissions):
        await interaction.response.send_message(
            "このコマンドには「ロールの管理」権限が必要です。", ephemeral=True
        )
    else:
        ...
```
- `@setup_roles.error`… `setup_roles`コマンドの実行中にエラーが起きたときに呼ばれる、専用のエラーハンドラーです。特に`app_commands.MissingPermissions`（先ほどの権限チェックに引っかかった場合に自動的に発生する例外）を`isinstance`で判定し、分かりやすい日本語のメッセージに変換して返しています。それ以外の予期しないエラーは、そのままエラー内容を表示します。

## 3. リアクションの監視と実際のロール付け外し（1870〜1929行）

```python
async def _handle_role_reaction(payload: discord.RawReactionActionEvent, add: bool):
    if payload.guild_id is None:
        return

    config = await async_load_config(payload.guild_id)
    panel_message_id = config.get("role_panel_message_id")
    if not panel_message_id or payload.message_id != panel_message_id:
        return

    emoji = str(payload.emoji)
    if emoji not in (EMOJI_COMMUTER, EMOJI_DORM):
        return
```
- `RawReactionActionEvent`… メッセージへのリアクションの追加・削除という「生の」イベント情報です。「生の」と付いているのは、Discord.pyには「メッセージがBotのキャッシュに載っていなくても発生する」低レベルなイベント（Raw版）と、「キャッシュに載っているメッセージに対してだけ発生する」より扱いやすい高レベルなイベントの2種類があり、ここでは前者（Raw版）を使っているためです。パネルメッセージがBotの起動後しばらく経ってキャッシュから外れていても、確実にリアクションを検知できるようにするためです。
- **早期リターンの積み重ね**によるフィルタリングになっています。DMでのリアクション（`guild_id is None`）は無視し、保存されているパネルのメッセージID（`role_panel_message_id`）と一致しないメッセージへのリアクションも無視し、🚃🏠以外の絵文字も無視します。これにより、サーバー内のあらゆるメッセージへの、あらゆるリアクションのたびにこの関数が呼ばれても、関係の無いものはごく短時間で処理を終えて無視できます。

```python
    guild = bot.get_guild(payload.guild_id)
    if guild is None:
        return

    member = guild.get_member(payload.user_id)
    if member is None or member.bot:
        return

    commuter_role = guild.get_role(config.get("commuter_role_id"))
    dorm_role = guild.get_role(config.get("dorm_role_id"))
    channel_id = config.get("role_panel_channel_id")
    channel = guild.get_channel(channel_id) if channel_id else None
```
- `member.is None or member.bot`… リアクションした相手がメンバー情報を取得できない、またはBot自身（もしくは他のBot）である場合は無視します。Botが自分自身のリアクション（`setup_roles`の中で最初に付けた🚃🏠）に反応して無限ループのような処理をしてしまわないようにするための、重要なガードです。

```python
    try:
        if add:
            if emoji == EMOJI_COMMUTER and commuter_role:
                await member.add_roles(commuter_role, reason="通生登録")
                if dorm_role and dorm_role in member.roles:
                    await member.remove_roles(dorm_role, reason="通生に変更のため")
                    if channel:
                        msg = await channel.fetch_message(panel_message_id)
                        await msg.remove_reaction(EMOJI_DORM, member)
            elif emoji == EMOJI_DORM and dorm_role:
                await member.add_roles(dorm_role, reason="寮生登録")
                if commuter_role and commuter_role in member.roles:
                    await member.remove_roles(commuter_role, reason="寮生に変更のため")
                    if channel:
                        msg = await channel.fetch_message(panel_message_id)
                        await msg.remove_reaction(EMOJI_COMMUTER, member)
        else:
            if emoji == EMOJI_COMMUTER and commuter_role:
                await member.remove_roles(commuter_role, reason="通生リアクション解除")
            elif emoji == EMOJI_DORM and dorm_role:
                await member.remove_roles(dorm_role, reason="寮生リアクション解除")
    except discord.Forbidden:
        pass
```
- `add`（リアクションが「追加」されたのか「削除」されたのか）で処理を分けます。
- リアクションが追加された場合（`add=True`）：対応するロールを付与し、**もう片方のロールを既に持っていれば、そちらは自動的に外します**。通生と寮生は普通どちらか一方のはずなので、両方のロールが同時に付いた状態にならないようにする、という業務ロジックです。さらに、外した方のロールに対応する自分のリアクションも、`msg.remove_reaction(EMOJI_DORM, member)`のように**サーバー側から強制的に取り消して**います。これにより、パネル上の見た目（誰がどちらにリアクションしているか）も、実際のロールの状態と食い違わないように保たれます。
- リアクションが取り消された場合（`add=False`）：対応するロールを単純に外すだけです。
- `except discord.Forbidden: pass`… `discord.Forbidden`は、Bot自身の権限不足でロールの付け外しに失敗したときに発生する例外です。この例外を`pass`（何もしない）で握りつぶし、処理全体をクラッシュさせずに終わらせています。前段で「Botより上位のロールでないか」を`setup_roles`実行時に確認していますが、その後でサーバーの権限設定が変更される可能性もあるため、実際の操作時にも念のため同じ種類のエラーに備えている、という2段構えの防御です。

```python
@bot.event
async def on_raw_reaction_add(payload: discord.RawReactionActionEvent):
    await _handle_role_reaction(payload, add=True)

@bot.event
async def on_raw_reaction_remove(payload: discord.RawReactionActionEvent):
    await _handle_role_reaction(payload, add=False)
```
- `@bot.event`… discord.pyが用意している、特定の種類のイベントが起きるたびに自動的に呼び出される関数を登録するデコレータです。`on_raw_reaction_add`/`on_raw_reaction_remove`という決まった名前の関数を定義しておくと、サーバー内のどこかでリアクションが追加/削除されるたびに、discord.pyのライブラリ側から自動的に呼び出されます。ここでは、実際の処理は先ほどの`_handle_role_reaction`に委譲し、`add`引数だけを`True`/`False`で使い分けています。

---

次は、「/id連携」コマンドと「/help」コマンドを解説します。 → [03_id連携とhelpコマンド.md](03_id連携とhelpコマンド.md)
