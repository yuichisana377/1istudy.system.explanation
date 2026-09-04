# Cardmaker.js その2：デッキ一覧の描画とフォルダ操作（812〜1518行）

[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の続きです。

★ 追記（2026/08/24）：01章末尾（右上のアカウント表示をタップ可能にする変更）で18行増えたため、この章（02）以降、`Cardmaker.js`本体を対象とする解説（02〜09章。10・11章は別ファイルの`Cardmaker-*.js`なので対象外）の行番号引用は、実際のコードより18行分ズレている（未修正）。関数名・変数名で検索する方が確実。

---

## 1. デッキ一覧の描画：`renderDeckListUI()`（812〜1016行）

ホーム画面（フォルダ・デッキ一覧）を実際に画面へ描画する、CardMakerの中でも中心的な関数です。長いので段階を追って見ていきます。

### 1.1 冒頭のガード（818〜827行）
```js
if (cmListDragActive) return;
if (currentFolderId && !folders.find(f => f.id === currentFolderId)) currentFolderId = null;
renderBreadcrumb();
renderInProgressUI();
const skeleton = document.getElementById('deck-skeleton');
if (skeleton) skeleton.style.display = 'none';
```
- ドラッグ並び替え中（`cmListDragActive`）なら何もせず終了（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)で説明した理由）。
- 今見ているフォルダが（他の端末での削除などにより）もう存在しなければ、ホーム（`null`）に強制的に戻す。
- パンくずリスト（今どのフォルダの中にいるかの表示）とプレイ中セクションを更新。
- JS読み込み前から見えている「読み込み中…」風の**スケルトン**（骨組みだけの仮表示、実データが来る前に「なんとなくこんな形」というプレースホルダーを見せておく手法）を、初回描画が終わった時点で隠す。

### 1.2 空っぽのときの表示（829〜841行）
```js
  const grid  = document.getElementById('deck-grid');
  const empty = document.getElementById('deck-list-empty');

  const childFolders = folderChildren(currentFolderId);
  const childDecks   = decks.filter(d => (d.folderId || null) === currentFolderId);

  if (!childFolders.length && !childDecks.length) {
    grid.style.display='none'; empty.style.display='block';
    document.getElementById('deck-list-empty-text').textContent =
      currentFolderId ? 'このフォルダにはまだ何もありません' : 'まだデッキがありません';
    if (pickMode) updatePickBar();
    return;
  }
  empty.style.display='none'; grid.style.display='flex';
```
今のフォルダの直下にある子フォルダ・子デッキを集め、両方とも0件なら「まだデッキがありません」（フォルダの中なら「このフォルダにはまだ何もありません」）を表示して終了します。

### 1.3 フォルダのカードを組み立てる（844〜884行）
```js
  const folderItems = childFolders.map(f => {
  const cnt = countDecksRecursive(f.id);
  const totalCards = countCardsRecursive(f.id);
  const unsureCount = countUnsureRecursive(f.id);              // ★ 追加
  const isLoadingThisFolder = loadingFolderIds.has(f.id);
  const folderPlayDisabled = totalCards === 0 || isLoadingThisFolder;
  const folderUnsureBadge = unsureCount > 0                     // ★ 追加
    ? `<span class="unsure-badge">${Icons.cmHtml('bookmark', {size:13})} ${unsureCount}</span>` : '';

  // ★ 追加：クイズ用デッキ選択モードでは、通常のプレイ/メニューボタンの代わりに
  //   チェックボックスを出す。フォルダ本体をタップすれば中を見に行けるのはそのまま。
  if (pickMode) {
    const eligibleCount = collectDecksInFolder(f.id).filter(isDeckQuizPickable).length;
    const impliedByAncestor = pickFolderAncestorSelected(f.parentId || null);
    const disabled = eligibleCount === 0 || impliedByAncestor;
    const checked = pickedFolderIds.has(f.id) || impliedByAncestor;
    const cbClass = disabled ? 'pick-checkbox disabled' : checked
      ? (pickedFolderIds.has(f.id) ? 'pick-checkbox checked' : 'pick-checkbox implied') : 'pick-checkbox';
    return { key: `folder:${f.id}`, html: `
  <div class="deck-card folder-card${disabled && !checked ? ' pick-disabled' : ''}" data-key="folder:${f.id}" onclick="openFolder('${f.id}')">
    <div class="${cbClass}" onclick="togglePickFolder('${f.id}', event)">${checked ? Icons.html('check', {size:12}) : ''}</div>
    <div class="deck-card-info">
      <div class="deck-card-title">${Icons.cmHtml('folder', {size:15})} ${esc(f.name)}</div>
      <div class="deck-card-meta">${eligibleCount > 0 ? `${eligibleCount} デッキが対象` : '対象にできるデッキがありません'}${folderUnsureBadge}</div>
    </div>
  </div>` };
  }

  // ★ 追加：フォルダの説明（設定されている場合だけ、1行に省略して表示）
  const folderDescLine = f.description
    ? `<div class="deck-card-desc">${esc(f.description)}</div>` : '';
  return { key: `folder:${f.id}`, html: `
  <div class="deck-card folder-card" data-key="folder:${f.id}" onclick="openFolder('${f.id}')">
    <div class="deck-card-info">
      <div class="deck-card-title">${Icons.cmHtml('folder', {size:15})} ${esc(f.name)}</div>
      <div class="deck-card-meta">${cnt} デッキ・${totalCards} 問${folderUnsureBadge}</div>
      ${folderDescLine}
    </div>
    <div class="deck-card-actions">
      <button class="btn btn-blue btn-sm" onclick="event.stopPropagation();openFolderPlayMode('${f.id}')"
        ${folderPlayDisabled?'disabled':''}>${isLoadingThisFolder ? '読み込み中…' : '▶ プレイ'}</button>
      <button class="icon-btn" onclick="event.stopPropagation();openFolderMenu('${f.id}')" title="メニュー">${Icons.html('edit', {size:14})}</button>
    </div>
  </div>` };
});
```
- `countDecksRecursive`/`countCardsRecursive`/`countUnsureRecursive`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)参照）で、そのフォルダ配下の合計デッキ数・カード数・「わからない」件数を計算。
- **クイズ用デッキ選択モード（`pickMode`）中は特別な見た目**：通常の「▶プレイ」「メニュー」ボタンの代わりに、チェックボックス風の見た目でフォルダ全体を選択できるようにします。選択対象にできるデッキが1件も無いフォルダはグレーアウト（`disabled`）。
- 通常モードでは、フォルダ名・件数に加えて「▶プレイ」ボタン（フォルダ内を一括で学習）と「メニュー」ボタン（名前変更・移動・削除）を表示します。
- それぞれのフォルダに`key: 'folder:フォルダID'`という並び順管理用のキーを持たせています。
- ★ 追加：`f.description`（フォルダの説明欄、任意）が設定されていれば、`.deck-card-desc`として件数の下に1行だけ（`text-overflow: ellipsis`で省略）表示します。`esc()`を通してから`innerHTML`テンプレートに入れているのは、他のユーザー入力（フォルダ名等）と同じXSS対策です。同じ説明は、フォルダを実際に開いているときはパンくずリストの下（`renderBreadcrumb()`、後述）にも全文表示されます。編集はフォルダ名の入力モーダル（5.3節）から行います。
- **★ 2026/09/04 追記**：`folderUnsureBadge`が件数バッジだけでなく`onclick="event.stopPropagation();showUnsureRatio(${unsureCount}, ${totalCards})"`を持つようになった。一覧では件数（分子）しか出せないため、タップすると`showUnsureRatio()`（新設）が`showCmAlert`で「◯ / ◯ 問」（何問中何問がわからないか）をダイアログ表示する。`event.stopPropagation()`はフォルダ本体の`onclick="openFolder(...)"`へタップが伝播してフォルダが開いてしまわないようにするため。

### 1.4 デッキのカードを組み立てる（885〜1001行）

まず表示順の初期値を決めます：
```js
  // ★ 非公開・公開のグループ位置はそのまま、各グループ内だけ新しい順（下が古い）に反転
  //   （※ ユーザーが手で並び替えた後は、下の applySavedListOrder() がこの初期順を上書きする）
  const unpublished = childDecks.filter(d => !d.filename).slice().reverse();
  const published    = childDecks.filter(d =>  d.filename).slice().reverse();
  const orderedDecks = [...unpublished, ...published];
```
- `d.filename`が無い＝まだサーバーに公開されていない下書きデッキ。それらを先に、公開済みデッキをあとに並べ、それぞれのグループ内は新しい順（`.reverse()`で配列の並びを逆転）にしています。ただしコメントにある通り、これはあくまで「ユーザーが手で並び替えていない場合の初期順」で、保存済みの並び順があれば後段の`applySavedListOrder`がこれを上書きします。

デッキ1件ごとに、かなり細かい表示の出し分けロジックがあります：
```js
  const deckItems = orderedDecks.map(d => {
    // ★ カード本体を未読み込みのデッキ（公開デッキで cardsLoaded=false）は
    //   d.cards が空のままなので、「わからない」バッジは読み込み後にしか出せない。
    //   ここでは読み込み済みの場合だけ計算する。
    let unsureBadge = '';
    if (d.cardsLoaded !== false) {
      const unsureSet   = getUnsureSet(d.id);
      const unsureCount = d.cards.filter(c => unsureSet.has(cardKey(c))).length;
      unsureBadge = unsureCount > 0 ? `<span class="unsure-badge">${Icons.cmHtml('bookmark', {size:13})} ${unsureCount}</span>` : '';
    }
    // ★ 問題数は常にサーバー側の count（軽量メタ情報）を優先して表示する。
    //   d.cards はカード本体が未読み込みの間は空配列なので、そちらを見てはいけない。
    //   （pubBadge の判定でも使うため、先に計算しておく）
    const questionCount = d.filename ? (d.count ?? d.cards.length) : d.cards.length;
    // ★ 公開状態バッジ：作成中／非公開／公開済み／未完成 のいずれか1つだけを表示する。
    //   （以前は「公開済み」と「未完成」を別々のバッジとして両方表示していたが、
    //   分かりにくいので同じ場所に1つだけ出すよう統合した）
    //   ★ 修正：以前は「サーバー登録済み・カード0枚」の場合だけを「作成中」と判定していたため、
    //     まだ一度も「公開して保存」（＝完成／未完成の選択）を経ていないデッキでも、
    //     ただの「保存」ボタンでカードを追加しただけで questionCount>0 になった途端に
    //     「未完成」バッジへ変わってしまっていた（＝「保存しただけなのに未完成と表示される」不具合）。
    //     ここでは d.notYetPublished（＝一度も明示的な「公開して保存」を経ていないかどうか）を
    //     カード枚数に関係なく最優先で判定し、以下の3段階に整理する。
    //     ・「保存」ボタンを押しただけ（公開フローを一度も経ていない）　　　　→ 常に「作成中」
    //     ・「公開して保存」で「未完成として公開する」を選んだことがある　　　→ 「未完成」
    //     ・「公開して保存」で「完成として公開する」を選んだ（その後の状態） → 「公開済み」
    const pubBadge = !d.filename
      ? (d.planPublish !== false
          ? `<span class="pub-badge inprogress">${Icons.html('dot', {size:13})} 作成中</span>`
          : `<span class="pub-badge local">${Icons.html('dot', {size:13})} 非公開</span>`)
      : (d.notYetPublished !== false)
        ? `<span class="pub-badge inprogress">${Icons.html('dot', {size:13})} 作成中${d.published_by ? `（${esc(d.published_by)}）` : ''}</span>`
        : d.incomplete
          ? `<span class="pub-badge draft">${Icons.html('dot', {size:13})} 未完成${d.published_by ? `（${esc(d.published_by)}）` : ''}</span>`
          : `<span class="pub-badge published">${Icons.html('dot', {size:13})} 公開済み${d.published_by ? `（${esc(d.published_by)}）` : ''}</span>`;
    // ★ クイズ過去問デッキだと分かるようにバッジを付ける（プレイ時の挙動が通常の
    //   フラッシュカードと違う＝一人用選択式モードになるため。編集もできない）。
    //   ★ 修正（2026/08/21）：以前はフォルダの位置（isDeckInFolderScope）で判定して
    //   いたが、他のフォルダへ移動できるようにしたのに合わせ、デッキ自身が持つ
    //   quizArchiveフラグ（フォルダを移動しても消えない）で判定するよう変更した。
    //   バッジの文言も「過去問」→「クイズ過去問」に変更（フォルダ名と揃えた）。
    const quizArchiveBadge = d.quizArchive
      ? `<span class="pub-badge archive">${Icons.html('dot', {size:13})} クイズ過去問</span>` : '';
    // ★ 追加：多肢選択デッキ（choiceMode有り）にも、同じ理由で分かるようにバッジを付ける
    //   （「クイズ過去問」フォルダの中でなくてもプレイ時は一人用選択式モードになるため）
    //   ★ quizArchiveBadge と意味が重複するため、そちらが出る場合はこちらは出さない。
    //   ★ 単一/複数は問題ごとに違いうるためデッキ単位では区別せず、常に「選択式」とだけ表示する。
    const choiceModeBadge = (!quizArchiveBadge && d.choiceMode)
      ? `<span class="pub-badge archive">${Icons.html('dot', {size:13})} 選択式</span>` : '';
    // ★ カード本体が未読み込みの間、プレイ／編集ボタンを押した瞬間に
    //   ネットワーク取得が走ることをユーザーに知らせるためのローディング表示。
    const isLoadingThis = loadingDeckIds.has(d.id);
    // ★ 追加：「作成中」（＝まだ一度も公開して保存していない）状態のデッキはプレイできないようにする。
    //   編集（openEditDeck / openDeckMenu）はこのフラグを見ないので、作成中でも引き続き編集は可能。
    const isInProgress = !d.filename ? (d.planPublish !== false) : (d.notYetPublished !== false);
    const playDisabled = questionCount === 0 || isLoadingThis || isInProgress;
    // ★ 科目名をタイトルの上に小さく表示する。表示名側に重複しないよう、
    //   デッキ名の先頭に「科目名 」が含まれる場合はそれを取り除いて表示する。
    const subjectLabel = d.subject
      ? `<div class="deck-card-subject">${esc(d.subject)}</div>` : '';
    const displayName = (d.subject && d.name.startsWith(d.subject + ' '))
      ? d.name.slice(d.subject.length + 1) : d.name;
    // ★ 並び順のキー：公開済みデッキは全員が同じ filename を持つので、それを共有キーにする。
    //   未公開（自分だけの下書き）デッキは他人には見えないデータなので、他の端末とは
    //   絶対に一致しないローカル専用キー（localdeck:）にし、サーバーには送らない。
    const orderKey = d.filename ? `deck:${d.filename}` : `localdeck:${d.id}`;

    // ★ 追加：クイズ用デッキ選択モードでは、プレイ/メニューボタンの代わりに
    //   カード全体をタップして選択できるチェックボックスUIにする。
    if (pickMode) {
      const eligible = isDeckQuizPickable(d);
      const impliedByAncestor = pickFolderAncestorSelected(d.folderId || null);
      const disabled = !eligible || impliedByAncestor;
      const checked = pickedDeckIds.has(d.id) || impliedByAncestor;
      const cbClass = disabled ? 'pick-checkbox disabled' : (checked ? 'pick-checkbox checked' : 'pick-checkbox');
      const ineligibleNote = !eligible
        ? `<div class="deck-card-note-ineligible">クイズには使えません（非公開・作成中のデッキ）</div>` : '';
      return { key: orderKey, html: `
    <div class="deck-card${disabled && !checked ? ' pick-disabled' : ''}" data-key="${orderKey}" onclick="togglePickDeck('${d.id}', event)">
      <div class="${cbClass}">${checked ? Icons.html('check', {size:12}) : ''}</div>
      <div class="deck-card-info">
        ${subjectLabel}
        <div class="deck-card-title">${esc(displayName)}</div>
        <div class="deck-card-meta">
          ${questionCount} 問
          ${pubBadge}
          ${quizArchiveBadge}
          ${choiceModeBadge}
          ${unsureBadge}
        </div>
        ${ineligibleNote}
      </div>
    </div>` };
    }

    // ★ 追加：デッキの説明（設定されている場合だけ、1行に省略して表示）
    const deckDescLine = d.description
      ? `<div class="deck-card-desc">${esc(d.description)}</div>` : '';
    return { key: orderKey, html: `
    <div class="deck-card" data-key="${orderKey}">
      <div class="deck-card-info">
        ${subjectLabel}
        <div class="deck-card-title">${esc(displayName)}</div>
        <div class="deck-card-meta">
          ${questionCount} 問
          ${pubBadge}
          ${quizArchiveBadge}
          ${choiceModeBadge}
          ${unsureBadge}
        </div>
        ${deckDescLine}
      </div>
      <div class="deck-card-actions">
        <button class="btn btn-blue btn-sm" onclick="openPlayMode('${d.id}')"
          ${playDisabled?'disabled':''}>${isLoadingThis ? '読み込み中…' : '▶ プレイ'}</button>
        <button class="icon-btn" onclick="openDeckMenu('${d.id}')" title="メニュー" ${isLoadingThis?'disabled':''}>${Icons.html('edit', {size:14})}</button>
      </div>
    </div>` };
  });
```
- **「わからない」バッジ**：カード本体が読み込み済み（`cardsLoaded !== false`）のときだけ計算（未読み込みのデッキは`cards`が空配列のままなので計算できないため）。
- **問題数の表示**：`d.filename ? (d.count ?? d.cards.length) : d.cards.length`。公開済みデッキはサーバー側の軽量な`count`情報（カード本体を読み込まなくても分かる件数）を優先し、下書きデッキは手元の`cards`配列の長さをそのまま使います。
- **公開状態バッジ（`pubBadge`）**：「作成中」「非公開」「未完成」「公開済み」の4種類のうちどれか1つだけを表示します。コメントに詳しい経緯が書かれていて、以前は「サーバー登録済み・カード0枚」だけを「作成中」と判定していたため、公開フローを経ずにただ「保存」しただけのデッキが、カードを1枚追加した途端に誤って「未完成」表示に変わってしまうバグがあったそうです。今は`d.notYetPublished`（一度も明示的な「公開して保存」を経ていないか）を最優先で見て、3段階に整理し直しています。
- **「クイズ過去問」バッジ**：`d.quizArchive`が立っていれば表示（プレイ時の挙動が普通のフラッシュカードと違う＝一人用選択式モードになるため。問題の編集もできません）。★ 修正（2026/08/21）：以前は「そのデッキが『クイズ過去問』システムフォルダの中にあるかどうか」（位置）で判定し、バッジの文言も「過去問」でしたが、そのデッキを他のフォルダへ移動できるようにしたのに合わせ、デッキ自身が持つ`quizArchive`フラグ（サーバー側の`quiz_archive`、フォルダを移動しても消えない）で判定するよう変更し、文言も「クイズ過去問」に変更しました。
- **「選択式」バッジ**：多肢選択デッキ（`d.choiceMode`）であれば表示。ただし「クイズ過去問」バッジと意味が重複するため、そちらが出ているときはこちらは出しません。
- **プレイボタンの無効化条件**：問題数0、読み込み中、または「作成中」（＝一度も公開フローを経ていない）状態のいずれかに該当すればプレイ不可（`playDisabled`）。**編集はこのフラグを見ないので、作成中でも編集は引き続き可能**、という注記もあります。
- **科目名の重複表示回避**：デッキ名の先頭に科目名がそのまま含まれている場合（例：「数学 二次関数」）、表示名からその部分を取り除いて、科目名は別枠（`subjectLabel`）で表示するようにしています。
- ★ 追加：`d.description`（デッキの説明欄、任意）が設定されていれば、フォルダカードと同じ`.deck-card-desc`（1行に省略表示）として`.deck-card-meta`の下に表示します。編集はデッキメニューの「デッキ名を変更する」から開く`modal-rename`モーダルで行います（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の「デッキ名・説明の変更」参照）。
- **並び順キー（`orderKey`）**：公開済みデッキは全員が同じ`filename`を持つので、それを共有キー（`deck:ファイル名`）にします。未公開デッキは他人には見えないデータなので、この端末専用のキー（`localdeck:内部ID`）にし、サーバーには送りません。
- こちらもクイズ用デッキ選択モード中は、通常のボタンの代わりにチェックボックスの見た目に切り替わります。
- **★ 2026/09/04 追記（プレイ進捗バッジ）**：通常表示（`pickMode`ではない方）の`.deck-card-meta`に、`choiceModeBadge`の直後・`unsureBadge`の直前へ`studyStatusBadge`が1つ追加された。
  ```js
  const studyProgress  = loadStudyProgress(false, d.id);
  const studyCompleted = loadCompletionRecord(false, d.id);
  const studyStatusBadge = studyProgress
    ? `<span class="pub-badge study-doing">${Icons.html('dot', {size:13})} プレイ中</span>`
    : studyCompleted
      ? `<span class="pub-badge study-done">${Icons.html('dot', {size:13})} 完了</span>`
      : '';
  ```
  `loadStudyProgress`/`loadCompletionRecord`（[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の「続きから／完了記録」参照）を直接見て、続きから再開できる記録があれば青（`pub-badge study-doing`＝「プレイ中」）、無くて完了記録だけあれば緑（`pub-badge study-done`＝「完了」）を表示する（両方ある場合は「今まさに再開できる」プレイ中を優先）。ホーム画面の「プレイ中のデッキ」「プレイ済み（完了）」欄（4節）は直近1週間のものしか出さないが、こちらは一覧の各デッキに常時つくバッジなので期間で消さず、記録が残っている限りずっと表示され続ける。
- **★ 2026/09/04 追記（わからないバッジのタップ）**：`unsureBadge`のスパンに`onclick="event.stopPropagation();showUnsureRatio(${unsureCount}, ${questionCount})"`が付き、タップすると「◯ / ◯ 問」（何問中何問がわからないか）が`showCmAlert`のダイアログで表示されるようになった（フォルダ側の`folderUnsureBadge`と同じ`showUnsureRatio()`を使う。1.3節の追記も参照）。

### 1.5 並び順の適用と、重複描画の最終防御（1003〜1020行）
```js
  // ★ フォルダ・デッキを合わせ、保存済みの並び順（ユーザーがドラッグして決めた順）があれば適用する
  const combinedItems = applySavedListOrder([...folderItems, ...deckItems], currentFolderId);
  // ★ 追加：最終防御として、万一同じキー（＝同じデッキ／フォルダ）が
  //   何らかの理由で2件並んでしまっていても、ここで必ず1件だけに絞ってから描画する。
  //   （並び順マージ処理などに未知の不具合があっても、画面上の「見た目の複製」だけは常に防げるようにする）
  const seenKeys = new Set();
  const dedupedItems = combinedItems.filter(it => {
    if (seenKeys.has(it.key)) return false;
    seenKeys.add(it.key);
    return true;
  });
  grid.innerHTML = dedupedItems.map(it => it.html).join('');
  if (pickMode) updatePickBar();
}
```
- `applySavedListOrder`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)）で、ユーザーがドラッグして決めた並び順を適用。
- 最後に、同じキー（＝同じデッキ／フォルダ）を持つ項目が万一2件以上並んでいた場合でも、最初の1件だけを残して残りを除外する「最終防御」を入れています。コメントによれば、これは「並び順マージ処理などに未知の不具合があっても、画面上に同じ項目が2つ表示される、という見た目の破綻だけは常に防ぐため」の保険です。

---

## 2. パンくずリスト（1023〜1042行）

```js
function renderBreadcrumb() {
  const bar = document.getElementById('folder-breadcrumb');
  const descBox = document.getElementById('folder-desc');
  if (!currentFolderId) {
    bar.style.display = 'none'; bar.innerHTML = '';
    descBox.style.display = 'none'; descBox.textContent = '';
    return;
  }
  const chain = [];
  let cur = folders.find(f => f.id === currentFolderId);
  while (cur) { chain.unshift(cur); cur = folders.find(f => f.id === cur.parentId); }
  bar.style.display = 'flex';
  bar.innerHTML = `<span class="crumb" onclick="openFolder(null)">🏠 ホーム</span>` +
    chain.map(f => `<span class="crumb-sep">/</span><span class="crumb" onclick="openFolder('${f.id}')">${esc(f.name)}</span>`).join('');
  // ★ 追加：今開いているフォルダ自身に説明が設定されていれば表示する
  const currentFolder = folders.find(f => f.id === currentFolderId);
  if (currentFolder && currentFolder.description) {
    descBox.textContent = currentFolder.description;
    descBox.style.display = '';
  } else {
    descBox.style.display = 'none'; descBox.textContent = '';
  }
}
```
- 今のフォルダから親を順にたどって配列の先頭に追加していく（`chain.unshift`）ことで、「ホーム › 数学 › 二次関数」のような経路の並びを作ります。
- ホームにいるとき（`currentFolderId`が`null`）はパンくず自体を非表示にします。
- ★ 追加：今開いているフォルダに説明（`description`、任意）が設定されていれば、パンくずの下の`#folder-desc`に**省略せず全文**表示します（一覧のフォルダカード側は1行に省略表示、という使い分け）。`textContent`で入れているのでユーザー入力をHTMLとして解釈させない安全な組み立て方です。

```js
function folderPathLabel(folderId) {
  if (!folderId) return null;
  const chain = [];
  let cur = folders.find(f => f.id === folderId);
  while (cur) { chain.unshift(cur); cur = folders.find(f => f.id === cur.parentId); }
  return chain.map(f => f.name).join(' / ');
}
```
`folderPathLabel(folderId)`（1036〜1042行）は同じ考え方で、パンくずと同じ経路を「/」区切りの1本の文字列にして返す関数です。検索画面で「今どこを検索対象にしているか」を表示するのに使われます。

---

## 3. 単語検索画面への入口（1044〜1060行）

```js
async function openSearchScreen() {
  await loadChunkWithFeedback('search', '/Cardmaker-search.js');
  return openSearchScreen(); // ★ この時点では本物の実装に差し替わっている
}
function onSearchInput() {
  loadChunk('search', '/Cardmaker-search.js').then(() => onSearchInput());
}
```
- これは検索機能そのものの実装ではなく、「まだ検索用のファイル（`Cardmaker-search.js`）が読み込まれていないときの、一時的な代役」です。
- HTMLの`onclick="openSearchScreen()"`のようなボタンは、この関数を最初に呼び出します。この関数は`Cardmaker-search.js`を読み込み終わるまで待ってから、**もう一度自分自身と同じ名前の関数を呼び出します**。実は`Cardmaker-search.js`が読み込まれると、その中で改めて`openSearchScreen`という同じ名前の（今度は本物の）関数が定義され、こちらの仮の実装を**上書き**します。そのため2回目の呼び出しでは、すでに本物の実装が動く、という仕組みです。
- 詳しい仕組みは[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)の`loadChunk`/`loadChunkWithFeedback`の節で説明します。

---

## 4. ホーム画面の「プレイ中」「プレイ済み」セクション（1062〜1248行）

### 4.1 「プレイ中（続きから再開できる）」の一覧（1062〜1125行）
```js
function getInProgressItems(scopeFolderId) {
  const items = [];
  for (const key of Object.keys(studyDataCache.progress)) {
    let isFolder, id;
    if (key.startsWith('deck:'))        { isFolder = false; id = key.slice('deck:'.length); }
    else if (key.startsWith('folder:')) { isFolder = true;  id = key.slice('folder:'.length); }
    else continue;

    const data = loadStudyProgress(isFolder, id);
    if (!data) continue; // 壊れている・空のデータは無視

    if (isFolder) {
      const folder = folders.find(f => f.id === id);
      if (!folder) continue; // フォルダが削除済みなら無視
      if (!isFolderInFolderScope(id, scopeFolderId)) continue; // ★ 表示範囲外なら除外
      items.push({ isFolder: true, id, name: folder.name, subject: '', icon: Icons.cmHtml('folder', {size:16}),
        idx: data.idx, total: data.order.length, updatedAt: data.updatedAt || 0 });
    } else {
      const deck = decks.find(d => d.id === id);
      if (!deck) continue; // デッキが削除済みなら無視
      if (!isDeckInFolderScope(id, scopeFolderId)) continue; // ★ 表示範囲外なら除外
      // ★ デッキ一覧のカードと同じく、科目名をタイトルの上に分けて表示する
      const displayName = (deck.subject && deck.name.startsWith(deck.subject + ' '))
        ? deck.name.slice(deck.subject.length + 1) : deck.name;
      items.push({ isFolder: false, id, name: displayName, subject: deck.subject || '', icon: Icons.html('cardmaker', {size:16}),
        idx: data.idx, total: data.order.length, updatedAt: data.updatedAt || 0 });
    }
  }
  const ONE_WEEK_MS = 7 * 24 * 60 * 60 * 1000;
  const now = Date.now();
  const recentItems = items.filter(it => (now - it.updatedAt) <= ONE_WEEK_MS); // ★ 追加：直近1週間以内にプレイしたものだけに絞る
  recentItems.sort((a, b) => b.updatedAt - a.updatedAt); // 新しく学習していた順
  return recentItems;
}
```
- 学習の途中経過（`studyDataCache.progress`、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)で説明）を全部調べ、「まだ存在するデッキ/フォルダ」「表示範囲内」「直近1週間以内に触っていた」の3条件を満たすものだけを、最近触った順に並べて返します。

```js
function renderInProgressUI() {
  const section = document.getElementById('inprogress-section');
  const scroll  = document.getElementById('inprogress-scroll');
  if (section && scroll) {
    // ★ ホームでは全体、フォルダ内ではそのフォルダ配下（サブフォルダ含む）だけに絞って表示する
    const items = getInProgressItems(currentFolderId);
    if (!items.length) {
      section.style.display = 'none'; scroll.innerHTML = '';
    } else {
      section.style.display = 'block';
      scroll.innerHTML = items.map(it => {
        const pct = Math.max(0, Math.min(100, Math.round(((it.idx) / it.total) * 100)));
        return `
        <div class="inprogress-card" onclick="resumeFromHome(${it.isFolder}, '${it.id}')">
          ${it.subject ? `<div class="inprogress-subject">${esc(it.subject)}</div>` : ''}
          <div class="inprogress-title">${it.icon} ${esc(it.name)}</div>
          <div class="inprogress-meta">${it.idx + 1} / ${it.total} 問</div>
          <div class="inprogress-bar-track"><div class="inprogress-bar-fill" style="width:${pct}%"></div></div>
          <div class="inprogress-resume-btn">${Icons.cmHtml('play', {size:14})} 続きから</div>
        </div>`;
      }).join('');
    }
  }
  renderCompletedUI(); // ★ 追加：プレイ済み（完了）欄も同時に更新する
}
```
- `renderInProgressUI()`（1101〜1125行）はこれを実際にホーム画面の「プレイ中のデッキ」欄に描画する関数です。進捗バー（`〇問中△問目`をパーセントに変換）付きのカードを並べ、最後に`renderCompletedUI()`（下記）も一緒に呼び出します。
- ★ 追加（2026/08/27）：「続きから」の再生アイコンは、絵文字（▶️）から`Icons.js`の`Icons.cmHtml('play', ...)`（色付きの塗りつぶし三角形SVG）に置き換えました。サイト全体で以前に行われた「絵文字はOS/ブラウザで見た目が揃わないため自作SVGアイコンに統一する」という置換作業に、この2箇所（本欄とプレイモード選択モーダル、後述）だけ漏れていたための対応です。

### 4.2 「プレイ済み（完了）」の一覧（1127〜1183行）

```js
function getCompletedItems(scopeFolderId) {
  const items = [];
  for (const key of Object.keys(studyDataCache.completed)) {
    let isFolder, id;
    if (key.startsWith('deck:'))        { isFolder = false; id = key.slice('deck:'.length); }
    else if (key.startsWith('folder:')) { isFolder = true;  id = key.slice('folder:'.length); }
    else continue;

    const data = loadCompletionRecord(isFolder, id);
    if (!data) continue; // 壊れている・空のデータは無視

    if (isFolder) {
      const folder = folders.find(f => f.id === id);
      if (!folder) continue; // フォルダが削除済みなら無視
      if (!isFolderInFolderScope(id, scopeFolderId)) continue; // ★ 表示範囲外なら除外
      items.push({ isFolder: true, id, name: folder.name, subject: '', icon: Icons.cmHtml('folder', {size:16}),
        total: data.total, completedAt: data.completedAt });
    } else {
      const deck = decks.find(d => d.id === id);
      if (!deck) continue; // デッキが削除済みなら無視
      if (!isDeckInFolderScope(id, scopeFolderId)) continue; // ★ 表示範囲外なら除外
      // ★ デッキ一覧のカードと同じく、科目名をタイトルの上に分けて表示する
      const displayName = (deck.subject && deck.name.startsWith(deck.subject + ' '))
        ? deck.name.slice(deck.subject.length + 1) : deck.name;
      items.push({ isFolder: false, id, name: displayName, subject: deck.subject || '', icon: Icons.html('cardmaker', {size:16}),
        total: data.total, completedAt: data.completedAt });
    }
  }
  const ONE_WEEK_MS = 7 * 24 * 60 * 60 * 1000;
  const now = Date.now();
  const recentItems = items.filter(it => (now - it.completedAt) <= ONE_WEEK_MS); // ★ 直近1週間以内に完了したものだけ
  recentItems.sort((a, b) => b.completedAt - a.completedAt); // 新しく完了した順
  return recentItems;
}

function renderCompletedUI() {
  const section = document.getElementById('completed-section');
  const scroll  = document.getElementById('completed-scroll');
  if (!section || !scroll) return;

  // ★ ホームでは全体、フォルダ内ではそのフォルダ配下（サブフォルダ含む）だけに絞って表示する
  const items = getCompletedItems(currentFolderId);
  if (!items.length) { section.style.display = 'none'; scroll.innerHTML = ''; return; }

  section.style.display = 'block';
  scroll.innerHTML = items.map(it => `
    <div class="completed-card" onclick="replayFromHome(${it.isFolder}, '${it.id}')">
      ${it.subject ? `<div class="completed-subject">${esc(it.subject)}</div>` : ''}
      <div class="completed-title">${it.icon} ${esc(it.name)}</div>
      <div class="completed-meta">${Icons.html('checkCircle', {size:14})} ${it.total} 問 完了</div>
      <div class="completed-replay-btn">${Icons.html('refresh', {size:14})} もう一度プレイ</div>
    </div>`).join('');
}
```
`getCompletedItems`/`renderCompletedUI`は、上記の「プレイ中」とほぼ同じ考え方で、完了記録（`studyDataCache.completed`）を対象にした版です。「直近1週間以内に完了したもの」を新しい順に表示します。

### 4.3 ホーム画面のカードをタップしたときの挙動（1185〜1248行）
```js
async function replayFromHome(isFolder, id) {
  if (isFolder) { await openFolderPlayMode(id); } else { await openPlayMode(id); }
}
```
- 「プレイ済み」カードをタップすると、通常のプレイモード選択画面（すべて／わからないだけ、など）を開き直します。

```js
async function resumeFromHome(isFolder, id) {
  if (isFolder) {
    const folder = folders.find(f => f.id === id);
    const targetDecks = collectDecksInFolder(id)
      .filter(d => (d.filename ? (d.count ?? d.cards.length) : d.cards.length) > 0);
    if (!targetDecks.length) return;

    loadingFolderIds.add(id);
    renderDeckListUI();
    // ★ プレイ開始時は毎回サーバーの最新カードを取りに行く（force=true）。
    //   キャッシュ済み（cardsLoaded=true）のまま開くと、他の人が直した最新の
    //   修正内容がプレイ画面に反映されない＝「もう直っていたのに気づかず
    //   重複して編集してしまう」事故につながるため。
    // ★ 修正：保留中のサーバー同期を待たずに強制リロードすると、同期前の
    //   古い内容（最悪カード0枚）で上書きされてしまうため、先に待ち合わせる。
    await Promise.all(targetDecks.map(d => waitForPendingSync(d.id)));
    const results = await Promise.all(targetDecks.map(d => ensureDeckCardsLoaded(d.id, true)));
    loadingFolderIds.delete(id);
    renderDeckListUI();

    if (results.some(r => !r.ok)) {
      await showCmAlert({ title: '読み込みに失敗しました', desc: '通信環境を確認してもう一度お試しください。' });
      return;
    }
    folderPlayDecks = targetDecks;
    studyIsFolder = true;
    studyFolderId = id;
    studyDeckId = null;
  } else {
    const deck = decks.find(d => d.id === id);
    if (!deck) return;

    loadingDeckIds.add(id);
    renderDeckListUI();
    // ★ プレイ開始時は毎回サーバーの最新カードを取りに行く（force=true）。理由は上と同じ。
    // ★ 修正：保留中のサーバー同期を待たずに強制リロードすると、同期前の
    //   古い内容（最悪カード0枚）で上書きされてしまうため、先に待ち合わせる。
    await waitForPendingSync(id);
    const result = await ensureDeckCardsLoaded(id, true);
    loadingDeckIds.delete(id);
    renderDeckListUI();

    if (!result.ok) {
      await showCmAlert({ title: '読み込みに失敗しました', desc: '通信環境を確認してもう一度お試しください。' });
      return;
    }
    studyIsFolder = false;
    studyDeckId = id;
  }
  startStudyMode('resume');
}
```
- 「プレイ中」カードをタップすると、モード選択を経由せず**直接続きから**学習画面を開きます。
- 注目すべき点は、プレイを始める前に**毎回サーバーの最新カードを強制的に取り直している**ことです。コメントに理由が書かれています：もしキャッシュ済みのまま開いてしまうと、他の人が先に直した最新の修正内容が反映されないまま学習してしまい、「もう直っていたのに気づかず、自分も同じ箇所を重複して編集してしまう」という事故につながるためです。
- さらに、その強制リロードの**前に**`waitForPendingSync`で「今まさにサーバーへ送信中の変更」が終わるのを待っています。これをしないと、送信中の変更が完了する前に強制リロードが割り込んでしまい、送信前の古い内容（最悪カード0枚）で上書きされてしまう危険があるためです。

---

## 5. フォルダの作成・名前変更・削除・移動（1251〜1518行）

### 5.1 フォルダを開く（1251〜1259行）
```js
function openFolder(id) {
  // ★ 一覧の並び替え（長押しドラッグ）を終えた直後のタップは無視する
  //   （指を離した瞬間に発生するクリックで、意図せずフォルダが開いてしまうのを防ぐ）
  if (Date.now() - cmDragJustEndedAt < 300) return;
  currentFolderId = id;
  renderDeckListUI();
  const body = document.querySelector('#screen-list .cm-scroll-body');
  if (body) body.scrollTop = 0;
}
```
- ドラッグ並び替えを終えた直後（300ミリ秒以内）のタップは無視します。指を離した瞬間に「クリック」として扱われてしまい、意図せずフォルダが開いてしまう事故を防ぐためのガードです。

### 5.2 新規追加の入口（1261〜1274行）
```js
// ── 追加（デッキ / フォルダ）の選択 ─────
function openAddChoice() { openModal('modal-add-choice'); }
function chooseNewDeck() { closeModal('modal-add-choice'); openNewSet(); }
async function chooseNewFolder() {
  closeModal('modal-add-choice');
  if (folderLevel(currentFolderId) >= MAX_FOLDER_DEPTH) {
    await showCmAlert({
      title: 'フォルダを作成できません',
      desc: `フォルダは${MAX_FOLDER_DEPTH}階層までしか作成できません。`,
    });
    return;
  }
  openFolderNameModal('create', null);
}
```
右下の＋ボタン →「新しいデッキを作成」または「新しいフォルダを作成」の選択メニューです。フォルダ作成を選ぶと、まず`folderLevel(currentFolderId) >= MAX_FOLDER_DEPTH`（3階層制限）をチェックし、超えていれば作成できない旨のアラートを出します。デッキ作成（`chooseNewDeck`）は[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で説明する`openNewSet()`を呼ぶだけです。

### 5.3 フォルダ名の入力（1291〜1324行）
```js
async function saveFolderName() {
  const input = document.getElementById('folder-name-input');
  const name = input.value.trim();
  if (!name) { shake('folder-name-input'); return; }
  if (await warnIfBugChars(name, 'folder-name-input')) return;
  const description = document.getElementById('folder-desc-input').value.trim(); // ★ 追加：フォルダの説明欄（任意）
  if (description && await warnIfBugChars(description, 'folder-desc-input')) return;

  const btn = document.querySelector('#modal-folder-name .btn-blue');
  const targetFolder = folderNameMode === 'rename' ? folders.find(f => f.id === folderNameTargetId) : null;
  const body = {
    guild_id: GUILD_ID,
    session_token: getLoginSession()?.session_token, // ★ 追加：変更にはログイン必須
    name,
    description, // ★ 追加：フォルダの説明欄（任意）
    parent_id: folderNameMode === 'rename' ? (targetFolder ? targetFolder.parentId : null) : currentFolderId,
    nickname: getLoginSession()?.nickname, // ★ 追加：運用ログの実行者表示用
  };
  if (folderNameMode === 'rename') body.id = folderNameTargetId;

  setBtnLoading(btn, true, '保存中…'); // ★ 修正：単なるdisabledだけでなくスピナーで「処理中」を明示する
  try {
    const res = await fetch(`${API_BASE}save_folder`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body), signal: AbortSignal.timeout(8000),
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
    await fetchAndMergeFolders();
    closeModal('modal-folder-name');
    renderDeckListUI();
  } catch(e) {
    await showCmAlert({ title: 'フォルダの保存に失敗しました', desc: e.message });
  } finally {
    setBtnLoading(btn, false);
  }
}
```
- 名前が空なら`shake()`（入力欄を左右に揺らすアニメーション、8節参照）で気づかせて中断。
- ★ 追加：`description`（フォルダの説明欄、任意。空でもOK）も同じモーダルの`#folder-desc-input`から読み取り、`save_folder`のリクエストボディに含めます。`openFolderNameModal(mode, folderId)`は、名前変更モードでモーダルを開くとき、対象フォルダの現在の`name`と`description`を両方の入力欄に反映してから開きます（デッキ側の`modal-rename`と同じ考え方、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)参照）。
- `warnIfBugChars`（[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で説明）で、表示や処理を壊しかねない特殊文字が含まれていないかチェック。
- `AbortSignal.timeout(8000)`は`AbortController`を使わずタイムアウトを指定する、より簡潔な書き方（8秒でタイムアウト）。
- 成功したらサーバーの最新フォルダ一覧を取り直し、モーダルを閉じて画面を再描画。失敗したら`try/catch`で捕まえてエラーダイアログを表示（`finally`でボタンのローディング状態を必ず解除）。

### 5.4 フォルダメニュー（1328〜1422行）
```js
let folderMenuTargetId = null;
function openFolderMenu(id) {
  folderMenuTargetId = id;
  const f = folders.find(x => x.id === id);
  document.getElementById('folder-menu-name').textContent = f ? f.name : '';
  // ★「クイズ過去問」フォルダ自身はシステムフォルダなので、改名・移動・削除の
  //   操作項目を隠し、代わりに説明だけ表示する（中身の操作は制限しない）。
  const isLocked = id === QUIZ_ARCHIVE_FOLDER_ID;
  document.getElementById('folder-menu-locked-note').style.display = isLocked ? '' : 'none';
  document.getElementById('folder-menu-rename-item').style.display = isLocked ? 'none' : '';
  document.getElementById('folder-menu-move-item').style.display   = isLocked ? 'none' : '';
  document.getElementById('folder-menu-delete-item').style.display = isLocked ? 'none' : '';
  openModal('modal-folder-menu');
}
```
- 「クイズ過去問」システムフォルダ自身を対象にした場合だけ、名前変更・移動・削除の項目を隠し、代わりに「自動管理されているため操作できません」という説明を表示します（中身の操作自体は制限されません）。

```js
async function folderMenuDelete() {
  closeModal('modal-folder-menu');
  const folder = folders.find(f => f.id === folderMenuTargetId);
  if (!folder) return;

  const descIds = folderDescendants(folder.id).map(f => f.id);
  const allFolderIds = [folder.id, ...descIds];
  const targetDecks = decks.filter(d => allFolderIds.includes(d.folderId || null));

  const desc = (targetDecks.length || descIds.length)
    ? `「${folder.name}」を削除すると、中にあるサブフォルダ ${descIds.length} 個とデッキ ${targetDecks.length} 個もすべて削除されます。`
    : `「${folder.name}」を削除しますか？`;
  const ok = await showCmConfirm({
    title: 'フォルダを削除しますか？', desc, okLabel: '削除する', okStyle: 'danger',
  });
  if (!ok) return;

  // 公開済みデッキはサーバー側からも削除する。ただし作成者本人以外の
  // デッキが混ざっている場合、そのデッキだけはサーバー側でブロックされる
  // （creator_approval_required）。以前はこのレスポンスを見ずに常に
  // フォルダ本体まで削除してしまい、「フォルダは消えたのに中の他人の
  // デッキだけ孤立して残る」という不整合が起きていた。1つでもブロック
  // されたら、フォルダ自体の削除も含めて中断する（既に削除できた分だけは
  // ローカルにも反映する）。
  const deletedDecks = [];
  const blockedDecks = [];
  for (const d of targetDecks) {
    if (!d.filename) { deletedDecks.push(d); continue; } // 非公開（ローカルのみの下書き）はそのまま対象
    try {
      const res = await fetch(`${API_BASE}delete_cards`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ guild_id: GUILD_ID, session_token: getLoginSession()?.session_token, filename: d.filename, nickname: getLoginSession()?.nickname }),
      });
      const data = await res.json();
      if (data.ok) deletedDecks.push(d);
      else blockedDecks.push({ deck: d, error: data.error });
    } catch(e) {
      blockedDecks.push({ deck: d, error: e.message });
    }
  }

  if (deletedDecks.length) {
    const removedIds = new Set(deletedDecks.map(d => d.id));
    decks = decks.filter(d => !removedIds.has(d.id));
    saveDecks(decks);
  }

  if (blockedDecks.length) {
    const notOwned = blockedDecks.filter(b => b.error === 'creator_approval_required');
    if (notOwned.length) {
      const names = notOwned.map(b => `「${b.deck.name}」`).join('、');
      await showCmAlert({
        title: 'フォルダを削除できませんでした',
        desc: `${names} は他の人が作成したデッキのため、フォルダごとは削除できません。個別にデッキメニューの「デッキを削除する」から削除を依頼してください。`,
      });
    } else {
      await showCmAlert({ title: '一部のデッキの削除に失敗しました', desc: 'フォルダの削除を中断しました。もう一度お試しください。' });
    }
    renderDeckListUI();
    return;
  }

  // フォルダ自体もサーバー（みんなで共有）から削除
  try {
    const res = await fetch(`${API_BASE}delete_folder`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: getLoginSession()?.session_token, id: folder.id, nickname: getLoginSession()?.nickname }), signal: AbortSignal.timeout(8000),
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
  } catch(e) {
    await showCmAlert({ title: 'サーバーからのフォルダ削除に失敗しました', desc: e.message });
    return;
  }

  if (allFolderIds.includes(currentFolderId)) currentFolderId = folder.parentId || null;
  await fetchAndMergeFolders();
  renderDeckListUI();
}
```
`folderMenuDelete()`（1344〜1422行）はフォルダ削除の本体で、かなり丁寧な手順を踏みます：
1. 削除対象のフォルダとその配下すべて（サブフォルダ・デッキ）を確認し、件数を含めた確認ダイアログを出す。
2. 配下の公開済みデッキを1件ずつサーバーの`/delete_cards`に削除依頼を送る。非公開（下書き）デッキはそのまま削除対象にする。
3. **作成者本人以外のデッキが混ざっていた場合**、そのデッキだけサーバー側で削除がブロックされます（`creator_approval_required`というエラー）。コメントによれば、これは以前「フォルダは消えたのに、他人のデッキだけが孤立して残ってしまう」という不整合を修正した結果の実装で、**1件でもブロックされたら、削除できた分だけローカルに反映しつつ、フォルダ本体の削除自体は中断**します。ブロックされた場合は「個別にデッキメニューから削除を依頼してください」と案内します。
4. 全デッキの削除に成功していれば、フォルダ自体もサーバーから削除し、もし今そのフォルダの中を見ていたら親フォルダ（またはホーム）に移動して再描画します。

### 5.5 移動先の選択（1437〜1518行）
```js
function renderMovePickerList() {
  const list = document.getElementById('move-picker-list');
  const currentParent = movePickerKind === 'deck'
    ? (decks.find(d => d.id === movePickerTargetId)?.folderId || null)
    : (folders.find(f => f.id === movePickerTargetId)?.parentId || null);

  const canMoveTo = movePickerKind === 'folder'
    ? (targetId) => canMoveFolderTo(movePickerTargetId, targetId)
    : (targetId) => canMoveDeckTo(movePickerTargetId, targetId);

  const rows = [];
  rows.push({ id: null, icon: Icons.html('home', {size:14}), label: 'ルート', level: 0, disabled: !canMoveTo(null) });

  function walk(parentId, level) {
    folderChildren(parentId).forEach(f => {
      const disabled = !canMoveTo(f.id);
      rows.push({ id: f.id, icon: Icons.cmHtml('folder', {size:14}), label: f.name, level, disabled });
      walk(f.id, level + 1);
    });
  }
  walk(null, 1);

  list.innerHTML = rows.map(r => {
    const isCurrent = r.id === currentParent;
    const cls = 'move-picker-row'
      + (r.disabled ? ' disabled' : '')
      + (isCurrent ? ' current' : '');
    const idAttr = r.id === null ? 'null' : `'${r.id}'`;
    const clickAttr = r.disabled ? '' : ` onclick="selectMoveTarget(${idAttr})"`;
    return `<div class="${cls}" style="padding-left:${8 + r.level * 18}px"${clickAttr}>${r.icon} ${esc(r.label)}${isCurrent ? ' <span class="move-picker-current-tag">現在</span>' : ''}</div>`;
  }).join('');
}
```
- `walk(parentId, level)`という**自分自身をもう一度呼び出す関数（再帰関数）**で、フォルダの階層構造をたどりながら、フォルダツリー全体をインデント付き（`level`が深いほど`padding-left`を大きく）の1本のリストに展開しています。
- 各行について、移動先として選べるかどうか（`canMoveFolderTo`/`canMoveDeckTo`、[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の条件）を判定し、選べない行は`disabled`のスタイルにしてクリックできないようにします。
- コメントにある通り、アイコン（固定のSVG文字列）とフォルダ名（ユーザー入力）は別々に扱い、フォルダ名だけを`esc()`に通しています。もしアイコンごと`esc()`してしまうと、SVGタグの`<`や`>`まで文字列としてエスケープされてしまい、アイコンが正しく描画されなくなるためです。

```js
async function selectMoveTarget(targetId) {
  closeModal('modal-move-picker');

  if (movePickerKind === 'deck') {
    const d = decks.find(x => x.id === movePickerTargetId);
    if (!d || !canMoveDeckTo(d.id, targetId)) return;

    // ★ 修正：公開済みデッキを移動する前に、必ずサーバーから最新のカード本体を
    //   取り直す。失敗時は loadDeckCardsWithRecovery が再試行・強制続行の
    //   選択肢を提示するので、移動できないまま詰むことはない。
    if (d.filename) {
      const loaded = await loadDeckCardsWithRecovery(d.id);
      if (!loaded) return; // ユーザーが「やめる」を選んだ場合は移動しない
    }

    d.folderId = targetId;
    saveDecks(decks);
    renderDeckListUI();
    // ★ 公開済みデッキはサーバー側（みんなの共有フォルダ情報）にも反映する
    if (d.filename) {
      const ok = await queueSyncDeckToServer(d);
      if (!ok) showBanner('サーバーへの移動の反映に失敗しました（ローカルには保存済み）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
    }
    return;
  }

  // フォルダの移動（みんなで共有）
  const f = folders.find(x => x.id === movePickerTargetId);
  if (!f || !canMoveFolderTo(f.id, targetId)) return;
  try {
    const res = await fetch(`${API_BASE}save_folder`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ guild_id: GUILD_ID, session_token: getLoginSession()?.session_token, id: f.id, name: f.name, parent_id: targetId, nickname: getLoginSession()?.nickname }),
      signal: AbortSignal.timeout(8000),
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error || '不明なエラー');
    await fetchAndMergeFolders();
    renderDeckListUI();
  } catch(e) {
    await showCmAlert({ title: 'フォルダの移動に失敗しました', desc: e.message });
  }
}
```
`selectMoveTarget(targetId)`（1476〜1518行）は実際に移動を実行する関数です。デッキの移動では、公開済みデッキの場合、移動前に必ずサーバーから最新のカード本体を取り直す（`loadDeckCardsWithRecovery`）処理が入っています。これは「移動しようとしたら、実は誰かが直前にそのデッキを更新していた」というケースに対応するためです。フォルダの移動は、名前はそのままで`parent_id`（親フォルダ）だけを変えて`/save_folder`に送ります。

---

続きは[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で、`fetchAndMergeDecks()`（サーバーからのデッキ取得）以降を解説します。
