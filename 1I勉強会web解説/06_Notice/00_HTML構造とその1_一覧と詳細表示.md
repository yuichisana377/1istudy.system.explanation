# お知らせページ — 全体像・一覧・詳細表示・下書き保存（1〜409行）

対象：`bot.1istudy.web/Notice.html`（240行）、`bot.1istudy.web/Notice.js`（801行）。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

## このページの特徴

Markdown対応のお知らせを投稿・閲覧できるページです。**[../01_index_予定管理.md](../01_index_予定管理.md)のコメントで触れた通り、このページのXSS対策の抜けが実際に見つかった経緯があり**、その修正が随所に反映されています。

`Notice.html`は、他ページには無い外部ライブラリを2つ読み込んでいます：
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/marked/12.0.2/marked.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/dompurify/3.1.5/purify.min.js"></script>
```
- **marked**：Markdown形式のテキストをHTMLに変換するライブラリ。
- **DOMPurify**：HTML文字列から、危険なタグ・属性（`<script>`など）を取り除く「消毒（サニタイズ）」ライブラリ。

お知らせ本文はユーザーが自由に書けるMarkdownであり、それをそのままHTML化して表示すると任意のスクリプトを埋め込めてしまうため、**必ずこの2つを組み合わせて使う**（`marked`で変換 →`DOMPurify`で消毒）ことが、このページのセキュリティ上の要になっています。

Notice.htmlは`Dropdown.js`を読み込んでいません（`<select>`要素自体がこのページに無いためです）。

続きは[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)で、削除・アップロード・編集を解説します。

---

## 1. 冒頭：ログイン・アカウント表示・起動（1〜145行）

6〜123行（`API_BASE`〜`renderDrawerAccount`）は[../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md](../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md)の`Timetable.js`と同じ構成です。**このページも、閲覧自体にはログインを要求せず、変更操作（投稿・実行済み切替・削除）の直前にだけ`requireLoginOrRedirect()`でガードします**。

```js
// ★ 追加：変更系の操作（投稿・実行済み切替・削除）はサーバー側もログイン必須に
//   なった（2026/08/19）。以前あった「匿名のまま投稿する」選択肢は廃止し、
//   未ログインなら必ずログイン画面へ誘導する。
function requireLoginOrRedirect() { ... }
```
- コメントに、[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)のシリーズ最初の記事でも触れた「以前は『匿名のまま投稿しますか？』という確認ダイアログで未ログイン投稿を許していたが、廃止した」という経緯が改めて記録されています。

```js
window.addEventListener('load', () => {
  loadNotices();
  prefetchOtherPages();
  checkNoticesUpdate();
  startRealtimeUpdates();
});
```
- 一覧を読み込み、他ページの先読みを行い、変更監視（[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)で解説）を初期化します。

```js
async function api(path, opts = {}) {
  const res = await fetch(API_BASE + path.replace(/^\/+/, ''), { headers: { "Content-Type": "application/json" }, ...opts });
  return res.json();
}
```
- 他ページの`api()`と違い、**ログイン中でも自動で`Authorization`ヘッダーを付けません**。このページの閲覧系API（一覧・詳細取得）はログイン不要なため、必要な場面（`setNoticeDone`・`deleteCurrentNotice`など）では呼び出し側が個別に`session_token`をリクエストボディに含める形になっています。

---

## 2. 一覧の読み込みと描画（147〜225行）

```js
async function loadNotices() {
  document.getElementById('notice-loading').style.display = 'block';
  document.getElementById('notice-content').innerHTML = '';
  try {
    const data = await api(`/list_notices?guild_id=${GUILD_ID}`);
    notices = data.ok ? data.notices : [];
  } catch (e) { notices = []; }
  document.getElementById('notice-loading').style.display = 'none';
  renderNotices();
}
```
- お知らせのメタ情報一覧（本文は含まない、軽量なリスト）を取得します。

```js
// ★ セキュリティ：ファイル名・投稿者ニックネームは利用者が自由入力できる値
//   （HTMLタグを含められる）なので、テンプレート文字列でHTMLを組み立てて
//   innerHTMLに流し込む方式は絶対に使わない（XSSになる）。必ずDOM APIと
//   textContentで組み立て、文字列がそのままHTMLとして解釈されないようにする。
function renderNotices() {
  const el = document.getElementById('notice-content');
  el.innerHTML = ''; // ここは固定文字列のクリアのみなので安全
  ...
  const undone = notices.filter(n => !n.done);
  const done    = notices.filter(n => n.done);
  const ordered = [...undone, ...done];
  const list = document.createElement('div');
  list.className = 'notice-list';
  ordered.forEach(n => {
    const card = document.createElement('div');
    card.className = 'notice-card' + (n.done ? ' notice-done' : '');
    card.addEventListener('click', () => openViewModal(n.filename));
    ...
    const nameEl = document.createElement('span');
    nameEl.className = 'notice-name';
    nameEl.appendChild(document.createTextNode(n.filename));
    ...
    list.appendChild(card);
  });
  el.appendChild(list);
}
```
- コメントに、このページで実際に見つかったXSSの経緯が明記されています。ファイル名や投稿者名は生徒が自由に付けられる値なので、`<img src=x onerror=...>`のようなファイル名を1件アップロードされるだけで、それを見た全員の画面で悪意あるコードが実行されてしまう危険がありました。修正後は、**このファイル全体を通じて`innerHTML`へのテンプレート文字列組み立てを一切使わず**、`document.createElement`＋`textContent`／`document.createTextNode`だけで一覧を組み立てています（[../02_Cardmaker/*](../02_Cardmaker/00_HTML構造とページ全体像.md)で見た多くの箇所と同じ、安全な組み立て方です）。
- 「実行済み」にしたお知らせは、元の新しい順の並びを保ったまま、一覧の一番下にグループごとまとめて表示されます（`[...undone, ...done]`という単純な連結）。

---

## 3. 実行済みの切り替え（227〜269行）

```js
async function setNoticeDone(filename, nextDone) {
  const session = requireLoginOrRedirect();
  if (!session) return;
  const n = notices.find(x => x.filename === filename);
  if (!n) return;
  n.done = nextDone; // ★ 楽観的に即座に反映（サーバー応答を待たず見た目を切り替える）
  renderNotices();
  updateViewDoneBtn();
  try {
    const res = await api('/set_notice_done', { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename, done: nextDone, nickname: session.nickname }) });
    if (!res.ok) throw new Error(res.error || '');
  } catch (e) {
    n.done = !nextDone;
    renderNotices();
    updateViewDoneBtn();
    showAppAlert({ title: '実行済みの切り替えに失敗しました', desc: '通信環境を確認してもう一度お試しください。' });
  }
}
```
- コメントの通り、サーバーの応答を待たずに**先に**見た目を切り替える「楽観的更新」です（[../04_StudyLog/03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](../04_StudyLog/03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)の課題達成が「悲観的更新」だったのとは対照的な選択です。「実行済みチェック」は取り消しが簡単で見た目の即応性が重視される操作、「課題の達成（ポイントが絡む）」は正確さが重視される操作、という違いが設計判断に表れていると考えられます）。もしサーバーへの反映が失敗したら、`n.done = !nextDone`で表示を元に戻し、エラーを伝えます。

`toggleNoticeDoneFromModal()`／`updateViewDoneBtn()`（254〜269行）は、詳細モーダルの「実行済みにする」ボタンからこの関数を呼ぶための橋渡しと、ボタン自体の見た目（テキスト・色）を今の状態に合わせて更新する関数です。

---

## 4. 詳細（プレビュー）モーダル（271〜321行）

```js
async function openViewModal(filename) {
  currentViewFilename = filename;
  document.getElementById('view-filename').textContent = filename;
  ...
  const data = await api(`/get_notice?filename=${encodeURIComponent(filename)}`);
  ...
  if (data.ok) {
    currentViewContent = data.content;
    renderNoticeBody(bodyEl, filename, data.content);
    ...
  } else {
    bodyEl.classList.add('notice-plain');
    bodyEl.textContent = '読み込みに失敗しました: ' + (data.error || '');
  }
}
```
- 一覧ではメタ情報しか持っていないため、実際の本文はモーダルを開いた瞬間に個別取得します（[../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の「一覧は軽量メタ情報のみ、本体は開いたときに取得」という設計と同じ考え方です）。

```js
function renderNoticeBody(bodyEl, filename, content) {
  const isMd = /\.md$/i.test(filename);
  if (isMd && window.marked && window.DOMPurify) {
    bodyEl.classList.add('markdown-body');
    const rawHtml = marked.parse(content, { breaks: true, gfm: true });
    bodyEl.innerHTML = DOMPurify.sanitize(rawHtml);
  } else {
    bodyEl.classList.add('notice-plain');
    bodyEl.textContent = content;
  }
}
```
- コメントに重要な安全対策が書かれています：「DOMPurifyが読み込めていない場合、サニタイズ無しでHTMLを描画する（＝XSS）フォールバックには絶対にしない。その場合はプレーンテキスト表示にとどめる（marked単体では危険なHTMLがそのまま通ってしまうため）」。`marked`（Markdown→HTML変換）だけでは、本文中に直接書かれた`<script>`のような生のHTMLタグもそのまま素通りしてしまいます。もし何らかの理由で外部CDNからの`DOMPurify`の読み込みが失敗した場合、**安全性を犠牲にしてでも表示を優先する**のではなく、Markdownとしての整形をあきらめてただの文字列として表示する、という安全側に倒した設計になっています。`if (isMd && window.marked && window.DOMPurify)`という条件式で、両方のライブラリが揃っているときだけMarkdown表示のパスに進みます。
- `.md`拡張子でなければ、常にプレーンテキスト表示です。

---

## 5. アップロード・編集モーダルのプレビュータブ（323〜349行）

```js
function switchNoticeTab(tab) {
  if (tab === 'preview') {
    const filename = document.getElementById('upload-filename').value.trim();
    const content = textarea.value;
    renderNoticeBody(previewEl, filename, content || '（内容がありません）');
    ...
  } else {
    ...
  }
}
```
- 「編集」「プレビュー」タブの切り替えです。プレビュー表示には4節の`renderNoticeBody`をそのまま再利用しています。入力中のファイル名の拡張子を見て、Markdownとして整形するかどうかを判断するため、実際に投稿する前から見た目を確認できます。

---

## 6. 下書きのローカル一時保存（351〜409行）

投稿・編集の途中で誤ってページを閉じてしまっても、入力内容が消えないようにする機能です。

```js
function draftKeyForEdit(originalFilename) { return 'notice_draft_edit_' + originalFilename; }
function scheduleDraftSave() {
  clearTimeout(draftSaveTimer);
  draftSaveTimer = setTimeout(saveDraftNow, 600);
}
```
- `scheduleDraftSave()`は、入力欄の`oninput`（[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)の`upload-filename`/`upload-content`）から呼ばれます。[../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md](../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md)の検索と同じ**デバウンス**の考え方で、入力のたびに即座に保存するのではなく、最後の入力から0.6秒操作が無ければ保存する、という間引きです。

```js
function saveDraftNow() {
  const filename = document.getElementById('upload-filename').value;
  const content = document.getElementById('upload-content').value;
  if (!filename.trim() && !content.trim()) return; // 空なら保存しない
  const key = isEditingNotice ? draftKeyForEdit(editingOriginalFilename) : DRAFT_KEY_NEW;
  try {
    localStorage.setItem(key, JSON.stringify({ filename, content, ts: Date.now() }));
    statusEl.innerHTML = ... + ' 下書きを自動保存しました（' + new Date().toLocaleTimeString('ja-JP') + '）';
  } catch (e) {}
}
```
- **新規投稿の下書き**（`DRAFT_KEY_NEW`という共通の1つのキー）と、**既存のお知らせを編集中の下書き**（`draftKeyForEdit(元のファイル名)`という、お知らせごとに別々のキー）を使い分けています。新規投稿の下書きは常に1つしか保持できませんが、編集中の下書きは編集対象のお知らせごとに別々に保存されます。

```js
function checkForDraft(key) {
  try {
    const raw = localStorage.getItem(key);
    if (!raw) { document.getElementById('draft-banner').style.display = 'none'; return; }
    pendingDraft = JSON.parse(raw);
    document.getElementById('draft-banner').style.display = 'flex';
  } catch (e) { document.getElementById('draft-banner').style.display = 'none'; }
}
```
- モーダルを開くたびに（[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)の`openUploadModal`/`openEditModal`から呼ばれます）、対応するキーに下書きが残っていないか確認し、あれば[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)で見た「保存されている下書きがあります」バナーを表示します。

`restoreDraft()`（390〜396行）はバナーの「復元する」ボタンから呼ばれ、下書きの内容を入力欄に反映します。`discardDraft()`（398〜403行）は「破棄する」ボタンから、下書き自体を消します。`clearDraftAfterSubmit()`（405〜409行）は、実際に投稿・保存が成功した後に呼ばれ、もう不要になった下書きを片付けます。

---

続きは[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)で、削除依頼・アップロード・編集モーダルの送信処理を解説します。
