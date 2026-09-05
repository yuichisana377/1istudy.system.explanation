# 遅延読み込みチャンク（1）：CSV一括読み込み・カード並び替え・単語検索

[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)までで`Cardmaker.js`本体を読み終えました。ここからは、[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で説明した「遅延読み込みチャンク」の仕組みによって、ページ表示後にバックグラウンドで追加ダウンロードされる5つの補助ファイルのうち、3つを解説します。

**注意**：これらのファイルは単体では動きません。すべて`Cardmaker.js`が先に読み込まれている前提で、そちらの変数（`decks`・`currentDeckId`など）や関数（`saveDecks`・`showCmAlert`など）をそのまま使っています。

---

## 1. `Cardmaker-csvimport.js`（246行）— CSV一括読み込み

「CSVから読み込む」ボタンから呼ばれる、[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)で「仮の窓口」として紹介した`handleCsvImport`/`handleChoiceCsvImport`の本体です。通常デッキ用と多肢選択デッキ用の2つの機能が入っています。

### 1.1 自前のCSV解析：`parseCSV(text)`（29〜56行）

```js
function parseCSV(text) {
  if (text.charCodeAt(0) === 0xFEFF) text = text.slice(1); // BOM除去
  const rows = [];
  let row = [], field = '', inQuotes = false;
  for (let i = 0; i < text.length; i++) {
    const c = text[i];
    if (inQuotes) {
      if (c === '"') {
        if (text[i + 1] === '"') { field += '"'; i++; }
        else { inQuotes = false; }
      } else { field += c; }
    } else if (c === '"') { inQuotes = true; }
    else if (c === ',') { row.push(field); field = ''; }
    else if (c === '\r') { /* 何もしない */ }
    else if (c === '\n') { row.push(field); rows.push(row); row = []; field = ''; }
    else { field += c; }
  }
  if (field.length || row.length) { row.push(field); rows.push(row); }
  return rows.filter(r => r.some(c => (c || '').trim() !== ''));
}
```
- CSV（カンマ区切りのテキストファイル）を、外部のライブラリに頼らず**自分でパース（解析）**しています。1文字ずつ読み進めながら、「今、ダブルクォートで囲まれた範囲の中にいるか（`inQuotes`）」を管理する、CSV解析の定番のやり方です。
- `text.charCodeAt(0) === 0xFEFF`：ファイルの先頭にある「BOM」（Byte Order Mark、Excelなどが文字コードを示すためにファイル冒頭に付ける特殊な目印文字）を取り除きます。これが残ったままだと、1問目のデータの先頭に見えない文字が紛れ込み、正しく認識されない原因になります。
- ダブルクォートの中に、さらにダブルクォートを2つ連続で書く（`""`）というのは、CSVの仕様で「本物の`"`の文字」を表す方法です（`if (text[i+1] === '"') { field += '"'; i++; }`の部分）。
- `\r`（キャリッジリターン）は無視し、`\n`（改行）が来たときだけ行の区切りとして扱うことで、Windows形式（`\r\n`）とそれ以外の改行コードの両方に対応しています。
- 最後に、すべての列が空文字列だけの完全な空行は結果から除外しています。

### 1.2 見出し行の自動認識（58〜74行）
```js
const CSV_HEADER_ALIASES = {
  question:    ['question', '問題', '問題文', 'q'],
  answer:      ['answer', '解答', '答え', 'a'],
  explanation: ['explanation', '解説', 'e'],
};
function detectCsvColumns(headerRow) {
  const norm = headerRow.map(h => (h || '').trim().toLowerCase());
  const idx = { question: -1, answer: -1, explanation: -1 };
  norm.forEach((h, i) => {
    for (const key of Object.keys(CSV_HEADER_ALIASES)) {
      if (idx[key] === -1 && CSV_HEADER_ALIASES[key].includes(h)) idx[key] = i;
    }
  });
  const isHeader = idx.question !== -1 && idx.answer !== -1;
  return { isHeader, idx };
}
```
- 1行目が「問題」「解答」（または英語表記など、あらかじめ用意した別名リスト）のどれかに一致すれば、それを見出し行とみなし、それぞれの列が何番目かを記録します。問題・解答の両方が見出しとして認識できた場合だけ「見出し行あり」と判定し、認識できなければ「1行目からすでにデータ」として扱います（呼び出し元で「問題,解答,解説」の固定順にフォールバックします）。

### 1.3 特殊文字の自動除去：`stripBugChars(str)`（76〜81行）
```js
function stripBugChars(str) {
  if (!str) return str;
  const bad = findBugChars(str);
  if (!bad.length) return str;
  return [...str].filter(ch => !bad.includes(ch)).join('');
}
```
- [08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で説明した`findBugChars`（トラブルの元になる特殊文字を見つける関数）を使い、該当する文字だけを取り除いた文字列を返します。手入力のカード作成では警告して入力をやり直させる（`warnIfBugChars`）方式でしたが、CSVの一括読み込みでは1行ごとに確認を挟むのは非現実的なため、**黙って自動的に取り除いてから取り込む**という違う方針を取っています。

### 1.4 通常デッキ用の取り込み：`handleCsvImport(event)`（83〜141行）
```js
let added = 0, skippedEmpty = 0, skippedDup = 0, sanitized = 0;
for (const r of dataRows) {
  let q = (r[idx.question] || '').trim();
  let a = (r[idx.answer] || '').trim();
  let e = idx.explanation !== -1 ? (r[idx.explanation] || '').trim() : '';
  if (!q || !a) { skippedEmpty++; continue; }
  const before = q + a + e;
  q = stripBugChars(q); a = stripBugChars(a); e = stripBugChars(e);
  if ((q + a + e) !== before) sanitized++;
  if (!q || !a) { skippedEmpty++; continue; }
  if (findDuplicateCardIndex(deck, q, a) !== -1) { skippedDup++; continue; }
  deck.cards.push({ id: genId(), question: q, answer: a, explanation: e, imgs_q: [], imgs_a: [], imgs_e: [] });
  added++;
}
```
- 1行ずつ、問題文・解答（・解説）を取り出し、空なら`skippedEmpty`を増やしてスキップ。
- 特殊文字を除去したあと、**除去した結果また空になってしまった場合**（例えば元々「見えない文字」だけの内容だった場合）も同様にスキップします。
- [01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の`findDuplicateCardIndex`（完全一致する既存カードが無いか）を使って重複をチェックし、重複していれば`skippedDup`を増やしてスキップします。CSV一括読み込みでは、手入力のとき（`warnIfDuplicateOrSameCard`）のように1件ずつ確認ダイアログを出すのは現実的でないため、確認を挟まず自動でスキップする方針です。
- 最後に、追加件数・重複スキップ件数・空欄スキップ件数・文字を除去した件数を集計し、`showCmAlert`で1つのメッセージにまとめて報告します。

### 1.5 多肢選択デッキ用の取り込み：`handleChoiceCsvImport(event)`（143〜246行）

考え方は同じですが、列の検出と正解の解釈が複雑になります。

```js
function detectChoiceCsvColumns(headerRow) {
  const norm = headerRow.map(h => (h || '').trim().toLowerCase());
  let qIdx = -1, correctIdx = -1;
  const choiceCols = [];
  norm.forEach((h, i) => {
    if (qIdx === -1 && CHOICE_CSV_HEADER_ALIASES.question.includes(h)) { qIdx = i; return; }
    if (correctIdx === -1 && CHOICE_CSV_HEADER_ALIASES.correct.includes(h)) { correctIdx = i; return; }
    const m = h.match(/^(?:選択肢|choice)\s*([1-5])$/) || h.match(/^([a-e])$/);
    if (m) {
      const num = /^[a-e]$/.test(m[1]) ? 'abcde'.indexOf(m[1]) + 1 : Number(m[1]);
      choiceCols.push({ col: i, num });
    }
  });
  choiceCols.sort((a, b) => a.num - b.num);
  const isHeader = qIdx !== -1 && correctIdx !== -1 && choiceCols.length >= CHOICE_MIN;
  return { isHeader, qIdx, correctIdx, choiceCols };
}
```
- 「選択肢1」「choice2」のような列名、または「a」〜「e」の1文字だけの列名の両方を受け付けます。正規表現の`(?:選択肢|choice)\s*([1-5])`は「『選択肢』または『choice』のあとに、空白があってもなくても1〜5の数字が続く」という意味です。
- `'abcde'.indexOf(m[1]) + 1`は、`a`〜`e`という文字を1〜5の数字に変換する、ちょっとした小技です（`'abcde'`という文字列の中で、その文字が何文字目にあるかを調べているだけです）。

```js
const choices = rawChoices.filter(c => c);
if (choices.length < CHOICE_MIN || choices.length > CHOICE_MAX) { skippedChoiceCount++; continue; }

const rawNums = correctRaw.split(/[,、\s]+/).map(s => s.trim()).filter(Boolean).map(Number);
const allValid = rawNums.length > 0 && rawNums.every(n => Number.isInteger(n) && n >= 1 && n <= choices.length);
if (!allValid) { skippedCorrect++; continue; }
const correct = [...new Set(rawNums.map(n => n - 1))].sort((a, b) => a - b);
```
- 空欄の選択肢列は`.filter(c => c)`で除外し、「詰めて」扱います（例えば選択肢3だけ空欄でも、残り4個がそのまま選択肢として認識されます）。結果が2〜5個の範囲外ならエラー扱いにしてスキップします。
- 正解列は「1,3」のようにカンマ・読点・空白のいずれかで区切られた数字の並びとして解釈します（`split(/[,、\s]+/)`は「カンマ・読点・空白のうち1文字以上が連続する部分」で分割する、という意味の正規表現）。すべての数字が「整数で、1〜選択肢数の範囲内」であることを確認し、1つでも不正な値があれば取り込みません。
- `[...new Set(...)]`で重複した番号（もし同じ番号が2回書かれていても）を除去してから、番号順に並べ直しています。

以降の重複チェック・保存・サーバー同期・結果報告は、通常デッキ用とほぼ同じ流れです。

---

## 2. `Cardmaker-cardreorder.js`（220行）— 作成済みカードのドラッグ並び替え

デッキ編集画面の「作成済みカード」一覧（[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)の`renderCreatedList`が描画する部分）を、専用の取っ手（⠿マーク）でドラッグして並び替える機能です。

[05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)で解説したホーム画面の並び替えと、基本の考え方（掴む→動かす→他のカードの中心を追い越したら`insertBefore`で入れ替える→端に近づいたら自動スクロール）はほぼ同じです。違いは主に2点です。

### 2.1 専用の取っ手（ハンドル）を使う
```js
// ── マウス操作（PC） ──
function onMouseDown(e) {
  const handle = e.target.closest('.drag-handle');
  if (!handle) return;
  const item = handle.closest('.created-item');
  if (!item) return;
  e.preventDefault();
  beginDrag(item, e.clientY);
  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp, { once: true });
}
function onMouseMove(e) { moveDrag(e.clientY); }
function onMouseUp() {
  window.removeEventListener('mousemove', onMouseMove);
  endDrag();
}
```
- ホーム画面の並び替えは「カード本体のどこでも長押し」でしたが、こちらは「⠿マーク（`.drag-handle`）を掴んだときだけ」ドラッグが始まります。これは、編集画面ではカード自体に「編集」「削除」ボタンがあり、カード本体全体を長押し対象にすると誤操作が増えるための使い分けと考えられます。

### 2.2 マウスとタッチを別々の仕組みで扱っている
```js
list.addEventListener('mousedown', onMouseDown);
list.addEventListener('touchstart', onTouchStart, { passive: false });
list.addEventListener('touchmove',  onTouchMove,  { passive: false });
list.addEventListener('touchend',   onTouchEnd);
list.addEventListener('touchcancel', onTouchEnd);
```
- [05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)の実装はPointer Events（マウスもタッチも統一的に扱える新しい仕組み）を使っていましたが、こちらは`mousedown`/`touchstart`という、より古くからある別々のイベントで実装されています。同じサイトの中でも作られたタイミングによって手法が異なる、という一例です。

```js
// ── タッチ操作（スマホ） ──
// ★ ハンドルに触れた指の identifier だけを追跡し、その指のtouchmoveだけを
//   ドラッグ処理として扱う（＝preventDefaultする）。もう片方の指のtouchmoveは
//   ここで何もしないので、ブラウザ標準の縦スクロールがそのまま働く。
//   これにより「片方の指でカードを移動させながら、もう片方の指でスクロール」ができる。
function onTouchStart(e) {
  const handle = e.target.closest('.drag-handle');
  if (!handle) return;
  if (dragTouchId !== null) return; // 既に別の指でドラッグ中なら何もしない
  const item = handle.closest('.created-item');
  if (!item) return;
  const touch = e.changedTouches[0];
  dragTouchId = touch.identifier;
  e.preventDefault();
  beginDrag(item, touch.clientY);
}
function findDragTouch(touchList) {
  if (dragTouchId === null) return null;
  for (let i = 0; i < touchList.length; i++) {
    if (touchList[i].identifier === dragTouchId) return touchList[i];
  }
  return null;
}
function onTouchMove(e) {
  const t = findDragTouch(e.changedTouches);
  if (!t) return; // ドラッグ中の指以外の動き（＝スクロール用の指）はここで無視する
  e.preventDefault();
  moveDrag(t.clientY);
}
function onTouchEnd(e) {
  const t = findDragTouch(e.changedTouches);
  if (!t) return;
  dragTouchId = null;
  endDrag();
}
```
- タッチ操作は複数の指を同時に検知できます（`e.changedTouches`は配列のように複数の指の情報を持てます）。`touch.identifier`は「この指」を区別するための番号です。ハンドルに触れた**その指の番号だけ**を`dragTouchId`として記録し、以降の`touchmove`ではその番号の指の動きだけをドラッグとして扱います。コメントによれば、これにより「片方の指でカードを動かしながら、もう片方の指で画面をスクロールする」という同時操作が可能になっています。

### 2.3 並び替え結果の反映（133〜160行）
```js
async function endDrag() {
  if (!dragEl) return;
  if (autoScrollRAF !== null) { cancelAnimationFrame(autoScrollRAF); autoScrollRAF = null; }
  dragEl.classList.remove('dragging');
  dragEl.style.transform = '';
  dragEl.style.zIndex = '';
  dragEl.style.boxShadow = '';
  dragEl.style.opacity = '';
  dragEl.style.position = '';
  dragEl = null;

  // ★ DOM上の最終的な並び順（data-key）から deck.cards を並び替える
  const orderedKeys = getItems().map(it => it.dataset.key);
  const deck = decks.find(d => d.id === currentDeckId);
  if (!deck) return;
  const byKey = new Map(deck.cards.map(c => [cardKey(c), c]));
  const newCards = orderedKeys.map(k => byKey.get(k)).filter(Boolean);
  if (newCards.length !== deck.cards.length) { renderCreatedList(); return; } // 念のための整合性チェック
  deck.cards = newCards;
  saveDecks(decks);
  renderCreatedList();

  // ★ 公開済みなら並び順もサーバーへ反映する（通知はしない）
  if (deck.filename) {
    const ok = await queueSyncDeckToServer(deck);
    if (!ok) showBanner('並び替えのサーバー反映に失敗しました（ローカルには保存済み）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
  }
}
```
- ドラッグ終了時の`#created-list`内の実際のDOM順（`data-key`の並び）を読み取り、`cardKey`をキーにした`Map`で実際のカードオブジェクトと突き合わせて、`deck.cards`配列そのものを新しい順序で作り直します。
- `if (newCards.length !== deck.cards.length)`：万一、対応するカードが見つからない（＝件数が合わない）事態になっていたら、並び替えは反映せず単に再描画するだけにする、という保険が入っています。
- 公開済みデッキなら、並び替えた結果もサーバーに同期します。

---

## 3. `Cardmaker-search.js`（132行）— 単語検索

「問題を検索」画面（[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)で紹介した`openSearchScreen`の本体）です。

### 3.1 検索対象の範囲（36〜50行）
```js
async function openSearchScreen() {
  searchScopeFolderId = currentFolderId;
  searchTargetDecks = null;
  document.getElementById('search-input').value = '';
  document.getElementById('search-results').innerHTML = '';
  const scopeLabel = folderPathLabel(searchScopeFolderId);
  setIconText(
    document.getElementById('search-scope-label'),
    scopeLabel ? Icons.cmHtml('folder', {size:14}) : Icons.html('logo', {size:14}),
    scopeLabel ? `${scopeLabel} の中を検索します` : 'すべてのデッキから検索します'
  );
  showScreen('search');
  setTimeout(() => document.getElementById('search-input').focus(), 200);
  await prepareSearchScope();
}
```
- 検索を開いた**その時点で表示していたフォルダ**（`currentFolderId`）を`searchScopeFolderId`として固定します。ホーム画面（フォルダを開いていない状態）から検索を開けば全体が対象、特定のフォルダの中から開けば、そのフォルダ（とサブフォルダ）だけが対象になります。
- `folderPathLabel`（[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)）で、今検索している範囲を「数学 / 二次関数 の中を検索します」のように分かりやすく表示します。

### 3.2 検索前の準備：`prepareSearchScope()`（52〜68行）
```js
const targets = collectDecksInFolder(searchScopeFolderId).filter(d => (d.filename ? (d.count ?? d.cards.length) : d.cards.length) > 0);
const unloaded = targets.filter(d => d.filename && !d.cardsLoaded);
if (unloaded.length) {
  statusEl.textContent = `問題データを読み込み中…（${unloaded.length}件のデッキ）`;
  await Promise.all(unloaded.map(d => ensureDeckCardsLoaded(d.id)));
}
statusEl.style.display = 'none';
if (!document.getElementById('screen-search').classList.contains('active')) return;
searchTargetDecks = targets;
runSearch();
```
- 検索対象範囲にあるデッキのうち、まだカード本体を読み込んでいないもの（一覧画面ではメタ情報しか持っていないため）をまとめて先読みします。読み込み中は件数を表示して、ユーザーが「固まった」と誤解しないようにしています。
- `if (!document.getElementById('screen-search').classList.contains('active')) return;`：この`await`で時間がかかっている間に、ユーザーがすでに検索画面から離れていた場合は、結果を反映せずに終わります（無駄な処理・見えない画面への反映を避ける）。

### 3.3 表記ゆれに強い検索：`normalizeForSearch(s)`（28〜34行）
```js
function normalizeForSearch(s) {
  if (!s) return '';
  let t = String(s).normalize('NFKC').toLowerCase();
  t = t.replace(/[ァ-ヶ]/g, ch => String.fromCharCode(ch.charCodeAt(0) - 0x60)); // カタカナ→ひらがな
  t = t.replace(/[\s　]+/g, ''); // 半角・全角の空白を除去
  return t;
}
```
- `.normalize('NFKC')`は、JS標準の「Unicode正規化」という機能で、全角英数字と半角英数字のような「見た目は違うが意味的には同じ」文字を統一する処理です。
- カタカナをひらがなに変換する処理は、カタカナとひらがなの文字コードが一定の差（`0x60`）で対応している、という文字コード表の性質を利用した力技です。
- 大文字/小文字の違い（`.toLowerCase()`）や空白の有無も無視することで、「Photosynthesis」で検索しても「photosynthesis」がヒットする、といった**多少の表記ゆれを許容する検索**を実現しています。
- **★ 2026/09/05 追記**：`normalizeForSearch()`はこのファイル（Cardmaker-search.js）専用のローカル関数だったが、「一覧で見る」画面内の検索（`Cardmaker-listview.js`、[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)参照）でも同じ表記ゆれ吸収を使いたくなったため、両チャンクから確実に参照できるコア（`Cardmaker.js`本体）側へ移した。このファイル単体では定義しなくなったが、呼び出し方は変わっていない。

### 3.4 検索の実行：`runSearch()`（75〜108行）
```js
const nq = normalizeForSearch(raw);
const hits = [];
for (const d of searchTargetDecks) {
  for (const c of d.cards) {
    const q = mathToPlainText(c.question), a = mathToPlainText(c.answer);
    if (normalizeForSearch(q).includes(nq) || normalizeForSearch(a).includes(nq)) {
      hits.push({ deckId: d.id, deckName: d.name, cardId: c.id, q, a });
    }
  }
}
```
- 対象デッキの全カードを1件ずつ総当たりで確認する、シンプルな線形探索です（デッキ数・カード数がこの規模であれば十分な速さです）。問題文・解答のどちらかに検索語が含まれていればヒットとします。
- `mathToPlainText`（[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)）を通してから比較しているので、`\sqrt{4}`のような生のLaTeX記法で保存されている古いカードでも、`√(4)`のような読みやすい形に変換されたうえで検索対象になります。

```js
function onSearchInput() {
  clearTimeout(searchDebounceTimer);
  searchDebounceTimer = setTimeout(runSearch, 150);
}
```
`onSearchInput()`（70〜73行）は、入力欄に何か入力されるたびに呼ばれますが、`clearTimeout`＋`setTimeout(runSearch, 150)`という**デバウンス（debounce）**という手法を使っています。1文字打つたびに即座に検索を実行するのではなく、「最後の入力から150ミリ秒操作が無ければ検索する」という形にすることで、高速に連続入力しているときに何度も無駄な検索処理が走らないようにしています。

### 3.5 検索結果から「一覧で見る」画面へ（110〜132行）
```js
async function openSearchResult(deckId, cardId) {
  const deck = decks.find(d => d.id === deckId);
  const card = deck.cards.find(c => c.id === cardId);
  if (!card) return;
  await loadChunkWithFeedback('listview', '/Cardmaker-listview.js');
  studyIsFolder = false;
  studyDeckId = deckId;
  listViewFilter = 'all';
  listViewReverse = false;
  document.getElementById('list-view-title').textContent = deck.name;
  pendingListViewScrollKey = cardKey(card);
  showScreen('list-view');
  renderListView();
}
```
- 検索結果をタップすると、編集画面ではなく「一覧で見る」画面（次のドキュメントで説明）を開き、該当の問題の位置まで自動でスクロールします。
- コメントに重要な注記があります。「一覧で見る」機能自体は`Cardmaker-listview.js`というまた別のチャンクに分離されているため、通常はCardmaker.js側の`openListView()`（[07_Cardmaker.js_その7_学習モードとクイズ再生.md](07_Cardmaker.js_その7_学習モードとクイズ再生.md)）という「仮の窓口」を経由してこのチャンクの読み込みを待ちますが、**検索結果からのこの遷移だけはその窓口を経由しない特別なルート**です。そのため、ここで個別に`loadChunkWithFeedback('listview', ...)`を呼んで、`renderListView()`（読み込みが終わらないと存在しない関数）を呼ぶ前に、確実に読み込みが完了しているようにしています。

---

続きは[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)で、残り2つのチャンク（一覧で見る画面・一人用選択式クイズ）を解説します。
