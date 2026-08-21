# Cardmaker.js その9：数式（理数モード）とリアルタイム更新（4588〜5117行）

[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)の続きです。`Cardmaker.js`本体の最後のパートになります。

---

## 1. 理数モードの設計方針（4588〜4602行のコメントより）

CardMakerでは、分数やルートを入力するときに`\(\sqrt{4}\)`のような**生のLaTeX記法**（KaTeXという数式表示ライブラリが理解できる、専門的な書き方）をユーザーに直接書かせるのではなく、`√(4)`や`(3)/(4)`のような**読みやすい簡易記法**だけを目にさせる、という方針を取っています。

- 入力欄・保存されるデータには、常にこの簡易記法だけが使われます。
- 実際にきれいな見た目（分数の横線、伸びる根号など）で描画したいときだけ、裏側で簡易記法をLaTeXに変換してからKaTeXに渡します。
- 既に古い形式（生のLaTeX）で保存されている過去のカードも、変換処理はパターンが一致しなければそのまま素通りするため、壊れずに今まで通り表示されます（後方互換）。

---

## 2. 簡易記法をLaTeXに変換する（4604〜4689行）

```js
function findMatchingParen(s, openIdx) {
  let depth = 0;
  for (let i = openIdx; i < s.length; i++) {
    if (s[i] === '(') depth++;
    else if (s[i] === ')') { depth--; if (depth === 0) return i; }
  }
  return -1;
}
```
- ある`(`に対応する`)`の位置を探す関数です。`(`が出るたびに`depth`（深さ）を1増やし、`)`が出るたびに1減らすことで、`((1)/(2))`のようにカッコが入れ子になっていても、正しく「対応するカッコ」を見つけられます（`depth`が0に戻った瞬間が、最初の`(`に対応する`)`）。

`simpleMathToLatexRaw(s)`（4617〜4648行）と`simpleMathToLatex(raw)`（4653〜4689行）は、ほぼ同じロジックの2段構えです：
- `√(...)`や`∛(...)`（ルート・立方根）を見つけたら、中身を`\sqrt{...}`に変換。
- `(分子)/(分母)`という形を見つけたら、`\frac{分子}{分母}`に変換。
- **中身（ルートの中・分数の分子/分母）も、もし数式が入っていれば再帰的に同じ変換をかける**（`simpleMathToLatexRaw`が自分自身をもう一度呼び出している部分）。これにより「ルートの中に分数がある」のような入れ子の数式も正しく変換できます。
- 該当しない部分の文字はそのままコピーします（`out += s[i]`）。

2つの関数の違いは、外側の`simpleMathToLatex`だけが変換結果を`\( ... \)`（KaTeXが「ここは数式です」と認識するための区切り記号）で囲む点です。内側（`simpleMathToLatexRaw`）で同じ区切り記号を付けてしまうと、KaTeXの引数の中に区切り記号が紛れ込んで表示が壊れてしまうため、あえて区切り記号を付けない「素の」変換として別に用意されています。

```js
function setMathText(el, raw) {
  if (!el) return;
  el.textContent = simpleMathToLatex(raw || '');
  if (window.renderMathInElement) {
    try {
      renderMathInElement(el, { delimiters: [{ left: '\\(', right: '\\)', display: false }], throwOnError: false });
    } catch (e) {}
  }
}
```
- まず変換結果をプレーンテキストとして要素に入れ、そのあとKaTeX（`renderMathInElement`、[00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)で触れた外部ライブラリ）に「`\(`〜`\)`で囲まれた部分だけを数式として描画してください」とお願いします。
- `window.renderMathInElement`が存在するかを毎回チェックしているのは、外部CDNからの読み込みに失敗した場合（オフラインなど）でも、その場合はただのテキストとして表示され続け、ページ全体が壊れないようにするためです。

---

## 3. 一覧などでの簡易プレビュー（4707〜4732行）

```js
function mathToPlainText(raw) {
  if (raw == null) return '';
  let s = String(raw);
  s = s.replace(/\\\(|\\\)/g, '');
  s = s.replace(/\\sqrt\[(.*?)\]\{([^{}]*)\}/g, (m, n, a) => `${n}√(${a})`);
  s = s.replace(/\\sqrt\{([^{}]*)\}/g, (m, a) => `√(${a})`);
  for (let i = 0; i < 3; i++) {
    s = s.replace(/\\frac\{([^{}]*)\}\{([^{}]*)\}/g, (m, a, b) => `(${a})/(${b})`);
  }
  s = s.replace(/\^\{([^{}]*)\}/g, (m, a) => a.length === 1 && MATH_SUP_MAP[a] ? MATH_SUP_MAP[a] : `^${a}`);
  s = s.replace(/_\{([^{}]*)\}/g, (m, a) => a.length === 1 && MATH_SUB_MAP[a] ? MATH_SUB_MAP[a] : `_${a}`);
  return s;
}
```
- これは2節の逆方向、**LaTeX形式（旧データ）を人が読みやすいプレーンテキストに変換する**関数です。カード一覧のプレビューのように、KaTeXによる本格的な描画ができない（改行やスタック表示に対応していない）場所で使われます。
- 新しい形式（`√(4)`など）はもともと読みやすい形なのでそのまま素通りし、古い形式（`\sqrt{4}`など）だけをここで変換します。
- `for (let i = 0; i < 3; i++)`で同じ置換を3回繰り返しているのは、分数の中にさらに分数が入っている（ネストしている）場合に対応するためです。正規表現による一括置換は入れ子構造を1回では処理しきれないため、外側から順に複数回繰り返して少しずつ内側まで変換していく、という力技のアプローチです。
- `^{2}`のような上付き文字は、1文字だけなら`MATH_SUP_MAP`（あらかじめ用意した「上付き文字っぽく見えるUnicode文字」の対応表、例：`2`→`²`）に置き換え、2文字以上ならそのまま`^2`のような表記で我慢する、という割り切った処理になっています。

`setSimpleMathText(el, raw)`（4729〜4732行）はこの関数を使った表示用のヘルパーで、KaTeXでの本格的な描画をしない、崩れにくい表示が必要な場所で使われます。

---

## 4. 理数記号パレット（入力補助UI）（4734〜4880行）

```js
const MATH_PAD_HTML = (function(){
  const supKeys = ['⁰','¹','²','³',...];
  const subKeys = ['₀','₁','₂','₃',...];
  const symKeys = ['±','∓','×','÷','≤','≥','≠','≈','∞','π','θ','°','∑','∫'];
  const keyBtn = c => `<button type="button" class="math-key" data-ch="${c}">${c}</button>`;
  return `...パレット全体のHTML...`;
})();
```
- 「理数記号」ボタンを押すと開くパレットのHTML全体を、即時実行関数で1回だけ組み立てて`MATH_PAD_HTML`という定数に入れておきます。ボタン類はすべて固定の記号なので`esc()`は不要（ユーザー入力を扱っていない）です。

```js
function initMathPads() {
  document.querySelectorAll('.math-pad').forEach(pad => {
    if (pad.dataset.built) return;
    pad.innerHTML = MATH_PAD_HTML;
    pad.dataset.built = '1';
    const target  = document.getElementById(pad.dataset.target);
    const preview = pad.querySelector('.math-preview');
    if (target && preview) {
      const update = () => setMathText(preview, target.value);
      target.addEventListener('input', update);
      pad._mathUpdate = update;
    }
    if (target) attachInlineSimplePreview(target);
  });
}
```
- ページ内には「問題文用」「解答用」「解説用」など複数の理数パレットが存在します（HTML側で`data-target`属性により、それぞれどの入力欄に対応するかが指定されています）。この関数は、まだ中身を組み立てていないパレット（`pad.dataset.built`が無いもの）だけに`MATH_PAD_HTML`を流し込み、二重に初期化しないようにしています。
- 各パレットの「プレビュー」欄が、対応する入力欄の`input`イベント（何か入力されるたび）に合わせて自動更新されるよう、リスナーを登録します。

```js
function attachInlineSimplePreview(target) {
  if (!target || target.dataset.simplePreviewAttached) return;
  target.dataset.simplePreviewAttached = '1';
  const preview = document.createElement('div');
  ...
  target.insertAdjacentElement('afterend', preview);
  const update = () => {
    const plain = mathToPlainText(target.value);
    preview.textContent = plain === target.value ? '' : plain;
  };
  target.addEventListener('input', update);
  update();
}
```
- 理数パレットを開かなくても、入力欄のすぐ下に「今の入力内容がどんな数式になるか」の軽量なプレビューを常時表示する機能です。`mathToPlainText`で変換した結果が、変換前の元の文字列と**同じ**なら（＝数式記法を含んでいない、ただの普通の文章なら）、プレビューは空欄のままにします（同じ内容を二重に表示しないための工夫）。

```js
function mathInsertChar(el, ch) { ... }
function mathInsertWrap(el, openStr, closeStr) { ... }
function mathInsertFraction(el) { ... }
```
- パレットのボタンを押したときに、実際に入力欄へ文字を挿入する3種類の関数です。すべて「選択範囲があればそこを対象にする、無ければカーソル位置に挿入する」という、テキストエディタの一般的な挙動を再現しています。`el.selectionStart`/`el.selectionEnd`で今選択されている範囲を取得し、その前後の文字列と挿入したい文字列をつなげて新しい`value`を組み立てます。
- `mathInsertFraction`だけ少し特殊で、選択していた文字列をそのまま分子にして`(選択部分)/(`という形にし、カーソルを分母の位置（選択が無ければ分子の位置）に移動させます。分数を作るときに「選択してからボタンを押すと、選択した内容がそのまま分子になる」という使い勝手を実現しています。

```js
document.addEventListener('click', function(e) {
  const btn = e.target.closest('.math-key');
  if (!btn) return;
  const pad = btn.closest('.math-pad');
  if (!pad) return;
  const target = document.getElementById(pad.dataset.target);
  if (!target) return;
  switch (btn.dataset.action) {
    case 'frac': mathInsertFraction(target); break;
    case 'sqrt': mathInsertWrap(target, '√(', ')'); break;
    case 'cbrt': mathInsertWrap(target, '∛(', ')'); break;
    default:     mathInsertChar(target, btn.dataset.ch || '');
  }
});
```
- パレットのボタンは複数箇所（各理数パレットの中）に同じクラス名で存在しますが、1つ1つに個別のクリックリスナーを付けるのではなく、`document`全体に1つだけリスナーを登録し、クリックされた要素が`.math-key`（パレットのボタン）かどうかを`closest`で判定する、という**イベント委任（イベントデリゲーション）**という手法を使っています。これにより、後からパレットが動的に増えても、改めてリスナーを付け直す必要がありません。

---

## 5. 起動処理（4882〜4885行）

```js
renderDeckList().then(() => { initPickModeFromUrl(); jumpToDeckFromUrl(); });
loadChunksInBackground();
prefetchOtherPages();
```
- `renderDeckList()`（デッキ一覧の取得・描画）が終わったら、クイズ選択モードの初期化とURLからのディープリンク処理（6節）を行います。
- それと並行して（`await`せずに）、他の機能チャンクの背景読み込み（[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)）と、他ページの先読みも同時に始めます。

---

## 6. Discord通知からのディープリンク：`jumpToDeckFromUrl()`（4887〜4923行）

Discordに流れる通知のリンク（例：`Cardmaker.html?deck=set_xxxx.json`）から直接開かれた場合、該当のデッキが入っているフォルダまで自動で移動し、そのデッキをハイライト表示する機能です。

```js
const targetFilename = params.get('deck');
if (!targetFilename) return;
history.replaceState(null, '', location.pathname + location.hash);
const deck = decks.find(d => d.filename === targetFilename);
if (!deck) return;
openFolder(deck.folderId || null);
requestAnimationFrame(() => {
  const grid = document.getElementById('deck-grid');
  const el = grid && grid.querySelector(`[data-key="deck:${CSS.escape(targetFilename)}"]`);
  if (!el) return;
  el.scrollIntoView({ behavior: 'smooth', block: 'center' });
  el.style.boxShadow = '0 0 0 3px #3b82f6, 0 4px 14px rgba(59,130,246,0.35)';
  el.style.transform = 'scale(1.02)';
  setTimeout(() => { el.style.boxShadow = ''; el.style.transform = ''; }, 2200);
});
```
- `history.replaceState`でURLから`?deck=...`を消しておくことで、あとでブックマークしたり再読み込みしたりしても、毎回同じ場所へ自動的に飛ばされ続けることを防いでいます。
- `CSS.escape(targetFilename)`：ファイル名にCSSセレクタとして特別な意味を持つ文字（`.`や`#`など）が含まれていても、`querySelector`が正しく動くように安全にエスケープする、ブラウザ標準の命令です。
- 見つけたデッキの要素まで滑らかにスクロールし、青い枠と拡大効果で2.2秒間だけハイライトします。

---

## 7. リアルタイム更新（4925〜5113行）

### 7.1 SSEでの即時反映（4925〜4947行）
```js
function startRealtimeUpdates() {
  try {
    const es = new EventSource(`${API_BASE}events?guild_id=${GUILD_ID}`);
    es.onmessage = () => { checkCardsUpdate(); checkFoldersUpdate(); checkOrderUpdate(); };
  } catch (e) {}
}
startRealtimeUpdates();
```
- [01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`で説明したSSE（Server-Sent Events）と同じ仕組みです。サーバーから「何か変わった」という通知が届くたびに、カード・フォルダ・並び順の3種類を**まとめて**チェックし直します。コメントによれば、サーバー側は「何が変わったか」の詳細までは教えてくれないため、念のため3つとも確認しに行きますが、実際に変わっていないものはこのあと説明するハッシュ比較で自然にスキップされるので、無駄にはなりません。

### 7.2 ハッシュ比較による変更検知（4949〜5000行、5040〜5095行）
`checkCardsUpdate()`／`checkFoldersUpdate()`／`checkOrderUpdate()`は、[01_index_予定管理.md](../01_index_予定管理.md)の`checkScheduleUpdate()`と同じ考え方（SHA-256ハッシュを前回と比較し、変わっていなければ何もしない）を、それぞれ`list_cards`（デッキのメタ情報）・`list_folders`（フォルダ一覧）・`list_order`（並び順）に対して行います。

```js
async function checkCardsUpdate() {
  ...
  await fetchAndMergeDecks();
  const activeScreen = document.querySelector('.screen.active')?.id;
  if (activeScreen === 'screen-list') { renderDeckListUI(); }
}
```
- 変更を検知したら、まずデータだけはバックグラウンドで`decks`／`localStorage`に反映します。ただし、**画面の再描画は一覧画面（`screen-list`）を見ているときだけ**行います。コメントには「編集中／プレイ中の画面はそのままにして、リロードもしない。データは反映されているので、次に一覧へ戻ったときには最新の状態になっている」とあります。今まさにカードを編集している最中に、裏で受け取った他人の更新で画面が突然作り直されてしまうと、入力中の内容が失われかねないため、このような使い分けになっています。

各チェック関数のあとには、SSEが機能しなかった場合の保険として`setInterval(..., 10000)`（10秒ごと）のポーリングも併設されています。

### 7.3 ページに戻ってきたときの強制リフレッシュ（5002〜5039行）
```js
let isForceRefreshing = false;
async function forceRefreshOnReturn() {
  if (isForceRefreshing) return;
  isForceRefreshing = true;
  try {
    await Promise.all([fetchAndMergeDecks(), fetchAndMergeFolders(), fetchAndMergeOrder(), fetchAndMergeStudyData()]);
    if (document.querySelector('.screen.active')?.id === 'screen-list') { renderDeckListUI(); }
    preloadUnsureBadges();
  } finally { isForceRefreshing = false; }
}
window.addEventListener('pageshow', (e) => {
  if (e.persisted) forceRefreshOnReturn();
  const overlay = document.getElementById('page-nav-loading');
  if (overlay) overlay.classList.remove('show');
});
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') forceRefreshOnReturn();
});
```
- コメントに、この処理が必要な理由が具体的に説明されています。ブラウザ（特にChrome）には「bfcache（back-forward cache）」という仕組みがあり、他のページに移動して「戻る」で復帰したとき、ページのスクリプトを再実行せず、**凍結していた状態をそのまま復元する**ことがあります。この場合、`setInterval`によるポーリングは（凍結されていたぶん）次のタイミングまで再開せず、しばらく古いデータのままになってしまいます。スマホでアプリを切り替えて長時間放置した場合も同様です。
- `pageshow`イベントの`e.persisted`（bfcacheから復元されたことを示すフラグ）や、`visibilitychange`イベント（タブ／アプリが表示状態に戻った）を検知した瞬間に、10秒待たず即座に最新データを取りに行くことで、この「しばらく古いデータのまま」という状態を解消しています。
- `isForceRefreshing`というフラグで、短時間に何度もこのイベントが発生しても、処理が重複して走らないようにガードしています。

### 7.4 学習データの更新チェック（5097〜5113行）
```js
async function checkStudyDataUpdate() {
  const activeScreen = document.querySelector('.screen.active')?.id;
  if (activeScreen !== 'screen-list') return;
  try {
    const changed = await fetchAndMergeStudyData();
    if (changed && document.querySelector('.screen.active')?.id === 'screen-list') { renderDeckListUI(); }
  } catch (e) {}
}
setInterval(checkStudyDataUpdate, 15000);
```
- 「わからない」マーク・進捗・完了記録は、フォルダ一覧などと違って**全員共通の1つのファイルではなく生徒ごとに別々のデータ**なので、ハッシュ比較の対象にはせず、単純に一覧画面を見ているときだけ15秒ごとに取得し直す、というシンプルな方式になっています。他の端末で「わからない」マークを付けた直後でも、この一覧を見ていればほぼ最新の状態に追従します。

---

## 8. 締めくくり（5115〜5117行）

```js
hideLoadingFallback();
```
- [00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)・[01_index_予定管理.md](../01_index_予定管理.md)で説明した「読み込み中…」保険オーバーレイを消す、お決まりの最後の1行です。`Cardmaker.js`はここで終わりです。

---

`Cardmaker.js`本体の解説はここまでです。続きは[10_遅延読み込みチャンク_CSVと並び替えと検索.md](10_遅延読み込みチャンク_CSVと並び替えと検索.md)・[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)で、これまで何度も「仮の窓口」として登場してきた5つの補助ファイルの中身を解説します。
