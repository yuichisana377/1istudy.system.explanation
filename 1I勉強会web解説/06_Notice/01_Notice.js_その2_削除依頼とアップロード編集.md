# Notice.js その2：削除依頼・アップロード・編集・リアルタイム更新（418〜982行）

[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の続きです。`Notice.js`最後のパートです。

---

## 1. 削除と削除依頼（418〜508行）

```js
async function deleteCurrentNotice() {
  const session = requireLoginOrRedirect();
  if (!session) return;
  if (!currentViewFilename) return;
  const ok = await showAppConfirm({
    title: '削除しますか？', desc: `「${currentViewFilename}」を削除します。この操作は取り消せません。`,
    okLabel: '削除する', danger: true,
  });
  if (!ok) return;

  const btn = document.getElementById('view-delete-btn');
  setBtnBusy(btn, true, '削除中…');
  try {
    const res = await api('/delete_notice', {
      method: 'POST',
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename: currentViewFilename, nickname: session.nickname })
    });
    setBtnBusy(btn, false);
    if (res.ok) {
      closeNoticeModal('view');
      await loadNotices();
    } else if (res.error === 'creator_approval_required') {
      // ★ 追加：投稿者本人以外は直接削除できない（サーバー側の作成者確認機能）。
      //   代わりに投稿者への削除依頼フォームを開く。
      openRequestDeleteModal(currentViewFilename, res.owner_nickname);
    } else {
      showAppAlert({ title: '削除に失敗しました', desc: res.error || '' });
    }
  } catch (e) {
    setBtnBusy(btn, false);
    showAppAlert({ title: 'サーバーに接続できませんでした' });
  }
}
```
- [../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`menuDelete`と同じ「作成者本人確認」の仕組みです。サーバーが`creator_approval_required`エラーを返したら、実際には削除せず、投稿者への削除依頼フォーム（1.1節）を開きます。

### 1.1 削除依頼フォーム（452〜508行）
```js
// ── 削除の確認依頼（投稿者本人以外が削除しようとしたとき） ──
// サーバーが creator_approval_required を返したときに deleteCurrentNotice()
// から呼ばれる。ここでは何も削除せず、理由を添えて /request_delete を叩き、
// 投稿者にDiscordで確認してもらうだけ。
let requestDeleteFilename = null;

function openRequestDeleteModal(filename, ownerNickname) {
  requestDeleteFilename = filename;
  document.getElementById('request-delete-desc').textContent =
    `「${filename}」の投稿者（${ownerNickname || '投稿者'}さん）に削除の確認が必要です。理由を書いて送信すると、投稿者にDiscordで確認が届きます。`;
  document.getElementById('request-delete-reason').value = '';
  const errEl = document.getElementById('request-delete-err');
  errEl.style.display = 'none';
  const btn = document.getElementById('request-delete-submit-btn');
  setBtnBusy(btn, false); btn.disabled = false; btn.textContent = '送信する';
  document.getElementById('modal-request-delete').classList.add('open');
}

async function submitRequestDelete() {
  const session = requireLoginOrRedirect();
  if (!session) return;
  if (!requestDeleteFilename) return;
  const reason = document.getElementById('request-delete-reason').value.trim();
  const errEl = document.getElementById('request-delete-err');
  errEl.style.display = 'none';
  if (!reason) {
    errEl.textContent = '理由を入力してください';
    errEl.style.display = 'block';
    return;
  }
  const btn = document.getElementById('request-delete-submit-btn');
  setBtnBusy(btn, true, '送信中…');
  try {
    const res = await api('/request_delete', {
      method: 'POST',
      body: JSON.stringify({
        guild_id: GUILD_ID, session_token: session.session_token,
        category: 'notice', filename: requestDeleteFilename, reason,
      }),
    });
    if (!res.ok) throw new Error(res.error || '送信に失敗しました');
    setBtnBusy(btn, false);
    closeNoticeModal('request-delete');
    closeNoticeModal('view');
    showAppAlert({
      icon: Icons.html('mailSent', {size:18}),
      title: '削除の確認を送りました',
      desc: res.notified_via === 'web_pending'
        ? '投稿者がDiscord未連携のため、次回サイトを開いたときに確認されます。'
        : '投稿者にDiscordで確認を送りました。承認されると削除されます。',
    });
  } catch (e) {
    setBtnBusy(btn, false);
    errEl.textContent = e.message;
    errEl.style.display = 'block';
  }
}
```
- [../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`submitRequestDelete`と同じ仕組みが、こちらは`category: 'notice'`として使われています。サーバー側の`/request_delete`という同じAPIが、デッキとお知らせの両方の削除依頼を共通で扱っていることが分かります。

---

## 2. アップロードモーダル（510〜588行、676〜688行）

```js
function openUploadModal() {
  isEditingNotice = false;
  editingOriginalFilename = null;

  document.getElementById('upload-filename').value = '';
  document.getElementById('upload-filename').disabled = false;
  noticeImages = {};
  document.getElementById('upload-content').value = '';
  document.getElementById('upload-file-input').value = '';
  document.getElementById('upload-file-input').closest('.field').style.display = '';
  document.getElementById('upload-ok').style.display = 'none';
  document.getElementById('upload-err').style.display = 'none';
  document.getElementById('filename-hint').textContent = '';
  document.getElementById('draft-status').textContent = '';

  document.querySelector('#modal-upload .modal-header h3').textContent = 'お知らせをアップロード';
  document.querySelector('#modal-upload .btn-primary').textContent = 'アップロードする';

  const session = getLoginSession();
  const display = document.getElementById('upload-uploader-display');
  display.textContent = session ? `${session.nickname} さん` : '未ログイン（匿名として投稿されます）';

  switchNoticeTab('edit');
  checkForDraft(DRAFT_KEY_NEW);

  document.getElementById('modal-upload').classList.add('open');
}
```
- 新規投稿用に、全ての入力欄を空にリセットしてからモーダルを開きます。`noticeImages = {};`で、前回の入力で追加した画像の控え（3節）も空にします。投稿者欄には「未ログイン（匿名として投稿されます）」という表示の分岐が残っていますが、現在はページ自体がログイン必須（[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の1節）なので、実際にはログイン中のニックネームが表示されます。送信時にも`submitUpload`内の`requireLoginOrRedirect()`で改めてログインを確認します。

```js
/** 詳細モーダルの「編集する」から呼ばれる：既存の内容をアップロードモーダルに読み込んで編集モードにする */
function openEditModal() {
  if (!currentViewFilename) return;

  isEditingNotice = true;
  editingOriginalFilename = currentViewFilename;
  closeNoticeModal('view');

  document.getElementById('upload-filename').value = currentViewFilename;
  document.getElementById('upload-filename').disabled = false; // ★ 編集時もファイル名（タイトル）を変更可能に
  noticeImages = {};
  document.getElementById('upload-content').value = /\.md$/i.test(currentViewFilename) ? collapseNoticeImages(currentViewContent || '') : (currentViewContent || '');
  document.getElementById('upload-file-input').value = '';
  document.getElementById('upload-file-input').closest('.field').style.display = 'none';
  document.getElementById('upload-ok').style.display = 'none';
  document.getElementById('upload-err').style.display = 'none';
  document.getElementById('filename-hint').textContent = '';
  document.getElementById('draft-status').textContent = '';

  document.querySelector('#modal-upload .modal-header h3').textContent = 'お知らせを編集';
  document.querySelector('#modal-upload .btn-primary').textContent = '更新する';

  const session = getLoginSession();
  const display = document.getElementById('upload-uploader-display');
  display.textContent = session ? `${session.nickname} さん` : '未ログイン（匿名として更新されます）';

  switchNoticeTab('edit');
  checkForDraft(draftKeyForEdit(editingOriginalFilename));

  document.getElementById('modal-upload').classList.add('open');
}
```
- 詳細モーダルの「編集する」から呼ばれ、**同じアップロードモーダルを流用**して、既存の内容を入力欄に反映してから開きます（見た目上はタイトルとボタンの文言だけが「編集」用に変わります）。編集時もファイル名を変えられるようにコメントで明記されているのは、これが「リネーム（3節で解説）」を実現するための重要な設計判断だからです。ファイル選択欄（PCから読み込む）は編集時には不要なので隠されます。
- `.md`のお知らせを編集するときは、本文を`collapseNoticeImages()`に通してから入力欄に入れます。本文に埋め込まれた画像（数MBになりうるdata URI）を`![名前](img:ID)`という短い目印に置き換え、実データは`noticeImages`に控えておく処理です（3節）。こうしないと、巨大な文字列のせいで入力欄の操作が重くなってしまいます。

```js
function onFilenameInput() {
  const hintEl = document.getElementById('filename-hint');
  const filename = document.getElementById('upload-filename').value.trim();
  if (isEditingNotice && editingOriginalFilename && filename && filename !== editingOriginalFilename) {
    hintEl.textContent = `「${editingOriginalFilename}」から名前が変更されます（保存時に移動されます）`;
  } else {
    hintEl.textContent = '';
  }
  scheduleDraftSave();
  refreshNoticePreviewIfOpen();
}
```
- 編集中にファイル名を変えると、「名前が変更されます」というヒントを表示しつつ、[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の下書き自動保存もスケジュールします。`refreshNoticePreviewIfOpen()`は、プレビュータブを開いていればその場で描き直す関数です（拡張子が変わると`.md`として整形するかどうかも変わるため）。

```js
function onLocalFileSelected(e) {
  const file = e.target.files[0];
  if (!file) return;
  const nameEl = document.getElementById('upload-filename');
  if (!nameEl.value.trim()) nameEl.value = file.name;

  const reader = new FileReader();
  reader.onload = () => {
    document.getElementById('upload-content').value = /\.md$/i.test(file.name) ? collapseNoticeImages(reader.result) : reader.result;
    refreshNoticePreviewIfOpen();
  };
  reader.readAsText(file, 'utf-8');
}
```
- PCから`.md`/`.txt`ファイルを選ぶと、`FileReader`（ブラウザ標準の、選ばれたファイルの中身を読み取る機能）でテキストとして読み込み、本文欄に反映します。ファイル名欄が空なら、選んだファイルの名前をそのまま使います。 `.md`ファイルの場合は、中に直接書かれた画像のdata URIも`collapseNoticeImages()`で目印に置き換えてから入力欄に入れます（編集モーダルを開くときと同じ理由です）。

---

## 3. 画像の添付（590〜674行）

`.md`のお知らせには画像を添付できます。サーバーに画像ファイルを別に保存する仕組みは作らず、**画像をdata URI（画像の中身を`data:image/jpeg;base64,...`という文字列にしたもの）としてMarkdown本文の中に直接埋め込む**方式です。お知らせは今まで通り「1つのテキストファイル」のままなので、一覧・削除・リネーム・運用ログなど、既存の仕組みをほとんど変えずに済みます。

入口は`Notice.html`の本文欄の下に追加されたボタンです：
```html
      <div style="margin-top:8px;">
        <button type="button" style="padding:6px 12px;font-size:13px;border:1px solid var(--border);border-radius:var(--r-md);background:var(--bg);color:var(--text);cursor:pointer;font-family:inherit;" onclick="document.getElementById('upload-image-input').click()">画像を追加（.md のみ）</button>
        <input type="file" id="upload-image-input" accept="image/jpeg,image/png,image/webp,image/gif" multiple style="display:none" onchange="onNoticeImagesSelected(event)">
      </div>
```
- ボタンを押すと、隠してある`<input type="file">`（JPEG/PNG/WebP/GIF、複数選択可）が開きます。選んだ画像は`onNoticeImagesSelected`（後述）に渡されます。

```js
// ============================================================
//  画像の添付（本文のMarkdownに data URI として埋め込む）
//  ・編集中のテキスト欄には短い目印 `![名前](img:ID)` だけを置き、実データは
//    noticeImages に保持する（巨大なbase64で入力欄が重くならないように）。
//  ・プレビュー・投稿・下書き保存の時だけ実データに展開する。
// ============================================================
let noticeImages = {};            // ID → data URI
const NOTICE_IMG_MAX_SIDE = 1280; // 長辺の上限px（超えたら縮小）
const NOTICE_IMG_MAX_TOTAL = 6000000; // 本文全体（画像込み）の上限文字数（サーバー側と同値）

function expandNoticeImages(text) {
  return String(text || '').replace(/\]\(img:(\w+)\)/g, (m, id) => noticeImages[id] ? '](' + noticeImages[id] + ')' : m);
}

function collapseNoticeImages(text) {
  return String(text || '').replace(/\]\((data:image\/(?:jpeg|png|webp|gif);base64,[A-Za-z0-9+\/=]+)\)/g, (m, uri) => {
    let id = Object.keys(noticeImages).find(k => noticeImages[k] === uri);
    if (!id) { id = 'i' + Date.now().toString(36) + Math.random().toString(36).slice(2, 6); noticeImages[id] = uri; }
    return '](img:' + id + ')';
  });
}
```
- data URIは1枚で数十万〜数百万文字にもなるため、それをそのまま入力欄（`<textarea>`）に入れると、表示も入力もとても重くなります。そこで**入力欄には`![名前](img:ID)`という短い目印だけを置き、本物のデータは`noticeImages`（ID→data URIの対応表）に控えておく**、という二重持ちにしています。
- `expandNoticeImages()`は目印を本物のdata URIに戻す関数で、プレビュー表示（[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の5節）と投稿（4節）の直前に使います。`noticeImages`に無いIDの目印はそのまま残します。
- `collapseNoticeImages()`はその逆で、本文中のdata URIを見つけるたびに目印へ置き換えます。同じ画像が既に控えてあればそのIDを使い回し、無ければ新しいIDを作って控えます。編集モーダルを開くときと、PCから`.md`ファイルを読み込んだときに使います（2節）。
- 先頭のコメントには「下書き保存の時だけ実データに展開する」とありますが、実際の`saveDraftNow()`は画像入りの本文を**下書き保存しない**作りになっています（[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の6節）。
- `NOTICE_IMG_MAX_TOTAL`（600万文字）は、サーバー側`bot.py`の`NOTICE_MAX_CONTENT_CHARS`と同じ値です（[../../1I勉強会bot解説/17_FlaskAPI_お知らせ/00_お知らせ一覧と詳細取得投稿API.md](../../1I勉強会bot解説/17_FlaskAPI_お知らせ/00_お知らせ一覧と詳細取得投稿API.md)の6節）。

```js
/** 画像ファイルを長辺NOTICE_IMG_MAX_SIDE以内に縮小し、JPEGのdata URIにする（透過PNGは白背景になる） */
async function compressNoticeImage(file) {
  const src = await readFileAsDataURL(file);
  const img = await new Promise((resolve, reject) => {
    const im = new Image();
    im.onload = () => resolve(im);
    im.onerror = () => reject(new Error('decode'));
    im.src = src;
  });
  const scale = Math.min(1, NOTICE_IMG_MAX_SIDE / Math.max(img.naturalWidth, img.naturalHeight));
  const w = Math.max(1, Math.round(img.naturalWidth * scale));
  const h = Math.max(1, Math.round(img.naturalHeight * scale));
  const canvas = document.createElement('canvas');
  canvas.width = w; canvas.height = h;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#fff';
  ctx.fillRect(0, 0, w, h);
  ctx.drawImage(img, 0, 0, w, h);
  return canvas.toDataURL('image/jpeg', 0.8);
}
```
- スマホで撮った写真をそのまま埋め込むと1枚で数MBになるため、**長辺を1280px以内に縮小し、JPEG（画質0.8）に変換**してからdata URIにします。`<canvas>`（ブラウザ上で絵を描ける領域）に描き直して`toDataURL`で書き出す、という手順です。
- JPEGは透明を表現できないため、先に白で塗りつぶしてから描いています（コメントの「透過PNGは白背景になる」）。GIFアニメーションも、この変換で最初の1コマだけの静止画になります。
- `readFileAsDataURL()`（612〜619行）は、`FileReader`でファイルをdata URIとして読み込む処理を`await`で待てる形（Promise）に包んだだけの小さな関数です。

```js
async function onNoticeImagesSelected(e) {
  const files = Array.from(e.target.files || []);
  e.target.value = '';
  if (!files.length) return;
  const filename = document.getElementById('upload-filename').value.trim();
  if (/\.txt$/i.test(filename)) {
    showNoticeErr('upload-err', '画像は .md のお知らせでのみ使えます（ファイル名を .md にしてください）');
    return;
  }
  const ta = document.getElementById('upload-content');
  for (const file of files) {
    if (!/^image\/(jpeg|png|webp|gif)$/i.test(file.type)) {
      showNoticeErr('upload-err', `「${file.name}」は対応していない形式です（JPEG/PNG/WebP/GIF）`);
      continue;
    }
    try {
      const uri = await compressNoticeImage(file);
      const id = 'i' + Date.now().toString(36) + Math.random().toString(36).slice(2, 6);
      noticeImages[id] = uri;
      const alt = file.name.replace(/\.[^.]+$/, '').replace(/[\[\]()]/g, '');
      const tag = `![${alt}](img:${id})`;
      const pos = ta.selectionEnd != null ? ta.selectionEnd : ta.value.length;
      const before = ta.value.slice(0, pos), after = ta.value.slice(pos);
      const pre = before && !before.endsWith('\n') ? '\n\n' : '';
      ta.value = before + pre + tag + '\n\n' + after;
      ta.selectionStart = ta.selectionEnd = (before + pre + tag + '\n\n').length;
    } catch (err) {
      showNoticeErr('upload-err', `「${file.name}」を読み込めませんでした`);
    }
  }
  onContentInput();
  refreshNoticePreviewIfOpen();
}
```
- ファイル名が`.txt`なら画像は使えない、とエラーにします（プレーンテキストのお知らせではMarkdownの画像記法が画像として表示されないため）。
- 対応形式（JPEG/PNG/WebP/GIF）以外は、その1枚だけエラーを出して次の画像に進みます。
- 縮小した画像を`noticeImages`に登録し、**カーソル位置**に`![ファイル名](img:ID)`を挿入します。前後に空行を入れ、画像が独立した段落になるようにしています。ファイル名から`[]()`を取り除いているのは、Markdownの画像記法が崩れないようにするためです。
- 最後に`onContentInput()`（下書き保存の予約）と`refreshNoticePreviewIfOpen()`（プレビューの描き直し）を呼びます。

表示側では、`Style.css`に次のルールが追加されています：
```css
.notice-body.markdown-body img {
  max-width: 100%;
  height: auto;
  border-radius: var(--r-md);
  margin: .4em 0;
}
```
- 大きな画像でも画面幅からはみ出さないよう、幅を本文の幅までに抑えています（`height: auto`で縦横比は保たれます）。画像を含む本文も[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の4節の`renderNoticeBody`（marked→DOMPurify）を通るため、安全対策はそのまま効いています。

---

## 4. アップロード（投稿・更新）の送信：`submitUpload()`（765〜859行）

```js
async function submitUpload() {
  const session = requireLoginOrRedirect();
  if (!session) return;
  const filename = document.getElementById('upload-filename').value.trim();
  let content    = document.getElementById('upload-content').value;

  if (!filename) { showNoticeErr('upload-err', 'ファイル名を入力してください'); return; }
  if (!/\.(md|txt)$/i.test(filename)) { showNoticeErr('upload-err', 'ファイル名は .md か .txt にしてください'); return; }
  if (!content.trim()) { showNoticeErr('upload-err', '内容が空です'); return; }
```
- 拡張子は`.md`または`.txt`のみに限定されています。

### 4.1 送信前のチェック（775〜796行）
```js
  // ★ 画像の目印を実データ（data URI）に戻して送る
  const sendContent = expandNoticeImages(content);
  if (sendContent !== content && /\.txt$/i.test(filename)) { showNoticeErr('upload-err', '画像は .md のお知らせでのみ使えます'); return; }
  if (sendContent.length > NOTICE_IMG_MAX_TOTAL) { showNoticeErr('upload-err', '画像が大きすぎます。枚数を減らしてください'); return; }
```
- その前の775〜791行では、本文に目に見えない特殊な文字（制御文字など）が混ざっていないかを調べ、見つかれば自動で取り除いて投稿するかを確認します（断った場合は`showNoticeBugHighlight`で場所をハイライト表示）。
- そのあと、入力欄の目印を`expandNoticeImages()`で本物の画像データに戻した`sendContent`を作ります。**サーバーに送るのは`content`ではなくこの`sendContent`**です。
- 画像が入っているのにファイル名が`.txt`のとき、また全体が`NOTICE_IMG_MAX_TOTAL`を超えるときは、送信前にエラーにします（サーバー側でも同じ上限で弾きますが、数MBを送ってから断られるより先に止めた方が早いためです）。

### 4.2 上書き確認（798〜811行）
```js
  // ★ 同じ名前（既存の別お知らせと同名）で保存しようとした場合は上書き確認する
  //   ・新規投稿で既存と同名 → 上書きするか確認
  //   ・編集でタイトルを既存の別名に変更 → 上書きするか確認
  //   ・編集で元の名前のまま（変更なし） → 確認不要（通常の更新）
  const excludeName = isEditingNotice ? editingOriginalFilename : null;
  const isDuplicate = notices.some(n => n.filename === filename && n.filename !== excludeName);
  if (isDuplicate) {
    const overwriteOk = await showAppConfirm({
      title: '上書きしますか？',
      desc: `「${filename}」という名前のお知らせは既に存在します。\n上書きしてもよろしいですか？`,
      okLabel: '上書きする', danger: true,
    });
    if (!overwriteOk) return;
  }
```
- 送信しようとしているファイル名が、**自分自身（編集中の元のファイル）を除いて**、既存のどれかと重複していないかチェックします。`excludeName`が「編集中の元のファイル名（変更なしで更新するなら重複チェックに引っかからないように）」または「新規投稿なら`null`（何も除外しない）」となることで、コメントの3つのケースがすべて正しく判定されます。

### 4.3 実際の送信とリネームの実現（813〜859行）
```js
  const uploader = session.nickname;

  const editing = isEditingNotice;
  const btnLabel = editing ? '更新する' : 'アップロードする';

  const btn = document.querySelector('#modal-upload .btn-primary');
  btn.disabled = true;
  btn.innerHTML = `<span class="spinner"></span>${editing ? '更新中…' : 'アップロード中…'}`;
  try {
    // /upload_notice は同名ファイルなら自動的に上書き（更新）してくれるため、
    // 新規投稿・編集どちらもこのエンドポイントを使う
    const res = await api('/upload_notice', {
      method: 'POST',
      body: JSON.stringify({ filename, content: sendContent, uploader, guild_id: GUILD_ID, session_token: session.session_token })
    });
    btn.disabled = false;
    btn.textContent = btnLabel;
    if (res.ok) {
      // ★ 編集でファイル名（タイトル）が変更された場合は、新しい名前で保存した後に古いファイルを削除して「移動」を完成させる
      if (editing && editingOriginalFilename && editingOriginalFilename !== filename) {
        try {
          await api('/delete_notice', {
            method: 'POST',
            body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename: editingOriginalFilename, nickname: uploader })
          });
        } catch (e) {
          // 古いファイルの削除に失敗しても、新しい内容の保存自体は成功しているため処理は続行する
        }
      }
      clearDraftAfterSubmit();
      showNoticeOk('upload-ok');
      if (editing) {
        currentViewContent = sendContent;
        currentViewFilename = filename;
        editingOriginalFilename = filename;
      }
      await loadNotices();
      setTimeout(() => closeNoticeModal('upload'), 700);
    } else {
      showNoticeErr('upload-err', res.error || 'エラーが発生しました');
    }
  } catch (e) {
    btn.disabled = false;
    btn.textContent = btnLabel;
    showNoticeErr('upload-err', 'サーバーに接続できませんでした');
  }
}
```
- コメントに「`/upload_notice`は同名ファイルなら自動的に上書き（更新）してくれるため、新規投稿・編集どちらもこのエンドポイントを使う」とあります。 送る本文は4.1節の`sendContent`（画像の実データ入り）で、編集後に手元へ控える`currentViewContent`も同じ`sendContent`にしています（次にもう一度「編集する」を押したとき、`collapseNoticeImages`で再び目印に戻されます）。
- **お知らせの「リネーム（ファイル名の変更）」という一見単純な操作が、実際には「新しい名前で保存」＋「古い名前を削除」という2段階の操作の組み合わせとして実現されています**。サーバー側のお知らせデータはファイル名をキーにして管理されているため、専用の「名前変更」APIが無く、既存の「アップロード」と「削除」を組み合わせることで擬似的にリネームを表現しています。もし古いファイルの削除が失敗しても、新しい内容の保存自体は既に成功しているため、コメントの通り「処理は続行する」（エラーとして扱わない）という判断がされています。この場合、結果的に新旧2つのファイルが残ってしまう可能性がありますが、内容としての「更新」自体は成功しているため、致命的な失敗とはみなさない、という割り切りです。

---

## 5. UIヘルパーとリアルタイム更新（861〜982行）

`showNoticeOk`/`showNoticeErr`/`closeNoticeModal`/`onBgClickNotice`（864〜880行）は、他ページで何度も見たパターンと同じ、成功・エラー表示とモーダルの閉じ方の共通関数です。

```js
// ============================================================
//  ★ リアルタイム更新（Server-Sent Events）
//  ─────────────────────────────
//  以前は初回読み込み時にしか一覧を取得しておらず、他の人がお知らせを
//  追加・編集・削除しても、ページを開き直すまで反映されなかった。
//  サーバーが実際に常時稼働しているので、変更があった瞬間にpushで
//  知らせてもらい、その場で一覧だけ静かに更新し直す。編集中のフォームや
//  開いているプレビューモーダルはそのまま触らない（一覧の再描画だけ行う）。
//  ・SSE接続が切れていた場合に備え、10秒間隔のフォールバックポーリングも残す。
// ============================================================

let lastNoticesHash = null;

async function checkNoticesUpdate() {
  try {
    const session = getLoginSession();
    const res = await fetch(`${API_BASE}list_notices?guild_id=${GUILD_ID}`, {
      cache: 'no-store',
      headers: session?.session_token ? { 'Authorization': 'Bearer ' + session.session_token } : {},
    });
    const txt = await res.text();
    const hash = await digestMessage(txt);

    // 初回は保存だけ
    if (lastNoticesHash === null) { lastNoticesHash = hash; return; }
    // ハッシュが変わっていなければ何もしない
    if (hash === lastNoticesHash) return;
    lastNoticesHash = hash;

    let data;
    try { data = JSON.parse(txt); } catch (e) { return; }
    if (!data.ok) return;
    notices = data.notices;
    renderNotices();
  } catch (e) {}
}
```
- 他ページと同じSHA-256ハッシュ比較の仕組みですが、コメントに「編集中のフォームや開いているプレビューモーダルはそのまま触らない（一覧の再描画だけ行う）」と明記されている点が重要です。この関数は`renderNotices()`（一覧の再描画）だけを行い、詳細モーダルやアップロードモーダルの中身には一切触れません。もし誰かが編集モーダルを開いて文章を書いている最中に、裏でこの更新が走ってモーダルの中身まで書き換えてしまったら、入力中の内容が失われる事故になりかねないため、影響範囲を一覧表示だけに限定する、という慎重な設計です。

`startRealtimeUpdates()`（919〜926行）は、他ページと同じSSE＋保険のポーリングの二段構成です。2026/11/14からは中身が`startLiveUpdates(checkNoticesUpdate);`の1行になり、SSE接続と保険のポーリング（SSE接続中は60秒に1回、切れていれば10秒に1回、タブが見えていない間は停止）は`Common.js`側でまとめて行います（[../01_index_予定管理.md](../01_index_予定管理.md)の`startLiveUpdates`の説明を参照）。以前あった`setInterval(checkNoticesUpdate, 10000)`は無くなりました。

ドロワー（`openDrawer`/`prefetchOtherPages`/`closeDrawer`とリンクのクリック処理・`pageshow`イベント、931〜978行）は、他ページと同じ構造です。

---

## まとめ

`Notice.js`は、このシリーズの中でも特にセキュリティ（保存型XSS）を意識した実装が目立つページでした。一覧の描画をDOM APIに統一したことに加え、Markdown表示では「サニタイズできないなら整形をあきらめてでも安全側に倒す」という明確な優先順位を持たせている点が特徴的です。また、リネームを「新規保存＋旧ファイル削除」の組み合わせで実現するなど、専用のAPIが無い操作を既存の仕組みの組み合わせで表現する工夫も見られました。

これで`Notice.html`＋`Notice.js`の解説はすべて完了です。続きは[../07_TanCheck/00_TanCheck解説.md](../07_TanCheck/00_TanCheck解説.md)で、単位チェッカーページ（`TanCheck.html`/`TanCheck.js`）を解説します。
