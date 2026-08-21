# Cardmaker.js その8：画像処理・モーダル・遅延読み込みの仕組み（4167〜4588行）

[07_Cardmaker.js_その7_学習モードとクイズ再生.md](07_Cardmaker.js_その7_学習モードとクイズ再生.md)の続きです。

## このパートで出てくる新しい用語

- **`canvas`（キャンバス）**：JSからピクセル単位で絵を描いたり、画像を加工したりできる、ブラウザの機能。画像のリサイズや回転補正はこのキャンバスの上で行われます。
- **EXIF（イグジフ）**：スマホなどで撮った写真データの中に、写真本体とは別に埋め込まれている「付加情報」（撮影日時、カメラの向きなど）。この付加情報の中の「向き（回転）」の情報を読み取ることで、「写真アプリでは正しい向きに見えるのに、Webサイトに載せると横倒しになる」という問題を防げます。
- **`DataView`／`ArrayBuffer`**：画像ファイルの中身を「意味のある文字列」としてではなく、**生のバイナリデータ（0と1の羅列）として直接読み書きする**ための仕組みです。EXIF情報の解析のような、ファイル形式の内部構造を直接読み解く処理で使われます。
- **JPEG圧縮**：画像のファイルサイズを小さくする処理。多少画質を犠牲にする代わりに、通信量やサーバーの容量を節約できます。

---

## 1. 画像の圧縮・向き補正（4167〜4260行）

```js
const IMG_MAX_DIMENSION = 1280;
const IMG_JPEG_QUALITY  = 0.72;
```
- コメントに「GitHub Contents API（1ファイルあたり実用上1MB程度が上限）に収まりやすくするため」とあります。カードに添付する画像は、サーバー側でGitHubリポジトリに保存される仕組みになっているらしく、そのAPIの実用上の制限に収まるよう、長辺を1280pxに縮小し、画質72%のJPEGとして保存する、という具体的な数値がここで決められています。

### 1.1 EXIFの向き情報を読み取る（4174〜4209行）
```js
function getExifOrientation(arrayBuffer) {
  const view = new DataView(arrayBuffer);
  if (view.byteLength < 4 || view.getUint16(0, false) !== 0xFFD8) return 1; // JPEGでない
  ...
}
```
- JPEGファイルは、先頭の2バイトが必ず`0xFFD8`という決まった値になっている、という仕様があり、それをチェックすることで「本当にJPEGファイルか」を確認しています。
- そこから先は、JPEGファイルの内部構造（「マーカー」と呼ばれる区切りの並び）を1つずつ読み進めながら、EXIF情報が入っている`0xFFE1`というマーカーを探し出します。見つかったら`readExifOrientation`で、その中に埋め込まれている「向き」の値（1〜8の数字で、無回転・上下反転・90度回転などを表す）を取り出します。この処理はJPEGファイル形式の仕様を直接読み解いている、かなり低レベルな（生データに近い）コードです。

### 1.2 向きの補正をcanvasの変形として適用する（4211〜4222行）
```js
function applyOrientationTransform(ctx, orientation, width, height) {
  switch (orientation) {
    case 2: ctx.transform(-1, 0, 0, 1, width, 0); break;
    ...
  }
}
```
- EXIFの向きの値（1〜8）ごとに、キャンバスに描く際の「変形行列」（画像を反転させたり回転させたりするための数値の組み合わせ）を適用します。こうすることで、スマホのカメラアプリでは正しい向きに見える写真が、CardMakerに添付したときも正しい向きで表示されるようになります。

### 1.3 実際に画像を圧縮する：`compressImageFile(file)`（4232〜4260行）
```js
if (file.type === 'image/jpeg') {
  const buf = await file.slice(0, 128 * 1024).arrayBuffer();
  orientation = getExifOrientation(buf);
}
```
- EXIF情報はファイルの先頭付近にしか無いため、ファイル全体ではなく最初の128KB分だけを読み取って解析しています（無駄なデータの読み込みを避ける工夫）。

```js
let width = img.naturalWidth || img.width;
let height = img.naturalHeight || img.height;
if (width > IMG_MAX_DIMENSION || height > IMG_MAX_DIMENSION) {
  if (width >= height) { height = Math.round(height * IMG_MAX_DIMENSION / width); width = IMG_MAX_DIMENSION; }
  else { width = Math.round(width * IMG_MAX_DIMENSION / height); height = IMG_MAX_DIMENSION; }
}
const swapDims = orientation >= 5 && orientation <= 8;
const canvas = document.createElement('canvas');
canvas.width  = swapDims ? height : width;
canvas.height = swapDims ? width  : height;
const ctx = canvas.getContext('2d');
ctx.fillStyle = '#ffffff';
ctx.fillRect(0, 0, canvas.width, canvas.height);
applyOrientationTransform(ctx, orientation, width, height);
ctx.drawImage(img, 0, 0, width, height);
return canvas.toDataURL('image/jpeg', IMG_JPEG_QUALITY);
```
- 長辺が1280pxを超えていれば、縦横比を保ったまま縮小するサイズを計算します。
- `swapDims`：EXIFの向きが5〜8（90度・270度回転を含む向き）の場合、見た目上の縦横が入れ替わるため、キャンバスの幅と高さもそれに合わせて入れ替えます。
- `ctx.fillStyle = '#ffffff'; ctx.fillRect(...)`：先にキャンバス全体を白で塗りつぶしています。コメントによれば、これは透過PNG画像をJPEGに変換する際に、透明だった部分が黒くなってしまう問題を防ぐためです（JPEGは透明度を扱えないため、何も塗らないと透明部分が黒として扱われてしまいます）。
- 最後に`canvas.toDataURL('image/jpeg', 0.72)`で、キャンバスの中身を「JPEG形式・画質72%の文字列（data URL、画像データをそのままテキストとして表現したもの）」に変換します。これがカードのデータに直接保存される画像の実体です。

---

## 2. 画像アップロードの入口と一覧表示（4262〜4320行）

```js
function addImage(t) { imgTarget=t; imgContext='editor'; imgInput.click(); }
function addModalImage(t) { imgTarget=t; imgContext='modal'; imgInput.click(); }
```
- 「画像を追加」ボタンは、画面上に隠されている`<input type="file">`（4262行目付近の`imgInput`）を、JSから`.click()`することで間接的にクリックさせます。どの欄（問題文/解答/解説のうちどれ、`t`の値）に、どちらの画面（新規作成フォームか編集モーダルか、`imgContext`）から呼ばれたかを、あらかじめグローバル変数に控えておきます。

```js
imgInput.addEventListener('change', async () => {
  const file = imgInput.files[0]; if (!file||!imgTarget) return;
  const target = imgTarget;
  const context = imgContext;
  imgInput.value = '';
  try {
    const dataUrl = await compressImageFile(file);
    if (context === 'modal') { editImgBuf[target].push(dataUrl); renderModalImgStrip(target); }
    else { imgBuf[target].push(dataUrl); renderImgStrip(target); }
  } catch(e) {
    await showCmAlert({ title: '画像の読み込みに失敗しました', desc: '別の画像で試してください。' });
  }
});
```
- ファイルが選ばれると、1節の圧縮処理を行い、結果をどちらのバッファ（新規作成用の`imgBuf`か、編集モーダル用の`editImgBuf`）に積むかを、控えておいた`context`で振り分けます。

```js
function renderImgThumbStrip(container, imgs, onRemove) {
  container.innerHTML = '';
  imgs.forEach((b, i) => {
    const wrap = document.createElement('div');
    ...
    const img = document.createElement('img');
    img.src = b;
    img.alt = '';
    img.addEventListener('click', () => openImgLightbox(img.src));
    const delBtn = document.createElement('button');
    delBtn.innerHTML = Icons.html('close', {size:12});
    delBtn.addEventListener('click', () => onRemove(i));
    ...
  });
}
```
- 追加した画像のサムネイル（削除ボタン付き）を並べる共通関数です。コメントには、`<img src="${b}">`のようなテンプレート文字列ではなく`document.createElement`で組み立てている理由が書かれています：カードの画像データ（`imgs_q`/`imgs_a`/`imgs_e`）は、サーバーの`save_cards`が中身を検証していないため、共有デッキ経由で他人が仕込んだ文字列が入る可能性があります（新規作成用の`imgBuf`はこのファイル内で作った安全な`data:`URLしか入りませんが、編集モーダル用の`editImgBuf`は既存カード＝他人が作った可能性のあるデータをそのまま引き継ぐため、両方に共通して使うこの関数は、常に安全な方の作り方に統一する必要があります）。

---

## 3. モーダル・ライトボックス・ドロワー（4322〜4387行）

```js
function openModal(id)  { document.getElementById(id).classList.add('open'); }
function closeModal(id) { document.getElementById(id).classList.remove('open'); }
function onOverlayClick(e,id) { if(e.target===document.getElementById(id)) closeModal(id); }
```
- モーダルの開閉は、これまでのページと同じく`open`クラスの付け外しだけのシンプルな仕組みです。

`openImgLightbox(src)`／`closeImgLightbox()`は、サムネイル画像をタップしたときに全画面で拡大表示する「ライトボックス」の開閉です。

ドロワー（`openDrawer`/`closeDrawer`/`prefetchOtherPages`）の仕組みは[01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`とほぼ同じです。先読み対象のファイル一覧だけが、CardMakerからは自分自身（`Cardmaker.js`）を除いた他ページのファイル（`Plan.js`・`Timetable.js`・`StudyLog.js`など）になっています。

---

## 4. バナー通知（4389〜4405行）

```js
function showBanner(msg, bg, color, iconHtml) {
  const banner = document.getElementById('save-ok-banner');
  banner.innerHTML = (iconHtml ? iconHtml + ' ' : '') + esc(msg);
  banner.style.background = bg;
  banner.style.color = color;
  banner.style.display = 'block';
  setTimeout(() => { banner.style.display = 'none'; banner.style.background = '#dcfce7'; banner.style.color = '#166534'; }, 3500);
}
```
- 画面上部に一時的に表示される「保存しました！」のような通知（このドキュメントでも何度も登場した`showBanner`の実体）です。3.5秒後に自動で消え、色も次に使われるときのためにデフォルト（緑系）へ戻しておきます。
- コメントには「`msg`は常にこのファイル内の固定文字列（呼び出し側にユーザー入力を渡す箇所は無い）なので、`innerHTML`で組み立てても安全」とありますが、それでも念のため`esc(msg)`は通しています。

---

## 5. 遅延読み込みチャンクの仕組み本体（4407〜4456行）

これまで何度も登場した「実際の機能は別ファイルに分離されている」という仕組みの、中核となる実装です。

```js
const _chunkPromises = {};
const _chunkDone = new Set();
function loadChunk(name, src) {
  if (_chunkPromises[name]) return _chunkPromises[name];
  _chunkPromises[name] = new Promise((resolve, reject) => {
    const s = document.createElement('script');
    s.src = src;
    s.onload = () => { _chunkDone.add(name); resolve(); };
    s.onerror = () => reject(new Error('chunk load failed: ' + src));
    document.head.appendChild(s);
  });
  return _chunkPromises[name];
}
```
- `document.createElement('script')`で新しい`<script>`タグをその場で作り、`src`（読み込み先のファイル）を指定して`<head>`に追加します。これによりブラウザがそのファイルをダウンロード・実行してくれます。**ビルドツールを使わずに「機能を後から追加でダウンロードする」を実現している方法**です（このサイト全体がビルドツールを使わない設計であることは[00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)でも触れました）。
- `_chunkPromises`は「そのチャンク（機能のかたまり）を読み込むための`Promise`」を名前ごとに覚えておく入れ物です。すでに読み込み中・読み込み済みなら、新しく`<script>`タグを作らず、同じ`Promise`を使い回します（同じファイルを二重に読み込んでしまうのを防ぐ）。

```js
async function loadChunkWithFeedback(name, src) {
  if (_chunkDone.has(name)) return loadChunk(name, src);
  const overlay = document.getElementById('page-nav-loading');
  if (overlay) overlay.classList.add('show');
  try {
    await loadChunk(name, src);
  } finally {
    if (overlay) overlay.classList.remove('show');
  }
}
```
- これまでのパートで何度も見てきた「仮の窓口」関数（`openSearchScreen`など）が呼んでいたのはこちらです。すでに読み込み済みならすぐに終わり、まだであれば「ページ遷移中」と同じローディング表示を出しながら待ちます。ユーザーがボタンを押したのに何も起きていないように見えてしまう事故を防ぐための、見た目のフィードバックです。

```js
async function loadChunksInBackground() {
  const chunks = [
    ['search', '/Cardmaker-search.js'],
    ['quizplay', '/Cardmaker-quizplay.js'],
    ['listview', '/Cardmaker-listview.js'],
    ['csvimport', '/Cardmaker-csvimport.js'],
    ['cardreorder', '/Cardmaker-cardreorder.js'],
  ];
  for (const [name, src] of chunks) {
    try { await loadChunk(name, src); } catch (e) { console.warn('[cardmaker]', e); }
  }
}
```
- こちらは、一覧の初期表示が終わったあとに呼ばれ（呼び出し箇所は[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)の起動処理で説明します）、5つのチャンクを**1つずつ順番に**（同時にではなく）読み込んでいきます。これにより、ユーザーが実際にその機能を使う前に、静かに裏側で準備が整っていきます。もし何かの理由で先に使われてしまっても、2節（各ページの「仮の窓口」）や`loadChunkWithFeedback`が読み込み完了を待ってくれるので、画面が壊れることはありません。

---

## 6. 共通ユーティリティ関数（4458〜4517行）

```js
function esc(s) { return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&#39;'); }
```
- 他ページの`esc()`（[01_index_予定管理.md](../01_index_予定管理.md)参照）と似ていますが、`"`（ダブルクォート）に加えて`'`（シングルクォート）もエスケープしている点が違います。コメントには「`value="${esc(x)}"`のような属性値の中でも安全に使えるようにするため」とあります。HTMLの属性はダブルクォートまたはシングルクォートのどちらでも囲めるため、両方をエスケープしておくことで、どちらの書き方の中に埋め込んでも安全になります。

```js
function setIconText(el, iconHtml, text) {
  el.innerHTML = '';
  if (iconHtml) el.insertAdjacentHTML('beforeend', iconHtml);
  el.appendChild(document.createTextNode((iconHtml ? ' ' : '') + text));
}
```
- 「アイコン＋ユーザー入力のテキスト」という組み合わせを安全に表示するための共通ヘルパーです。アイコン部分（このファイル内に固定で書かれた安全なHTML）は`insertAdjacentHTML`でそのまま挿入し、テキスト部分（デッキ名・フォルダ名などユーザー入力）は`document.createTextNode`（文字列をそのまま「ただの文字」として扱うノードを作る命令）で追加します。こうすることで、テキスト部分に`<script>`のような文字列が入っていても、それはHTMLタグとしては一切解釈されず、ただの文字として表示されるだけになります。

```js
function renderImgList(container, imgs) {
  container.innerHTML = '';
  (imgs || []).forEach(s => {
    const img = document.createElement('img');
    img.src = s;
    ...
  });
}
```
- 学習画面などで、保存済みのカードの画像を表示する共通関数です。2節の`renderImgThumbStrip`と同じ理由（画像データが他人の入力である可能性があるため）で、`<img src="${s}">`のような文字列組み立てを避け、`img.src = s`という**DOMのプロパティへの直接代入**を使っています。コメントによれば、この方法なら`s`の中身が何であっても（たとえ`"`を含んでいても）HTMLの属性として解釈されることはなく、安全です。

```js
function autoResize(el) { el.style.height='auto'; el.style.height=el.scrollHeight+'px'; }
function shake(id) {
  const el=document.getElementById(id); el.style.borderColor='#EF4444'; el.focus();
  setTimeout(()=>el.style.borderColor='',700);
}
```
- `autoResize`：テキストエリアの高さを、入力されている内容の量に合わせて自動的に伸び縮みさせます（`scrollHeight`＝実際の中身の高さを取得し、それに合わせて`height`を再設定する、よく使われるテクニック）。
- `shake`：入力必須の欄が空だったときなどに、その欄を赤い枠にしてフォーカスを当て、0.7秒後に枠の色を戻す関数です（このドキュメントでも何度も登場した「気づかせるための演出」）。

```js
function setBtnLoading(btn, loading, loadingText) {
  if (loading) {
    if (btn.dataset.origHtml === undefined) btn.dataset.origHtml = btn.innerHTML;
    btn.disabled = true;
    btn.classList.add('btn-loading');
    btn.innerHTML = `<span class="btn-spinner"></span>${loadingText ? esc(loadingText) : ''}`;
  } else {
    btn.disabled = false;
    btn.classList.remove('btn-loading');
    if (btn.dataset.origHtml !== undefined) { btn.innerHTML = btn.dataset.origHtml; delete btn.dataset.origHtml; }
  }
}
```
- ボタンを「処理中」の見た目（スピナー＋押せない状態）に切り替える、汎用的な関数です。`btn.dataset.origHtml`（ボタンのDOM要素に紐づく、自由に使えるデータ置き場）に、切り替える前の元の中身を一時的に退避しておき、`loading=false`に戻すときに復元します。コメントには「サーバー通信が終わるまで見た目が何も変わらず、本当に押せたのか分かりにくい、という問題を解消するために使う」とあります。

---

## 7. 入力チェック：バグの元になる特殊文字を弾く（4518〜4586行）

このドキュメントで何度も登場した`warnIfBugChars`の実体です。デッキ名・カードの問題文など、ユーザーが自由入力できるあらゆる場所でこのチェックが呼ばれています。

```js
const BUG_CHAR_RANGES = [
  [0xE000, 0xF8FF],   // 私用領域（外字・gaiji）
  [0xFDD0, 0xFDEF],   // 非文字コードポイント
];
const BUG_CHAR_CODES = new Set([0xFFFE, 0xFFFF]);
const INVISIBLE_CHAR_RANGES = [
  [0x200B, 0x200C], // ゼロ幅スペース、ZWNJ
  [0x2060, 0x2064], // Word Joinerなど
  [0x2066, 0x2069], // 双方向テキストの分離文字
  [0x202A, 0x202E], // 双方向テキストの埋め込み・上書き
  [0xE0000, 0xE007F], // Unicodeタグ文字
];
const INVISIBLE_CHAR_CODES = new Set([0x00AD, 0x180E, 0xFEFF]);
```
- 文字は1つ1つに「コードポイント」という番号が割り当てられています（例えば`A`は65番）。ここでは、その番号の**範囲**を指定することで、「表示上は何も見えないのに存在する文字（ゼロ幅スペースなど）」「Unicodeの仕様上まだ定義されていない、本来は存在しないはずの符号位置」「文字の並び順を偽装できる、見えない制御文字」といった、トラブルの元になりやすい文字たちをまとめてリストアップしています。
- コメントによれば、これらを弾く理由は「他の端末や外部サービス（GitHub等）でエラーや文字化けの原因になるため」です。一方で、絵文字を組み合わせるための特殊な文字（ZWJ）や、日本語の異体字（同じ漢字でも微妙に字形が違うものを指定する仕組み）に使われる文字は、正規の用途なので許可リスト（`isAllowedInvisible`）に入れて例外扱いにしています。

```js
function findBugChars(str) {
  if (!str) return [];
  const found = [];
  for (const ch of String(str)) {
    const cp = ch.codePointAt(0);
    if (isAllowedInvisible(cp)) continue;
    const isCtrl   = cp < 0x20 && ch !== '\t' && ch !== '\n' && ch !== '\r';
    const isDel    = cp === 0x7F;
    const isLoneSg = cp >= 0xD800 && cp <= 0xDFFF; // 孤立サロゲート
    const isRange  = BUG_CHAR_RANGES.some(([s, e]) => cp >= s && cp <= e) || BUG_CHAR_CODES.has(cp);
    const isInvis  = INVISIBLE_CHAR_RANGES.some(([s, e]) => cp >= s && cp <= e) || INVISIBLE_CHAR_CODES.has(cp);
    if ((isCtrl || isDel || isLoneSg || isRange || isInvis) && !found.includes(ch)) found.push(ch);
  }
  return found;
}
```
- `for (const ch of String(str))`という書き方は、絵文字のような「複数のコードポイントで1つの文字として表示されるもの」も正しく1文字ずつ数えられる、`for`ループの特別な書き方です（単純な配列のインデックスアクセスだと、絵文字の途中で分割してしまうことがあります）。
- 文字列を1文字ずつチェックし、「制御文字（タブ・改行・復帰は許可）」「削除文字（DEL）」「孤立サロゲート（本来ペアで使うべき文字コードが片方だけ存在する、壊れたデータ）」「上記の範囲リストに該当するもの」のいずれかに当てはまれば、問題のある文字として記録します（同じ文字を何度も記録しないよう`!found.includes(ch)`でチェック）。

```js
async function warnIfBugChars(str, fieldId) {
  const bad = findBugChars(str);
  if (bad.length === 0) return false;
  await showCmAlert({ title: '使用できない文字が含まれています', desc: `...\n\n該当文字：${bad.join(' ')}\n\n...` });
  if (fieldId) shake(fieldId);
  return true;
}
```
- 問題のある文字が見つかれば、実際にどの文字が該当したかを具体的に示すアラートを出し、対象の入力欄を`shake`で揺らします。戻り値が`true`（＝入力NG）の場合、呼び出し元（`saveCard`など多数の保存処理）はそこで処理を中断します。

---

続きは[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)で、理数記号（分数・ルート）の入力と、リアルタイム更新の仕組みを解説します。
