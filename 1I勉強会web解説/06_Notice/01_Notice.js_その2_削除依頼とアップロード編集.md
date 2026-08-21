# Notice.js その2：削除依頼・アップロード・編集・リアルタイム更新（411〜801行）

[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の続きです。`Notice.js`最後のパートです。

---

## 1. 削除と削除依頼（411〜505行）

```js
async function deleteCurrentNotice() {
  const session = requireLoginOrRedirect();
  if (!session) return;
  const ok = await showAppConfirm({ title: '削除しますか？', desc: `「${currentViewFilename}」を削除します。この操作は取り消せません。`, okLabel: '削除する', danger: true });
  if (!ok) return;
  ...
  const res = await api('/delete_notice', { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename: currentViewFilename, nickname: session.nickname }) });
  ...
  if (res.ok) {
    closeNoticeModal('view');
    await loadNotices();
  } else if (res.error === 'creator_approval_required') {
    openRequestDeleteModal(currentViewFilename, res.owner_nickname);
  } else {
    showAppAlert({ title: '削除に失敗しました', desc: res.error || '' });
  }
}
```
- [../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`menuDelete`と同じ「作成者本人確認」の仕組みです。サーバーが`creator_approval_required`エラーを返したら、実際には削除せず、投稿者への削除依頼フォーム（2節）を開きます。

### 1.1 削除依頼フォーム（448〜505行）
```js
function openRequestDeleteModal(filename, ownerNickname) {
  requestDeleteFilename = filename;
  document.getElementById('request-delete-desc').textContent = `「${filename}」の投稿者（${ownerNickname || '投稿者'}さん）に削除の確認が必要です。理由を書いて送信すると、投稿者にDiscordで確認が届きます。`;
  ...
}
async function submitRequestDelete() {
  ...
  const res = await api('/request_delete', { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, category: 'notice', filename: requestDeleteFilename, reason }) });
  if (!res.ok) throw new Error(res.error || '送信に失敗しました');
  closeNoticeModal('request-delete');
  closeNoticeModal('view');
  showAppAlert({ title: '削除の確認を送りました', desc: res.notified_via === 'web_pending' ? '投稿者がDiscord未連携のため、次回サイトを開いたときに確認されます。' : '投稿者にDiscordで確認を送りました。承認されると削除されます。' });
}
```
- [../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`submitRequestDelete`と同じ仕組みが、こちらは`category: 'notice'`として使われています。サーバー側の`/request_delete`という同じAPIが、デッキとお知らせの両方の削除依頼を共通で扱っていることが分かります。

---

## 2. アップロードモーダル（507〜566行）

```js
function openUploadModal() {
  isEditingNotice = false;
  editingOriginalFilename = null;
  document.getElementById('upload-filename').value = '';
  document.getElementById('upload-filename').disabled = false;
  ...
  document.getElementById('upload-file-input').closest('.field').style.display = '';
  ...
  const session = getLoginSession();
  display.textContent = session ? `${session.nickname} さん` : '未ログイン（匿名として投稿されます）';
  switchNoticeTab('edit');
  checkForDraft(DRAFT_KEY_NEW);
  document.getElementById('modal-upload').classList.add('open');
}
```
- 新規投稿用に、全ての入力欄を空にリセットしてからモーダルを開きます。表示だけは未ログインでも「匿名として投稿されます」と案内していますが、実際に送信しようとすると（`submitUpload`内の`requireLoginOrRedirect()`により）ログインを求められます。

```js
function openEditModal() {
  if (!currentViewFilename) return;
  isEditingNotice = true;
  editingOriginalFilename = currentViewFilename;
  closeNoticeModal('view');
  document.getElementById('upload-filename').value = currentViewFilename;
  document.getElementById('upload-filename').disabled = false; // ★ 編集時もファイル名（タイトル）を変更可能に
  document.getElementById('upload-content').value = currentViewContent || '';
  document.getElementById('upload-file-input').closest('.field').style.display = 'none';
  ...
  document.querySelector('#modal-upload .modal-header h3').textContent = 'お知らせを編集';
  document.querySelector('#modal-upload .btn-primary').textContent = '更新する';
  ...
}
```
- 詳細モーダルの「編集する」から呼ばれ、**同じアップロードモーダルを流用**して、既存の内容を入力欄に反映してから開きます（見た目上はタイトルとボタンの文言だけが「編集」用に変わります）。編集時もファイル名を変えられるようにコメントで明記されているのは、これが「リネーム（3節で解説）」を実現するための重要な設計判断だからです。ファイル選択欄（PCから読み込む）は編集時には不要なので隠されます。

```js
function onFilenameInput() {
  const hintEl = document.getElementById('filename-hint');
  const filename = document.getElementById('upload-filename').value.trim();
  if (isEditingNotice && editingOriginalFilename && filename && filename !== editingOriginalFilename) {
    hintEl.textContent = `「${editingOriginalFilename}」から名前が変更されます（保存時に移動されます）`;
  } else { hintEl.textContent = ''; }
  scheduleDraftSave();
}
```
- 編集中にファイル名を変えると、「名前が変更されます」というヒントを表示しつつ、[00_HTML構造とその1_一覧と詳細表示.md](00_HTML構造とその1_一覧と詳細表示.md)の下書き自動保存もスケジュールします。

```js
function onLocalFileSelected(e) {
  const file = e.target.files[0];
  if (!file) return;
  const nameEl = document.getElementById('upload-filename');
  if (!nameEl.value.trim()) nameEl.value = file.name;
  const reader = new FileReader();
  reader.onload = () => { document.getElementById('upload-content').value = reader.result; };
  reader.readAsText(file, 'utf-8');
}
```
- PCから`.md`/`.txt`ファイルを選ぶと、`FileReader`（ブラウザ標準の、選ばれたファイルの中身を読み取る機能）でテキストとして読み込み、本文欄に反映します。ファイル名欄が空なら、選んだファイルの名前をそのまま使います。

---

## 3. アップロード（投稿・更新）の送信：`submitUpload()`（596〜667行）

```js
if (!filename) { showNoticeErr('upload-err', 'ファイル名を入力してください'); return; }
if (!/\.(md|txt)$/i.test(filename)) { showNoticeErr('upload-err', 'ファイル名は .md か .txt にしてください'); return; }
if (!content.trim()) { showNoticeErr('upload-err', '内容が空です'); return; }
```
- 拡張子は`.md`または`.txt`のみに限定されています。

### 3.1 上書き確認（606〜619行）
```js
// ★ 同じ名前（既存の別お知らせと同名）で保存しようとした場合は上書き確認する
//   ・新規投稿で既存と同名 → 上書きするか確認
//   ・編集でタイトルを既存の別名に変更 → 上書きするか確認
//   ・編集で元の名前のまま（変更なし） → 確認不要（通常の更新）
const excludeName = isEditingNotice ? editingOriginalFilename : null;
const isDuplicate = notices.some(n => n.filename === filename && n.filename !== excludeName);
if (isDuplicate) {
  const overwriteOk = await showAppConfirm({ title: '上書きしますか？', desc: `「${filename}」という名前のお知らせは既に存在します。\n上書きしてもよろしいですか？`, okLabel: '上書きする', danger: true });
  if (!overwriteOk) return;
}
```
- 送信しようとしているファイル名が、**自分自身（編集中の元のファイル）を除いて**、既存のどれかと重複していないかチェックします。`excludeName`が「編集中の元のファイル名（変更なしで更新するなら重複チェックに引っかからないように）」または「新規投稿なら`null`（何も除外しない）」となることで、コメントの3つのケースがすべて正しく判定されます。

### 3.2 実際の送信とリネームの実現（621〜667行）
```js
const res = await api('/upload_notice', { method: 'POST', body: JSON.stringify({ filename, content, uploader, guild_id: GUILD_ID, session_token: session.session_token }) });
...
if (res.ok) {
  // ★ 編集でファイル名（タイトル）が変更された場合は、新しい名前で保存した後に古いファイルを削除して「移動」を完成させる
  if (editing && editingOriginalFilename && editingOriginalFilename !== filename) {
    try {
      await api('/delete_notice', { method: 'POST', body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, filename: editingOriginalFilename, nickname: uploader }) });
    } catch (e) {
      // 古いファイルの削除に失敗しても、新しい内容の保存自体は成功しているため処理は続行する
    }
  }
  clearDraftAfterSubmit();
  ...
}
```
- コメントに「`/upload_notice`は同名ファイルなら自動的に上書き（更新）してくれるため、新規投稿・編集どちらもこのエンドポイントを使う」とあります。
- **お知らせの「リネーム（ファイル名の変更）」という一見単純な操作が、実際には「新しい名前で保存」＋「古い名前を削除」という2段階の操作の組み合わせとして実現されています**。サーバー側のお知らせデータはファイル名をキーにして管理されているため、専用の「名前変更」APIが無く、既存の「アップロード」と「削除」を組み合わせることで擬似的にリネームを表現しています。もし古いファイルの削除が失敗しても、新しい内容の保存自体は既に成功しているため、コメントの通り「処理は続行する」（エラーとして扱わない）という判断がされています。この場合、結果的に新旧2つのファイルが残ってしまう可能性がありますが、内容としての「更新」自体は成功しているため、致命的な失敗とはみなさない、という割り切りです。

---

## 4. UIヘルパーとリアルタイム更新（669〜801行）

`showNoticeOk`/`showNoticeErr`/`closeNoticeModal`/`onBgClickNotice`（672〜688行）は、他ページで何度も見たパターンと同じ、成功・エラー表示とモーダルの閉じ方の共通関数です。

```js
// ★ 以前は初回読み込み時にしか一覧を取得しておらず、他の人がお知らせを
//   追加・編集・削除しても、ページを開き直すまで反映されなかった。
//   サーバーが実際に常時稼働しているので、変更があった瞬間にpushで
//   知らせてもらい、その場で一覧だけ静かに更新し直す。編集中のフォームや
//   開いているプレビューモーダルはそのまま触らない（一覧の再描画だけ行う）。
async function checkNoticesUpdate() {
  ...
  if (lastNoticesHash === null) { lastNoticesHash = hash; return; }
  if (hash === lastNoticesHash) return;
  lastNoticesHash = hash;
  ...
  notices = data.notices;
  renderNotices();
}
```
- 他ページと同じSHA-256ハッシュ比較の仕組みですが、コメントに「編集中のフォームや開いているプレビューモーダルはそのまま触らない（一覧の再描画だけ行う）」と明記されている点が重要です。この関数は`renderNotices()`（一覧の再描画）だけを行い、詳細モーダルやアップロードモーダルの中身には一切触れません。もし誰かが編集モーダルを開いて文章を書いている最中に、裏でこの更新が走ってモーダルの中身まで書き換えてしまったら、入力中の内容が失われる事故になりかねないため、影響範囲を一覧表示だけに限定する、という慎重な設計です。

`startRealtimeUpdates()`（729〜736行）とその後の`setInterval(checkNoticesUpdate, 10000)`（739行）は、他ページと同じSSE＋10秒間隔ポーリングの二段構成です。

ドロワー（`openDrawer`/`prefetchOtherPages`/`closeDrawer`とリンクのクリック処理・`pageshow`イベント、741〜797行）は、他ページと同じ構造です。

---

## まとめ

`Notice.js`は、このシリーズの中でも特にセキュリティ（保存型XSS）を意識した実装が目立つページでした。一覧の描画をDOM APIに統一したことに加え、Markdown表示では「サニタイズできないなら整形をあきらめてでも安全側に倒す」という明確な優先順位を持たせている点が特徴的です。また、リネームを「新規保存＋旧ファイル削除」の組み合わせで実現するなど、専用のAPIが無い操作を既存の仕組みの組み合わせで表現する工夫も見られました。

他ページも同じ形式で解説していきます。
