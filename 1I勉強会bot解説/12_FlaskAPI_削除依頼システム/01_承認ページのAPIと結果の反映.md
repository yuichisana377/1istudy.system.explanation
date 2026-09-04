# 承認ページのAPIと結果の反映（`bot.py` 3766〜3891行）

対象：`bot.py`の`/pending_delete_requests`・`/delete_request_info`・`/respond_delete_request`。削除依頼システムの後半、「依頼された側」が確認・応答する部分です。

## 1. `/pending_delete_requests`：Web確認待ち一覧（3766〜3798行）

```python
@app.route("/pending_delete_requests", methods=["GET"])
def pending_delete_requests():
    """ログイン中の本人宛の、Web確認待ちの削除依頼一覧を返す
    （/request_deleteがDiscord未連携でDMを送れなかったケースの受け皿）。
    PendingDeleteCheck.jsがサイトを開くたびに呼び、あれば確認モーダルを出す。"""
    guild_id = request.args.get("guild_id")
    if not guild_id:
        return jsonify({"ok": False, "error": "missing guild_id"})
    try:
        guild_id = int(guild_id)
    except (TypeError, ValueError):
        return jsonify({"ok": False, "error": "invalid guild_id"})
    student_id, err = require_member_session(session_token_from_request(), guild_id)
    if err:
        return err

    items, _ = load_pending_delete_requests(guild_id)
    now = time.time()
    mine = [
        it for it in items
        if str(it.get("owner_id")) == str(student_id)
        and now - it.get("created_at", 0) <= DELETE_REQUEST_TOKEN_TTL_SEC
        # ★ トークンと同じ期限で表示から外す
    ]
    return jsonify({"ok": True, "requests": [
        {
            "token": it.get("token"), "category": it.get("category"),
            "target_name": it.get("target_name"), "requester_nickname": it.get("requester_nickname"),
            "reason": it.get("reason"),
        }
        for it in mine
    ]})
```
- [00_削除依頼トークンと送信フロー.md](00_削除依頼トークンと送信フロー.md)で見た`/request_delete`が、DM送信に失敗した場合に`pending_delete_requests_{guild_id}.json`へ控えていた依頼を、**ログイン中の本人宛の分だけ**絞り込んで返すAPIです。フロント側の共通コンポーネント`PendingDeleteCheck.js`（[../../1I勉強会web解説](../../1I勉強会web解説/01_index_予定管理.md)で紹介済み）が、サイトを開くたびにこのAPIを呼び、該当する依頼があれば確認モーダルを表示します。
- `str(it.get("owner_id")) == str(student_id)`… 保存されている全guildの依頼の中から、今ログインしている本人（`student_id`）が作成者（`owner_id`）である依頼だけを取り出します。
- `now - it.get("created_at", 0) <= DELETE_REQUEST_TOKEN_TTL_SEC`… トークンの有効期限（14日）と**全く同じ期限**で、表示対象から外しています。コメントの通り、これはトークン自体が既に期限切れになっている依頼を、いつまでも確認待ちとして表示し続けないようにするための整合性です。もしこの期限チェックが無いと、既に無効なトークンへのリンクを含む確認モーダルが表示され続けてしまい、押しても「リンクが無効です」というエラーにしかならない、という不親切な状態になります。
- レスポンスに含めるフィールドは、承認ページで表示するために最低限必要なものだけに絞られています（`owner_id`のような内部的な情報は含まれません）。

## 2. `/delete_request_info`：承認ページ向けの情報取得（3800〜3827行）

```python
@app.route("/delete_request_info", methods=["GET"])
def delete_request_info():
    """
    トークンさえ分かれば閲覧できる（DMで本人にだけ届く前提）ため、
    ログイン不要にしてある。"""
    token = request.args.get("token", "")
    payload = resolve_delete_request_token(token)
    if not payload:
        return jsonify({"ok": False, "error": "リンクが無効か、期限切れです。"})
    category = payload.get("category")
    filename = payload.get("filename")
    target_name, detail_lines = _delete_target_summary(category, filename)
    exists = (
        os.path.isfile(_data_path(f"{CARDS_DIR}/{filename}")) if category == "deck"
        else os.path.isfile(_data_path(f"{NOTICES_DIR}/{filename}"))
    )
    return jsonify({
        "ok": True, "category": category, "target_name": target_name,
        "detail_lines": detail_lines,
        "requester_nickname": payload.get("requester_nickname"),
        "owner_nickname": payload.get("owner_nickname"),
        "reason": payload.get("reason"),
        "requested_at": payload.get("_t"),
        "already_gone": not exists,
    })
```
- これが、[../../1I勉強会web解説/10_DeleteApproval/00_DeleteApproval解説.md](../../1I勉強会web解説/10_DeleteApproval/00_DeleteApproval解説.md)で見た`DeleteApproval.js`の`init()`が呼んでいるAPIの実体です。コメントの通り、**ログイン不要**です。「トークンさえ分かれば閲覧できる」という設計は、[デスクトップの設計資料](../../1I勉強会webの裏側（設計の話）.md)にもある通り、「トークン自体がDMで本人にだけ届く合言葉の代わり」という考え方に基づいています。
- `resolve_delete_request_token(token)`でトークンを検証し、`_delete_target_summary`で**今この瞬間の最新の内容**を取得します（[00_削除依頼トークンと送信フロー.md](00_削除依頼トークンと送信フロー.md)で解説した通り、依頼時点のスナップショットではありません）。
- `exists`… 対象のファイルが今も実際に存在するかどうかを確認します。`already_gone: not exists`… もし既に削除済み、または既に別の経路で処理済みであれば、フロント側はこれを見て「以下の内容は依頼された時点のものです」という注記を表示できます（[../../1I勉強会web解説/10_DeleteApproval/00_DeleteApproval解説.md](../../1I勉強会web解説/10_DeleteApproval/00_DeleteApproval解説.md)で見た通りです）。

## 3. `/respond_delete_request`：承認/拒否の実行（3829〜3890行）

```python
@app.route("/respond_delete_request", methods=["POST"])
def respond_delete_request():
    """ログイン不要（DMで本人にだけ届いたtokenの所持自体を本人確認の代わりにしている）。"""
    data = request.json or {}
    payload = resolve_delete_request_token(data.get("token", ""))
    if not payload:
        return jsonify({"ok": False, "error": "リンクが無効か、期限切れです。"})
    action = data.get("action")
    if action not in ("approve", "reject"):
        return jsonify({"ok": False, "error": "invalid action"})
```
- `action not in ("approve", "reject")`… ここでも[00_削除依頼トークンと送信フロー.md](00_削除依頼トークンと送信フロー.md)の`category`と同じホワイトリスト方式の検証です。

```python
    category = payload.get("category")
    filename = payload.get("filename")
    owner_nickname = payload.get("owner_nickname") or "作成者"
    requester_id = payload.get("requester_id")
    requester_nickname = payload.get("requester_nickname") or "依頼者"
    guild_id = payload.get("guild_id")
    target_name, _lines = _delete_target_summary(category, filename)

    if action == "reject":
        # ★ 2026/09/04〜：デッキの削除依頼は運用ログに出さない（お知らせは従来通り記録する）
        if category != "deck":
            log_event(
                "notice",
                f"「{target_name}」の削除依頼を{owner_nickname}さんが却下しました。",
                actor=owner_nickname,
            )
        result = jsonify({"ok": True, "action": "reject"})
    else:
        note = f"（{requester_nickname}さんの削除依頼を{owner_nickname}さんが承認）"
        if category == "deck":
            result = _delete_card_deck_file(filename, owner_nickname, approval_note=note)
        else:
            result = _delete_notice_file(filename, owner_nickname, approval_note=note)
```
- `action`によって処理が分かれます。
  - `"reject"`（拒否）… 実際のファイルは何も変更せず、対象がお知らせなら運用ログに「却下しました」という記録を残します（★ 2026/09/04〜：デッキの場合は「デッキの依頼関係は運用ログに表示しなくて良い」というユーザーの指定により、この記録自体を残さなくなった）。
  - `"approve"`（承認）… ここで初めて、後の章で見る実際の削除関数（`_delete_card_deck_file`/`_delete_notice_file`）が呼ばれ、本当にファイルが削除されます。`approval_note`という引数を渡している点に注目してください。これにより、削除関数側の運用ログには「〇〇さんの削除依頼を△△さんが承認」という、**なぜ・誰の判断でこの削除が行われたのか**という文脈まで一緒に記録されます。単に「削除されました」だけでなく、承認フローを経た削除であることが後から追跡できるようになっています（この削除そのものの記録は「依頼」の記録ではなく通常のデッキ／お知らせ削除の記録なので、上記の変更後も対象外にはしていない）。

```python
    if guild_id:
        try:
            items, sha = load_pending_delete_requests(guild_id)
            new_items = [it for it in items if it.get("token") != data.get("token")]
            if len(new_items) != len(items):
                save_pending_delete_requests(guild_id, new_items, sha)
        except Exception:
            pass
```
- **後片付けの処理**です。コメントの通り、`pending_delete_requests_{guild_id}.json`（DMが送れなかった場合のWeb確認待ちの控え）にこの依頼のトークンが残っていれば、承認・拒否どちらの経路で応答されても取り除きます。これにより、一度応答した依頼が、次にサイトを開いたときにもう一度確認モーダルとして出てくることはありません。
- `if len(new_items) != len(items):`… 実際に何か取り除かれた場合（＝該当するトークンが本当に控えに存在していた場合）だけ、わざわざ保存し直します。何も変わっていないのに無駄な書き込みをしない、という配慮です。
- `except Exception: pass`… この後片付け自体は補助的な処理なので、失敗しても承認/拒否という主目的の処理結果には影響させません。

```python
    if guild_id and requester_id:
        try:
            outcome = "承認され、削除されました" if action == "approve" else "却下されました"
            send_discord_dm(
                int(guild_id), requester_id, "🗑 削除依頼の結果",
                f"「{target_name}」の削除依頼は{owner_nickname}さんに{outcome}。",
            )
        except Exception:
            pass

    return result
```
- 最後に、**依頼した本人にも結果を伝えます**。承認された場合も拒否された場合も、依頼者はどうなったかを知りたいはずなので、こちらもDMで通知します。コメントの通り、これもベストエフォート（未連携や送信失敗があっても、承認・拒否という主目的の処理自体は既に成立しているので、それを覆さない）です。
- `return result`… `action`の分岐で作られた`result`（`reject`ならその場で作った成功レスポンス、`approve`なら実際の削除関数が返したレスポンス）を、ここで初めて返します。後片付けと依頼者への通知は、この`return`の**前**、つまりレスポンスを組み立てた**後**に行われている点に注目してください。これにより、仮にこれらの補助的な処理が多少時間がかかっても、削除（または却下）という主目的の処理結果自体は既に確定しており、その後の処理がどうなろうと結果が変わることはありません。

---

これで「削除依頼システム」の章が終わりました。この仕組みは、後の章で見る`/delete_cards`と`/delete_notice`の両方から共通して使われます。次は、ニックネーム変更・学習ログ一覧・ポイント・課題達成のAPIをまとめて解説します。 → [../13_FlaskAPI_その他小さめAPI/00_ニックネーム変更とポイント課題達成API.md](../13_FlaskAPI_その他小さめAPI/00_ニックネーム変更とポイント課題達成API.md)
