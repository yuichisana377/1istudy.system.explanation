# StudyLog.js その4：勉強タイマー（複数端末同期）（1203〜1640行）

[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)の続きです。ここは`StudyLog.js`の中でも特に作り込まれた機能で、「同じアカウントで複数の端末を使っても、タイマーの状態が正しく同期される」ことを目指しています。

---

## 1. 設計の背景（1221〜1234行のコメントより）

「以前はタイマーの開始時刻をブラウザの`localStorage`だけに保存していたため、①別端末・別ブラウザからは状態が見えず、二重に計測を始められる、②タブを閉じる／バックグラウンドに置くとJSが止まり、『3時間経過』の検知が遅れる（＝精度が低い。時には全く違う経過時間で『破棄』と誤判定されてしまう）という問題があった」。

これを解決するため、**開始・一時停止・再開の「時刻」はすべてサーバー側で管理**し、ブラウザ側は「今サーバーはどういう状態と言っているか」を定期的に問い合わせて、それを表示に反映するだけ、という役割分担に変更されています。3時間経過の自動休憩判定やDiscordへの通知も、ブラウザのタブが開いているかどうかに関係なく、サーバー側が正確に行います。

---

## 2. 経過時間の表示：`updateTimerUI()`（1206〜1219行）

```js
function pad(n) { return String(n).padStart(2, "0"); }
function updateTimerUI() {
  var t = Math.floor(timerSec);
  var h = Math.floor(t / 3600);
  var m = Math.floor((t % 3600) / 60);
  var s = t % 60;
  document.getElementById("timer-display").textContent = pad(h)+":"+pad(m)+":"+pad(s);
  var hint = document.getElementById("timer-pts-hint");
  if (timerRunning && !timerIsPaused) {
    var remaining = (lastAwardedMin + 5) * 60 - timerSec;
    hint.textContent = remaining > 0 ? "次の +1pt まで " + remaining + "秒" : "";
  } else { hint.textContent = ""; }
}
```
- `timerSec`（秒数）を時・分・秒に分解して`00:00:00`形式で表示します。
- 「次の+1ptまであと〇秒」というヒント表示は、`lastAwardedMin`（最後にポイントを付与した分数の区切り）を基準に、次の5分区切りまでの残り秒数を逆算しています。

---

## 3. サーバーとの通信（1235〜1267行）

```js
async function timerApiState() {
  try {
    // ★ session_tokenはURLクエリに載せない（ブラウザ履歴・アクセスログ・Refererに
    //   残るリスクがあるため）。Authorizationヘッダで送る。
    return await api("/timer_state?guild_id=" + GUILD_ID, {
      headers: { "Authorization": "Bearer " + SESSION_TOKEN },
    });
  } catch(e) { return null; }
}
async function timerApiStart() {
  try {
    return await api("/timer_start", { method: "POST",
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }) });
  } catch(e) { return null; }
}
async function timerApiPause() {
  try {
    return await api("/timer_pause", { method: "POST",
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }) });
  } catch(e) { return null; }
}
async function timerApiResume() {
  try {
    return await api("/timer_resume", { method: "POST",
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }) });
  } catch(e) { return null; }
}
async function timerApiStop() {
  try {
    return await api("/timer_stop", { method: "POST",
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }) });
  } catch(e) { return null; }
}
```
- タイマーの5つの状態変化（今の状態を聞く・開始・一時停止・再開・停止）に対応する、5つのAPI呼び出し関数です。すべて通信が失敗したら`null`を返すだけの、シンプルな薄いラッパーです。
- `timerApiState`だけ`headers`を明示的に指定していますが、これは`GET`リクエストなので`api()`の第2引数（`opts`）が持つデフォルトの`Authorization`付与に頼らず、念のため明示している形です（コメントには「`session_token`はURLクエリに載せない」という、[../02_Cardmaker/06_Cardmaker.js_その6_カード編集と学習データ同期.md](../02_Cardmaker/06_Cardmaker.js_その6_カード編集と学習データ同期.md)でも見た同じ理由が書かれています）。

---

## 4. 経過時間の計算とサーバー状態の反映（1269〜1295行）

```js
function computeElapsedSec() {
  if (timerRunning && runStartClientEpoch != null) {
    return accumulatedSec + Math.floor((Date.now() - runStartClientEpoch) / 1000);
  }
  return accumulatedSec;
}
```
- 「これまでに確定している累計秒（`accumulatedSec`）」＋「今まさに計測中の区間の経過秒（今の時刻 − 区間の開始時刻）」という2つを足したものが、表示すべき経過時間です。計測が止まっている（一時停止中や停止後）なら、確定済みの累計秒だけを返します。

```js
function applyServerTimerState(resp) {
  var clockOffset = (resp.server_now != null) ? (resp.server_now - Date.now()) : 0;
  accumulatedSec  = Math.round(resp.accumulated_sec || 0);
  runStartClientEpoch = (resp.state === "running" && resp.run_start_epoch != null)
    ? (resp.run_start_epoch - clockOffset)
    : null;
  timerRunning  = (resp.state === "running");
  timerIsPaused = (resp.state === "paused");
  pauseReason   = resp.pause_reason || null;
  timerSec = computeElapsedSec();
  lastAwardedMin = Math.floor(timerSec / 60 / 5) * 5;
}
```
- サーバーから届いた状態を、この端末のローカル変数に反映する中心的な関数です。ここで注目すべきは「時計のズレの補正」です：
  ```js
  var clockOffset = (resp.server_now != null) ? (resp.server_now - Date.now()) : 0;
  runStartClientEpoch = ... (resp.run_start_epoch - clockOffset)
  ```
  サーバーが「計測開始時刻はこの時刻でした」と教えてくれても、**サーバーの時計とこの端末の時計がぴったり合っている保証はありません**。そこでまず、サーバーが一緒に送ってくれる「今のサーバー時刻（`server_now`）」と、この端末の「今の時刻（`Date.now()`）」の差（`clockOffset`）を計算し、サーバーの開始時刻からこの差を引くことで、**「この端末の時計で見たときの、正しい開始時刻」**に変換しています。コメントにも「端末の時計が多少ズレていても表示上の経過時間は正確になる」とある通り、これにより、スマホの時計が数秒ズレていても、表示される経過時間そのものは正確に保たれます。
- `lastAwardedMin = Math.floor(timerSec / 60 / 5) * 5;`：サーバーから最新の状態を取得した際、「本来もう付与されているはずのポイント区切り」を計算し直しています。コメントに「離れていた間の5分区切りポイントを二重付与しないよう」とあります。もしこれをしないと、他の端末で進んでいた分のポイントを、こちらの端末がまた最初から数え直して二重に加算してしまう危険があります。

---

## 5. 定期同期：`startSyncPolling()`／`syncTimerFromServer()`（1297〜1346行）

```js
function startSyncPolling() {
  if (timerSyncInterval) clearInterval(timerSyncInterval);
  timerSyncInterval = setInterval(syncTimerFromServer, 20000);
}
```
- 20秒ごとに、サーバーの「本当の」状態と同期します。コメントに「他端末での一時停止／再開、3時間ごとの自動休憩などを検知するため」とあります。

```js
async function syncTimerFromServer() {
  var res = await timerApiState();
  if (!res || !res.ok) return;

  if (res.state === "idle") {
    // 保存・破棄が別端末で済んだ
    clearInterval(timerInterval);     timerInterval     = null;
    clearInterval(timerSyncInterval); timerSyncInterval = null;
    var onTimerScreen =
      document.getElementById("timer-main").style.display === "block" ||
      document.getElementById("timer-confirm").style.display === "block";
    if (onTimerScreen) timerReset();
    return;
  }

  if (res.state === "paused" && timerRunning) {
    // ★ 3時間経過による自動休憩、または他端末での一時停止
    clearInterval(timerInterval); timerInterval = null;
    applyServerTimerState(res);
    updateTimerUI();
    document.getElementById("btn-pause").textContent = "▶ 再開";
    if (pauseReason === "checkpoint") {
      document.getElementById("timer-status").textContent = "休憩中...（3時間経過したため自動的に休憩にしました。再開すると続きから計測できます）";
      notifyUserBrowserOnly("StudyLog", "3時間が経過したため、自動的に休憩（一時停止）にしました。「再開」から続きを計測できます。");
    } else {
      document.getElementById("timer-status").textContent = "休憩中...（他の端末で一時停止されました）";
    }
    startSyncPolling();
    return;
  }

  if (res.state === "running" && timerIsPaused) {
    applyServerTimerState(res);
    document.getElementById("btn-pause").textContent    = "⏸ 休憩";
    document.getElementById("timer-status").textContent = "計測中...（他の端末で再開されました）";
    startInterval();
    return;
  }

  if (res.state === "running") {
    applyServerTimerState(res); // ズレ補正
  }
}
```
`syncTimerFromServer()`（1304〜1346行）は、サーバーから返ってきた状態と、この端末が今思っている状態を比較し、**ズレていたら追従する**処理です：

- **`res.state === "idle"`**（サーバー側ではもうタイマーが動いていない）：これは、**別の端末で既に保存または破棄が行われた**ことを意味します。今この端末がまだタイマー画面（計測中 or 確認画面）を表示していれば、`timerReset()`（8節）で表示をリセットします。
- **`res.state === "paused" && timerRunning`**（サーバーは休憩中と言っているのに、この端末はまだ計測中だと思っている）：3時間経過による自動休憩、または他端末での一時停止を検知したケースです。`pauseReason === "checkpoint"`（3時間経過による自動休憩）の場合は`notifyUserBrowserOnly`（6節）でブラウザ通知も出します。
- **`res.state === "running" && timerIsPaused`**（サーバーは計測中と言っているのに、この端末はまだ休憩中だと思っている）：他端末で再開されたケースです。
- **`res.state === "running"`（それ以外、両方running）**：状態自体は変わっていませんが、時計のズレなどを補正するため、`applyServerTimerState(res)`を呼んで数値をサーバー基準に合わせ直します（コメントに「ズレ補正」とあります）。

---

## 6. 表示更新ループとブラウザ通知（1348〜1413行）

```js
function startInterval() {
  if (timerInterval) clearInterval(timerInterval);
  lastCheckpointMin = Math.floor(timerSec / 60 / 180) * 180;
  timerInterval = setInterval(function() {
    timerSec = computeElapsedSec();
    var curMin = Math.floor(timerSec / 60);
    if (curMin > 0 && curMin % 180 === 0 && curMin > lastCheckpointMin) {
      lastCheckpointMin = curMin;
      syncTimerFromServer();
    }
    if (curMin > 0 && curMin % 5 === 0 && curMin > lastAwardedMin) {
      lastAwardedMin = curMin;
      myPoints++;
      allPoints[STUDENT.id] = (allPoints[STUDENT.id] || 0) + 1;
      floatPoints("+1pt");
      updatePointDisplay();
    }
    updateTimerUI();
  }, 500);
  startSyncPolling();
}
```
- 0.5秒ごとに画面表示を更新するループです。0.5秒という短い間隔にしているのは、秒単位の表示をなめらかに更新するためだと考えられます。
- **3時間（180分）区切りに到達したら**、20秒ごとの通常の同期を待たず、その場ですぐ`syncTimerFromServer()`を呼びます。コメントには「タブが開いている間は最大20秒のポーリング待ちを避け、素早く休憩へ切り替わったことを反映できるようにする。実際の判定・DM通知はタブの状態に関わらずサーバー側が正確に行う」とあります。つまり、この即時同期はあくまで**見た目の反応を速くするための工夫**で、実際に「3時間経過したから休憩にする」という判断自体は、必ずサーバー側（`bot.py`のバックグラウンド処理）が行っています。
- **5分区切りに到達したら**、その場でポイントを1つ加算する見た目の演出（`floatPoints`）を出します。この時点ではまだサーバーに「ポイントをください」と申請しているわけではなく、あくまで**見た目上の先行加算**です（実際のポイント確定は、記録を保存するとき＝`postLog`のタイミングで行われます）。

```js
function ensureNotifyPermission() {
  if (typeof Notification === "undefined") return;
  if (Notification.permission === "default") { Notification.requestPermission(); }
}
function notifyUser(title, body) {
  api("notify_dm", { method: "POST", body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN, title: title, message: body }) })
    .then(function(res) { if (!res || !res.ok) { notifyUserBrowserOnly(title, body); } })
    .catch(function(err) { notifyUserBrowserOnly(title, body); });
}
function notifyUserBrowserOnly(title, body) {
  if (typeof Notification !== "undefined" && Notification.permission === "granted") {
    try { new Notification(title, { body: body }); return; } catch(e) {}
  }
  showAppAlert({ title: title, desc: body });
}
```
- `notifyUser`は、まずDiscordのDM通知（Discord連携済みの場合のみ機能する、サーバー側の`/notify_dm`）を試み、失敗・未連携なら`notifyUserBrowserOnly`（ブラウザ標準の通知機能、それも使えなければ自作の`showAppAlert`）にフォールバックする、3段階の通知経路です。コメントに「Discord連携済みなら本人のDiscordへ直接DM通知を送る。これはブラウザのタブを閉じていても、他のサイトを見ていても届く」とあります。
- `ensureNotifyPermission()`は、ブラウザの通知許可を求める`Notification.requestPermission()`を呼びますが、**ユーザーがタイマーを開始ボタンを押した瞬間**（7節の`timerStart`から呼ばれる）に実行されます。多くのブラウザは、ユーザーの明示的な操作をきっかけにしないと許可ダイアログを出さない・出しても信頼されにくいため、このタイミングで呼ぶのは理にかなっています。

---

## 7. 開始・一時停止/再開・停止（1415〜1520行）

### 7.1 `timerStart()`（1415〜1453行）
```js
async function timerStart() {
  if (timerRunning || timerIsPaused) return;
  ensureNotifyPermission(); // ★ ユーザー操作（クリック）のタイミングで許可をリクエスト
  var btn = document.getElementById("btn-start");
  if (btn) btn.disabled = true; // ★ 連打防止（応答が返るまで）

  var res = await timerApiStart();

  if (!res || res.ok === false) {
    if (res && res.error === "not_logged_in") { forceReLogin(); return; }
    if (res && res.error === "already_paused") {
      // ★ 他の端末で既に一時停止中（3時間経過での自動休憩を含む） → こちらもその状態に合わせる
      applyServerTimerState(res);
      document.getElementById("btn-start").disabled = true;
      document.getElementById("btn-pause").disabled  = false;
      document.getElementById("btn-stop").disabled   = false;
      document.getElementById("btn-pause").textContent    = "▶ 再開";
      document.getElementById("timer-status").textContent =
        res.pause_reason === "checkpoint"
          ? "休憩中...（3時間経過したため自動的に休憩にしました。再開すると続きから計測できます）"
          : "休憩中...（他の端末で開始された記録に接続しました）";
      updateTimerUI();
      startSyncPolling();
      return;
    }
    if (btn) btn.disabled = false;
    showAppAlert({ title: "通信エラーのためタイマーを開始できませんでした", desc: "もう一度お試しください。" });
    return;
  }

  applyServerTimerState(res);
  document.getElementById("btn-start").disabled = true;
  document.getElementById("btn-pause").disabled = false;
  document.getElementById("btn-stop").disabled  = false;
  document.getElementById("btn-pause").textContent = "⏸ 休憩";
  document.getElementById("timer-status").textContent =
    res.joined ? "計測中...（他の端末で開始された記録に接続しました）" : "計測中...";
  startInterval();
}
```
- `res.error === "already_paused"`という特別なケースが用意されています。これは、実は別の端末で既にタイマーが動いていて（あるいは3時間経過で自動休憩中で）、この端末から「開始」しようとしても、サーバー側は「新しく始める」のではなく「既存の記録に接続する」動きをする、ということを意味しています。`res.joined`（新規開始ではなく、既存の記録に合流した）というフラグも用意されており、表示メッセージをそれに応じて変えています。

### 7.2 `timerPauseResume()`（1455〜1492行）
今の状態（`timerIsPaused`）に応じて、休憩にするか再開するかを切り替えます。休憩にする処理でサーバーとの同期に失敗した場合、
```js
async function timerPauseResume() {
  if (!timerRunning && !timerIsPaused) return;
  var btn = document.getElementById("btn-pause");
  if (btn) btn.disabled = true; // ★ 連打防止（応答が返るまで）

  if (!timerIsPaused) {
    // 休憩する
    clearInterval(timerInterval); timerInterval = null;
    var res = await timerApiPause();
    if (!res || res.ok === false) {
      if (res && res.error === "not_logged_in") { forceReLogin(); return; }
      // サーバーとの同期に失敗した場合は最新状態を取り直して表示だけ合わせる
      var st = await timerApiState();
      if (st && st.ok) applyServerTimerState(st);
      startInterval();
      if (btn) btn.disabled = false;
      return;
    }
    applyServerTimerState(res);
    document.getElementById("btn-pause").textContent      = "▶ 再開";
    document.getElementById("timer-status").textContent   = "休憩中...";
    document.getElementById("timer-pts-hint").textContent = "";
  } else {
    // 再開する
    var res2 = await timerApiResume();
    if (!res2 || res2.ok === false) {
      if (res2 && res2.error === "not_logged_in") { forceReLogin(); return; }
      showAppAlert({ title: "通信エラーのため再開できませんでした", desc: "もう一度お試しください。" });
      if (btn) btn.disabled = false;
      return;
    }
    applyServerTimerState(res2);
    document.getElementById("btn-pause").textContent    = "⏸ 休憩";
    document.getElementById("timer-status").textContent = "計測中...";
    startInterval();
  }
  if (btn) btn.disabled = false;
}
```
- 単にエラーを表示するだけでなく、**改めてサーバーに今の本当の状態を聞き直して**表示を合わせ直す、という丁寧な回復処理が入っています。

### 7.3 `timerStop()`（1494〜1520行）
```js
async function timerStop() {
  clearInterval(timerInterval); timerInterval = null;
  var wasRunning = timerRunning;
  timerRunning = false; timerIsPaused = true;

  // ★ ここではまだ保存/破棄が決まっていないため、サーバー側の記録は
  //   すぐに消さず「一時停止」扱いで経過時間だけ確定させておく。
  //   これにより他の端末には「一時停止しました」と正しく伝わり、
  //   計測中の表示が理由もなく途中でリセットされるのを防げる。
  //   （実際にサーバー側の記録を消すのは、保存 or 破棄が確定した時）
  if (wasRunning) {
    var res = await timerApiPause();
    if (res && res.ok) applyServerTimerState(res);
  }
  startSyncPolling(); // ★ 確認画面を見ている間も、他端末での保存/破棄を検知できるようにする

  // ★ 2026/09/05 追記：CardMakerでのプレイ開始がこのタイマーを自動的に
  //   始めていた場合、そのメモを引き継ぐ（後述consumePendingTimerMemo）。
  var pendingMemo = consumePendingTimerMemo();

  var mins = Math.floor(timerSec / 60);
  if (mins < 1) {
    await showAppAlert({ title: "1分未満のため記録できません" });
    timerApiStop(); // 記録として残す価値が無いので、ここでサーバー側もクリアする
    timerReset(); return;
  }
  document.getElementById("timer-main").style.display    = "none";
  document.getElementById("timer-confirm").style.display = "block";
  document.getElementById("conf-time").textContent       = mins + "分 " + pad(timerSec % 60) + "秒";
  document.getElementById("conf-time").dataset.min       = mins;
  if (pendingMemo) document.getElementById("conf-memo").value = pendingMemo;
}
```
- コメントに重要な設計判断が書かれています：「ここではまだ保存/破棄が決まっていないため、サーバー側の記録はすぐに消さず『一時停止』扱いで経過時間だけ確定させておく。これにより他の端末には『一時停止しました』と正しく伝わり、計測中の表示が理由もなく途中でリセットされるのを防げる。実際にサーバー側の記録を消すのは、保存または破棄が確定した時」。「ストップ」ボタンを押した直後は、確認画面（保存するか、破棄するか、まだ迷える状態）に過ぎないため、サーバー側のデータもまだ完全には消さず、あくまで「一時停止」という中間状態にとどめています。
- 1分未満の計測は、記録として残す価値が無いと判断され、その場で自動的に破棄されます。

### 7.4 ★ 2026/09/05 追記：CardMakerからの自動計測・自動記録の連携

「CardMakerをプレイした場合、勉強ログのタイマーが起動していなければ自動で計測。メモ欄には、CardMaker「〇〇」をプレイ　にする」「途中終了だとカウントされ続けちゃう。最後まで行くまたは途中で終了したらタイマー止めて自動記録」「科目もそのデッキの科目に合わせて」という3段階の要望を受けて追加した連携。CardMakerとStudyLogは別ページ（`Cardmaker.html`/`StudyLog.html`）だが、同一オリジンなので`localStorage`を共有できることを利用している。

**開始（CardMaker側、`maybeAutoStartStudyTimer(memo, subject)`。`startStudyMode()`が学習画面を開くたびに呼ぶ）**：
1. `GET /timer_state`でタイマーの現在状態を確認する。
2. `"idle"`（誰も計測していない）のときだけ`POST /timer_start`でこの本人の計測を開始する。既に`"running"`／`"paused"`（本人が既に何かを計測中）なら何もしない＝横取りしない。
3. 開始できたら、`localStorage`のキー`"cm_pending_timer_memo"`に`{ memo: "CardMaker「◯◯」をプレイ", subject: deck.subject, ts: Date.now() }`を保存しておく。◯◯はプレイ中のデッキ名／フォルダ名（`studyBaseTitle`）、`subject`はプレイ中のデッキが持つ科目（フォルダをまとめてプレイ中は科目が1つに定まらないため`null`）。CardMakerの科目プルダウン（`modal-rename-subject`）とStudyLogの科目プルダウン（`SUBJECTS`）はどちらも同じ`/channels`（Discordのチャンネル名一覧）から作られているため、文字列がそのまま一致する。
4. このキーは「まだ誰も（CardMaker自身も含めて）このタイマーを片付けていない」ことの目印としても使われる（後述）。
5. 全体を`try/catch`で囲み、ログインしていない・通信に失敗した場合は何もしない（プレイ自体は絶対に止めないベストエフォート）。

**終了（CardMaker側、`finishAutoTimerIfPending()`。以下2箇所から呼ばれる）**：
- **最後まで完走**：`renderStudyCard()`が`studyIdx >= studyCards.length`（完了画面）に達した瞬間。
- **途中で画面を離れた**：`showScreen(id)`が、直前まで`screen-study`がアクティブで、かつ`id !== 'study'`（＝学習画面を離れる）ときに呼ぶ。カードの編集モーダルを開くだけ（画面遷移を伴わない）や「もう一度」（`shuffleStudy()`、同じ`screen-study`のまま）では呼ばれない。

処理内容：
1. `cm_pending_timer_memo`が無ければ何もしない（このプレイでは自動開始していない＝既に本人が別画面で片付け済み、を含む）。
2. `GET /timer_state`で今も`"running"`か確認する。`"running"`でなければ（既に誰かが止めた等）目印だけ消して終了。
3. `POST /timer_pause`で一時停止し、確定した`accumulated_sec`を受け取る。1分未満なら`POST /timer_stop`で破棄する（`saveTimer`/`timerStop`の「1分未満は記録できません」と同じ基準）。
4. `pending.subject`が無ければ**自動保存できない**（`/add_study_log`は科目必須のため）。この場合は`timer_pause`で確定済みの「一時停止」状態のままサーバーに残し、目印（`cm_pending_timer_memo`）も消さずに終える。データは失われず、次に本人がStudyLogを開いたときに`timerStop()`→`consumePendingTimerMemo()`の通常経路でメモ（と、分かっていれば科目）が引き継がれて手動で保存できる。
5. `pending.subject`があれば`POST /add_study_log`（`method: "timer"`、`date`は`new Date().toISOString().slice(0,10)`＝`todayStr()`と同じ形式）でそのまま記録し、成功したら`POST /timer_stop`でサーバー側のタイマーを片付け、目印も消す。保存自体が失敗した場合（例：サーバー側の不正防止クールダウンに引っかかった等）は何もしない＝一時停止のまま残し、後で手動対応できるようにする。
6. 全体を`try/catch`で囲んだベストエフォート。

**引き継ぎ（StudyLog側、`consumePendingTimerMemo()`）**：
```js
function consumePendingTimerMemo() {
  var KEY = "cm_pending_timer_memo";
  try {
    var raw = localStorage.getItem(KEY);
    if (!raw) return null;
    localStorage.removeItem(KEY);
    var data = JSON.parse(raw);
    if (!data || !data.memo || !data.ts) return null;
    var ONE_DAY_MS = 24 * 60 * 60 * 1000;
    if (Date.now() - data.ts > ONE_DAY_MS) return null;
    return { memo: data.memo, subject: data.subject || null };
  } catch (e) { return null; }
}
```
- `timerStop()`（確認画面を出す直前）で必ず1回呼ぶ。保存に至るか（`mins >= 1`）1分未満で自動破棄されるかに関わらず、**停止のたびに必ず消費**する（`localStorage.removeItem`を先に呼んでから中身を見ている点に注目）。これにより、次に始まる無関係なタイマー（例：後日、本人がCardMakerを経由せず手動で開始したもの）に古いメモが紛れ込むことを防ぐ。
- `ts`（開始時刻）が1日以上前なら`null`を返して無視する。CardMaker側で開始だけして何日も停止せずに放置していたような場合、久しぶりに停止したときに古い（もう文脈を忘れているであろう）メモが誤って復元されるのを避けるための保険。
- 戻り値がある場合、`timerStop()`が`conf-memo`欄に`memo`を代入する。`subject`があれば、`conf-subject`の`<option>`に一致するものがある場合だけ選択する（`SUBJECTS`に無い値なら既定のまま＝選択肢自体が存在しないため）。実運用では、この経路（StudyLog側での引き継ぎ）が使われるのは主に「CardMaker側で科目が分からず自動保存できなかった」ケースで、科目が分かっている場合は通常`finishAutoTimerIfPending()`が先に保存・後片付けまで終えている。

---

## 8. 保存・修正・破棄・リセット（1522〜1609行）

```js
async function saveTimer() {
  var sub  = document.getElementById("conf-subject").value;
  var memo = document.getElementById("conf-memo").value.trim();
  var mins = parseInt(document.getElementById("conf-time").dataset.min);
  var btnEl    = document.querySelector('button[onclick*="saveTimer"]');
  var editBtn  = document.querySelector('button[onclick*="editTimer"]');
  var discBtn  = document.querySelector('button[onclick*="discardTimer"]');

  if (btnEl && btnEl.disabled) return; // ★ 連打防止：送信中は何もしない

  // ★ 前回のタイマー記録から「今回記録しようとしている分数」以上の
  //    実時間が経過していない場合は保存させない（誤操作・二重送信防止）
  var last = getTimerLastLog();
  if (last && last.at) {
    var elapsedMs  = Date.now() - last.at;
    var requiredMs = mins * 60 * 1000;
    if (elapsedMs < requiredMs) {
      var remainMin = Math.ceil((requiredMs - elapsedMs) / 60000);
      await showAppAlert({
        title: "この記録は保存できません",
        desc: "前回の記録からまだ十分な時間が経過していないため、あと約" + remainMin + "分待つ必要があります。",
      });
      return;
    }
  }

  setButtonLoading(btnEl, true, "保存中…");
  if (editBtn) editBtn.disabled = true;
  if (discBtn) discBtn.disabled = true;

  var result = await postLog({ date: todayStr(), subject: sub, minutes: mins, memo: memo,
            student_id: STUDENT.id, nickname: STUDENT.nickname, method: "timer" });

  if (!result.ok) {
    // ★ サーバー側の不正防止チェックで拒否された場合など。
    //    確認画面はそのまま残し、ユーザーがもう一度試せるようにする。
    setButtonLoading(btnEl, false);
    if (editBtn) editBtn.disabled = false;
    if (discBtn) discBtn.disabled = false;
    showAppAlert({ title: "保存に失敗しました", desc: result.error || "" });
    return;
  }

  setTimerLastLog(mins); // ★ 保存成功後に記録
  timerApiStop(); // ★ サーバー側のタイマー状態も後片付け（自動停止からの保存の場合など。結果は待たない）

  var okEl = document.getElementById("timer-ok");
  okEl.style.display = "block";
  setTimeout(function() {
    okEl.style.display = "none";
    setButtonLoading(btnEl, false);
    if (editBtn) editBtn.disabled = false;
    if (discBtn) discBtn.disabled = false;
    timerReset(); showTab("home");
  }, 1200);
}
```
`saveTimer()`（1522〜1577行）は、[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)の`saveManual`と似た構造で、こちらは「前回のタイマー記録から、**今回記録しようとしている分数以上**の実時間が経過していないと保存できない」というクールダウンチェックを行います。コメントに「タイマーの経過時間を改ざんして即座に長時間記録するのを防ぐ」とあります。もしこのチェックが無いと、悪意のある利用者がブラウザの開発者ツールなどで`timerSec`の値を直接書き換えて、実際には計測していない長時間の記録を送信できてしまう可能性があります（この端末側のチェックも「あくまで早めに気づかせるため」で、[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)の手入力と同様、最終的な防御はサーバー側にもあります）。
- 保存が成功したら、サーバー側のタイマー状態（7.3節で「一時停止」のまま残していたもの）を`timerApiStop()`で完全にクリアします。この呼び出しの結果は`await`せず、結果を待たずに次の処理（成功メッセージの表示など）に進んでいます（後片付けなので、ユーザーを待たせる必要が無いためです）。

```js
async function editTimer() {
  var el  = document.getElementById("conf-time");
  var cur = parseInt(el.dataset.min);
  var v   = await showAppPrompt({ title: "分数を修正", label: "分数を修正してください", value: String(cur), inputType: "number" });
  if (v && parseInt(v) >= 1) {
    el.dataset.min = parseInt(v); el.textContent = parseInt(v) + "分 00秒";
  }
}
```
`editTimer()`（1579〜1586行）は、確認画面で計測結果の分数を手動で修正する機能です。`showAppPrompt`（`Dialog.js`共通の入力ダイアログ）を使い、入力された値が1以上ならその場で表示を書き換えます。

```js
async function discardTimer() {
  const ok = await showAppConfirm({ title: "この計測結果を破棄しますか？", okLabel: "破棄する", danger: true });
  if (ok) {
    timerApiStop(); // ★ サーバー側のタイマー状態も後片付け（結果は待たない）
    timerReset(); showTab("home");
  }
}
```
`discardTimer()`（1587〜1593行）は確認のあと`timerApiStop()`でサーバー側の記録も消し、`timerReset()`で表示をリセットします。

```js
function timerReset() {
  clearInterval(timerInterval);     timerInterval     = null;
  clearInterval(timerSyncInterval); timerSyncInterval = null;
  timerSec = 0; timerRunning = false; timerIsPaused = false; pauseReason = null;
  accumulatedSec = 0; runStartClientEpoch = null; lastAwardedMin = 0; lastCheckpointMin = 0;
  document.getElementById("timer-display").textContent   = "00:00:00";
  document.getElementById("timer-status").textContent    = "準備完了";
  document.getElementById("timer-pts-hint").textContent  = "";
  document.getElementById("btn-start").disabled  = false;
  document.getElementById("btn-pause").disabled  = true;
  document.getElementById("btn-stop").disabled   = true;
  document.getElementById("btn-pause").textContent = "⏸ 休憩";
  document.getElementById("timer-main").style.display    = "block";
  document.getElementById("timer-confirm").style.display = "none";
  document.getElementById("conf-memo").value = "";
}
```
`timerReset()`（1594〜1609行）はタイマーに関するあらゆる状態変数とタイマー処理（`setInterval`）を止めて初期状態に戻す、後片付け用の関数です。

---

## 9. ページを開いた瞬間の復元：`restoreTimer()`（1611〜1640行）

```js
async function restoreTimer() {
  var res = await timerApiState();
  if (!res || !res.ok) return;

  if (res.state !== "running" && res.state !== "paused") return; // idle：何もしない

  applyServerTimerState(res);
  document.getElementById("btn-start").disabled = true;
  document.getElementById("btn-pause").disabled = false;
  document.getElementById("btn-stop").disabled  = false;
  updateTimerUI();

  if (res.state === "paused") {
    document.getElementById("btn-pause").textContent    = "▶ 再開";
    document.getElementById("timer-status").textContent =
      res.pause_reason === "checkpoint"
        ? "休憩中...（3時間経過したため自動的に休憩にしました。再開すると続きから計測できます）"
        : "休憩中...（離れていた間も保持されていました）";
    startSyncPolling();
  } else {
    document.getElementById("btn-pause").textContent    = "⏸ 休憩";
    document.getElementById("timer-status").textContent = "計測中...（離れていた間も継続していました）";
    startInterval();
  }
}
```
- ページを開いた瞬間（[01_StudyLog.js_その1_ログインとアカウント設定.md](01_StudyLog.js_その1_ログインとアカウント設定.md)の起動処理から呼ばれる）に、まずサーバーに「今タイマーはどうなっていますか」と尋ねます。もし何かの記録が計測中・休憩中であれば、その状態をそのまま復元して続きから表示します。コメントに「これにより『別端末で計測中/休憩中の記録』もそのまま復元でき、また3時間ごとの自動休憩もサーバー基準の正確な状態としてすぐに分かるようになる（以前の`localStorage`ベースの判定はタブが開いていない間はズレる／進まないことがあった）」とあります。この関数が、この節の冒頭で説明した設計方針の「入口」にあたる部分です。

---

続きは[05_StudyLog.js_その5_ドロワーとリアルタイム更新.md](05_StudyLog.js_その5_ドロワーとリアルタイム更新.md)で、ドロワー・実用ヘルパー関数・リアルタイム更新を解説します。
