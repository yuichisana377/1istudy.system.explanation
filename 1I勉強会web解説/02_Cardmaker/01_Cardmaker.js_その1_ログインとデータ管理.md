# Cardmaker.js その1：ログイン・データの持ち方・共通ダイアログ（1〜812行）

[00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)の続きです。用語は[01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

このパートで出てくる新しい用語：
- **キャッシュ**：サーバーに毎回問い合わせなくても済むように、一度取得したデータをこの端末に控えておくこと。届くまでの間だけ「とりあえずこれを表示しておく」という一時的なコピーとして使う。
- **`Promise`／`resolve`**：非同期処理（サーバーへの問い合わせなど）の「結果が出たら渡します」という約束を表す入れ物。`new Promise(resolve => {...})`は「あとで`resolve(値)`が呼ばれたら、`await`で待っている側にその値が届く」という仕組みを作る書き方。難しければ「結果が出たときに答えを渡す仕組み」くらいの理解でOK。
- **`AbortController`**：通信に時間がかかりすぎたときに、途中で強制的に打ち切るための道具。「〇秒経っても返事が無ければ諦める」というタイムアウト処理に使う。

---

## 1. ログインとアカウント表示（6〜138行）

冒頭は[01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`とほぼ同じ内容です：`API_BASE`/`GUILD_ID`/`SESSION_KEY`/`LOGIN_PATH`の定義、`getLoginSession()`、開いた瞬間の強制ログイン確認（IIFE）、`renderDrawerAccount()`（ドロワー下部のアカウント表示）。重複する説明は省略し、CardMaker独自の部分だけ説明します。

```js
const STUDENT = (function() {
  var s = getLoginSession() || {};
  return { id: s.student_id, nickname: s.nickname, color: s.color, textColor: s.text_color };
})();
```
- ログイン情報から、このページで繰り返し使う4つの値（学籍番号・ニックネーム・アバターの色・文字色）だけを取り出して`STUDENT`という定数にまとめています。他ページのように毎回`getLoginSession()`を呼び直すのではなく、ページを開いた瞬間の値を固定で使い回す設計です（StudyLog.jsと同じ考え方だとコメントにあります）。

```js
function applyAccountHeader() {
  var avatarEl   = document.getElementById("header-avatar");
  var nicknameEl = document.getElementById("header-nickname");
  var idEl       = document.getElementById("header-id");
  if (!STUDENT.nickname) return; // ★ 通常はここに来る前にログイン画面へ飛んでいるはずだが念のため

  if (avatarEl) {
    avatarEl.textContent      = STUDENT.nickname.slice(0, 2).toUpperCase();
    avatarEl.style.background = STUDENT.color;
    avatarEl.style.color      = STUDENT.textColor;
  }
  if (nicknameEl) nicknameEl.textContent = STUDENT.nickname;
  if (idEl)       idEl.textContent       = STUDENT.id;
}
applyAccountHeader();
```
- 画面右上のアバター・ニックネーム・学籍番号を`STUDENT`の値で埋めます。「念のため」というコメントの通り、本来ここに到達する前にログインチェックで弾かれているはずですが、万が一のための防御的なチェックが入っています。

- 108行目、ログアウト確認は`showAppConfirm`（他ページ共通のDialog.js）ではなく`showCmConfirm`（このファイル自身が702行目以降で定義する、CardMaker専用の確認ダイアログ）を使っています。

---

## 2. デッキ・フォルダのデータの持ち方（140〜392行）

### 2.1 保存場所の基本（140〜143行）
```js
const STORE_KEY = 'cardmaker_decks_v1';
function loadDecks() { try { return JSON.parse(localStorage.getItem(STORE_KEY)) || []; } catch { return []; } }
function saveDecks(d) { localStorage.setItem(STORE_KEY, JSON.stringify(d)); }
function genId() { return Date.now().toString(36) + Math.random().toString(36).slice(2,6); }
```
- デッキのデータは`localStorage`（この端末に保存される領域）に丸ごとJSONとして保存されています。サーバーとのやり取りは別に行われますが、**この端末での表示・操作は基本的にこの`localStorage`の中身を見て行われる**という設計です（サーバーと完全同期するのではなく、この端末用のキャッシュとして機能する）。
- `genId()`は新しいカードやデッキにIDを振るための関数。「今の時刻を36進数の文字列にしたもの」＋「ランダムな文字列」を組み合わせて、ほぼ確実に他とかぶらない文字列を作っています。

### 2.2 フォルダの持ち方（145〜160行）
```js
const FOLDER_CACHE_KEY = 'cardmaker_folders_cache_v1';
function loadFoldersCache() { try { return JSON.parse(localStorage.getItem(FOLDER_CACHE_KEY)) || []; } catch { return []; } }
function saveFoldersCache(f) { localStorage.setItem(FOLDER_CACHE_KEY, JSON.stringify(f)); }
const MAX_FOLDER_DEPTH = 3;
const QUIZ_ARCHIVE_FOLDER_ID = "quiz_archive_root";
let folders = loadFoldersCache();
let currentFolderId = null; // null = ルート
```
- コメントにある通り、フォルダの「本体」はサーバー側（GitHub上の`folders.json`、みんなで共有）にあります。ここでの`localStorage`保存は、あくまで「サーバーから最新版が届くまでの間、画面が真っ白にならないよう、前回取得した内容をとりあえず表示しておくためのキャッシュ」です。
- `MAX_FOLDER_DEPTH = 3`：フォルダは最大3階層まで。
- `QUIZ_ARCHIVE_FOLDER_ID`：「クイズ過去問」という特別なフォルダのID。これはBotのサーバー側と同じ固定値で、名前変更・削除・移動ができない「システムフォルダ」として扱われます（後述のガード処理で保護されています）。
- `currentFolderId`：今どのフォルダの中を見ているか（`null`なら一番上の階層＝ホーム）。

### 2.3 ホーム画面の並び順（169〜239行）

デッキ・フォルダをドラッグして並べ替えた順番をどう扱うか、という仕組みです。少し複雑なので、コメントの要点を整理します。

- 並び順のうち「フォルダ」「公開済みデッキ」は**みんなで共有**（サーバー側の`list_order.json`に保存）。一方「まだ公開していない、自分だけの下書きデッキ」の並び順は他人には関係ないデータなので、**この端末のローカル保存のみ**にとどめます。
- `isSharedOrderKey(key)`：並び順を表すキー文字列が`'folder:'`または`'deck:'`で始まっていれば「共有される項目」、`'localdeck:'`で始まっていれば「自分だけの下書き」と判定します。
- `mergeSharedOrderIntoLocal(localOrder, sharedOrder)`：サーバー側の共有並び順を、この端末のローカル並び順に「差し込む」処理です。共有項目（フォルダ・公開デッキ）の並びはサーバー側を正解として反映しつつ、自分の下書きデッキだけは今の位置のまま維持します。こうすることで「他の人が並べ替えた結果」と「自分だけの下書きの位置」を両立させています。
- `getSavedListOrder(folderId)`／`saveListOrder(folderId, keys)`：あるフォルダ（`null`ならホーム）の並び順を読み書きする関数。読み込み時に上記のマージ処理を毎回行い、結果を再保存してキャッシュを更新します。
- `applySavedListOrder(items, folderId)`：実際に画面に表示するとき、保存済みの並び順があればその順に並べ替え、まだ並び順に登場していない新しいアイテム（新規作成分など）は末尾に追加します。
- `cmDragJustEndedAt`／`cmListDragActive`：長押しドラッグ操作中であることを示すフラグ。ドラッグ中に画面の再描画が走ってしまうと、掴んでいた要素が新しく作り直されたDOM（画面の部品）から浮いてしまい、「同じ項目が一時的に2つ表示される」という不具合が起きるため、ドラッグ中は再描画を止めるためのガードとして使われます（詳細は[05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)）。

### 2.4 サーバーとの同期関数（252〜303行）
```js
async function fetchAndMergeOrder() {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 5000);
  const res = await fetch(`${API_BASE}list_order`, { signal: controller.signal, cache: 'no-store' });
  clearTimeout(timer);
  const data = await res.json();
  if (!data.ok) return false;
  sharedOrderCache = data.order || {};
  saveSharedOrderCache(sharedOrderCache);
  return true;
}
```
- `AbortController`を使って「5秒待っても返事が無ければ通信を打ち切る」タイムアウトを実装しています。`controller.signal`を`fetch`に渡しておくと、`controller.abort()`が呼ばれた瞬間にその通信は中断されます。
- `cache: 'no-store'`は、ブラウザが古い結果を勝手に使い回してしまわないようにする指定です（他の部員が作ったフォルダがすぐ反映されない、という不具合を防ぐため）。

`pushSharedOrderToServer(folderId, keys)`（266〜287行）は逆方向、この端末で決めた並び順のうち共有部分だけをサーバーに送る関数です。8秒のタイムアウト付きで`/save_order`にPOSTし、成功したらローカルのキャッシュも更新します。

`fetchAndMergeFolders()`（290〜303行）はフォルダ一覧をサーバーから取得してキャッシュに反映する関数。同じくタイムアウト・`no-store`付きです。

### 2.5 フォルダの階層構造を扱う関数群（305〜392行）

ここは「木構造（親子関係を持つデータ）を扱うときによく出てくるパターン」の集まりです。1つずつ短く説明します。

- `folderLevel(id)`：あるフォルダが何階層目にあるか（親をたどって数える）。
- `folderChildren(parentId)`：あるフォルダの直下の子フォルダ一覧（名前順にソート）。
- `folderDescendants(id)`：あるフォルダの配下すべて（子・孫…を再帰的に集める）。
- `maxLevelInSubtree(id)`：あるフォルダとその配下の中で、一番深い階層がどこまであるか。
- `canMoveFolderTo(folderId, newParentId)`：あるフォルダを別の場所に移動できるかどうかの判定。「自分自身の中には移動できない」「クイズ過去問フォルダ自体は動かせない」「移動先が自分の子孫だと循環してしまうのでNG」「移動すると3階層を超えてしまう場合はNG」「クイズ過去問フォルダの中身はその外に出せない」という複数の条件をチェックしています。
- `countDecksRecursive(folderId)`／`countCardsRecursive(folderId)`：あるフォルダ配下（サブフォルダ含む）の合計デッキ数・合計カード数。
- `countUnsureRecursive(folderId)`：あるフォルダ配下の「わからない」マークが付いたカードの合計数。まだカード本体を読み込んでいないデッキは数えられないのでスキップする、という注意書きがあります。
- `collectDecksInFolder(folderId)`：あるフォルダ配下の全デッキを1つの配列に集める。
- `isDeckInFolderScope`/`isFolderInFolderScope`：あるデッキ／フォルダが、指定したフォルダの範囲内（サブフォルダ含む）にあるかどうかの判定。
- `canMoveDeckTo(deckId, targetFolderId)`：デッキ版の移動可否判定。フォルダのような階層数の制限は無く、今のところ特別な制限もありません（常に`true`）。★ 以前は「クイズ過去問フォルダの中のデッキは外に出せない」という制限がありましたが、2026/08/21にユーザーの要望で撤廃されました。クイズ過去問デッキも他のフォルダへ移動できますが、代わりに「問題の編集はできない」という別の制限が設けられています（`openDeckMenu`・`bot.py`側の`save_cards`を参照。[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)参照）。

---

## 3. クイズ用デッキ選択モード（pickMode）（394〜498行）

Quiz.htmlの「クイズを作る」→「デッキを選ぶ」から`?pick=quiz`というURLパラメータ付きでCardMakerを開くと、普段のデッキ一覧の見た目のまま、デッキやフォルダをタップして**複数選択**できるモードに切り替わります。

- `isDeckQuizPickable(d)`：あるデッキがクイズの4択自動生成に使えるかどうか。「非公開（下書き）ではない」「公開作業が途中でない」「カードが1枚以上ある」の3条件。
- `pickFolderAncestorSelected(folderId)`：あるフォルダの親をたどっていって、すでに選択済みのフォルダがあるかどうか。あれば「上のフォルダごと選ばれているので、これは自動的に含まれている」という扱いになります。
- `togglePickDeck(deckId, ev)`／`togglePickFolder(folderId, ev)`：デッキ／フォルダのタップで選択状態を切り替える。フォルダを選んだときは、その配下の個別選択（デッキ・子フォルダ）は「フォルダ選択に含まれて冗長」になるので、自動的に整理（選択解除）されます。
- `computePickedDecks()`：最終的に選ばれている内容を、実際にクイズへ渡す「デッキのファイル名一覧」に展開します。フォルダ選択は配下の対象デッキへ、個別選択とあわせて重複なく1つのリストにまとめます。
- `updatePickBar()`：画面下部のバーに「〇件のデッキを選択中」と表示し、1件も選ばれていなければ決定ボタンを押せないようにします。
- `pickModeCancel()`／`pickModeConfirm()`：キャンセルすると元のQuiz.htmlに戻る。決定すると、選んだデッキ一覧を`sessionStorage`に控えてからQuiz.htmlに戻ります（Quiz.js側がこれを読み取って使う）。
- `initPickModeFromUrl()`：URLの`?pick=quiz`を見て、このモードを開始する初期化関数。`history.replaceState(...)`でURLからこのパラメータを消しておく（ブラウザの履歴には残さない）、というひと工夫もされています。

---

## 4. カードデータの共通処理（500〜621行）

```js
function resolveDeckIdFromDragKey(key) {
  const d = key.startsWith('deck:')
    ? decks.find(x => x.filename === key.slice('deck:'.length))
    : decks.find(x => x.id === key.slice('localdeck:'.length));
  return d ? d.id : null;
}
```
- ドラッグ&ドロップで使う「`deck:ファイル名`」または「`localdeck:内部id`」という形式の文字列（並び順のキー）から、実際のデッキを逆引きする共通関数です。

```js
function cardToServerPayload(c) {
  const base = { id: c.id, question: c.question, answer: c.answer, explanation: c.explanation || '',
    imgs_q: c.imgs_q || [], imgs_a: c.imgs_a || [], imgs_e: c.imgs_e || [] };
  if (Array.isArray(c.choices)) {
    base.choices = c.choices;
    base.correct_indices = Array.isArray(c.correct_indices) ? c.correct_indices : [];
  }
  return base;
}
```
- カード1件を、サーバーに送るためのシンプルな形（オブジェクト）に変換する共通関数です。多肢選択デッキのカード（`choices`/`correct_indices`を持つ）の場合はその情報も含めます。コメントには「これを含めずに送ってしまうと、選択式デッキを1回でも同期した瞬間に選択肢データがサーバー側から消えてしまう」という注意が書かれています。実際に起きたバグの再発防止として、この関数が用意されているようです。

```js
function hashStr(str) {
  let h = 0;
  for (let i = 0; i < str.length; i++) { h = (h * 31 + str.charCodeAt(i)) | 0; }
  return h.toString(36);
}
function cardKey(c) {
  return c.id || ('h_' + hashStr((c.question || '') + '||' + (c.answer || '')));
}
```
- `hashStr`は文字列から簡単なハッシュ値（[01_index_予定管理.md](../01_index_予定管理.md)参照）を計算する自作の関数です（1文字ずつ`h = h*31 + 文字コード`を繰り返す、よく使われる簡易ハッシュのやり方）。
- `cardKey(c)`：カードを一意に識別するためのキーを作ります。IDが振られていればそれを使い、無ければ（公開後にサーバーから取り込まれたカードなど）問題文と解答からハッシュ値を計算して代用します。これにより、並び替えやサーバーとの同期のときに「配列の何番目か」ではなく「このカード自体」を安定して指し示せるようになります。

---

## 5. 多肢選択デッキの「選択肢入力欄」ウィジェット（526〜594行）

このブロックは、カード新規作成フォームとカード編集モーダルの**両方から使い回される共通部品**です。選択肢は2〜5個、正解はチェックボックスで選び、チェックした個数がそのまま「1つだけ＝択一問題」「2つ以上＝複数回答問題」の判定になります（デッキ単位やカード単位で別途モード切り替えを持たない、という設計）。

```js
function readChoiceEditorState(prefix) {
  const rows = document.querySelectorAll(`#${prefix}-rows .modal-choice-row`);
  const choices = [];
  const correct = [];
  rows.forEach((row, i) => {
    choices.push(document.getElementById(`${prefix}-choice-${i}`).value);
    const inp = document.getElementById(`${prefix}-correct-${i}`);
    if (inp && inp.checked) correct.push(i);
  });
  return { choices, correct };
}
```
- 今、画面上の入力欄に入っている選択肢のテキストと、チェックされている正解のインデックスを読み取って返します。`prefix`によって「新規作成フォーム用」か「編集モーダル用」かを切り替えられる汎用設計です。

```js
function renderChoiceEditorRows(prefix, choices, correctIdx) {
  const n = choices.length;
  const rowsHtml = choices.map((val, i) => `
    <div class="modal-choice-row" data-idx="${i}">
      <input type="checkbox" id="${prefix}-correct-${i}" value="${i}">
      <input type="text" class="modal-input" id="${prefix}-choice-${i}" placeholder="選択肢 ${CHOICE_LETTERS[i] || ''}" maxlength="80" style="margin-bottom:0">
      ${n > CHOICE_MIN ? `<button type="button" class="choice-remove-btn" data-ridx="${i}" title="この選択肢を削除">${Icons.html('close', {size:14})}</button>` : ''}
    </div>`).join('');
  const addBtnHtml = n < CHOICE_MAX
    ? `<button type="button" class="block-action-btn" id="${prefix}-add-btn" style="margin-top:.25rem">＋ 選択肢を追加</button>` : '';
  const container = document.getElementById(`${prefix}-rows`);
  container.innerHTML = rowsHtml + addBtnHtml;

  // ★ value属性への直接埋め込みはエスケープ事故（クォート等）の元になるため、
  //   空要素を描画してから .value / .checked をJSで設定する
  choices.forEach((val, i) => { document.getElementById(`${prefix}-choice-${i}`).value = val; });
  correctIdx.forEach(i => { const el = document.getElementById(`${prefix}-correct-${i}`); if (el) el.checked = true; });

  const addBtn = document.getElementById(`${prefix}-add-btn`);
  if (addBtn) addBtn.onclick = () => addChoiceRow(prefix);
  container.querySelectorAll('.choice-remove-btn').forEach(btn => {
    btn.onclick = () => removeChoiceRow(prefix, Number(btn.dataset.ridx));
  });
}
```
- 選択肢の入力行を作り直す関数です。ここで注目したいのは、入力欄の`value`（今入っている文字列）を**HTML文字列の中に直接埋め込んでいない**点です。まず空の入力欄だけをHTMLとして作ってから、あとで`.value = val`とJSで代入しています。コメントに「value属性への直接埋め込みはエスケープ事故（クォート等）の元になるため」とある通り、`"`を含む文字列を`value="${val}"`のように直接埋め込んでしまうと、その`"`のところで属性が途切れてHTMLが壊れてしまう危険があるため、これを避ける安全な書き方をしています。
- 選択肢が2個までしか無ければ削除ボタンは出さない（`CHOICE_MIN`未満にはできない）、5個に達したら追加ボタンを出さない（`CHOICE_MAX`）という、上限・下限のガードも組み込まれています。

`addChoiceRow(prefix)`／`removeChoiceRow(prefix, idx)`（580〜594行）は、それぞれ選択肢を1個追加・削除して`renderChoiceEditorRows`を呼び直す関数です。削除のときは、削除した選択肢より後ろの正解インデックスを1つずつ繰り上げる処理（`correct.filter(...).map(...)`）も行っています。

---

## 6. カード保存前の重複・矛盾チェック（609〜689行）

このサイトが「間違った入力を強制的にブロックする」のではなく「気づかせた上で、本人の判断に委ねる」という設計方針を取っている一例です。

```js
function normalizeForDupCheck(s) { return (s || '').trim(); }
function findDuplicateCardIndex(deck, q, a, excludeIdx = -1) {
  if (!deck || !Array.isArray(deck.cards)) return -1;
  const nq = normalizeForDupCheck(q), na = normalizeForDupCheck(a);
  return deck.cards.findIndex((c, i) =>
    i !== excludeIdx && normalizeForDupCheck(c.question) === nq && normalizeForDupCheck(c.answer) === na
  );
}
```
- `normalizeForDupCheck`は前後の空白だけを取り除く関数（大文字/小文字や全角/半角はそのまま＝「完全一致」判定をあえて厳密にしている）。
- `findDuplicateCardIndex`は、今から保存しようとしているカードと「問題文・解答が両方とも完全一致」するカードが、すでにそのデッキの中に無いかを探します。`excludeIdx`は編集時に「自分自身」を比較対象から除外するためのものです。

```js
async function warnIfDuplicateOrSameCard(deck, q, a, e, excludeIdx = -1) {
  // ①同じ問題・答えの組み合わせが既にある
  // ②解答が問題文と完全一致
  // ③解答が解説と完全一致
  // → それぞれ該当したら showCmConfirm で警告し、「やめる」が選ばれたら true（保存を中断すべき）を返す
}
```
- 3つのケースそれぞれについて、「入力ミスの可能性があります。このまま保存しますか？」という確認ダイアログを出します。ユーザーが「内容を確認する」（＝保存しない）を選んだ場合は`true`を返し、呼び出し元（`saveCard`など）はそこで保存処理を止めます。「このまま保存する」を選べば、意図的な内容として保存が続行されます。

---

## 7. 自作の確認ダイアログ（691〜795行）

[00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)で触れた通り、CardMakerは他ページ共通の`Dialog.js`を使わず、独自にこの3つの関数を持っています。理由はコメントに書かれている通り、「Cardmaker.cssの既存クラスをそのまま流用して動的にモーダルを生成することで、新規CSSを追加せずに他のモーダルと完全に同じ見た目・アニメーションになる」ためです。

3つとも共通のパターンで作られています：
1. `document.createElement`で空の`<div>`を作り、`innerHTML`でモーダルの中身を組み込む（タイトル・説明文・ボタンは全部`esc()`済み）。
2. `document.body`に追加してから、`requestAnimationFrame`（次の画面更新のタイミングで実行、を意味するブラウザの機能）で`open`クラスを付けてアニメーション開始。
3. ボタンが押された・背景がクリックされたら、`open`クラスを外してフェードアウトさせてから（180ミリ秒後に）要素ごと削除し、`Promise`の`resolve`で結果を呼び出し元に返す。

- `showCmConfirm({title, desc, okLabel, cancelLabel, okStyle})`：キャンセル/実行の2択（OS標準の`confirm()`の代替）。押された結果を`true`/`false`で返します。`await showCmConfirm({...})`のように呼び出せば、「OKが押されるまで待って、結果を受け取る」という書き方ができます。
- `showCmAlert({title, desc, okLabel})`：ボタン1つだけの通知（OS標準の`alert()`の代替）。
- `showCmChoiceDialog({title, desc, choices, cancelLabel})`：3つ以上の選択肢から1つを選ぶダイアログ。`choices`は`{icon, label, sub, value}`の配列で、選ばれた項目の`value`を返す（キャンセル時は`null`）。

---

## 8. 画面切り替えの中心：`showScreen()`（797〜809行）

```js
function showScreen(id) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.getElementById('screen-' + id).classList.add('active');
  window.scrollTo(0, 0);
  if (id === 'list') {
    decks = loadDecks();
    folders = loadFoldersCache();
    sharedOrderCache = loadSharedOrderCache();
    renderDeckListUI();
    setTimeout(() => renderDeckList(), 0);
  }
}
```
- [00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)で説明した通り、CardMakerは複数の`<div class="screen">`を最初から全部HTML内に用意しておき、この関数で見せる画面を切り替える簡易的な「ルーター」です。全部の画面から一旦`active`クラスを外し、指定された画面にだけ付け直します。
- 画面を切り替えるたびにページの一番上までスクロールを戻します。
- 一覧画面（`'list'`）に戻ってきたときだけ特別に、`localStorage`のキャッシュ（デッキ・フォルダ・共有並び順）を読み直してから画面を再描画し（`renderDeckListUI()`）、さらに`setTimeout(..., 0)`で少し遅らせてサーバーへの最新データ取得（`renderDeckList()`、[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で解説）も呼びます。「まずキャッシュで即座に表示し、そのあとサーバーの最新情報で更新する」という、体感速度を優先した2段階の作りです。

---

続きは[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)で、`renderDeckListUI()`の中身（一覧の実際の描画処理）から解説します。
