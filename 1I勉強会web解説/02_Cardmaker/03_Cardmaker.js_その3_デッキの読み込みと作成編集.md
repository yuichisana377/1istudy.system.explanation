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
- **`description: s.description || (existing && existing.description) || null`（★ 追加）**… デッキの説明欄（任意）。`subject`（科目）と全く同じ組み立て方（サーバー値を優先し、無ければこの端末の記録にフォールバック）です。サーバー側の索引にも`description`が含まれる（[../../1I勉強会bot解説/14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/00_カードデータ層と索引管理.md)の`_meta_from_card_data`参照）ため、一覧を取得した時点で（デッキ本体を開かなくても）説明を表示できます。

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
```js
async function fetchCardSetOnce(filename, timeoutMs = DECK_LOAD_BASE_TIMEOUT_MS) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    const res = await fetch(`${API_BASE}get_card_set?filename=${encodeURIComponent(filename)}`, { signal: controller.signal, cache: 'no-store' });
    clearTimeout(timer);
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
    return data;
  } catch (e) {
    clearTimeout(timer);
    throw e;
  }
}
```
1回分の通信を行うだけのシンプルな関数です。`AbortController`でタイムアウトを設定し、失敗したら`throw`（例外を投げる）で呼び出し元に伝えます。

### 2.3 `ensureDeckCardsLoaded(deckId, force)`（1647〜1708行）
デッキを開く前に呼ばれる、カード本体を確実に用意するための中心的な関数です。

```js
async function ensureDeckCardsLoaded(deckId, force = false) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck) return { ok: false, reason: 'not_found' };
  if (!deck.filename) { deck.cardsLoaded = true; return { ok: true }; }
  if (deck.cardsLoaded && !force) return { ok: true };

  // ★ 直前まで一覧（list_cardsのメタ情報）で分かっていた問題数を控えておく。
  //   これと比べて、実際に取得できたカード数が不自然に少なければ
  //   「サーバーはok:trueを返したが、実は異常な状態だった」とみなして
  //   失敗扱いにする（＝空データでdeck.cardsを上書きしない）ための安全策。
  //   ★ 修正：deck.count（サーバー由来のメタ情報）だけでなく、この端末に
  //     既に読み込み済みのカード実数（deck.cards.length）も比較対象に含める。
  //     何らかの理由でサーバーへの同期がまだ済んでいない状態でも、
  //     「今ローカルにある枚数より減っている」場合は同じく異常とみなし、
  //     せっかく手元にあるカードを空／少ない件数で上書きしないようにする。
  const knownCount = deck.cardsLoaded ? deck.cards.length : 0;
  const metaCount = typeof deck.count === 'number' ? deck.count : 0;
  const expectedCount = Math.max(knownCount, metaCount) || null;
  // ★ カード枚数が多い（＝データ量が大きい）デッキほどタイムアウトを延ばす
  const timeoutMs = deckLoadTimeoutMs(expectedCount);

  loadingDeckIds.add(deckId);
  if (document.querySelector('.screen.active')?.id === 'screen-list') renderDeckListUI();

  try {
    // ★ 修正：まず1回試し、タイムアウトも含むネットワークエラーの場合だけ、
    //   間を置いて（500ms）もう一度だけ静かに自動再試行する。
    //   これにより、一時的な遅延・瞬断だけでユーザーに失敗を見せてしまうことを防ぐ。
    let data;
    try {
      data = await fetchCardSetOnce(deck.filename, timeoutMs);
    } catch (firstErr) {
      await new Promise(r => setTimeout(r, 500));
      data = await fetchCardSetOnce(deck.filename, timeoutMs);
    }
    const fetchedCards = data.cards || [];

    // ★ 安全策：サーバーが ok:true を返していても、直前まで分かっていた問題数
    //   （または、この端末に既に読み込み済みだった実際の枚数）より
    //   取得できたカード数が少ない場合は、通信は成功していても内容としては
    //   信用できないので「失敗」として扱う。
    //   これにより、編集画面が空／一部欠けた状態で開いてしまい、そのまま公開して
    //   サーバー側（または手元）の本物のカードを少ないデータで上書きしてしまう事故を防ぐ。
    if (expectedCount !== null && expectedCount > 0 && fetchedCards.length < expectedCount) {
      console.warn(`[cardmaker] get_card_set が${fetchedCards.length}件しか返しませんでしたが、${expectedCount}件のはずです。 filename=${deck.filename}`);
      return { ok: false, reason: 'mismatch', expectedCount, fetchedCount: fetchedCards.length };
    }

    deck.cards = fetchedCards;
    deck.cardsLoaded = true;
    deck.count = deck.cards.length;
    // ★ カード本体取得時にもサーバー側の未完成フラグを取り込んでおく（念のため）
    if ('incomplete' in data) deck.incomplete = !!data.incomplete;
    saveDecks(decks);
    return { ok: true };
  } catch(e) {
    return { ok: false, reason: 'network' };
  } finally {
    loadingDeckIds.delete(deckId);
    if (document.querySelector('.screen.active')?.id === 'screen-list') renderDeckListUI();
  }
}
```
- 「一覧のメタ情報で分かっている枚数」と「この端末に既にキャッシュされている枚数」の**大きい方**を「期待される枚数」とします。
- サーバーが「成功」と答えても、実際に返ってきたカードの枚数が期待より少なければ、**通信としては成功していても内容は信用しない**という安全策です。コメントには「これが無いと、編集画面が空／一部欠けた状態で開いてしまい、そのまま公開すると本物のカードが少ないデータで上書きされて消えてしまう事故につながる」とあります。
- 一時的な通信の遅延・瞬断に強くするため、1回目が失敗したら0.5秒待って**静かにもう一度だけ**自動で再試行してから、最終的な失敗として扱います。

### 2.4 失敗したときの回復手段：`loadDeckCardsWithRecovery(deckId)`（1720〜1762行）
```js
async function loadDeckCardsWithRecovery(deckId) {
  while (true) {
    const result = await ensureDeckCardsLoaded(deckId, true);
    if (result.ok) return true;

    if (result.reason === 'mismatch') {
      // ★ 判定前に最新のメタ情報を取り直す（ローカルの古いcountによる誤判定を防ぐ）
      try { await fetchAndMergeDecks(); } catch(e) {}
      const deck = decks.find(d => d.id === deckId);
      if (!deck) return false;

      // メタ情報を更新した結果、期待件数が0（＝本当に空が正解）になっていれば、
      // ここで改めて通常読み込みすれば矛盾なく成功するはず
      if (deck.count === 0) continue;

      const choice = await showCmChoiceDialog({
        title: '問題データの読み込みに不整合があります',
        desc: `一覧では${result.expectedCount}問のはずですが、サーバーから0問しか取得できませんでした。\nこのまま開いて保存すると、サーバー側のデータが消える可能性があります。`,
        choices: [
          { icon: Icons.html('refresh', {size:20}), label: 'もう一度試す', sub: 'まずはこちらをおすすめします', value: 'retry' },
          { icon: Icons.html('warning', {size:20}), label: '空のまま開く（上級者向け）', sub: '保存すると中身が消える可能性があります', value: 'force' },
        ],
        cancelLabel: 'やめる',
      });
      if (choice === 'retry') continue;
      if (choice === 'force') {
        const d = decks.find(x => x.id === deckId);
        if (d) { d.cards = []; d.cardsLoaded = true; saveDecks(decks); }
        return true;
      }
      return false; // やめる
    }

    // ネットワークエラー・その他の場合
    const retry = await showCmConfirm({
      title: '読み込みに失敗しました',
      desc: '通信環境を確認してもう一度お試しください。',
      okLabel: 'もう一度試す', cancelLabel: 'やめる',
    });
    if (!retry) return false;
    // ループして再試行
  }
}
```
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
```js
// ── デッキメニュー ─────────────────────
function openDeckMenu(id) {
  menuTargetId = id;
  const deck = decks.find(d => d.id === id);
  document.getElementById('menu-deck-name').textContent = deck.name;
  document.getElementById('menu-unpublish-item').style.display = deck.filename ? '' : 'none';
  // ★ 追加（2026/08/21）：クイズ過去問デッキは問題を編集できない
  //   （フォルダ移動・デッキ名の変更・非公開に戻す・削除は引き続き可能）。
  //   サーバー側（save_cards）でも強制しているが、そもそもメニューに
  //   出さないことで迷わせない。
  document.getElementById('menu-edit-item').style.display = deck.quizArchive ? 'none' : '';
  document.getElementById('menu-quiz-archive-note').style.display = deck.quizArchive ? '' : 'none';
  openModal('modal-deck-menu');
}
```
`openDeckMenu(id)`はメニューモーダルを開くだけのシンプルな関数（非公開デッキなら「非公開に戻す」の項目を隠す）。**★ 追加（2026/08/21）**：デッキが`quizArchive`（クイズ過去問由来）であれば、「カードを編集する」メニュー項目（`menu-edit-item`）を隠し、代わりに「クイズ過去問デッキは問題を編集できません」という注意書き（`menu-quiz-archive-note`）を表示します。「デッキ名を変更する」「フォルダに移動する」「非公開に戻す」「デッキを削除する」の各項目は隠しません（引き続き操作できます）。サーバー側（`bot.py`の`save_cards`）でも同じ制限を独立に強制していますが、そもそも選べなくすることで生徒を迷わせない、という考え方です。

```js
// ★ デッキ一覧から、そのデッキを元にした「みんなでクイズ」のホスト作成画面
//   （Quiz.html）へ遷移する。
function startQuizFromDeck(deckId) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck || !deck.filename) {
    showCmAlert({ title: 'クイズを始められません', desc: '公開済みのデッキだけ「みんなでクイズ」を始められます。先に公開してください。' });
    return;
  }
  const url = `Quiz.html?mode=host&deck=${encodeURIComponent(deck.filename)}&name=${encodeURIComponent(deck.name)}`;
  location.href = url;
}
```
`startQuizFromDeck(deckId)`は、そのデッキを元に「みんなでクイズ」のホスト作成画面（`Quiz.html`）へ移動する関数です。公開済みデッキでなければ「先に公開してください」と案内します。

### 4.2 非公開に戻す：`menuUnpublish()`（1830〜1866行）
```js
async function menuUnpublish() {
  closeModal('modal-deck-menu');
  const deck = decks.find(d => d.id === menuTargetId);
  if (!deck || !deck.filename) return;
  const ok = await showCmConfirm({
    title: '非公開に戻しますか？',
    desc: `「${deck.name}」をGitHubから削除して非公開に戻します。`,
    okLabel: '非公開に戻す', okStyle: 'danger',
  });
  if (!ok) return;
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 8000);
    const res = await fetch(`${API_BASE}delete_cards`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: getLoginSession()?.session_token, filename: deck.filename, nickname: getLoginSession()?.nickname }), signal: controller.signal,
    });
    clearTimeout(timer);
    const data = await res.json();
    if (!data.ok) {
      // ★ 追加：作成者本人以外は直接削除できない（サーバー側の作成者確認機能）。
      //   ローカルからは何も消さず、代わりに作成者への削除依頼フォームを開く。
      if (data.error === 'creator_approval_required') {
        openRequestDeleteModal('deck', deck.filename, deck.name, data.owner_nickname);
        return;
      }
      throw new Error(data.error || '削除失敗');
    }
    deck.filename = null; deck.count = undefined; deck.published_by = null; deck.incomplete = false;
    deck.planPublish = false; // ★ 追加：明示的に非公開へ戻した場合は「作成中」ではなく「非公開」表示にする
    deck.notYetPublished = true; // ★ 追加：再度公開する場合は改めて「公開して保存」を経る必要がある状態に戻す
    saveDecks(decks); renderDeckListUI();
    showBanner('非公開に戻しました', '#f1f5f9', '#334155', Icons.cmHtml('unpublish', {size:15}));
  } catch(e) {
    await showCmAlert({ title: 'GitHubからの削除に失敗しました', desc: e.message });
  }
}
```
サーバーの`/delete_cards`にリクエストを送り、成功したらこの端末側のデッキ情報から`filename`を消して「非公開」の状態に戻します。作成者本人以外がこの操作をしようとすると、サーバー側が`creator_approval_required`エラーを返し、その場合は削除するのではなく「作成者への削除依頼フォーム」（4.3節）を開きます。

### 4.3 削除依頼フォーム（1868〜1919行）
```js
// ── 削除の確認依頼（作成者本人以外が削除／非公開に戻そうとしたとき） ──
// サーバーが creator_approval_required を返したときに menuDelete()/
// menuUnpublish() から呼ばれる。ここでは何も削除せず、理由を添えて
// /request_delete を叩き、作成者にDiscordで確認してもらうだけ。
let requestDeleteCtx = null; // { category, filename, targetName }

function openRequestDeleteModal(category, filename, targetName, ownerNickname) {
  requestDeleteCtx = { category, filename, targetName };
  document.getElementById('request-delete-desc').textContent =
    `「${targetName}」の作成者（${ownerNickname || '作成者'}さん）に削除の確認が必要です。理由を書いて送信すると、作成者にDiscordで確認が届きます。`;
  document.getElementById('request-delete-reason').value = '';
  document.getElementById('request-delete-err').style.display = 'none';
  const btn = document.getElementById('request-delete-submit-btn');
  btn.disabled = false; btn.textContent = '送信する';
  openModal('modal-request-delete');
}

async function submitRequestDelete() {
  if (!requestDeleteCtx) return;
  const reason = document.getElementById('request-delete-reason').value.trim();
  const errEl = document.getElementById('request-delete-err');
  errEl.style.display = 'none';
  if (!reason) {
    errEl.textContent = '理由を入力してください';
    errEl.style.display = '';
    return;
  }
  const btn = document.getElementById('request-delete-submit-btn');
  btn.disabled = true; btn.textContent = '送信中…';
  try {
    const session = getLoginSession();
    const res = await fetch(`${API_BASE}request_delete`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        guild_id: GUILD_ID, session_token: session?.session_token,
        category: requestDeleteCtx.category, filename: requestDeleteCtx.filename, reason,
      }),
      signal: AbortSignal.timeout(10000),
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '送信に失敗しました');
    closeModal('modal-request-delete');
    const via = data.notified_via === 'web_pending'
      ? '作成者がDiscord未連携のため、次回サイトを開いたときに確認されます。'
      : '作成者にDiscordで確認を送りました。承認されると削除されます。';
    showBanner(via, '#dcfce7', '#166534', Icons.html('mailSent', {size:15}));
  } catch (e) {
    btn.disabled = false; btn.textContent = '送信する';
    errEl.textContent = e.message;
    errEl.style.display = '';
  }
}
```
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
```js
// ── 新規作成 ──────────────────────────
function openNewSet() {
  document.getElementById('new-set-name').value = '';
  document.getElementById('new-plan-publish').checked = true; // ★ 追加：毎回デフォルトで「公開予定」に戻す
  // ★ 追加：多肢選択デッキのトグルも毎回OFFへ戻す
  document.getElementById('new-choice-mode-enabled').checked = false;
  showScreen('new');
  loadSubjects();
  setTimeout(() => document.getElementById('new-set-name').focus(), 200);
}

async function loadSubjects() {
  const sel = document.getElementById('new-subject');
  sel.innerHTML = '<option value="">読み込み中…</option>';
  try {
    // ★ cache: 'no-store' を追加：科目（チャンネル）一覧が古いまま
    //   表示され続けることを防ぐため。
    const res  = await fetch(`${API_BASE}channels?guild_id=${GUILD_ID}`, { cache: 'no-store' });
    const data = await res.json();
    if (!data.ok || !data.channels.length) throw new Error();
    sel.innerHTML = '<option value="">科目を選択（任意）</option>' +
      data.channels.map(c => `<option value="${esc(c.name)}">${esc(c.name)}</option>`).join('');
  } catch(e) {
    sel.innerHTML = '<option value="">（科目を取得できませんでした）</option>';
  }
}
```
`openNewSet()`は入力欄をリセットし（公開予定トグルは毎回デフォルトON、多肢選択トグルは毎回OFF）、`screen-new`を表示して`loadSubjects()`で科目一覧を読み込みます。

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

### 5.3 空デッキをサーバーに先行登録：`announceNewDeckToServer(deckId)`（2559〜2614行）
```js
async function announceNewDeckToServer(deckId) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck) return;
  const session = getLoginSession();
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 8000);
    // ★ 修正：以前はここで常に cards: [] を送ってしまっていたため、
    //   （例：デッキ名編集モーダルで「公開予定」を後からONにした場合など）
    //   既にローカルで作成済みのカードが無視され、サーバー側は「0枚」として
    //   登録されてしまっていた。その結果、次に編集画面を開いた際に強制的な
    //   最新化（force reload）でローカルのカードがサーバー側の0枚で
    //   上書きされて消えてしまう、という重大な不具合につながっていた。
    //   ここでは必ず「今ローカルにある実際のカード」をそのまま送る
    //   （まだ1枚も無ければ結果的に空配列になるだけで、これまで通り）。
    const cards = deck.cards.map(cardToServerPayload);
    const res = await fetch(`${API_BASE}save_cards`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: deck.name,
        cards,
        guild_id: GUILD_ID,
        session_token: session ? session.session_token : undefined,
        subject: deck.subject || null,
        description: deck.description || null, // ★ 追加：デッキの説明欄（任意）
        folder_id: deck.folderId || null,
        publisher_id: session ? session.student_id : null,
        publisher_nickname: session ? session.nickname : '匿名',
        silent: true,      // ★ 作成しただけなのでDiscord通知はしない
        incomplete: true,  // ★ まだ「保存して公開」を経ていないので「未完成（作成中）」扱いにする
        choice_mode: deck.choiceMode || null, // ★ 多肢選択デッキかどうか
      }),
      signal: controller.signal,
    });
    clearTimeout(timer);
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
    // ★ POST完了までの間に他の同期処理でdecks配列が入れ替わっている可能性があるため、
    //   必ずこの時点で最新のdecksからidで探し直してから更新・保存する。
    decks = loadDecks();
    const target = decks.find(d => d.id === deckId);
    if (target) {
      target.filename = data.filename;
      target.count = cards.length; // ★ 修正：実際に送ったカード数を反映する（常に0にしない）
      target.cardsLoaded = true;
      target.published_by = session ? session.nickname : '匿名';
      target.incomplete = true;
      target.notYetPublished = true; // ★ まだ「公開して保存」を経ていないので「作成中」のまま
      saveDecks(decks);
    }
  } catch (e) {
    // ★ サーバー登録に失敗した場合は、これまで通りこの端末だけの下書き
    //   （filenameなし）として続行する。次にカードを保存して公開すれば
    //   その時にサーバーへ反映される。
  }
}
```
- 既存の`save_cards`というAPI（カード保存用）を、あえて「中身が空（または少数）の状態」で呼び出すことで、デッキそのものをサーバーに登録します。
- `silent: true`：この時点ではDiscordへの通知はしません（作成しただけなので、まだ知らせるべき内容が無いため）。
- `incomplete: true`：「未完成（作成中）」として登録します。
- `description: deck.description || null`（★ 追加）：デッキの説明欄（任意）を`subject`と同じ扱いで一緒に送ります。作成直後はまだ`deck.description`が設定されていないことが多く、その場合は`null`のまま登録されます（後から`saveRename`で設定・変更できます、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)参照）。
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
  // ★ 追加：デッキに説明が設定されていれば #edit-deck-desc に表示する（無ければ隠す）
  const descBox = document.getElementById('edit-deck-desc');
  if (deck.description) { descBox.textContent = deck.description; descBox.style.display = ''; }
  else { descBox.style.display = 'none'; descBox.textContent = ''; }
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
- ★ 追加：`deck.description`（デッキの説明欄、任意）が設定されていれば、編集画面の一番上（CSV読み込みブロックより前）に`#edit-deck-desc`として全文表示します。編集はこの画面からではなく、デッキメニューの「デッキ名を変更する」から開く`modal-rename`モーダルで行います（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)参照）。

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
  // ★ 追加：多肢選択デッキの選択肢入力欄も、カードを1枚保存するたびに空へ戻す
  const deck = decks.find(d => d.id === currentDeckId);
  if (deck && deck.choiceMode) {
    renderChoiceEditorRows('ta-choice', ['', ''], []);
  }
}
```
- 問題文・解答・解説の入力欄と、添付画像のバッファをまとめて空にします。
- `el.dispatchEvent(new Event('input', {bubbles: true}))`は、コードから「今、入力欄に入力があったことにする」という**擬似的なイベントを発生させる**書き方です。これにより、本来はユーザーが実際にタイプしたときだけ動く「数式プレビューの更新」処理（[09_Cardmaker.js_その9_数式入力とリアルタイム更新.md](09_Cardmaker.js_その9_数式入力とリアルタイム更新.md)で説明）を、プログラム側からも同じように動かすことができます。

---

続きは[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)で、CSV読み込みの入口からカードの保存・デッキの公開までを解説します。
