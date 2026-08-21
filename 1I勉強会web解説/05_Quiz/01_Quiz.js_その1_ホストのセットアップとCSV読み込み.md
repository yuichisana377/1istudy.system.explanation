# Quiz.js その1：ホストのセットアップ画面・CSV読み込み・ルーム作成（298〜576行）

[00_HTML構造とその1_起動と参加画面.md](00_HTML構造とその1_起動と参加画面.md)の続きです。

---

## 1. セットアップ画面を開く（290〜329行）

```js
let hsAllowLateJoin = false;
function setLateJoinOption(v) {
  hsAllowLateJoin = v;
  document.querySelectorAll('#hs-late-join-toggle .qz-toggle-opt').forEach((b, i) => {
    b.classList.toggle('active', (i === 1) === v);
  });
}
```
- 「途中参加を許可する/しない」トグルです。`(i === 1) === v`は、「2番目のボタン（インデックス1、＝『許可する』）と、選ばれた値`v`が一致していれば`active`にする」という、2つのボタンのうちどちらか一方だけをアクティブにするための、少し圧縮した書き方です。

```js
async function goHostSetupScreen() {
  showScreenQ('host-setup');
  document.getElementById('hs-error').textContent = '';
  setLateJoinOption(false);
  if (launchDeckInfo && launchDeckInfo.filename) {
    document.getElementById('hs-title').value = launchDeckInfo.name || '';
    setHostSource('deck');
    hsSelectedDecks = [{ filename: launchDeckInfo.filename, name: launchDeckInfo.name || launchDeckInfo.filename }];
    hsDeckPickerLocked = true;
    document.querySelectorAll('#hs-source-toggle .qz-toggle-opt').forEach(b => b.disabled = true);
  } else {
    document.querySelectorAll('#hs-source-toggle .qz-toggle-opt').forEach(b => b.disabled = false);
    hsDeckPickerLocked = false;
  }
  renderSelectedDecks();
  if (!document.getElementById('hs-manual-list').children.length) { addManualQuestion(); }
}
```
- [00_HTML構造とその1_起動と参加画面.md](00_HTML構造とその1_起動と参加画面.md)で見た`launchDeckInfo`（CardMakerのデッキメニューから直接来た場合）があれば、タイトルを自動入力し、デッキ選択を**そのデッキに固定**します（`hsDeckPickerLocked = true`にして、問題の作り方トグル自体も押せなくします）。これは「このデッキでクイズを始める」という明確な意図で来ているため、選び直しをさせない、という親切設計です。
- 手動作成の入力欄が1つも無ければ、`addManualQuestion()`（3節）で最低1問分の入力欄を用意しておきます。

```js
function startFreshHostSetup() {
  launchDeckInfo = null;
  hsSelectedDecks = [];
  hsDeckPickerLocked = false;
  goHostSetupScreen();
}
```
- ホーム画面の「クイズを作る（ホスト）」ボタンから呼ばれます。コメントに「前回の選択を持ち越さず、まっさらな状態から始める」とあります。

---

## 2. デッキ選択の表示（331〜369行）

```js
function openDeckPicker() {
  location.href = 'Cardmaker.html?pick=quiz';
}
```
- 「デッキを選ぶ」ボタンは、[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md](../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md)で解説した、CardMakerの「クイズ用デッキ選択モード」へページ遷移するだけの、シンプルな入口です。

```js
function renderSelectedDecks() {
  const emptyWrap = document.getElementById('hs-deck-picker-empty');
  const selectedWrap = document.getElementById('hs-deck-picker-selected');
  const listEl = document.getElementById('hs-selected-deck-list');
  const changeBtn = document.getElementById('hs-change-decks-btn');
  changeBtn.style.display = hsDeckPickerLocked ? 'none' : '';

  if (!hsSelectedDecks.length) {
    emptyWrap.style.display = '';
    selectedWrap.style.display = 'none';
    return;
  }
  emptyWrap.style.display = 'none';
  selectedWrap.style.display = '';
  listEl.innerHTML = hsSelectedDecks.map((d, i) => `
    <div class="qz-deck-chip">
      <span class="qz-deck-chip-name">${escapeHtml(d.name || d.filename)}</span>
      ${hsDeckPickerLocked ? '' : `<button type="button" class="qz-deck-chip-remove" onclick="removeSelectedDeck(${i})"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="#dc2626" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" style="display:inline-block;vertical-align:-3px;flex-shrink:0" aria-hidden="true"><path d="M6 6l12 12"/><path d="M18 6L6 18"/></svg></button>`}
    </div>`).join('');
}
```
- 選ばれているデッキが無ければ「デッキを選ぶ」ボタンだけの表示、あれば選ばれたデッキをチップ（小さなタグ）として並べる表示に切り替えます。デッキ選択が固定されている（`hsDeckPickerLocked`）場合は、削除ボタンも「選び直す」ボタンも表示しません。

```js
function removeSelectedDeck(i) {
  hsSelectedDecks.splice(i, 1);
  renderSelectedDecks();
}

function setHostSource(src) {
  document.querySelectorAll('#hs-source-toggle .qz-toggle-opt').forEach(b => {
    b.classList.toggle('active', b.dataset.src === src);
  });
  document.getElementById('hs-deck-panel').style.display = src === 'deck' ? '' : 'none';
  document.getElementById('hs-manual-panel').style.display = src === 'manual' ? '' : 'none';
}
```
`removeSelectedDeck(i)`（358〜361行）は配列から該当インデックスを取り除いて再描画するだけの単純な関数、`setHostSource(src)`（363〜369行）は「デッキから自動作成」「自分で問題を作る」の2つのパネルを切り替えます。

---

## 3. 手動での問題作成（371〜393行）

```js
let manualQCounter = 0;
function addManualQuestion() {
  manualQCounter++;
  const idx = manualQCounter;
  const wrap = document.createElement('div');
  wrap.className = 'qz-manual-card';
  wrap.dataset.qid = idx;
  const letters = ['A', 'B', 'C', 'D'];
  wrap.innerHTML = `
    <div class="qz-manual-card-head">
      <b>問題 #${document.getElementById('hs-manual-list').children.length + 1}</b>
      <button type="button" class="qz-manual-remove" onclick="removeManualQuestion(${idx})">削除</button>
    </div>
    <input class="qz-input mq-question" placeholder="問題文" maxlength="200">
    ${letters.map((l, i) => `
      <div class="qz-manual-choice-row">
        <input type="radio" name="mq-correct-${idx}" value="${i}" ${i === 0 ? 'checked' : ''}>
        <input class="qz-input mq-choice" placeholder="選択肢 ${l}" maxlength="80">
      </div>`).join('')}
  `;
  document.getElementById('hs-manual-list').appendChild(wrap);
  return wrap;
}
```
- `manualQCounter`という、削除されても再利用されない**通し番号**を`data-qid`として各問題カードに振っています。「今何番目に表示されているか」ではなく、この一意な番号で各カードを識別することで、途中の問題を削除しても、残りのカードの削除ボタンが正しい対象を指し続けます。
- 各問題には`name="mq-correct-<idx>"`が振られたラジオボタン（A〜D）が4つあり、同じ`name`を持つラジオボタン同士は「1つしか選べないグループ」としてブラウザが自動的に扱ってくれます（ネイティブのHTML機能で、JSで排他制御を書く必要がありません）。デフォルトでAが選択済みです。
- この関数は作成した`wrap`要素自体を返り値として返しています。これは4節のCSV読み込み処理が、この戻り値を使って作られたばかりの入力欄に直接値を書き込むために利用されます。

---

## 4. CSVからの一括読み込み（395〜510行）

[../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md](../02_Cardmaker/10_遅延読み込みチャンク_CSVと並び替えと検索.md)で見た`Cardmaker-csvimport.js`の`parseCSV`と、`parseCSV`関数自体は**ほぼ同一のコード**が、このファイルにも独立してコピーされています（BOM除去・ダブルクォートの処理・改行の扱いまで完全に同じロジックです）。

このページ独自なのは、見出し列の検出と正解の解釈です：
```js
const QUIZ_CSV_HEADER_ALIASES = {
  question: ['question', '問題', '問題文', 'q'],
  choiceA:  ['choicea', '選択肢a', 'a'],
  choiceB:  ['choiceb', '選択肢b', 'b'],
  choiceC:  ['choicec', '選択肢c', 'c'],
  choiceD:  ['choiced', '選択肢d', 'd'],
  correct:  ['correct', 'answer', '正解', '正答'],
};
function detectQuizCsvColumns(headerRow) {
  const norm = headerRow.map(h => (h || '').trim().toLowerCase());
  const idx = { question: -1, choiceA: -1, choiceB: -1, choiceC: -1, choiceD: -1, correct: -1 };
  norm.forEach((h, i) => {
    for (const key of Object.keys(QUIZ_CSV_HEADER_ALIASES)) {
      if (idx[key] === -1 && QUIZ_CSV_HEADER_ALIASES[key].includes(h)) idx[key] = i;
    }
  });
  const isHeader = [idx.question, idx.choiceA, idx.choiceB, idx.choiceC, idx.choiceD].every(v => v !== -1);
  return { isHeader, idx };
}
```
- 「問題」列と選択肢A〜D列の**5つすべて**が見出しとして認識できた場合だけ、見出し行ありと判定します（「正解」列は必須条件に含めていません。おそらく正解列が見つからなくても、デフォルトのA正解として取り込みを続行できるようにするための設計です）。

```js
function resolveCorrectIndex(correctRaw, choices) {
  const s = (correctRaw || '').trim();
  if (!s) return 0;
  const letterIdx = { a: 0, b: 1, c: 2, d: 3 }[s.toLowerCase()];
  if (letterIdx !== undefined) return letterIdx;
  const num = Number(s);
  if (Number.isInteger(num) && num >= 1 && num <= 4) return num - 1;
  const matchIdx = choices.findIndex(c => c.trim() === s);
  return matchIdx !== -1 ? matchIdx : 0;
}
```
- 「正解」列の値を、3段階の方法で解釈しようと試みます：①`A`〜`D`の文字、②`1`〜`4`の数字、③選択肢そのものの文言と完全一致するもの。どれにも当てはまらなければ、コメントにある通り「判定できない場合はAを正解にする（取り込み後に一覧で見直せるため）」という、安全側に倒したフォールバックです。

```js
async function handleQuizCsvImport(event) {
  const file = event.target.files[0];
  event.target.value = ''; // 同じファイルを連続選択してもonchangeが発火するようにリセット
  if (!file) return;

  let text;
  try {
    text = await file.text();
  } catch (e) {
    await showConfirm({ title: '読み込みに失敗しました', desc: 'CSVファイルを読み込めませんでした。', okLabel: 'OK', cancelLabel: '閉じる' });
    return;
  }

  const rows = parseCSV(text);
  if (!rows.length) {
    await showConfirm({ title: '読み込めるデータがありません', desc: 'CSVファイルの中身が空のようです。', okLabel: 'OK', cancelLabel: '閉じる' });
    return;
  }

  let { isHeader, idx } = detectQuizCsvColumns(rows[0]);
  const dataRows = isHeader ? rows.slice(1) : rows;
  if (!isHeader) idx = { question: 0, choiceA: 1, choiceB: 2, choiceC: 3, choiceD: 4, correct: 5 };

  let added = 0, skipped = 0;
  for (const r of dataRows) {
    const question = (r[idx.question] || '').trim();
    const choices = [idx.choiceA, idx.choiceB, idx.choiceC, idx.choiceD].map(i => (r[i] || '').trim());
    if (!question || choices.some(c => !c)) { skipped++; continue; }
    const correctRaw = idx.correct !== -1 ? (r[idx.correct] || '') : '';
    const correctIndex = resolveCorrectIndex(correctRaw, choices);

    const wrap = addManualQuestion();
    // ★ .value への代入はmaxlength属性による制限を受けないため、入力欄と
    //   同じ上限（問題文200字・選択肢80字）に合わせて明示的に切り詰める。
    wrap.querySelector('.mq-question').value = question.slice(0, 200);
    [...wrap.querySelectorAll('.mq-choice')].forEach((inp, i) => { inp.value = choices[i].slice(0, 80); });
    [...wrap.querySelectorAll('input[type=radio]')].forEach((rad, i) => { rad.checked = (i === correctIndex); });
    added++;
  }

  const parts = [`${added}問を追加しました`];
  if (skipped) parts.push(`問題文または選択肢が空の${skipped}行をスキップしました`);
  await showConfirm({ title: 'CSVの読み込みが完了しました', desc: parts.join('\n'), okLabel: 'OK', cancelLabel: '閉じる' });
}
```
- 1行ずつ、`addManualQuestion()`（3節）で新しい問題カードを作らせ、その戻り値（`wrap`）に対して直接値を書き込んでいきます。コメントに「読み込んだ内容は`addManualQuestion()`と同じ入力欄に反映されるので、取り込み後に見直し・修正してから作成できる」とある通り、CSV取り込み後も普通に手で編集できる、という一貫した設計です。
- `wrap.querySelector('.mq-question').value = question.slice(0, 200);`：コメントには「`.value`への代入は`maxlength`属性による制限を受けないため、入力欄と同じ上限（問題文200字・選択肢80字）に合わせて明示的に切り詰める」とあります。`maxlength`属性はユーザーがキーボードで入力する際の文字数制限であり、JSから`.value`に直接値を代入する場合はこの制限が適用されません。もしCSVに201文字の問題文があった場合、そのままでは制限を超えた値が入力欄に入ってしまうため、`.slice(0, 200)`で明示的に切り詰めています。

---

## 5. ルームの作成：`submitCreateRoom()`（534〜576行）

```js
const isDeckSrc = document.querySelector('#hs-source-toggle .qz-toggle-opt[data-src="deck"]').classList.contains('active');
const body = withAuth({ title, allow_late_join: hsAllowLateJoin });
if (isDeckSrc) {
  if (!hsSelectedDecks.length) { errEl.textContent = 'デッキを選んでください'; return; }
  body.source = 'deck';
  body.deck_filenames = hsSelectedDecks.map(d => d.filename);
  const numQ = document.getElementById('hs-num-questions').value;
  if (numQ) body.num_questions = Number(numQ);
} else {
  const questions = collectManualQuestions();
  if (!questions.length) { errEl.textContent = '問題を1つ以上入力してください'; return; }
  for (const q of questions) {
    if (!q.question || q.choices.some(c => !c)) { errEl.textContent = '問題文と4つの選択肢をすべて入力してください'; return; }
  }
  body.source = 'manual';
  body.questions = questions;
}
```
- 「デッキから自動作成」か「手動で作成」かで、送信するデータの中身を丸ごと出し分けます。デッキ選択モードでは選ばれたデッキのファイル名一覧と出題数を、手動モードでは`collectManualQuestions()`（519〜532行、各問題カードの入力欄から値を集める関数）の結果を送ります。
- 手動作成の場合、送信前に全問題について「問題文があり、4つの選択肢が全部埋まっているか」をチェックし、1つでも不備があれば全体を送信せず中断します。

```js
const btn = document.getElementById('hs-create-btn');
btn.disabled = true; btn.textContent = '作成中…';
const data = await apiPost('quiz_create', body, 12000);
btn.disabled = false; btn.textContent = 'クイズを作成する';
if (!data.ok) { errEl.textContent = quizErrorText(data.error); return; }
roomCode = data.code;
isHost = true;
renderedQIndex = -1; renderedState = null;
history.replaceState(null, '', `${location.pathname}?code=${data.code}`);
document.getElementById('hl-title').textContent = title || 'みんなでクイズ';
showScreenQ('host-lobby');
startPolling();
```
- タイムアウトを12秒（他の通信の8秒デフォルトより長め）にしているのは、デッキから4択問題を自動生成する処理（サーバー側での計算）に多少時間がかかることを見込んでのことだと考えられます。
- 成功したら、発行されたルームコードをURLに記録し、ホストのロビー画面に切り替えて、ポーリング（[02_Quiz.js_その2_対戦の進行とリアルタイム同期.md](02_Quiz.js_その2_対戦の進行とリアルタイム同期.md)で解説）を開始します。

---

続きは[02_Quiz.js_その2_対戦の進行とリアルタイム同期.md](02_Quiz.js_その2_対戦の進行とリアルタイム同期.md)で、リアルタイムなクイズ進行（ポーリング・出題・回答・結果発表）を解説します。
