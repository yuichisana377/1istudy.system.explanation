# Cardmaker.js その2：デッキ一覧の描画とフォルダ操作（812〜1523行）

[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の続きです。

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
今のフォルダの直下にある子フォルダ・子デッキを集め、両方とも0件なら「まだデッキがありません」（フォルダの中なら「このフォルダにはまだ何もありません」）を表示して終了します。

### 1.3 フォルダのカードを組み立てる（844〜884行）
`childFolders.map(f => {...})`で、フォルダ1つごとに表示用のHTML片を作ります。
- `countDecksRecursive`/`countCardsRecursive`/`countUnsureRecursive`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)参照）で、そのフォルダ配下の合計デッキ数・カード数・「わからない」件数を計算。
- **クイズ用デッキ選択モード（`pickMode`）中は特別な見た目**：通常の「▶プレイ」「メニュー」ボタンの代わりに、チェックボックス風の見た目でフォルダ全体を選択できるようにします。選択対象にできるデッキが1件も無いフォルダはグレーアウト（`disabled`）。
- 通常モードでは、フォルダ名・件数に加えて「▶プレイ」ボタン（フォルダ内を一括で学習）と「メニュー」ボタン（名前変更・移動・削除）を表示します。
- それぞれのフォルダに`key: 'folder:フォルダID'`という並び順管理用のキーを持たせています。

### 1.4 デッキのカードを組み立てる（885〜1001行）

まず表示順の初期値を決めます：
```js
const unpublished = childDecks.filter(d => !d.filename).slice().reverse();
const published    = childDecks.filter(d =>  d.filename).slice().reverse();
const orderedDecks = [...unpublished, ...published];
```
- `d.filename`が無い＝まだサーバーに公開されていない下書きデッキ。それらを先に、公開済みデッキをあとに並べ、それぞれのグループ内は新しい順（`.reverse()`で配列の並びを逆転）にしています。ただしコメントにある通り、これはあくまで「ユーザーが手で並び替えていない場合の初期順」で、保存済みの並び順があれば後段の`applySavedListOrder`がこれを上書きします。

デッキ1件ごとに、かなり細かい表示の出し分けロジックがあります：
- **「わからない」バッジ**：カード本体が読み込み済み（`cardsLoaded !== false`）のときだけ計算（未読み込みのデッキは`cards`が空配列のままなので計算できないため）。
- **問題数の表示**：`d.filename ? (d.count ?? d.cards.length) : d.cards.length`。公開済みデッキはサーバー側の軽量な`count`情報（カード本体を読み込まなくても分かる件数）を優先し、下書きデッキは手元の`cards`配列の長さをそのまま使います。
- **公開状態バッジ（`pubBadge`）**：「作成中」「非公開」「未完成」「公開済み」の4種類のうちどれか1つだけを表示します。コメントに詳しい経緯が書かれていて、以前は「サーバー登録済み・カード0枚」だけを「作成中」と判定していたため、公開フローを経ずにただ「保存」しただけのデッキが、カードを1枚追加した途端に誤って「未完成」表示に変わってしまうバグがあったそうです。今は`d.notYetPublished`（一度も明示的な「公開して保存」を経ていないか）を最優先で見て、3段階に整理し直しています。
- **「過去問」バッジ**：そのデッキが「クイズ過去問」システムフォルダの中にあれば表示（プレイ時の挙動が普通のフラッシュカードと違うため）。
- **「選択式」バッジ**：多肢選択デッキ（`d.choiceMode`）であれば表示。ただし「過去問」バッジと意味が重複するため、そちらが出ているときはこちらは出しません。
- **プレイボタンの無効化条件**：問題数0、読み込み中、または「作成中」（＝一度も公開フローを経ていない）状態のいずれかに該当すればプレイ不可（`playDisabled`）。**編集はこのフラグを見ないので、作成中でも編集は引き続き可能**、という注記もあります。
- **科目名の重複表示回避**：デッキ名の先頭に科目名がそのまま含まれている場合（例：「数学 二次関数」）、表示名からその部分を取り除いて、科目名は別枠（`subjectLabel`）で表示するようにしています。
- **並び順キー（`orderKey`）**：公開済みデッキは全員が同じ`filename`を持つので、それを共有キー（`deck:ファイル名`）にします。未公開デッキは他人には見えないデータなので、この端末専用のキー（`localdeck:内部ID`）にし、サーバーには送りません。
- こちらもクイズ用デッキ選択モード中は、通常のボタンの代わりにチェックボックスの見た目に切り替わります。

### 1.5 並び順の適用と、重複描画の最終防御（1003〜1016行）
```js
const combinedItems = applySavedListOrder([...folderItems, ...deckItems], currentFolderId);
const seenKeys = new Set();
const dedupedItems = combinedItems.filter(it => {
  if (seenKeys.has(it.key)) return false;
  seenKeys.add(it.key);
  return true;
});
grid.innerHTML = dedupedItems.map(it => it.html).join('');
```
- `applySavedListOrder`（[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)）で、ユーザーがドラッグして決めた並び順を適用。
- 最後に、同じキー（＝同じデッキ／フォルダ）を持つ項目が万一2件以上並んでいた場合でも、最初の1件だけを残して残りを除外する「最終防御」を入れています。コメントによれば、これは「並び順マージ処理などに未知の不具合があっても、画面上に同じ項目が2つ表示される、という見た目の破綻だけは常に防ぐため」の保険です。

---

## 2. パンくずリスト（1018〜1038行）

```js
function renderBreadcrumb() {
  const bar = document.getElementById('folder-breadcrumb');
  if (!currentFolderId) { bar.style.display = 'none'; bar.innerHTML = ''; return; }
  const chain = [];
  let cur = folders.find(f => f.id === currentFolderId);
  while (cur) { chain.unshift(cur); cur = folders.find(f => f.id === cur.parentId); }
  bar.style.display = 'flex';
  bar.innerHTML = `<span class="crumb" onclick="openFolder(null)">🏠 ホーム</span>` +
    chain.map(f => `<span class="crumb-sep">/</span><span class="crumb" onclick="openFolder('${f.id}')">${esc(f.name)}</span>`).join('');
}
```
- 今のフォルダから親を順にたどって配列の先頭に追加していく（`chain.unshift`）ことで、「ホーム › 数学 › 二次関数」のような経路の並びを作ります。
- ホームにいるとき（`currentFolderId`が`null`）はパンくず自体を非表示にします。

`folderPathLabel(folderId)`（1032〜1038行）は同じ考え方で、パンくずと同じ経路を「/」区切りの1本の文字列にして返す関数です。検索画面で「今どこを検索対象にしているか」を表示するのに使われます。

---

## 3. 単語検索画面への入口（1040〜1056行）

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

## 4. ホーム画面の「プレイ中」「プレイ済み」セクション（1058〜1244行）

### 4.1 「プレイ中（続きから再開できる）」の一覧（1058〜1121行）
```js
function getInProgressItems(scopeFolderId) {
  const items = [];
  for (const key of Object.keys(studyDataCache.progress)) {
    // key が "deck:xxx" か "folder:xxx" かで isFolder/id を判定
    // loadStudyProgress() で進捗データを読み、対応するデッキ/フォルダがまだ存在するかチェック
    // scopeFolderId（表示範囲）に含まれないものは除外
    // 見つかったものを items に積む
  }
  const ONE_WEEK_MS = 7 * 24 * 60 * 60 * 1000;
  const now = Date.now();
  const recentItems = items.filter(it => (now - it.updatedAt) <= ONE_WEEK_MS);
  recentItems.sort((a, b) => b.updatedAt - a.updatedAt);
  return recentItems;
}
```
- 学習の途中経過（`studyDataCache.progress`、[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)で説明）を全部調べ、「まだ存在するデッキ/フォルダ」「表示範囲内」「直近1週間以内に触っていた」の3条件を満たすものだけを、最近触った順に並べて返します。
- `renderInProgressUI()`（1097〜1121行）はこれを実際にホーム画面の「▶️ プレイ中のデッキ」欄に描画する関数です。進捗バー（`〇問中△問目`をパーセントに変換）付きのカードを並べ、最後に`renderCompletedUI()`（下記）も一緒に呼び出します。

### 4.2 「プレイ済み（完了）」の一覧（1123〜1179行）
`getCompletedItems`/`renderCompletedUI`は、上記の「プレイ中」とほぼ同じ考え方で、完了記録（`studyDataCache.completed`）を対象にした版です。「直近1週間以内に完了したもの」を新しい順に表示します。

### 4.3 ホーム画面のカードをタップしたときの挙動（1181〜1244行）
```js
async function replayFromHome(isFolder, id) {
  if (isFolder) { await openFolderPlayMode(id); } else { await openPlayMode(id); }
}
```
- 「プレイ済み」カードをタップすると、通常のプレイモード選択画面（すべて／わからないだけ、など）を開き直します。

```js
async function resumeFromHome(isFolder, id) {
  // フォルダ／デッキそれぞれについて：
  //   読み込み中フラグを立てて再描画
  //   保留中のサーバー同期を待つ（waitForPendingSync）
  //   最新のカードを強制的にサーバーから取り直す（ensureDeckCardsLoaded(id, true)）
  //   失敗したらエラーを表示して中断
  //   成功したら「続きから」の状態で学習モードを開始（startStudyMode('resume')）
}
```
- 「プレイ中」カードをタップすると、モード選択を経由せず**直接続きから**学習画面を開きます。
- 注目すべき点は、プレイを始める前に**毎回サーバーの最新カードを強制的に取り直している**ことです。コメントに理由が書かれています：もしキャッシュ済みのまま開いてしまうと、他の人が先に直した最新の修正内容が反映されないまま学習してしまい、「もう直っていたのに気づかず、自分も同じ箇所を重複して編集してしまう」という事故につながるためです。
- さらに、その強制リロードの**前に**`waitForPendingSync`で「今まさにサーバーへ送信中の変更」が終わるのを待っています。これをしないと、送信中の変更が完了する前に強制リロードが割り込んでしまい、送信前の古い内容（最悪カード0枚）で上書きされてしまう危険があるためです。

---

## 5. フォルダの作成・名前変更・削除・移動（1246〜1513行）

### 5.1 フォルダを開く（1246〜1255行）
```js
function openFolder(id) {
  if (Date.now() - cmDragJustEndedAt < 300) return;
  currentFolderId = id;
  renderDeckListUI();
  ...
}
```
- ドラッグ並び替えを終えた直後（300ミリ秒以内）のタップは無視します。指を離した瞬間に「クリック」として扱われてしまい、意図せずフォルダが開いてしまう事故を防ぐためのガードです。

### 5.2 新規追加の入口（1257〜1270行）
右下の＋ボタン →「新しいデッキを作成」または「新しいフォルダを作成」の選択メニューです。フォルダ作成を選ぶと、まず`folderLevel(currentFolderId) >= MAX_FOLDER_DEPTH`（3階層制限）をチェックし、超えていれば作成できない旨のアラートを出します。

### 5.3 フォルダ名の入力（1272〜1320行）
```js
async function saveFolderName() {
  const name = input.value.trim();
  if (!name) { shake('folder-name-input'); return; }
  if (await warnIfBugChars(name, 'folder-name-input')) return;
  ...
  const res = await fetch(`${API_BASE}save_folder`, { method: 'POST', ... body: JSON.stringify(body), signal: AbortSignal.timeout(8000) });
  const data = await res.json();
  if (!data.ok) throw new Error(data.error || '不明なエラー');
  await fetchAndMergeFolders();
  closeModal('modal-folder-name');
  renderDeckListUI();
}
```
- 名前が空なら`shake()`（入力欄を左右に揺らすアニメーション、8節参照）で気づかせて中断。
- `warnIfBugChars`（[08_Cardmaker.js_その8_画像処理と基盤機能.md](08_Cardmaker.js_その8_画像処理と基盤機能.md)で説明）で、表示や処理を壊しかねない特殊文字が含まれていないかチェック。
- `AbortSignal.timeout(8000)`は`AbortController`を使わずタイムアウトを指定する、より簡潔な書き方（8秒でタイムアウト）。
- 成功したらサーバーの最新フォルダ一覧を取り直し、モーダルを閉じて画面を再描画。失敗したら`try/catch`で捕まえてエラーダイアログを表示（`finally`でボタンのローディング状態を必ず解除）。

### 5.4 フォルダメニュー（1322〜1418行）
```js
function openFolderMenu(id) {
  ...
  const isLocked = id === QUIZ_ARCHIVE_FOLDER_ID;
  document.getElementById('folder-menu-locked-note').style.display = isLocked ? '' : 'none';
  document.getElementById('folder-menu-rename-item').style.display = isLocked ? 'none' : '';
  ...
}
```
- 「クイズ過去問」システムフォルダ自身を対象にした場合だけ、名前変更・移動・削除の項目を隠し、代わりに「自動管理されているため操作できません」という説明を表示します（中身の操作自体は制限されません）。

`folderMenuDelete()`（1340〜1418行）はフォルダ削除の本体で、かなり丁寧な手順を踏みます：
1. 削除対象のフォルダとその配下すべて（サブフォルダ・デッキ）を確認し、件数を含めた確認ダイアログを出す。
2. 配下の公開済みデッキを1件ずつサーバーの`/delete_cards`に削除依頼を送る。非公開（下書き）デッキはそのまま削除対象にする。
3. **作成者本人以外のデッキが混ざっていた場合**、そのデッキだけサーバー側で削除がブロックされます（`creator_approval_required`というエラー）。コメントによれば、これは以前「フォルダは消えたのに、他人のデッキだけが孤立して残ってしまう」という不整合を修正した結果の実装で、**1件でもブロックされたら、削除できた分だけローカルに反映しつつ、フォルダ本体の削除自体は中断**します。ブロックされた場合は「個別にデッキメニューから削除を依頼してください」と案内します。
4. 全デッキの削除に成功していれば、フォルダ自体もサーバーから削除し、もし今そのフォルダの中を見ていたら親フォルダ（またはホーム）に移動して再描画します。

### 5.5 移動先の選択（1420〜1513行）
```js
function renderMovePickerList() {
  const canMoveTo = movePickerKind === 'folder'
    ? (targetId) => canMoveFolderTo(movePickerTargetId, targetId)
    : (targetId) => canMoveDeckTo(movePickerTargetId, targetId);
  const rows = [];
  rows.push({ id: null, icon: ..., label: 'ルート', level: 0, disabled: !canMoveTo(null) });
  function walk(parentId, level) {
    folderChildren(parentId).forEach(f => {
      rows.push({ id: f.id, icon: ..., label: f.name, level, disabled: !canMoveTo(f.id) });
      walk(f.id, level + 1);
    });
  }
  walk(null, 1);
  list.innerHTML = rows.map(r => `<div class="..." style="padding-left:${8 + r.level * 18}px" ...>${r.icon} ${esc(r.label)}...</div>`).join('');
}
```
- `walk(parentId, level)`という**自分自身をもう一度呼び出す関数（再帰関数）**で、フォルダの階層構造をたどりながら、フォルダツリー全体をインデント付き（`level`が深いほど`padding-left`を大きく）の1本のリストに展開しています。
- 各行について、移動先として選べるかどうか（`canMoveFolderTo`/`canMoveDeckTo`、[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の条件）を判定し、選べない行は`disabled`のスタイルにしてクリックできないようにします。
- コメントにある通り、アイコン（固定のSVG文字列）とフォルダ名（ユーザー入力）は別々に扱い、フォルダ名だけを`esc()`に通しています。もしアイコンごと`esc()`してしまうと、SVGタグの`<`や`>`まで文字列としてエスケープされてしまい、アイコンが正しく描画されなくなるためです。

`selectMoveTarget(targetId)`（1471〜1513行）は実際に移動を実行する関数です。デッキの移動では、公開済みデッキの場合、移動前に必ずサーバーから最新のカード本体を取り直す（`loadDeckCardsWithRecovery`）処理が入っています。これは「移動しようとしたら、実は誰かが直前にそのデッキを更新していた」というケースに対応するためです。フォルダの移動は、名前はそのままで`parent_id`（親フォルダ）だけを変えて`/save_folder`に送ります。

---

続きは[03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)で、`fetchAndMergeDecks()`（サーバーからのデッキ取得）以降を解説します。
