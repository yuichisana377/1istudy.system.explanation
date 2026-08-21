# Cardmaker.js その6：カード編集モーダルと学習データの同期（2982〜3668行）

[[05_Cardmaker.js_その5_ホーム画面のドラッグ並び替え.md]]の続きです。

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

### 2.4 モーダルの中身を組み立てる：`openCardEditModalCommon(deckId, c, context)`（3075〜3122行）
```js
editingIsQuizChoice = Array.isArray(c.choices) && c.choices.length >= CHOICE_MIN;
document.getElementById('modal-edit-q').value = mathToPlainText(c.question);
...
if (editingIsQuizChoice) {
  const correctIndices = Array.isArray(c.correct_indices) ? c.correct_indices
    : (typeof c.correct_index === 'number' ? [c.correct_index] : []);
  renderChoiceEditorRows('modal-edit-choice', c.choices, correctIndices);
  editImgBuf = { q: [...(c.imgs_q || [])], a: [], e: [] };
  ...
} else {
  ...
  editImgBuf = { q: [...(c.imgs_q || [])], a: [...(c.imgs_a || [])], e: [...(c.imgs_e || [])] };
  ...
}
```
- カードが`choices`（選択肢）を持っていれば多肢選択カードとして扱い、通常の解答/解説欄の代わりに選択肢入力欄を表示します。
- `correct_index`（単数形、古いデータ形式）と`correct_indices`（複数形、現在の形式）の両方に対応しているのは、過去のデータ形式を今も引き続き正しく開けるようにするための後方互換コードです。
- 画像は「モーダル専用のバッファ（`editImgBuf`）」にコピーしてから編集します。元の配列を直接いじらず、保存ボタンを押したときにまとめて書き戻す設計です（キャンセルした場合に元のデータを傷つけない）。

### 2.5 保存：`saveCardEdit()`（3124〜3198行）
```js
if (editingIsQuizChoice) { return saveQuizChoiceCardEdit(q, errBar); }
```
- 選択式カードの場合は別の関数（2.6節）に処理を委ねます。

通常カードの場合：
```js
const idx = deck.cards.findIndex(c => cardKey(c) === editingCardKey);
...
const card = deck.cards[idx];
card.question = q; card.answer = a; card.explanation = e;
card.imgs_q = [...editImgBuf.q]; card.imgs_a = [...editImgBuf.a]; card.imgs_e = [...editImgBuf.e];

const studySameIdx = studyCards.findIndex(sc => cardKey(sc) === editingCardKey && (sc.__deckId || studyDeckId) === deck.id);
if (studySameIdx !== -1) {
  const sc = studyCards[studySameIdx];
  sc.question = q; sc.answer = a; sc.explanation = e;
  ...
}
```
- `editingCardKey`（開いたときに記録しておいた`cardKey`）を使って、対象カードを配列の中から探し直します（インデックス番号を直接覚えておくのではなく、内容から作られる安定キーで探すことで、途中で並び替えが起きていても正しいカードを見つけられます）。
- **学習中の配列（`studyCards`）にも同じ変更を書き戻しています**。コメントによれば、以前は`deck.cards`と同じオブジェクトを参照していたため自動的に反映されていましたが、カード編集の前に`deck.cards`を丸ごと読み込み直すよう変更した結果、もはや「同じオブジェクト」とは限らなくなったため、明示的にコピーする必要が生じた、という経緯が説明されています。
- 保存後、どこから開かれたか（`editingContext`：`'study'`/`'listview'`/それ以外）に応じて、更新すべき画面を出し分けます。

### 2.6 選択式カードの保存：`saveQuizChoiceCardEdit(q, errBar)`（3201〜3250行）
考え方は同じですが、選択肢の入力チェック（すべて埋まっているか、正解が1つ以上選ばれているか）を行い、`answer`欄も正解の選択肢文言から自動的に作り直します。`delete card.correct_index`で、もし古い形式のフィールドが残っていれば片付ける、という掃除も行っています。

### 2.7 学習画面の表示だけを更新：`refreshStudyCardDisplay(c)`（3253〜3271行）
```js
const qText = studyReverse ? c.answer   : c.question;
const qImgs = studyReverse ? c.imgs_a   : c.imgs_q;
const aText = studyReverse ? c.question : c.answer;
const aImgs = studyReverse ? c.imgs_q   : c.imgs_a;
```
- 「問題と解答を逆にする」反転モードが有効なら、表示する内容そのものを入れ替えます（データ自体は変更しません）。カードをめくった状態（表か裏か）は維持したまま、内容だけを最新化する関数です。

---

## 3. デッキ名の変更（3273〜3359行）

```js
async function openRename(id) {
  ...
  const currentName = currentSubject && deck.name.startsWith(currentSubject + ' ')
    ? deck.name.slice(currentSubject.length + 1) : deck.name;
  document.getElementById('modal-rename-input').value = currentName;
  const planRow = document.getElementById('modal-rename-plan-publish-row');
  if (!deck.filename) {
    planRow.style.display = '';
    document.getElementById('modal-rename-plan-publish').checked = deck.planPublish !== false;
  } else {
    planRow.style.display = 'none';
  }
  ...
}
```
- デッキ名から科目名の接頭辞を取り除いて入力欄に表示します（一覧の表示ロジックと同じ考え方）。
- 「公開予定」トグルは、まだサーバー未登録（下書き）のデッキのときだけ表示します。すでに公開済みのデッキを非公開に戻したい場合は、別の「非公開に戻す」メニューを使う設計です。

```js
async function saveRename() {
  ...
  if (!deck.filename && planPublishChanged && deck.planPublish) {
    await announceNewDeckToServer(deck.id);
    renderDeckListUI();
  }
  if (deck.filename) {
    await waitForPendingSync(deck.id);
    const loaded = await loadDeckCardsWithRecovery(deck.id);
    if (!loaded) { showBanner('名前の変更はローカルには反映されています（サーバーへの反映は未実施）', ...); return; }
    const ok = await queueSyncDeckToServer(deck);
    if (!ok) showBanner('サーバーへの名前変更の反映に失敗しました', ...);
  }
}
```
- 「公開予定」が新たにONにされた場合、この時点でサーバーに先行登録します（[[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md]]の`announceNewDeckToServer`）。
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

`syncDeckToServer(deck)`自体（3385〜3416行）は、実際にデッキの内容をサーバーの`save_cards`に送信する処理です。`silent: true`（通知なし）で送られるのがポイントで、カードを1枚編集するたびにDiscordへ通知が飛ぶと煩わしいため、通知が飛ぶのは「公開して保存」（[[04_Cardmaker.js_その4_カード保存と公開.md]]の`publishDeck`）のときだけに限定されています。

---

## 5. 学習データの端末間共有（3418〜3466行）

```js
const STUDY_DATA_CACHE_KEY = 'cardmaker_study_data_cache_v1';
function loadStudyDataCache() { ... }
function saveStudyDataCache() { ... }
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

## 6. サーバーとのやり取り（3469〜3518行）

`fetchAndMergeStudyData()`と`pushStudyDataToServer(path, body)`は、これまで見てきたパターン（`AbortController`でタイムアウト、`cache: 'no-store'`）と同じ考え方の取得・送信用関数です。1点注目すべきなのは：

```js
const qs = new URLSearchParams({ guild_id: GUILD_ID });
const res = await fetch(`${API_BASE}get_study_data?${qs.toString()}`, {
  ...,
  headers: { 'Authorization': 'Bearer ' + session.session_token },
});
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

## 8. みんなの「わかる率」バッジ（3557〜3590行）

```js
async function loadUnderstandingBadge() {
  ...
  const targetDecks = studyIsFolder
    ? folderPlayDecks.filter(d => d.filename)
    : decks.filter(d => d.id === studyDeckId && d.filename);
  const filenames = [...new Set(targetDecks.map(d => d.filename))];
  if (!filenames.length) return;
  ...
  const data = await res.json();
  if (!data.ok || !data.studied) return;
  const pct = Math.round((data.understood / data.studied) * 100);
  badge.innerHTML = `... わかる率 ${pct}%`;
  ...
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

`renderStudyTitle()`（3653〜3663行）は学習画面のタイトルを安全に組み立てる関数です。コメントに「以前は絵文字を末尾に付け足す正規表現による文字列操作をしていたのをやめ、常にここから再描画する方式にした」とあります。デッキ名・フォルダ名（ユーザー入力）は`esc()`を通し、アイコン部分はこのファイル内の固定HTMLとして別に組み立てることで、安全性と表示の一貫性を両立させています。

---

## 10. フォルダ単位でまとめて学習を始める：`openFolderPlayMode(folderId)`（3667〜3742行）

```js
const targetDecks = collectDecksInFolder(folderId).filter(d => (d.filename ? (d.count ?? d.cards.length) : d.cards.length) > 0);
if (!targetDecks.length) return;
loadingFolderIds.add(folderId);
renderDeckListUI();
await Promise.all(targetDecks.map(d => waitForPendingSync(d.id)));
let pending = targetDecks;
while (pending.length) {
  const results = await Promise.all(pending.map(d => ensureDeckCardsLoaded(d.id, true)));
  pending = pending.filter((d, i) => !results[i].ok);
  if (!pending.length) break;
  const retry = await showCmConfirm({ title: '読み込みに失敗しました', desc: `${pending.length}件のデッキが読み込めませんでした。...`, ... });
  if (!retry) { loadingFolderIds.delete(folderId); renderDeckListUI(); return; }
}
```
- フォルダ内の全デッキ（カードが1枚もあるものだけ）を対象に、それぞれの保留中の同期を待ってから、強制的に最新のカードを取り直します。
- 一部のデッキだけ読み込みに失敗した場合、**失敗したデッキだけを対象に**再試行できるようにしています（`pending`を「まだ失敗しているデッキだけ」に絞り込んでループを繰り返す）。コメントには「1回失敗しただけで行き止まりのアラートを出して終わらせず、タイムアウトを含む一時的な通信エラーでフォルダのプレイを諦めなくて済むように」という意図が書かれています。

このあとは、実際に学習開始モーダル（`modal-play-mode`）の中身（対象問題数、「わからない」件数、「続きから」の有無など）を組み立てて表示します。「みんなでクイズを始める」の項目は、単一デッキが前提の機能なのでフォルダプレイでは非表示にする、という注記もあります。

---

続きは[[07_Cardmaker.js_その7_学習モードとクイズ再生.md]]で、実際のフラッシュカード学習画面・一人用クイズの入口を解説します。
