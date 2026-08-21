# Timetable.js その2：時間割編集モーダル（780〜974行）

[[01_Timetable.js_その1_週表示と月間カレンダー.md]]の続きです。

---

## 1. モーダルを開く・初期化する（783〜831行）

```js
function openTTEditModal() {
  closeTTFab();
  initCal('tt-edit', true);
  initCal('tt-edit-end', true);
  resetTTEditForm();
  const ttSubjectEl = document.getElementById('tt-edit-subject');
  if (ttSubjectEl) {
    const opts = channels.map(c => `<option value="${escapeAttr(c.name)}">${escapeAttr(c.name)}</option>`).join('');
    ttSubjectEl.innerHTML = '<option value="">科目を選択</option>' + opts;
  }
  switchTTMode('change');
  renderTTOverridesList();
  document.getElementById('modal-tt-edit').classList.add('open');
}
```
- カレンダーを2つ（対象日付用・複数日設定の終了日用）初期化し、フォームをリセットし、科目セレクトを最新の科目一覧で更新してから、デフォルトで「授業変更」タブを開いた状態でモーダルを表示します。同時に、下部の「保存済みの変更一覧」（[[03_Timetable.js_その3_学期設定と保存処理.md]]で解説）も更新します。

`resetTTEditForm()`（800〜831行）は、4つの編集モードすべての入力欄をまとめて初期状態に戻す関数です。「複数日にまとめて適用する」チェックボックスも毎回オフに戻され、対応する終了日欄も隠されます。

---

## 2. 「複数日にまとめて適用」トグル（833〜845行）

```js
function onMultiDateToggle() {
  const checked   = !!document.getElementById('tt-edit-multi')?.checked;
  const endField  = document.getElementById('tt-edit-end-date-field');
  const dateLabel = document.getElementById('tt-edit-date-label');
  if (endField)  endField.style.display = checked ? '' : 'none';
  if (dateLabel) dateLabel.textContent  = checked ? '開始日' : '対象日付';
  if (ttEditMode === 'day-change') {
    const d = calState['tt-edit']?.selected;
    if (d) renderDayChangePreview(d);
  }
}
```
- 夏休みのような「同じ変更を何日分もまとめて適用したい」場面向けの機能です。チェックがONになると、終了日の入力欄を表示し、日付欄のラベルを「対象日付」から「開始日」に変え、選んだ期間全体に同じ変更を適用する、という意味に切り替わります。
- 曜日変更モードを見ているときは、この切り替えに合わせてプレビュー（3節）も更新します。

---

## 3. 編集タイプの切り替え（846〜920行）

### 3.1 タブ切り替え：`switchTTMode(mode)`（846〜866行）
```js
function switchTTMode(mode) {
  ttEditMode = mode;
  ['change', 'period-holiday', 'day-change', 'holiday'].forEach(m => {
    const btn = document.getElementById('tt-mode-btn-' + m);
    if (btn) btn.classList.toggle('active', m === mode);
  });
  document.getElementById('tt-edit-change-fields').style.display         = (mode === 'change')         ? '' : 'none';
  document.getElementById('tt-edit-period-holiday-fields').style.display = (mode === 'period-holiday') ? '' : 'none';
  document.getElementById('tt-edit-day-change-fields').style.display     = (mode === 'day-change')     ? '' : 'none';
  document.getElementById('tt-edit-holiday-fields').style.display        = (mode === 'holiday')        ? '' : 'none';
  document.getElementById('tt-edit-period-field').style.display          = (mode === 'change' || mode === 'period-holiday') ? '' : 'none';
  if (mode === 'day-change') {
    const d = calState['tt-edit']?.selected;
    if (d) renderDayChangePreview(d);
  }
}
```
- 「授業変更」「1コマ休み」「曜日変更」「休校設定」の4タブを切り替える中心の関数です。4つの入力欄ブロックのうち、選ばれたタブに対応するものだけを表示し、他は隠します。
- 「時限」の選択欄は、「授業変更」と「1コマ休み」の2つのモードでだけ使う（曜日変更・終日休校は時限を問わない操作のため）ので、それに合わせて表示・非表示を切り替えます。

### 3.2 曜日変更のプレビュー：`renderDayChangePreview(dateStr)`（876〜920行）
「曜日変更」タブは、「今日は金曜日だけど、月曜日の時間割で授業をする」といった状況（台風による日程調整などを想定）のためのモードです。コピー元の曜日を選ぶと、実際に入れ替わる内容をプレビュー表示します。

```js
const isMulti = document.getElementById('tt-edit-multi')?.checked;
if (isMulti) {
  const endDate = calState['tt-edit-end']?.selected;
  container.innerHTML = `... ${dateStr} 〜 ${endDate || '（終了日未選択）'} の期間中の平日すべてに、${DAY_KEY_LABEL[sourceDayKey]}曜日の時間割を適用します（土日は自動的にスキップされます） ...`;
  return;
}
```
- 複数日モードのときは、1日ごとの詳細なプレビューではなく「この期間の平日すべてに適用します」という簡単な説明文だけを表示します（対象日数が多くなりうるため、1日ずつのプレビューは現実的でないからだと考えられます）。

```js
const targetPeriods = getTimetableForDate(dateStr)[targetDayKey] || [];
const sourcePeriods = getTimetableForDate(dateStr)[sourceDayKey] || [];
const rows = targetPeriods.map((_, i) => {
  const periodNum = i + 1;
  const src = sourcePeriods[i];
  const subjectText = src ? src.subject : '（コマなし・空きコマ扱い）';
  return `<div ...>${periodNum}限　${subjectText}</div>`;
}).join('');
```
- 単一日モードのときは、実際の時限ごとに「入れ替え後どうなるか」を1行ずつ表示します。**対象日の元々のコマ数（`targetPeriods.length`）を基準にループする**点がポイントです。もしコピー元の曜日にそのコマ数分の授業が無ければ（例えば月曜が4コマ、金曜が5コマの場合）、「コマなし・空きコマ扱い」と表示されます。

---

## 4. 実際に上書きを保存する「1日分」の関数群（925〜972行）

ここからは、4つの編集モードそれぞれについて、**1日分**の変更をサーバーとローカルの両方に保存する関数です。「複数日にまとめて適用」機能から同じ関数がループで何度も呼ばれることを見越して、あえて「1日分だけを担当する部品」として独立させてあります（[[03_Timetable.js_その3_学期設定と保存処理.md]]の`submitTTEdit()`が実際にこれらをループで呼び出します）。

```js
async function applyHolidayForDate(date, reason, note) {
  const key = `holiday:${date}`;
  const session = getLoginSession();
  try { await api(TT_API.HOLIDAY, { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session?.session_token, date, reason, note, key, nickname: session?.nickname }) }); } catch(_) {}
  ttOverrides[key] = { key, type: 'holiday', date, reason, note };
}
```
- `applyHolidayForDate`（終日休校）・`applyPeriodHolidayForDate`（1コマだけの休み）・`applyChangeForDate`（授業変更）は、どれも同じパターンです：サーバーへの保存リクエストを送り（失敗しても`catch(_) {}`で握りつぶして処理は止めない）、その結果を待たずに**その場でローカルの`ttOverrides`にも同じ内容を反映**します。これにより、通信が多少遅くても、画面上はすぐに変更後の内容が見えるようになります（サーバーへの反映に失敗していても、この端末の表示は先に進む、という楽観的な更新です）。

```js
async function applyDayChangeForDate(date, sourceDayKey, note) {
  const targetDayKey = dateToDayKey(date);
  if (!targetDayKey) return; // 土日は自動的にスキップ
  const targetPeriods = getTimetableForDate(date)[targetDayKey] || [];
  const sourcePeriods = getTimetableForDate(date)[sourceDayKey] || [];
  for (let i = 0; i < targetPeriods.length; i++) {
    const periodNum = i + 1;
    delete ttOverrides[`change:${date}:${periodNum}`];
    delete ttOverrides[`period_holiday:${date}:${periodNum}`];
  }
  for (let i = 0; i < targetPeriods.length; i++) {
    const periodNum = i + 1;
    const src = sourcePeriods[i];
    if (src) {
      await applyChangeForDate(date, periodNum, src.subject, src.items || [], note);
    } else {
      await applyPeriodHolidayForDate(date, periodNum, 'コマなし', note);
    }
  }
}
```
- 「曜日変更」は、実は独立した専用のデータ形式を持たず、**内部的には「授業変更」と「1コマ休み」の組み合わせとして実現されています**。まず対象日の全コマ分の既存の上書き（変更・時限休み）を一旦すべて削除し（`delete ttOverrides[...]`）、そのあとコピー元の曜日の内容を1コマずつ確認しながら、内容があれば`applyChangeForDate`（授業変更として登録）、無ければ`applyPeriodHolidayForDate`（「コマなし」という理由の時限休みとして登録）を、それぞれ`await`で1つずつ順番に呼び出しています。
- 土日（`dateToDayKey`が`null`を返す日）は、この関数の先頭で即座に`return`することで、処理をスキップします。これは、複数日モードで「夏休み中の平日すべてに適用」のようなケースで、期間中に含まれる土日をこの関数が自動的に無視するために使われています（[[03_Timetable.js_その3_学期設定と保存処理.md]]の`submitTTEdit`が期間内の全日付についてこの関数を呼ぶため、土日もそのまま渡されてきますが、ここで弾かれます）。

---

続きは[[03_Timetable.js_その3_学期設定と保存処理.md]]で、学期（前期・後期）の基本時間割設定と、実際の保存処理（`submitTTEdit`）を解説します。
