# Cardmaker.js その7：学習モードと一人用クイズの入口（3750〜4167行）

[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の続きです。

---

## 1. 一人用クイズへの入口（3750〜3760行）

```js
async function startSoloQuiz(deckId) {
  await loadChunkWithFeedback('quizplay', '/Cardmaker-quizplay.js');
  return startSoloQuiz(deckId);
}
```
- これまで何度も出てきたパターンと同じで、実際のクイズ再生ロジックは`Cardmaker-quizplay.js`（[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)）に分離されており、この関数は読み込みの「仮の窓口」です。

---

## 2. プレイモード選択：`openPlayMode(deckId)`（3762〜3837行）

デッキの「▶プレイ」ボタンを押したときの入口です。

```js
if (isDeckInFolderScope(deckId, QUIZ_ARCHIVE_FOLDER_ID) || deck.choiceMode) {
  if (!deck.filename) return startSoloQuiz(deckId);
  const choice = await showCmChoiceDialog({
    title: deck.name,
    choices: [
      { icon: ..., label: '一人でプレイ', sub: '選択式クイズに一人で挑戦する', value: 'solo' },
      { icon: ..., label: 'みんなでクイズを始める', sub: '友達とオンラインで早押し4択', value: 'multi' },
    ],
  });
  if (choice === 'multi') return startQuizFromDeck(deckId);
  if (choice === 'solo') return startSoloQuiz(deckId);
  return;
}
```
- 「クイズ過去問」フォルダの中のデッキ、または多肢選択デッキは、通常のフラッシュカード学習（すべて/わからないだけ/続きから）ではなく、選択式（4択のような形式）の一人用クイズでプレイします。
- 公開済みデッキであれば、その前に「一人でプレイ」か「みんなでクイズを始める」かを軽く選ばせます。非公開デッキは「みんなでクイズ」が使えない（後述）ため、この選択自体をスキップしていきなり一人用クイズを始めます。

通常のフラッシュカードデッキの場合は、下に続く処理でプレイモード選択モーダル（`modal-play-mode`）を開く準備をします：

```js
await waitForPendingSync(deckId);
let result = await ensureDeckCardsLoaded(deckId, true);
while (!result.ok) {
  const retry = await showCmConfirm({ title: '読み込みに失敗しました', ... });
  if (!retry) return;
  result = await ensureDeckCardsLoaded(deckId, true);
}
```
- ここでも「保留中の同期を待つ→強制的に最新カードを取得」という同じ順序が守られています。失敗した場合は、`while`ループで何度でも「もう一度試す」を選べるようにしています。

```js
document.getElementById('play-mode-quiz-item').style.display = deck.filename ? '' : 'none';
```
- 「みんなでクイズを始める」の項目は、公開済み（`filename`あり）のデッキだけに表示されます。コメントによれば、`Quiz.js`はサーバーの`get_card_set`でデッキを取得する仕組みのため、まだサーバーに登録されていない非公開のローカル限定デッキは対象にできないためです。

```js
function onReverseModeToggleChange() {
  const reversed = document.getElementById('reverse-mode-checkbox').checked;
  const row = document.getElementById('auto-grade-toggle-row');
  row.style.display = reversed ? 'none' : '';
  if (reversed) document.getElementById('auto-grade-checkbox').checked = false;
}
```
- 「問題と解答を反転する」トグルがONの間は、「自動採点」トグル自体を隠し、内部的にも強制的にOFFに戻します。反転モード中に自動採点をしようとすると、採点の対象（何を正解として比較するか）がずれてしまうための保護です。

---

## 3. 学習を開始する：`startStudyMode(mode)`（3849〜3939行）

`mode`は`'all'`（すべて）／`'unsure'`（わからないだけ）／`'resume'`（続きから）のいずれかです。

### 3.1 「続きから」データの上書き確認（3855〜3869行）
```js
if (mode !== 'resume') {
  const existing = loadStudyProgress(studyIsFolder, progressId);
  if (existing) {
    const proceed = await showCmConfirm({
      title: '「続きから」のデータが消えます',
      desc: '保存されている再開位置は破棄され、最初からのプレイになります。\nこのまま始めますか？',
      okLabel: 'このまま始める', cancelLabel: 'キャンセル', okStyle: 'danger',
    });
    if (!proceed) return;
  }
}
```
- 「すべて」や「わからないだけ」を選んだ場合、これから始める処理（後述）が保存されていた「続きから」の再開データを問答無用で消してしまいます。気づかないうちに再開位置が失われてしまわないよう、事前に一言確認を挟んでいます。

### 3.2 「続きから」を選んだ場合（3873〜3900行）
```js
const saved = loadStudyProgress(studyIsFolder, progressId);
if (!saved) return;
studyReverse = saved.reverse;
studyAutoGrade = !saved.reverse && !!saved.autoGrade;
studyMode = saved.mode || 'all';
studyShuffled = !!saved.shuffled;

let pool;
if (studyIsFolder) { pool = []; folderPlayDecks.forEach(d => d.cards.forEach(c => pool.push({ ...c, __deckId: d.id }))); ... }
else { const deck = decks.find(d => d.id === studyDeckId); pool = deck ? [...deck.cards] : []; ... }
const byKey = new Map(pool.map(c => [cardKey(c), c]));
studyCards = saved.order.map(k => byKey.get(k)).filter(Boolean);
if (!studyCards.length) return;
studyIdx = Math.min(saved.idx, studyCards.length - 1);
```
- 保存されていた「並び順（カードキーの配列）」を、**今の最新のカードデータ**から作った`Map`（キー→カード本体、の対応表）と突き合わせて、実際の学習カード配列（`studyCards`）を組み立て直します。こうすることで、保存後に他の人がカードを編集していても、その最新の内容で続きから再開できます。もし途中で削除されたカードがあれば`.filter(Boolean)`で自然に除外されます（`byKey.get(k)`が`undefined`になるため）。
- `studyIdx = Math.min(saved.idx, studyCards.length - 1)`：もしカードが削除されて全体の枚数が減っていた場合でも、保存されていた位置が配列の範囲外にならないよう安全に調整しています。

### 3.3 「すべて」「わからないだけ」を選んだ場合（3901〜3931行）
```js
if (studyIsFolder) {
  const merged = [];
  folderPlayDecks.forEach(d => {
    const unsure = mode === 'unsure' ? getUnsureSet(d.id) : null;
    d.cards.forEach(c => {
      if (mode === 'unsure' && !unsure.has(cardKey(c))) return;
      merged.push({ ...c, __deckId: d.id });
    });
  });
  studyCards = merged;
  ...
} else {
  ...
}
studyIdx = 0;
clearStudyProgress(studyIsFolder, progressId);
```
- フォルダをまとめて学習している場合、各カードに`__deckId`（元々どのデッキに属していたか）を付け足しておきます。これは、フォルダ横断で1つの学習セッションとして扱いながらも、「このカードはどのデッキ由来か」を後から参照できるようにするための工夫です（学習画面での「わからない」マークの保存先や、カード編集時にどのデッキを更新すべきか、を判断するのに使われます）。
- 新しく選び直した場合は、古い「続きから」のデータは矛盾するため`clearStudyProgress`で消しておきます。

---

## 4. 一覧表示画面への入口（3942〜3956行）

`openListView()`も、2節・[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で見た「遅延読み込みチャンクの仮の窓口」パターンと同じです。実体は`Cardmaker-listview.js`（[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)）にあります。

コメントには、この入口を経由しない例外（検索結果からの直接ジャンプ）についても触れられており、「一覧で見る」画面が実際に開くまで押せないはずのボタン（`setListViewFilter`など）は、この入口が読み込みを待つことで間接的に守られている、という設計上の注記があります。

---

## 5. 学習カードの表示：`renderStudyCard()`（3958〜4052行）

### 5.1 元の問題番号バッジ（3958〜3980行）
```js
function updateStudyOriginalNumberBadge(c) {
  let badge = document.getElementById('study-orig-num-badge');
  if (!badge) {
    const label = document.querySelector('.study-q-tag');
    if (!label) return;
    badge = document.createElement('span');
    badge.id = 'study-orig-num-badge';
    ...
    label.appendChild(badge);
  }
  const deckId = c.__deckId || studyDeckId;
  const deck = decks.find(d => d.id === deckId);
  if (!deck) { badge.textContent = ''; return; }
  const origIdx = deck.cards.findIndex(x => cardKey(x) === cardKey(c));
  badge.textContent = origIdx !== -1 ? String(origIdx + 1) : '';
}
```
- シャッフルすると、画面上の「3 / 20」のような進捗表示は「シャッフル後の再生順」の位置になってしまい、元々デッキの何問目だったかが分からなくなります。このバッジは、常に「元のデッキ内での並び順」の番号だけを別途表示するためのものです。
- バッジ用の`<span>`要素はHTMLに最初から用意されているわけではなく、初回だけJSで動的に作って「問題」ラベルの隣に挿入し、以降は使い回します。

### 5.2 学習完了時の処理（3982〜3995行）
```js
if (studyIdx >= studyCards.length) {
  document.getElementById('study-content').style.display = 'none';
  document.getElementById('study-done').style.display    = 'flex';
  ...
  clearStudyProgress(studyIsFolder, progressId);
  saveCompletionRecord(studyIsFolder, progressId, studyCards.length);
  renderInProgressUI();
  return;
}
```
- 全部のカードを見終えたら、完了画面に切り替え、「続きから」のデータは不要になるので消し、代わりに「完了した」という記録（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の`saveCompletionRecord`）を残します。ホーム画面の「プレイ中」「プレイ済み」欄もここで最新化されます。

### 5.3 通常のカード表示（3996〜4052行）
```js
const c = studyCards[studyIdx];
markCardSeen(studyIsFolder ? c.__deckId : studyDeckId, c);
const qText = studyReverse ? c.answer   : c.question;
...
```
- カードを表示するたびに`markCardSeen`（「みんなのわかる率」用の学習済み記録、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)）を呼びます。
- 反転モードなら問題欄と解答欄の中身を入れ替えて表示します。

```js
const answerInputWrap = document.getElementById('study-answer-input-wrap');
answerInputWrap.style.display = '';
answerInput.value = '';
...
document.getElementById('reveal-answer-btn').textContent = studyAutoGrade ? '採点する' : '答えを見る';
```
- コメントによれば、解答入力欄は反転モードかどうかに関わらず**常に**表示されます。反転モード中は自動採点自体が行われませんが（`studyAutoGrade`が常に`false`になるよう別の場所で強制されている）、入力欄自体は「自分で書いてみて確認する」という使い方ができるよう、あえて隠していません。

```js
document.getElementById('study-next').innerHTML = studyIdx === studyCards.length-1 ? ('完了 ' + Icons.html('check', {size:14})) : '次へ →';
updateUnsureBtn();
saveStudyProgress();
```
- 最後のカードなら「次へ」ボタンの文言を「完了」に変え、「わからない」ボタンの見た目を更新し、**カードを表示するたびに毎回**`saveStudyProgress()`（学習の続きを保存）を呼びます。これにより、学習中にアプリを閉じても、次に開いたときにちょうど今見ていたカードから再開できます。

---

## 6. 答えを見る・自動採点（4054〜4097行）

```js
function revealAnswer() {
  document.getElementById('study-answer-panel').classList.add('show');
  document.getElementById('study-reveal-bar').style.display = 'none';
  document.getElementById('study-nav').style.display = '';
  if (studyAutoGrade) gradeCurrentAnswer();
  updateUnsureBtn();
}
```
- 「答えを見る」（自動採点モードなら「採点する」）ボタンが押されたときの処理です。自動採点モードなら`gradeCurrentAnswer()`で判定を行います。

```js
function normalizeAnswerText(s) { return (s || '').toLowerCase().replace(/[\s　]/g, ''); }
function gradeCurrentAnswer() {
  const card = studyCards[studyIdx];
  const input = ...;
  const correctText = studyReverse ? card.question : card.answer;
  const normInput = normalizeAnswerText(input);
  const isCorrect = normInput !== '' && normInput === normalizeAnswerText(correctText);
  ...
  if (!isCorrect) {
    const key = cardKey(card);
    const deckId = card.__deckId || studyDeckId;
    const unsure = getUnsureSet(deckId);
    if (!unsure.has(key)) { unsure.add(key); saveUnsureSet(deckId, unsure); }
  }
}
```
- `normalizeAnswerText`は、比較の前に「小文字化」＋「半角/全角スペースの除去（`\s`が半角の空白、`　`が全角スペースの文字コード）」を行う関数です。これにより、大文字/小文字の違いや余分な空白があっても、内容が同じなら正解と判定できるようにしています（逆に言うと、これ以外の表記ゆれ、例えば送り仮名の違いなどは不正解と判定されます）。
- **不正解だった場合は自動的に「わからない」マークが付きます**（すでに付いていれば何もしません）。逆に正解だったからといって、既存の「わからない」マークを勝手に外すことはしません（コメントにその方針が明記されています）。

---

## 7. 「わからない」ボタンとカード送り（4099〜4152行）

`updateUnsureBtn()`／`toggleUnsure()`は、現在のカードの「わからない」マークの状態を見た目に反映・切り替える関数です（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の`getUnsureSet`/`saveUnsureSet`を使用）。

```js
function isStudyOverlayModalOpen() {
  return !!document.querySelector('[id^="modal-"].open');
}
function studyMove(dir) {
  if (isStudyOverlayModalOpen()) return;
  studyIdx += dir; renderStudyCard();
}
```
- `isStudyOverlayModalOpen()`は、学習画面の上に何らかのモーダル（idが`modal-`で始まる要素で`open`クラスが付いているもの）が開いているかを確認する関数です。コメントに実際にあったバグが記録されています：カード編集モーダルはCardMakerの画面切り替え（`showScreen`）を使わず、学習画面の上に重ねて表示される作りのため、モーダルが開いていても`document.querySelector('.screen.active')?.id`は`'screen-study'`のままになってしまいます。そのため、モーダルを開いた状態で（Androidなどで）キーボードの矢印キー相当の操作をすると、モーダルの裏で学習カードが意図せず先に進んでしまう不具合があったそうです。この関数で明示的にモーダルの有無を確認することで、その事故を防いでいます。

```js
function shuffleStudy() {
  for (let i=studyCards.length-1;i>0;i--) {
    const j = Math.floor(Math.random()*(i+1));
    [studyCards[i],studyCards[j]]=[studyCards[j],studyCards[i]];
  }
  studyIdx = 0;
  studyShuffled = true;
  ...
}
```
- カードをシャッフルする、標準的な「Fisher-Yatesシャッフル」というアルゴリズムです。配列の末尾から順に、「まだ確定していない範囲」からランダムに1つ選んで入れ替える、という操作を繰り返すことで、偏りのない完全なランダム順を作ります。`[a, b] = [b, a]`という書き方は、JSで2つの変数の値を入れ替える定番の書き方（配列の分割代入を利用したテクニック）です。

---

## 8. キーボードショートカット（4154〜4165行）

```js
document.addEventListener('keydown', e => {
  if (document.querySelector('.screen.active')?.id !== 'screen-study') return;
  if (isStudyOverlayModalOpen()) return;
  const tag = document.activeElement?.tagName;
  if (tag === 'INPUT' || tag === 'TEXTAREA') return;
  if (e.key==='ArrowRight') studyMove(1);
  if (e.key==='ArrowLeft' && studyIdx>0) studyMove(-1);
  if (e.key===' ') { e.preventDefault(); revealAnswer(); }
});
```
- PCで学習しているとき、右矢印キーで次のカード、左矢印キーで前のカード、スペースキーで「答えを見る」を操作できます。
- 学習画面を見ていないとき、モーダルが開いているとき、そして**今まさに入力欄（`INPUT`/`TEXTAREA`）にフォーカスが当たっているとき**は、このショートカットを無効化しています。最後の条件が無いと、自動採点の解答入力欄に文字を打とうとして半角スペースを押しただけで「答えを見る」が誤発火してしまう、といった問題が起きるためです。

---

続きは[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で、画像の圧縮・回転補正、モーダルの開閉、そして「遅延読み込みチャンク」の仕組み本体を解説します。
