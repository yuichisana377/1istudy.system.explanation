# Timetable.js その3：学期の基本時間割設定と保存処理（974〜1346行）

[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)の続きです。

---

## 1. 学期データの下ごしらえ（985〜1004行）

コメントに、この機能全体の考え方がまとめられています：「前期／後期のように、期間ごとにまるごと違う基本時間割を、それぞれ独立したデータとしてサーバーに保存する。学期は`id`ごとに完全に別データなので、後期の内容を編集・保存しても前期のデータは一切変更されない。表示側は`getTimetableForDate(dateStr)`（[01_Timetable.js_その1_週表示と月間カレンダー.md](01_Timetable.js_その1_週表示と月間カレンダー.md)）が、その日付が含まれる学期のデータを自動で選んで使う」。

```js
function emptyTermTimetable() { return { mon: [], tue: [], wed: [], thu: [], fri: [] }; }
function cloneDayPeriods(periods) { return (periods || []).map(p => ({ subject: p.subject || '', items: [...(p.items || [])] })); }
function cloneTimetableForNewTerm() {
  const base = getTimetableForDate(getDateStr(new Date()));
  const copy = emptyTermTimetable();
  TERM_DAY_KEYS.forEach(k => { copy[k] = cloneDayPeriods(base[k]); });
  return copy;
}
```
- `cloneTimetableForNewTerm()`：新しい学期を作るときの初期値として、**今日時点で実際に使われている時間割**（学期が設定されていなければ`DEFAULT_TIMETABLE`）をコピーして使います。コメントの通り、これは「ゼロから全コマ入力しなくて済むようにするため」の親切設計です。`cloneDayPeriods`が各曜日のコマ配列を1つずつ複製しているのは、元のデータをそのまま参照してしまうと、編集中の変更が意図せず元データ（`DEFAULT_TIMETABLE`など）自体を書き換えてしまう事故につながるため、必ず独立したコピーを作っています。

```js
function escapeAttr(s) {
  return String(s == null ? '' : s).replace(/&/g, '&amp;').replace(/"/g, '&quot;').replace(/</g, '&lt;');
}
```
- これがこのファイル全体で使われている「エスケープ関数」の実体です。[../01_index_予定管理.md](../01_index_予定管理.md)の`esc()`とほぼ同じ働き（`&`/`"`/`<`を無害な表記に変換）ですが、`>`のエスケープは行っていません。名前が違うだけで、実質的には同じ役割の関数がこのファイルにも独自に定義されている、という状況です。

---

## 2. 学期設定モーダルの開閉とフォーム管理（1006〜1048行）

```js
function openTermModal() {
  closeTTFab();
  resetTermForm();
  loadTerms().then(renderTermList);
  document.getElementById('modal-tt-term').classList.add('open');
}
function resetTermForm() {
  termEditState = { id: null, activeDay: 'mon', timetable: cloneTimetableForNewTerm() };

  const nameSel = document.getElementById('tt-term-name-sel');
  const nameInp = document.getElementById('tt-term-name-inp');
  if (nameSel) nameSel.value = '前期';
  if (nameInp) { nameInp.style.display = 'none'; nameInp.value = ''; }

  initCal('tt-term-start', true);
  initCal('tt-term-end', true);
  resetCal('tt-term-start', '開始日を選択');
  resetCal('tt-term-end', '終了日を選択');

  document.querySelectorAll('#tt-term-day-tabs .tt-mode-btn').forEach((b, i) => {
    b.classList.toggle('active', i === 0);
  });
  renderTermDayEditor();

  const err = document.getElementById('tt-term-err');
  if (err) err.style.display = 'none';
  resetLoading(document.getElementById('tt-term-submit-btn'), '保存する');
}
```
- `termEditState`は、学期編集モーダルで**今まさに入力中の内容**を保持するオブジェクトです。`id`が`null`なら「新規作成中」、値が入っていれば「既存の学期を編集中」を意味します。
- モーダルを開くたびに`loadTerms()`でサーバーから最新の学期一覧を取り直し、`renderTermList()`（4節）で登録済み学期の一覧を更新します。

```js
function onTermNameSel() {
  const sel = document.getElementById('tt-term-name-sel');
  const inp = document.getElementById('tt-term-name-inp');
  if (!sel || !inp) return;
  if (sel.value === '__custom__') { inp.style.display = 'block'; inp.focus(); }
  else { inp.style.display = 'none'; }
}
function getTermNameValue() {
  const sel = document.getElementById('tt-term-name-sel');
  if (!sel) return '';
  if (sel.value === '__custom__') return (document.getElementById('tt-term-name-inp')?.value || '').trim();
  return sel.value;
}
```
`onTermNameSel()`/`getTermNameValue()`（1036〜1048行）は、学期名の入力欄が「前期」「後期」の定型選択か「自由入力…」かで、実際に使う値の取り出し方を切り替える、[../01_index_予定管理.md](../01_index_予定管理.md)の`getCatValue`と同じパターンの関数です。

---

## 3. 曜日ごとの時限編集（1050〜1100行）

```js
function switchTermDay(day) {
  if (!termEditState) return;
  termEditState.activeDay = day;
  const idx = TERM_DAY_KEYS.indexOf(day);
  document.querySelectorAll('#tt-term-day-tabs .tt-mode-btn').forEach((b, i) => {
    b.classList.toggle('active', i === idx);
  });
  renderTermDayEditor();
}
function renderTermDayEditor() {
  const container = document.getElementById('tt-term-day-editor');
  if (!container || !termEditState) return;
  const day = termEditState.activeDay;
  const periods = termEditState.timetable[day] || [];

  const rows = periods.map((p, i) => `
    <div class="tt-term-period-row">
      <div class="tt-term-period-num">${i + 1}限</div>
      <div class="tt-term-period-fields">
        <input type="text" value="${escapeAttr(p.subject)}" placeholder="科目名"
          oninput="updateTermPeriod('${day}',${i},'subject',this.value)">
        <input type="text" value="${escapeAttr((p.items || []).join(','))}" placeholder="持ち物（カンマ区切り）"
          oninput="updateTermPeriod('${day}',${i},'items',this.value)">
      </div>
      <button type="button" class="tt-term-period-del" onclick="removeTermPeriod('${day}',${i})" title="このコマを削除"><svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#dc2626" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M6 6l12 12"/><path d="M18 6L6 18"/></svg></button>
    </div>`).join('');

  container.innerHTML =
    (rows || `<div style="font-size:13px;color:var(--text-tertiary);padding:6px 0">まだコマがありません</div>`) +
    `<button type="button" class="tt-btn-secondary" onclick="addTermPeriod('${day}')">＋ コマを追加</button>`;
}
```
- 月〜金の曜日タブを切り替えながら、その曜日の時限（コマ）を自由に追加・削除・編集できるエディタです。
- 各入力欄の`oninput`属性に、`updateTermPeriod('mon', 0, 'subject', this.value)`のように、直接**その場のインデックス番号を埋め込んだ**JSコードを書いている点が特徴です。`this.value`は「今この`<input>`に入力されている値」を指すJSの書き方です。

```js
function updateTermPeriod(day, idx, field, value) {
  if (!termEditState) return;
  const p = termEditState.timetable[day][idx];
  if (!p) return;
  if (field === 'items') p.items = value.split(',').map(s => s.trim()).filter(Boolean);
  else p.subject = value;
}
```
- 入力があるたびに`termEditState`（今編集中のデータ）を直接書き換えます。この関数は画面の再描画を一切行いません（入力欄自体は既にそこにあるので、再描画すると逆にカーソル位置が飛んでしまう可能性があるため、データの更新だけに専念しています）。
- `addTermPeriod`/`removeTermPeriod`はコマの追加・削除を行い、こちらは配列の要素数自体が変わるため`renderTermDayEditor()`で表示を作り直します。

---

## 4. 登録済み学期の一覧・編集・削除（1102〜1190行）

```js
function renderTermList() {
  const el = document.getElementById('tt-term-list');
  if (!el) return;
  if (!terms.length) {
    el.innerHTML = '<div style="font-size:13px;color:var(--text-tertiary)">まだ学期が登録されていません（未登録の期間はデフォルトの時間割が使われます）</div>';
    return;
  }
  const sorted = [...terms].sort((a, b) => (a.start_date < b.start_date ? -1 : 1));
  el.innerHTML = sorted.map(t => `
    <div class="tt-term-list-row">
      <div class="tt-term-list-info">
        <div class="tt-term-list-name">${escapeAttr(t.name)}</div>
        <div class="tt-term-list-range">${escapeAttr(t.start_date)} 〜 ${escapeAttr(t.end_date)}</div>
      </div>
      <button type="button" class="tt-btn-secondary" onclick="editTermFromList('${t.id}')">編集</button>
      <button type="button" class="tt-term-period-del" onclick="deleteTermFromList('${t.id}')" title="削除"><svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#dc2626" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M6 6l12 12"/><path d="M18 6L6 18"/></svg></button>
    </div>`).join('');
}
```
- 登録済みの学期を開始日順に並べて一覧表示します。`YYYY-MM-DD`形式の文字列は、そのまま`<`で比較しても日付の前後関係と一致するため（[../01_index_予定管理.md](../01_index_予定管理.md)でも触れた性質）、特別な日付変換をせずに`a.start_date < b.start_date`で並べ替えられています。

```js
function editTermFromList(id) {
  const t = terms.find(x => x.id === id);
  if (!t) return;

  termEditState = {
    id: t.id,
    activeDay: 'mon',
    timetable: (() => {
      const copy = emptyTermTimetable();
      TERM_DAY_KEYS.forEach(k => { copy[k] = cloneDayPeriods(t.timetable[k]); });
      return copy;
    })(),
  };

  const nameSel = document.getElementById('tt-term-name-sel');
  const nameInp = document.getElementById('tt-term-name-inp');
  if (nameSel && nameInp) {
    if (t.name === '前期' || t.name === '後期') {
      nameSel.value = t.name;
      nameInp.style.display = 'none';
    } else {
      nameSel.value = '__custom__';
      nameInp.style.display = 'block';
      nameInp.value = t.name;
    }
  }

  initCal('tt-term-start', true);
  initCal('tt-term-end', true);
  calState['tt-term-start'].selected = t.start_date;
  calState['tt-term-end'].selected   = t.end_date;
  const [sy, sm, sd] = t.start_date.split('-');
  const [ey, em, ed] = t.end_date.split('-');
  const startText = document.getElementById('tt-term-start-date-text');
  const endText   = document.getElementById('tt-term-end-date-text');
  if (startText) { startText.textContent = `${sy}年${parseInt(sm)}月${parseInt(sd)}日`; startText.style.color = 'var(--text)'; }
  if (endText)   { endText.textContent   = `${ey}年${parseInt(em)}月${parseInt(ed)}日`; endText.style.color   = 'var(--text)'; }
  renderCal('tt-term-start');
  renderCal('tt-term-end');

  switchTermDay('mon');

  const err = document.getElementById('tt-term-err');
  if (err) err.style.display = 'none';
}
```
- 既存の学期を編集モードで開く関数です。ここでも1節の`cloneDayPeriods`が使われ、`terms`配列に入っている元データを直接書き換えないよう、複製してから`termEditState`にセットしています（キャンセルした場合に元のデータが壊れないようにするための配慮です）。

```js
async function deleteTermFromList(id) {
  const session = requireLoginOrRedirect();
  if (!session) return;
  const t = terms.find(x => x.id === id);
  if (!t) return;
  const ok = await showAppConfirm({
    title: '削除しますか？', desc: `「${t.name}」（${t.start_date}〜${t.end_date}）を削除します。`,
    okLabel: '削除する', danger: true,
  });
  if (!ok) return;
  try {
    const res = await api(TERM_API.DELETE, { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, id, nickname: session.nickname }) });
    if (res.ok) {
      await loadTerms();
      renderTermList();
      renderTimetable();
      if (termEditState && termEditState.id === id) resetTermForm();
    } else {
      showErr('tt-term-err', res.error || '削除に失敗しました');
    }
  } catch (e) {
    showErr('tt-term-err', 'サーバーに接続できませんでした');
  }
}
```
`deleteTermFromList(id)`（1167〜1190行）は確認ダイアログのあと、サーバーの`/delete_term`に削除を依頼し、成功したら学期一覧・時間割表示の両方を更新します。**もし削除した学期が、今まさに編集中の学期だった場合は`resetTermForm()`でフォームもリセットします**（存在しなくなったデータを編集し続けてしまう状態を防ぐため）。

---

## 5. 学期の保存：`submitTermSave()`（1192〜1235行）

```js
const name      = getTermNameValue();
const startDate = calState['tt-term-start']?.selected;
const endDate   = calState['tt-term-end']?.selected;
if (!name)                { showErr('tt-term-err', '学期名を入力してください'); return; }
if (!startDate)            { showErr('tt-term-err', '開始日を選択してください'); return; }
if (!endDate)              { showErr('tt-term-err', '終了日を選択してください'); return; }
if (endDate < startDate)   { showErr('tt-term-err', '終了日は開始日以降にしてください'); return; }
```
- 学期名・開始日・終了日の入力チェックに加え、`endDate < startDate`という**文字列としての比較**で日付の前後関係が正しいかもチェックしています（4節と同じ理由で、`YYYY-MM-DD`形式ならこの比較で正しく判定できます）。

```js
const btn = document.getElementById('tt-term-submit-btn');
setLoading(btn, '保存中…');
try {
  const res = await api(TERM_API.SAVE, {
    method: 'POST',
    body: JSON.stringify({
      guild_id:   GUILD_ID,
      session_token: session.session_token,
      id:         termEditState.id || undefined,
      name,
      start_date: startDate,
      end_date:   endDate,
      timetable:  termEditState.timetable,
      nickname:   session.nickname,
    })
  });
  resetLoading(btn, '保存する');
  if (res.ok) {
    showOk('tt-term-ok');
    await loadTerms();
    renderTermList();
    renderTimetable();
    resetTermForm();
  } else {
    showErr('tt-term-err', res.error || '保存に失敗しました');
  }
} catch (e) {
  resetLoading(btn, '保存する');
  showErr('tt-term-err', 'サーバーに接続できませんでした');
}
```
- `id: termEditState.id || undefined`：既存の学期を編集中なら、その`id`をそのまま送って上書き保存。新規作成中（`id`が`null`）なら`undefined`を送ることで、サーバー側に「新規作成」と判断させます（JSでは`JSON.stringify`する際、値が`undefined`のプロパティはそもそも出力されないため、サーバー側は`id`キー自体が無いリクエストとして受け取ります）。
- 成功したら学期一覧・時間割表示の両方を更新し、フォームをリセットして次の入力に備えます。

---

## 6. 複数日への一括適用（1237〜1247行）

```js
function enumerateDates(startStr, endStr) {
  const dates = [];
  let cur = new Date(startStr + 'T00:00:00');
  const end = new Date(endStr + 'T00:00:00');
  while (cur <= end) {
    dates.push(getDateStr(cur));
    cur.setDate(cur.getDate() + 1);
  }
  return dates;
}
```
- 開始日から終了日まで（両端を含む）、1日ずつ日付文字列を並べた配列を作ります。`cur.setDate(cur.getDate() + 1)`で`Date`オブジェクトを1日ずつ進め、`while`ループで終了日を超えるまで繰り返す、日付範囲を列挙する定番のやり方です。

---

## 7. 時間割編集の実際の保存：`submitTTEdit()`（1249〜1308行）

[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)で見た「1日分の保存関数」（`applyHolidayForDate`など）を、実際にどう呼び出すかがここにまとまっています。

```js
const isMulti = document.getElementById('tt-edit-multi')?.checked;
let dateList = [startDate];
if (isMulti) {
  const endDate = calState['tt-edit-end']?.selected;
  if (!endDate) { showErr('tt-edit-err', '終了日を選択してください'); return; }
  if (endDate < startDate) { showErr('tt-edit-err', '終了日は開始日以降の日付にしてください'); return; }
  dateList = enumerateDates(startDate, endDate);
}
```
- 「複数日にまとめて適用」がオフなら、対象は選択した1日だけ（`[startDate]`という1件の配列）。オンなら、6節の`enumerateDates`で期間全体の日付リストを作ります。

```js
try {
  if (ttEditMode === 'holiday') {
    const reason = document.getElementById('tt-edit-holiday-reason').value;
    const note   = document.getElementById('tt-edit-holiday-note').value.trim();
    for (const d of dateList) await applyHolidayForDate(d, reason, note);

  } else if (ttEditMode === 'period-holiday') {
    // ★ 1コマだけの休み
    const period = parseInt(document.getElementById('tt-edit-period').value);
    const reason = document.getElementById('tt-edit-period-holiday-reason').value.trim() || '休み';
    const note   = document.getElementById('tt-edit-period-holiday-note').value.trim();
    for (const d of dateList) await applyPeriodHolidayForDate(d, period, reason, note);

  } else if (ttEditMode === 'day-change') {
    const sourceDayKey = document.getElementById('tt-day-change-source').value;
    if (!sourceDayKey) { resetLoading(btn, '保存する'); showErr('tt-edit-err', 'コピー元の曜日を選択してください'); return; }
    if (!isMulti && !dateToDayKey(startDate)) { resetLoading(btn, '保存する'); showErr('tt-edit-err', '土日は選択できません'); return; }
    const note = document.getElementById('tt-day-change-note').value.trim();
    for (const d of dateList) await applyDayChangeForDate(d, sourceDayKey, note);

  } else {
    // 授業変更（1コマ）
    const period   = parseInt(document.getElementById('tt-edit-period').value);
    const subject  = document.getElementById('tt-edit-subject').value.trim();
    const itemsRaw = document.getElementById('tt-edit-items').value.trim();
    const items    = itemsRaw ? itemsRaw.split(',').map(s => s.trim()).filter(Boolean) : [];
    const note     = document.getElementById('tt-edit-note').value.trim();
    if (!subject) { resetLoading(btn, '保存する'); showErr('tt-edit-err', '科目を選択してください'); return; }
    for (const d of dateList) await applyChangeForDate(d, period, subject, items, note);
  }

  saveTTOverrideLocal();
  resetLoading(btn, '保存する');
  showOk('tt-edit-ok');
  resetTTEditForm();
  renderTTOverridesList();
  renderTimetable();
} catch(e) {
  resetLoading(btn, '保存する');
  showErr('tt-edit-err', '保存に失敗しました: ' + e.message);
}
```
- 今選ばれている編集モード（`ttEditMode`）に応じて、入力欄からそのモード専用の値を取り出し、`dateList`に含まれる**全ての日付**に対して、[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)で見た「1日分の保存関数」を`for...of`ループで1つずつ順番に（`await`しながら）呼び出します。これにより「複数日にまとめて適用」機能が実現されています。夏休みのような長期間を指定すると、その日数分だけ通信が発生することになります。
- 授業変更モードでは`if (!subject) { ...; return; }`で科目が選ばれていないと保存できないようにするなど、モードごとに個別の入力チェックも行われています。
- 全ての日付の保存処理が終わったら、最後にまとめて`saveTTOverrideLocal()`でローカル保存し、フォームをリセットし、変更一覧・時間割表示の両方を更新します。

---

## 8. 保存済み変更の一覧と削除（1309〜1345行）

```js
async function deleteTTOverride(key) {
  const session = requireLoginOrRedirect();
  if (!session) return;
  try { await api(TT_API.DELETE, { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, key, nickname: session.nickname }) }); } catch(_) {}
  delete ttOverrides[key];
  saveTTOverrideLocal();
  renderTTOverridesList();
  renderTimetable();
}
```
- 時間割編集モーダル下部の「保存済みの変更一覧」から、1件の上書き（休講・時限休み・授業変更のいずれか）を削除します。サーバーへの削除リクエストが失敗しても（`catch(_) {}`）、ローカルの`ttOverrides`からは削除し、画面には反映します（[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)の`applyHolidayForDate`などと同じ「楽観的更新」の考え方）。

```js
function renderTTOverridesList() {
  const el   = document.getElementById('tt-overrides-list');
  const keys = Object.keys(ttOverrides).sort();
  if (!keys.length) {
    el.innerHTML = '<div style="font-size:13px;color:var(--text-tertiary);padding:10px 0">保存済みの変更はありません</div>';
    return;
  }
  el.innerHTML = keys.map(key => {
    const ov = ttOverrides[key];
    let info = '', badge = '';
    if (ov.type === 'holiday') {
      const note = ov.note ? `（${escapeAttr(ov.note)}）` : '';
      info  = `${escapeAttr(ov.date)}　${escapeAttr(ov.reason)}${note}`;
      badge = `<span class="override-badge-holiday">休校</span>`;
    } else if (ov.type === 'period_holiday') {
      const note = ov.note ? `（${escapeAttr(ov.note)}）` : '';
      info  = `${escapeAttr(ov.date)} ${ov.period}限　${escapeAttr(ov.reason)}${note}`;
      badge = `<span class="override-badge-holiday">1コマ休み</span>`;
    } else {
      info  = `${escapeAttr(ov.date)} ${ov.period}限 → ${escapeAttr(ov.subject)}`;
      badge = `<span class="override-badge-change">変更</span>`;
    }
    return `<div class="override-row">
      <div class="override-info">${badge} ${info}</div>
      <button class="override-del-btn" onclick="deleteTTOverride('${key}')" title="削除"><svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#dc2626" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M6 6l12 12"/><path d="M18 6L6 18"/></svg></button>
    </div>`;
  }).join('');
}
```
- `ttOverrides`のキー（`holiday:2026-08-20`のような形式の文字列）をアルファベット順に並べているだけですが、キーの先頭が種類・そのあと日付という形式になっているため、結果的に「種類ごとにまとまり、かつ日付順」に近い並びになります。
- 上書きの種類（`ov.type`）に応じて、表示するラベル文言とバッジの見た目を出し分けています。

---

続きは[04_Timetable.js_その4_予定管理モーダル共通処理.md](04_Timetable.js_その4_予定管理モーダル共通処理.md)で、`Plan.js`と共通の予定管理機能（このページでも使える追加・編集・削除）を解説します。
