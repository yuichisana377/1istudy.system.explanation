# Timetable.js その4：予定管理の共通機能・カレンダー・リアルタイム更新（1347〜1871行）

[03_Timetable.js_その3_学期設定と保存処理.md](03_Timetable.js_その3_学期設定と保存処理.md)の続きです。ここが`Timetable.js`最後のパートです。

---

## 1. 予定管理機能（1347〜1602行）— `Plan.js`との重複部分

このブロック（`loadChannels`/`loadPlans`/`renderChannelOptions`/`parsePlanContent`/`renderSelectList`/`selectPlan`/`openModal`/`closeModal`/`onBgClick`/`updatePointsVisibility`/`renderPointsChips`/`pickPoints`/`submitAdd`/`submitEdit`/`submitDelete`/`setLoading`/`resetLoading`/`showOk`/`showErr`/`getCatValue`/`onCatSel`）は、[../01_index_予定管理.md](../01_index_予定管理.md)で解説した`Plan.js`の同名関数と、ロジックがほぼ**そのままコピーされています**。時間割ページでも右下のFABから予定の追加・編集・削除ができるようにするため、`Plan.js`の該当部分をそのまま持ち込んだと考えられます。詳しい説明は各関数について[../01_index_予定管理.md](../01_index_予定管理.md)の対応する節を参照してください。ここでは**Timetable.js特有の違い**だけを説明します。

### 1.1 `loadPlans()`の違い
```js
async function loadPlans() {
  try {
    const data = await api(`/list_schedule?guild_id=${GUILD_ID}`);
    plans = data.ok ? data.plans : [];
  } catch(e) { plans = []; }
  renderTimetable(); // ★ 月間カレンダーの「予定あり」表示に反映させる
}
```
- `Plan.js`の`loadPlans`は`renderPlans()`（予定一覧の描画）を呼びますが、こちらは`renderTimetable()`を呼びます。これは、取得した予定データを**月間カレンダーの「予定あり」の点**（[01_Timetable.js_その1_週表示と月間カレンダー.md](01_Timetable.js_その1_週表示と月間カレンダー.md)の`buildMonthCalendarHtml`）に反映させるためです。また、`Plan.js`のようなページング（未来分/過去分の分割読み込み）は無く、**一度に全件**取得しています。

### 1.2 `renderChannelOptions()`の違い
```js
function renderChannelOptions() {
  const opts = channels.map(c => `<option value="${escapeAttr(c.name)}">${escapeAttr(c.name)}</option>`).join('');
  document.getElementById('add-subject').innerHTML  = opts || '<option value="">（なし）</option>';
  document.getElementById('edit-subject').innerHTML = '<option value="">— 変更しない —</option>' + opts;
  const ttSubjectEl = document.getElementById('tt-edit-subject');
  if (ttSubjectEl) ttSubjectEl.innerHTML = '<option value="">科目を選択</option>' + opts;
}
```
- 予定管理モーダルの科目セレクトに加えて、**時間割編集モーダルの科目セレクトも同時に更新**しています。1つの関数が、このページにある2種類の科目選択欄をまとめて面倒を見ている形です。

### 1.3 `submitAdd`/`submitEdit`/`submitDelete`の違い（重要）
`Plan.js`の対応する関数と見比べると、次の違いがあります：

- **`requireLoginOrRedirect()`によるログイン確認の呼び出しが、このページの3つの関数には無い**（`Plan.js`は関数の先頭で必ず呼んでいました）。
- **送信するデータに`session_token`と`nickname`が含まれていない**（`Plan.js`は`session.session_token`と`session.nickname`をボディに含めていました）。

```js
const body = { guild_id: GUILD_ID, date, subject, category, content: contentToSend };
...
const res = await api('/add_schedule', { method: 'POST', body: JSON.stringify(body) });
```
- `api()`関数自体はログイン中であれば自動で`Authorization`ヘッダーを付けてくれるため（[01_Timetable.js_その1_週表示と月間カレンダー.md](01_Timetable.js_その1_週表示と月間カレンダー.md)参照）、通信そのものはログイン中なら認証されます。ただし、`Plan.js`のような「未ログインなら送信前にログイン画面へ誘導する」という**事前の親切なガード**が無いため、未ログインのままこのフォームに入力して送信ボタンを押すと、サーバー側に拒否されてから初めて（`showErr`のエラー表示で）気づく形になります。これは`Plan.js`と挙動が異なる箇所で、実装上の統一が取れていない部分と考えられます。

---

## 2. 自作カレンダー（1636〜1722行）

`initCal`/`resetCal`/`renderCal`/`moveCal`/`pickDate`/`toggleCal`は、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`の同名関数とほぼ同じです。1点だけ、このページ独自の処理が`pickDate`に追加されています：

```js
function pickDate(e, id, ds) {
  ...
  // ★ 曜日変更モードで日付を選んだら、入れ替えプレビューを更新
  //   （開始日・終了日どちらを変更してもプレビューは開始日ベースで再描画）
  if ((id === 'tt-edit' || id === 'tt-edit-end') && ttEditMode === 'day-change') {
    const d = calState['tt-edit']?.selected;
    if (d) renderDayChangePreview(d);
  }
}
```
- 時間割編集モーダルの対象日付・終了日カレンダー（`'tt-edit'`/`'tt-edit-end'`）で日付が選ばれたとき、今「曜日変更」モードを見ているなら、[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)の`renderDayChangePreview`を呼んでプレビューを更新します。開始日を変えても終了日を変えても、**プレビュー自体は常に開始日を基準に**再描画される、という注記もコメントにあります。

---

## 3. ドロワー（1724〜1780行）

`openDrawer`/`prefetchOtherPages`/`closeDrawer`と、ドロワーリンクのクリック処理・`pageshow`イベントの処理は、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`と同じ構造です。先読み対象のファイル一覧が、このページ自身（`Timetable.js`）を除いた他ページのリストになっている点だけが違います。

---

## 4. リアルタイム更新（1782〜1871行）

### 4.1 3種類のデータをまとめて監視する
```js
let watchHashes = {
  schedule:  null, // 予定・課題（list_schedule。ttHomeworksもここから取得）
  overrides: null, // 時間割変更・休校（list_timetable）
  terms:     null, // 学期ごとの基本時間割（list_terms）
};
```
- コメントに「以前は課題表示（`ttHomeworks`）用に別途GitHub上の静的JSONも監視していたが、`ttHomeworks`も`list_schedule`（ライブAPI）から取得するようになったため、`schedule`のハッシュ1つで両方の変化を検知できる」という、過去の実装からの整理の経緯が書かれています。

```js
async function hashOfUrl(url, headers) {
  const res = await fetch(url, headers ? { headers } : undefined);
  const txt = await res.text();
  return digestMessage(txt);
}
```
- [../01_index_予定管理.md](../01_index_予定管理.md)の`digestMessage`（SHA-256ハッシュ計算）を使い、指定したURLの内容のハッシュ値を計算する共通関数です。

### 4.2 変更チェック：`checkForUpdates()`（1825〜1853行）
```js
const session = getLoginSession();
const scheduleAuthHeaders = (session && session.session_token) ? { "Authorization": "Bearer " + session.session_token } : null;
const [scheduleHash, overridesHash, termsHash] = await Promise.all([
  hashOfUrl(`${API_BASE}list_schedule?guild_id=${GUILD_ID}`, scheduleAuthHeaders),
  hashOfUrl(`${API_BASE}${TT_API.LIST}?guild_id=${GUILD_ID}`),
  hashOfUrl(`${API_BASE}${TERM_API.LIST}?guild_id=${GUILD_ID}`),
]);
const isFirstCheck = watchHashes.schedule === null;
const changed = !isFirstCheck && (
  scheduleHash  !== watchHashes.schedule  ||
  overridesHash !== watchHashes.overrides ||
  termsHash     !== watchHashes.terms
);
watchHashes = { schedule: scheduleHash, overrides: overridesHash, terms: termsHash };
if (changed) { await refreshWatchedData(); }
```
- `Promise.all`で3つのURLのハッシュ値を**同時に**取得し、そのうち**どれか1つでも**前回と変わっていれば「変更あり」と判定します。3種類をまとめて1つの`checkForUpdates`関数で扱うことで、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`にあったような「予定用」「時間割用」に分かれた複数の監視処理を1本化しています。
- `list_schedule`だけ、ログイン中なら`Authorization`ヘッダーを付けて問い合わせています（未ログインで見えるデータと、ログイン中に見えるデータで内容が違う可能性があるため、ログイン状態に応じた正しいハッシュを計算する必要があるからだと考えられます）。

```js
async function refreshWatchedData() {
  await Promise.all([ loadTTHomeworks(), loadTTOverrides(), loadTerms(), loadPlans() ]);
  renderTimetable();
}
```
- 変更を検知したら、4種類のデータを**すべて**再取得してから、まとめて1回だけ`renderTimetable()`を呼びます。個別にどれが変わったかを区別せず全部を取り直すシンプルな実装で、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`（未来分の予定だけを差分更新する、より細かい実装）と比べると、やや大まかな更新方法と言えます。

### 4.3 SSEとフォールバックポーリング（1855〜1867行）
```js
function startRealtimeUpdates() {
  try {
    const es = new EventSource(`${API_BASE}events?guild_id=${GUILD_ID}`);
    es.onmessage = () => { checkForUpdates(); };
  } catch (e) {}
}
startRealtimeUpdates();
setInterval(checkForUpdates, 10000);
```
- [../01_index_予定管理.md](../01_index_予定管理.md)・[../02_Cardmaker/*](../02_Cardmaker/00_HTML構造とページ全体像.md)で何度も登場したのと同じ、SSEでの即時反映＋10秒間隔のフォールバックポーリングという2段構えの仕組みです。

---

## まとめ

`Timetable.js`は、時間割そのものの機能（週表示・月間カレンダー・時限ごとの上書き・曜日入れ替え・学期管理）を新しく組み立てつつ、予定管理機能については`Plan.js`のコードをほぼそのまま持ち込む、という構成になっていました。時間割特有の部分では、「1日分の処理を部品として切り出し、複数日への一括適用ではそれをループで呼ぶ」「曜日変更を、実は『授業変更』と『1コマ休み』の組み合わせとして内部的に実現する」といった、コードの再利用を意識した設計が随所に見られます。一方で、予定管理部分の`submitAdd`等に見られたように、`Plan.js`側で後から追加された改善（ログイン確認の事前ガードなど）が、こちらのコピーには反映されていない箇所もありました。

他ページも同じ形式で解説していきます。
