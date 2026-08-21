# Cardmaker.js その4：カードの保存とデッキの公開（2131〜2403行）

[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の続きです。

---

## 1. CSV読み込みの入口（2131〜2145行）

```js
async function handleCsvImport(event) {
  await loadChunkWithFeedback('csvimport', '/Cardmaker-csvimport.js');
  return handleCsvImport(event);
}
```
- [02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)の`openSearchScreen`と同じパターンです。実際のCSV取り込み処理は`Cardmaker-csvimport.js`（[10_遅延読み込みチャンク_CSVと並び替えと検索.md](10_遅延読み込みチャンク_CSVと並び替えと検索.md)）に分離されており、この関数はそのファイルを読み込み終えるまでの「仮の窓口」です。読み込みが終わると本物の同名関数に置き換わり、自分自身をもう一度呼び出すことで実際の処理につながります。

---

## 2. 多肢選択カードをフォームから追加：`addChoiceCardFromForm(deck, q)`（2147〜2185行）

多肢選択デッキの入力フォームから、カードを1枚追加する処理です。通常デッキの「解答/解説」を使った追加処理（3節）と役割は同じですが、選択肢特有のチェックが加わります。

```js
const { choices: rawChoices, correct } = readChoiceEditorState('ta-choice');
const choices = rawChoices.map(c => c.trim());
const anyInput = !!q || choices.some(c => c);
if (!anyInput) return 'skip';
if (!q) { shake('ta-q'); return 'invalid'; }
const emptyIdx = choices.findIndex(c => !c);
if (emptyIdx !== -1) { shake(`ta-choice-choice-${emptyIdx}`); return 'invalid'; }
if (correct.length === 0) {
  await showCmAlert({ title: '正解を1つ以上選んでください', desc: '選択肢の左のチェックボックスで、正解を選んでください。1つだけ選べば択一問題、2つ以上選べば複数回答問題になります。' });
  return 'invalid';
}
```
- 何も入力されていなければ`'skip'`（何もせず終了、「次へ」を押しただけの場合の正常な挙動）。
- 問題文が空、選択肢のどれかが空、正解が1つも選ばれていない、のいずれかならエラーを示して`'invalid'`を返します（呼び出し元はここで処理を止め、編集を続けさせます）。

```js
const answerText = correct.map(i => choices[i]).join(' / ');
if (await warnIfDuplicateOrSameCard(deck, q, answerText, '')) return 'invalid';
deck.cards.push({
  id: genId(), question: q, answer: answerText, explanation: '',
  choices, correct_indices: correct.slice().sort((x, y) => x - y),
  imgs_q: [...imgBuf.q], imgs_a: [], imgs_e: [],
});
saveDecks(decks);
document.getElementById('edit-counter').textContent = deck.cards.length + '枚';
if (deck.filename) queueSyncDeckToServer(deck);
```
- `answer`（解答欄）には、正解に選ばれた選択肢の文言を「/」でつなげたテキストを入れます。コメントによれば、これは「単語検索・一覧表示など、`answer`が文字列であることを前提にした既存のコードをそのまま動かすため」の工夫です。多肢選択デッキ専用の特別な表示ロジックを別に作らず、既存のコードを再利用できるようにしています。
- 保存後、そのデッキが既にサーバーに登録済み（`filename`あり）なら、`queueSyncDeckToServer`（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)で説明）でサーバーへの反映も予約します。

---

## 3. カードを保存する：`saveCard(mode)`（2187〜2269行）

「次へ」「保存」「保存して公開」の3つのボタンが、すべてこの1つの関数から呼ばれます（`mode`が`false`/`'local'`/`'publish'`）。

### 3.1 カード自体の追加（2188〜2217行）
```js
const isChoiceDeck = deck && !!deck.choiceMode;
if (isChoiceDeck) {
  const added = await addChoiceCardFromForm(deck, q);
  if (added === 'invalid') return;
} else {
  const a = document.getElementById('ta-a').value.trim();
  const e = document.getElementById('ta-e').value.trim();
  if (q || a) {
    if (!q || !a) { shake(!q ? 'ta-q' : 'ta-a'); return; }
    if (await warnIfBugChars(q, 'ta-q')) return;
    if (await warnIfBugChars(a, 'ta-a')) return;
    if (await warnIfBugChars(e, 'ta-e')) return;
    if (await warnIfDuplicateOrSameCard(deck, q, a, e)) return;
    deck.cards.push({ id:genId(), question:q, answer:a, explanation: e, imgs_q:[...imgBuf.q], imgs_a:[...imgBuf.a], imgs_e:[...imgBuf.e] });
    saveDecks(decks);
    document.getElementById('edit-counter').textContent = deck.cards.length + '枚';
    if (deck.filename) queueSyncDeckToServer(deck);
  }
}
```
- 多肢選択デッキなら2節の関数を、通常デッキなら「問題文・解答両方が入力されていること」「特殊文字が混ざっていないこと（`warnIfBugChars`）」「重複・矛盾が無いこと（`warnIfDuplicateOrSameCard`）」を確認してからカードを追加します。
- **サーバー登録済みのデッキは、カードを1枚追加するたびに毎回サーバーへも反映します**（`queueSyncDeckToServer`）。コメントによれば、これをしないと「編集画面を出てもう一度開いたときの強制リロードで、サーバー側のまだこのカードを知らない古いデータに上書きされ、せっかく追加したカードがローカルごと消えてしまう」という不具合があったため、この即時反映が入っています。

### 3.2 公開する：`mode === 'publish'`（2218〜2246行）
```js
if (!getLoginSession()) {
  const proceedAnon = await showCmConfirm({
    title: 'ログインしていません', desc: 'このまま公開すると「匿名」として公開されます。',
    cancelLabel: 'ログイン画面へ', okLabel: '匿名のまま公開する',
  });
  if (!proceedAnon) { sessionStorage.setItem('post_login_redirect', location.href); location.href = LOGIN_PATH; return; }
}
const choice = await showCmChoiceDialog({
  title: 'このデッキは完成していますか？',
  desc: '未完成として公開すると、Discordへの通知は送られません。\nあとから編集して完成にできます。',
  choices: [
    { icon: Icons.html('checkCircle', {size:20}), label: '完成として公開する',   sub: '通知が送信されます',   value: 'complete' },
    { icon: Icons.html('dot', {size:26, color:'#eab308'}), label: '未完成として公開する', sub: '通知は送信されません', value: 'draft' },
  ],
});
if (!choice) return;
publishDeck(deck.id, choice === 'complete');
```
- 未ログインでも公開自体は可能ですが、「匿名として公開されますがよいですか」という確認を挟みます。ログイン画面に行くことも選べ、その場合は戻ってこられるよう`post_login_redirect`に現在のURLを控えます。
- 「完成」か「未完成」かを選ばせ、未完成を選ぶとDiscordへの通知（みんなに知らせるお知らせ）を送らずに公開できます。あとから編集して完成にすることもできる、という柔軟な運用です。
- 実際の公開処理は`publishDeck`（4節）が担当します。

### 3.3 保存だけする：`mode === 'local'`（2247〜2263行）
```js
saveDecks(decks);
if (deck.filename) {
  const saveBtn = document.getElementById('btn-save-local');
  setBtnLoading(saveBtn, true, '保存中…');
  const ok = await queueSyncDeckToServer(deck);
  setBtnLoading(saveBtn, false);
  if (!ok) showBanner('サーバーへの保存に失敗しました（ローカルには保存済み）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
}
showScreen('list');
```
- サーバー登録済みのデッキであれば、「保存」ボタンでも公開確認は挟まず、静かに（通知なしで）サーバーへ反映してから一覧画面に戻ります。ここでも「反映しないと次に開いたときの強制リロードで直前の変更が消えてしまう」という、3.1節と同じ理由の対策が入っています。

### 3.4 それ以外（「次へ」）（2264〜2268行）
入力欄をクリアして（`clearEditor()`）、作成済みカード一覧を更新し、フォーカスを問題文欄に戻すだけです。

---

## 4. 実際にサーバーへ公開する：`publishDeck(deckId, isComplete)`（2271〜2352行）

```js
async function publishDeck(deckId, isComplete = true) {
  saveDecks(decks); showScreen('list');

  // showScreen('list') 実行後の最新の decks から対象デッキを取得する
  const deck = decks.find(d => d.id === deckId);
  if (!deck) return;

  const session = getLoginSession();
```
- 冒頭のコメントに重要な設計上の注意が書かれています。`showScreen('list')`を呼ぶと、その内部で`decks = loadDecks()`が実行され、**`decks`配列全体が新しいオブジェクト群に入れ替わります**。もし関数の引数として渡された`deck`オブジェクトをそのまま使い続けてしまうと、それは「もう配列に含まれていない孤立した参照」になってしまい、後で行う`filename`の更新が実際には保存されない、というバグにつながります。そのため、**`deckId`（文字列）だけを受け取り、必要になるたびに毎回`decks.find(...)`で最新の配列から探し直す**、という設計にしています。

```js
  const cards = deck.cards.map(cardToServerPayload);
  const isFirstPublish = deck.notYetPublished !== false;
  const body = {
    name: deck.name,
    cards,
    guild_id: GUILD_ID,
    session_token: session ? session.session_token : undefined,
    subject: deck.subject || null,                       // ★ 科目ごとのチャンネル振り分け用
    folder_id: deck.folderId || null,                     // ★ フォルダ所属（みんなで共有）
    publisher_id: session ? session.student_id : null,     // ★ 公開者の学籍番号
    publisher_nickname: session ? session.nickname : '匿名', // ★ 公開者のニックネーム
    silent: !isComplete, // ★ 未完成として公開する場合は通知しない
    incomplete: !isComplete, // ★ 未完成フラグをサーバーに保存し、他の人の端末にも表示させる
    first_publish: isFirstPublish, // ★ 追加：通知文言を「公開」/「更新」どちらにするか判定するためのヒント
    choice_mode: deck.choiceMode || null, // ★ 追加：多肢選択デッキかどうか
  };
  if (deck.filename) body.filename = deck.filename;
```
- `cardToServerPayload`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)）を使うことで、多肢選択デッキの選択肢情報も欠けずに送られます。コメントによれば、以前はここだけ独自に固定6フィールドへ詰め替えていたため、多肢選択デッキを初めて公開した瞬間に選択肢データが失われてしまうバグがあったそうです。
- `first_publish`：サーバー側の「`filename`が既に存在するかどうか」だけでは判定できない問題への対応です。「作成中」として先行登録済みのデッキ（`announceNewDeckToServer`で既に`filename`が振られている）が、初めて本当に「公開して保存」されたとき、サーバー側からすると`filename`が既にあるので「更新」と判定されてしまい、通知の文言が「新規公開しました」ではなく「更新されました」という誤った内容になってしまう問題があったそうです。`deck.notYetPublished`（一度も公開フローを経ていないか）を見て、「これが実質的な初回公開かどうか」を明示的にサーバーへ伝えることでこれを解決しています。

```js
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 10000);
    const res  = await fetch(`${API_BASE}save_cards`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body), signal: controller.signal,
    });
    clearTimeout(timer);
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');

    // ★ ここが重要：POST完了までの間にバックグラウンド同期（10秒ごとのポーリング等）
    //   で decks 配列が再び入れ替わっている可能性があるため、必ずこの時点で
    //   もう一度 loadDecks() し、id で探し直してから更新・保存する。
    decks = loadDecks();
    const target = decks.find(d => d.id === deckId);
    if (target) {
      target.filename = data.filename;
      target.count = target.cards.length;
      target.cardsLoaded = true; // ★ 今まさに公開したデッキなのでカード本体は既にこの端末にある
      target.published_by = session ? session.nickname : '匿名';
      target.incomplete = !isComplete; // ★ 一覧の未完成バッジ表示用に保持（サーバーにも保存済み）
      target.notYetPublished = false; // ★ 追加：「公開して保存」を実際に経たので、以降は未完成／公開済みで判定する
      saveDecks(decks);
    }
    renderDeckListUI();
    showBanner(
      isComplete ? '保存して公開しました！' : '未完成として公開しました（通知なし）',
      isComplete ? '#dcfce7' : '#fef9c3',
      isComplete ? '#166534' : '#854d0e',
      isComplete ? Icons.html('checkCircle', {size:15}) : Icons.html('dot', {size:15})
    );
  } catch(e) {
    showBanner('ローカルに保存しました（GitHub同期失敗）', '#fffbeb', '#92400e', Icons.html('save', {size:15}));
  }
}
```
- ここでも「POST完了までの間にバックグラウンドの同期処理で`decks`配列が入れ替わっている可能性がある」ため、更新の直前にもう一度`loadDecks()`で読み直してから対象を探し直す、という同じ慎重さが繰り返されています。CardMaker全体を通じて、**「非同期処理の後は、待っている間に状態が変わっているかもしれないので、古い参照を信用せず必ず最新の状態から探し直す」**という設計方針が徹底されているのが分かります。
- 成功したら`showBanner`（[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で説明する、画面上部に一時的に出るお知らせ）で結果を伝え、失敗時は「ローカルには保存済み」であることを伝えて安心させます（サーバーへの反映は失敗しても、手元のデータが消えるわけではないことを明示）。

---

## 5. 作成済みカード一覧の描画（2354〜2376行）

```js
function renderCreatedList() {
  const deck = decks.find(d => d.id === currentDeckId);
  const section = document.getElementById('created-section');
  const list    = document.getElementById('created-list');
  if (!deck||!deck.cards.length) { section.style.display='none'; return; }
  section.style.display='block';
  list.innerHTML = deck.cards.map((c,i) => `
    <div class="created-item" data-key="${esc(cardKey(c))}">
      <span class="drag-handle" title="ドラッグして並び替え">⠿</span>
      <div class="created-item-num">${i+1}</div>
      <div class="created-item-body">
        <div class="created-item-q">${esc(mathToPlainText(c.question))}</div>
        <div class="created-item-a">${esc(mathToPlainText(c.answer))}</div>
      </div>
      <div class="created-item-btns">
        <button class="btn btn-ghost btn-sm" onclick="openCardEditModal(${i})">編集</button>
        <button class="btn btn-danger btn-sm" onclick="deleteCardFromDeck(${i})">削除</button>
      </div>
    </div>`).join('');
}
```
- 編集画面下部の「作成済みカード」一覧を描画します。各行に`data-key`として`cardKey(c)`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)で説明した安定キー）を持たせておくことで、あとで説明するドラッグ並び替え処理が「どのカードがどこに動いたか」を正しく追跡できるようにしています。
- `mathToPlainText(...)`（[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)で説明）は、理数記号の入力データをプレビュー用の読みやすいテキストに変換する関数です。

このあとの2378〜2403行はコメントのみで、「作成済みカードのドラッグ並び替え」の実体は`Cardmaker-cardreorder.js`（[10_遅延読み込みチャンク_CSVと並び替えと検索.md](10_遅延読み込みチャンク_CSVと並び替えと検索.md)）に分離されている、という説明が書かれています。

---

続きは[05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)で、ホーム画面（デッキ・フォルダ一覧）の長押しドラッグ並び替えの仕組みを解説します。
