# Quiz.js その2：対戦の進行とリアルタイム同期（577〜971行）

[01_Quiz.js_その1_ホストのセットアップとCSV読み込み.md](01_Quiz.js_その1_ホストのセットアップとCSV読み込み.md)の続きです。`Quiz.js`最後のパートです。

---

## 1. ポーリングとSSE（577〜622行）

```js
function startPolling() {
  stopPolling();
  pollOnce();
  pollHandle = setInterval(pollOnce, 400);
  tickHandle = setInterval(tickQuizClocks, 200);
  startQuizRealtime();
}
```
- コメントに「遅延低減：定期ポーリングを1000ms→400msに短縮した上で、他の参加者の操作（参加・開始・回答など）をSSEで即座に検知し、次の定期ポーリングを待たずにその場で取得し直す」とあります。これまでのページ（[../01_index_予定管理.md](../01_index_予定管理.md)など）では10秒間隔のポーリングが「SSEが切れたときの保険」でしたが、**早押しクイズという性質上、遅延が致命的**になるため、保険として使うポーリング自体を400ミリ秒という非常に短い間隔にしています。
- `tickHandle`（200ミリ秒ごと）は、通信を伴わない**見た目だけの更新**（タイマーバーの減り方・カウントダウンの数字）を担当する、別のタイマーです。通信頻度と表示更新頻度を分けることで、通信は400msごとで十分としつつ、見た目の滑らかさは200msごとに保っています。

```js
function startQuizRealtime() {
  try {
    sseHandle = new EventSource(`${API_BASE}events?guild_id=${GUILD_ID}`);
    sseHandle.onmessage = () => { pollOnce(); };
  } catch (e) {}
}
```
- SSE通知が届くたびに、次の400msポーリングを待たずすぐさま`pollOnce()`を呼びます。

```js
async function pollOnce() {
  if (!roomCode) return;
  const data = await apiGet('quiz_state', withAuth({ code: roomCode }));
  if (!data.ok) {
    if (data.error === 'room_not_found') {
      stopPolling();
      await showConfirm({ title: 'このクイズは終了しました', desc: 'ホストが退出したか、時間が経ちすぎたため終了しました。', okLabel: 'ホームに戻る', cancelLabel: '閉じる' });
      backToHomeFromResult();
    }
    return;
  }
  isHost = !!data.is_host;
  applyServerNow(data);
  renderRoom(data.room);
}
```
- 400msごとに（そしてSSE通知のたびに）、ルームの最新状態を丸ごと取得して`renderRoom()`（2節）に渡すだけの、シンプルな仕組みです。ルームが見つからなくなっていたら（ホストが終了した等）、通知してホームへ戻します。

---

## 2. 状態に応じた画面の出し分け：`renderRoom(room)`（627〜641行）

```js
function renderRoom(room) {
  lastRoomSnapshot = room;
  if (room.state === 'lobby') { renderLobby(room); }
  else if (room.state === 'countdown') { renderCountdown(room); }
  else if (room.state === 'intro') { renderIntro(room); }
  else if (room.state === 'question' || room.state === 'reveal') {
    if (isHost) renderHostPlay(room); else renderPlayerPlay(room);
  } else if (room.state === 'ranking') { renderRanking(room); }
  else if (room.state === 'ended') { renderResult(room); }
}
```
- サーバーが管理する`room.state`（このルームが今どの段階にあるか）を見て、対応する描画関数に振り分けるだけの、シンプルな「状態遷移に応じた描画」です。**進行のロジック自体（いつ次の状態に移るか）はすべてサーバー側が担当**しており、クライアント側は「今の状態をそのまま描画する」ことに徹しています。これは、[../02_Cardmaker/09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](../02_Cardmaker/09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)などで見た「サーバーを正とする」設計方針を、ゲーム進行という文脈でも一貫して適用したものです。★ 追加：`"ranking"`（正解発表と次の問題の間に挟む、スコア順の途中経過）の分岐が増えました。詳しくは8節で解説します。

### 2.1 カウントダウン画面（643〜661行）
```js
let lastCountdownStartedAt = null;
function renderCountdown(room) {
  showScreenQ('countdown');
  const el = document.getElementById('cd-num');
  el.dataset.startedAt = room.countdown_started_at || '';
  el.dataset.limit = room.countdown_duration_sec || '';
  if (room.countdown_started_at !== lastCountdownStartedAt) {
    lastCountdownStartedAt = room.countdown_started_at;
    lastCountdownShown = null;
  }
}
```
- コメントに過去のバグ修正が記録されています：「以前はここで毎回`lastCountdownShown`をリセットしていたため、ポーリング（1秒間隔）のたびに『同じ数字』へポップ演出をやり直してしまい、『5,5,4,4,3,3…』のように各数字が2回ずつ表示されて見えるバグがあった。本当に新しいカウントダウンが始まった時（`countdown_started_at`が変わった時）だけリセットする」。ポーリングの頻度と、実際に表示すべき数字が変わる頻度がズレていることで起きた表示バグを、「開始時刻が変わったときだけ」という条件に直すことで解決しています。

### 2.2 ロビー画面（670〜693行）
```js
function renderLobby(room) {
  const screenId = isHost ? 'host-lobby' : 'player-lobby';
  showScreenQ(screenId);
  const titleEl = document.getElementById(isHost ? 'hl-title' : 'pl-title');
  titleEl.textContent = room.title;

  if (isHost) {
    document.getElementById('hl-count').textContent = `参加者 ${room.players.length}人`;
    document.getElementById('hl-players').innerHTML = playerChipsHtml(room.players);
    document.getElementById('hl-start-btn').disabled = room.players.length === 0;
  } else {
    document.getElementById('pl-status').textContent = `${room.host_nickname} さんが開始するのを待っています…（参加者 ${room.players.length}人）`;
    document.getElementById('pl-players').innerHTML = playerChipsHtml(room.players);
  }
}

function playerChipsHtml(players) {
  if (!players.length) return `<p style="color:var(--text-dim);font-size:13px;">まだ誰も参加していません</p>`;
  return players.map(p => `
    <div class="qz-player-chip">
      <span class="qz-avatar" style="background:${escapeHtml(p.color)};color:${escapeHtml(p.text_color)}">${escapeHtml((p.nickname || '').slice(0, 2).toUpperCase())}</span>
      ${escapeHtml(p.nickname)}
    </div>`).join('');
}
```
- ホストと参加者で表示先の画面ID自体を切り替えつつ、参加者が0人ならホストの「クイズを始める」ボタンを押せなくします。

---

## 3. 出題・回答・正解発表の共通描画：`renderPlayScreen(room, opts)`（695〜786行）

コメントに設計意図が説明されています：「ホストも1人の参加者として一緒に回答する。出題画面はホスト用／プレイヤー用でほぼ同じ処理になるため、共通ロジックをここにまとめる」。`opts`引数として、ホスト画面用・参加者画面用それぞれの要素IDのセットを渡すことで、**1つの関数でどちらの画面も描画できる**ようにしています（`renderHostPlay`/`renderPlayerPlay`、5節）。

### 3.1 問題が変わったかどうかの検知（704〜707行）
```js
const qChanged = room.current_q !== renderedQIndex;
const stateChanged = room.state !== renderedState;
if (qChanged) hasAnsweredThisQ = false;
renderedQIndex = room.current_q; renderedState = room.state;
```
- `renderedQIndex`/`renderedState`（グローバル変数）に「直近に描画した問題番号・状態」を覚えておき、今回のデータと比較します。問題番号が変わっていれば、`hasAnsweredThisQ`（この問題に回答済みか）フラグをリセットします。

### 3.2 選択肢ボタンの再構築（719〜724行）
```js
const choicesEl = document.getElementById(choicesId);
if (qChanged || stateChanged || !choicesEl.dataset.built || Number(choicesEl.dataset.built) !== room.current_q) {
  choicesEl.innerHTML = room.question.choices.map((c, i) => `
    <button class="qz-choice-btn ${CHOICE_CLASSES[i]}" onclick="submitAnswer(${i}, '${choicesId}', '${waitingNoteId}')">${escapeHtml(c)}</button>`).join('');
  choicesEl.dataset.built = room.current_q;
}
```
- 選択肢ボタン自体は、**問題が変わったとき（または初回）だけ**作り直します。`choicesEl.dataset.built`に「今表示している選択肢がどの問題番号のものか」を記録しておき、それと今回のデータの`current_q`が一致していれば、ボタンの再構築自体をスキップします。400msごとの高頻度なポーリングのたびに、変わっていない部分までDOMを作り直すのは無駄なので、必要なとき（問題が切り替わったとき）だけに絞る最適化です。

★ 追加（2026/08/27）：「選択肢は下ぞろえにしてほしい（したすぎると端末によっては隠れてしまうので注意）。結果発表はいまのまま」との要望を受け、正解発表前だけ選択肢を画面下側へ寄せるようにした。初版は選択肢欄自体に`margin-top:auto`を付ける方式だったが、直後に「ボタンの位置はパッと切り替えずにスライドしていくように」との追加要望があり、`margin-top:auto`はCSSトランジションで滑らかに補間できない（`auto`は本質的にアニメーション不可な値）ため、専用の空要素（`.qz-choice-spacer`）の`flex-grow`をトグルする方式に作り直した。
```js
const choiceSpacerEl = choicesEl.previousElementSibling;
if (choiceSpacerEl && choiceSpacerEl.classList.contains('qz-choice-spacer')) {
  choiceSpacerEl.classList.toggle('qz-choices-bottom', !revealed);
}
```
```css
.qz-choice-spacer{ flex-grow:0; transition:flex-grow .35s ease; }
.qz-choice-spacer.qz-choices-bottom{ flex-grow:1; }
```
- `<div class="qz-choice-spacer">`は問題文（`qz-question-text`）と選択肢欄（`qz-choice-grid`）の間に置かれた、中身が空のdiv。`.qz-body`は`flex-direction:column`なので、この要素の`flex-grow`を0→1にすると、余っている縦方向の空間（＝問題文が短くて余っている分）をこの要素が独り占めして伸び、結果として選択肢欄より下（選択肢・正解発表パネル等）が一塊のまま下へ押しやられる。`flex-grow`は数値（`<number>`）なので`margin:auto`と違いCSSトランジションで正しく補間され、正解発表の前後でスーッとスライドする。
- 「したすぎると端末によっては隠れてしまう」という懸念への対処が、実はこの手法の肝。`flex-grow`は**実際に余っているスペースの範囲内でしか**伸びない性質を持つ（他の要素を強制的に縮めてまでスペースを作ったりはしない）。つまり問題文が長い・画面が低いなどで余白がそもそも無い場合は自動的に0のまま＝今まで通りの詰めた配置になるだけで、無理に画面外へ押し出されることは無い。特別な分岐やメディアクエリを書かずに、CSSの標準的な性質だけで安全側に倒れる設計になっている（`margin:auto`版でも成り立っていた性質で、`flex-grow`版に切り替えても維持されている）。
- 選択肢欄自体のidは`choicesId`（`opts`経由でホスト/参加者それぞれの実IDが渡ってくる）だが、隣に置いた`.qz-choice-spacer`には専用のidを振らず、`choicesEl.previousElementSibling`（＝HTML上で選択肢欄の直前に置かれた要素）で辿っている。ホスト画面・参加者画面どちらのHTMLでも「質問文→spacer→選択肢欄」の順に並べてあることが前提。

同じタイミングで「まだスワイプできちゃう。ボタンちょっとしたすぎ」との指摘も受け、Quiz.cssを2点調整した：
- `body.qz-fixed-screen`（出題画面・途中経過画面の表示中だけ付く、`showScreenQ`参照）に`touch-action:none`を追加。`overflow:hidden`・`overscroll-behavior`（`html`側で全ページ共通に設定済み）だけでは、iOS Safari等でタッチのスワイプ操作そのもの（弾む・わずかに動く挙動）を止めきれない端末があったための、より強い指定。途中経過の参加者一覧（`.qz-scroll-list`）だけは人数が多い場合に備えて`touch-action:pan-y`で縦スワイプを明示的に許可し直している。
- 出題画面の`.qz-body`の下側パディングに`env(safe-area-inset-bottom, 0px)`を追加（`calc(1.25rem + env(safe-area-inset-bottom, 0px))`）。選択肢を下寄せするようになったことで、ホームインジケーター等のセーフエリアに選択肢が近づきすぎる端末があったための対応。
- 正解発表（`revealed`）になったらこのクラスを外し、今まで通りの配置に戻す（「結果発表はいまのまま」という要望通り、見た目を変えないため）。

### 3.3 選択肢の見た目の更新（726〜743行）
```js
[...choicesEl.children].forEach((btn, i) => {
  const picked = yourAnswer === i;
  btn.disabled = answered || revealed;
  btn.classList.toggle('qz-picked', picked);
  if (revealed) {
    const isCorrect = i === room.question.correct_index;
    btn.classList.toggle('qz-correct-flash', isCorrect);
    btn.classList.toggle('qz-wrong-flash', picked && !isCorrect);
    btn.classList.toggle('qz-dim', !isCorrect && !picked);
  } else {
    btn.classList.remove('qz-correct-flash', 'qz-wrong-flash');
    btn.classList.toggle('qz-dim', answered && !picked);
  }
});
```
- 選択肢ボタン自体（3.2節）とは別に、**見た目のクラス（色や薄さ）だけはポーリングのたびに毎回更新**しています。これにより、正解発表（`revealed`）になった瞬間に、正解の選択肢が光り、自分が選んだ間違った選択肢が赤く光る、という演出がスムーズに反映されます。
- コメントに「回答直後（発表前）も、選んでいない残りの選択肢を薄くして、『自分が押したのはこれ』を最後まではっきり見せ続ける」とあります。

### 3.4 正解・不正解のフィードバック（745〜769行）
```js
  const feedbackEl = document.getElementById(feedbackId);
  const waitingNote = document.getElementById(waitingNoteId);
  if (revealed && yourAnswer !== undefined) {
    feedbackEl.style.display = '';
    if (room.your_correct) {
      const bonus = room.first_correct_nickname === STUDENT.nickname;
      feedbackEl.className = 'qz-answer-feedback ok';
      feedbackEl.innerHTML = bonus ? (Icons.html('celebrate', {size:15}) + ' 正解！一番乗りボーナスで +12点！') : (Icons.html('checkCircle', {size:15}) + ' 正解！ +10点');
    } else {
      feedbackEl.className = 'qz-answer-feedback ng';
      feedbackEl.innerHTML = Icons.html('wrong', {size:15}) + ' 不正解…';
    }
    waitingNote.style.display = 'none';
  } else if (revealed && yourAnswer === undefined) {
    feedbackEl.style.display = '';
    feedbackEl.className = 'qz-answer-feedback ng';
    feedbackEl.innerHTML = Icons.html('timer', {size:15}) + ' 時間切れで未回答でした';
    waitingNote.style.display = 'none';
  } else if (answered) {
    feedbackEl.style.display = 'none';
    waitingNote.style.display = '';
  } else {
    feedbackEl.style.display = 'none';
    waitingNote.style.display = 'none';
  }
```
- 「正解した（かつ一番乗りだった）」「正解した（一番乗りではない）」「不正解」「時間切れで未回答」「回答済みで発表待ち」「未回答」という、いくつものパターンを網羅的に出し分けています。正解の基本点は10点、一番乗りボーナスは+2点（合計12点）と、HTMLのヒーロー文言（`正解10点、一番早く正解すると+2点`）と一致する数値がここに実装されています。

### 3.5 正解発表パネル（774〜785行）
```js
  // ★ 修正：以前はホスト画面（hp-reveal）だけに「一番早く正解した人」と
  //   ミニ順位表を表示していた。参加者にはホストと同じ情報を見せていなかった
  //   ため、ホスト・参加者共通のこの関数から両方の画面を描画するようにする。
  const revealPanel = document.getElementById(revealPanelId);
  if (room.state === 'reveal') {
    revealPanel.style.display = '';
    document.getElementById(firstBadgeId).innerHTML = room.first_correct_nickname
      ? `<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#eab308" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M13 3 4 14h6l-1 7 9-11h-6Z"/></svg> 一番早く正解：${escapeHtml(room.first_correct_nickname)} さん（+2点ボーナス）`
      : '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#eab308" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M13 3 4 14h6l-1 7 9-11h-6Z"/></svg> 正解者はいませんでした';
    document.getElementById(leaderboardId).innerHTML = miniLeaderboardHtml(room.players);
  } else {
    revealPanel.style.display = 'none';
  }

  updateTimerBarFor(room, timerbarId);
}
```
- コメントに「以前はホスト画面だけに『一番早く正解した人』とミニ順位表を表示していた。参加者にはホストと同じ情報を見せていなかったため、ホスト・参加者共通のこの関数から両方の画面を描画するようにする」という改善の経緯があります。
- ★ 修正（2026/08/27）：`firstBadgeId`への代入は、以前このドキュメントの旧版では`textContent`と誤って転記されていましたが、実際には`innerHTML`です（`textContent`のままだとSVGタグがそのまま文字として表示されてしまうバグが実際にあり、修正済みです）。あわせて`first_correct_nickname`（ニックネーム）も`escapeHtml()`を通すよう修正されています。

```js
// ★ 修正：正解発表(reveal)中は、サーバーが players に answered/correct を
//   含めて返すようになったので、その問題への各参加者の◯×も一緒に表示する。
//   （出題中は correct フィールド自体が来ないので、その場合は何も表示しない）
function miniLeaderboardHtml(players) {
  return players.slice(0, 5).map((p, i) => `
    <div class="qz-lb-row">
      <span class="qz-lb-rank">${i + 1}</span>
      <span class="qz-avatar" style="background:${escapeHtml(p.color)};color:${escapeHtml(p.text_color)}">${escapeHtml((p.nickname || '').slice(0, 2).toUpperCase())}</span>
      <span class="qz-lb-name">${escapeHtml(p.nickname)}</span>
      ${quizMarkHtml(p)}
      <span class="qz-lb-score">${p.score}点</span>
    </div>`).join('');
}

function quizMarkHtml(p) {
  if (typeof p.correct === 'undefined') return ''; // 出題中など、正誤情報が無い場合は何も出さない
  if (!p.answered) return `<span class="qz-lb-mark unanswered" title="未回答">―</span>`;
  return p.correct
    ? `<span class="qz-lb-mark correct" title="正解">◯</span>`
    : `<span class="qz-lb-mark wrong" title="不正解"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="#dc2626" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M6 6l12 12"/><path d="M18 6L6 18"/></svg></span>`;
}
```
- 上位5人だけに絞ったミニ順位表を、各参加者の◯×付きで表示します。コメントに「正解発表(reveal)中は、サーバーが`players`に`answered`/`correct`を含めて返すようになった」とあり、出題中はこの情報自体が送られてこないため、`quizMarkHtml`はその場合何も表示しません（`p.correct`が`undefined`かどうかで判定）。
- ★ 変更（2026/08/27）：ここで並んでいる`players`は、以前はスコア降順でしたが、サーバー側（[../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md](../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md)の`_quiz_room_players_json_press_order`）で「その問題に回答したのが早い順（正誤にかかわらず）」に並べ替えられたものが送られてくるようになりました。クライアント側の`miniLeaderboardHtml`自体は`players.slice(0, 5)`で受け取った順にそのまま表示するだけなので、コードに変更はありません（サーバー側の並び順を変えるだけで見た目が変わる、「サーバーを正とする」設計の一例です）。全体のスコア順位は、次の問題までの間に挟まる`"ranking"`画面（8節）・最終結果（6節）の方で見せます。

---

## 4. タイマーバーとカウントダウンの滑らかな更新（830〜881行）

```js
function updateTimerBarFor(room, elId) {
  const el = document.getElementById(elId);
  if (room.state === 'reveal') {
    el.dataset.startedAt = room.reveal_started_at || '';
    el.dataset.limit = room.reveal_duration_sec || '';
  } else {
    el.dataset.startedAt = room.question_started_at || '';
    el.dataset.limit = room.time_limit_sec || '';
  }
}
```
- タイマーバーの「基準となる開始時刻・制限時間」を、データ属性として要素に保存しておくだけの関数です。実際の見た目の更新（毎フレームのアニメーション）は別のループ（`tickQuizClocks`）が担当します。出題中は「問題の制限時間」、正解発表中は「次の問題へ自動で進むまでの残り時間」を表示する、という切り替えです。

```js
function tickQuizClocks() {
  const serverNow = Date.now() + quizClockOffsetMs;
  ['hp-timerbar', 'pp-timerbar'].forEach(id => {
    const el = document.getElementById(id);
    if (!el || !el.dataset.startedAt || !el.dataset.limit) return;
    const started = Number(el.dataset.startedAt) * 1000;
    const limit = Number(el.dataset.limit) * 1000;
    const remain = Math.max(0, limit - (serverNow - started));
    el.style.transform = `scaleX(${remain / limit})`;
  });
  tickCountdownNum(serverNow);
}
```
- [00_HTML構造とその1_起動と参加画面.md](00_HTML構造とその1_起動と参加画面.md)で見た`quizClockOffsetMs`（時計のズレ補正）を使って、「サーバー時計での今の時刻」を計算し、そこから残り時間の割合を求めて、`scaleX(...)`（CSSの拡大縮小変形）でタイマーバーの幅を表現しています。200ミリ秒ごとにこの計算をやり直すことで、通信なしに滑らかにバーが減っていく演出を実現しています。

```js
let lastCountdownShown = null; // ★ 追加：直前に表示した数字（変わった時だけポップ演出を出し直すため）
function tickCountdownNum(serverNow) {
  const el = document.getElementById('cd-num');
  if (!el || !el.dataset.startedAt || !el.dataset.limit) return;
  const started = Number(el.dataset.startedAt) * 1000;
  const limit = Number(el.dataset.limit) * 1000;
  const remainSec = Math.max(1, Math.ceil((limit - (serverNow - started)) / 1000));
  if (remainSec !== lastCountdownShown) {
    lastCountdownShown = remainSec;
    el.textContent = String(remainSec);
    // ★ 数字が変わるたびに qzCountPop アニメーションを最初から再生させる
    //   （同じアニメーション名を付け直すだけではブラウザが「変化なし」と
    //   判断して再生してくれないため、一度 none にしてから戻すテクニックを使う）。
    el.style.animation = 'none';
    void el.offsetWidth; // 強制リフローでスタイルの変更を確定させる
    el.style.animation = '';
  }
}
```
- カウントダウンの数字表示です。`el.style.animation = 'none'`のあと`void el.offsetWidth`（要素の幅を読み取るだけの、値を使わない命令）を挟んでから`el.style.animation = ''`に戻す、というテクニックが使われています。コメントに「同じアニメーション名を付け直すだけではブラウザが『変化なし』と判断して再生してくれないため、一度`none`にしてから戻すテクニックを使う」とあります。`el.offsetWidth`を読み取る操作は、ブラウザに「今すぐ実際のレイアウト計算をしてください」と強制する副作用があり（これを**強制リフロー**と呼びます）、これを挟むことで「アニメーション無し」の状態が本当に一度ブラウザに反映されてから、再びアニメーション有りに戻すことができ、結果としてアニメーションを最初からやり直させることができます。

---

## 5. 回答の送信とホスト操作（883〜937行）

```js
async function submitAnswer(choiceIndex, choicesId = 'pp-choices', waitingNoteId = 'pp-waiting-note') {
  if (hasAnsweredThisQ) return;
  hasAnsweredThisQ = true;
  const choicesEl = document.getElementById(choicesId);
  [...choicesEl.children].forEach((btn, i) => {
    const picked = i === choiceIndex;
    btn.disabled = true;
    btn.classList.toggle('qz-picked', picked);
    btn.classList.toggle('qz-dim', !picked);
  });
  const waitingNote = document.getElementById(waitingNoteId);
  if (waitingNote) waitingNote.style.display = '';
  await apiPost('quiz_answer', withAuth({ code: roomCode, choice_index: choiceIndex }));
  pollOnce();
}
```
- 回答ボタンが押された瞬間、**サーバーへの送信を待たずに**、その場でボタンを無効化し、選んだ選択肢をハイライトして「他の人の解答を待っています…」を表示します（[../02_Cardmaker/*](../02_Cardmaker/00_HTML構造とページ全体像.md)で見た「楽観的更新」に近い、見た目の即時反映です）。そのあとで実際にサーバーへ送信し、成功・失敗に関わらず`pollOnce()`で最新状態を取り直します。

```js
async function hostStart() {
  const btn = document.getElementById('hl-start-btn');
  btn.disabled = true;
  const data = await apiPost('quiz_start', withAuth({ code: roomCode }));
  if (!data.ok) { btn.disabled = false; showAppAlert({ title: 'エラー', desc: quizErrorText(data.error) }); return; }
  applyServerNow(data);
  renderRoom(data.room);
}
async function confirmQuitHost() {
  if (quitting) return;
  const ok = await showConfirm({
    title: 'クイズを終了しますか？',
    desc: '進行中のクイズを終了し、ホームに戻ります。参加者にはそこまでの結果が表示されます。',
    okLabel: '終了する',
  });
  if (!ok) return;
  quitting = true;
  await apiPost('quiz_end', withAuth({ code: roomCode }));
  quitting = false;
  stopPolling();
  backToHomeFromResult();
}
async function confirmQuitPlayer() {
  if (quitting) return;
  const ok = await showConfirm({
    title: 'クイズから退出しますか？',
    desc: 'これまでの得点は保存されません。',
    okLabel: '退出する',
  });
  if (!ok) return;
  quitting = true;
  await apiPost('quiz_leave', withAuth({ code: roomCode }));
  quitting = false;
  stopPolling();
  backToHomeFromResult();
}
```
`hostStart()`（902〜909行）はロビーからクイズを開始するホスト専用の操作、`confirmQuitHost()`/`confirmQuitPlayer()`（910〜937行）はそれぞれホスト・参加者が確認ダイアログのあとクイズから抜ける処理です。`quitting`フラグで、確認ダイアログを閉じている間の二重実行を防いでいます。
- ★ 追加（2026/08/27）：`confirmQuitPlayer()`自体は元からありましたが、呼び出せるボタンは以前ロビー画面（`#screen-player-lobby`）にしかなく、出題が始まってしまうと参加者は退出できませんでした。出題画面（`#screen-player-play`）のヘッダーにも同じボタンを追加し、対戦中いつでも退出できるようにしました（ホスト用の出題画面には元々`confirmQuitHost()`ボタンがありました）。

---

## 6. 結果発表：`renderResult(room)`（942〜965行）

```js
// ★ 修正（2026/08/27）：room.playersはサーバー側で既にスコア降順（同点は
//   回答時間合計が短い方が上）に並べ済みなので、ここで再ソートしない。
const sorted = room.players;
const podiumOrder = [sorted[1], sorted[0], sorted[2]]; // 2位・1位・3位の順で表示（真ん中が1位）
// ★ 修正（2026/08/27）：金・銀・銅が同じ形のアイコンを色だけ変えて表示していたため
//   「全部1位に見える」という指摘があった。線画から塗りつぶしに変え、色もはっきり離した。
const medalMap = [
  `<svg width="18" height="18" viewBox="0 0 24 24" fill="#94a3b8" stroke="#fff" stroke-width="1.3" ...>...</svg>`, // 銀（明度を落とした濃いグレー）
  `<svg width="18" height="18" viewBox="0 0 24 24" fill="#f5b400" stroke="#fff" stroke-width="1.3" ...>...</svg>`, // 金
  `<svg width="18" height="18" viewBox="0 0 24 24" fill="#c2703d" stroke="#fff" stroke-width="1.3" ...>...</svg>`, // 銅（彩度を上げた銅色）
];
const podiumClass = ['qz-podium-2', 'qz-podium-1', 'qz-podium-3'];
document.getElementById('result-podium').innerHTML = podiumOrder.map((p, i) => {
  if (!p) return `<div class="qz-podium-col ${podiumClass[i]}"></div>`;
  return `
    <div class="qz-podium-col ${podiumClass[i]}">
      <div class="qz-podium-name">${escapeHtml(p.nickname)}</div>
      <div class="qz-podium-score">${p.score}点</div>
      <div class="qz-podium-bar"><span class="qz-podium-medal">${medalMap[i]}</span></div>
    </div>`;
}).join('');
// ★ 追加：自分の行をひと目で分かるようにハイライトする
document.getElementById('result-list').innerHTML = sorted.map((p, i) => `
  <div class="qz-lb-row${p.id === STUDENT.id ? ' qz-lb-row-me' : ''}">
    <span class="qz-lb-rank">${i + 1}</span>
    <span class="qz-avatar" style="background:${escapeHtml(p.color)};color:${escapeHtml(p.text_color)}">${escapeHtml((p.nickname || '').slice(0, 2).toUpperCase())}</span>
    <span class="qz-lb-name">${escapeHtml(p.nickname)}</span>
    <span class="qz-lb-score">${p.score}点</span>
  </div>`).join('');
```
- 表彰台の見た目（左に2位、真ん中に1位、右に3位、という一般的な表彰台のレイアウト）に合わせて並び替えています。参加者が2人以下の場合、`sorted[2]`（3位）は`undefined`になりますが、`podiumOrder.map`内の`if (!p) return ...`分岐で、存在しない順位は空の列（メダル無しの空の`<div>`）として描画され、エラーにはなりません。
- ★ 修正（2026/08/27）：以前は`[...room.players].sort((a, b) => b.score - a.score)`とスコアだけで**クライアント側が独自に再ソート**していましたが、これだとサーバー側が同点タイブレーク（回答時間の合計）まで含めて並べ替えた順番が、ここで台無しになってしまっていました。サーバー側の並び順をそのまま信用する（`const sorted = room.players;`）ように直し、`renderPlayScreen`のミニ順位表（3.5節）と同じ「サーバーを正とする」設計に統一しました。
- ★ 追加：自分の行に`qz-lb-row-me`クラスを付け、色を変えてひと目で分かるようにしました。

`document.getElementById('result-list')`には、上位に限らず**全参加者**の順位・得点を一覧表示します。

---

## 7. 締めくくり（967〜971行）

```js
initQuizApp();
hideLoadingFallback();
```
- [00_HTML構造とその1_起動と参加画面.md](00_HTML構造とその1_起動と参加画面.md)で説明した起動処理を実際に呼び出し、最後に「読み込み中…」保険オーバーレイを消します。

---

## 8. 追記（2026/08/27）：途中経過ランキング画面・途中参加の見学待機・レイアウト修正

ユーザーから「Kahootのように、正解発表の後に順位の上げ下げをアニメーションで見せてほしい」「途中参加は次の問題から」「タイマーがスクロールしないと見えない」等、複数の要望を受けてまとめて対応しました。

### 8.1 途中経過画面：`renderRanking(room)`（新規）

```js
function renderRanking(room) {
  showScreenQ('ranking');
  const listEl = document.getElementById('rk-list');

  const myIndex = room.players.findIndex(p => p.id === STUDENT.id);
  document.getElementById('rk-my-rank').textContent = myIndex !== -1
    ? `あなたは 現在 ${myIndex + 1}位 / ${room.players.length}人中`
    : '';

  const firstTops = {}; // First：描画し直す前の各行の位置
  [...listEl.children].forEach(row => {
    firstTops[row.dataset.playerId] = row.getBoundingClientRect().top;
  });

  listEl.innerHTML = room.players.map((p, i) => `
    <div class="qz-lb-row${p.id === STUDENT.id ? ' qz-lb-row-me' : ''}" data-player-id="${p.id}">
      <span class="qz-lb-rank">${i + 1}</span>
      <span class="qz-avatar" ...>${escapeHtml((p.nickname || '').slice(0, 2).toUpperCase())}</span>
      <span class="qz-lb-name">${escapeHtml(p.nickname)}</span>
      <span class="qz-lb-score">${p.score}点</span>
    </div>`).join('');

  const myRow = listEl.querySelector('.qz-lb-row-me');
  if (myRow) myRow.scrollIntoView({ block: 'nearest' });

  if (!Object.keys(firstTops).length) return; // 初回（1問目終了後）は前回位置が無いのでアニメーションなしで出す

  const rows = [...listEl.children];
  rows.forEach(row => {
    const oldTop = firstTops[row.dataset.playerId];
    if (oldTop === undefined) return;
    const delta = oldTop - row.getBoundingClientRect().top;
    if (!delta) return;
    row.style.transition = 'none';
    row.style.transform = `translateY(${delta}px)`;
  });
  void listEl.offsetHeight; // 強制リフローでtransformを確定させてから外す
  rows.forEach(row => {
    row.style.transition = 'transform .5s cubic-bezier(.2,.8,.2,1)';
    row.style.transform = '';
  });
}
```
- サーバー側の`room.players`は既にスコア降順（同点タイブレーク込み）で並んでいるので、そのまま`i + 1`を順位として表示するだけです。
- 順位変動のアニメーションは**FLIP手法**（First・Last・Invert・Play）で実装しています。①（First）DOMを描画し直す**前**に、既存の各行（`data-player-id`で識別）の画面上の位置を`getBoundingClientRect().top`で記録する → ②（Last）新しい順位でDOMを丸ごと作り直す → ③（Invert）各行について「新しい位置」と「元の位置」の差分（`delta`）を計算し、`transform: translateY(delta px)`で**わざと元の位置に見えるようにずらす**（この時点では`transition: none`でアニメーションさせない） → ④ `void listEl.offsetHeight`で強制リフローし、ブラウザに③のズレを一旦確定させる → ⑤（Play）`transition`を有効にしてから`transform`を`''`（本来の新しい位置）に戻す、という流れです。⑤の瞬間、ブラウザは「ズレた位置」から「本来の位置」へ**トランジションで滑らかに動かして**くれます。DOM要素自体は既に新しい位置にあるのに、見た目だけ一瞬「前の位置にいた」ように偽装してから正しい位置へアニメーションさせる、という考え方がFLIPの核心です。
- 1問目終了後の最初の`"ranking"`画面には「前回の位置」がまだ無い（`listEl`が空）ため、`firstTops`が空のまま`return`し、アニメーション無しでそのまま表示します。新規途中参加者などその回だけ現れた行も、`firstTops[id]`が`undefined`になるため個別にアニメーション対象から外れます。
- `rk-my-rank`（新規要素）に「あなたは現在◯位/N人中」を表示し、自分の行には`qz-lb-row-me`クラスでハイライトを付けます。一覧が参加者過多で内部スクロールする場合に自分の行が隠れたままにならないよう、`scrollIntoView({ block: 'nearest' })`で必要な時だけ表示範囲に入れます。

### 8.1.1 追記（2026/08/27）：5問に1度だけ挟む + モーションを増やす

ユーザーから「毎問だと間延びする、5問に1度でいい」「もう少しモーションを多めに。ただし表示時間は今のままで」という要望を受け、`"ranking"`を挟む頻度と演出をさらに調整しました。

- **サーバー側**（`bot.py`、[00_クイズルームの設計とヘルパー関数.md](../../1I勉強会bot解説/15_FlaskAPI_クイズ/00_クイズルームの設計とヘルパー関数.md)参照）：`QUIZ_RANKING_INTERVAL = 5`を追加し、`"reveal"`から次への遷移で「今revealし終えた問題の番号が5の倍数（5問目・10問目…）」の時だけ`"ranking"`へ、それ以外は`"ranking"`を飛ばして直接次の問題の`"intro"`へ進むよう変更しました。**`QUIZ_RANKING_DURATION_SEC`（表示時間・4秒）自体は変更していません**——「頻度は減らすが、出た時の長さは変えない」という要望通りです。
- **フロント側**（`renderRanking`にモーションを3つ追加。表示時間には影響しません）：
  1. **順位変動バッジ**：`firstTops`と同時に`firstRanks`（描画し直す前の各行の順位＝旧`i+1`）も記録しておき、新しい順位と比べて上下していれば、順位数字の右に▲（上昇・緑）／▼（下降・赤）の小さいバッジ（`.qz-rank-move`）を追加します。絵文字や`▲`/`▼`のようなUnicode文字は使わず、`.qz-choice-a`等の4択ボタンと同じ「CSSのborderで作る三角形」で表現しています（サイト全体の「OSに見た目を依存させない」方針の踏襲）。バッジ自体も`qzRankPop`というちょっと弾む（overshootする）`@keyframes`で登場します。
  2. **行の移動をより弾ませる**：FLIPの⑤（Play）で使う`transition`のイージングを`cubic-bezier(.2,.8,.2,1)`から`cubic-bezier(.34,1.56,.64,1)`（値が1を超える区間があるため、目標位置を一瞬追い越してから収まる「弾む」動き）に、`.5s`から`.6s`に変更しました。
  3. **初回表示（前回位置が無くFLIPできない回）の演出追加**：以前は`firstTops`が空なら何もせずそのまま表示していましたが、今は各行に`qzRowIn`という`@keyframes`（下からふわっと現れる）を、`animationDelay`を行の順番（`i`）に応じて少しずつずらして（`i * 35ms`、最大350ms）付けるようにしました。上から順に一拍ずつ遅れて現れる、いわゆる「stagger（スタッガー）」演出です。
- これらのCSS（`.qz-rank-move`・`@keyframes qzRankPop`・`@keyframes qzRowIn`・`#rk-list .qz-lb-rank`の上書き）は`Quiz.css`に追加しましたが、**`#rk-list`配下だけに効くセレクタで絞っています**。同じ`.qz-lb-row`クラスは正解発表(reveal)中のミニ順位表や最終結果画面でも使い回されているため、スコープを絞らないとそちらにも意図せず影響してしまうためです。

### 8.2 途中参加でまだ見学中の場合：`renderPlayScreen`の`spectating`分岐（追加）

```js
if (room.spectating) {
  if (spectateNote) spectateNote.style.display = '';
  document.getElementById(questionId).textContent = '';
  document.getElementById(choicesId).innerHTML = '';
  document.getElementById(choicesId).dataset.built = '';
  document.getElementById(feedbackId).style.display = 'none';
  document.getElementById(waitingNoteId).style.display = 'none';
  document.getElementById(revealPanelId).style.display = 'none';
  document.getElementById(answeredCountId).textContent = '';
  const timerEl = document.getElementById(timerbarId);
  delete timerEl.dataset.startedAt; delete timerEl.dataset.limit;
  timerEl.style.transform = 'scaleX(0)';
  return;
}
```
- サーバー側（[../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md](../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md)参照）が、途中参加でまだこの問題に混ざれない本人にだけ`room.spectating: true`を返すようになったのを受けて、通常の問題・選択肢の描画をすべて`return`で打ち切り、代わりに専用の案内（`#pp-spectate-note`「待機中…（次の問題から参加します）」）だけを表示します。タイマーバーの`dataset`も消してから幅を0にし、直前の問題の残り時間表示が変な形で残らないようにしています。ホストは常に`active_from_q: 0`（最初からアクティブ）なので、この分岐に入ることはありません（`renderHostPlay`は`spectateNoteId`を渡していません）。

### 8.3 表彰台のメダルアイコン（6節の再掲）・待機画面のアイコン・レイアウトの修正

- 6節で触れた通り、表彰台のメダルアイコンを線画（色違いだけ）から塗りつぶし＋色をよりはっきり離したものに変更しました。
- 参加者ロビー画面（`#screen-player-lobby`）の待機中アイコンだった絵文字`⏳`は、[../02_Cardmaker/00_HTML構造とページ全体像.md](../02_Cardmaker/00_HTML構造とページ全体像.md)等で触れられている「見た目はOSの標準UI・絵文字に頼らない」という全ページ共通方針からこの画面だけ漏れていたため、`Icons.js`の`timer`アイコンと同じ形のSVGに置き換え、`qzBreathe`（ゆっくり明滅・拡大縮小する）アニメーションを付けました。
- 出題画面（`#screen-host-play`/`#screen-player-play`）・途中経過画面（`#screen-ranking`）は、タイマーバー等がページの高さの都合でスクロールしないと見えないことがあったため、`height:100dvh; overflow:hidden`で画面ぴったりに収め、ページ自体のスクロールを止めました。途中経過の参加者一覧だけは、人数が多く入りきらない場合に備えて`.qz-scroll-list`（`overflow-y:auto`）で中身だけスクロールできるようにしています。

---

## まとめ

`Quiz.js`は、他ページと共通のパターン（`esc()`によるエスケープ、SSE＋ポーリング、`AbortController`によるタイムアウト）を踏襲しつつ、リアルタイム性が特に重要なこのページ特有の工夫として、**ポーリング間隔を極端に短く（400ms）する**、**通信を伴う更新と見た目だけの更新（200msのtickループ）を分離する**、**サーバー時計とのズレを補正してタイマー表示をピッタリ合わせる**、**DOM要素の再構築を必要最小限に絞る（選択肢ボタンは問題が変わったときだけ作り直す）**といった、ゲームらしい体験のための最適化が随所に見られました。

これで`Quiz.html`＋`Quiz.js`の解説はすべて完了です。続きは[../06_Notice/00_HTML構造とその1_一覧と詳細表示.md](../06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で、お知らせページ（`Notice.html`/`Notice.js`）の全体像から解説します。
