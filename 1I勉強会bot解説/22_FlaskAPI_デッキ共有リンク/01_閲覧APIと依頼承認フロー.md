# 共有デッキの閲覧APIと、本人以外からの依頼承認フロー（`bot.py` 5162〜5426行）

前回（[00_共有リンクの発行と取り消し.md](00_共有リンクの発行と取り消し.md)）は共有リンクの発行・一覧・取り消し、および全デッキ横断の一覧API（`/list_my_deck_shares`）を見ました。今回は、実際にリンクを開いたときに使われる閲覧API（`/deck_share_info`）と、デッキの作成者本人以外が共有したい場合の依頼〜承認フロー（`/request_deck_share`〜`/respond_deck_share_request`）を見ていきます。仕組み全体は[12_FlaskAPI_削除依頼システム](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)の削除確認依頼とほぼ同じ形をしています。

## 1. `/deck_share_info`：ログイン不要の閲覧API（5162〜5191行）

```python
@app.route("/deck_share_info", methods=["GET"])
def deck_share_info():
    token = request.args.get("token", "")
    guild_id = _parse_share_token_guild(token)
    if guild_id is None:
        return jsonify({"ok": False, "error": "リンクが正しくありません。"})
    items, _ = load_deck_shares(guild_id)
    entry = next((it for it in items if it.get("token") == token), None)
    if not entry:
        return jsonify({"ok": False, "error": "リンクが無効です。"})
    if entry.get("revoked_at"):
        return jsonify({"ok": False, "error": "このリンクは取り消されました。"})
    if entry.get("expires_at", 0) <= time.time():
        return jsonify({"ok": False, "error": "リンクの有効期限が切れています。"})
    data, _ = get_card_file(entry["filename"])
    if not data:
        return jsonify({"ok": False, "error": "デッキが見つかりません。作成者が削除した可能性があります。"})
    return jsonify({
        "ok": True,
        "name": data.get("name", entry.get("deck_name")),
        "subject": data.get("subject"),
        "cards": data.get("cards", []),
        "choice_mode": data.get("choice_mode"),
        "incomplete": bool(data.get("incomplete", False)),
        "shared_by": entry.get("created_by_nickname"),
        "expires_at": entry.get("expires_at"),
    })
```
- このAPIは**ログイン処理を一切呼びません**（`require_login_json`も`require_member_session`も出てきません）。トークンさえ分かれば誰でも呼べる、というのがこの機能の目的そのものだからです（トークンが「合言葉」の役割を果たします）。
- チェックの順序に注目してください：①`_parse_share_token_guild`でtokenの形式そのものが正しいか（guild_id部分を取り出せるか）→②`deck_shares_{guild_id}.json`の中に一致するエントリがあるか→③取り消し済みでないか→④期限切れでないか→⑤対象のデッキファイルが今も実在するか、の5段階です。存在しない・無効・取り消し済み・期限切れ・デッキ削除済みのどの場合も、理由をそれぞれ違うメッセージで返している点にも注目してください（[../../1I勉強会web解説/12_DeckShare/00_解説.md](../../1I勉強会web解説/12_DeckShare/00_解説.md)で見るDeckShare.htmlは、これをそのままエラー画面に表示します）。
- レスポンスに含まれるフィールドが**閲覧に必要な最小限**である点も重要です。`get_card_set`（[14_FlaskAPI_CardMaker/01_一覧取得と保存API.md](../14_FlaskAPI_CardMaker/01_一覧取得と保存API.md)）が返す`filename`（内部のファイル名）は含まれていません。これは、共有された側にファイル名という「内部識別子」を渡す必要が無い（渡すと将来的に別のAPIと組み合わせて悪用される余地を増やすだけ）という考え方です。`published_by`（本来の公開者情報）の代わりに`shared_by`（今回のリンクを発行した人のニックネームだけ）を返しているのも同様の理由です。

## 2. `/request_deck_share`：本人以外からの依頼（5193〜5282行）

```python
@app.route("/request_deck_share", methods=["POST"])
def request_deck_share():
    data = request.json or {}
    guild_id, requester_id, requester_nickname, err = require_login_json(data)
    ...
    deck_data, _ = get_card_file(filename)
    if not deck_data:
        return jsonify({"ok": False, "error": "ファイルが見つかりません"})
    owner_id, owner_nickname = _deck_owner(filename)
    if not owner_id:
        return jsonify({"ok": False, "error": "作成者が記録されていないため、確認を送れません。"})
    if str(owner_id) == str(requester_id):
        return jsonify({"ok": False, "error": "本人はこの手続きを使わず直接共有リンクを作成できます。"})
```
- ここまでは[12_FlaskAPI_削除依頼システム](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)の`/request_delete`とほぼ同じ構造（ログイン必須・理由の必須/文字数/不正文字チェック・作成者不明なら拒否・本人なら案内）です。

```python
    # ★ 依頼時点で、承認後にリンクを持つことになる依頼者本人の本日の上限を
    #   あらかじめ確認しておく（上限に達しているのに依頼だけ飛んでしまい、
    #   承認後に無駄になる事故を避ける。承認後の発行時にも再確認している）。
    share_items, _ = load_deck_shares(guild_id)
    if _deck_share_count_today(share_items, requester_id) >= DECK_SHARE_DAILY_LIMIT:
        return jsonify({"ok": False, "error": "share_limit_reached"})
```
- 削除依頼には無い、この機能特有のチェックです。依頼を送る時点で、承認後に実際にリンクを受け取ることになる**依頼者本人**の1日の上限を先読みしています。コメントの通り、これは必須のチェックではありません（発行時の`/create_deck_share`側でも同じチェックが行われるため、無くても壊れません）が、無いと「わざわざ作成者に確認DMを送ったのに、依頼者の上限が既に切れていて結局発行できない」という無駄が起こり得るため、事前に弾いています。

```python
    token = create_share_request_token({
        "guild_id": guild_id, "filename": filename, "deck_name": deck_name,
        "owner_id": owner_id, "owner_nickname": owner_nickname,
        "requester_id": requester_id, "requester_nickname": requester_nickname,
        "reason": reason,
    })
    review_url = f"{SHARE_APPROVAL_URL}?token={token}"
    message = (
        f"{requester_nickname}さんが、あなたが作成したカードデッキ\n"
        f"「{deck_name}」の外部共有リンク発行を依頼しています。\n\n"
        ...
    )
    notified_via = "discord_dm"
    try:
        send_discord_dm(guild_id, owner_id, "🔗 共有の確認依頼", message)
    except Exception:
        try:
            items, sha = load_pending_share_requests(guild_id)
            items.append({...})
            save_pending_share_requests(guild_id, items, sha)
        except DataWriteError as write_err:
            return jsonify({"ok": False, "error": f"local_write_failed: {write_err}"})
        notified_via = "web_pending"
```
- ステートレストークンを発行してDMを送り、失敗したら`pending_share_requests_{guild_id}.json`に控えを残す、という二段構えは[12_FlaskAPI_削除依頼システム](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)の`/request_delete`と全く同じパターンです（`except Exception`で理由を問わず全ての送信失敗を同じフォールバックに乗せる方針も含めて）。この控えは`/pending_share_requests`（次項）で拾われ、`PendingShareCheck.js`（[../../1I勉強会web解説/12_DeckShare/00_解説.md](../../1I勉強会web解説/12_DeckShare/00_解説.md)参照。`PendingDeleteCheck.js`と同じ構造の共通スクリプト）が次回サイトを開いたときに確認モーダルを出します。

## 3. `/pending_share_requests`（5284〜5314行）

```python
@app.route("/pending_share_requests", methods=["GET"])
def pending_share_requests():
    ...
    items, _ = load_pending_share_requests(guild_id)
    now = time.time()
    mine = [
        it for it in items
        if str(it.get("owner_id")) == str(student_id)
        and now - it.get("created_at", 0) <= SHARE_REQUEST_TOKEN_TTL_SEC
    ]
    return jsonify({"ok": True, "requests": [...]})
```
- `/pending_delete_requests`（[12_FlaskAPI_削除依頼システム/01_承認ページのAPIと結果の反映.md](../12_FlaskAPI_削除依頼システム/01_承認ページのAPIと結果の反映.md)）と全く同じ構造です。ログイン中の本人（`owner_id`が自分と一致するもの）宛の、まだトークンが有効期限内（`SHARE_REQUEST_TOKEN_TTL_SEC`＝14日）の依頼だけを返します。

## 4. `/share_request_info`・`/respond_deck_share_request`（5316〜5425行）

```python
@app.route("/share_request_info", methods=["GET"])
def share_request_info():
    token = request.args.get("token", "")
    payload = resolve_share_request_token(token)
    if not payload:
        return jsonify({"ok": False, "error": "リンクが無効か、期限切れです。"})
    filename = payload.get("filename")
    exists = os.path.isfile(_data_path(f"{CARDS_DIR}/{filename}"))
    deck_name = payload.get("deck_name")
    if exists:
        current, _ = get_card_file(filename)
        if current:
            deck_name = current.get("name") or deck_name
    return jsonify({
        "ok": True, "deck_name": deck_name,
        "requester_nickname": payload.get("requester_nickname"),
        "reason": payload.get("reason"),
        "already_gone": not exists,
    })
```
- `ShareApproval.html`（DMのリンク先）が表示内容を取得するためのAPIです。`_delete_target_summary`と同じ考え方で、依頼時点のデッキ名ではなく**閲覧時点の最新のデッキ名**を出そうとします（デッキが既に削除されていれば、依頼時点にトークンへ埋め込んでおいた名前にフォールバックします）。ログイン不要な理由も削除確認と同じで、「DMで本人にだけ届くリンク」であること自体を本人確認の代わりにしています。

```python
@app.route("/respond_deck_share_request", methods=["POST"])
def respond_deck_share_request():
    data = request.json or {}
    payload = resolve_share_request_token(data.get("token", ""))
    ...
    if action == "approve":
        if not os.path.isfile(_data_path(f"{CARDS_DIR}/{filename}")):
            return jsonify({"ok": False, "error": "デッキが見つかりません。既に削除されている可能性があります。"})
        items, sha = load_share_grants(guild_id)
        now = int(time.time())
        items.append({
            "filename": filename, "requester_id": requester_id, "requester_nickname": requester_nickname,
            "owner_id": owner_id, "owner_nickname": owner_nickname,
            "granted_at": now, "expires_at": now + SHARE_GRANT_TTL_SEC, "used_at": None,
        })
        save_share_grants(guild_id, items, sha)
        log_event(...)
        result_message = "承認しました。依頼者が共有リンクを発行できるようになりました。"
    else:
        log_event(...)
        result_message = "却下しました。依頼者にはその旨が伝わります。"
```
- **ここが削除確認依頼との一番大きな違いです**。`/respond_delete_request`は承認と同時に実際の削除処理（`_delete_card_deck_file`）を実行しますが、`respond_deck_share_request`は承認しても**共有リンクそのものは作りません**。代わりに`deck_share_grants_{guild_id}.json`へ「このデッキ・この依頼者に対する、未使用・14日以内有効なグラント」を1件追加するだけです。実際にリンクが発行されるのは、依頼者が改めてCardMakerで「共有リンクを作る」を押し、[00_共有リンクの発行と取り消し.md](00_共有リンクの発行と取り消し.md)で見た`/create_deck_share`が`_find_usable_grant`でこのグラントを見つけたときです。
- なぜこの1段階を挟むのか（コメントより）：承認から発行までの間に**Discord DMという「失敗しうる経路」を挟みたくなかった**ためです。もし承認と同時にリンクを発行して、それをDMで依頼者に届ける設計にしていたら、そのDMが届かなかった場合（依頼者がDiscord未連携等）に「リンクは存在するのに、誰も知らない」という状態になってしまいます。グラント方式なら、DMが失敗しても依頼者はいつでも自分でCardMakerのボタンを押し直すだけで発行できるため、詰まる経路がありません。

```python
    if guild_id and requester_id:
        try:
            if action == "approve":
                outcome_msg = (
                    f"「{deck_name}」の共有リンク発行依頼が{owner_nickname}さんに承認されました。\n"
                    f"CardMakerでこのデッキのメニューから「共有リンクを作る」をもう一度押すと発行できます。"
                )
            else:
                outcome_msg = f"「{deck_name}」の共有リンク発行依頼は{owner_nickname}さんに却下されました。"
            send_discord_dm(int(guild_id), requester_id, "🔗 共有依頼の結果", outcome_msg)
        except Exception:
            pass
```
- 依頼者への結果通知は`/respond_delete_request`と同じく**ベストエフォート**（失敗しても承認/却下そのものは成立させる）です。ただし削除確認依頼と違い、この通知が届かなくても実害はほぼ無いことが上のコメントで明言されています（依頼者はCardMaker側で気づいて自分でボタンを押せばよいため）。

---

これでサーバー側（`bot.py`）の解説は終わりです。CardMaker側のUI（デッキメニューの「共有リンクを作る」・依頼フォーム）と、ログイン不要の閲覧ページ`DeckShare.html`・承認ページ`ShareApproval.html`については、[../../1I勉強会web解説/12_DeckShare/00_解説.md](../../1I勉強会web解説/12_DeckShare/00_解説.md)を参照してください。
