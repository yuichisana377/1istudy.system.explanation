# ルーム状態のJSON化・4択の自動生成・過去問アーカイブ（`bot.py` 4877〜5128行）

対象：`bot.py`の`_quiz_room_players_json`・`_quiz_room_snapshot`・`_pick_distractors`・`_build_deck_questions`・`_validate_manual_questions`・`_archive_manual_quiz`・`_archive_room_if_needed`。

## 1. 参加者一覧の整形：`_quiz_room_players_json`（7507〜7522行）

```python
def _quiz_room_players_json(room):
    """スコア順（全体順位）の一覧。同点の場合は、それまでの回答時間の合計
    （total_answer_time）が短い方＝素早く押せていた方を上位にする。
    ロビー・"ranking"（途中経過）画面・最終結果("ended")で使う共通の並び順。"""
    players = sorted(room["players"].values(), key=lambda p: _quiz_rank_key(room, p))
    # 同点時の順位決めに使った回答時間も一緒に返して順位表に出す
    # （平均点モードでは1問あたりの平均、それ以外は合計。単位はクライアントに渡す）
    time_unit = "秒/問" if room.get("score_mode") == "average" else "秒"
    return [
        {
            "id": p["id"],
            "nickname": p["nickname"],
            "color": p["color"],
            "text_color": p["text_color"],
            "score": _quiz_display_score(room, p),
            "score_unit": _quiz_score_unit(room),
            "time_sec": round(_quiz_rank_key(room, p)[1], 1),
            "time_unit": time_unit,
        }
        for p in players
    ]
```
- ★ 追加：`time_sec`・`time_unit`は、同点のときの順位決めに使った「回答にかかった時間」（`_quiz_rank_key`の2要素目。通常は合計秒数、平均点モードでは1問あたりの平均秒数）です。途中経過・最終結果の順位表に、点数の左隣へ小さく表示されます（同点でも順位の理由が見えるように）。
- `sorted(..., key=lambda p: (-p["score"], p.get("total_answer_time", 0.0)))`… スコアの**降順**（マイナスを付けることで、通常は昇順のソートを降順に反転させています）で並べ、順位表として使えるようにします。**★ 追加**：タプルの2要素目`total_answer_time`（それまでに回答にかかった時間の合計。押すのが早いほど小さい値）が、スコアが同点だったときのタイブレークです。Pythonのタプル比較は左から順に評価されるため、まず`-p["score"]`（スコア降順）で決着が付き、それでも同じ場合だけ`total_answer_time`（昇順＝速い方が上）で決まります。この一覧はロビー・`"ranking"`（途中経過）画面・最終結果(`"ended"`)で使う「全体順位」です。
- 以前はこの関数に`include_correct`引数があり、`True`のとき各参加者の「今回の問題に正解したか」まで含めていましたが、**この役割は下記の`_quiz_room_players_json_press_order`という別関数に分離されました**（全体順位の並び順と、reveal中のミニ順位表の並び順が別物になったため。詳しくは次項）。

## 1.5 正解発表中専用の並び順：`_quiz_room_players_json_press_order`（★ 追加）

```python
def _quiz_room_players_json_press_order(room):
    """正解発表(reveal)中のミニ順位表専用：全体スコア順ではなく、
    「今回の問題に回答したのが早い順（正誤にかかわらず）」で並べる
    （Kahoot風の演出。全体順位は"ranking"画面・最終結果の方で見せる）。
    時間切れで未回答の人は末尾にまとめ、その中はスコア順で安定させる。"""
    def sort_key(p):
        answered_at = p.get("cur_answered_at")
        if answered_at is not None:
            return (0, answered_at)
        return (1, -_quiz_display_score(room, p, raw=True))
    players = sorted(room["players"].values(), key=sort_key)
    result = []
    for p in players:
        # ★ 出題中(question)にこれを渡すと、まだ発表前の正誤がバレてしまうため、
        #   この関数自体をrevealのときにしか呼ばない設計にしてある。
        answered = p["cur_answer"] is not None
        result.append({
            "id": p["id"],
            "nickname": p["nickname"],
            "color": p["color"],
            "text_color": p["text_color"],
            "score": _quiz_display_score(room, p),
            "score_unit": _quiz_score_unit(room),
            "answered": answered,
            "correct": bool(p["cur_correct"]) if answered else None,
        })
    return result
```
- 正解発表(reveal)中のミニ順位表だけは、全体スコア順ではなく**「その問題に回答したのが早い順（正誤にかかわらず）」**で見せる、というKahoot風の演出の実装です。`sort_key`が返すタプルの1要素目（`0`か`1`）で「回答済みグループ」と「未回答（時間切れ）グループ」に大きく分け、回答済みグループの中では`cur_answered_at`（回答した時刻）の昇順＝早押し順、未回答グループの中はスコア降順で安定させています。
- `include_correct`（各参加者の今回の問題への正誤）は、この関数を呼ぶこと自体が「reveal中」であることを前提にしているため常に含めます。以前のように呼び出し側が`True`/`False`を選ぶ必要はなくなりました（この関数自体をreveal以外から呼ばない、という規律に一本化されています）。

## 2. ルーム状態のスナップショット：`_quiz_room_snapshot`（7552〜7649行）

これは、フロント側がポーリングで受け取る、その時点でのルームの状態全体です。**`state`の値によって、含まれる情報が意図的に変わります**。

```python
def _quiz_room_snapshot(room, student_id):
    snap = _quiz_room_snapshot_raw(room, student_id)
    if _is_guest_id(student_id):
        # ★ ゲスト（ログインしていない人）には他の参加者の学籍番号を渡さない。自分以外のidは
        #   このルーム内だけで通用する不透明な値に置き換える（フロントは「自分の行か」の比較にしか使わない）。
        def _mask(pid):
            if pid == student_id:
                return pid
            return "p:" + hmac.new(SESSION_SECRET.encode(), (room["code"] + "|" + str(pid)).encode(), hashlib.sha256).hexdigest()[:10]
        for key in ("players",):
            for p in snap.get(key) or []:
                p["id"] = _mask(p.get("id"))
    return snap

def _quiz_room_snapshot_raw(room, student_id):
    snap = {
        "code": room["code"],
        "title": room["title"],
        "state": room["state"],
        "host_nickname": room["host_nickname"],
        "score_mode": room.get("score_mode", "total"),
        "players": _quiz_room_players_json(room),
    }
    if room["state"] == "countdown":
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "countdown_started_at": room["countdown_started_at"],
            "countdown_duration_sec": QUIZ_COUNTDOWN_DURATION_SEC,
        })
    elif room["state"] == "intro":
        # ★ 「第N問」表示中は、まだ問題文・選択肢は渡さない
        #   （question状態になってから渡せば十分で、渡す情報は少ない方がよい）。
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "intro_started_at": room["intro_started_at"],
            "intro_duration_sec": QUIZ_INTRO_DURATION_SEC,
            # ★ 追加：ボーナス問題なら「第N問」の時点から予告する（盛り上げるため）
            "is_bonus": room["current_q"] in room.get("bonus_indices", set()),
        })
```
- `"intro"`（「第N問」を見せている段階）では、**まだ問題文や選択肢自体を渡していません**。コメントにある通り「question状態になってから渡せば十分で、渡す情報は少ない方がよい」という、最小限の情報しか渡さない方針です。

```python
    elif room["state"] == "ranking":
        # ★ 追加：正解発表の後、次の問題に進む前に見せる「途中経過」。
        #   playersは常にスコア順（tie-breakは回答時間の合計）で入っているので、
        #   そのままフロントで一覧表示・順位変動アニメーションに使える。
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "ranking_started_at": room["ranking_started_at"],
            "ranking_duration_sec": QUIZ_RANKING_DURATION_SEC,
        })
```
- `"ranking"`（★ 追加）では、専用のフィールドは`current_q`・`total_questions`・開始時刻・表示秒数だけです。肝心の順位一覧は、`snap`の先頭で常にセットされている`"players": _quiz_room_players_json(room)`（スコア順・タイブレーク込み）をそのまま使い回せるため、ここで改めて上書きする必要がありません。

```python
    elif room["state"] in ("question", "reveal"):
        q = room["questions"][room["current_q"]]
        revealed = room["state"] == "reveal"
        # ★ 変更：問題の形（4択/並び替え・画像枚数）は _quiz_question_payload にまとめた
        #   （正解は revealed のときだけ載る。下の「正解番号は…」のコメントの方針はそのまま）。
        question_payload = _quiz_question_payload(q, revealed)
        # ★ 追加：ボーナス問題（正解時の得点2倍）かどうかをクライアントにも知らせる
        #   （画面に「ボーナス問題！2倍！」等を表示するため。得点計算自体はquiz_answer側）。
        if room["current_q"] in room.get("bonus_indices", set()):
            question_payload["is_bonus"] = True
        # ★ 正解番号は、発表(reveal)されるまでは誰にも渡さない（レスポンスを
        #   devtools等で覗かれてカンニングされるのを防ぐ）。ホストも今は
        #   1プレイヤーとして参加するため、ホストだけ特別扱いはしない。
        # ★ 途中参加でまだ「見学中」（active_from_q未到達）の人は、回答受付数・
        #   合計人数のどちらにもカウントしない（自分では回答できないため）。
        active_players = [p for p in room["players"].values() if p.get("active_from_q", 0) <= room["current_q"]]
        snap.update({
            "current_q": room["current_q"],
            "total_questions": len(room["questions"]),
            "question": question_payload,
            "question_started_at": room["question_started_at"],
            "time_limit_sec": room["time_limit_sec"],
            "answered_count": sum(1 for p in active_players if p["cur_answer"] is not None),
            "total_players": len(active_players),
        })
        if revealed:
            snap["first_correct_nickname"] = room.get("first_correct_nickname")
            snap["reveal_started_at"] = room.get("reveal_started_at")
            snap["reveal_duration_sec"] = room.get("reveal_duration_sec", QUIZ_REVEAL_DURATION_SEC)
            # ★ 発表中だけ、押した順（正誤にかかわらず）のミニ順位表に players を上書きする
            snap["players"] = _quiz_room_players_json_press_order(room)
        player = room["players"].get(student_id)
        if player is not None:
            if player.get("active_from_q", 0) > room["current_q"]:
                # ★ 追加：途中参加で今の問題にはまだ混ざれない「見学中」の状態。
                #   フロントはこれを見て、回答UIの代わりに待機画面を出す。
                snap["spectating"] = True
            elif player["cur_answer"] is not None:
                snap["your_answer"] = player["cur_answer"]
                if revealed:
                    snap["your_correct"] = bool(player["cur_correct"])
                    # ★ 追加：ボーナス問題（得点2倍）だと固定の「+10点」「+12点」表示が
                    #   実際の加点と食い違うため、quiz_answer()で確定した実際の加点を
                    #   そのままフロントに渡す（表示側で倍率を再計算させない）。
                    if player.get("cur_points"):
                        snap["your_points"] = player["cur_points"]
    return snap
```
- **これがこの関数で最も重要な安全対策です**：`question_payload`には常に`question`（問題文）と`choices`（選択肢）が含まれますが、`correct_index`（正解の番号）は`revealed`（＝`state == "reveal"`）のときだけ含まれます。もしこのガードが無く、出題中から正解番号がレスポンスに含まれていたら、ブラウザの開発者ツールでネットワーク通信の中身を見るだけで、誰でも正解が分かってしまいます（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)の設計方針にあった「正解番号はホストにも発表されるまでは一切渡さない」の実装がここです）。
- **★ 追加：`active_players`と`spectating`**：途中参加を許可したルームでは、`question`/`reveal`中に参加した人（`active_from_q`が今の`current_q`より大きい人）を、`answered_count`/`total_players`の集計対象から外します。さらに、その本人自身に対するレスポンスにだけ`spectating: true`を含め、フロント側がこれを見て「問題・選択肢の代わりに待機画面を出す」判断ができるようにしています（他の参加者から見た`players`一覧には、見学中かどうかを示すフラグは含まれません＝スコア0のまま普通に参加者として並びます）。
- **★ 変更：`players`の上書き元**：以前は`revealed`のとき`_quiz_room_players_json(room, include_correct=True)`（全体スコア順のまま、正誤情報だけ足す）でしたが、今は`_quiz_room_players_json_press_order(room)`（押した順に並べ替えたうえで正誤情報を含める）に変わりました。全体のスコア順位を見せる場面は`"ranking"`画面・最終結果に分離されたため、reveal中のミニ順位表は「このKahoot的な演出専用の並び順」に特化しています。
- `revealed`のときだけ`_quiz_room_players_json(room, include_correct=True)`で`players`を**上書き**しています。これにより、正解発表の瞬間だけ、他の参加者の正誤（◯×）が一覧に反映されます。
- `player["cur_answer"]`（**自分自身**の回答）は、`revealed`かどうかに関わらず渡されます。自分が何を選んだかは、他の参加者に隠す必要が無い（自分自身の情報だから）ためです。ただし、自分の回答が「正解だったかどうか」（`your_correct`）は、他の参加者の正誤と同様に`revealed`のときだけ渡されます。

## 3. 見分けにくい誤答の生成：`_pick_distractors`（7651〜7704行）

```python
def _pick_distractors(correct: str, pool: list, k: int, ranked: bool = False) -> list:
    """
    正解(correct)に対する誤答をpoolからk個選ぶ。
    ★ 完全ランダムに選ぶと、他のカードの答えと文字数も内容もかけ離れた
      誤答ばかりになりがちで、見た目だけで消去法に正解できてしまっていた。
      正解と文字列として近い（綴り・字面が似ている）ものを優先候補にし、
      その中からランダムに選ぶことで、きちんと覚えていないと迷うような
      4択にする。
      ★ さらに、綴りの類似度（SequenceMatcher）だけだと「文字数が全然違う
      せいで一目で誤答と分かる」選択肢が紛れ込みやすかったため、文字数の
      近さもスコアに加味する。また、候補の絞り込みを従来（上位9件から
      ランダムに3件）よりタイトにする（上位6件から3件）ことで、
      「似ているが選ばれなかった」紛らわしい候補が混ざりにくくし、
      パッと見で消去法が効きにくい4択にする。
    """
    # ★ 改善（2026/11/14）：採点（類似度の計算）は _distractor_shortlist の1か所にまとめた。
    #   以前は同じ採点関数がここにも重複して書かれており、CardMakerのキャッシュ生成では
    #   「即席4択用」と「AIに渡す候補用」で同じ採点を2回やり直していた。
    #   ranked=True は「pool は既に採点順に並んでいる」という意味（採点を省く）。
    #   文字種の近さを加味したため、以前（綴り0.7・長さ0.3）とは選ばれ方が変わっている。

    scored = pool if ranked else _distractor_shortlist(correct, pool, len(pool))
    pool_size = max(k, min(len(scored), k * 2))
    return random.sample(scored[:pool_size], k)


def _distractor_shortlist(correct: str, pool: list, k: int) -> list:
    """_pick_distractorsと同じ類似度スコアで、上位k件を（ランダム抽選せず）そのまま返す。
    AIに渡す候補を絞り込むための下ごしらえとして使う（プロンプトを軽くする目的）。"""
    def _classes(s):
        # 文字種（漢字／ひらがな／カタカナ／その他）の構成比。漢字だけの語に、ひらがな主体の語が
        # 並ぶと一目で消去できてしまうため、文字種の近さも加味する。
        c = {"k": 0, "h": 0, "t": 0, "o": 0}
        for ch in s:
            o = ord(ch)
            if 0x4e00 <= o <= 0x9fff:
                c["k"] += 1
            elif 0x3040 <= o <= 0x309f:
                c["h"] += 1
            elif 0x30a0 <= o <= 0x30ff:
                c["t"] += 1
            else:
                c["o"] += 1
        n = max(len(s), 1)
        return {k: v / n for k, v in c.items()}
    correct_cls = _classes(correct)
    def _score(a):
        seq_ratio = difflib.SequenceMatcher(None, correct, a).ratio()
        longer = max(len(correct), len(a), 1)
        length_ratio = 1 - abs(len(correct) - len(a)) / longer
        a_cls = _classes(a)
        class_ratio = 1 - sum(abs(correct_cls[k] - a_cls[k]) for k in correct_cls) / 2
        return seq_ratio * 0.5 + length_ratio * 0.25 + class_ratio * 0.25
    return sorted(pool, key=_score, reverse=True)[:k]
```
- **2026/11/14〜：採点（`_score`）は`_distractor_shortlist`の1か所だけにあります。** 以前は全く同じ`_score`が`_pick_distractors`の中にも重複して書かれていました。今の`_pick_distractors`は、`_distractor_shortlist`に「全件を採点順に並べて」と頼み（`k`に`len(pool)`を渡す）、その上位から選ぶだけです。並び順も選ばれ方も以前と同じです。`ranked=True`は「`pool`は既に採点順に並んでいる」という意味で、採点を省きます（CardMakerのキャッシュ生成が使います。[../14_FlaskAPI_CardMaker/04_4択事前生成キャッシュ.md](../14_FlaskAPI_CardMaker/04_4択事前生成キャッシュ.md)参照）。
- これはクイズの「面白さ・妥当性」に関わる、興味深いアルゴリズムです。単に不正解の選択肢をランダムに選ぶだけだと、コメントの通り「正解だけ極端に短い/長い」のような、内容を知らなくても見た目だけで消去法により正解できてしまう、質の低い4択になりがちでした。
- `difflib.SequenceMatcher(None, correct, a).ratio()`… [../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md](../02_データ保存基盤/00_ファイル読み書きとSHA排他制御.md)などで登場した`difflib`ライブラリの、今度は**類似度を測る**機能です。2つの文字列がどれだけ似ているか（0〜1の比率）を計算します。
- `_score`関数は、**文字列としての似ている度合い（70%の重み）**と**文字数の近さ（30%の重み）**を組み合わせたスコアです。コメントの追加説明にある通り、綴りの類似度だけだと「文字数が全然違うせいで一目で誤答と分かる」選択肢が紛れ込みやすかったため、文字数の近さも加味するよう改良されています。
- `sorted(pool, key=_score, reverse=True)`… 候補全体をスコアの高い順（正解に似ている順）に並べます。
- `pool_size = max(k, min(len(scored), k * 2))`… 上位何件から実際に選ぶかを決めます。`k=3`（3つの誤答が必要）なら、`pool_size`は最大6件です。コメントにある通り、以前は「上位9件からランダムに3件」だったものを「上位6件から3件」とタイトに絞ることで、「似ているが選ばれなかった紛らわしい候補が混ざりにくく」し、消去法が効きにくい4択にする改善が行われています。
- `random.sample(scored[:pool_size], k)`… 上位の候補群から、最終的にランダムに`k`個を選びます。**完全にスコア順で機械的に選ぶのではなく、上位群からランダムに選ぶ**ことで、同じデッキから何度クイズを作っても、毎回少しずつ違う顔ぶれの誤答になるようにしています。

## 4. デッキからの自動問題生成：`_build_deck_questions`（8124〜8212行）

```python
def _build_deck_questions(guild_id, deck_filenames, num_questions):
    """単語カードのデッキ（表面=question／裏面=answer）から、答えをシャッフルした
    4択問題を自動生成する。
    ★ deck_filenames は単一のファイル名（文字列）でも、複数のファイル名のリストでも
      受け取れる。複数指定した場合（Web側でフォルダごと選んだ場合を含む）は、
      それぞれのデッキのカードをまとめて1つの問題プールとして扱い、
      distractor（不正解の選択肢）も選んだデッキ全体から選ぶ。
    (questions, error_code) を返す（成功時 error_code は None）。"""
    if isinstance(deck_filenames, str):
        deck_filenames = [deck_filenames]
    if not isinstance(deck_filenames, list) or not deck_filenames:
        return None, "deck_not_found"
    # ★ 重複を除きつつ順序は維持する
    seen_filenames = set()
    deck_filenames = [f for f in deck_filenames if not (f in seen_filenames or seen_filenames.add(f))]
    if len(deck_filenames) > QUIZ_MAX_SOURCE_DECKS:
        return None, "too_many_decks"
```
- `deck_filenames`は文字列でもリストでも受け付ける、柔軟な引数設計です。`isinstance(deck_filenames, str): deck_filenames = [deck_filenames]`で、単一の文字列が来た場合は1要素のリストに正規化してから、以降は常にリストとして扱います。
- `[f for f in deck_filenames if not (f in seen_filenames or seen_filenames.add(f))]`… これは、Pythonでよく使われる「重複除去しつつ順序を保つ」ためのトリックです。`seen_filenames.add(f)`は常に`None`（Pythonの`set.add`は戻り値を返さない）を返すため、`f in seen_filenames`が`False`（まだ見ていない）のときだけ、`or`の右側で`.add(f)`が実行されて`seen_filenames`に追加され、リスト内包表記の条件全体としては`not (False or None)` = `not None` = `True`となり、その要素が残ります。既に見た要素は`f in seen_filenames`が`True`になるため、`.add`は評価されず（`or`の短絡評価）、条件全体が`False`になって除外されます。

```python
    cards = []
    order_questions = []  # ★ 追加：並び替えカードから作った問題（4択の誤答プールとは別に持つ）
    for deck_filename in deck_filenames:
        if not isinstance(deck_filename, str) or "/" in deck_filename or "\\" in deck_filename or ".." in deck_filename:
            return None, "deck_not_found"
        data, _ = get_card_file_for_guild(deck_filename, guild_id)
        if data is None:
            return None, "deck_not_found"
        for c in data.get("cards", []):
            if not (isinstance(c, dict) and str(c.get("question") or "").strip()):
                continue
            if card_type(c) == "order":
                # ★ 変更：並び替えカードも出題対象にする（kind:"order"の問題になる）。
                #   ただし4択の誤答プール（cards）には入れない（「a → b → c」は誤答として不自然）。
                oq = _quiz_order_question_from_card(c, deck_filename)
                if oq is not None:
                    order_questions.append(oq)
                continue
            if not str(c.get("answer") or "").strip():
                continue
            # ★ 追加：どのデッキ・どのカード由来かを覚えておく（クライアントには一切送らない、
            #   quiz_answer()が不正解時に元デッキへ「わからない」を裏で付けるためだけに使う）。
            #   idの無い古いカードは対象外にする（CardMaker側のcardKey()のハッシュ
            #   フォールバックをPython側で厳密に再現するのは割に合わないため）。
            c = dict(c)
            c["_src_deck_filename"] = deck_filename
            c["_src_card_id"] = c.get("id")
            cards.append(c)
    unique_answers = {c["answer"].strip() for c in cards}
    # ★ 4択（正解1つ＋不正解3つ）を作るには、答えの異なり（ユニークな値）が
    #   最低4つ必要。html側の注意書き（「答えの種類が4つ以上あるデッキが必要」）と対応。
    # ★ 変更：並び替えカードは誤答プールが要らないので、4択が作れないデッキでも
    #   並び替えカードがあればそれだけで出題する。どちらも無ければ deck_too_small。
    if len(cards) < 4 or len(unique_answers) < 4:
        cards = []
    if not cards and not order_questions:
        return None, "deck_too_small"
```
- ここでも[../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md](../12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)と同じパストラバーサル対策があります。
- 指定された全デッキから、問題文・解答の両方が空でないカードだけを`cards`に集約します。
- `unique_answers = {c["answer"].strip() for c in cards}`… 集合（set）内包表記で、重複を除いた解答の種類数を数えます。コメントの通り「4択（正解1つ＋不正解3つ）を作るには、答えの異なり（ユニークな値）が最低4つ必要」というルールがここでチェックされます。

```python
    n = min(num_questions, len(candidates)) if num_questions else len(candidates)
    n = max(1, min(n, QUIZ_MAX_QUESTIONS, len(candidates)))

    picked = random.sample(candidates, n)
    questions = []
    for kind, card in picked:
        if kind == "order":
            questions.append(card)
            continue
        correct = card["answer"].strip()
        pool = list(dict.fromkeys(  # 同じ答え文言の重複を除去しつつ順序を保つ
            c["answer"].strip() for c in cards if c is not card and c["answer"].strip() != correct
        ))
        if len(pool) < 3:
            return None, "deck_too_small"
        choices = _pick_distractors(correct, pool, 3) + [correct]
        random.shuffle(choices)
        questions.append({
            "question": card["question"].strip(),
            "choices": choices,
            "correct_index": choices.index(correct),
            # ★ 追加：quiz_answer()が不正解時に元デッキへ裏で「わからない」を付けるための
            #   参照（クライアントには送らない。question_payload等には含めていない）。
            "source_deck_filename": card.get("_src_deck_filename"),
            "source_card_id": card.get("_src_card_id"),
            **_quiz_card_media(card),
        })
    return questions, None
```
- `n = max(1, min(n, QUIZ_MAX_QUESTIONS, len(cards)))`… 出題数を「1問以上」「上限30問（`QUIZ_MAX_QUESTIONS`）以内」「実際に使えるカード数以内」の3つの制約すべてに収まるよう調整します。
- `random.sample(cards, n)`で、実際に出題するカードをランダムに選びます。

### 4.1 追記（2026/08/27）：不正解を元デッキへ裏で「わからない」として記録するための参照

ユーザーから「みんなでクイズで間違えたものもひっそりわからないつけておいて（表面上には表示せず内部で）」との要望を受け、`cards`収集時（181行目のブロック）に各カードへ`_src_deck_filename`（そのカードが属するデッキのファイル名）・`_src_card_id`（`card.get("id")`。idの無い古いカードは`None`のまま＝対象外）を控えるようにし、`questions.append(...)`（216行目）にも`"source_deck_filename"`・`"source_card_id"`として引き継ぐようにした。

- これらのキーは**Webクライアントには一切送らない**（`question_payload`＝`{"question": ..., "choices": ...}`には含めていない）。目的は[03_ルーム参加と進行API.md](03_ルーム参加と進行API.md)の`/quiz_answer`が不正解時に参照するためだけで、あくまでサーバー内部の記録用。
- ローカルAI（Ollama）による誤答強化（後述7節、`_ai_pick_quiz_distractors`）は`new_questions[i] = {...}`で問題dictを作り直すため、ここでも同じ2キーを引き継ぐよう修正してある（引き継がないと、AI強化が効いた問題だけ「わからない」の裏付けができなくなる）。
- ホストが手入力した「オリジナル4択」（`_validate_manual_questions`、`source=="manual"`）には元カードという概念自体が無いため、この2キーは付けない（`quiz_answer`側もNoneチェックで自動的に対象外になる）。
- 各カードについて、`pool`（そのカード以外の、正解と異なる解答の集合。`dict.fromkeys(...)`は順序を保ったまま重複除去する、リスト内包の`set`版に近いテクニック）を作り、[3. 見分けにくい誤答の生成：`_pick_distractors`（4953〜4976行）](#3-見分けにくい誤答の生成pick_distractors49534976行)の`_pick_distractors`で3つの誤答を選びます。
- `choices = _pick_distractors(...) + [correct]`で4つの選択肢を作り、`random.shuffle(choices)`でシャッフルしてから、`correct_index`（正解が何番目に来たか）を記録します。この時点ではまだ、誰が見ても正解が分からないよう、正解の位置自体もランダムです。

## 5. 手入力問題の検証：`_validate_manual_questions`（8214〜8238行）

```python
def _validate_manual_questions(raw_questions):
    """ホストが手入力した問題データを検証する。
    成功時: ((questions, check_fields), None) / 失敗時: (None, error_code)"""
    if not isinstance(raw_questions, list) or not raw_questions:
        return None, "invalid_questions"
    questions = []
    check_fields = {}
    for i, q in enumerate(raw_questions):
        if not isinstance(q, dict):
            return None, "invalid_questions"
        question_text = str(q.get("question") or "").strip()
        choices = q.get("choices")
        correct_index = q.get("correct_index")
        if not question_text or not isinstance(choices, list) or len(choices) != 4:
            return None, "invalid_questions"
        choices = [str(c or "").strip() for c in choices]
        if any(not c for c in choices):
            return None, "invalid_questions"
        if not isinstance(correct_index, int) or isinstance(correct_index, bool) or not (0 <= correct_index < 4):
            return None, "invalid_questions"
        check_fields[f"問題{i+1}の問題文"] = question_text
        for j, c in enumerate(choices):
            check_fields[f"問題{i+1}の選択肢{j+1}"] = c
        questions.append({"question": question_text, "choices": choices, "correct_index": correct_index})
    return (questions, check_fields), None
```
- ホストが「自分で問題を作る」オリジナル4択モードで入力したデータを検証します。デッキから自動生成する場合と違い、ここでは**利用者が自由に入力したテキスト**を扱うため、非常に丁寧なバリデーションが行われています。
- `len(choices) != 4`… 選択肢は必ずちょうど4つでなければなりません。
- `if not isinstance(correct_index, int) or isinstance(correct_index, bool) or not (0 <= correct_index < 4):`… [../09_FlaskAPI_予定管理/01_学習ログ記録APIのなりすまし対策と連打対策.md](../09_FlaskAPI_予定管理/01_学習ログ記録APIのなりすまし対策と連打対策.md)の`add_study_log`で見たのと同じ、`bool`が`int`のサブクラスであることへの対策が、ここにも登場しています。加えて、正解番号が0〜3の範囲に収まっているかも確認します。
- `check_fields`… 検証したテキストを、後で呼び出し元が[../02_データ保存基盤/02_設定ファイルと不正文字チェック.md](../02_データ保存基盤/02_設定ファイルと不正文字チェック.md)の`reject_if_bug_chars`にまとめて渡すための辞書です。この関数自身は不正文字チェックまでは行わず、その責務は呼び出し元に委ねられています。

## 6. クイズ結果のCardMakerアーカイブ（8240〜8329行）

```python
def _archive_manual_quiz(guild_id, title, questions, student_id, nickname, players=None):
    """
    ★ ホストが「自分で問題を作る」（オリジナル4択）で作成したクイズを、
      CardMakerの「クイズ過去問」フォルダにデッキとして自動保存する。
      いつでも一人用選択式モードで遊べる「過去問」として残すため。
      呼び出し元は _archive_room_if_needed()（クイズが終了した時点で呼ばれる）。
    ・questions は _validate_manual_questions の戻り値そのもの
      （[{"question", "choices"(4件), "correct_index"}, ...]）。
    ・choice_mode/choices/correct_indices は、CardMaker側の選択式デッキ共通
      フォーマット。単一/複数正解はデッキ単位ではなく問題ごとに
      correct_indices の個数で決まる（CardMaker側の仕様）。Quiz.js自体は
      4択・単一正解の固定フォーマットのままで、ここでの変換にしか影響しない。
    ・answer（正解の選択肢文言）も入れておく。これにより単語検索・一覧表示・
      作成済みリストなど、「answerは文字列である」という前提の既存コードを
      一切変更せずに動かせる（choices/correct_indices は選択式UIだけが見る）。
    ・アーカイブに失敗しても、クイズ自体の進行は失敗させない（ベストエフォート）。
    ・players（room["players"]、任意）を渡すと、各プレイヤーがゲーム中に
      不正解だった問題（player["wrong_indices"]、quiz_answer参照）を、
      今まさに作成したこの過去問デッキへ裏で「わからない」として記録する。
      「デッキから自動作成」と違い元デッキが存在しないため、ゲーム終了で
      この過去問デッキ自体が生まれた今がその唯一の機会になる。
    """
    try:
        _ensure_quiz_archive_folder(guild_id)
        cards = [{
            "id": secrets.token_hex(6),
            "question": q["question"],
            "answer": q["choices"][q["correct_index"]],
            "choices": q["choices"],
            "correct_indices": [q["correct_index"]],
            "explanation": "",
            "imgs_q": [], "imgs_a": [], "imgs_e": [],
            "type": "choice",  # ★ 追加：カード種別を明示（並び替えカード対応で導入したtype）
            # ★ 追加：カード単位の作成者記録（ホストが全問を作ったのでホスト本人）。「作成した
            #   問題数が多い人」を基準にする各種機能（CSVダウンロード・削除依頼の通知先等）で使う。
            "created_by_id": student_id,
            "created_by_nickname": nickname,
        } for q in questions]
        filename = generate_card_filename(guild_id)
        card_payload = {
            "name": title,
            "cards": cards,
            "subject": None,
            "folder_id": QUIZ_ARCHIVE_FOLDER_ID,
            "published_by": {"id": student_id, "nickname": nickname},
            "incomplete": False,
            "choice_mode": True,  # ★ 選択式デッキであることのマーカー（単一/複数は問題ごとにcorrect_indicesの個数で決まる）
            "quiz_archive": True,  # ★ 追加（2026/08/21）：クイズ過去問デッキであることのマーカー。
            # フォルダ位置に依存させると、フォルダの外へ移動できるようにした際に判定できなくなる
            # （save_cardsが問題編集を禁止する対象の特定にも使う）ため、デッキ自身に持たせる。
            "guild_id": guild_id,  # ★ 複数サーバー対応：このデッキがどのサーバー向けかを記録する
        }
        put_card_file(filename, card_payload)
        index_change = upsert_cards_index_entry(guild_id, filename, card_payload)
        change = deck_file_diff(guild_id, f"{CARDS_DIR}/{filename}", None, card_payload)
        detail = [c for c in (change, index_change) if c]
        log_event(
            guild_id,
            "card",
            f"みんなでクイズの結果を「{title}」として「クイズ過去問」に保存しました（{len(cards)}問）。",
            actor=nickname,
            detail=detail if detail else None,
        )
        # ★ 各プレイヤーが不正解だった問題を、今作った過去問デッキへ裏で「わからない」として記録する
        #   （プレイヤーごとに1回の保存にまとめる）。デッキ作成が失敗した場合はここへ来ないので正しい。
        for pid, p in (players or {}).items():
            if _is_guest_id(pid):
                continue  # ★ ゲストには学習データが無い（study_data_*を作らない）
            ids = [cards[i]["id"] for i in p.get("wrong_indices") or [] if 0 <= i < len(cards)]
            if ids:
                _silently_mark_unsure(guild_id, pid, filename, ids)
    except Exception as e:
        print(f"[WARN] クイズ過去問の保存に失敗しました（クイズの進行自体は続行）: {e}")
```
- **CardMakerとクイズという、2つの独立した機能が繋がる面白い箇所です**。ホストが手作りしたオリジナルのクイズ問題を、クイズが終わった後に**自動的にCardMakerのデッキとして保存**し、いつでも一人用の選択式モードで復習できるようにしています。
- `answer`フィールドにも正解の選択肢文言を入れている理由がコメントに書かれています：「これにより単語検索・一覧表示・作成済みリストなど、『answerは文字列である』という前提の既存コードを一切変更せずに動かせる」。CardMaker側の多くの機能は、通常のフラッシュカードデッキ（`answer`が単純な文字列）を前提に作られています。選択式デッキという新しい概念を追加するにあたり、既存のコードを大きく書き換えるのではなく、**選択式デッキも`answer`フィールドだけは互換性のために律儀に埋めておく**ことで、既存の機能をそのまま動かし続けられるようにする、という実務的な工夫です。
- **`quiz_archive: True`（2026/08/21追加）**… このデッキが「クイズ過去問」由来であることを示す専用フラグです。[../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)・[../18_FlaskAPI_カードフォルダと並び順/00_フォルダのツリー構造と操作API.md](../18_FlaskAPI_カードフォルダと並び順/00_フォルダのツリー構造と操作API.md)で詳しく解説していますが、以前はこの性質を`folder_id`（フォルダの位置）だけで判定していました。ユーザーの要望で「デッキを他のフォルダへ移動できるようにしたいが、問題の編集はできないままにしたい」という仕様に変わったのに伴い、位置に依存しないこの専用フィールドへ切り替わっています。

```python
def _archive_room_if_needed(room):
    """
    ★ QUIZ_ROOMS_LOCK を保持している状態で呼び出すこと。オリジナル4択（source=="manual"）の
      クイズが終了した瞬間に1回だけCardMakerへ保存する（room["archived"] で二重登録を防ぐ）。
      ファイル書き込みは別スレッドで行い、全ルーム共通のこのロックを握ったままにしない。
    """
    if room.get("source") != "manual" or room.get("archived"):
        return
    room["archived"] = True
    players = {pid: {"wrong_indices": list(p.get("wrong_indices") or [])} for pid, p in room["players"].items()}  # 写し
    args = (room["guild_id"], room["title"], room["questions"], room["host_id"], room["host_nickname"], players)
    def _run():
        with _QUIZ_ARCHIVE_LOCK:
            _archive_manual_quiz(*args)
    Thread(target=_run, daemon=True).start()
```
- **2026/11/14〜：保存は別スレッドで行います。** この関数は`QUIZ_ROOMS_LOCK`（全ルーム共通のロック）を握った状態で呼ばれます。以前はそのまま`_archive_manual_quiz`を呼んでいたため、デッキの書き込み・索引の更新・運用ログ・全プレイヤー分の「わからない」の書き込みが終わるまで、**他のルームも含めた全てのクイズの進行とAPIが待たされていました**（[03_ルーム参加と進行API.md](03_ルーム参加と進行API.md)の`/quiz_answer`が「ロック内ではファイルI/Oをしない」と決めているのと食い違っていました）。今は、ロックの中では「保存済みの印を付ける」「必要なデータを写し取る」だけを行い、実際の保存は`Thread`に任せます。
  - `players`は`room["players"]`そのものではなく、`wrong_indices`だけを写した辞書です。別スレッドが読んでいる最中に`/quiz_leave`で参加者が抜けても影響を受けないようにするためです。
  - `_QUIZ_ARCHIVE_LOCK`は保存専用の別のロックです。2つのルームがほぼ同時に終わったとき、過去問フォルダの作成と索引の更新が同時に走ると片方が消えるおそれがあるので、保存どうしは1件ずつ順番に行います。`QUIZ_ROOMS_LOCK`とは別物なので、クイズの進行は止めません。
  - クイズ終了の直後に一瞬だけ「過去問デッキがまだ無い」時間ができますが、保存が終わると索引の更新に伴う通知（SSE）が飛ぶので、CardMaker側の一覧は自動で追いつきます。
- `room.get("source") != "manual"`… デッキから自動生成されたクイズ（`source == "deck"`）は、元々CardMaker上に存在するデッキそのものなので、改めてアーカイブする必要がありません。オリジナル問題（`source == "manual"`）の場合だけアーカイブします。

### 6.1 追記（2026/08/27）：オリジナル4択も、不正解を過去問デッキへ裏で「わからない」として記録

[03_ルーム参加と進行API.md](03_ルーム参加と進行API.md)の`/quiz_answer`で追加した「デッキから自動作成の不正解を元デッキへ裏でわからない付ける」機能について、ユーザーから「デッキから自動生成でなくても過去問として残るんだからわからないつけて」と指摘があった。オリジナル4択（`source=="manual"`）には元デッキが無いため、`/quiz_answer`の時点ではまだ対象デッキが存在しない——**この過去問デッキ自体が生まれる瞬間（クイズ終了時のアーカイブ）が、唯一の記録タイミング**になる。

- 各プレイヤーの`player`辞書に`"wrong_indices": []`を追加（新規参加者・途中参加者の両方の初期化箇所）。`/quiz_answer`が不正解と判定するたびに`player["wrong_indices"].append(room["current_q"])`する。
- `_archive_manual_quiz`に`players`（`room["players"]`）という新しい引数（省略可、デフォルト`None`）を追加。`cards`（`questions`と同じ順序・同じインデックスで作られる、6節参照）を作り終えた直後に、`players`の各プレイヤーの`wrong_indices`を見て、該当インデックスの`cards[i]["id"]`を`_silently_mark_unsure(guild_id, pid, filename, cards[i]["id"])`（2026/11/14〜は、プレイヤーごとにカードidをリストにまとめて`_silently_mark_unsure(guild_id, pid, filename, ids)`と1回だけ呼びます。1問ごとに学習データを読み書きしていたのを、1人につき1回の保存にしました）（[03_ルーム参加と進行API.md](03_ルーム参加と進行API.md)参照）へ渡す。
  - **`questions[i]`と`cards[i]`が同じインデックスで対応している**ことがこの実装の前提（`cards = [{...} for q in questions]`というリスト内包表記で、順序を変えずに1:1変換しているため成立する）。「問題ごとに正解番号（`current_q`）を覚えておき、あとで同じ番号の位置にあるカードを引く」という、`source_deck_filename`/`source_card_id`（デッキから自動作成の場合、01節参照）とは全く違うアプローチだが、目的は同じ。
  - デッキ作成（`put_card_file`）が失敗した場合はこのマーキング処理まで到達しない（`try`の外に出ない＝過去問デッキ自体が存在しないので、マークのしようがなく、それで正しい）。
- ホストも`room["players"]`に含まれる1人のプレイヤーなので、ホスト自身が不正解だった問題も同じ仕組みで自動的にマークされる。
- `room.get("archived")`（既にアーカイブ済みフラグ）… コメントの通り、この関数は「自然に終了した場合」と「ホストが手動で終了させた場合」の**両方の経路から呼ばれる可能性がある**ため、`room["archived"] = True`を立ててから実際の保存処理を呼ぶことで、同じクイズ結果が誤って2つのデッキとして二重に保存されてしまうのを防いでいます。

## 7. ローカルAIによる誤答の強化（2026/08/26追加、`bot.py` 6058〜6063行・6336〜6510行）

「デッキから自動作成」で作られる4択は、上の`_pick_distractors`が**綴りの類似度だけ**で誤答を選んでいるため、綴りは似ていても意味的には全く見当違いな誤答が混ざることがありました。これをサーバー上に立てたローカルAI（Ollama、`qwen2.5-coder:7b`）に判断させ、「意味的にも紛らわしい」誤答へ差し替える仕組みを追加しています（[[local-coder-ai]]と同じOllamaコンテナを流用）。

### 7-1. 有効/無効の切り替えと設定

```python
OLLAMA_HOST = (os.environ.get("OLLAMA_HOST") or "").strip().rstrip("/")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL") or "qwen2.5-coder:7b"
QUIZ_AI_TIMEOUT_SEC = 45        # Ollamaへの問い合わせ1回あたりのタイムアウト（バックグラウンド実行なので余裕を持たせる）
QUIZ_AI_SHORTLIST_SIZE = 12    # AIに渡す誤答候補の数（プロンプトを軽くするため、類似度上位のみに絞る）
```
- `OLLAMA_HOST`が未設定（開発機など）なら、この機能は**一切動かず**、従来通り`_pick_distractors`だけで誤答が決まります。本番サーバーでは`.env`に`OLLAMA_HOST=http://ollama:11434`を設定し、あらかじめ`docker network connect pythonbot1istudy_default ollama`でBotコンテナ（ネットワーク`pythonbot1istudy_default`）とOllamaコンテナ（元々は`bridge`ネットワークのみ、ホストの`127.0.0.1:11434`にしか公開していない）を同じDockerネットワークに繋いでおく必要があります（ホスト側のポート公開範囲は変更していないので、外部への公開状況は変わりません）。

### 7-2. なぜ`quiz_create`をブロックしないのか

Web側の`quiz_create`はクライアントで12秒のタイムアウトが設定されています（`Quiz.js`の`apiPost('quiz_create', body, 12000)`）。一方ローカルAIはCPU動作でモデルも大きめ（7B）のため、応答に数秒〜数十秒かかることがあります（[[local-coder-ai]]のREADMEに実測あり）。そこで、AIへの問い合わせは`quiz_create`の応答後に**バックグラウンドスレッド**で行い、ルームが「ロビー」状態（ホストが「開始」を押す前、参加者を待っている間）の間だけ裏で差し替えます。

```python
if source == "deck" and OLLAMA_HOST:
    # ★ ローカルAIによる誤答強化はロビー中に裏で行う（quiz_create自体はブロックしない）。
    #   OLLAMA_HOST未設定（開発機など）なら従来通りスレッドすら起動しない。
    Thread(target=_ai_enhance_quiz_room_choices, args=(code, guild_id, deck_filenames), daemon=True).start()
```
- `source == "deck"`のときだけ（ホストが手入力した`manual`クイズは対象外＝人が作った問題にAIが手を入れる必要は無い）。

### 7-3. `_ai_enhance_quiz_room_choices`：安全な差し替えのタイミング制御

```python
def _ai_enhance_quiz_room_choices(code, guild_id, deck_filenames):
    """
    ★ 「デッキから自動作成」のクイズについて、ロビー中（ホストが「開始」する前）に
      バックグラウンドでローカルAI（Ollama）に問い合わせ、誤答（不正解の選択肢）を
      文字列の綴りだけでなく意味的にも紛らわしいものへ差し替える。
      quiz_create自体はこの処理を待たずに即座に応答を返す（Web側のタイムアウトは
      12秒に対し、CPU動作のローカルAIは数秒〜数十秒かかることがあるため、
      作成リクエストの応答をブロックしない設計にしてある）。
      ホストが開始してしまった後（room["state"] != "lobby"）は差し替えを行わない
      （出題中・発表中のデータを書き換えると、既に配信済みの選択肢と食い違う
      不整合が起きるため）。呼び出しはこの関数専用のバックグラウンドスレッドから行い、
      失敗・タイムアウト時は何もしない（既存の類似度ベースの誤答のまま進行する）。
    """
    with QUIZ_ROOMS_LOCK:
        room = QUIZ_ROOMS.get(code)
        if room is None or room["state"] != "lobby":
            return
        questions = room["questions"]
    try:
        improved = _ai_pick_quiz_distractors(questions, guild_id, deck_filenames)
    except Exception as e:
        print(f"[WARN] クイズ選択肢のAI強化に失敗しました（既存の選択肢のまま続行）: {e}")
        return
    if not improved:
        return
    with QUIZ_ROOMS_LOCK:
        room = QUIZ_ROOMS.get(code)
        if room is None or room["state"] != "lobby":
            return  # 差し替え中にホストが開始してしまった場合は捨てる（進行中データとの不整合防止）
        room["questions"] = improved
```
- AIへの問い合わせの**前後2回**、`room["state"] == "lobby"`であることを確認しています。もしホストがAIの応答を待たずに「開始」してしまい、出題（`question`）や発表（`reveal`）が始まっていたら、`questions`を書き換えるのを中断します。出題中に選択肢の中身が裏で変わってしまうと、既にクライアントへ配信済みの選択肢・正解番号と食い違う不整合が起きるためです。
- `try/except`で例外を丸ごと握りつぶし、失敗時はログを出すだけで既存の選択肢のまま進行します（[00_クイズルームの設計とヘルパー関数.md](00_クイズルームの設計とヘルパー関数.md)で見た「アーカイブ失敗してもクイズ進行は止めない」のと同じ、ベストエフォートの考え方）。

### 7-4. `_ai_pick_quiz_distractors`：候補の絞り込み＋AIへの問い合わせ

```python
def _ai_pick_quiz_distractors(questions, guild_id, deck_filenames):
    """questions（_build_deck_questionsの出力）を受け取り、各問題の誤答3つを
    ローカルAIでより紛らわしいものに差し替えた新しいリストを返す。
    1問ごとに差し替えの成否を判定し、失敗した問題だけ元のままにする。
    AIへの問い合わせ自体が失敗・応答不正だった場合はNoneを返す
    （呼び出し元は既存の選択肢をそのまま使うこと）。"""
    answer_pool = _collect_deck_unique_answers(guild_id, deck_filenames)
    if len(answer_pool) < 2:
        return None

    items = []
    for i, q in enumerate(questions):
        if (q.get("kind") or "choice") == "order":
            continue  # ★ 並び替え問題には誤答が無いのでAI強化の対象外（元のまま残す）
        correct = q["choices"][q["correct_index"]]
        candidates = _distractor_shortlist(
            correct, [a for a in answer_pool if a != correct], QUIZ_AI_SHORTLIST_SIZE
        )
        items.append({"i": i, "question": q["question"], "correct": correct, "candidates": candidates})

    if not items:
        return None  # ★ 4択の問題が1問も無い（並び替え問題だけ）ならAIに問い合わせない
    # ★ 実測（qwen2.5-coder:7b・CPU動作）で3問あたり約35秒かかったため、問題数に応じて
    #   タイムアウトを伸ばす（固定45秒だと出題数が多いクイズで確実に間に合わない）。
    #   バックグラウンドスレッドでの実行なのでWeb側の応答をブロックする心配は無い。
    timeout = max(QUIZ_AI_TIMEOUT_SEC, 20 * len(items))
    result = _ollama_generate_json(_build_quiz_distractor_prompt(items), timeout=timeout)
    ...
```
- `_collect_deck_unique_answers`… `_build_deck_questions`と同じデッキ群から、もう一度「解答（answer）の重複無し一覧」を集め直します（クイズ作成時に選ばれた問題だけでなく、デッキ全体の解答を候補プールにするため）。
- `_distractor_shortlist`… `_pick_distractors`と全く同じ類似度スコア（綴りの類似度70%＋文字数の近さ30%）で並べ替え、**上位12件（`QUIZ_AI_SHORTLIST_SIZE`）だけ**をAIに渡します。全候補をそのまま渡すとプロンプトが膨らみ、CPU動作のAIの応答がさらに遅くなるための絞り込みです。★ 既存の類似度アルゴリズムを「AIに渡す候補の下ごしらえ」として再利用しているのがポイントで、綴りが似た候補の中から**意味的にも紛らわしいものをAIに選ばせる**、という2段構えになっています。
- 全問題ぶんをまとめて**1回のプロンプト**にして`_ollama_generate_json`に渡します（1問ずつ問い合わせると、CPU動作のAIでは往復回数分だけ遅くなるため）。
- タイムアウトは`max(QUIZ_AI_TIMEOUT_SEC, 20 * len(items))`（＝問題数×20秒、最低でも45秒）で動的に決めています。実機（`qwen2.5-coder:7b`・CPUのみ）で3問あたり約35秒かかることを確認しており、固定45秒のままだと出題数の多いクイズ（最大30問）で確実に間に合いません。バックグラウンドスレッドでの実行なのでWeb側の応答をブロックする心配は無いため、時間がかかる方向に振っています。

### 7-5. プロンプトの設計と「併用」方針（`_build_quiz_distractor_prompt`）

AIには「まずcandidates（綴りが似ている実在の解答）の中から意味的に紛らわしいものを3つ選べ、3つ集まらない場合に限り新しく考えて補ってよい」と指示しています。実在するデッキの解答を優先させているのは、AIに完全に自由作文させると事実として不自然・意味不明な誤答が混ざるリスクがあるためで、綴り類似度による絞り込み候補が十分ある通常のデッキでは、実質的に「候補の中から選ぶ」動作になります。出力は`{"questions": [{"i": 0, "distractors": [...]}, ...]}`というJSON1本のみを指示し、Ollama側の`"format": "json"`オプションと組み合わせて壊れた応答を減らしています。

### 7-6. `_sanitize_ai_distractors`：AI応答の検証

```python
def _sanitize_ai_distractors(distractors, correct: str, question: str = ""):
    """AIが返した誤答候補を検証する。重複なく・correctと異なり・禁止文字を含まない
    文字列がちょうど3件そろえばそのリストを、そろわなければNoneを返す
    （None＝この問題は差し替えず、既存の選択肢のまま使う）。"""
    if not isinstance(distractors, list):
        return None
    max_len = max(60, len(correct) * 3)
    kanji = _is_kanji_writing(question, correct)
    cleaned = []
    seen = {correct.strip()}
    for d in distractors:
        text = str(d or "").strip()
        if not text or len(text) > max_len or text in seen or find_bug_chars(text):
            continue
        # 漢字の書き取りは、correctと同じ字数の漢字だけの語に限る（かな書きや別形式は不採用）
        if kanji and (len(text) != len(correct.strip()) or not all(0x4e00 <= ord(ch) <= 0x9fff or ord(ch) == 0x3005 for ch in text)):
            continue
        seen.add(text)
        cleaned.append(text)
        if len(cleaned) == 3:
            break
    return cleaned if len(cleaned) == 3 else None
```
- AIの応答は「JSONとして正しい」ことしか保証されないため（Ollamaの`format: "json"`はJSONとして妥当であることまでしか強制しない）、中身は改めて検証します：文字列であること、空でないこと、正解と異なること（大文字小文字等はそのまま比較）、重複が無いこと、[../02_データ保存基盤/02_設定ファイルと不正文字チェック.md](../02_データ保存基盤/02_設定ファイルと不正文字チェック.md)の`find_bug_chars`で制御文字等を含まないこと、長さが正解の3倍または60文字を超えないこと（AIが長文で暴走するのを防ぐ）。
- **1問につき、有効な誤答がちょうど3件そろわなければその問題ごと`None`**（＝差し替えず、綴り類似度ベースの元の選択肢のまま）にします。「一部だけ差し替える」のような中途半端な救済はせず、AIの応答が信頼できる問題だけを丸ごと採用する設計です。
- `_ai_pick_quiz_distractors`の呼び出し元（`new_questions`構築ループ）でも、`pick`が無い（AIがその問題番号を返さなかった）・`_sanitize_ai_distractors`が`None`を返した、のどちらの場合も**その問題は`questions`の元の値のまま**にして`new_questions`に残します。全30問中1問だけAIの応答がおかしくても、他の29問には影響しません。

### 7-7. まとめ図

```
quiz_create（deck）
  ├─ _build_deck_questions()  … 従来通り即座に4択を生成（この処理自体は変更なし）
  ├─ ルームをstate="lobby"で作成し、即クライアントへ応答
  └─ [OLLAMA_HOST設定時のみ] バックグラウンドスレッドを起動
       └─ _ai_enhance_quiz_room_choices
            ├─ state=="lobby" 確認
            ├─ _ai_pick_quiz_distractors
            │    ├─ _collect_deck_unique_answers … 解答プールを再収集
            │    ├─ _distractor_shortlist × 問題数 … 綴り類似度で候補を12件に絞り込み
            │    ├─ _build_quiz_distractor_prompt → _ollama_generate_json … AIへ一括問い合わせ
            │    └─ _sanitize_ai_distractors × 問題数 … 応答を検証、問題ごとに採否判定
            └─ state=="lobby" を再確認してから room["questions"] を差し替え
```

---

次は、実際にルームを作成・参加・進行させるAPI群（`/quiz_create`から`/quiz_leave`まで）を解説します。 → [02_ルーム操作API.md](02_ルーム作成と過去問ランキング.md)
