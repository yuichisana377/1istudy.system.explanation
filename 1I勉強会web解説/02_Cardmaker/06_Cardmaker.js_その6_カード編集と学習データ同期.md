# Cardmaker.js その6：カード編集モーダルと学習データの同期（2982〜3668行）

[05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)の続きです。

---

## 1. カードの削除・編集終了（2982〜3002行）

`deleteCardFromDeck(idx)`は確認ダイアログのあと`deck.cards.splice(idx, 1)`（配列から指定位置の要素を1個取り除く命令）でカードを削除し、サーバー登録済みならすぐ同期します。理由は追加のときと同じで、反映しないと次に開いたときの強制リロードで削除前のカードが復活してしまうためです。

`confirmLeaveEdit()`は編集画面の「←」ボタンから呼ばれ、確認してから一覧に戻ります。

---

## 2. カード編集モーダル（3004〜3271行）

作成済みカードを個別に編集するための、共通のモーダルウィンドウです。デッキ編集画面からも、学習中の画面（「このカードを編集」ボタン）からも、同じモーダルが使われます。

### 2.1 開く前の最新化：`reloadCardBeforeEdit(deckId)`（3017〜3034行）
```js
async function reloadCardBeforeEdit(deckId) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck || !deck.filename) return true;
  await waitForPendingSync(deckId);
  return await loadDeckCardsWithRecovery(deckId);
}
```
- 学習中の画面からカードを編集する場合に呼ばれます。他の人が先に同じデッキを編集して公開していた場合、古いキャッシュのまま編集して保存すると、その人の変更を上書きして消してしまう危険があるため、モーダルを開く前に必ず最新化します。ここでも「まず保留中の同期を待つ→それから強制リロード」という、これまで何度も出てきた慎重な順序が守られています。

### 2.2 デッキ編集画面からの場合は最新化を省略：`openCardEditModal(idx)`（3036〜3052行）
```js
async function openCardEditModal(idx) {
  const deck = decks.find(d => d.id === currentDeckId);
  if (!deck) return;
  const freshCard = deck.cards[idx];
  if (!freshCard) return;
  openCardEditModalCommon(deck.id, freshCard, 'editor');
}
```
- こちらは強制リロードを**しません**。コメントに理由が説明されています：`openEditDeck()`がこの編集セッションの最初にすでにサーバーから最新化済みで、それ以降の追加・削除もローカル→サーバーの順で同期キューに積まれているため、「今まさにこの端末で開いている編集画面の中」であれば、ローカルの内容が最新であることは保証されています。もしここで毎回強制的にサーバーから取り直すと、直前に送った変更がサーバー側にまだ反映しきっていない（数秒かかることがある）タイミングでは、その反映前の古い内容で上書きしてしまい、「10番のカードを作って次へ→すぐ6番を編集」のような操作でカードが消える事故につながっていた、という過去の不具合が記録されています。

### 2.3 学習中に編集：`editCurrentStudyCard()`（3054〜3071行）
今表示中のカードを、`reloadCardBeforeEdit`で最新化してから編集モーダルを開きます。最新化した結果、もしそのカードが既に（他の人によって）削除されていたら、「このカードは既に削除されています」と伝えて処理を中断します。

### 2.4 モーダルの中身を組み立てる：`openCardEditModalCommon(deckId, c, context)`（3091〜3138行）
```js
function openCardEditModalCommon(deckId, c, context) {
  editingDeckId  = deckId;
  editingCardKey = cardKey(c);
  editingContext = context;
  // ★ choices を持つカードは、通常の解答/解説欄の代わりに選択肢入力欄を表示する。
  //   単一/複数正解はデッキ／カードのどちらにもモードを持たせず、常にチェックボックスで
  //   表示し、チェックした個数だけで自動的に決まる。
  editingIsQuizChoice = Array.isArray(c.choices) && c.choices.length >= CHOICE_MIN;

  document.getElementById('modal-edit-q').value = mathToPlainText(c.question);
  autoResize(document.getElementById('modal-edit-q'));
  document.getElementById('modal-edit-q').dispatchEvent(new Event('input', { bubbles: true }));

  document.getElementById('modal-edit-answer-block').style.display      = editingIsQuizChoice ? 'none' : '';
  document.getElementById('modal-edit-choices-block').style.display     = editingIsQuizChoice ? '' : 'none';
  document.getElementById('modal-edit-explanation-block').style.display = editingIsQuizChoice ? 'none' : '';

  if (editingIsQuizChoice) {
    // ★ 旧形式（correct_index単数）のカードにも対応する
    const correctIndices = Array.isArray(c.correct_indices) ? c.correct_indices
      : (typeof c.correct_index === 'number' ? [c.correct_index] : []);
    renderChoiceEditorRows('modal-edit-choice', c.choices, correctIndices);
    // ★ 問題文の画像だけはこのモードでも使う（imgs_q）
    editImgBuf = { q: [...(c.imgs_q || [])], a: [], e: [] };
    renderModalImgStrip('q');
  } else {
    document.getElementById('modal-edit-a').value = mathToPlainText(c.answer);
    document.getElementById('modal-edit-e').value = mathToPlainText(c.explanation||'');
    ['modal-edit-a','modal-edit-e'].forEach(id => {
      const el = document.getElementById(id);
      autoResize(el);
      el.dispatchEvent(new Event('input', { bubbles: true })); // ★ 既存の数式プレビューを反映させる
    });

    // ★ 追加：既存の画像をモーダル専用バッファへコピーして表示する
    //   （元の配列を直接触らず、保存時にまとめて書き戻すため）
    editImgBuf = {
      q: [...(c.imgs_q || [])],
      a: [...(c.imgs_a || [])],
      e: [...(c.imgs_e || [])],
    };
    ['q','a','e'].forEach(k => renderModalImgStrip(k));
  }

  document.getElementById('card-edit-ok').style.display  = 'none';
  document.getElementById('card-edit-err').style.display = 'none';
  openModal('modal-card-edit');
}
```
- カードが`choices`（選択肢）を持っていれば多肢選択カードとして扱い、通常の解答/解説欄の代わりに選択肢入力欄を表示します。
- `correct_index`（単数形、古いデータ形式）と`correct_indices`（複数形、現在の形式）の両方に対応しているのは、過去のデータ形式を今も引き続き正しく開けるようにするための後方互換コードです。
- 画像は「モーダル専用のバッファ（`editImgBuf`）」にコピーしてから編集します。元の配列を直接いじらず、保存ボタンを押したときにまとめて書き戻す設計です（キャンセルした場合に元のデータを傷つけない）。

### 2.5 保存：`saveCardEdit()`（3140〜3214行）
```js
async function saveCardEdit() {
  const q = document.getElementById('modal-edit-q').value.trim();
  const errBar = document.getElementById('card-edit-err');

  // ★「クイズ過去問」デッキの4択カードは、通常の解答/解説とは別の保存経路にする
  if (editingIsQuizChoice) {
    return saveQuizChoiceCardEdit(q, errBar);
  }

  const a = document.getElementById('modal-edit-a').value.trim();
  const e = document.getElementById('modal-edit-e').value.trim();
  if (!q || !a) {
    errBar.innerHTML = Icons.html('close', {size:14}) + ' 問題文と解答は必須です';
    errBar.style.display = 'block';
    setTimeout(() => errBar.style.display = 'none', 3000);
    return;
  }
  if (await warnIfBugChars(q, 'modal-edit-q')) return;
  if (await warnIfBugChars(a, 'modal-edit-a')) return;
  if (await warnIfBugChars(e, 'modal-edit-e')) return;
  const deck = decks.find(d => d.id === editingDeckId);
  if (!deck) { closeModal('modal-card-edit'); return; }
  const idx = deck.cards.findIndex(c => cardKey(c) === editingCardKey);
  if (idx === -1) { closeModal('modal-card-edit'); return; }

  if (await warnIfDuplicateOrSameCard(deck, q, a, e, idx)) return;

  // 既存オブジェクトを直接書き換える
  const card = deck.cards[idx];
  card.question    = q;
  card.answer      = a;
  card.explanation = e;
  // ★ 追加：画像もモーダルバッファから書き戻す
  card.imgs_q = [...editImgBuf.q];
  card.imgs_a = [...editImgBuf.a];
  card.imgs_e = [...editImgBuf.e];

  // ★ studyCards 側（プレイ中の配列）にも同期する。
  //   以前は deck.cards と同じオブジェクト参照だったため自動的に反映されていたが、
  //   カード編集前に deck.cards を丸ごと読み込み直すようになったため、
  //   もはや同じ参照とは限らない。取りこぼさないよう明示的に書き戻す。
  const studySameIdx = studyCards.findIndex(sc =>
    cardKey(sc) === editingCardKey && (sc.__deckId || studyDeckId) === deck.id
  );
  if (studySameIdx !== -1) {
    const sc = studyCards[studySameIdx];
    sc.question = q; sc.answer = a; sc.explanation = e;
    sc.imgs_q = [...editImgBuf.q]; sc.imgs_a = [...editImgBuf.a]; sc.imgs_e = [...editImgBuf.e];
  }

  saveDecks(decks);
  closeModal('modal-card-edit');

  if (editingContext === 'study') {
    refreshStudyCardDisplay(card);
  } else if (editingContext === 'listview') {
    // ★ 追加：一覧表示画面から編集した場合はその一覧を再描画する
    renderListView();
  } else {
    renderCreatedList();
  }

  // ★ 公開済みならサーバー（GitHub）側にも反映する
  if (deck.filename) {
    const ok = await queueSyncDeckToServer(deck);
    if (ok) {
      showBanner('保存しました', '#dcfce7', '#166534', Icons.html('save', {size:15}));
    } else {
      showBanner('サーバーへの反映に失敗しました（ローカルには保存済み）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
    }
  } else {
    // 未公開デッキはローカル保存のみ
    showBanner('保存しました（ローカル）', '#dcfce7', '#166534', Icons.html('save', {size:15}));
  }
}
```
- 選択式カードの場合は先頭で別の関数（2.6節）に処理を委ねます。
- `editingCardKey`（開いたときに記録しておいた`cardKey`）を使って、対象カードを配列の中から探し直します（インデックス番号を直接覚えておくのではなく、内容から作られる安定キーで探すことで、途中で並び替えが起きていても正しいカードを見つけられます）。
- **学習中の配列（`studyCards`）にも同じ変更を書き戻しています**。コメントによれば、以前は`deck.cards`と同じオブジェクトを参照していたため自動的に反映されていましたが、カード編集の前に`deck.cards`を丸ごと読み込み直すよう変更した結果、もはや「同じオブジェクト」とは限らなくなったため、明示的にコピーする必要が生じた、という経緯が説明されています。
- 保存後、どこから開かれたか（`editingContext`：`'study'`/`'listview'`/それ以外）に応じて、更新すべき画面を出し分けます。

### 2.6 選択式カードの保存：`saveQuizChoiceCardEdit(q, errBar)`（3217〜3266行）
考え方は同じですが、選択肢の入力チェック（すべて埋まっているか、正解が1つ以上選ばれているか）を行い、`answer`欄も正解の選択肢文言から自動的に作り直します。`delete card.correct_index`で、もし古い形式のフィールドが残っていれば片付ける、という掃除も行っています。

### 2.7 学習画面の表示だけを更新：`refreshStudyCardDisplay(c)`（3269〜3287行）
```js
function refreshStudyCardDisplay(c) {
  // ★ 反転モードなら問題⇔解答を入れ替えて表示する（データ自体は変えない）
  const qText = studyReverse ? c.answer   : c.question;
  const qImgs = studyReverse ? c.imgs_a   : c.imgs_q;
  const aText = studyReverse ? c.question : c.answer;
  const aImgs = studyReverse ? c.imgs_q   : c.imgs_a;

  setMathText(document.getElementById('study-q-text'), qText);
  renderImgList(document.getElementById('study-q-imgs'), qImgs);
  setMathText(document.getElementById('study-a-text'), aText);
  renderImgList(document.getElementById('study-a-imgs'), aImgs);
  const explWrap = document.getElementById('study-expl-wrap');
  if (c.explanation) {
    setMathText(document.getElementById('study-e-text'), c.explanation);
    explWrap.style.display = '';
  } else {
    explWrap.style.display = 'none';
  }
}
```
- 「問題と解答を逆にする」反転モードが有効なら、表示する内容そのものを入れ替えます（データ自体は変更しません）。カードをめくった状態（表か裏か）は維持したまま、内容だけを最新化する関数です。

---

## 3. デッキ名の変更（3290〜3375行）

```js
let renamingDeckId = null;
async function openRename(id) {
  renamingDeckId = id;
  const deck = decks.find(d => d.id === id);
  const currentSubject = deck.subject || '';
  const currentName = currentSubject && deck.name.startsWith(currentSubject + ' ')
    ? deck.name.slice(currentSubject.length + 1) : deck.name;
  document.getElementById('modal-rename-input').value = currentName;
  // ★ 追加：まだサーバー未登録（非公開・作成中のローカル下書き）のデッキだけ
  //   「公開予定」トグルを表示する。既にサーバー登録済み（filenameあり）の
  //   デッキは、公開予定を取り消したい場合は既存の「非公開に戻す」メニューを使う。
  const planRow = document.getElementById('modal-rename-plan-publish-row');
  if (!deck.filename) {
    planRow.style.display = '';
    document.getElementById('modal-rename-plan-publish').checked = deck.planPublish !== false;
  } else {
    planRow.style.display = 'none';
  }
  const sel = document.getElementById('modal-rename-subject');
  sel.innerHTML = '<option value="">読み込み中…</option>';
  openModal('modal-rename');
  try {
    // ★ cache: 'no-store' を追加
    const res  = await fetch(`${API_BASE}channels?guild_id=${GUILD_ID}`, { cache: 'no-store' });
    const data = await res.json();
    if (!data.ok || !data.channels.length) throw new Error();
    sel.innerHTML = '<option value="">科目なし</option>' +
      data.channels.map(c =>
        `<option value="${esc(c.name)}"${c.name === currentSubject ? ' selected' : ''}>${esc(c.name)}</option>`
      ).join('');
  } catch(e) {
    // ★ 修正：currentSubject（deck.subject）はデッキ作成者が自由に設定できる文字列。
    //   未エスケープでinnerHTMLへ挿入するとXSSになるため esc() を通す。
    sel.innerHTML = `<option value="${esc(currentSubject)}">${esc(currentSubject) || '（取得失敗）'}</option>`;
  }
  setTimeout(() => document.getElementById('modal-rename-input').focus(), 150);
}
```
- デッキ名から科目名の接頭辞を取り除いて入力欄に表示します（一覧の表示ロジックと同じ考え方）。
- 「公開予定」トグルは、まだサーバー未登録（下書き）のデッキのときだけ表示します。すでに公開済みのデッキを非公開に戻したい場合は、別の「非公開に戻す」メニューを使う設計です。
- 科目名（`select`欄）はサーバーの`channels`から取得しますが、失敗した場合は現在の科目名（`currentSubject`）だけを選択肢として表示します。`currentSubject`はデッキ作成者が自由に設定できる文字列なので、`esc()`を通さずに`innerHTML`へ入れると保存型XSSになる、という点にコメントで注意が促されています。

```js
async function saveRename() {
  const subject = document.getElementById('modal-rename-subject').value;
  const input   = document.getElementById('modal-rename-input').value.trim();
  if (!input) return;
  if (await warnIfBugChars(input, 'modal-rename-input')) return;
  const deck = decks.find(d => d.id === renamingDeckId);
  const newName = subject ? `${subject} ${input}` : input;
  // ★ 追加：まだサーバー未登録のデッキのみ、公開予定トグルの変更を反映する
  const wasPlanPublish = deck.planPublish !== false;
  let planPublishChanged = false;
  if (!deck.filename) {
    const nowPlanPublish = document.getElementById('modal-rename-plan-publish').checked;
    if (nowPlanPublish !== wasPlanPublish) planPublishChanged = true;
    deck.planPublish = nowPlanPublish;
  }
  deck.subject = subject;
  deck.name    = newName;
  saveDecks(decks);
  closeModal('modal-rename');
  renderDeckListUI();

  // ★ 追加：公開予定が「なし→あり」に変わった場合、この時点でサーバーへ登録し、
  //   他の人の一覧にも「作成中」として表示されるようにする。
  if (!deck.filename && planPublishChanged && deck.planPublish) {
    await announceNewDeckToServer(deck.id);
    renderDeckListUI();
  }

  // ★ 公開済みならサーバー側のファイルも更新する（通知はしない）
  //   ※ カード本体が未読み込みでも、renameだけならcardsが空でも
  //     サーバー側は既存ファイルの中身を維持したまま名前だけ変えたいところだが、
  //     save_cards は cards を丸ごと上書きする仕様なので、未読み込みのまま
  //     送るとカードが消えてしまう。そのため rename 前に必ず読み込んでおく。
  //   ★ 修正：cardsLoaded=true のキャッシュがあっても古い可能性があるため必ず
  //     最新化する。失敗時も loadDeckCardsWithRecovery が回復手段を提示するので、
  //     rename操作だけがずっとできなくなる、ということはない。
  if (deck.filename) {
    // ★ 追加：この直後の強制リロードでローカルの変更（追加/削除など未同期分）が
    //   消えてしまわないよう、まず直前の同期処理が終わるのを待つ。
    await waitForPendingSync(deck.id);
    const loaded = await loadDeckCardsWithRecovery(deck.id);
    if (!loaded) {
      showBanner('名前の変更はローカルには反映されています（サーバーへの反映は未実施）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
      return;
    }
    const ok = await queueSyncDeckToServer(deck);
    if (!ok) showBanner('サーバーへの名前変更の反映に失敗しました', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
  }
}
```
- 「公開予定」が新たにONにされた場合、この時点でサーバーに先行登録します（[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`announceNewDeckToServer`）。
- すでに公開済みのデッキの名前を変えるときは、コメントにある通り注意が必要です：サーバーの`save_cards`は「カードを丸ごと上書きする」仕様なので、名前だけ変えたいつもりでも、もしカード本体が未読み込みのまま送ってしまうと、サーバー側のカードが空っぽで上書きされてしまいます。そのため、名前変更の前に必ずカード本体を最新化してから送信しています。

---

## 4. サーバー同期を「順番待ちの列」にする仕組み（3361〜3416行）

CardMakerのあちこちで登場してきた`queueSyncDeckToServer`と`waitForPendingSync`の正体がここで説明されています。

```js
const deckSyncPromises = new Map();
function queueSyncDeckToServer(deck) {
  const prev = deckSyncPromises.get(deck.id) || Promise.resolve();
  const next = prev.then(() => syncDeckToServer(deck)).catch(() => false);
  deckSyncPromises.set(deck.id, next);
  return next;
}
async function waitForPendingSync(deckId) {
  const pending = deckSyncPromises.get(deckId);
  if (pending) { try { await pending; } catch(e) {} }
}
```
- `deckSyncPromises`は「デッキID → そのデッキに対する直近のサーバー同期処理」を覚えておく`Map`（キーと値の対応を管理する入れ物）です。
- `queueSyncDeckToServer`が呼ばれるたびに、「前回の同期処理（`prev`）が終わったら、次の同期処理を始める」という形で**同期処理を1本の列に並べて直列につなげていきます**。`.then(() => ...)`は「前の処理が終わったらこれを実行する」という書き方です。
- コメントに、なぜこの仕組みが必要かの理由が書かれています：カードを次々に追加・削除する場面（例：10問連続で作成）では、1回ごとの同期完了を律儀に待っていると操作のテンポが悪くなるので、待たずに次々進められるようにしたい。しかし、「作成済みリストから別のカードをタップして編集する」ときは、直前の追加分の同期がまだ完了していない状態で強制的にサーバーから読み込み直すと、その追加分がまだサーバーに存在しないまま古い内容で上書きされて消えてしまいます。
- そこで、「同期処理は`queueSyncDeckToServer`を通して裏で順番に進めておく（普段は待たない）」「強制的にサーバーから読み込み直す直前にだけ`waitForPendingSync`でその完了を待ち合わせる」という2段構えにすることで、**操作のテンポの良さ**と**データが消えない安全性**を両立させています。これがCardMaker全体で繰り返し出てきた「まず`waitForPendingSync`、それから強制リロード」というパターンの正体です。

```js
async function syncDeckToServer(deck) {
  try {
    const cards = deck.cards.map(cardToServerPayload); // ★ choices/correct_indicesも含めて画像も同期する
    const session = getLoginSession();
    const res = await fetch(`${API_BASE}save_cards`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: deck.name,
        cards,
        filename: deck.filename,
        guild_id: GUILD_ID,
        session_token: session ? session.session_token : undefined,
        subject: deck.subject || null,
        folder_id: deck.folderId || null, // ★ フォルダ所属（みんなで共有）
        publisher_id: session ? session.student_id : null,
        publisher_nickname: deck.published_by || (session ? session.nickname : '匿名'),
        silent: true, // ★ 通知しない
        incomplete: !!deck.incomplete, // ★ 未完成フラグを維持したままサーバーへ反映する
        choice_mode: deck.choiceMode || null, // ★ 多肢選択デッキかどうかを維持したまま反映する
      }),
      signal: AbortSignal.timeout(10000),
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
    deck.count = deck.cards.length;
    saveDecks(decks);
    return true;
  } catch (e) {
    return false;
  }
}
```
`syncDeckToServer(deck)`自体（3385〜3416行）は、実際にデッキの内容をサーバーの`save_cards`に送信する処理です。`silent: true`（通知なし）で送られるのがポイントで、カードを1枚編集するたびにDiscordへ通知が飛ぶと煩わしいため、通知が飛ぶのは「公開して保存」（[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)の`publishDeck`）のときだけに限定されています。

---

## 5. 学習データの端末間共有（3418〜3466行）

```js
const STUDY_DATA_CACHE_KEY = 'cardmaker_study_data_cache_v1';
function loadStudyDataCache() {
  try {
    const d = JSON.parse(localStorage.getItem(STUDY_DATA_CACHE_KEY));
    if (d && typeof d === 'object') {
      return { unsure: d.unsure || {}, progress: d.progress || {}, completed: d.completed || {}, seen: d.seen || {} };
    }
  } catch (e) {}
  return { unsure: {}, progress: {}, completed: {}, seen: {} };
}
function saveStudyDataCache() {
  try { localStorage.setItem(STUDY_DATA_CACHE_KEY, JSON.stringify(studyDataCache)); } catch (e) {}
}
let studyDataCache = loadStudyDataCache();
```
- 「わからない」マーク・学習の続き（進捗）・完了記録の3種類をまとめて`studyDataCache`という1つのオブジェクトで管理し、`localStorage`にキャッシュしています。コメントによれば、以前はブラウザの`localStorage`だけに保存していたため、別の端末を使うと見えなかった問題があり、ログイン（学籍番号）に紐付けてサーバー側にも保存するように変更した経緯があります。

```js
function studyDataDeckKey(deck) {
  if (!deck) return null;
  return deck.filename || ('local:' + deck.id);
}
```
- ここに重要なバグ修正の記録があります。以前は、この端末専用のローカルID（`deck.id`）をそのままサーバー同期のキーとして使っていました。しかし`deck.id`は「この端末が初めてそのデッキを見たときに、その場で採番するローカル限定のID」であり、**同じ公開済みデッキでも、端末（あるいは同じ端末の別ブラウザ）ごとにバラバラの値**になってしまいます。そのため、ある端末で付けた「わからない」マークが、そのデッキを別のIDとして認識している他の端末には決して同じキーとして現れず、「別端末で変更しても反映されない」という不具合が起きていました。修正後は、公開済みデッキなら**サーバーが発行し以降ずっと変わらない`deck.filename`**を必ずキーに使うようにし、まだ公開していないローカル限定のデッキだけは（他の端末には存在しないデータなので）引き続きローカルIDを使う、という区別になっています。

---

## 6. サーバーとのやり取り（3486〜3534行）

`fetchAndMergeStudyData()`と`pushStudyDataToServer(path, body)`は、これまで見てきたパターン（`AbortController`でタイムアウト、`cache: 'no-store'`）と同じ考え方の取得・送信用関数です。1点注目すべきなのは：

### 追記（2026/08/26）：送信中に定期同期が割り込んで変更が消える不具合

「わからないボタンを押しても、次のプレイ時にリセットされてしまう」という報告を受けて調査した結果、以下の競合が原因と判明した：

1. `saveUnsureSet()`（[07_Cardmaker.js_その7](07_Cardmaker.js_その7_学習モードとクイズ再生.md)参照）が`pushStudyDataToServer('save_unsure', ...)`を呼ぶが、これは`await`されない“投げっぱなし”の非同期処理。
2. 一覧画面（`screen-list`）を見ている間は`checkStudyDataUpdate()`（[09_Cardmaker.js_その9](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)参照）が15秒おきに`fetchAndMergeStudyData()`を呼び、サーバーの内容で`studyDataCache`を**丸ごと上書き**する。
3. 1の送信がまだサーバーに届く前に2が実行されると、サーバー側はまだ更新されていない（＝わからないマークが付く前の）古い状態を返してくるため、それで上書きされ、直前に押したはずのマークがローカルからも消えてしまう。

`pushStudyDataToServer`が返すPromiseを`_pendingStudyDataPushes`という配列で覚えておき、`fetchAndMergeStudyData()`側で「送信中のものがあれば、それらが片づくまで待ってから取得する」ように修正した：

```js
let _pendingStudyDataPushes = [];

async function pushStudyDataToServer(path, body) {
  const session = getLoginSession();
  if (!session || !session.session_token) return false;
  const promise = _pushStudyDataToServerImpl(path, body);
  _pendingStudyDataPushes.push(promise);
  try {
    return await promise;
  } finally {
    _pendingStudyDataPushes = _pendingStudyDataPushes.filter(p => p !== promise);
  }
}
```

```js
async function fetchAndMergeStudyData() {
  const session = getLoginSession();
  if (!session || !session.session_token) return false;
  if (_pendingStudyDataPushes.length) {
    await Promise.allSettled(_pendingStudyDataPushes);
  }
  try {
    ...（以下は変更なし）...
```

- 実際の送信処理は`_pushStudyDataToServerImpl`という別関数に切り出し、`pushStudyDataToServer`自身はPromiseの登録・後始末だけを担当する形にした（呼び出し元からは今まで通り`pushStudyDataToServer(...)`を呼ぶだけでよく、シグネチャは変えていない）。
- `Promise.allSettled`を使っているため、送信が失敗した場合でも（成功・失敗を問わず）待ち終わった時点で取得へ進む。送信自体が失敗した場合はサーバー側が更新されないため、結局その回だけは上書きでローカルの変更が失われる可能性が残るが、これは「そもそも送信できていない＝オフライン等の別問題」であり、今回直したかった「送信は成功しているのにタイミングだけで消える」という理不尽な失われ方は解消される。
- この修正はCardMakerの4択機能とは無関係の、以前からの既存コードに潜んでいたバグだった。

#### さらに追記（同日）：取得の通信中に新しい変更が始まった場合の再チェック

コードレビューで、上の修正だけではまだ狭い隙間が残っていることが分かった：`fetchAndMergeStudyData`が「送信中のものが無い」ことを確認して`fetch`を開始した**その後**（＝通信している間）に新しい「わからない」操作が始まった場合、待ち合わせのタイミングを過ぎているのでその新しい変更を待たない。`fetch`が返ってきた時点ではまだサーバーがその新しい変更を反映していないかもしれず、その古いスナップショットで上書きしてしまう可能性が残っていた。

応答を受け取った直後、上書きする直前にもう一度`_pendingStudyDataPushes`を確認し、何かあれば上書きせずに諦める（`return false`）よう追加した：

```js
const data = await res.json();
if (!data.ok) return false;
// ★ 追加：通信中に新しい変更が始まっていたら、この応答は「その変更を
//   含まない古いスナップショット」かもしれないので、上書きせずに諦める
//   （＝ローカルの変更を守る。次回のポーリングで改めて取得し直せば十分）。
if (_pendingStudyDataPushes.length) return false;
studyDataCache = { ... };
```

同じ強化を、[05_Cardmaker.js_その5](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)で解説した`fetchAndMergeOrder`（並び順の同期）にも適用した。

#### さらに追記（同日）：「わかる率」が付けたり外したりするうちにどんどん下がる不具合

「わからないを付けたり外したりしていると、わかる率がどんどん下がっていく（外しても戻らない）」という報告を受けて、さらに深掘りした。「わかる率」（`/deck_understanding`）は常にその時点の`seen`（学習済み）と`unsure`（わからない、現在マーク中のものだけ）から**都度計算し直す**設計（[19_FlaskAPI_学習進捗データ](../../1I勉強会bot解説/19_FlaskAPI_学習進捗データ/00_解説.md)参照。累積ではなく、常に「今マークされているかどうか」の割合）なので、`unsure`の中身さえ正しければ外した瞬間に率も戻るはずだった。

原因は`pushStudyDataToServer`が**直列化されていなかった**こと。「わからない」をONにしてすぐOFFにする、といった短時間の連続操作では、2つの`POST /save_unsure`リクエストがほぼ同時に発行される。`fetch()`はブラウザ・ネットワークの都合で発行順どおりに完了するとは限らず、サーバー側もFlaskのスレッドで並行に処理するため、**先に発行した「ONにする」リクエストの処理が、後から発行した「OFFにする」リクエストより遅れて完了する**（＝追い越す）ことがあった。`/save_unsure`はサーバー側で該当デッキの`unsure`配列を**丸ごと置き換える**方式（差分適用ではない）なので、追い越しが起きると「OFFにする」の結果を「ONにする」が後から上書きしてしまい、外したはずのマークが復活する。これが繰り返されると、外す操作が確率的に効かなくなっていき、率が下がる方向にだけ偏っていく。

同じ生徒のファイルへの保存はすべて`pushStudyDataToServer`を通るため、ここで直列化（前回の送信が完全に完了するまで次を送信しない）するよう修正した：

```js
let _studyDataPushChain = Promise.resolve();

async function pushStudyDataToServer(path, body) {
  const session = getLoginSession();
  if (!session || !session.session_token) return false;
  const waitForTurn = _studyDataPushChain.catch(() => {});
  let myTurnDone;
  _studyDataPushChain = new Promise(resolve => { myTurnDone = resolve; });
  await waitForTurn;
  const promise = _pushStudyDataToServerImpl(path, body);
  _pendingStudyDataPushes.push(promise);
  try {
    return await promise;
  } finally {
    _pendingStudyDataPushes = _pendingStudyDataPushes.filter(p => p !== promise);
    myTurnDone();
  }
}
```

- 呼ばれるたびに「自分の番」を表す新しいPromise（`myTurnDone`で解決する）を`_studyDataPushChain`にセットし直し、**前回の番**（`waitForTurn`）が終わるまで`await`で待ってから実際の送信に入る、という数珠つなぎ（チェーン）方式。これにより、発行した順番どおりにしかサーバーへ届かなくなる（前の送信が完全に終わる＝レスポンスまで受け取るまで、次の送信を開始すらしない）。
- `.catch(() => {})`は保険（`_pushStudyDataToServerImpl`自体は例外を投げない設計だが、万一チェーンのどこかが失敗しても後続の番待ちが永久に詰まらないようにするため）。
- 同じ理由で`pushSharedOrderToServer`（並び順、[05_Cardmaker.js_その5](05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md)参照）にも同じ直列化を適用した。
- ★ この修正は同じブラウザタブ内での連続操作を対象にしたもの。複数端末・複数タブから同時に同じ生徒のデータを書き換えるケース（稀）まで完全に順序保証するには、サーバー側にもより厳密な仕組みが必要になるが、今回報告された「同じ画面内で付けたり外したりする」という使い方に対しては、これで解消する。

以下は修正前のオリジナル実装（考え方は同じ）：

```js
async function fetchAndMergeStudyData() {
  const session = getLoginSession();
  if (!session || !session.session_token) return false;
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 5000);
    // ★ session_tokenはURLクエリに載せない（ブラウザ履歴・アクセスログ・Refererに
    //   残るリスクがあるため）。Authorizationヘッダで送る。
    const qs = new URLSearchParams({ guild_id: GUILD_ID });
    const res = await fetch(`${API_BASE}get_study_data?${qs.toString()}`, {
      signal: controller.signal, cache: 'no-store',
      headers: { 'Authorization': 'Bearer ' + session.session_token },
    });
    clearTimeout(timer);
    const data = await res.json();
    if (!data.ok) return false;
    studyDataCache = {
      unsure:    data.data.unsure    || {},
      progress:  data.data.progress  || {},
      completed: data.data.completed || {},
      seen:      data.data.seen      || {},
    };
    saveStudyDataCache();
    return true;
  } catch (e) {
    return false; // 通信失敗時はローカルキャッシュのまま使い続ける
  }
}

// 学習データをサーバーへ送る共通処理（失敗しても操作自体は止めない）
async function _pushStudyDataToServerImpl(path, body) {
  const session = getLoginSession();
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 8000);
    const res = await fetch(`${API_BASE}${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: session.session_token, ...body }),
      signal: controller.signal,
    });
    clearTimeout(timer);
    const data = await res.json();
    return !!data.ok;
  } catch (e) {
    return false;
  }
}
```
- コメントに「`session_token`はURLクエリに載せない（ブラウザ履歴・アクセスログ・Refererに残るリスクがあるため）。`Authorization`ヘッダーで送る」とあります。ログインを証明するトークンのような秘密情報は、URLの一部（`?token=...`のような形）に含めると、ブラウザの閲覧履歴やサーバーのアクセスログにそのまま記録として残ってしまう危険があるため、URLではなく通信の「ヘッダー」という、ログに残りにくい部分に載せる、というセキュリティ上の配慮です。

---

## 7. 「わからない」マークと「学習済み」記録（3520〜3555行）

`getUnsureSet`/`saveUnsureSet`は、あるデッキの「わからない」マークが付いたカードの集合（`Set`）を読み書きする関数です。5節で説明した`studyDataDeckKey`を通して、端末に依存しない共通のキーで保存されます。

```js
function markCardSeen(deckId, card) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck || !deck.filename) return;
  const key = deck.filename;
  const already = studyDataCache.seen[key] || [];
  const cKey = cardKey(card);
  if (already.includes(cKey)) return;
  const arr = [...already, cKey];
  studyDataCache.seen[key] = arr;
  saveStudyDataCache();
  pushStudyDataToServer('save_seen', { deck_id: key, seen: arr });
}
```
- 「わからない」マークとは違い、一度学習した（画面に表示した）という記録は取り消されません（学習した事実そのものは変わらないため）。これは次節の「みんなのわかる率」を計算するための材料です。非公開デッキは「他の人」がいないので集計する意味が無く、対象外になっています。

---

## 8. みんなの「わかる率」バッジ（3577〜3606行）

```js
async function loadUnderstandingBadge() {
  const badge = document.getElementById('study-understand-badge');
  if (!badge) return;
  badge.style.display = 'none';
  const session = getLoginSession();
  if (!session || !session.session_token) return;
  const targetDecks = studyIsFolder
    ? folderPlayDecks.filter(d => d.filename)
    : decks.filter(d => d.id === studyDeckId && d.filename);
  const filenames = [...new Set(targetDecks.map(d => d.filename))];
  if (!filenames.length) return;
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 5000);
    // ★ session_tokenはURLクエリに載せない（get_study_dataと同じ理由）。Authorizationヘッダで送る。
    const qs = new URLSearchParams({ guild_id: GUILD_ID, filenames: filenames.join(',') });
    const res = await fetch(`${API_BASE}deck_understanding?${qs.toString()}`, {
      signal: controller.signal, cache: 'no-store',
      headers: { 'Authorization': 'Bearer ' + session.session_token },
    });
    clearTimeout(timer);
    const data = await res.json();
    // ★ まだ誰も（自分も含め）1枚も学習していなければ、0%という誤解を招く表示はしない
    if (!data.ok || !data.studied) return;
    const pct = Math.round((data.understood / data.studied) * 100);
    badge.innerHTML = `${Icons.cmHtml('globe', {size:13})} わかる率 ${pct}%`;
    badge.title = `学習済みカードのうち「わからない」が付いていない割合（みんなの合計 ${data.understood}/${data.studied}）`;
    badge.style.display = '';
  } catch (e) {} // 通信失敗時は出さないだけ（学習自体は止めない）
}
```
- 学習画面右上に表示される「わかる率」バッジです。「その公開デッキを学習した全員分の、学習済みカードのうち今『わからない』マークが付いていない割合」をサーバーに計算してもらい、パーセント表示します。フォルダをまとめて学習している場合は、フォルダ内の公開デッキ全部を対象にまとめて集計します。
- `if (!data.ok || !data.studied) return;`：まだ誰も（自分も含めて）1枚も学習していない場合、`0%`という誤解を招く表示を避けるため、バッジ自体を出しません。

---

## 9. 学習の続き・完了記録（3592〜3665行）

```js
function saveStudyProgress() {
  const id = studyIsFolder ? studyFolderId : studyDeckId;
  if (!id || !studyCards.length) return;
  const data = {
    order: studyCards.map(c => cardKey(c)),
    idx: studyIdx,
    mode: studyMode,
    reverse: studyReverse,
    autoGrade: studyAutoGrade,
    shuffled: studyShuffled,
    updatedAt: Date.now(),
  };
  const key = studyItemKey(studyIsFolder, id);
  studyDataCache.progress[key] = data;
  saveStudyDataCache();
  pushStudyDataToServer('save_study_progress', { key, data });
}
```
- 保存するのは「そのときのカードの並び順（`cardKey`の配列）」「今何問目か」「'all'/'unsure'どちらのモードだったか」「反転モードだったか」「自動採点モードだったか」「シャッフル済みだったか」で、**カードの内容そのものは保存しません**。コメントによれば、これは「常に最新の`decks`から引き直すため、続きから再開したときに、その間に行われた編集や画像追加の内容とズレないようにするため」の設計です。
- `loadStudyProgress`はこの逆で、保存されているデータが壊れていないか（配列が空でないか、`idx`が範囲内か）を確認してから返します。
- `saveCompletionRecord`/`loadCompletionRecord`は、学習を最後まで終えたという記録（完了日時と問題数だけ）を、進捗とは別のキーで保存します。

```js
function renderStudyTitle() {
  const prefixIcon = studyIsFolder ? Icons.cmHtml('folder', {size:16}) + ' ' : '';
  const suffixIcons =
    (studyReverse  ? ' ' + Icons.html('refresh', {size:14})  : '') +
    (studyShuffled ? ' ' + Icons.html('shuffle', {size:14}) : '');
  document.getElementById('study-title').innerHTML = prefixIcon + esc(studyBaseTitle) + suffixIcons;
}
```
`renderStudyTitle()`（3673〜3679行）は学習画面のタイトルを安全に組み立てる関数です。コメントに「以前は絵文字を末尾に付け足す正規表現による文字列操作をしていたのをやめ、常にここから再描画する方式にした」とあります。デッキ名・フォルダ名（ユーザー入力）は`esc()`を通し、アイコン部分（フォルダアイコン、反転/シャッフル中のアイコン）はこのファイル内の固定HTMLとして別に組み立てることで、安全性と表示の一貫性を両立させています。

---

## 10. フォルダ単位でまとめて学習を始める：`openFolderPlayMode(folderId)`（3684〜）

```js
async function openFolderPlayMode(folderId) {
  const folder = folders.find(f => f.id === folderId);
  const targetDecks = collectDecksInFolder(folderId)
    .filter(d => (d.filename ? (d.count ?? d.cards.length) : d.cards.length) > 0);
  if (!targetDecks.length) return;

  loadingFolderIds.add(folderId);
  renderDeckListUI();

  // ★ プレイ開始時は毎回サーバーの最新カードを取りに行く（force=true）。
  //   キャッシュ済みでも取り直すことで、他の人が直した修正がすぐプレイ画面に反映される。
  // ★ 修正：直前の編集でのサーバー同期がまだ終わっていない状態で強制リロードすると、
  //   同期前の古い内容（最悪カード0枚）で上書きされてしまうため、各デッキの保留中の
  //   同期を先に待ってから読み込み直す。
  await Promise.all(targetDecks.map(d => waitForPendingSync(d.id)));
  // ★ 修正：1回失敗しただけで行き止まりのアラートを出して終わらせず、
  //   失敗したデッキだけを対象に「もう一度試す」を選べるようにする
  //   （タイムアウトを含む一時的な通信エラーでフォルダのプレイを諦めなくて済むように）。
  let pending = targetDecks;
  while (pending.length) {
    const results = await Promise.all(pending.map(d => ensureDeckCardsLoaded(d.id, true)));
    pending = pending.filter((d, i) => !results[i].ok);
    if (!pending.length) break;
    const retry = await showCmConfirm({
      title: '読み込みに失敗しました',
      desc: `${pending.length}件のデッキが読み込めませんでした。通信環境を確認してもう一度お試しください。`,
      okLabel: 'もう一度試す', cancelLabel: 'やめる',
    });
    if (!retry) { loadingFolderIds.delete(folderId); renderDeckListUI(); return; }
  }
```
- フォルダ内の全デッキ（カードが1枚もあるものだけ）を対象に、それぞれの保留中の同期を待ってから、強制的に最新のカードを取り直します。
- 一部のデッキだけ読み込みに失敗した場合、**失敗したデッキだけを対象に**再試行できるようにしています（`pending`を「まだ失敗しているデッキだけ」に絞り込んでループを繰り返す）。コメントには「1回失敗しただけで行き止まりのアラートを出して終わらせず、タイムアウトを含む一時的な通信エラーでフォルダのプレイを諦めなくて済むように」という意図が書かれています。

このあとは、実際に学習開始モーダル（`modal-play-mode`）の中身（対象問題数、「わからない」件数、「続きから」の有無など）を組み立てて表示します。「みんなでクイズを始める」の項目は、単一デッキが前提の機能なのでフォルダプレイでは非表示にする、という注記もあります。

---

続きは[07_Cardmaker.js_その7_学習モードとクイズ再生.md](07_Cardmaker.js_その7_学習モードとクイズ再生.md)で、実際のフラッシュカード学習画面・一人用クイズの入口を解説します。
