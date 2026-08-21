# Cardmaker.js その3：デッキの読み込み・デッキメニュー・新規作成（1528〜2129行）

[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)の続きです。

---

## 1. サーバーからデッキ一覧を取得する：`fetchAndMergeDecks()`（1528〜1597行）

一覧画面に表示する「軽量なメタ情報だけ」（カード本体は含まない：名前・件数・科目・公開者など）をサーバーから取得し、この端末の`decks`配列に反映します。カード本体（問題文・解答・画像）は、実際にそのデッキを開くときに別途取得します（次節）。これにより、デッキ数や画像が増えても一覧の表示自体は軽いままになります。

`fetched`を組み立てる`.map(s => {...})`の中で、サーバーから来た情報と、この端末に元々あった情報（`existing`）をどう組み合わせるかが細かく調整されています：

```js
const cachedIsStale = existing && existing.cardsLoaded
  && typeof s.count === 'number' && s.count < existing.cards.length;
const keepLoadedCards = existing && existing.cardsLoaded && !cachedIsStale;
```
- すでにカード本体を読み込み済みなら、それを引き継いで再取得を省略したいところですが、**サーバー側の最新枚数がこの端末のキャッシュより減っている場合は引き継がない**という条件が付いています。コメントによれば、これをやらないと「他の人がカードを減らした後、この端末が古い（多い）枚数をキャッシュとして持ち続けてしまい、後で説明する枚数不一致チェックが永遠に満たされず、そのデッキが二度と開けなくなる」という「詰み」状態を招くためです。

`notYetPublished`（「作成中」かどうか）と`folderId`（フォルダ所属）についても、それぞれ「この端末に記録があればそれを優先し、無ければサーバー側の情報や、旧来の判定基準にフォールバックする」という、後方互換性を意識した組み立てになっています。

- **`quizArchive: !!s.quiz_archive`（2026/08/21追加）**… そのデッキがクイズ過去問由来かどうかのフラグを、サーバー側の索引（`s.quiz_archive`、[../../1I勉強会bot解説/14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)参照）からそのまま引き継ぎます。以前はこのフラグ自体が存在せず、`isDeckInFolderScope`（そのデッキが今どのフォルダにあるか）で都度判定していましたが、デッキをクイズ過去問フォルダの外へ移動できるようにしたのに伴い、位置に依存しないこの専用フィールドへ切り替わりました。4.1節の「カードを編集する」メニューの表示/非表示や、`renderDeckListUI()`のバッジ表示に使われます。

最後に：
```js
const localOnly = decks.filter(d => !d.filename && !publishedNames.has(d.name)).map(d => ({ ...d, cardsLoaded: true }));
decks = [...localOnly, ...fetched];
saveDecks(decks);
```
- まだ一度もサーバーに登録していない、この端末だけの下書きデッキ（`localOnly`）はそのまま残し、サーバーから取得した公開済みデッキ一覧と合体させて保存します。

---

## 2. デッキを開くときのカード本体取得（1599〜1762行）

### 2.1 タイムアウト時間を枚数に応じて決める（1622〜1630行）
```js
const DECK_LOAD_BASE_TIMEOUT_MS = 20000; // 基本値：20秒
const DECK_LOAD_PER_CARD_MS     = 150;   // カード1枚につき+150ms
const DECK_LOAD_MAX_TIMEOUT_MS  = 90000; // 上限：90秒
function deckLoadTimeoutMs(expectedCount) {
  if (!expectedCount || expectedCount <= 0) return DECK_LOAD_BASE_TIMEOUT_MS;
  const extended = DECK_LOAD_BASE_TIMEOUT_MS + expectedCount * DECK_LOAD_PER_CARD_MS;
  return Math.min(DECK_LOAD_MAX_TIMEOUT_MS, extended);
}
```
- 画像を多く含む大きなデッキだと通信に時間がかかるため、一律のタイムアウトだと「実際には成功していたのに時間切れで失敗扱いになる」ことがあったそうです。カードの枚数が多いほどタイムアウトを延ばし（1枚あたり150ミリ秒）、ただし青天井にはせず上限90秒で頭打ちにする、というバランスを取った調整です。

### 2.2 実際の取得：`fetchCardSetOnce`（1632〜1645行）
1回分の通信を行うだけのシンプルな関数です。`AbortController`でタイムアウトを設定し、失敗したら`throw`（例外を投げる）で呼び出し元に伝えます。

### 2.3 `ensureDeckCardsLoaded(deckId, force)`（1647〜1709行）
デッキを開く前に呼ばれる、カード本体を確実に用意するための中心的な関数です。

```js
const knownCount = deck.cardsLoaded ? deck.cards.length : 0;
const metaCount = typeof deck.count === 'number' ? deck.count : 0;
const expectedCount = Math.max(knownCount, metaCount) || null;
```
- 「一覧のメタ情報で分かっている枚数」と「この端末に既にキャッシュされている枚数」の**大きい方**を「期待される枚数」とします。

```js
if (expectedCount !== null && expectedCount > 0 && fetchedCards.length < expectedCount) {
  console.warn(`[cardmaker] get_card_set が${fetchedCards.length}件しか返しませんでしたが、${expectedCount}件のはずです。 filename=${deck.filename}`);
  return { ok: false, reason: 'mismatch', expectedCount, fetchedCount: fetchedCards.length };
}
```
- サーバーが「成功」と答えても、実際に返ってきたカードの枚数が期待より少なければ、**通信としては成功していても内容は信用しない**という安全策です。コメントには「これが無いと、編集画面が空／一部欠けた状態で開いてしまい、そのまま公開すると本物のカードが少ないデータで上書きされて消えてしまう事故につながる」とあります。
- 一時的な通信の遅延・瞬断に強くするため、1回目が失敗したら0.5秒待って**静かにもう一度だけ**自動で再試行してから、最終的な失敗として扱います（1671〜1681行）。

### 2.4 失敗したときの回復手段：`loadDeckCardsWithRecovery(deckId)`（1720〜1762行）
`while(true)`（条件が常に真＝明示的に`return`するまで繰り返すループ）を使い、失敗の種類ごとにユーザーに選択肢を提示します：
- **`mismatch`（枚数不一致）**：まず最新のメタ情報を取り直し、それでも期待件数が0でなければ、`showCmChoiceDialog`で「もう一度試す」「空のまま開く（上級者向け）」の2択を提示。「空のまま開く」を選ぶと、あえて0件のまま`cardsLoaded=true`にして先に進めます（保存すると中身が消える可能性がある、という警告付き）。
- **それ以外（ネットワークエラー等）**：「もう一度試す」か「やめる」かのシンプルな確認。
- どちらのケースでも「行き止まりで諦めるしかない」状態にはならないよう設計されています。

---

## 3. `renderDeckList()`とバックグラウンドでの「わからない」バッジ先読み（1764〜1790行）

```js
async function renderDeckList() {
  decks = loadDecks();
  folders = loadFoldersCache();
  renderDeckListUI();
  try {
    await Promise.all([fetchAndMergeDecks(), fetchAndMergeFolders(), fetchAndMergeOrder(), fetchAndMergeStudyData()]);
    renderDeckListUI();
    preloadUnsureBadges();
  } catch(e) {}
}
```
- まずキャッシュ（`localStorage`）の内容で即座に描画してから、`Promise.all`（複数の非同期処理を同時に走らせ、全部終わるのを待つ）で4種類のサーバー問い合わせを並行して行い、終わったら最新の内容で再描画します。「まずキャッシュ、次に本物」という2段階の考え方は他の箇所（`showScreen`など）とも共通しています。

```js
function preloadUnsureBadges() {
  const targets = decks.filter(d => d.filename && d.cardsLoaded === false && (studyDataCache.unsure[d.filename] || []).length > 0);
  targets.forEach(d => ensureDeckCardsLoaded(d.id));
}
```
- 「わからない」バッジは、そのデッキのカード本体を読み込んでいないと件数を計算できません。コメントによれば、Safariなど普段使っている端末ではキャッシュが残っているので気づきにくいのですが、Discordアプリ内蔵ブラウザのようにストレージが毎回まっさらな環境だと、一覧を開いた直後は「わからない」バッジが1件も出ない、という問題がありました。この関数は、「わからない」の記録があるのにカード本体は未読み込みのデッキだけを裏でこっそり読み込み直し（`await`せずに呼びっぱなし）、あとからバッジを反映させます。

---

## 4. デッキメニュー（1793〜1960行）

### 4.1 メニューを開く（1793〜1828行）
`openDeckMenu(id)`はメニューモーダルを開くだけのシンプルな関数（非公開デッキなら「非公開に戻す」の項目を隠す）。

```js
document.getElementById('menu-edit-item').style.display = deck.quizArchive ? 'none' : '';
document.getElementById('menu-quiz-archive-note').style.display = deck.quizArchive ? '' : 'none';
```
- **★ 追加（2026/08/21）**：デッキが`quizArchive`（クイズ過去問由来）であれば、「カードを編集する」メニュー項目（`menu-edit-item`）を隠し、代わりに「クイズ過去問デッキは問題を編集できません」という注意書き（`menu-quiz-archive-note`）を表示します。「デッキ名を変更する」「フォルダに移動する」「非公開に戻す」「デッキを削除する」の各項目は隠しません（引き続き操作できます）。サーバー側（`bot.py`の`save_cards`）でも同じ制限を独立に強制していますが、そもそも選べなくすることで生徒を迷わせない、という考え方です。

`startQuizFromDeck(deckId)`は、そのデッキを元に「みんなでクイズ」のホスト作成画面（`Quiz.html`）へ移動する関数です。公開済みデッキでなければ「先に公開してください」と案内します。

### 4.2 非公開に戻す：`menuUnpublish()`（1830〜1866行）
サーバーの`/delete_cards`にリクエストを送り、成功したらこの端末側のデッキ情報から`filename`を消して「非公開」の状態に戻します。

```js
if (data.error === 'creator_approval_required') {
  openRequestDeleteModal('deck', deck.filename, deck.name, data.owner_nickname);
  return;
}
```
- 作成者本人以外がこの操作をしようとすると、サーバー側がこのエラーを返し、その場合は削除するのではなく「作成者への削除依頼フォーム」（4.3節）を開きます。

### 4.3 削除依頼フォーム（1868〜1919行）
`openRequestDeleteModal`／`submitRequestDelete`は、作成者本人でない人が削除・非公開化しようとしたときに、理由を書いて`/request_delete`に送信する処理です。成功すると、作成者にDiscordのDMで確認が届く（Discord連携が無い場合は「次回サイトを開いたときに確認される」という案内に変わる）旨のメッセージを表示します。

### 4.4 完全に削除する：`menuDelete()`（1921〜1960行）
```js
async function menuDelete() {
  closeModal('modal-deck-menu');
  const okDelete = await showCmConfirm({
    title: 'このデッキを削除しますか？', desc: 'この操作は取り消せません。',
    okLabel: '削除する', okStyle: 'danger',
  });
  if (!okDelete) return;
  const deck = decks.find(d => d.id === menuTargetId);
  if (deck && deck.filename) {
    try {
      const res = await fetch(`${API_BASE}delete_cards`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ guild_id: GUILD_ID, session_token: getLoginSession()?.session_token, filename: deck.filename, nickname: getLoginSession()?.nickname }),
      });
      const data = await res.json();
      if (!data.ok) {
        if (data.error === 'creator_approval_required') {
          openRequestDeleteModal('deck', deck.filename, deck.name, data.owner_nickname);
          return;
        }
        throw new Error(data.error || '削除失敗');
      }
    } catch(e) {
      const localOnly = await showCmConfirm({
        title: 'GitHubからの削除に失敗しました',
        desc: 'ローカルからだけ削除しますか？',
        okLabel: 'ローカルから削除', okStyle: 'danger',
      });
      if (!localOnly) return;
    }
  }
  decks = decks.filter(d => d.id !== menuTargetId);
  saveDecks(decks); renderDeckList();
}
```
- コメントに「以前はこのレスポンスの中身（`data.ok`）を見ておらず、サーバー側が削除に失敗しても気づかずローカルの一覧からだけ消えてしまっていた（サーバーには残ったまま＝他の端末には残り続ける不整合）」というバグの記録があります。今は必ず`data.ok`を確認するよう修正されています。
- 通信自体が失敗した場合（`catch`）は「ローカルからだけ削除しますか？」という確認を出し、ユーザーの意思でローカルだけの削除に切り替えられるようにしています。

---

## 5. 新しいデッキの作成（1963〜2079行）

### 5.1 新規作成画面を開く（1963〜1988行）
`openNewSet()`は入力欄をリセットし（公開予定トグルは毎回デフォルトON、多肢選択トグルは毎回OFF）、`screen-new`を表示して科目一覧を読み込みます。

### 5.2 `startEdit()`（1989〜2017行）— 「作成 →」ボタン
```js
const name = subject ? `${subject} ${input}` : input;
const deck = { id: genId(), name, subject, cards: [], cardsLoaded: true, folderId: currentFolderId, planPublish, notYetPublished: true, choiceMode };
decks.push(deck); saveDecks(decks);
if (planPublish) { await announceNewDeckToServer(deck.id); }
```
- 科目が選ばれていれば、デッキ名の先頭に自動で付け足します（例：「数学」＋「二次関数」→「数学 二次関数」）。
- `notYetPublished: true`：まだ一度も「公開して保存」を経ていない状態を表す印。これが立っている間は、カードが何枚あっても常に「作成中」バッジとして扱われます（[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)参照）。
- 「公開予定」トグルがONなら、この時点（まだカードを1枚も入力していない段階）で`announceNewDeckToServer`を呼び、サーバーにも空のデッキとして登録します。これにより、作成した瞬間から他の部員の一覧にも「🟠 作成中」として表示されるようになります。

### 5.3 空デッキをサーバーに先行登録：`announceNewDeckToServer(deckId)`（2025〜2079行）
```js
const cards = deck.cards.map(cardToServerPayload);
const res = await fetch(`${API_BASE}save_cards`, {
  method: 'POST', ...,
  body: JSON.stringify({ name: deck.name, cards, ..., silent: true, incomplete: true, choice_mode: deck.choiceMode || null }),
});
```
- 既存の`save_cards`というAPI（カード保存用）を、あえて「中身が空（または少数）の状態」で呼び出すことで、デッキそのものをサーバーに登録します。
- `silent: true`：この時点ではDiscordへの通知はしません（作成しただけなので、まだ知らせるべき内容が無いため）。
- `incomplete: true`：「未完成（作成中）」として登録します。
- コメントには重要な過去のバグ修正が記録されています：「以前は常に`cards: []`（空配列固定）を送ってしまっていたため、デッキ名編集モーダルで後から『公開予定』をONにしたようなケースで、既にローカルにあったカードが無視されてサーバー側は0枚として登録されてしまっていた。その結果、次に編集画面を開いた際の強制的な最新化で、ローカルのカードがサーバー側の0枚で上書きされて消えてしまう」という重大な不具合があったため、今は必ず「今ローカルにある実際のカード」を送るように修正されています。
- 送信中に他の同期処理が割り込んでいる可能性があるため、更新前に`decks = loadDecks()`で**もう一度最新の状態を読み直してから**目的のデッキを探して更新する、という慎重な作りになっています。
- サーバーへの登録が失敗しても（`catch`で握りつぶし）、この端末だけの下書きとして続行できるので、機能自体は壊れません。

---

## 6. カード編集画面を開く（2084〜2129行）

```js
async function openEditDeck(deckId) {
  currentDeckId = deckId;
  const deck = decks.find(d => d.id === deckId);
  if (!deck) return;

  await waitForPendingSync(deckId);
  const ok = await loadDeckCardsWithRecovery(deckId);
  if (!ok) return; // ユーザーが「やめる」を選んだ場合は編集画面を開かない

  document.getElementById('edit-deck-title').textContent = deck.name;
  document.getElementById('btn-save-local').style.display = '';
  document.getElementById('btn-done').textContent = deck.filename ? '公開して保存' : '保存して公開';
  applyCardFormChoiceMode(deck.choiceMode);
  clearEditor(); renderCreatedList(); showScreen('edit');
  setTimeout(() => document.getElementById('ta-q').focus(), 200);
}
```
- 編集は「サーバー全体を丸ごと上書き保存する」操作につながるため、たとえキャッシュ済みでも**必ず**最新のカードを取り直します（`loadDeckCardsWithRecovery`、2節参照）。
- その前に`waitForPendingSync(deckId)`（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)で説明）で、直前の操作によるサーバーへの送信がまだ終わっていなければそれを待ちます。これをしないと、送信中の変更が完了する前に強制リロードが割り込み、データが消えてしまう危険があります。
- コメントには「以前は公開済み・作成中のデッキだと『保存』ボタン自体を隠し、常に完全な『公開して保存』（ログイン確認・完成/未完成の選択・Discord通知）を通らないと編集を中断できなかった」という改善の経緯も書かれています。単に作業を保存していったん戻りたいだけのときにも毎回この確認が挟まれるのは不便だったため、「保存」ボタンは常に表示されるよう改善されました。

```js
function applyCardFormChoiceMode(choiceMode) {
  const isChoice = !!choiceMode;
  document.getElementById('qa-csv-block').style.display         = isChoice ? 'none' : '';
  document.getElementById('qa-choice-csv-block').style.display  = isChoice ? '' : 'none';
  document.getElementById('qa-answer-block').style.display      = isChoice ? 'none' : '';
  document.getElementById('qa-explanation-block').style.display = isChoice ? 'none' : '';
  document.getElementById('qa-choices-block').style.display     = isChoice ? '' : 'none';
  if (isChoice) { renderChoiceEditorRows('ta-choice', ['', ''], []); }
}
```
- 多肢選択デッキかどうかで、入力フォームの見た目を丸ごと切り替えます。多肢選択デッキでは「解答/解説」欄の代わりに「選択肢」欄を表示し、CSV読み込みも通常デッキ用と選択式デッキ用で別のブロックに出し分けます。

```js
function clearEditor() {
  ['q','a','e'].forEach(k => {
    const el = document.getElementById('ta-'+k);
    el.value = '';
    autoResize(el);
    el.dispatchEvent(new Event('input', { bubbles: true })); // ★ 数式プレビューもクリアする
    imgBuf[k] = [];
    document.getElementById('imgs-'+k).innerHTML = '';
  });
  ...
}
```
- 問題文・解答・解説の入力欄と、添付画像のバッファをまとめて空にします。
- `el.dispatchEvent(new Event('input', {bubbles: true}))`は、コードから「今、入力欄に入力があったことにする」という**擬似的なイベントを発生させる**書き方です。これにより、本来はユーザーが実際にタイプしたときだけ動く「数式プレビューの更新」処理（[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)で説明）を、プログラム側からも同じように動かすことができます。

---

続きは[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)で、CSV読み込みの入口からカードの保存・デッキの公開までを解説します。
