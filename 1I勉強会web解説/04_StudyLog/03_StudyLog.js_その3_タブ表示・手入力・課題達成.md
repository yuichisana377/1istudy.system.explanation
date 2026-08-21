# StudyLog.js その3：タブ切り替え・手入力での記録・課題の達成（1023〜1202行）

[[02_StudyLog.js_その2_ランキングと記録の描画.md]]の続きです。

---

## 1. タブ切り替え（1026〜1031行）

```js
function showTab(name) {
  ["home","manual","timer","tasks"].forEach(function(t) {
    document.getElementById("tab-btn-" + t).classList.toggle("active", t === name);
    document.getElementById("tab-" + t).classList.toggle("active",     t === name);
  });
}
```
- 4つのタブ名の配列をループし、タブボタンと対応する画面の両方に、選ばれたタブと一致するものだけ`active`クラスを付けます。[[../01_index_予定管理.md]]の`switchPlanView`と同じ考え方の、シンプルなタブ切り替えです。

---

## 2. 手入力での記録：`saveManual()`（1036〜1106行）

```js
var btnEl = document.querySelector('button[onclick*="saveManual"]');
```
- ボタンの参照を`id`ではなく、「`onclick`属性の中に`"saveManual"`という文字列を含む`<button>`」という条件で探しています。少し珍しい探し方ですが、HTML側にこのボタン専用の`id`が振られていないため、代わりにこの方法で見つけていると考えられます。

```js
if (btnEl && btnEl.disabled) return; // ★ 連打防止：送信中は何もしない
if (!min || min < 1) { ...エラー表示... return; }
```
- 分数が未入力・1未満ならエラーを表示して中断します。

### 2.1 二重の連打防止チェック（1055〜1083行）
```js
// ★ 教科を問わず、本人の直近の手入力から MANUAL_COOLDOWN_MS 経つまで不可（連打防止）
var manualMap  = getManualLastLogMap();
var allTimes   = Object.keys(manualMap).map(function(k) { return manualMap[k]; });
var lastAnyAt  = allTimes.length ? Math.max.apply(null, allTimes) : null;
if (lastAnyAt) {
  var elapsedAnyMs = Date.now() - lastAnyAt;
  if (elapsedAnyMs < MANUAL_COOLDOWN_MS) { ...エラー表示... return; }
}
// ★ 同じ教科での連続手入力は、前回の記録から MANUAL_COOLDOWN_MS 経つまで不可
var lastAt = manualMap[sub];
if (lastAt) { ...同様のチェック... }
```
- [[01_StudyLog.js_その1_ログインとアカウント設定.md]]で紹介した`getManualLastLogMap()`（`localStorage`に保存された、教科ごとの最終記録時刻）を使い、**2段階**でクールダウンをチェックしています：①どの教科でもいいので、直前の手入力から20秒（`MANUAL_COOLDOWN_MS`）経っているか、②特に同じ教科での直前の記録から20秒経っているか。`Math.max.apply(null, allTimes)`は、配列の中の最大値を求める（`Math.max(...allTimes)`と同等の、やや古い書き方の）イディオムです。
- コメントに「サーバー側（`bot.py`の`add_study_log`）にも同じ判定があり、そちらが最終防衛。ここはあくまで早めにエラーを見せるためのUX用チェック」とあります。つまり、この`localStorage`ベースのチェックは**あくまで親切のため**で、本当の不正防止はサーバー側で行われています（クライアント側のチェックは、`localStorage`を消したりJSを操作したりすれば回避できてしまうため、それだけに頼るのは安全ではありません）。

### 2.2 実際の送信（1085〜1106行）
```js
setButtonLoading(btnEl, true, "保存中…");
var result = await postLog({ date: todayStr(), subject: sub, minutes: min, memo: memo, student_id: STUDENT.id, nickname: STUDENT.nickname, method: "manual" });
if (!result.ok) { ...エラー表示... return; }
setManualLastLog(sub); // ★ 保存成功後に記録
document.getElementById("m-minutes").value = "";
document.getElementById("m-memo").value    = "";
setButtonLoading(btnEl, false);
okEl.style.display = "block";
setTimeout(function() { okEl.style.display = "none"; showTab("home"); }, 1200);
```
- [[01_StudyLog.js_その1_ログインとアカウント設定.md]]の`postLog()`を呼び、成功した場合だけ`setManualLastLog(sub)`でクールダウン用の時刻を更新します（**送信が失敗した場合はクールダウンを記録しません**。そうしないと、失敗しているのに次の記録までさらに20秒待たされる、という理不尽な状態になってしまいます）。
- 成功メッセージを1.2秒表示してから、自動的にホームタブに切り替えます（記録した結果がすぐホーム画面のサマリー・ランキングに反映されているのを見せるための演出です）。

---

## 3. 課題の達成・取り消し：`toggleTask(id)`（1117〜1201行）

コメントに設計方針がまとめられています：「サーバーへの送信が成功したのを確認してからローカル状態を更新する（失敗時はローカルを変更しない＝ポーリングで勝手に戻る現象を防止）」「送信中は多重クリックを防ぐため`pendingTaskIds`でボタンを無効化」「達成済みをもう一度押すと`/uncomplete_task`を呼んで未達成に戻す」。

### 3.1 達成にする（1126〜1158行）
```js
var t   = TASKS_JSON.find(function(x) { return x.id === id; });
var pts = t ? t.points : DEFAULT_TASK_POINTS;
try {
  var data = await api("/complete_task", { method: "POST", body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN, task_id: id }) });
  if (!data || data.ok === false) {
    if (data && data.error === "not_logged_in") { forceReLogin(); return; }
    throw new Error("server rejected complete_task");
  }
  var entry = { id: id, date: todayStr(), points: pts, nickname: STUDENT.nickname };
  completedTasks.push(entry);
  if (!allCompletedTasks[STUDENT.id]) allCompletedTasks[STUDENT.id] = [];
  allCompletedTasks[STUDENT.id].push(entry);
  ...
  allPoints[STUDENT.id] = (data.total != null) ? data.total : (allPoints[STUDENT.id] || 0) + pts;
  myPoints = allPoints[STUDENT.id];
  updatePointDisplay();
  floatPoints("+" + pts + "pt");
} catch (e) {
  showAppAlert({ title: "通信エラーのため達成にできませんでした", desc: "もう一度お試しください。" });
}
```
- サーバーが成功を返した**あとで初めて**、ローカルの`completedTasks`（自分の達成一覧）・`allCompletedTasks`（全員分の達成一覧）・`allPoints`（累計ポイント）を更新します。ここでコメントの「サーバーへの送信が成功したのを確認してからローカル状態を更新する」という方針が実際に守られていることが分かります。これは[[../02_Cardmaker/*]]で見た「楽観的更新（先にローカルを変えて、失敗したら後で戻す）」とは逆の、**悲観的更新**（結果が分かってから初めて反映する）の考え方です。コメントの「失敗時はローカルを変更しない＝ポーリングで勝手に戻る現象を防止」という理由から、達成・未達成という「状態」を扱うこの機能では、先にローカルを変えてしまうと、もし失敗したときにポーリングで最新化された際に「一瞬達成したように見えたのに、また未達成に戻る」という混乱を招くバグにつながる、と判断されたのだと考えられます。

### 3.2 未達成に戻す（1159〜1196行）
考え方は同じで、`/uncomplete_task`という別のAPIを呼び、成功したらローカルの状態から取り除きます。

```js
var revertedTotal = (data2.total != null)
  ? data2.total
  : Math.max(0, (allPoints[STUDENT.id] || 0) - (removed ? removed.points : 0));
```
- サーバーが最新の合計ポイント（`data2.total`）を返してくれればそれをそのまま使い、万一返ってこなかった場合のフォールバックとして、クライアント側でも「今のポイントから、取り消した分のポイントを引く」計算をします。`Math.max(0, ...)`で、引き算の結果が万一マイナスになってもポイントが0未満にはならないよう安全策を入れています。

### 3.3 共通の後処理（1198〜1201行）
```js
pendingTaskIds.delete(id);
renderTasks();
renderAll();
```
- 成功・失敗どちらの場合でも、「送信中」フラグを解除し、課題一覧とホーム画面全体を再描画します。

---

続きは[[04_StudyLog.js_その4_タイマー機能.md]]で、勉強タイマー（開始・休憩・停止・複数端末同期）を解説します。
