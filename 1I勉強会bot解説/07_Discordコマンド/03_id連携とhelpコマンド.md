# `/id連携`・`/help` コマンド（`bot.py` 1931〜2015行）

対象：`bot.py`の「/id連携」「/help」セクション。

## 1. `/id連携`：生徒IDとDiscordアカウントの連携（1931〜1995行）

コメントにある通り、これを一度実行してもらうことで、StudyLog側からのDM通知（3時間タイマー超過など）を本人のDiscordに直接送れるようになります。ブラウザのタブを閉じていても、他のサイトを見ていても、Discordアプリ側の通知として届きます（Discord自体の通知設定がオフの場合は届きません）。

なりすまし対策として、[../06_Discordアカウント連携/00_連携コードとOAuthステート.md](../06_Discordアカウント連携/00_連携コードとOAuthステート.md)で解説した「StudyLog側にログイン済みの状態でのみ発行できるワンタイムコード」を要求する方式になっています。生徒IDを知っているだけの第三者は、このコードを発行できないため、なりすまし連携はできません。

```python
@bot.tree.command(name="id連携", description="StudyLogで発行した連携コードを使って、DiscordアカウントをStudyLogと連携する")
@app_commands.describe(code="StudyLogにログインした状態で発行した連携コード（8桁・5分間有効）")
async def link_student_id(interaction: discord.Interaction, code: str):
    await interaction.response.defer(ephemeral=True)
    guild_id = interaction.guild.id

    sid = consume_link_code(guild_id, code.strip().upper())
    if not sid:
        await interaction.followup.send(
            "連携コードが無効か、期限切れです。...",
            ephemeral=True
        )
        return
```
- コマンド名自体が`"id連携"`という日本語です。discord.pyのスラッシュコマンドは日本語の名前も付けられ、Discord上には`/id連携`という形でそのまま表示されます。
- `code.strip().upper()`… ユーザーが入力したコードの前後の余分な空白を取り除き（`.strip()`）、大文字に統一（`.upper()`）してから検証します。[../06_Discordアカウント連携/00_連携コードとOAuthステート.md](../06_Discordアカウント連携/00_連携コードとOAuthステート.md)の`_generate_link_code`で見た通り、コードは元々大文字と数字だけで生成されますが、生徒がスマホのソフトウェアキーボードなどでうっかり小文字で入力してしまっても、正しく認識できるようにするための配慮です。
- `consume_link_code`（前の回で解説した「1回使い切り」のコード検証関数）が`None`を返した場合（コードが無効・期限切れ・既に使用済み）は、その旨を伝えて処理を打ち切ります。

```python
    users = load_users(guild_id)
    matched = next((u for u in users if u["id"] == sid), None)
    if not matched:
        await interaction.followup.send(
            "連携コードに対応する生徒データが見つかりませんでした。...",
            ephemeral=True
        )
        return
    nickname = matched.get("nickname", sid)
```
- コードから取り出せた学籍番号（`sid`）が、実際に`users_{guild_id}.json`に存在する生徒かどうかを確認します。（コードの発行自体がログイン済みの本人に限られているため、通常はここで見つからないことは無いはずですが、念のための整合性チェックです。）
- `matched.get("nickname", sid)`… ニックネームが見つかればそれを、無ければ学籍番号自体をニックネーム代わりに使う、というフォールバックです。

```python
    try:
        links = load_discord_links(guild_id)
        links[sid] = interaction.user.id
        save_discord_links(guild_id, links)
    except DataWriteError as e:
        await interaction.followup.send(f"連携の保存に失敗しました（データ保存エラー）: {e}", ephemeral=True)
        return
```
- ここが実際の連携本体です。`discord_links_{guild_id}.json`（DM通知用の対応表）に、`{学籍番号: 今コマンドを実行したDiscordユーザーのID}`を書き加えて保存します。`interaction.user.id`が「今このコマンドを実行している、Discord上の本人のユーザーID」です。

```python
    try:
        await interaction.user.send(f"{sid}の{nickname}さんの通知登録が完了しました。")
        await interaction.followup.send(
            f"連携が完了しました！ 確認のDMを送信しましたので届いているか確認してください。",
            ephemeral=True
        )
    except discord.Forbidden:
        await interaction.followup.send(
            "連携情報は保存しましたが、確認DMを送れませんでした。\n"
            "サーバーアイコンを右クリック →「プライバシー設定」→「ダイレクトメッセージ」をオンにしてから、"
            "StudyLogでもう一度コードを発行し /id連携 を実行してください。",
            ephemeral=True
        )
    except Exception as e:
        await interaction.followup.send(
            f"連携情報は保存しましたが、確認DMの送信中にエラーが発生しました: {e}",
            ephemeral=True
        )
```
- コメントの通り、連携が正しく機能するかをその場で確認できるように、**試しに確認のDMを送信**しています。これは単なる「完了しました」の通知であると同時に、「DM通知が本当に届く状態になっているか」の実地テストも兼ねています。
- `interaction.user.send(...)`… Discordでは、ユーザー側のプライバシー設定によっては、Botから一方的にDMを送ることが拒否されることがあります。その場合は`discord.Forbidden`という例外が発生します。この場合、連携情報自体は既に保存済みであることを伝えつつ、DMを受け取れるようにする設定変更の手順（サーバーごとのプライバシー設定でDMを許可する）を具体的に案内しています。
- それ以外の予期しないエラー（Discord側の一時的な障害など）は、`except Exception as e:`でまとめて捕まえ、少なくとも連携情報自体は保存済みであることを伝えます。
- **どのケースでも、連携情報の保存自体は既に成功しているという事実を先に確定させ、DMが送れるかどうかは別問題として扱っている**点が設計上のポイントです。DM送信の失敗によって、連携という主目的の処理まで巻き込まれて失敗扱いになることはありません。

## 2. `/help`：コマンド一覧（1998〜2014行）

```python
@bot.tree.command(name="help", description="使えるコマンド一覧")
async def help_command(interaction: discord.Interaction):
    msg = (
        "📘 **使えるコマンド一覧**\n\n"
        "**/add** — 予定を登録する\n"
        "**/list** — 予定を表示する\n"
        "**/delete** — 予定を削除する\n"
        "**/edit** — 予定を編集する\n"
        "**/setchannel** — 通知チャンネルを設定する（通生／寮生／お知らせ用を選択可）\n"
        "**/setup_roles** — 通生/寮生 振り分けパネルを投稿する\n"
        "**/id連携** — StudyLogにログインして発行した連携コードを使い、DiscordアカウントをStudyLogと連携する（DM通知を受け取れるようになる）\n"
        "**webページ** - https://1istudyweb.pages.dev/\n"
    )
    await interaction.response.send_message(msg, ephemeral=True)
```
- これまで見てきた全コマンドを簡潔にまとめた、ヘルプメッセージを1本の文字列として組み立てて返すだけの、シンプルなコマンドです。
- `await interaction.response.send_message(msg, ephemeral=True)`… ここまでの他のコマンドと違い、`defer`（先延ばし）を使わず、直接`response.send_message`で即座に応答しています。これは、この処理が外部との通信やファイルの読み書きを一切行わない、一瞬で終わる処理だからです。3秒以内の応答が確実にできる処理では、わざわざ`defer`で一旦「処理中です」を挟む必要はありません。

---

これでDiscordのスラッシュコマンド群（`/add`・`/list`・`/delete`・`/edit`・`/setchannel`・`/setup_roles`・`/id連携`・`/help`）を一通り見ました。次は、時間になったら自動的に動く「自動通知」と、勉強タイマーの監視ループを解説します。 → [../08_自動通知とスケジューラー監視/00_定期通知の仕組み.md](../08_自動通知とスケジューラー監視/00_定期通知の仕組み.md)
