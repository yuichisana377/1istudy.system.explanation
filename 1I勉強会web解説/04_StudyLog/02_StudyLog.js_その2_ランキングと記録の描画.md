# StudyLog.js その2：起動処理・週間ランキング・ホーム画面の描画（273〜1022行）

[01_StudyLog.js_その1_ログインとアカウント設定.md](01_StudyLog.js_その1_ログインとアカウント設定.md)の続きです。

---

## 1. 起動処理（273〜341行）

```js
window.addEventListener("load", function() {
  applySession();
  setTodayLabel();
  restoreTimer();
  initTaskListEvents();
  initMyLogsEvents();
  if (new URLSearchParams(location.search).get("openAccount")) openAccountModal();

  const logsPromise = loadLogs();
  logsPromise.then(renderLogs);

  Promise.all([
    logsPromise, loadUsers(), loadSubjects(), loadPoints(),
    loadCompletedTasks(), loadAllCompletedTasks(), loadTasks()
  ]).then(function() {
    renderSubjectDropdown();
    renderAll();
    renderTasks();
  });
  prefetchOtherPages();
});
```
- コメントに、体感速度を上げるための工夫が説明されています：「『ログ一覧』の表示が、ポイント・達成済み課題・全ユーザー名簿など他のデータ取得が終わるまで待たされていて遅く感じられていた。`renderLogs()`は`logs`（`loadLogs()`の結果）だけで描画できるので、他の読み込みを待たず最優先で取得し、届いた時点ですぐ表示する」。
- `const logsPromise = loadLogs();`で発行した通信の`Promise`を、`.then(renderLogs)`（届き次第すぐ描画）と、`Promise.all([logsPromise, ...])`（他の読み込みと合わせて全部揃ってからもう一度描画）の**両方で使い回しています**。同じ`loadLogs()`を2回呼ぶと通信が重複してしまいますが、1回発行した`Promise`は何度でも`.then`で結果を受け取れるという性質を利用して、二重フェッチを避けながら「届いたらすぐ」と「全部揃ったら最終的に正しく」の両方を実現しています。
- `if (new URLSearchParams(location.search).get("openAccount")) openAccountModal();`：[00_HTML構造と全体像.md](00_HTML構造と全体像.md)で触れた、他ページの「アカウント設定」リンクから`?openAccount=1`付きでこのページに来た場合の自動オープン処理です。

```js
function applySession() {
  var avatarEl   = document.getElementById("header-avatar");
  ...
  attachAccountClickHandlers();
}
function attachAccountClickHandlers() {
  hideLegacyAccountButton();
  var nicknameEl = document.getElementById("header-nickname");
  var idEl       = document.getElementById("header-id");
  var avatarEl   = document.getElementById("header-avatar");
  [nicknameEl, idEl, avatarEl].forEach(function(el) {
    if (!el) return;
    el.style.cursor = "pointer";
    el.title = "タップしてアカウント設定を開く";
    el.onclick = openAccountModal;
  });
}
```
- ヘッダーのアバター・ニックネーム・学籍番号の3つの要素それぞれに、クリックで[01_StudyLog.js_その1_ログインとアカウント設定.md](01_StudyLog.js_その1_ログインとアカウント設定.md)の`openAccountModal()`を開くよう仕込みます。コメントによれば、以前は専用の「⚙アカウント」ボタンがありましたが、今はヘッダーの表示自体をタップできるようにする方式に変わり、その名残の古いボタンを`hideLegacyAccountButton()`で隠す処理も残っています。

---

## 2. 日付ユーティリティ（636〜658行）

```js
function getWeekRange() {
  var now = new Date(), day = now.getDay();
  var diff = day === 0 ? -6 : 1 - day;
  var mon = new Date(now); mon.setDate(now.getDate() + diff); mon.setHours(0,0,0,0);
  var sun = new Date(mon); sun.setDate(mon.getDate() + 6);   sun.setHours(23,59,59,999);
  return { mon: mon, sun: sun };
}
```
- 今週の月曜0:00と日曜23:59:59を求める関数です。`diff = day === 0 ? -6 : 1 - day`は、[../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md](../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md)で見た「今週の月曜日を求める」計算と同じ考え方を、日曜日を`0`とする`getDay()`の値から直接「月曜日までの日数差」として計算する書き方です（日曜日なら6日**戻る**ので`-6`、それ以外の曜日なら`1 - day`日進める/戻る）。
- `getThisWeekLogs()`はこの範囲を使って、全ログから今週分だけを絞り込みます。

---

## 3. 今週の獲得ポイント計算：`calcWeeklyPoints(wl)`（663〜694行）

ランキングやサマリー表示のあちこちから呼ばれる、週間ポイントの集計ロジックです。

```js
var map = {};
// ① 勉強ログ分（全ユーザー）
wl.forEach(function(l) {
  if (!map[l.student_id]) map[l.student_id] = 0;
  map[l.student_id] += Math.floor(l.minutes / 5);
});
```
- 勉強時間からのポイントは「5分につき1pt」（コメントにある計算式通り）で、`Math.floor`（切り捨て）で端数を切り捨てます。

```js
// ② 課題達成分（全ユーザー・今週達成したもの）
Object.keys(allCompletedTasks).forEach(function(sid) {
  (allCompletedTasks[sid] || []).forEach(function(e) {
    if (!e.date) return;
    var d = new Date(e.date); d.setHours(0, 0, 0, 0);
    if (d < r.mon || d > r.sun) return;
    var pts;
    if (e.points != null) { pts = e.points; }
    else { var task = TASKS_JSON.find(function(t) { return t.id === e.id; }); pts = task ? task.points : DEFAULT_TASK_POINTS; }
    if (!map[sid]) map[sid] = 0;
    map[sid] += pts;
  });
});
```
- 課題達成によるポイントは、**達成日が今週に入っているものだけ**を対象にします。ポイント数は、達成記録自体に`points`が保存されていればそれを優先し（達成した時点でのポイント数を記録として残しているということです）、無ければ現在の`TASKS_JSON`（今表示されている課題データ）から探して補完し、それも見つからなければデフォルトの5ptにフォールバックします。この3段階のフォールバックにより、「達成後に課題側のポイント設定が変わっても、過去の達成記録の集計結果はブレない」ことをある程度保っています。

---

## 4. ポイント表示のアニメーション（699〜713行）

```js
function floatPoints(txt) {
  var wrap = document.getElementById("point-wrap");
  if (!wrap) return;
  var old = wrap.querySelector(".sl-pts-pop");
  if (old) old.remove();
  var el = document.createElement("span");
  el.className   = "sl-pts-pop fly";
  el.textContent = txt;
  wrap.appendChild(el);
  el.addEventListener("animationend", function() { el.remove(); });
}
```
- ポイントを獲得したときに、ヘッダーのポイントバッジのそばに「+5pt」のような文字がふわっと浮かび上がって消えるアニメーション用の要素を作ります。`animationend`イベント（CSSアニメーションが終わった瞬間に発火する）をリスナーとして登録し、アニメーションが終わったら自分自身をDOMから削除する、という「使い捨ての演出要素」の作り方です。

---

## 5. ランキングの集計（718〜764行）

```js
function topWithTies(arr, key) {
  if (!arr.length) return [];
  var sorted = arr.slice().sort(function(a, b) { return b[key] - a[key]; });
  var result = [];
  var rank = 0, prev = null;
  for (var i = 0; i < sorted.length; i++) {
    if (sorted[i][key] !== prev) { rank = i + 1; prev = sorted[i][key]; }
    if (rank > 3) break;
    result.push(Object.assign({ rank: rank }, sorted[i]));
  }
  return result;
}
```
- `key`（`min`や`pts`）の値で降順に並べ、**上位3位までを、同点者も含めて**取り出す関数です。ポイントは、注目すべきは`rank`の決め方です：値が前の人と同じなら、順位番号（`rank`）を進めません（`prev`と比較して変化が無ければ`rank`はそのまま）。これにより、「1位が2人同点なら、次の人は3位からではなく、そのまま2位タイで並ぶ」という一般的な順位の付け方（スキップ方式ではない、いわゆる「1224」形式ではなく「1134」的な付け方）が実現されています。「`rank > 3`になったら打ち切る」条件のおかげで、同点が続く限りは3位を超えても表示され得ますが（例えば3位タイが5人いれば5人とも表示される）、順位そのものが4位になった時点で処理を止めます。

```js
function buildRankData(wl) {
  nicknameMap[STUDENT.id] = STUDENT.nickname;
  var timeMap = {};
  wl.forEach(function(l) {
    if (!timeMap[l.student_id]) { timeMap[l.student_id] = { nickname: nicknameMap[l.student_id] || l.nickname, min: 0 }; }
    timeMap[l.student_id].min += l.minutes;
  });
  var weekPtsRaw = calcWeeklyPoints(wl);
  var ptsMap = {};
  Object.keys(weekPtsRaw).forEach(function(sid) { ptsMap[sid] = { nickname: nicknameMap[sid] || sid, pts: weekPtsRaw[sid] }; });
  return { byTime: topWithTies(Object.values(timeMap), "min"), byPts:  topWithTies(Object.values(ptsMap),  "pts") };
}
```
- 今週のログ（`wl`）から、生徒ごとの合計勉強時間（`timeMap`）を集計し、4節の`calcWeeklyPoints`と合わせて、「勉強時間ランキング」「ポイントランキング」の2つのトップ3データを作ります。

---

## 6. ホーム画面全体の描画：`renderAll()`（769〜785行）

```js
function renderAll() {
  var wl  = getThisWeekLogs();
  var tot = wl.reduce(function(s,l){ return s+l.minutes; }, 0);
  var myWeekMin = wl.filter(function(l){ return l.student_id === STUDENT.id; }).reduce(function(s,l){ return s+l.minutes; }, 0);
  var myWeekPts = calcWeeklyPoints(wl)[STUDENT.id] || 0;
  var myTotalMin = logs.filter(function(l){ return l.student_id === STUDENT.id; }).reduce(function(s,l){ return s+l.minutes; }, 0);
  document.getElementById("my-week-time").textContent  = myWeekMin + "分";
  document.getElementById("my-week-pts").textContent   = myWeekPts + "pt";
  document.getElementById("my-total-time").textContent = myTotalMin + "分";
  renderRankings(wl);
  renderLogs();
  renderEveryone(wl, tot);
}
```
- ホーム画面のサマリー（今週の勉強時間・今週のポイント・累計勉強時間）を計算・表示し、ランキング・自分のログ一覧・みんなの記録の3つの描画関数をまとめて呼び出す、いわば「ホーム画面全体を作り直す」総合関数です。ニックネーム変更やログ削除など、表示に影響する操作のあとには、必ずこの関数が呼ばれます。

`renderRankings(wl)`（787〜793行）は5節の`buildRankData`の結果を、`rankHTML`（8節）でHTML化して2つのランキング列にそれぞれ流し込みます。

### 6.1 ランキング行のHTML組み立て：`rankHTML(sorted, valFn, valClass, nameKey)`（795〜811行）
```js
var medals = ["sl-r1","sl-r2","sl-r3"];
return sorted.map(function(u) {
  var rank     = u.rank || 1;
  var name     = u[nameKey] || u.nickname || "—";
  var isMe     = name === STUDENT.nickname;
  var youBadge = isMe ? '<span class="sl-you-badge">あなた</span>' : "";
  var medalCls = medals[rank - 1] || "sl-rn";
  return '<div class="sl-rank-row">' +
    '<div class="sl-rank-num ' + medalCls + '">' + rank + '</div>' +
    '<div class="sl-rank-name">' + esc(name) + youBadge + '</div>' +
    '<div class="sl-rank-val ' + valClass + '">' + valFn(u) + '</div>' +
  '</div>';
}).join("");
```
- `valFn`（値をどう表示するか、`"分"`を付けるか`"pt"`を付けるか）を関数として受け取ることで、勉強時間ランキングとポイントランキングの**両方**をこの1つの関数で描画できるようにしています。
- 1位・2位・3位にはそれぞれ専用のCSSクラス（金・銀・銅のようなメダル色を想定）が付き、4位以降は`sl-rn`（通常の番号表示）になります。
- `isMe = name === STUDENT.nickname`という判定で、自分の名前と一致する行に「あなた」バッジを付けています。ここは`student_id`ではなく**ニックネームの文字列比較**で「自分かどうか」を判定している点に注意が必要です（`timeMap`/`ptsMap`のキーは`student_id`ですが、`rankHTML`に渡す時点では既にニックネームに変換されてしまっているため、こういう比較方法になっていると考えられます）。

---

## 7. 自分のログ一覧（813〜883行）

```js
function renderLogs() {
  var myLogs = logs.filter(function(l) { return l.student_id === STUDENT.id; });
  ...
  el.innerHTML = myLogs.slice().reverse().map(function(l) {
    return '<div class="sl-log-item">' +
      '<div class="sl-log-header">' +
        '<span class="sl-log-subject">' + esc(l.subject) + '</span>' +
        '<span class="sl-log-right">' +
          '<span class="sl-log-min">' + l.minutes + '分</span>' +
          '<button class="sl-log-del-btn" data-log-time="' + escAttr(l.time) + '" title="この記録を削除">' + Icons.html('trash', {size:14}) + '</button>' +
        '</span>' +
      '</div>' +
      '<div class="sl-log-meta">' + l.date + ' · ' + esc(l.nickname) + '</div>' +
      (l.memo ? '<div class="sl-log-memo">' + esc(l.memo) + '</div>' : '') +
    '</div>';
  }).join("");
}
```
- 自分のログだけを絞り込み、`.reverse()`で新しい順に並べ替えてから表示します（元の配列は古い順に積まれていく想定のため）。
- 削除ボタンには、直接`onclick="deleteMyLog('...')"`のようにログの時刻データを埋め込むのではなく、`data-log-time`という**データ属性**として持たせています。

```js
function initMyLogsEvents() {
  var el = document.getElementById("log-list");
  if (!el || el.dataset.boundClick) return;
  el.dataset.boundClick = "1";
  el.addEventListener("click", function(e) {
    var btn = e.target.closest(".sl-log-del-btn");
    if (btn && !btn.disabled) deleteMyLog(btn.dataset.logTime, btn);
  });
}
```
- これは[../02_Cardmaker/09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](../02_Cardmaker/09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)で紹介した「イベント委任」パターンです。一覧全体（`#log-list`）に1回だけクリックリスナーを登録し、実際にクリックされた場所が削除ボタンかどうかを`closest`で判定します。コメントには「一度だけ登録すればOK」という理由に加え、`initTaskListEvents`のコメントに「inline `onclick`に生のIDや文字列を直接埋め込むと、内容に引用符（`'`や`"`）が含まれた場合にHTML/JSが壊れてボタンが反応しなくなる」という重要な注記があります。もし削除対象を`onclick="deleteMyLog('${l.time}')"`のように直接文字列として埋め込んでいたら、時刻データや科目名にたまたま引用符が含まれるだけでこのHTMLごと壊れてしまいます。データ属性＋イベント委任の組み合わせは、この種の事故を避けるための安全な設計です。
- `el.dataset.boundClick`というフラグで、この関数が複数回呼ばれても、リスナーが二重に登録されないようにしています。

```js
async function deleteMyLog(timeKey, btn) {
  ...
  data = await api("/delete_study_log", { method: "POST", body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN, time: timeKey }) });
  ...
  logs = logs.filter(function(l) { return !(l.student_id === STUDENT.id && l.time === timeKey); });
  if (data.total != null) { allPoints[STUDENT.id] = data.total; myPoints = data.total; updatePointDisplay(); }
  renderAll();
}
```
- コメントにある通り、勉強ログの削除に対応してポイントも自動的に差し引かれます（サーバー側の処理）。クライアント側では、削除に成功したらローカルの`logs`配列から該当エントリを取り除き、サーバーが返してきた最新の合計ポイント（`data.total`）で表示を更新します。

---

## 8. みんなの記録（885〜947行）

```js
var memberIds = {};
Object.keys(nicknameMap).forEach(function(id) { memberIds[id] = true; });
Object.keys(allPoints).forEach(function(id) { memberIds[id] = true; });
Object.keys(allCompletedTasks).forEach(function(id) { memberIds[id] = true; });
memberIds[STUDENT.id] = true;
```
- 「メンバー別 今週の記録」に表示する対象者を決める処理です。ニックネーム対応表・累計ポイント・達成済み課題という**3つの異なるデータソースそれぞれに登場する学籍番号を、全部まとめて（重複排除して）対象にする**、という組み立て方です。これにより、「今週まだ何も記録していないが、過去にポイントを持っている人」なども一覧に含められます。

```js
var members = Object.keys(memberIds).map(function(id) {
  return { id: id, nickname: nicknameMap[id] || id, min: weekMinMap[id] || 0, pts: weekPtsRaw[id] || 0 };
}).sort(function(a, b) {
  return (b.min - a.min) || (b.pts - a.pts) || a.nickname.localeCompare(b.nickname, "ja");
});
```
- 並び順は「勉強時間が多い順→同じなら獲得ポイントが多い順→それも同じならニックネームの五十音順」という3段階の優先順位です。`||`演算子は、左側が`0`（＝差が無い、同点）のときだけ右側の条件に進む、という並び替え条件をつなげる際の定番の書き方です。

このあとの「みんなの勉強ログ」欄（logs全件の一覧）は、7節の`renderLogs()`とほぼ同じ組み立てですが、こちらは全ユーザー分を対象にし、削除ボタンはありません（自分以外のログは削除できないため）。

---

## 9. 課題一覧（950〜1022行）

```js
function renderTasks() {
  var doneIds = completedTasks.map(function(e) { return e.id; });
  var sorted = TASKS_JSON.slice().sort(function(a, b) {
    var aDone = doneIds.includes(a.id) ? 1 : 0;
    var bDone = doneIds.includes(b.id) ? 1 : 0;
    if (aDone !== bDone) return aDone - bDone; // 未達成が先、達成済みが後ろ
    return new Date(a.due) - new Date(b.due); // 同じ達成状態同士は締切が近い順
  });
  ...
}
```
- 並び順は「未達成のものを先に、達成済みは後ろに」という大分類のあと、それぞれのグループ内では締切が近い順、という2段階のソートです。

```js
var btnLabel = pending ? '<span class="sl-spinner" ...></span>送信中…' : (done ? (Icons.html('checkCircle', {size:13}) + " 達成済み") : "達成する");
```
- ボタンの表示は「送信中（`pendingTaskIds`に含まれる、＝今まさにサーバーへ送信中）」「達成済み」「達成する（未達成）」の3状態を切り替えます。

```js
var noteDot  = t.note ? ('<span class="note-dot" title="備考あり">' + Icons.html('memo', {size:13}) + '</span>') : '';
var noteHtml = t.note ? '<div class="sl-task-note">' + esc(t.note) + '</div>' : '';
```
- コメントに「備考は普段は隠しておき、タップで表示する（[../01_index_予定管理.md](../01_index_予定管理.md)の詳細表示と同じ考え方）」とあります。備考の中身自体はHTMLとして最初から埋め込んでおき、CSSの`display`ではなく専用のクラス（`open`）の有無で開閉を切り替える方式です（`toggleTaskNote`、下記）。

```js
function initTaskListEvents() {
  var el = document.getElementById("task-list");
  if (!el || el.dataset.boundClick) return;
  el.dataset.boundClick = "1";
  el.addEventListener("click", function(e) {
    var btn = e.target.closest(".sl-task-btn");
    if (btn) { if (btn.disabled) return; toggleTask(btn.dataset.taskId); return; }
    var body = e.target.closest(".sl-task-body.has-note");
    if (body) toggleTaskNote(body);
  });
}
function toggleTaskNote(bodyEl) {
  var noteEl = bodyEl.querySelector(".sl-task-note");
  if (!noteEl) return;
  noteEl.classList.toggle("open");
}
```
- 課題リストも7節と同じイベント委任パターンです。「達成する」ボタンがクリックされたか、備考付きの本文（`.has-note`）がクリックされたかを`closest`で振り分け、それぞれ`toggleTask`（[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)で解説）と`toggleTaskNote`（備考の開閉）を呼び分けます。

---

続きは[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)で、タブ切り替え・手入力での記録・課題の達成処理を解説します。
