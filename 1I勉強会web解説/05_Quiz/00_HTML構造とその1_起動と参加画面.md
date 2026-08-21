# みんなでクイズページ — 全体像・起動処理・参加画面（1〜394行）

対象：`bot.1istudy.web/Quiz.html`（294行）、`bot.1istudy.web/Quiz.js`（971行）。用語は[[../01_index_予定管理.md]]の「0. ミニ用語辞典」も参照してください。

## このページの特徴

「みんなでクイズ」は、CardMakerのデッキから自動生成した（または自分で作った）4択問題を、複数人でオンライン早押し形式で遊ぶ機能です。他のページと違い**ドロワーメニューを持たず**、CardMakerからのリンク経由でのみアクセスする単発の目的ページです（[[../02_Cardmaker/00_HTML構造とページ全体像.md]]参照）。

`Quiz.html`は`Cardmaker.html`と同じ「1枚のHTMLに全画面を用意しておき、JSで`active`クラスを付け替える」方式のSPAです。画面は多く、ゲームの進行段階に対応しています：

| id | 画面 | 役割 |
|---|---|---|
| `screen-loading` | 読み込み中 | 最初に一瞬だけ表示 |
| `screen-home` | ホーム | 「参加する」「クイズを作る」の選択 |
| `screen-join` | 参加：ルーム選択 | 参加できるクイズの一覧 |
| `screen-host-setup` | ホスト：セットアップ | タイトル・問題の作り方・途中参加可否などを設定 |
| `screen-host-lobby` | ホスト：ロビー | 参加者を待つ画面 |
| `screen-countdown` | カウントダウン | スタート直後の「まもなく始まります」 |
| `screen-intro` | 「第N問」表示 | 毎問の直前に一瞬映る |
| `screen-host-play` | ホスト：出題中/結果発表 | ホストも参加者として回答する |
| `screen-player-play` | プレイヤー：出題中 | 参加者用の出題画面 |
| `screen-result` | 結果発表 | 最終順位（ホスト・参加者共通） |
| `screen-player-lobby` | プレイヤー：ロビー | 開始待ち |

`qz-confirm-overlay`は共通の確認モーダル（3節）で、これも常にHTML内に用意されています。

このドキュメントでは、起動処理と「参加する」画面（1〜394行）を解説します。ホストのセットアップ画面の続きと実際の対戦進行は[[01_Quiz.js_その1_ホストのセットアップとCSV読み込み.md]]・[[02_Quiz.js_その2_対戦の進行とリアルタイム同期.md]]で扱います。

---

## 1. ログインと通信ヘルパー（7〜76行）

```js
(function () {
  const s = getLoginSession();
  if (!s || !s.session_token) {
    sessionStorage.setItem('post_login_redirect', location.href);
    location.replace(LOGIN_PATH);
  }
})();
```
- [[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md]]と同じ、開いた瞬間の強制ログインチェックです。

```js
async function apiGet(path, params = {}, timeoutMs = 8000) {
  const { session_token, ...rest } = params;
  const qs = new URLSearchParams(rest).toString();
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    const res = await fetch(`${API_BASE}${path}?${qs}`, {
      signal: controller.signal, cache: 'no-store',
      headers: session_token ? { 'Authorization': 'Bearer ' + session_token } : {},
    });
    return await res.json();
  } catch (e) { return { ok: false, error: 'network' }; }
  finally { clearTimeout(timer); }
}
```
- `const { session_token, ...rest } = params;`という書き方（分割代入＋残りをまとめる`...rest`）で、渡された`params`から`session_token`だけを取り出し、それ以外（`rest`）をURLのクエリパラメータにします。コメントの通り「`session_token`はURLクエリに載せない」ためで、代わりに`Authorization`ヘッダーとして送ります。
- 通信が失敗した場合、`throw`で例外を投げるのではなく`{ ok: false, error: 'network' }`という**通常の戻り値の形**で返しています。これにより、呼び出し側は`try/catch`を書かなくても、いつも同じ`if (!data.ok)`というパターンでエラーを扱えます。

```js
function withAuth(extra = {}) {
  return { guild_id: GUILD_ID, session_token: STUDENT.sessionToken, ...extra };
}
```
- 通信のたびに`guild_id`と`session_token`を書く手間を省く、小さなヘルパーです。`{ ...extra }`で追加のパラメータを合体させます。

```js
function showConfirm({ title, desc = '', okLabel = 'OK', cancelLabel = 'キャンセル' }) {
  return new Promise(resolve => {
    const overlay = document.getElementById('qz-confirm-overlay');
    ...
    function finish(v) { overlay.classList.remove('active'); okBtn.onclick = null; cancelBtn.onclick = null; resolve(v); }
    okBtn.onclick = () => finish(true);
    cancelBtn.onclick = () => finish(false);
  });
}
```
- コメントに「`Cardmaker.js`の`showCmConfirm`と同じ考え方の簡易版」とあります。ただし、[[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md]]の`showCmConfirm`のように毎回新しい要素を作るのではなく、**HTMLに最初から用意されている1つのモーダル要素**（`qz-confirm-overlay`）の中身を毎回書き換えて使い回す、少し軽量な作り方です。

---

## 2. グローバル状態と時計のズレ補正（106〜136行）

```js
let quizClockOffsetMs = 0;
function applyServerNow(data) {
  if (data && typeof data.server_now === 'number') {
    quizClockOffsetMs = data.server_now - Date.now();
  }
}
```
- コメントに理由が詳しく書かれています：「タイマーバーは`question_started_at`（サーバー時計での開始時刻）を、この端末の`Date.now()`と直接比較して残り時間を計算しているため、端末の時計がサーバーとズレていると（特にスマホでよくある）進み方がホストと食い違って見えていた」。[[../04_StudyLog/04_StudyLog.js_その4_タイマー機能.md]]の`applyServerTimerState`で見たのと同じ「時計のズレ補正」の考え方を、ここでも`quizClockOffsetMs`という1つの数値としてシンプルに扱っています。サーバーからの応答が届くたびにこの関数を呼んで補正値を更新し直します。

```js
let launchDeckInfo = null;
let hsSelectedDecks = [];
let hsDeckPickerLocked = false;
```
- `launchDeckInfo`：CardMakerのデッキメニューから「みんなでクイズを始める」で直接来た場合の、単発デッキ情報。
- `hsSelectedDecks`：「デッキから自動作成」で選ばれているデッキ一覧。コメントに「`Cardmaker.html`（いつもの単語のホーム画面）を`?pick=quiz`で開いて選んでもらい、`sessionStorage`経由でここへ受け取る」とあります。これは[[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md]]で解説した「クイズ用デッキ選択モード（pickMode）」の受け手側の実装です。

---

## 3. 起動処理：`initQuizApp()`（141〜177行）

```js
const params = new URLSearchParams(location.search);
const mode = params.get('mode');
const codeParam = (params.get('code') || '').toUpperCase();
document.getElementById('home-account').textContent = STUDENT.nickname ? `${STUDENT.nickname} さん` : '';

if (mode === 'host') {
  const deckParam = params.get('deck');
  if (deckParam) {
    launchDeckInfo = { filename: deckParam, name: params.get('name') ? decodeURIComponent(params.get('name')) : '' };
  } else if (params.get('fromPicker') === '1') {
    try {
      const picked = JSON.parse(sessionStorage.getItem('quizDeckPicker') || '[]');
      if (Array.isArray(picked) && picked.length) hsSelectedDecks = picked;
    } catch {}
  }
  sessionStorage.removeItem('quizDeckPicker');
  history.replaceState(null, '', location.pathname);
  showScreenQ('home');
  goHostSetupScreen();
} else if (codeParam) {
  showScreenQ('home');
  joinRoomByCode(codeParam);
} else if (mode === 'join') {
  showScreenQ('home');
  goJoinScreen();
} else {
  showScreenQ('home');
}
```
- URLのパラメータを見て、このページがどんな状況で開かれたかを判定し、4通りに分岐します：
  1. `?mode=host&deck=...`：CardMakerのデッキメニューから、特定の1つのデッキで直接ホスト設定画面へ。
  2. `?mode=host&fromPicker=1`：[[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md]]のデッキ選択モードから「決定」して戻ってきた場合。`sessionStorage`に控えられていた選択結果を読み取ります。
  3. `?code=XXXXXX`：共有リンクなど、コード付きURLで直接開かれた場合。コメントに「参加者は普段コードを意識しなくてよいが、リンク共有自体は引き続き使える」とあります。
  4. `?mode=join`：参加画面へ直接。
  5. それ以外：普通にホーム画面。
- `history.replaceState(null, '', location.pathname)`：ホスト設定へ進む場合、URLからクエリパラメータを消しておきます（このあとの操作でリロードされても、また同じ処理が繰り返されないようにするため）。

---

## 4. 「参加する」画面（189〜285行）

```js
function goJoinScreen() {
  showScreenQ('join');
  loadRoomList();
  startRoomListPolling();
}
```
- コードを手入力させる代わりに、**参加できるルームの一覧から選ぶ**という設計です。

```js
async function loadRoomList() {
  const data = await apiGet('quiz_list_rooms', withAuth());
  ...
  const rooms = data.rooms || [];
  ...
  listEl.innerHTML = rooms.map(r => {
    const inProgress = r.state !== 'lobby';
    const joinable = r.state === 'lobby' || r.allow_late_join;
    const statusText = !inProgress ? '参加受付中' : (joinable ? `...プレイ中（第${r.current_q + 1}問）` : '...プレイ中・途中参加不可');
    const tag = joinable ? 'button' : 'div';
    const typeAttr = joinable ? ' type="button"' : '';
    const clickAttr = joinable ? ` onclick="joinRoomByCode('${r.code}')"` : '';
    ...
    return `<${tag} class="qz-room-row${disabledClass}"${typeAttr}${clickAttr}>...</${tag}>`;
  }).join('');
}
```
- 各ルームの「参加できるかどうか」（`joinable`）に応じて、**HTMLのタグ自体を`<button>`か`<div>`かで出し分けています**。参加できないルーム（開始済み・途中参加不可）は、押せない`<div>`として表示だけされ、タップしても何も起きません。テンプレート文字列の中で`${tag}`を開始タグと終了タグの両方に埋め込むことで、1つの`return`文で両方のパターンをカバーしています。

```js
function startRoomListPolling() {
  stopRoomListPolling();
  roomListHandle = setInterval(loadRoomList, 3000);
}
function stopRoomListPolling() {
  if (roomListHandle) clearInterval(roomListHandle);
  roomListHandle = null;
}
```
- コメントに「参加できるルームは一覧表示中に増えたり（新規作成）消えたり（開始・終了）するため、一覧画面を見ている間だけ定期的に取り直す（他の画面に移ったら止める）」とあります。3秒ごとという、他ページの10秒間隔ポーリングよりも短い間隔になっているのは、「今まさに始まりそうなクイズを見逃さない」ことを重視した設定と考えられます。この一覧のためだけにSSEではなく単純なポーリングを使っているのは、対戦中の細かいリアルタイム性（次節以降で扱う`startQuizRealtime`）とは要求される精度が違うためかもしれません。

```js
async function joinRoomByCode(code) {
  code = (code || '').trim().toUpperCase();
  if (!code) return;
  stopRoomListPolling();
  const data = await apiPost('quiz_join', withAuth({ code }));
  if (!data.ok) {
    await showConfirm({ title: '参加できませんでした', desc: quizErrorText(data.error), okLabel: 'OK', cancelLabel: '閉じる' });
    showScreenQ('join');
    loadRoomList();
    startRoomListPolling();
    return;
  }
  roomCode = code;
  isHost = !!data.is_host;
  renderedQIndex = -1; renderedState = null;
  applyServerNow(data);
  history.replaceState(null, '', `${location.pathname}?code=${code}`);
  renderRoom(data.room);
  startPolling();
}
```
- 参加ルームのコードは大文字に統一（`.toUpperCase()`）してから送信しています。
- 失敗したら、確認ダイアログでエラー内容を伝え、一覧画面に戻ってポーリングを再開します（コメントに「共有リンクからの直接参加で失敗した場合の受け皿にもなる」とあり、3節で見た「コード付きURLで直接開かれた」ケースの失敗時にも、この処理が自然に一覧画面へ誘導してくれます）。
- 成功したら、URLに`?code=XXXXXX`を付けておきます。これにより、ページを再読み込みしても同じルームへの参加を試みることができます。

```js
function quizErrorText(code) {
  return {
    room_not_found: 'そのコードのクイズは見つかりませんでした',
    quiz_already_started: 'このクイズはもう始まっています',
    not_logged_in: 'ログインが切れています。ログインし直してください',
    network: '通信に失敗しました。もう一度お試しください',
    deck_not_found: '選んだデッキが見つかりませんでした。デッキを選び直してください',
    deck_too_small: '選んだデッキ（合計）に、答えの種類が4つ以上ありません。デッキを追加するか選び直してください',
    too_many_decks: '一度に選べるデッキの数が多すぎます。選ぶデッキを減らしてください',
  }[code] || (code ? `エラー: ${code}` : '不明なエラーが発生しました');
}
```
- サーバーが返してくるエラーコード（英語の識別子）を、日本語の分かりやすい文言に変換する対応表です。`{...}[code]`という書き方は、オブジェクトを「その場で作って、すぐに`[code]`でプロパティを取り出す」というテクニックで、対応表を変数に入れずに1回だけ使い捨てる場合によく使われます。`|| (...)`で、対応表に無いコードが来た場合のフォールバック文言も用意されています。

---

続きは[[01_Quiz.js_その1_ホストのセットアップとCSV読み込み.md]]で、ホストのクイズ作成画面（デッキ選択・手動作成・CSV読み込み・ルーム作成）を解説します。
