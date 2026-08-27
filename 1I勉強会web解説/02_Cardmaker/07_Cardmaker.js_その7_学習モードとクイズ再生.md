# Cardmaker.js その7：学習モードと一人用クイズの入口（3750〜4167行）

[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の続きです。

---

## 1. （廃止）一人用クイズへの入口だった`startSoloQuiz`

★ 2026/08/27・2回目の変更で、この関数と専用画面（`screen-quiz-play`）自体を廃止した。経緯は9-7節を参照。クイズ過去問・多肢選択デッキも、下の2〜3節で見る通常デッキと全く同じ`openPlayMode`/`startStudyMode`/`screen-study`を通る（`Cardmaker-quizplay.js`にはデッキ完走後のスコア送信・ランキング取得だけが残っている＝[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)参照）。

---

## 2. プレイモード選択：`openPlayMode(deckId)`（3762〜3837行）

デッキの「▶プレイ」ボタンを押したときの入口です。

### 2.1 変遷（2026/08/27）

初版はここで「クイズ過去問」フォルダの中のデッキ・多肢選択デッキだけを別扱いにし、通常のプレイモード選択モーダル（`modal-play-mode`）を経由せず、専用の簡易ダイアログ（`showCmChoiceDialog`で「一人でプレイ」/「みんなでクイズを始める」を選ばせる）へ直接分岐させていた。ユーザーから「過去問についてはプレイを押したときの選択肢は、続きから・すべてのカード・みんなでクイズ・一覧でミルといったようにふつうのデッキと同じでいい。違うのは逆モードや自動採点４択が表示されない」という指摘を受け、**専用ダイアログを廃止し、通常デッキと全く同じ`modal-play-mode`を経由させる**方式に作り直した。

```js
async function openPlayMode(deckId) {
  const deck = decks.find(d => d.id === deckId);
  if (!deck) return;
  studyIsFolder = false;
  studyDeckId = deckId;
```
- 通常デッキ・クイズ過去問/多肢選択デッキのどちらであっても、まず`studyIsFolder`/`studyDeckId`をセットする（以前は専用ダイアログの分岐がこれより前にあり、通過できないケースがあった）。

続くカード読み込み（`waitForPendingSync`→`ensureDeckCardsLoaded`、失敗時の再試行モーダル）は変更なし。読み込み後、モーダルの中身を組み立てる際に分岐が入る：

```js
  document.getElementById('reverse-mode-checkbox').checked = false;
  document.getElementById('auto-grade-checkbox').checked = false;
  document.getElementById('four-choice-checkbox').checked = false;
  // ★ 追加：クイズ過去問・多肢選択デッキは反転モード・自動採点系のトグル自体を出さない
  const isQuizDeck = isQuizPlayDeck(deck);
  document.getElementById('reverse-toggle-row').style.display = isQuizDeck ? 'none' : '';
  if (isQuizDeck) {
    document.getElementById('auto-grade-toggle-row').style.display = 'none';
    document.getElementById('four-choice-toggle-row').style.display = 'none';
  } else {
    onReverseModeToggleChange();
    onAutoGradeToggleChange();
  }
  document.getElementById('play-mode-deck-name').textContent = deck.name;
```
- `isQuizPlayDeck(deck)`（新設のヘルパー、`isDeckInFolderScope(deck.id, QUIZ_ARCHIVE_FOLDER_ID) || deck.choiceMode`をまとめただけ）でクイズ過去問/多肢選択デッキかどうかを判定し、そうであれば反転モード（`reverse-toggle-row`）・自動採点（`auto-grade-toggle-row`）・4択にする（`four-choice-toggle-row`）の3行を丸ごと`display:none`にする。それ以外（通常デッキ）は従来通り`onReverseModeToggleChange()`/`onAutoGradeToggleChange()`で状態に応じて出し分ける。
- これ以降（「続きから」項目の表示・「わからないカードだけ」の件数計算・「みんなでクイズを始める」の表示条件・`openModal('modal-play-mode')`）は通常デッキ・クイズ過去問デッキで完全に共通。「一覧で見る」（`onclick="openListView()"`）も元々このモーダルの一項目としてあったため、クイズ過去問デッキでも自動的に選べるようになった（詳細は4節・[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)参照。選択式デッキに反転モードの概念が無いこと・正解以外の選択肢も小さく添えて表示すること・「クイズ過去問」では個別カードの編集ボタンを出さないことが、通常デッキの一覧表示と異なる点）。

通常のフラッシュカードデッキの場合は、下に続く処理でプレイモード選択モーダル（`modal-play-mode`）を開く準備をします：

```js
  // ★ 修正：直前にカードを追加/削除した際のサーバー同期（queueSyncDeckToServer）が
  //   まだ完了していない状態で強制リロードすると、その同期前の古い（最悪カード0枚の）
  //   内容をサーバーから取得して上書きしてしまい、「中身があるのに0問で完了」に
  //   なってしまう不具合があった。force reloadの前に必ず保留中の同期を待つ。
  await waitForPendingSync(deckId);
  // ★ 修正：1回失敗しただけで行き止まりのアラートを出して終わらせず、
  //   loadDeckCardsWithRecovery と同様に「もう一度試す」を選べるようにする
  //   （タイムアウトを含む一時的な通信エラーでプレイを諦めなくて済むように）。
  let result = await ensureDeckCardsLoaded(deckId, true);
  while (!result.ok) {
    const retry = await showCmConfirm({
      title: '読み込みに失敗しました',
      desc: '通信環境を確認してもう一度お試しください。',
      okLabel: 'もう一度試す', cancelLabel: 'やめる',
    });
    if (!retry) return;
    result = await ensureDeckCardsLoaded(deckId, true);
  }
```
- ここでも「保留中の同期を待つ→強制的に最新カードを取得」という同じ順序が守られています。失敗した場合は、`while`ループで何度でも「もう一度試す」を選べるようにしています。

```js
  document.getElementById('reverse-mode-checkbox').checked = false; // ★ プレイモード選択のたびに未チェックへリセット
  document.getElementById('auto-grade-checkbox').checked = false; // ★ 追加：自動採点トグルも未チェックへリセット
  onReverseModeToggleChange(); // ★ 追加：反転OFFなので自動採点トグルを表示状態にする
  document.getElementById('play-mode-deck-name').textContent = deck.name;
  document.getElementById('play-mode-all-sub').textContent = `${deck.cards.length} 問`;
  const unsure = getUnsureSet(deckId);
  const unsureCount = deck.cards.filter(c => unsure.has(cardKey(c))).length;
  const unsureItem = document.getElementById('play-mode-unsure-item');
  if (unsureCount > 0) {
    document.getElementById('play-mode-unsure-sub').textContent = `${unsureCount} 問`;
    unsureItem.style.display = '';
  } else {
    unsureItem.style.display = 'none';
  }

  // ★ 続きから再開できる場合は「続きから」の項目を表示する
  const savedD = loadStudyProgress(false, deckId);
  const resumeItemD = document.getElementById('play-mode-resume-item');
  if (savedD) {
    document.getElementById('play-mode-resume-sub').textContent = `${savedD.idx + 1} / ${savedD.order.length} 問から`;
    resumeItemD.style.display = '';
  } else {
    resumeItemD.style.display = 'none';
  }

  // ★「みんなでクイズを始める」：公開済み（filenameあり）のデッキだけ表示する
  //   （Quiz.jsはサーバーのget_card_setでデッキを取得するため、非公開のローカル限定
  //   デッキは対象外）。
  document.getElementById('play-mode-quiz-item').style.display = deck.filename ? '' : 'none';

  // ★ 反転トグルを必ず見せるため、わからないカードの有無に関わらずモーダルを開く
  openModal('modal-play-mode');
}
```
- 「みんなでクイズを始める」の項目は、公開済み（`filename`あり）のデッキだけに表示されます。コメントによれば、`Quiz.js`はサーバーの`get_card_set`でデッキを取得する仕組みのため、まだサーバーに登録されていない非公開のローカル限定デッキは対象にできないためです。

```js
// ★ 追加：反転モードのON/OFFに応じて自動採点トグルの表示を切り替える。
//   反転モード（問題と解答を逆にする）中は自動採点の対象がずれてしまうため、
//   反転ONの間はトグル自体を隠し、内部的にもOFFへ強制的に戻しておく。
function onReverseModeToggleChange() {
  const reversed = document.getElementById('reverse-mode-checkbox').checked;
  const row = document.getElementById('auto-grade-toggle-row');
  row.style.display = reversed ? 'none' : '';
  if (reversed) document.getElementById('auto-grade-checkbox').checked = false;
}
```
- 「問題と解答を反転する」トグルがONの間は、「自動採点」トグル自体を隠し、内部的にも強制的にOFFに戻します。反転モード中に自動採点をしようとすると、採点の対象（何を正解として比較するか）がずれてしまうための保護です。

---

## 3. 学習を開始する：`startStudyMode(mode)`（3865〜3956行）

`mode`は`'all'`（すべて）／`'unsure'`（わからないだけ）／`'resume'`（続きから）のいずれかです。

### 3.0 クイズ過去問/多肢選択デッキも同じ画面でプレイする（2026/08/27、2回目の変更）

★ 初版（1回目の変更）は、クイズ過去問/多肢選択デッキだけ`startSoloQuiz`（専用画面`screen-quiz-play`）に丸ごと振り分けていた。ユーザーから「プレイ中の画面もほかのカードでのプレイ画面とほぼ同じにしてほしい」「編集ボタンだけ消して、それ以外は同じ画面」という指摘を受け、**専用画面自体を廃止し、通常デッキと全く同じ`screen-study`（フラッシュカード画面）でプレイする**方式に作り直した（9-7節に詳しい）。`isQuizPlayDeck(deck)`はそのままだが、使い道が「別画面に飛ばす分岐」から「同じ画面の中でクイズ用の値を使う分岐」に変わっている。

```js
let studyIsQuizDeck = false;
let studyQuizAnswers = new Map(); // cardKey → isCorrect（クイズプレイ中の各カードの最新の正誤）
let studyQuizSelected = new Set(); // 複数正解モードで、まだ確定前に選んでいる選択肢

async function startStudyMode(mode) {
  const progressId = studyIsFolder ? studyFolderId : studyDeckId;
  const quizDeck = !studyIsFolder ? decks.find(d => d.id === studyDeckId) : null;
  studyIsQuizDeck = isQuizPlayDeck(quizDeck);

  if (!studyIsQuizDeck) {
    studyReverse = document.getElementById('reverse-mode-checkbox').checked;
    studyAutoGrade = !studyReverse && document.getElementById('auto-grade-checkbox').checked;
    studyFourChoice = studyAutoGrade && document.getElementById('four-choice-checkbox').checked;
  } else {
    studyReverse = false; studyAutoGrade = false; studyFourChoice = false;
  }
```
- `studyIsQuizDeck`はモジュールグローバル（`studyIsFolder`等と同格）になり、`renderStudyCard`・`currentStudyChoiceEntry`・`shuffleStudy`・キーボードショートカットなど画面のあちこちから参照される。
- クイズ過去問/多肢選択デッキは、2節の通り反転・自動採点系のチェックボックス自体を表示していないため読み取らない（`false`に固定）。

「続きから」データの上書き確認（3.1節、共通のまま）のあと、`mode==='resume'`（3.2節）／`else`（3.3節）どちらの分岐も、**もう`startSoloQuiz`へ丸投げしない**。同じ`studyCards`/`studyIdx`を組み立てる処理を両デッキ種別で共有しつつ、クイズ過去問/多肢選択デッキのときだけ次の2点を追加で行う：

1. **選択式として遊べるカードだけに絞る**（`normalizeQuizPlayableCards(cards)`という新設ヘルパー。以前`Cardmaker-quizplay.js`の`startSoloQuiz`内にあった「`choices`が`CHOICE_MIN`件未満のカードを除外し、`correct_index`単数を`correct_indices`配列へ正規化する」ロジックをそのまま移設したもの）。`mode==='resume'`なら復元用の`pool`に、`else`なら`deck.cards`（または`mode==='unsure'`ならその中の絞り込み前）に適用する。絞った結果0件なら、`showCmAlert`で「選択式の問題がありません」／「わからないカードはありません」を出して中断する（通常デッキには無い、クイズ過去問固有のガード）。
2. **`studyQuizAnswers`の初期化/復元**：新規開始（`else`分岐）なら空の`Map`にリセット、`resume`なら`saved.quizAnswers`（配列化して保存されている、後述4.x節ではなく`saveStudyProgress`の解説＝6節参照）から復元する。スコアは素点のカウンタではなく、この`Map`（cardKey→正誤）から都度数え直す方式にしてある（「← 前へ」で戻って答え直しても二重加点しないため。9-7節で詳述）。

`setupFourChoiceIfNeeded()`（AI4択の事前準備、6節）の呼び出しも`if (!studyIsQuizDeck)`でガードするようになった（クイズ過去問/多肢選択デッキは選択肢が最初からカードに入っているため不要）。最後に、画面を開く直前に次の1行が増えている：

```js
  document.getElementById('study-edit-btn').style.display = (quizDeck && quizDeck.quizArchive) ? 'none' : '';
```
- 「クイズ過去問」デッキ（`quizArchive`。ホストが作ったオリジナル4択の自動保存分）はデッキメニューの「編集」自体を隠している読み取り専用デッキ（[3節の元記事](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)の`openDeckMenu`参照）なので、学習画面の「編集」ボタンも同様に隠す。**`studyIsQuizDeck`より狭い条件**（`quizDeck.quizArchive`）を使っているのがポイントで、ユーザーが自作した多肢選択デッキ（`choiceMode`はあるが`quizArchive`ではない）はカード編集が引き続きできるため対象外にしてある。

### 3.1 「続きから」データの上書き確認
```js
  // ★ 追加：「すべてのカード」「わからないカードだけ」を選んだ場合、
  //   既に「続きから」の再開データが残っていると、この後の処理で
  //   問答無用でそのデータが破棄されてしまう（clearStudyProgress）。
  //   気づかないうちに再開位置が消えてしまわないよう、事前に確認する。
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

  closeModal('modal-play-mode');
```
- 「すべて」や「わからないだけ」を選んだ場合、これから始める処理（後述）が保存されていた「続きから」の再開データを問答無用で消してしまいます。気づかないうちに再開位置が失われてしまわないよう、事前に一言確認を挟んでいます。

### 3.2 「続きから」を選んだ場合
```js
  if (mode === 'resume') {
    // ★ 保存された進捗（カードキーの並び順・位置・モード・反転設定・シャッフル済みか）を復元する。
    //   カード本体は常に最新の decks / folderPlayDecks から引き直すので、
    //   編集や画像追加が続きから再開に影響しない。
    const saved = loadStudyProgress(studyIsFolder, progressId);
    if (!saved) return; // 万が一データが消えていた場合は何もしない
    studyReverse = saved.reverse;
    studyAutoGrade = !saved.reverse && !!saved.autoGrade; // ★ 追加：保存されていた自動採点設定を復元
    studyMode = saved.mode || 'all';
    studyShuffled = !!saved.shuffled; // ★ シャッフル済みだったかどうかを復元（タイトル表示用）

    let pool;
    if (studyIsFolder) {
      pool = [];
      folderPlayDecks.forEach(d => d.cards.forEach(c => pool.push({ ...c, __deckId: d.id })));
      const folder = folders.find(f => f.id === studyFolderId);
      studyBaseTitle = folder ? folder.name : 'フォルダ';
    } else {
      const deck = decks.find(d => d.id === studyDeckId);
      pool = deck ? [...deck.cards] : [];
      studyBaseTitle = deck ? deck.name : '';
    }
    const byKey = new Map(pool.map(c => [cardKey(c), c]));
    // ★ order は保存時点の並び順（シャッフル済みならその並び）をそのまま記録しているので、
    //   ここで単純にキーから引き直すだけで、シャッフルした状態のまま正しく再開できる。
    studyCards = saved.order.map(k => byKey.get(k)).filter(Boolean);
    if (!studyCards.length) return; // カードが全部消えていた場合は何もしない
    studyIdx = Math.min(saved.idx, studyCards.length - 1);
```
- 保存されていた「並び順（カードキーの配列）」を、**今の最新のカードデータ**から作った`Map`（キー→カード本体、の対応表）と突き合わせて、実際の学習カード配列（`studyCards`）を組み立て直します。こうすることで、保存後に他の人がカードを編集していても、その最新の内容で続きから再開できます。もし途中で削除されたカードがあれば`.filter(Boolean)`で自然に除外されます（`byKey.get(k)`が`undefined`になるため）。
- `studyIdx = Math.min(saved.idx, studyCards.length - 1)`：もしカードが削除されて全体の枚数が減っていた場合でも、保存されていた位置が配列の範囲外にならないよう安全に調整しています。

### 3.3 「すべて」「わからないだけ」を選んだ場合
```js
  } else {
    studyMode = mode;
    studyShuffled = false; // ★ 「すべて」「わからないだけ」を選び直した場合はシャッフル状態をリセット
    if (studyIsFolder) {
      // フォルダ内の全デッキのカードを、どのデッキ由来かのタグ付きでまとめる
      const merged = [];
      folderPlayDecks.forEach(d => {
        const unsure = mode === 'unsure' ? getUnsureSet(d.id) : null;
        d.cards.forEach(c => {
          if (mode === 'unsure' && !unsure.has(cardKey(c))) return;
          merged.push({ ...c, __deckId: d.id }); // ★ 元のデッキidを保持
        });
      });
      studyCards = merged;
      const folder = folders.find(f => f.id === studyFolderId);
      studyBaseTitle = folder ? folder.name : 'フォルダ';
    } else {
      const deck = decks.find(d => d.id === studyDeckId);
      if (mode === 'unsure') {
        const unsure = getUnsureSet(studyDeckId);
        studyCards = deck.cards.filter(c => unsure.has(cardKey(c)));
      } else {
        studyCards = [...deck.cards];
      }
      studyBaseTitle = deck.name;
    }
    studyIdx = 0;
    // ★ 「すべて」「わからないだけ」を新しく選び直した場合は、
    //   古い「続きから」データを破棄する（そのまま残すと内容と矛盾するため）
    clearStudyProgress(studyIsFolder, progressId);
  }

  renderStudyTitle();
  document.getElementById('study-done-sub').textContent = `全 ${studyCards.length} 問完了！`;
  showScreen('study');
  document.getElementById('study-done').style.display    = 'none';
  document.getElementById('study-content').style.display = 'flex';
  renderStudyCard();
  loadUnderstandingBadge(); // ★ 追加：みんなの「わかる率」を右上に読み込む（非同期・表示はブロックしない）
}
```
- フォルダをまとめて学習している場合、各カードに`__deckId`（元々どのデッキに属していたか）を付け足しておきます。これは、フォルダ横断で1つの学習セッションとして扱いながらも、「このカードはどのデッキ由来か」を後から参照できるようにするための工夫です（学習画面での「わからない」マークの保存先や、カード編集時にどのデッキを更新すべきか、を判断するのに使われます）。
- 新しく選び直した場合は、古い「続きから」のデータは矛盾するため`clearStudyProgress`で消しておきます。

---

## 4. 一覧表示画面への入口（3942〜3956行）

`openListView()`も、2節・[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で見た「遅延読み込みチャンクの仮の窓口」パターンと同じです。実体は`Cardmaker-listview.js`（[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)）にあります。

コメントには、この入口を経由しない例外（検索結果からの直接ジャンプ）についても触れられており、「一覧で見る」画面が実際に開くまで押せないはずのボタン（`setListViewFilter`など）は、この入口が読み込みを待つことで間接的に守られている、という設計上の注記があります。

---

## 5. 学習カードの表示：`renderStudyCard()`（3981〜4068行）

### 5.1 元の問題番号バッジ（3974〜3996行）
```js
// ★ 追加：プレイ中のカードが「元のデッキ順で何問目か」を、
//   青色の「問題」ラベル（.study-q-tag）の横に番号だけ表示する。
//   ─────────────────────────────────────────
//   シャッフルすると study-prog-label（例:「3 / 20」）は再生順の位置に
//   なってしまい、元の問題番号が分からなくなる。このバッジは常に
//   元のデッキ内でのカード順（deck.cards内でのインデックス）の番号だけを表示する。
//   バッジ要素はHTML側に無いので、初回はJSで動的に作って隣に挿入する。
function updateStudyOriginalNumberBadge(c) {
  let badge = document.getElementById('study-orig-num-badge');
  if (!badge) {
    const label = document.querySelector('.study-q-tag');
    if (!label) return; // 「問題」ラベルが見つからなければ何もしない
    badge = document.createElement('span');
    badge.id = 'study-orig-num-badge';
    badge.style.cssText = 'margin-left:4px;';
    label.appendChild(badge); // ★「問題」の文字のすぐ右（タグの中）に入れる
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

```js
function renderStudyCard() {
  const progressId = studyIsFolder ? studyFolderId : studyDeckId;
```

### 5.2 学習完了時の処理
```js
  if (studyIdx >= studyCards.length) {
    document.getElementById('study-content').style.display = 'none';
    document.getElementById('study-done').style.display    = 'flex';
    document.getElementById('study-prog-fill').style.width  = '100%';
    document.getElementById('study-prog-label').textContent = `${studyCards.length} / ${studyCards.length}`;
    const doneBadge = document.getElementById('study-orig-num-badge');
    if (doneBadge) doneBadge.textContent = '';
    clearStudyProgress(studyIsFolder, progressId); // ★ 完了したら続きデータは不要になるので消す
    saveCompletionRecord(studyIsFolder, progressId, studyCards.length); // ★ 追加：完了したことを記録する
    renderInProgressUI(); // ★ 追加：ホームの「プレイ中」「プレイ済み」欄を最新状態に更新
    return;
  }
```
- 全部のカードを見終えたら、完了画面に切り替え、「続きから」のデータは不要になるので消し、代わりに「完了した」という記録（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の`saveCompletionRecord`）を残します。ホーム画面の「プレイ中」「プレイ済み」欄もここで最新化されます。

### 5.3 通常のカード表示
```js
  const c = studyCards[studyIdx];
  markCardSeen(studyIsFolder ? c.__deckId : studyDeckId, c); // ★ 追加：みんなの「わかる率」用に学習済み記録

  // ★ 反転モードなら「問題」欄に解答、「解答」欄に問題文を出す（解説はそのまま解答側に表示）
  const qText = studyReverse ? c.answer   : c.question;
  const qImgs = studyReverse ? c.imgs_a   : c.imgs_q;
  const aText = studyReverse ? c.question : c.answer;
  const aImgs = studyReverse ? c.imgs_q   : c.imgs_a;

  setMathText(document.getElementById('study-q-text'), qText);
  renderImgList(document.getElementById('study-q-imgs'), qImgs);
  // ★ フォルダをまとめてプレイしている場合、この問題がどのカードデッキ由来かを表示する
  const deckTag = document.getElementById('study-deck-tag');
  if (studyIsFolder) {
    const srcDeck = decks.find(d => d.id === c.__deckId);
    if (srcDeck) {
      setIconText(deckTag, Icons.html('cardmaker', {size:14}), srcDeck.name);
      deckTag.style.display = '';
    } else {
      deckTag.style.display = 'none';
    }
  } else {
    deckTag.style.display = 'none';
  }
  document.getElementById('study-answer-panel').classList.remove('show');
  document.getElementById('study-reveal-bar').style.display = 'flex';
  document.getElementById('study-nav').style.display = 'none';
```
- カードを表示するたびに`markCardSeen`（「みんなのわかる率」用の学習済み記録、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)）を呼びます。
- 反転モードなら問題欄と解答欄の中身を入れ替えて表示します。

```js
  // ★ 修正：解答入力欄は反転モードかどうかに関わらず常に表示する（自問自答の確認用）。
  //   反転モード中は studyAutoGrade が常に false になる（onReverseModeToggleChange /
  //   startStudyMode 側で強制）ため、ここで欄を表示していても自動採点（○×判定）は
  //   行われない。あくまで「入力欄を使って自分で書いてみる」ことだけができる。
  const answerInputWrap = document.getElementById('study-answer-input-wrap');
  const answerInput = document.getElementById('study-answer-input');
  answerInputWrap.style.display = '';
  answerInput.value = '';
  const gradeResult = document.getElementById('study-grade-result');
  gradeResult.style.display = 'none';
  gradeResult.className = 'study-grade-result';
  document.getElementById('reveal-answer-btn').textContent = studyAutoGrade ? '採点する' : '答えを見る';

  setMathText(document.getElementById('study-a-text'), aText);
  renderImgList(document.getElementById('study-a-imgs'), aImgs);
  const explWrap = document.getElementById('study-expl-wrap');
  if (c.explanation) { setMathText(document.getElementById('study-e-text'), c.explanation); explWrap.style.display = ''; }
  else { explWrap.style.display = 'none'; }
  const pct = studyCards.length > 1 ? (studyIdx/(studyCards.length-1))*100 : 100;
  document.getElementById('study-prog-fill').style.width  = pct + '%';
  document.getElementById('study-prog-label').textContent = `${studyIdx+1} / ${studyCards.length}`;
  updateStudyOriginalNumberBadge(c); // ★ 追加：シャッフル時も元の問題番号がわかるように表示
```
- コメントによれば、解答入力欄は反転モードかどうかに関わらず**常に**表示されます。反転モード中は自動採点自体が行われませんが（`studyAutoGrade`が常に`false`になるよう別の場所で強制されている）、入力欄自体は「自分で書いてみて確認する」という使い方ができるよう、あえて隠していません。

```js
  // ★ 答えを見る前・見た後、両方の「前へ」ボタンの有効/無効を同期
  document.getElementById('study-prev').disabled     = studyIdx === 0;
  document.getElementById('study-prev-pre').disabled = studyIdx === 0;
  document.getElementById('study-next').innerHTML = studyIdx === studyCards.length-1 ? ('完了 ' + Icons.html('check', {size:14})) : '次へ →';
  updateUnsureBtn();
  saveStudyProgress(); // ★ カードを表示するたびに現在位置を保存し、次回「続きから」を出せるようにする
}
```
- 最後のカードなら「次へ」ボタンの文言を「完了」に変え、「わからない」ボタンの見た目を更新し、**カードを表示するたびに毎回**`saveStudyProgress()`（学習の続きを保存）を呼びます。これにより、学習中にアプリを閉じても、次に開いたときにちょうど今見ていたカードから再開できます。

---

## 6. 答えを見る・自動採点（4070〜4113行）

```js
function revealAnswer() {
  document.getElementById('study-answer-panel').classList.add('show');
  document.getElementById('study-reveal-bar').style.display = 'none';
  document.getElementById('study-nav').style.display = '';
  if (studyAutoGrade) gradeCurrentAnswer(); // ★ 追加：自動採点モードなら○×判定を行う
  updateUnsureBtn();
}
```
- 「答えを見る」（自動採点モードなら「採点する」）ボタンが押されたときの処理です。自動採点モードなら`gradeCurrentAnswer()`で判定を行います。

```js
// ★ 追加：自動採点まわりの処理
//   ─────────────────────────────────────────
//   入力欄の解答と正解テキストを正規化（前後の空白・全角スペースを除去し小文字化）して比較し、
//   一致していれば○正解、そうでなければ×不正解と判定する。
//   ×だった場合は自動で「わからない」にマークする（既にマーク済みなら何もしない）。
//   ○だった場合は既存の「わからない」マークを勝手に外したりはしない。
function normalizeAnswerText(s) {
  return (s || '').toLowerCase().replace(/[\s　]/g, '');
}
function gradeCurrentAnswer() {
  const card = studyCards[studyIdx];
  if (!card) return;
  const inputEl = document.getElementById('study-answer-input');
  const input = inputEl ? inputEl.value : '';
  const correctText = studyReverse ? card.question : card.answer; // 自動採点は反転モードでは使わない想定だが念のため
  const normInput = normalizeAnswerText(input);
  const isCorrect = normInput !== '' && normInput === normalizeAnswerText(correctText);

  const result = document.getElementById('study-grade-result');
  const mark = document.getElementById('grade-mark');
  const userAnswerEl = document.getElementById('grade-user-answer');
  result.style.display = 'flex';
  result.className = 'study-grade-result ' + (isCorrect ? 'correct' : 'incorrect');
  mark.innerHTML = isCorrect ? '○ 正解' : (Icons.html('close', {size:14}) + ' 不正解');
  userAnswerEl.textContent = 'あなたの解答：' + (input.trim() ? input : '（未入力）');

  if (!isCorrect) {
    const key = cardKey(card);
    const deckId = card.__deckId || studyDeckId;
    const unsure = getUnsureSet(deckId);
    if (!unsure.has(key)) {
      unsure.add(key);
      saveUnsureSet(deckId, unsure);
    }
  }
}
```
- `normalizeAnswerText`は、比較の前に「小文字化」＋「半角/全角スペースの除去（`\s`が半角の空白、`　`が全角スペースの文字コード）」を行う関数です。これにより、大文字/小文字の違いや余分な空白があっても、内容が同じなら正解と判定できるようにしています（逆に言うと、これ以外の表記ゆれ、例えば送り仮名の違いなどは不正解と判定されます）。
- 正解/不正解の見た目（○/×マーク・入力した内容の表示）もここで組み立てています。
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
  studyShuffled = true; // ★ 追加：シャッフル済み状態にする。以降の saveStudyProgress で保存され、
                        //   「続きから」で再開したときもこのシャッフル順のまま復元される。
  renderStudyTitle(); // ★ タイトルにシャッフル中を表示（studyShuffledは直前にtrueへ更新済み）
  document.getElementById('study-done').style.display    = 'none';
  document.getElementById('study-content').style.display = 'flex';
  renderStudyCard();
  saveStudyProgress(); // ★ 念のため即座に保存しておく（renderStudyCard内でも保存されるが二重に確実化）
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
- **（2026/08/26追加）** 自動採点＋「4択にする」モードで、かつ今表示中のカードが実際に4択になっている（`studyChoicesMap`に登録されている）場合は、スペースキーでの`revealAnswer()`（テキスト入力前提の採点）を呼ばない。4択は下記9章の`answerStudyChoice()`（ボタンクリック）でしか答えられないため。

---

## 9. 自動採点＋「4択にする」モード（2026/08/26追加、2026/08/27にトグルの見せ方を変更）

「自動採点する（解答入力）」をONにすると、さらに「4択にする（みんなでクイズと同じ形式）」というトグルを選べる。ONにすると、解答をテキストで入力する代わりに、[../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md](../../1I勉強会bot解説/15_FlaskAPI_クイズ/01_ルーム状態のJSON化と問題の自動生成.md)で解説した「みんなでクイズ」と同じ4択ボタン（`qp-choices`/`qp-choice-btn`のCSSをそのまま流用）で答える形式になる。

### 9-1. トグルの表示制御（2026/08/27に変更）

初版（2026/08/26）は「自動採点する」をONにした時だけ「4択にする」の行がその場に現れる、折りたたみ式のサブトグルだった。ユーザーから「普通にプレイするとき、4択にするはたたまず表示。押されたら自動採点も自動でオン」という要望を受け、**「4択にする」を（反転モード中を除いて）常に表示する独立したトグルに変更**し、ONにした側から自動採点を連動させる向きに逆転させた。

```js
function onReverseModeToggleChange() {
  const reversed = document.getElementById('reverse-mode-checkbox').checked;
  document.getElementById('auto-grade-toggle-row').style.display = reversed ? 'none' : '';
  document.getElementById('four-choice-toggle-row').style.display = reversed ? 'none' : '';
  if (reversed) {
    document.getElementById('auto-grade-checkbox').checked = false;
    document.getElementById('four-choice-checkbox').checked = false;
  }
}

function onAutoGradeToggleChange() {
  const on = document.getElementById('auto-grade-checkbox').checked;
  if (!on) document.getElementById('four-choice-checkbox').checked = false;
}

function onFourChoiceToggleChange() {
  if (!document.getElementById('four-choice-checkbox').checked) return;
  const autoGrade = document.getElementById('auto-grade-checkbox');
  if (!autoGrade.checked) { autoGrade.checked = true; onAutoGradeToggleChange(); }
}
```
- 表示・非表示は反転モードだけで決める（`onReverseModeToggleChange`が自動採点行・4択行の両方を同時に出し分ける）。反転モードとの併用は元々意味を持たないため、反転ON時はこれまで通り両方隠して強制OFFにする。
- `onAutoGradeToggleChange`は「表示を切り替える」役目を失い、「自動採点をOFFにしたら4択も連動してOFFにする」（4択は自動採点が前提の機能なので、チェックだけ残ると`studyFourChoice`の実際の判定とUIの見た目がズレてしまうのを防ぐ）片方向の後始末だけが残った。
- 新設の`onFourChoiceToggleChange`（`four-choice-checkbox`の`onchange`）が、要望の「押されたら自動採点も自動でオン」を実現する。4択をONにした瞬間、まだ自動採点がOFFならこちらを先にONにしてから`onAutoGradeToggleChange()`を呼ぶ（この時点では4択は既にONなので、後始末側で消されることはない）。

### 9-2. 選択肢の組み立て：`setupFourChoiceIfNeeded`（デッキ全体を1問題プールに）

`startStudyMode`内、`studyCards`が確定した直後（`renderStudyCard()`を呼ぶ前）に呼ばれる。

```js
function setupFourChoiceIfNeeded() {
  studyChoicesMap = new Map();
  _fourChoiceAiRunToken++;
  if (!studyFourChoice || !studyCards.length) return;

  const poolByDeck = new Map();
  function poolFor(deckId) { /* deck.cardsのanswerを重複除去して集める */ }

  studyCards.forEach(card => {
    const correct = ((studyReverse ? card.question : card.answer) || '').trim();
    if (!correct) return;
    const deckId = card.__deckId || studyDeckId;
    const pool = poolFor(deckId).filter(a => a !== correct);
    if (pool.length <= 10) return; // プールが薄いカードは4択にできない（下記の追記参照）
    studyChoicesMap.set(cardKey(card), buildChoiceEntry(correct, pool));
  });

  if (studyChoicesMap.size) scheduleFourChoiceAiEnhancement();
}
```
- **選択肢のプールはデッキ単位**。フォルダをまとめて再生している場合は、カードごとに`card.__deckId`（そのカードが元々属していたデッキ）でプールを切り分ける（フォルダ内の他デッキの解答は混ぜない）。
- **（2026/08/26追記）プール基準を段階的に厳格化**：当初はbot.py側`_build_deck_questions`と同じ「不正解3つを選ぶには答えの異なりが最低3つ必要」という数学的な最低ラインをそのまま採用していたが、ちょうど3つしか無い場合は`buildChoiceEntry`のスコアによる選別が一切効かず（3件全部をそのまま誤答にするしかない）、「明らかに関係ない誤答」がそのまま混ざってしまうという指摘を受け、まず「6件以上」へ引き上げた。それでもまだ選別の余地が乏しいとの指摘を受け、最終的に**「10件を超える（11件以上）」**へさらに厳しくした。基準を上げるほど対象デッキが絞られる代わりに、4択にできたデッキの候補プールは元々大きくなり、AIに渡す候補（`shortlist`、下記参照）も自然と充実する。
- **1問だけ4択にできなくても、その問題だけ通常入力にフォールバックする**：`studyChoicesMap`に登録されなかったカードは、後述の`renderStudyCard()`が自動的に通常の解答入力欄を表示する。デッキ全体を諦めさせるような全体判定（アラート等）はしていない。
- `buildChoiceEntry(correct, pool)`は、`_bigramSimilarity`（2文字bigramのDice係数、Python側`difflib.SequenceMatcher.ratio()`の簡易的な代替）と文字数の近さを7:3で組み合わせたスコアで`pool`を並べ替え、上位3〜6件からランダムに3件を誤答として選ぶ（`_pick_distractors`と同じ「上位群からランダムに選ぶことで、正解に対して消去法が効きにくい4択にする」考え方）。上位`FOUR_CHOICE_AI_SHORTLIST_SIZE`件（2026/08/26に12→16→**40**へ2段階で拡大。プール基準の厳格化で対象デッキの候補が元々増えたこと、および「綴り類似度の事前絞り込みだけだと、記述式の解答で本当は紛らわしい候補が順位落ちして漏れる」という指摘に合わせた）は`shortlist`としてエントリに保持しておき、次項のAI強化に使う。サーバー側もこれに合わせて`CARDMAKER_AI_SHORTLIST_SIZE`（40、[03_AI誤答強化API.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/03_AI誤答強化API.md)参照）という専用定数を新設した（みんなでクイズ側の`QUIZ_AI_SHORTLIST_SIZE`＝12はユーザーの明示的な希望で現状維持のため、共有せず分けてある）。
- **AIの応答を待たずに即座に学習を始められる**のがポイント。ここで作った4択は同期的（一瞬）に決まるため、`renderStudyCard()`はこの直後にすぐ呼ばれ、プレイヤーを待たせない。

### 9-2b. サーバー事前生成キャッシュの取得：`applyServerChoiceCaches`（2026/08/26追加）

「同じデッキを何人が学習しても、AIの計算がその都度捨てられて無駄になっている」
という指摘を受け、デッキを「公開」保存したタイミングでサーバー側が
バックグラウンドで先に選択肢を作っておく仕組み（`four_choice_cache_<filename>.json`、
[../../1I勉強会bot解説/14_FlaskAPI_CardMaker/04_4択事前生成キャッシュ.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/04_4択事前生成キャッシュ.md)参照）を追加した。
`setupFourChoiceIfNeeded`は`async`関数になり、`startStudyMode`側も
`await setupFourChoiceIfNeeded();`と待つように変更した。

```js
async function setupFourChoiceIfNeeded() {
  ...（プール構築は変更なし）...
  if (!studyChoicesMap.size) return;

  const serverCoveredKeys = await applyServerChoiceCaches();
  scheduleFourChoiceAiEnhancement(serverCoveredKeys);
}

async function applyServerChoiceCaches() {
  const covered = new Set();
  ...
  for (const deckId of deckIds) {
    const deck = decks.find(d => d.id === deckId);
    if (!deck || !deck.filename) continue; // 下書き（未公開）デッキは対象外
    const res = await fetch(
      `${API_BASE}cardmaker_choice_cache?guild_id=${GUILD_ID}&filename=${encodeURIComponent(deck.filename)}`,
      { headers: { 'Authorization': 'Bearer ' + session.session_token }, signal: AbortSignal.timeout(5000) }
    );
    const data = await res.json();
    ...（各カードについて isValidServerDistractors で検証してから studyChoicesMap を上書き、covered に記録）...
  }
  return covered;
}
```
- **学習開始が一瞬だけ長くなる**：`await`を挟むため、以前は完全に同期（一瞬）だった学習開始が、サーバーへの問い合わせ1回ぶん（ファイル読み込みだけなので数十〜数百ms程度）だけ待つようになった。その代わり、間に合えば**最初のカードから**AI強化済みの選択肢を使える（ローカルでのAI強化は数十秒かかることがあるのと対照的）。
- **`isValidServerDistractors`で検証**：サーバーのキャッシュは「デッキ公開時点のスナップショット」に基づくため、その後カードの解答が編集されている可能性がある。正解と重複する・件数が3つそろわない等の場合は使わず、ローカルで組み立てた即席4択のまま残す。
- **`Authorization: Bearer`ヘッダーで認証**：クエリ文字列にトークンを載せない（`ServiceInfo.js`の`loadSystemLog`や`Cardmaker-quizplay.js`の`quiz_archive_leaderboard`取得と同じ慣習）。
- **サーバー側で取得できたカードは`covered`に記録し、`scheduleFourChoiceAiEnhancement`へ渡して除外する**：二重にAIへ問い合わせないようにするため（次項参照）。
- **通信に失敗しても学習は止まらない**：`fetch`が失敗した場合はそのデッキ分だけ諦め（`catch`で握りつぶす）、ローカルの即席4択＋ローカルAI強化に完全にフォールバックする。デッキが「作成中（下書き、filenameが無い）」の場合もサーバー側に事前生成の対象が無いため、最初から問い合わせをスキップする。

### 9-3. バックグラウンドAI強化：`scheduleFourChoiceAiEnhancement`

```js
function isDescriptiveAnswerText(s) {
  if (!s) return false;
  if (s.length >= 20) return true; // ある程度長い解答は記述系とみなす
  if (/[。、．，,.!?！？]/.test(s)) return true; // 句読点を含む＝文章の可能性が高い
  if (/\s/.test(s.trim())) return true; // 単語区切りのスペースを含む＝説明文っぽい
  return false;
}

// skipKeys: applyServerChoiceCaches()が既にサーバーの事前生成結果で差し替え済みのカードキー一覧
async function scheduleFourChoiceAiEnhancement(skipKeys) {
  const myToken = _fourChoiceAiRunToken;
  const session = getLoginSession();
  if (!session || !session.session_token) return;

  const entries = studyCards.map((card, idx) => ({ idx, key: cardKey(card) }))
    .filter(e => studyChoicesMap.has(e.key) && !(skipKeys && skipKeys.has(e.key)));
  // 記述系カードを問い合わせの先頭へ（Array.sortは安定ソート）
  entries.sort((a, b) => {
    const ea = studyChoicesMap.get(a.key), eb = studyChoicesMap.get(b.key);
    return (isDescriptiveAnswerText(ea.choices[ea.correctIndex]) ? 0 : 1)
         - (isDescriptiveAnswerText(eb.choices[eb.correctIndex]) ? 0 : 1);
  });
  let start = 0, isFirstBatch = true;
  while (start < entries.length) {
    if (myToken !== _fourChoiceAiRunToken) return;
    const batchSize = isFirstBatch ? 1 : 3;
    const batch = entries.slice(start, start + batchSize);
    start += batchSize;
    isFirstBatch = false;
    const items = batch.map(e => { /* {i, question, correct, candidates: shortlist} */ });
    const res = await fetch(`${API_BASE}cardmaker_ai_distractors`, { ... });
    if (myToken !== _fourChoiceAiRunToken) return;
    const data = await res.json();
    if (!data || !data.ok) return; // 以降のバッチも見込みが薄いので打ち切る
    (data.questions || []).forEach(q => {
      // studyChoicesMap の該当カードを、AIが選んだ誤答で作り直す
      if (q.i === studyIdx && !studyChoiceAnswered) renderStudyChoices(studyChoicesMap.get(key));
    });
  }
}
```
- サーバー側APIは[../../1I勉強会bot解説/14_FlaskAPI_CardMaker/03_AI誤答強化API.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/03_AI誤答強化API.md)の`/cardmaker_ai_distractors`（みんなでクイズのAI強化と同じヘルパーを再利用した汎用エンドポイント）。
- **（2026/08/26追記）記述系カードをAI問い合わせの先頭に回す**：`isDescriptiveAnswerText()`は、解答が「単語1つ」ではなく「説明文っぽい」（20文字以上・句読点を含む・スペース区切りを含む）かどうかの簡易判定。綴り類似度＋文字数の近さだけの即席4択は、単語同士ならそれなりに機能するが、記述式の解答（文章）だと綴りが近くても意味は無関係、ということが起きやすく「消去法で一目で分かる誤答」が混ざりやすい。AIによる強化の価値がより大きいこの手のカードを、`entries.sort()`で問い合わせ順の先頭へ回す（安定ソートなので、記述系同士・単語系同士それぞれの中では元の出題順を維持する）。
- **（2026/08/26追記）5問→3問→「最初の1問だけ1問ずつ」に段階的に分割**：CPU動作のローカルAI（`qwen2.5-coder:7b`）は1バッチの応答時間がバッチサイズにほぼ比例するため、バッチを小さくするほど先頭付近のカード（＝記述系優先で並べ替え済み）に早く結果が届く。「もっと早く4択に反映してほしい」という要望を受けて段階的に縮小し、最終的には**最初の1件だけバッチサイズ1**（一番乗りの改善結果を最速で届ける）、**2件目以降は3件ずつ**（総リクエスト数と速度のバランス）という構成にした。
- **実測でわかった限界**：記述式（長文）の解答・候補10件規模という現実的な条件で実測したところ、1問だけの問い合わせでも約35秒かかった（プロンプトのトークン数が解答の長さに比例して増えるため）。CPUのみ・GPU無しという環境の制約によるもので、バッチサイズの調整だけでは超えられない下限にほぼ到達している。「もっと速い軽量モデル（`qwen2.5-coder:3b`）に替える」案も検証したが、7bより**遅く**（57秒 vs 28.6秒）、しかも候補をほぼ丸ごと返すだけで実質的に選別していないという結果になり、見送った（詳細は[03_AI誤答強化API.md](../../1I勉強会bot解説/14_FlaskAPI_CardMaker/03_AI誤答強化API.md)参照）。
- **（2026/08/26追記）今表示中のカードはその場で差し替える**：以前は`studyChoicesMap`を更新するだけで、実際の描画は次に`renderStudyCard()`が呼ばれるとき（＝カードを送ったり戻ったりしたとき）まで反映されなかった。プレイヤーがまさに見ている最中のカード（`q.i === studyIdx`）が、まだ回答していない（`!studyChoiceAnswered`）状態でAI応答が届いた場合は、`renderStudyChoices()`をその場で呼び直し、選択肢をすぐに差し替える。既に回答済みのカードは触らない（正誤判定の結果が後から変わって見えると混乱するため）。
- **`_fourChoiceAiRunToken`によるキャンセル**：`setupFourChoiceIfNeeded()`が呼ばれるたび（＝学習をやり直す・別のデッキを始める等）にインクリメントされる。バッチのループ中、送信前後で自分の`myToken`が最新かどうかを確認し、古い学習セッションのために動いていたループはそこで静かに終了する（新しいセッションの`studyChoicesMap`に、前のセッションのAI応答が紛れ込むのを防ぐ）。
- **失敗時は静かに諦める**：通信エラー・`ai_unavailable`（Ollama未設定）・`ai_failed`のいずれでも、例外を投げたりアラートを出したりせず、それ以降のバッチも打ち切ってそのまま返る。既に`setupFourChoiceIfNeeded()`で組み立てた綴り類似度ベースの4択がそのまま使われ続けるため、学習自体は何の影響も受けない。
- **未ログインなら最初から呼ばない**：CardMaker自体がページ全体でログイン必須だが、念のためのガード。

### 9-4. 表示・採点：`renderStudyCard`の分岐と`currentStudyChoiceEntry`（2026/08/27に一般化）

★ 選択肢の出所が「AIが組み立てた4択（`studyChoicesMap`、単一正解のみ）」1種類だけだった頃は`studyFourChoice ? studyChoicesMap.get(cardKey(c)) : null`で済んでいたが、クイズ過去問/多肢選択デッキ統合（3.0節）で「カード自身が最初から持っている`choices`/`correct_indices`（複数正解もありうる）」という2つ目の出所が増えたため、`currentStudyChoiceEntry(card)`という共通ヘルパーに一般化された。戻り値の形も`{choices, correctIndex}`（単数）から`{choices, correctIndices}`（配列）に変わっている。

```js
function currentStudyChoiceEntry(card) {
  if (!card) return null;
  if (studyIsQuizDeck) {
    if (!Array.isArray(card.choices) || card.choices.length < CHOICE_MIN) return null;
    const correctIndices = Array.isArray(card.correct_indices) ? card.correct_indices
      : (typeof card.correct_index === 'number' ? [card.correct_index] : []);
    if (!correctIndices.length) return null;
    return { choices: card.choices, correctIndices };
  }
  if (!studyFourChoice) return null;
  const e = studyChoicesMap.get(cardKey(card));
  return e ? { choices: e.choices, correctIndices: [e.correctIndex] } : null;
}
```
- `studyIsQuizDeck`と`studyFourChoice`は同じセッション内で同時にtrueになることは無い（`isQuizPlayDeck`な単一デッキだけが`studyIsQuizDeck`になり、その場合`openPlayMode`が反転・自動採点トグル自体を隠している＝3.0節）ため、この`if`/`if`は実質`if/else if`と同じ意味になる。

```js
const choiceEntry = currentStudyChoiceEntry(c);
answerInputWrap.style.display = choiceEntry ? 'none' : '';
choiceWrap.style.display = choiceEntry ? '' : 'none';
document.getElementById('reveal-answer-btn').style.display = choiceEntry ? 'none' : '';
if (choiceEntry) renderStudyChoices(choiceEntry);
```
`choiceEntry`が無ければ（選択式にできないカードであれば）今まで通りの解答入力欄になる。ある場合は選択肢欄（`study-choice-wrap`）を表示し、「答えを見る」ボタンだけを隠す（`study-reveal-bar`自体は「← 前へ」ボタンのぶんだけ表示したままにする）——ここまではクイズ過去問デッキでも通常デッキの4択サブモードでも完全に同じコードパスで、分岐は`currentStudyChoiceEntry`の中だけに閉じている。

```js
function renderStudyChoices(entry) {
  studyChoiceAnswered = false;
  studyQuizSelected = new Set();
  const isMulti = entry.correctIndices.length > 1;
  const el = document.getElementById('study-choices');
  el.innerHTML = entry.choices.map((c, i) => `
    <button type="button" class="qp-choice-btn" onclick="${isMulti ? `toggleStudyChoiceMulti(${i})` : `answerStudyChoice(${i})`}">
      <b>${CHOICE_LETTERS[i]}.</b> <span id="study-choice-text-${i}"></span>
    </button>`).join('');
  entry.choices.forEach((c, i) => setMathText(document.getElementById(`study-choice-text-${i}`), c));
  document.getElementById('study-choice-submit-wrap').style.display = isMulti ? '' : 'none';
}
```
- **複数正解モード（`isMulti`）への対応が新設された**。以前（AI4択専用だった頃）は常に単一正解だったため無かった分岐。正解が2個以上のカード（クイズ過去問・多肢選択デッキだけで起こりうる）は、ボタンを押すたびに選択をON/OFFする`toggleStudyChoiceMulti(idx)`（旧`Cardmaker-quizplay.js`の`toggleQuizPlayMultiChoice`と同じ考え方）にし、選び終えたら`study-choice-submit-wrap`内の「この内容で回答する」ボタン（`submitStudyChoiceMulti()`）で確定する。単一正解のカードは従来通り、ボタンを押した瞬間に`answerStudyChoice(idx)`で即採点する。

```js
function answerStudyChoice(idx) {
  if (studyChoiceAnswered) return;
  const card = studyCards[studyIdx];
  const entry = card && currentStudyChoiceEntry(card);
  if (!entry) return;
  const isCorrect = entry.correctIndices.includes(idx);
  finishStudyChoiceAnswer(card, entry, isCorrect, new Set([idx]));
}

function submitStudyChoiceMulti() {
  if (studyChoiceAnswered || studyQuizSelected.size === 0) return;
  const card = studyCards[studyIdx];
  const entry = card && currentStudyChoiceEntry(card);
  if (!entry) return;
  const correctSet = new Set(entry.correctIndices);
  const isCorrect = correctSet.size === studyQuizSelected.size && [...correctSet].every(i => studyQuizSelected.has(i));
  finishStudyChoiceAnswer(card, entry, isCorrect, studyQuizSelected);
}

function finishStudyChoiceAnswer(card, entry, isCorrect, selectedSet) {
  studyChoiceAnswered = true;
  // 答案パネル表示・○×判定表示・選択肢ボタンの色分け（qp-correct/qp-wrong/qp-dim）は
  // 旧一人用選択式クイズ（answerQuizPlay）と同じ考え方。userAnswerEl.textContentは
  // 意図的に空のまま（「あなたの解答：〇〇」は出さない、下の教訓参照）。
  if (studyIsQuizDeck) {
    studyQuizAnswers.set(cardKey(card), isCorrect); // ★ 素点ではなくMapで最新の正誤を持つ（9-7節）
    saveStudyProgress();
  }
  autoMarkUnsureForCard(card, isCorrect);
  updateUnsureBtn();
}
```
`answerStudyChoice`（単一正解）・`submitStudyChoiceMulti`（複数正解）はどちらも判定結果を`finishStudyChoiceAnswer`に渡すだけの薄い関数になった。選んだ瞬間に採点まで行う（クイズと同じ「選ぶ＝回答確定」の1ステップ）。間違えたら自動で「わからない」にマークする処理は、通常の`gradeCurrentAnswer()`と共通の`autoMarkUnsureForCard()`を引き続き使っている。
- ★ 教訓（2026/08/27）：`finishStudyChoiceAnswer`は元々`grade-user-answer`要素（テキスト入力の自動採点モードで使う「あなたの解答：〇〇」表示、`gradeCurrentAnswer()`と共有）に選んだ選択肢を書き出していたが、選んだ選択肢はボタン自体の色分け（正解=緑・選んだ誤答=赤）で既に示されており冗長、というユーザーの指摘で撤去した。ただし単に行を消すのではなく`userAnswerEl.textContent = ''`と明示的に空にしている。理由は、同じ学習セッション内でこのカードの前にテキスト入力の自動採点モードのカードがあった場合、`grade-user-answer`に前のカードの「あなたの解答：〇〇」がそのまま残っており、何もしないと（`result.style.display='flex'`で同じコンテナごと再表示される際に）古いテキストが一瞬見えてしまうため。

### 9-5. 「続きから」再開への対応

通常デッキ（AI4択サブモード）は`saveStudyProgress()`が`fourChoice: studyFourChoice`も保存し、`startStudyMode('resume')`側で`studyFourChoice = studyAutoGrade && !!saved.fourChoice`として復元する。ただし`studyChoicesMap`自体（実際の選択肢の中身）は保存されず、再開のたびに`setupFourChoiceIfNeeded()`で新しく組み立て直す（＝再開すると誤答の顔ぶれは毎回変わる。正解・出題順は`saved.order`から復元されるため変わらない）。

クイズ過去問/多肢選択デッキは`choices`/`correct_indices`自体がカードに入っているため、この「誤答を組み立て直す」手間自体が無い。代わりに保存・復元するのは各問題の正誤（`studyQuizAnswers`、9-7節）で、こちらは`saveStudyProgress()`の`quizAnswers`フィールドに乗る。

### 9-6. コードレビューで見つかった細かい修正（2026/08/26）

- **`scheduleFourChoiceAiEnhancement`のトークン再チェック漏れ**：`res.json()`も`await`を挟む処理のため、その間に学習がやり直された（`setupFourChoiceIfNeeded()`が呼ばれ`_fourChoiceAiRunToken`が進んだ）可能性がある。`fetch`直後だけでなく`res.json()`の直後でも`if (myToken !== _fourChoiceAiRunToken) return;`を確認するようにした（無いと、やり直し後の新しい`studyChoicesMap`へ古いセッションのAI応答が紛れ込むことがあった）。
- **`applyServerChoiceCaches`の並列化**：デッキごとに`for...of`で直列に`await fetch`していたため、フォルダ再生でデッキ数が多いと最悪「デッキ数×5秒」待つ設計になっていた（1デッキなら数百ms程度、というこの節の説明と矛盾する動き方）。`Promise.all`で全デッキぶんを並行取得するよう修正し、待ち時間を「一番遅い1件ぶん」に収めた。
- **`buildChoiceEntry`のソート効率化**：`[...pool].sort((a, b) => _distractorScore(correct, b) - _distractorScore(correct, a))`は比較のたびにスコアを再計算する（`O(n log n)`回呼ばれる）ため、候補が多い（最大40件）デッキほど無駄が大きい。各候補のスコアを先に1回だけ計算してから並べ替える（decorate-sort-undecorate）方式に変更。学習開始前に同期的に呼ばれる関数なので、体感の学習開始速度に直接影響する。
- **選択肢欄の防御的クリア**：4択にできないカードへ切り替わったとき、`#study-choices`のDOM（前のカードの選択肢ボタン、`onclick`が前のカードのインデックスに紐づいたまま）を残さずクリアするようにした。現状`choiceWrap`が`display:none`で隠すため実害は無いが、将来この欄を`renderStudyChoices`を経由せず再表示する変更が入った場合に、古いカードの選択肢が誤って押せてしまう事故を防ぐ。

### 9-7. クイズ過去問デッキの完走：スコア・ランキング・「← 前へ」との整合性（2026/08/27）

3.0節の通り専用画面を廃止したため、デッキ完走後のスコア表示・サーバーへの送信・ランキング取得も`renderStudyCard`の完了分岐（`studyIdx >= studyCards.length`）へ統合されている。

```js
if (studyIdx >= studyCards.length) {
  // …略（進捗クリア・完了記録・renderInProgressUIは通常デッキと共通）…
  const rankEl = document.getElementById('study-done-rank');
  document.getElementById('study-done-leaderboard').innerHTML = '';
  if (studyIsQuizDeck) {
    const scoreCount = [...studyQuizAnswers.values()].filter(Boolean).length;
    document.getElementById('study-done-sub').textContent = `${scoreCount} / ${studyCards.length} 問正解！`;
    rankEl.style.display = '';
    submitQuizArchiveScoreForStudy(studyDeckId, scoreCount, studyCards.length);
  } else {
    document.getElementById('study-done-sub').textContent = `全 ${studyCards.length} 問完了！`;
    rankEl.style.display = 'none';
    rankEl.textContent = '';
  }
  return;
}
```
- **スコアは加算式のカウンタではなく`studyQuizAnswers`（cardKey→正誤のMap）から都度数え直す**設計になっている。理由は「← 前へ」ボタン——クイズ過去問デッキでも通常デッキと全く同じ`study-nav`をそのまま使っているため、答えた後のカードに何度でも戻れる——との整合性。もし単純に「正解したら+1」というカウンタにしていた場合、同じカードに「← 前へ」で戻って答え直すたびに二重・三重に加点されてしまう。`finishStudyChoiceAnswer`（9-4節）は毎回`studyQuizAnswers.set(cardKey(card), isCorrect)`で**同じカードのキーを上書き**するだけなので、答え直しても最新の判定1回分としてしか数えられない。
- `study-done-sub`（通常デッキでは「全N問完了！」の固定文言）を、クイズ過去問デッキでは「◯ / N 問正解！」というスコア表示として流用している。専用の別要素を新設せず、既存の要素を条件分岐で使い回すことで、通常デッキとの見た目の統一（「編集ボタンだけ消して、それ以外は同じ画面」）を保っている。
- `study-done-rank`・`study-done-leaderboard`（新設要素。`study-done`内、`study-done-sub`の直後）は、クイズ過去問デッキのときだけ表示する。`submitQuizArchiveScoreForStudy(deckId, score, total)`は`Cardmaker-quizplay.js`（遅延読み込みチャンク。詳細は[11_遅延読み込みチャンク_一覧表示とクイズ再生.md](11_遅延読み込みチャンク_一覧表示とクイズ再生.md)）に定義されており、この2つのidへ直接描画する。呼び出しはawaitせず「投げっぱなし」（結果を待たずに完了画面自体はすぐ表示する）。
- `shuffleStudy()`（「もう一度」ボタン、およびトップバーのシャッフルボタン）も、`studyIsQuizDeck`なら`studyQuizAnswers`を空の`Map`にリセットするよう変更した。シャッフル＝`studyIdx`を0に戻す「やり直し」なので、正誤記録だけ古いまま残ると、シャッフル後にまだ答えていないカードの分まで前回の判定がスコアに混ざってしまう。
- キーボードショートカット（8節）のスペースキー判定`inChoiceMode`も、`studyFourChoice && studyChoicesMap.has(...)`から`!!currentStudyChoiceEntry(curCard)`に一般化されている。
- ★ 教訓：AI4択のバックグラウンド強化（9-3節、`scheduleFourChoiceAiEnhancement`）が表示中のカードの選択肢をその場で差し替える箇所（`renderStudyChoices(studyChoicesMap.get(key))`）は、`renderStudyChoices`が受け取る形が`{choices, correctIndex}`から`{choices, correctIndices}`（9-4節）に変わったことで見落としがちなバグになっていた。`renderStudyChoices(currentStudyChoiceEntry(card))`に直して解消した——**関数の入力の形を変えるリファクタでは、その関数を呼んでいる箇所を全数`grep`で洗い出す**という、このプロジェクトで何度も出てきた教訓が再び当てはまる例。

---

続きは[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で、画像の圧縮・回転補正、モーダルの開閉、そして「遅延読み込みチャンク」の仕組み本体を解説します。
