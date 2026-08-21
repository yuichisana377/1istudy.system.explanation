# `/list_cards`・`/get_card_set`・`/save_cards`（`bot.py` 4399〜4590行）

対象：`bot.py`の一覧取得・詳細取得・保存の3つのAPI。`/save_cards`は、単体のエンドポイントとしては`bot.py`全体でも屈指の複雑さです。

## 1. `/list_cards`：デッキ一覧（4399〜4407行）

```python
@app.route("/list_cards", methods=["GET"])
def list_cards():
    try:
        index, _ = load_cards_index()
        if index is None:
            index = rebuild_cards_index()
        return jsonify({"ok": True, "sets": index})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)})
```
- [[00_カードデータ層と索引管理.md]]で見た`cards_index.json`をそのまま返すだけです。もし索引ファイル自体がまだ存在しなければ、その場で`rebuild_cards_index`（全デッキファイルを読み直しての再構築）を行います。全デッキ本体を毎回読むのではなく、この軽量な索引だけを返すことで、一覧表示が高速に行えます。

## 2. `/get_card_set`：1デッキの詳細取得（4409〜4431行）

```python
@app.route("/get_card_set", methods=["GET"])
def get_card_set():
    filename = request.args.get("filename")
    ...
    data, _ = get_card_file(filename)
    if data is None:
        return jsonify({"ok": False, "error": "ファイルが見つかりません"})
    return jsonify({
        "ok": True, "filename": filename,
        "name": data.get("name", filename), "cards": data.get("cards", []),
        "subject": data.get("subject"), "folder_id": data.get("folder_id"),
        "published_by": (data.get("published_by") or {}).get("nickname"),
        "incomplete": bool(data.get("incomplete", False)),
        "choice_mode": data.get("choice_mode"),
    })
```
- 1つのデッキファイルの中身を丸ごと返します（カードの配列を含む、こちらは「重い」データです）。デッキを開いて実際に問題を見る画面で使われます。
- `(data.get("published_by") or {}).get("nickname")`… レスポンスとして返す公開者情報は`nickname`だけで、`id`（内部的な学籍番号）は含まれていません。誰でも呼べるこのAPIから、他人の学籍番号がそのまま漏れてしまわないようにするための配慮です。

## 3. `/save_cards`：デッキの保存（4433〜4590行）

これがこの章で最も複雑なエンドポイントです。新規作成・既存デッキの編集・「作成中」からの初回公開という、複数の異なる状況を1つのAPIでまとめて扱っています。

### 3-1. 入力の受け取り（4435〜4467行）

```python
    data     = request.json
    guild_id, _student_id, _nickname, err = require_login_json(data)
    if err:
        return err
    name, cards, filename, subject, folder_id = data.get("name"), data.get("cards"), data.get("filename"), data.get("subject"), data.get("folder_id")
```
- **重要な注意点がコメントに書かれています**：`publisher_id`/`publisher_nickname`は、他の「変更」APIとは違って**あえてクライアントの自己申告のまま信用しています**。
  ```python
  # ★ 注意：publisher_id/publisher_nicknameはあえてクライアント自己申告のまま
  #   信用している（ここをログインセッションの値で強制的に上書きすると、
  #   「他の人が公開したデッキに自分がカードを追加/編集しても、公開者表示は
  #   元の公開者のまま変わらない」という共同編集の仕様が壊れてしまうため）。
  publisher_id       = data.get("publisher_id")
  publisher_nickname = data.get("publisher_nickname") or "匿名"
  ```
  これは[[../05_ユーザーとセッション/01_セッショントークンと権限チェック.md]]で見た`require_login_json`の原則（自己申告のニックネームを信用しない）の**例外**です。理由は、CardMakerが「みんなでデッキを共同編集できる」機能を持っているためです。もし常にログインセッションの値で`publisher_nickname`を上書きしてしまうと、Aさんが公開したデッキをBさんが後から編集しただけで、「公開者」の表示がBさんに変わってしまいます。フロント側（`Cardmaker.js`）は、デッキの`published_by`を「元の公開者のまま維持する」ように意図的に送ってきているため、サーバー側もそれをそのまま尊重しています。この関数の呼び出し自体には`require_login_json`でログインを必須にしているため、「誰もログインしていないのに勝手な名前で保存できる」という根本的な問題自体は防がれています。
- `silent`（通知しない）、`incomplete`（作成中フラグ）、`first_publish`（後述）、`choice_mode`（選択式デッキか）といった、フロント側から送られてくる各種のフラグも受け取ります。

### 3-2. フォルダ移動制限とファイル名の決定（4481〜4508行）

```python
    is_update = bool(filename)
    if not filename:
        filename = generate_card_filename()

    sha = None
    existing_published_by = None
    if is_update:
        existing_data, sha = get_card_file(filename)
        if existing_data is not None:
            old_folder_id = existing_data.get("folder_id")
            folders_for_check, _ = load_card_folders()
            if _is_in_archive_scope(folders_for_check, old_folder_id) and not _is_in_archive_scope(folders_for_check, folder_id):
                return jsonify({"ok": False, "error": "クイズ過去問フォルダの外には移動できません"})
            existing_published_by = existing_data.get("published_by")
```
- `filename`が送られてきていれば既存デッキの更新（`is_update = True`）、無ければ`generate_card_filename()`で新しいファイル名を発行します。
- `_is_in_archive_scope(...)`… 「クイズ過去問」という特別なフォルダの中身は、その外に移動できないという制約がここで強制されています。コメントの通り、これは「ここが唯一のデッキ`folder_id`書き込み経路なので、サーバー側の実効的な強制はここで行う」という判断です。フロント側にも同様のガード（UI上で選べなくする）があるはずですが、それを迂回してAPIを直接叩かれた場合に備え、**サーバー側でも独立して同じルールを強制する**、これまでの回で何度も見てきたのと同じ考え方です。

```python
    final_publisher_id = (existing_published_by or {}).get("id") or publisher_id
```
- ここに、コメントで説明されている**実際にあった不具合の修正**が凝縮されています。
  ```python
  # ★ 追加：published_by.id は「デッキの作成者」を表す唯一の場所（削除の
  #   作成者確認機能で使う）。nicknameは元からクライアント側（deck.published_by
  #   キャッシュ）が「元の公開者のまま維持する」よう送ってきているが、idは
  #   syncDeckToServerが毎回“今ログインしている編集者自身”のstudent_idを
  #   送ってきてしまっており、他人のデッキを1回編集しただけで作成者IDが
  #   編集者にすり替わってしまっていた。既に published_by.id が記録されている
  #   更新（＝作成中の初回公開より後）では、クライアントの値を無視して
  #   元のidを維持する。初回公開時（記録がまだ無い場合）だけクライアントの
  #   publisher_idをそのまま採用する。
  ```
  `nickname`（表示名）は前述の通りクライアントが正しく「元の公開者のまま」送ってくる設計でしたが、`id`（[[../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md]]の削除依頼機能などで、「本当の作成者は誰か」を判定するために使われる、最も重要な情報）については、フロント側の実装に**バグ**があり、常に「今ログインしている編集者自身」のIDを送ってきてしまっていました。結果として、他人のデッキを1回編集しただけで、作成者IDが編集者本人にすり替わってしまうという不具合が起きていました。
  この行は、その不具合をサーバー側で吸収する修正です。`(existing_published_by or {}).get("id")`（既存データに記録されている、本来の作成者ID）が既にあれば、それを最優先で使い、クライアントから送られてきた`publisher_id`は**無視**します。既存の記録が無い場合（初めての公開）だけ、クライアントの値をそのまま採用します。**フロント側のバグを、サーバー側の1行のコードで無害化している**、実務でよくある種類の防御的なコードです。

### 3-3. 保存と索引の更新（4510〜4536行）

```python
    card_payload = {
        "name": name, "cards": cards, "subject": subject, "folder_id": folder_id,
        "published_by": {"id": final_publisher_id, "nickname": publisher_nickname},
        "incomplete": incomplete, "choice_mode": choice_mode,
    }
    try:
        put_card_file(filename, card_payload, sha)
    except DataWriteError as e:
        return jsonify({"ok": False, "error": f"local_write_failed: {e}"})

    index_change = None
    try:
        index_change = upsert_cards_index_entry(filename, card_payload)
    except DataWriteError as e:
        print(f"[WARN] cards_index の更新に失敗しました: {e}")
```
- デッキ本体（`words/{filename}`）を保存した後、[[00_カードデータ層と索引管理.md]]の`upsert_cards_index_entry`で索引も更新します。索引の更新が失敗しても、コメントの通り「カード本体の保存自体は成功しているので、索引更新の失敗は警告に留める」だけで、APIとしては成功扱いのまま処理を続けます。次回`/list_cards`が呼ばれたときに索引が再構築されるため、実害は小さいという判断です。**「主目的の処理」と「付随的な後処理」を明確に区別し、後者の失敗で前者の成功を台無しにしない**という設計は、このファイルを通して繰り返し登場するパターンです。

### 3-4. Discord通知（4538〜4578行）

```python
    if guild_id and not silent:
        try:
            ...
            is_actual_update = is_update and not bool(first_publish)
            action = "更新" if is_actual_update else "公開"
```
- `silent`が`True`なら通知はスキップされます（フロント側が下書きの自動保存など、通知するまでもない保存の際に使います）。
- **`first_publish`フラグの必要性**もコメントで説明されています：
  ```python
  # ★「作成中」として announceNewDeckToServer 経由で先にファイルだけ
  #   登録済みのデッキは、実際に公開したタイミングでも filename が
  #   既に存在するため、is_update（＝ファイルの有無）だけで判定すると
  #   「更新されました」という誤った通知文言になってしまう。
  ```
  CardMakerには「作成中」の未公開デッキという概念があり、これは既にファイルとして存在します。そのため、単純に「ファイルが既にあるかどうか（`is_update`）」だけで「更新」か「公開（新規）」かを判定すると、**初めて公開した瞬間なのに「更新されました」という誤った文言**になってしまいます。これを避けるため、フロント側が明示的に「これが初めての公開かどうか」を`first_publish`として伝えてきて、それを優先的に使います。

```python
            deck_url = f"{CARDMAKER_URL}?deck={filename}"
            embed = discord.Embed(
                title=f"📇 単語カードが{action}されました",
                description=(
                    f"[{name}]({deck_url})\n"
                    f"{publisher_nickname}さんによって{action}（{len(cards)}問）"
                ),
                color=discord.Color.blue(),
            )
```
- **リンクの埋め込み方にも過去の失敗が反映されています**：
  ```python
  # ★ 修正：プレーンテキストの「[デッキ名](url)」はDiscordの通常メッセージでは
  #   マスクされたリンクとして描画されない（そのまま文字列として表示されてしまう）ため、
  #   埋め込み（Embed）のdescriptionにマスクリンクとして書く形に変更した。
  ```
  Markdownの`[表示テキスト](url)`という書き方（マスクされたリンク、リンク先URLを隠して短い文言だけを表示する記法）は、Discordの**普通のメッセージ本文**では機能せず、そのままの文字列として表示されてしまいます。これは`discord.Embed`（埋め込みメッセージ）の`description`の中でのみ機能する、というDiscord特有の仕様上の制約です。これに気づかずに実装すると、リンクのつもりが「[デッキ名](https://...)」というただの記号の羅列がそのまま表示されてしまう、という失敗をします。この修正で、通知全体を`Embed`として組み立てる形に変更されています。

```python
            target_channel = get_subject_channel_by_name(guild, subject) if subject else None
            if not target_channel:
                config = load_config(guild_id_int)
                channel_id = config.get("notice_channel_id")
                target_channel = bot.get_channel(channel_id) if channel_id else None
            if target_channel:
                asyncio.run_coroutine_threadsafe(
                    target_channel.send(embed=embed), bot.loop
                ).result(timeout=10)
```
- 通知先のチャンネルは、まず科目に対応するチャンネル（`subject`があれば）を優先し、無ければ`/setchannel`で設定された「お知らせ用」チャンネル（`notice_channel_id`）にフォールバックします。

### 3-5. 運用ログへの記録（4580〜4590行）

```python
    is_actual_update = is_update and not bool(first_publish)
    old_deck_for_diff = existing_data if (is_update and existing_data) else None
    change = deck_file_diff(f"{CARDS_DIR}/{filename}", old_deck_for_diff, card_payload)
    detail = [c for c in (change, index_change) if c]  # ★ デッキ本体＋索引ファイルの両方の変更を載せる
    log_event(
        "card",
        f"カードデッキ「{name}」を{'更新' if is_actual_update else '公開'}しました（{len(cards)}問）。",
        actor=publisher_nickname,
        detail=detail if detail else None,
    )
    return jsonify({"ok": True, "filename": filename, "is_update": is_update})
```
- `detail = [c for c in (change, index_change) if c]`… [[00_カードデータ層と索引管理.md]]で見た「デッキ本体の差分」と「索引ファイルの差分」の**両方**を、運用ログの`detail`（複数ファイルの変更をまとめて表示できる形式）に含めています。これにより、Web側の運用ログを開くと、「デッキ本体のファイルで何が変わったか」と「索引ファイルのこのデッキのエントリがどう変わったか」の**両方**が、それぞれ折りたためる形で表示されます。

---

次は、他人のデッキを削除するときの本人確認（作成者判定）と、実際の削除処理を解説します。 → [[02_デッキ削除と作成者確認.md]]
