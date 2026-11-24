# お知らせページ — 全体像・一覧・詳細表示・下書き保存（1〜416行）

対象：`bot.1istudy.web/Notice.html`（266行）、`bot.1istudy.web/Notice.js`（983行）。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

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

## 1. 冒頭：ログイン・アカウント表示・起動（1〜135行）

冒頭（6〜105行）は`API_BASE`・`GUILD_ID`（この端末が覚えているサーバー。複数サーバー対応で、`localStorage`の`current_guild`から読み、分からなければ`Login.html`へ誘導）・下書きキーの移行処理・ドロワーのアカウント表示などで、[../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md](../03_Timetable/01_Timetable.js_その1_週表示と月間カレンダー.md)の`Timetable.js`とほぼ同じ構成です。以前は閲覧だけならログイン不要でしたが、**現在はページを開いた瞬間に未ログインならログイン画面へ誘導する、全面ログイン必須のページ**になっています。

```js
const SESSION_KEY = 'sl_session';
// ★ 追加（2026/08/24）：以前は閲覧（GET）は誰でもOKだったが、「予定一覧
//   以外はサーバー参加済みの人だけ見られるように」というユーザーの明示的な
//   指示で、Cardmaker.js/StudyLog.js/Quiz.jsと同様、開いた瞬間に未ログイン
//   ならログイン画面へ誘導する全面ログイン必須のページに変更した。
(function() {
  var s = getLoginSession();
  if (!s || !s.session_token) {
    sessionStorage.setItem('post_login_redirect', location.href);
    location.replace(LOGIN_PATH);
  }
})();
```
- コメントの通り、2026/08/24に「予定一覧以外はサーバー参加済みの人だけ見られるように」という方針になり、`Cardmaker.js`などと同じく即座にログイン画面へ転送する即時実行関数が置かれました。
- 変更操作（投稿・実行済み切替・削除）の直前に呼ぶ`requireLoginOrRedirect()`は、現在は全ページ共通の`Common.js`に移されています（以前の「匿名のまま投稿しますか？」という確認ダイアログは廃止済み）。

```js
window.addEventListener('load', () => {
  loadNotices();
  prefetchOtherPages(); // ★ 追加：メニューを開くのを待たず、初期表示後に自動で他ページを裏で先読み
  // ★ 変更監視用のハッシュを、初回読み込みの内容で起点合わせしておく
  //  （比較なしで保存するだけの「初回チェック」）
  checkNoticesUpdate();
  startRealtimeUpdates();
});
```
- 一覧を読み込み、他ページの先読みを行い、変更監視（[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)で解説）を初期化します。

```js
async function api(path, opts = {}) {
  // ★ 追加（2026/08/24）：list_notices/get_notice等の閲覧系APIがログイン
  //   必須になったため、他ページのapi()ヘルパーと同様、ログイン済みなら
  //   常にAuthorizationヘッダを自動で付ける（session_tokenをURLクエリに
  //   載せないためでもある）。
  const session = getLoginSession();
  const headers = Object.assign(
    { "Content-Type": "application/json" },
    (session && session.session_token) ? { "Authorization": "Bearer " + session.session_token } : {},
    opts.headers || {}
  );
  const res = await fetch(API_BASE + path.replace(/^\/+/, ''), { cache: 'no-store', ...opts, headers });
  return res.json();
}
```
- コメントの通り、一覧・詳細取得などの閲覧系APIもログイン必須になったため、ログイン中なら**常に`Authorization: Bearer <session_token>`ヘッダーを自動で付けます**（トークンをURLクエリに載せないためでもあります）。`cache: 'no-store'`で、ブラウザに古い応答が残るのも防いでいます。変更系（`setNoticeDone`など）は、これに加えて従来通りリクエストボディにも`guild_id`・`session_token`を含めています。

---

## 2. 一覧の読み込みと描画（137〜215行）

```js
async function loadNotices() {
  document.getElementById('notice-loading').style.display = 'block';
  document.getElementById('notice-content').innerHTML = '';
  try {
    const data = await api(`/list_notices?guild_id=${GUILD_ID}`);
    notices = data.ok ? data.notices : [];
  } catch (e) {
    notices = [];
  }
  document.getElementById('notice-loading').style.display = 'none';
  renderNotices();
}
```
- お知らせのメタ情報一覧（本文は含まない、軽量なリスト）を、今のサーバー（`guild_id`）を指定して取得します。

```js
// ★ セキュリティ：ファイル名・投稿者ニックネームは利用者が自由入力できる値
//   （HTMLタグを含められる）なので、テンプレート文字列でHTMLを組み立てて
//   innerHTMLに流し込む方式は絶対に使わない（XSSになる）。必ずDOM APIと
//   textContentで組み立て、文字列がそのままHTMLとして解釈されないようにする。
function renderNotices() {
  const el = document.getElementById('notice-content');
  el.innerHTML = ''; // ここは固定文字列のクリアのみなので安全

  if (!notices.length) {
    const empty = document.createElement('div');
    empty.className = 'empty-msg';
    empty.textContent = 'お知らせはまだありません';
    el.appendChild(empty);
    return;
  }

  // ★ 「実行済み」にしたお知らせは一覧の一番下にまとめる
  //   （元々の並び順＝新しい順は、未実行・実行済みそれぞれのグループ内では維持する）
  const undone = notices.filter(n => !n.done);
  const done    = notices.filter(n => n.done);
  const ordered = [...undone, ...done];

  const list = document.createElement('div');
  list.className = 'notice-list';

  ordered.forEach(n => {
    const card = document.createElement('div');
    card.className = 'notice-card' + (n.done ? ' notice-done' : '');
    card.addEventListener('click', () => openViewModal(n.filename));

    const badge = document.createElement('span');
    badge.className = 'notice-badge ' + extBadgeClass(n.ext);
    badge.textContent = (n.ext || '').toUpperCase();

    const nameEl = document.createElement('span');
    nameEl.className = 'notice-name';
    nameEl.appendChild(document.createTextNode(n.filename));
    const metaParts = [];
    if (n.uploader) metaParts.push(`${n.uploader}さん`);
    if (n.uploaded_at) metaParts.push(n.uploaded_at);
    if (metaParts.length) {
      const meta = document.createElement('span');
      meta.className = 'notice-meta';
      meta.textContent = metaParts.join(' ・ ');
      nameEl.appendChild(meta);
    }

    const arrow = document.createElement('span');
    arrow.className = 'notice-arrow';
    arrow.textContent = '›';

    card.appendChild(badge);
    card.appendChild(nameEl);
    card.appendChild(arrow);
    list.appendChild(card);
  });

  el.appendChild(list);
}
```
- コメントに、このページで実際に見つかったXSSの経緯が明記されています。ファイル名や投稿者名は生徒が自由に付けられる値なので、`<img src=x onerror=...>`のようなファイル名を1件アップロードされるだけで、それを見た全員の画面で悪意あるコードが実行されてしまう危険がありました。修正後は、**このファイル全体を通じて`innerHTML`へのテンプレート文字列組み立てを一切使わず**、`document.createElement`＋`textContent`／`document.createTextNode`だけで一覧を組み立てています（[../02_Cardmaker/*](../02_Cardmaker/00_HTML構造とページ全体像.md)で見た多くの箇所と同じ、安全な組み立て方です）。
- 「実行済み」にしたお知らせは、元の新しい順の並びを保ったまま、一覧の一番下にグループごとまとめて表示されます（`[...undone, ...done]`という単純な連結）。

---

## 3. 実行済みの切り替え（217〜259行）

```js
// ★ 「実行済み」の切り替え本体。全員共有の状態として、押した瞬間に
//   一覧の下へ薄く移動する。詳細モーダルの「実行済みにする」ボタン
//   （編集する・削除するの並び）からのみ呼ばれる。
async function setNoticeDone(filename, nextDone) {
  const session = requireLoginOrRedirect();
  if (!session) return;
  const n = notices.find(x => x.filename === filename);
  if (!n) return;
  n.done = nextDone; // ★ 楽観的に即座に反映（サーバー応答を待たず見た目を切り替える）
  renderNotices();
  updateViewDoneBtn();
  try {
    const res = await api('/set_notice_done', {
      method: 'POST',
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename, done: nextDone, nickname: session.nickname }),
    });
    if (!res.ok) throw new Error(res.error || '');
  } catch (e) {
    // ★ サーバーへの反映に失敗した場合は表示も元に戻す
    n.done = !nextDone;
    renderNotices();
    updateViewDoneBtn();
    showAppAlert({ title: '実行済みの切り替えに失敗しました', desc: '通信環境を確認してもう一度お試しください。' });
  }
}
```
- コメントの通り、サーバーの応答を待たずに**先に**見た目を切り替える「楽観的更新」です（[../04_StudyLog/03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](../04_StudyLog/03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)の課題達成が「悲観的更新」だったのとは対照的な選択です。「実行済みチェック」は取り消しが簡単で見た目の即応性が重視される操作、「課題の達成（ポイントが絡む）」は正確さが重視される操作、という違いが設計判断に表れていると考えられます）。もしサーバーへの反映が失敗したら、`n.done = !nextDone`で表示を元に戻し、エラーを伝えます。

```js
// 詳細モーダルの「実行済みにする」ボタンから
function toggleNoticeDoneFromModal() {
  if (!currentViewFilename) return;
  const n = notices.find(x => x.filename === currentViewFilename);
  if (!n) return;
  setNoticeDone(currentViewFilename, !n.done);
}

// 詳細モーダルのボタン表示を、今開いているお知らせの実行済み状態に合わせる
function updateViewDoneBtn() {
  const btn = document.getElementById('view-done-btn');
  if (!btn || !currentViewFilename) return;
  const n = notices.find(x => x.filename === currentViewFilename);
  const isDone = !!(n && n.done);
  btn.innerHTML = isDone ? (Icons.html('check', {size:14}) + ' 実行済み') : '実行済みにする';
  btn.classList.toggle('is-done', isDone);
}
```
`toggleNoticeDoneFromModal()`／`updateViewDoneBtn()`（243〜259行）は、詳細モーダルの「実行済みにする」ボタンからこの関数を呼ぶための橋渡しと、ボタン自体の見た目（テキスト・色）を今の状態に合わせて更新する関数です。

---

## 4. 詳細（プレビュー）モーダル（261〜311行）

```js
async function openViewModal(filename) {
  currentViewFilename = filename;
  document.getElementById('view-filename').textContent = filename;
  document.getElementById('view-meta').textContent = '';
  const bodyEl = document.getElementById('view-body');
  bodyEl.innerHTML = '';
  document.getElementById('view-loading').style.display = 'block';
  document.getElementById('modal-view').classList.add('open');
  updateViewDoneBtn(); // ★ 追加：「実行済みにする」ボタンの表示を、このお知らせの状態に合わせる

  try {
    const data = await api(`/get_notice?guild_id=${GUILD_ID}&filename=${encodeURIComponent(filename)}`);
    document.getElementById('view-loading').style.display = 'none';
    if (data.ok) {
      currentViewContent = data.content;
      renderNoticeBody(bodyEl, filename, data.content);
      const metaParts = [];
      if (data.uploader) metaParts.push(`${data.uploader}さん`);
      if (data.uploaded_at) metaParts.push(data.uploaded_at);
      document.getElementById('view-meta').textContent = metaParts.join(' ・ ');
    } else {
      bodyEl.classList.add('notice-plain');
      bodyEl.textContent = '読み込みに失敗しました: ' + (data.error || '');
    }
  } catch (e) {
    document.getElementById('view-loading').style.display = 'none';
    bodyEl.classList.add('notice-plain');
    bodyEl.textContent = 'サーバーに接続できませんでした';
  }
}
```
- 一覧ではメタ情報しか持っていないため、実際の本文はモーダルを開いた瞬間に個別取得します（[../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の「一覧は軽量メタ情報のみ、本体は開いたときに取得」という設計と同じ考え方です）。

```js
/** .md は GitHub 風に Markdown レンダリング、.txt はプレーンテキスト表示 */
function renderNoticeBody(bodyEl, filename, content) {
  const isMd = /\.md$/i.test(filename);
  // ★ セキュリティ：DOMPurifyが読み込めていない場合、サニタイズ無しでHTMLを
  //   描画する（＝XSS）フォールバックには絶対にしない。その場合はプレーン
  //   テキスト表示にとどめる（marked単体では危険なHTMLがそのまま通ってしまうため）。
  if (isMd && window.marked && window.DOMPurify) {
    bodyEl.classList.remove('notice-plain');
    bodyEl.classList.add('markdown-body');
    const rawHtml = marked.parse(content, { breaks: true, gfm: true });
    bodyEl.innerHTML = DOMPurify.sanitize(rawHtml);
  } else {
    bodyEl.classList.remove('markdown-body');
    bodyEl.classList.add('notice-plain');
    bodyEl.textContent = content;
  }
}
```
- コメントに重要な安全対策が書かれています：「DOMPurifyが読み込めていない場合、サニタイズ無しでHTMLを描画する（＝XSS）フォールバックには絶対にしない。その場合はプレーンテキスト表示にとどめる（marked単体では危険なHTMLがそのまま通ってしまうため）」。`marked`（Markdown→HTML変換）だけでは、本文中に直接書かれた`<script>`のような生のHTMLタグもそのまま素通りしてしまいます。もし何らかの理由で外部CDNからの`DOMPurify`の読み込みが失敗した場合、**安全性を犠牲にしてでも表示を優先する**のではなく、Markdownとしての整形をあきらめてただの文字列として表示する、という安全側に倒した設計になっています。`if (isMd && window.marked && window.DOMPurify)`という条件式で、両方のライブラリが揃っているときだけMarkdown表示のパスに進みます。
- `.md`拡張子でなければ、常にプレーンテキスト表示です。

---

## 5. アップロード・編集モーダルのプレビュータブ（317〜345行）

```js
function refreshNoticePreviewIfOpen() {
  const previewBtn = document.getElementById('tab-preview-btn');
  if (previewBtn && previewBtn.classList.contains('active')) switchNoticeTab('preview');
}

function switchNoticeTab(tab) {
  const editBtn = document.getElementById('tab-edit-btn');
  const previewBtn = document.getElementById('tab-preview-btn');
  const textarea = document.getElementById('upload-content');
  const previewEl = document.getElementById('upload-preview');

  if (tab === 'preview') {
    const filename = document.getElementById('upload-filename').value.trim();
    const content = textarea.value;
    // renderNoticeBody は .md 拡張子のときのみ Markdown レンダリング、それ以外はプレーン表示
    previewEl.innerHTML = '';
    previewEl.className = 'notice-body';
    renderNoticeBody(previewEl, filename, expandNoticeImages(content) || '（内容がありません）');
    previewEl.style.display = 'block';
    textarea.style.display = 'none';
    previewBtn.classList.add('active');
    editBtn.classList.remove('active');
  } else {
    previewEl.style.display = 'none';
    textarea.style.display = 'block';
    editBtn.classList.add('active');
    previewBtn.classList.remove('active');
  }
}
```
(m) => (m)

---

## 6. 下書きのローカル一時保存（347〜416行）

投稿・編集の途中で誤ってページを閉じてしまっても、入力内容が消えないようにする機能です。

```js
function draftKeyForEdit(originalFilename) {
  const key = 'notice_draft_edit_' + GUILD_ID + '_' + originalFilename;
  // ★ 修正：DRAFT_KEY_NEWと同じ移行漏れ（旧キー 'notice_draft_edit_' + originalFilename）。
  //   呼ばれるたびに（該当ファイルの下書きが無ければ何もしないので軽い）移行を試みる。
  _migrateNoticeDraftKey('notice_draft_edit_' + originalFilename, key);
  return key;
}

function scheduleDraftSave() {
  clearTimeout(draftSaveTimer);
  draftSaveTimer = setTimeout(saveDraftNow, 600);
}
```
- `draftKeyForEdit()`は、編集中のお知らせごとの下書きキーを作ります。複数サーバー対応でキーに`GUILD_ID`を含めるようになったため、古い形式のキーに残っている下書きを新しいキーへ移す`_migrateNoticeDraftKey()`も呼んでいます。
- `scheduleDraftSave()`は、入力欄の`oninput`（[../01_index_予定管理.md](../01_index_予定管理.md)の`upload-filename`/`upload-content`）から呼ばれます。[../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md](../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md)の検索と同じ**デバウンス**の考え方で、入力のたびに即座に保存するのではなく、最後の入力から0.6秒操作が無ければ保存する、という間引きです。

```js
function saveDraftNow() {
  const filename = document.getElementById('upload-filename').value;
  const rawContent = document.getElementById('upload-content').value;
  const content = rawContent;
  const statusEl = document.getElementById('draft-status');

  if (!filename.trim() && !content.trim()) return; // 空なら保存しない

  const key = isEditingNotice ? draftKeyForEdit(editingOriginalFilename) : DRAFT_KEY_NEW;
  // 画像入りは数MBになり localStorage（他機能と共用）を圧迫するため、下書き保存しない
  if (expandNoticeImages(rawContent) !== rawContent || /data:image\//.test(rawContent) || rawContent.length > 200000) {
    try { localStorage.removeItem(key); } catch (e) {}
    statusEl.textContent = '画像を含むお知らせは下書き自動保存の対象外です';
    return;
  }
  try {
    localStorage.setItem(key, JSON.stringify({ filename, content, ts: Date.now() }));
    statusEl.innerHTML = Icons.html('save', {size:14}) + ' 下書きを自動保存しました（' + new Date().toLocaleTimeString('ja-JP') + '）';
  } catch (e) {
    // localStorage が使えない環境では何もしない
  }
}
```
- **新規投稿の下書き**（`DRAFT_KEY_NEW`＝サーバーごとに1つのキー）と、**既存のお知らせを編集中の下書き**（`draftKeyForEdit(元のファイル名)`という、お知らせごとに別々のキー）を使い分けています。新規投稿の下書きは常に1つしか保持できませんが、編集中の下書きは編集対象のお知らせごとに別々に保存されます。
- **画像入りの本文は下書き保存しません**。画像はdata URIとして数MBになることがあり、他の機能と共用している`localStorage`（容量はおおむね5MB程度）を圧迫してしまうためです。編集欄に画像の目印がある（`expandNoticeImages`で中身が変わる）、data URIが直接書かれている、または20万文字を超える場合は、そのキーの古い下書きも消したうえで「画像を含むお知らせは下書き自動保存の対象外です」と表示して終わります。

```js
function checkForDraft(key) {
  try {
    const raw = localStorage.getItem(key);
    if (!raw) { document.getElementById('draft-banner').style.display = 'none'; return; }
    pendingDraft = JSON.parse(raw);
    document.getElementById('draft-banner').style.display = 'flex';
  } catch (e) {
    document.getElementById('draft-banner').style.display = 'none';
  }
}
```
- モーダルを開くたびに（[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)の`openUploadModal`/`openEditModal`から呼ばれます）、対応するキーに下書きが残っていないか確認し、あれば[../01_index_予定管理.md](../01_index_予定管理.md)で見た「保存されている下書きがあります」バナーを表示します。

```js
function restoreDraft() {
  if (!pendingDraft) return;
  document.getElementById('upload-filename').value = pendingDraft.filename || '';
  document.getElementById('upload-content').value = pendingDraft.content || '';
  onFilenameInput();
  document.getElementById('draft-banner').style.display = 'none';
}

function discardDraft() {
  const key = isEditingNotice ? draftKeyForEdit(editingOriginalFilename) : DRAFT_KEY_NEW;
  try { localStorage.removeItem(key); } catch (e) {}
  pendingDraft = null;
  document.getElementById('draft-banner').style.display = 'none';
}

function clearDraftAfterSubmit() {
  const key = isEditingNotice ? draftKeyForEdit(editingOriginalFilename) : DRAFT_KEY_NEW;
  try { localStorage.removeItem(key); } catch (e) {}
  document.getElementById('draft-status').textContent = '';
}
```
`restoreDraft()`（397〜403行）はバナーの「復元する」ボタンから呼ばれ、下書きの内容を入力欄に反映します。`discardDraft()`（405〜410行）は「破棄する」ボタンから、下書き自体を消します。`clearDraftAfterSubmit()`（412〜416行）は、実際に投稿・保存が成功した後に呼ばれ、もう不要になった下書きを片付けます。

---

続きは[01_Notice.js_その2_削除依頼とアップロード編集.md](01_Notice.js_その2_削除依頼とアップロード編集.md)で、削除依頼・アップロード・編集モーダルの送信処理を解説します。
