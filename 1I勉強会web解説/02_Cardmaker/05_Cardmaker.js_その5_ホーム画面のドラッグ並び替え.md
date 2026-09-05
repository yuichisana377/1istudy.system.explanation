# Cardmaker.js その5：ホーム画面の長押しドラッグ並び替え（2403〜2980行）

[04_Cardmaker.js_その4_カード保存と公開.md](04_Cardmaker.js_その4_カード保存と公開.md)の続きです。ここは、CardMakerの中でも特に細かい調整が積み重なった部分です。まず必要な用語を押さえてから読み進めてください。

## このパートで出てくる新しい用語

- **タッチ操作／マウス操作の共通化（Pointer Events）**：スマホの指での操作（タッチ）も、PCのマウスでの操作も、JSでは別々の仕組み（`touchstart`系のイベントと`mousedown`系のイベント）で扱うことができますが、このコードは**Pointer Events**という新しめの仕組み（`pointerdown`/`pointermove`/`pointerup`）を使って、指でもマウスでも同じコードで扱えるようにしています。
- **`touch-action`（CSSのプロパティ）**：スマホのブラウザに対して「この部品の上では、指でどんな標準ジェスチャー（スクロール等）を許可するか」を指定するCSSの設定です。`'none'`にすると、その部品の上ではブラウザ標準のスクロールなどが一切効かなくなり、代わりにJSで動きを完全に自作できるようになります。
- **`requestAnimationFrame`（rAF）**：「次に画面が実際に描き変わるタイミングに合わせて、この処理を1回実行してください」とブラウザにお願いする命令です。何かをアニメーションさせたいとき、`setTimeout`で無理に間隔を指定するより滑らかに動きます。
- **`getBoundingClientRect()`**：ある部品が、画面上の「今どこに・どれくらいの大きさで」表示されているかを調べる命令。上端・下端・幅・高さなどがまとめて返ってきます。
- **`elementFromPoint(x, y)`**：画面上のある座標（x, y）に、今何の部品が表示されているかを調べる命令。「指の真下に何があるか」を知りたいときに使います。
- **`setPointerCapture`**：「これ以降の指の動きは、最初に触れたこの部品が責任を持って受け取り続ける」と指定する命令。指がその部品の外に出ても、動きの情報を取りこぼさずに済みます。
- **バイブレーション（`navigator.vibrate`）**：対応する端末で、端末を短く振動させる命令。「掴んだ」ことを指の感覚でも伝えるための演出です。

---

## この機能の全体像

デッキ・フォルダの一覧（ホーム画面）を長押しすると、そのままドラッグして順番を並び替えられます。CardMakerの他の並び替え（作成済みカードの並び替えなど）と違って専用の「掴む取っ手（⠿マーク）」を出さず、**カード本体をどこでも長押しすればそのまま掴める**、というスマホでもやりやすい操作感を目指しています。実装全体は`(function setupListDragReorder() {...})();`という即時実行関数（IIFE）の中に閉じ込められており、外部に余計な変数を漏らさない作りです。ページが読み込まれた瞬間に1回だけ実行され、以降はイベントリスナーとして待ち構えます。

この機能が解決しようとしている問題は主に3つです：
1. **普通のスワイプ（画面スクロール）と、並び替えのための長押しドラッグをどう区別するか**
2. **並び替え中に指がフォルダの上に重なったら、どうやってそのフォルダの中に入れるか**
3. **並び替え中に、裏で動いている自動更新の再描画とぶつからないようにするにはどうするか**（これは[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)の`cmListDragActive`ですでに触れた話です）

---

## 1. 調整用の定数と状態変数（2408〜2482行）

```js
const LONG_PRESS_MS  = 380; // これだけ指を止めたままにすると並び替えモードに入る
const MOVE_CANCEL_PX = 10;  // 判定前にこれ以上動いたら「スクロール」とみなし長押しをキャンセル
const TOUCH_ACTION_DELAY_MS = 60;
grid.style.touchAction = 'pan-y';
```
- `pan-y`は「縦方向のスクロールだけはブラウザの標準動作に任せる」という指定です。以前は常に`'none'`にして縦スクロールもJSで手動再現していたものの、ネイティブのスクロール特有の滑らかさ・慣性・端まで到達したときの弾む感じ（ラバーバンド）には及ばず「うまくスクロールできない」という不満につながっていた、とコメントに書かれています。

このあと大量の状態変数（`pressTimer`・`dragEl`・`hoverFolderEl`・`manualScrollActive`など）が定義されますが、それぞれの役割は各処理の説明の中で触れていきます。「様子見してから並び替えモードに切り替える」という設計のため、状態管理がかなり細かくなっています。

---

## 2. 長押しが始まってから、実際にドラッグが始まるまで（2806〜2868行）

```js
function onPointerDown(e) {
  if (e.pointerType === 'mouse' && e.button !== 0) return;
  if (dragEl) return; // 既に別の指・別のポインタでドラッグ中
  const item = e.target.closest('.deck-card');
  if (!item || item.parentElement !== grid) return;
  // ▶プレイ／✏️メニューなどのボタンから始まった場合は、通常のタップ操作を優先する
  if (e.target.closest('button, .btn, .icon-btn, a')) return;

  pressItem = item;
  pressPointerId = e.pointerId;
  pressStartX = e.clientX;
  pressStartY = e.clientY;
```
- マウスなら左クリック以外は無視、すでに別の指でドラッグ中なら無視。押された場所が「▶プレイ」「メニュー」などのボタンの上だった場合も、通常のタップ操作を優先してドラッグ判定には入りません（`closest('button, .btn, ...')`で、押された要素の祖先にボタンが無いか確認）。
- `pressItem`/`pressPointerId`/`pressStartX`/`pressStartY`… まだ「長押しが確定したかどうか分からない」様子見の段階の情報を控えておきます。指がどの部品の、どの指（`pointerId`）で、どの座標から始まったかを覚えておくことで、後で「長押しが成立したか」「動きすぎてキャンセルすべきか」を判定できるようにします。

```js
    cmListDragActive = true;
```
- 重要な点として、**長押しがまだ確定していない「様子見」の段階から**すでに`cmListDragActive`を立てています。コメントによれば、これをしないと、長押し確定前の待ち時間中にバックグラウンドの自動更新が一覧を再描画してしまい、掴もうとしていた要素が新しいDOMから浮いた「孤立した古い要素」になってしまう、という不具合（デッキが一時的に2つ表示される）が起きるためです。

```js
if (e.pointerType === 'touch') {
  manualScrollParent = findScrollParent(item);
  manualScrollLastY = e.clientY;
  manualScrollActive = false;
  touchActionTimer = setTimeout(() => {
    touchActionTimer = null;
    if (pressItem !== item) return;
    touchActionItem = item;
    touchActionItem.style.touchAction = 'none';
  }, TOUCH_ACTION_DELAY_MS);
}
```
- ここがこの機能で一番工夫されている部分です。指を置いた瞬間に即座に`touch-action: none`にしてしまうと、ただ普通にスワイプしてスクロールしたいだけの操作でも、その一瞬だけネイティブスクロールが止められてしまい「スワイプしづらい」原因になっていたそうです。そこで、指を置いてから60ミリ秒（`TOUCH_ACTION_DELAY_MS`）だけ**様子を見て**、その間にあまり動かなければ「これは長押し（並び替え）をしたいのだ」と判断してから`touch-action: none`を適用する、という2段構えにしています。

```js
pressTimer = setTimeout(() => {
  pressTimer = null;
  if (!pressItem) return;
  if (!pressItem.isConnected || pressItem.parentElement !== grid) { cancelPress(); return; }
  try { pressItem.setPointerCapture(pressPointerId); } catch (_) {}
  lastClientX = pressStartX;
  beginDrag(pressItem, pressStartY);
}, LONG_PRESS_MS);
```
- 380ミリ秒（`LONG_PRESS_MS`）経過しても指がその場に留まっていれば、実際にドラッグを開始します（`beginDrag`、3節）。その直前に`pressItem.isConnected`（その部品が今も画面のDOMツリーの中に実在するか）をチェックし、もし裏の再描画によって浮いてしまっていたら、静かにドラッグ開始を諦めます。

---

## 3. ドラッグの開始・移動・終了（2524〜2794行）

### 3.1 `beginDrag(item, clientY, initialDy)`（2540〜2558行）
```js
function beginDrag(item, clientY, initialDy) {
  initialDy = initialDy || 0; // ★ 追加：フォルダ切り替え直後の再開時、指の位置とカードの見た目を
                               //   一致させるための初期オフセット（通常の掴み始めは0でよい）
  dragEl = item;
  cmListDragActive = true; // ★ ドラッグ中は renderDeckListUI() 側で再描画をスキップさせる
  startY = clientY - initialDy;
  lastClientY = clientY;
  dragOriginY = clientY; // ★ 追加：自動スクロール発動判定の基準点
  scrollParent = findScrollParent(grid);
  dragEl.classList.add('dragging');
  dragEl.style.position = 'relative';
  dragEl.style.zIndex = '10';
  dragEl.style.boxShadow = '0 6px 18px rgba(0,0,0,.20)';
  dragEl.style.opacity = '0.92';
  dragEl.style.touchAction = 'none';
  dragEl.style.transform = `translateY(${initialDy}px) scale(1.02)`;
  if (navigator.vibrate) navigator.vibrate(12); // ★ つかんだ瞬間に軽い振動でフィードバック（対応端末のみ）
  if (autoScrollRAF === null) autoScrollRAF = requestAnimationFrame(autoScrollTick);
}
```
- 掴んだカードに影を付けて少し浮かせ、わずかに拡大表示（`scale(1.02)`）して「今これを掴んでいます」と分かるようにします。対応端末では12ミリ秒の短い振動も鳴らします。
- `initialDy`という引数は普段は0ですが、後述の「フォルダを自動で開いて掴んだままドラッグを続ける」処理（5節）のときだけ、指の位置とカードの見た目がずれないようにするための補正値として使われます。

### 3.2 `moveDrag(clientX, clientY)`（2544〜2576行）— 指が動くたびに呼ばれる
```js
const dy = clientY - startY;
dragEl.style.transform = `translateY(${dy}px) scale(1.02)`;

const dragRect = dragEl.getBoundingClientRect();
const dragCenter = dragRect.top + dragRect.height / 2;
const items = getItems();
for (const other of items) {
  if (other === dragEl) continue;
  const r = other.getBoundingClientRect();
  const otherCenter = r.top + r.height / 2;
  const otherIsAfter = !!(dragEl.compareDocumentPosition(other) & Node.DOCUMENT_POSITION_FOLLOWING);
  if (otherIsAfter && dragCenter > otherCenter) {
    grid.insertBefore(dragEl, other.nextSibling);
    startY = clientY;
    dragEl.style.transform = 'translateY(0px) scale(1.02)';
    break;
  } else if (!otherIsAfter && dragCenter < otherCenter) {
    grid.insertBefore(dragEl, other);
    startY = clientY;
    dragEl.style.transform = 'translateY(0px) scale(1.02)';
    break;
  }
}
```
- 掴んでいるカードを、指の動いた分だけ`transform: translateY(...)`（見た目上の位置だけを動かす、実際のDOM構造上の並びは変えない）でついてこさせます。
- 同時に、掴んでいるカードの中心が、他のどのカードの中心を追い越したかを毎回チェックします。`compareDocumentPosition`は「このカードは自分より後ろにあるか」をブラウザに聞く命令で、それと現在の位置関係を組み合わせることで、「下に動かして、後ろにあるカードの中心を追い越した→そのカードの後ろに割り込む」「上に動かして、前にあるカードの中心を追い越した→そのカードの前に割り込む」という判定をしています。
- `grid.insertBefore(dragEl, other)`で、実際のDOM構造（HTMLの要素の並び）そのものを並べ替えます。並べ替えた瞬間に`startY`を今の指の位置に更新し、見た目のズレをリセットします。

このあと`checkHoverFolder(clientX, clientY)`（4節）を呼び、フォルダの上に重なっていないかもチェックします。

### 3.3 自動スクロール（2503〜2522行）
指がドラッグ中に画面の上端・下端近くまで来ると、一覧が自動的にスクロールします。
```js
if (lastClientY < rect.top + edge && lastClientY < dragOriginY - EDGE_ARM_PX) {
  speed = -maxSpeed * Math.min(1, (rect.top + edge - lastClientY) / edge);
} else if (lastClientY > rect.bottom - edge && lastClientY > dragOriginY + EDGE_ARM_PX) {
  speed = maxSpeed * Math.min(1, (lastClientY - (rect.bottom - edge)) / edge);
}
```
- 端に近いほどスクロール速度を上げる（端ぎりぎりなら最大速度`maxSpeed`）計算です。
- ここで注目すべきは`dragOriginY - EDGE_ARM_PX`という条件が一緒に付いている点です。コメントによれば、「掴んだ位置がたまたま画面の上端／下端付近だった場合、指を全く動かしていないのに自動スクロールが始まり、一瞬で一番上/下まで並び替わってしまう」というバグが過去にあったため、**「掴んだ位置から実際にその方向へある程度（`EDGE_ARM_PX`=24px）動かした場合にだけ」**自動スクロールを有効にする、という条件を追加で足しています。
- `requestAnimationFrame(autoScrollTick)`で自分自身を毎フレーム呼び直すことで、指を止めたままでも連続してスクロールし続けます。

### 3.4 `endDrag()`（2765〜2794行）— 指を離したとき
```js
const orderedKeys = getItems().map(it => it.dataset.key);
saveListOrder(currentFolderId, orderedKeys);
if (orderedKeys.some(isSharedOrderKey)) {
  pushSharedOrderToServer(currentFolderId, orderedKeys).then(ok => {
    if (!ok) showBanner('並び替えのサーバー反映に失敗しました（この端末には保存済み）', '#fffbeb', '#92400e', Icons.html('warning', {size:15}));
  });
}
cmDragJustEndedAt = Date.now();
```
- 見た目のスタイル（影・拡大・浮かせる効果）を全部元に戻し、今の`#deck-grid`の中の実際の並び順（`data-key`の並び）をそのまま`saveListOrder`で保存します。
- 共有される項目（フォルダ・公開デッキ）が1件でも含まれていれば、サーバーにも反映します。自分だけの下書きデッキしか動かしていなければ、サーバー通信自体を省略します。
- 最後に`cmDragJustEndedAt`を記録（[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)の`openFolder`が、指を離した直後の誤クリックを無視するのに使う値）。

### 追記（2026/08/26）：`pushSharedOrderToServer`が定期同期に上書きされる競合の修正

コードレビューで、`pushSharedOrderToServer`にも[06_Cardmaker.js_その6](06_Cardmaker.js_その6_カード編集と学習データ同期.md)の`pushStudyDataToServer`と全く同じ競合バグが見つかった：この関数はサーバーへの送信を待たずに`.then(...)`で結果を扱うだけの“投げっぱなし”寄りの使われ方をしており、一覧画面を見ている間10秒おきに走る`checkOrderUpdate`→`fetchAndMergeOrder`が、送信がサーバーに届く前に割り込むと、まだ反映されていない古い並び順で`sharedOrderCache`を丸ごと上書きしてしまう。指でドラッグして決めた並びが、他の生徒の並び替えを拾いに行っただけの定期同期のせいで一瞬で元に戻って見える、という不具合が起きうる状態だった。

`study_data`側で確立した直し方（送信中のPromiseを`_pendingOrderPushes`に記録し、`fetchAndMergeOrder`側でそれらの完了を待ってから取得する。さらに、取得の通信中に新しい送信が始まっていた場合は上書き自体をスキップする）をそのまま適用した。

---

## 4. フォルダの上に重ねると自動で開く（2578〜2669行）

iOSのホーム画面でアプリのアイコンを別のアプリの上に重ねるとフォルダが作られる、あの操作感を再現した機能です。

```js
function checkHoverFolder(clientX, clientY) {
  if (!dragEl || hoverOpenInProgress) return;

  // ① まず「パンくず付近まで持ち上げたら親フォルダへ戻る」ゾーンを判定する
  //   （フォルダの中にいる時だけ。ルート表示中は戻り先が無いので対象外）
  const exitZone = getExitZoneRect();
  if (exitZone && clientY <= exitZone.bottom) {
    const parentFolder = folders.find(f => f.id === currentFolderId);
    const parentId = parentFolder ? (parentFolder.parentId ?? null) : null;
    const dragKey = dragEl.dataset.key;
    let ok = true;
    if (dragKey.startsWith('folder:')) {
      ok = canMoveFolderTo(dragKey.slice('folder:'.length), parentId);
    } else {
      const deckId = resolveDeckIdFromDragKey(dragKey);
      ok = deckId ? canMoveDeckTo(deckId, parentId) : true;
    }
    if (ok) {
      applyHoverTarget(exitZone.el, parentId);
      return;
    }
  }

  // dragEl自身が指の真下にあるとelementFromPointがそれを拾ってしまうため、
  // 判定中だけ一時的にpointer-eventsを外して「透明」にする
  const prevPE = dragEl.style.pointerEvents;
  dragEl.style.pointerEvents = 'none';
  const under = document.elementFromPoint(clientX, clientY);
  dragEl.style.pointerEvents = prevPE;

  const folderCard = under ? under.closest('.folder-card') : null;
  let targetFolderId = null;

  if (folderCard && folderCard.parentElement === grid && folderCard !== dragEl) {
    const fid = folderCard.dataset.key.slice('folder:'.length);
    const dragKey = dragEl.dataset.key;
    // 掴んでいるのがフォルダで、その移動先が自分自身／自分の子孫フォルダの場合は
    // 開けない（無限ループ・不正な階層構造の防止。canMoveFolderToで判定）
    if (dragKey.startsWith('folder:')) {
      const draggedFolderId = dragKey.slice('folder:'.length);
      if (canMoveFolderTo(draggedFolderId, fid)) targetFolderId = fid;
    } else {
      const deckId = resolveDeckIdFromDragKey(dragKey);
      if (!deckId || canMoveDeckTo(deckId, fid)) targetFolderId = fid;
    }
  }

  if (targetFolderId) {
    applyHoverTarget(folderCard, targetFolderId);
  } else {
    clearHoverFolder();
  }
}
```
- `document.elementFromPoint(x, y)`で「指の真下に今何が表示されているか」を調べます。ただし、そのままだと掴んでいるカード自身（指のすぐ下にあるので）が拾われてしまうため、判定する一瞬だけ`pointerEvents = 'none'`（このカードはクリック・タップの対象外、と一時的に無効化する）にして、その裏にある本当のフォルダを検出できるようにしています。
- 見つかったフォルダが、今掴んでいる項目の移動先として許可されているか（`canMoveFolderTo`/`canMoveDeckTo`、[01_Cardmaker.js_その1_ログインとデータ管理.md](01_Cardmaker.js_その1_ログインとデータ管理.md)）も確認します。
- 「パンくず付近まで持ち上げたら親フォルダに戻れる」という逆方向の操作（`getExitZoneRect`）も同じ仕組みで実現しています。フォルダを「開く」動きも「出る（親に戻る）」動きも、結局は「今のフォルダから、目的のフォルダIDへ移動する」という同じ処理（次節の`autoOpenFolderDuringDrag`）で扱えるためです。

> **★ 2026/09/04 追記（バグ修正）**：次節`autoOpenFolderDuringDrag`のフォルダ分岐が
> サーバーへ送るリクエストに`guild_id`/`session_token`を含めておらず、
> サーバー側の`require_login_json`に「missing guild_id」で常に弾かれていた
> （＝フォルダをドラッグして別フォルダへ入れても、ローカルの見た目上は移動した
> ように見えるのに、実際にはサーバーへ一度も反映されていなかった）。デッキメニュー
> の「移動」（`selectMoveTarget`）側は元々`guild_id`/`session_token`を送っていて
> 無事だったため、この2つの経路で挙動が食い違っていた形。次節のコードにこの
> 2フィールドを追加して修正した。

> **★ 2026/09/05 追記（受け皿の変更）**：`getExitZoneRect()`が参照する対象が
> パンくずリスト（`#folder-breadcrumb`、±16pxの余白）から、専用の受け皿バー
> （`#folder-exit-dropzone`、`Cardmaker.html`に新設）へ変わった。この受け皿は
> `.topbar`の直下・`.cm-scroll-body`の外に置かれた「ドラッグ中かつフォルダの
> 中にいる間だけ表示される」大きめのバーで、`beginDrag()`/`autoOpenFolderDuringDrag()`
> （フォルダの出入りで`currentFolderId`が変わるたび）/`endDrag()`から呼ばれる
> `updateFolderExitDropzoneVisibility()`が表示・非表示を管理する。パンくず（狭く、
> スクロールで見えなくなることもある）に正確に指を重ねる必要があったのを、
> 画面上部いっぱいの当てやすいスペースに変えたことで、「フォルダから出す」操作の
> 成功率を上げる狙い。`getExitZoneRect()`の戻り値の形（`{top, bottom, el}`）自体は
> 変わっていないので、`checkHoverFolder`側の呼び出し方には変更が無い。

```js
function applyHoverTarget(el, targetFolderId) {
  if (hoverFolderEl === el) return;
  clearHoverFolder();
  hoverFolderEl = el;
  hoverFolderEl.style.outline = '3px solid #3b82f6';
  hoverFolderEl.style.outlineOffset = '-3px';
  hoverFolderTimer = setTimeout(() => { hoverFolderTimer = null; autoOpenFolderDuringDrag(targetFolderId); }, HOVER_OPEN_MS);
}
```
- 同じフォルダの上に留まり続けている間だけ青い枠線でハイライトし、0.65秒（`HOVER_OPEN_MS`）経過したら実際にそのフォルダを開く処理を呼びます。対象から外れたら（`clearHoverFolder`）タイマーとハイライトをリセットします。

> **★ 2026/09/05 追記（見失いへの猶予）**：「フォルダ同士を重ねても中に移動
> できない（ことがある）」という報告を受けて調査したところ、指先のわずかな
> 震え・`elementFromPoint`の一瞬の誤検出などでフォルダの真上から一瞬だけ外れると、
> `checkHoverFolder`の`else`分岐がその場で`clearHoverFolder()`を呼び、
> `hoverFolderTimer`（0.65秒の判定タイマー）ごとリセットしてしまっていた。
> 指を完全に静止させ続けるのは難しいため、これが起き続けると**タイマーが
> 一度も完了せず、いつまで経ってもフォルダが開かない**ように見える。
> 対策として、対象を見失っても即座には`clearHoverFolder()`せず、
> `HOVER_MISS_GRACE_MS`（200ms）だけ待ってから消す`scheduleClearHoverFolder()`
> を新設し、`checkHoverFolder`の`else`分岐をこちらに差し替えた。猶予時間内に
> `applyHoverTarget`が再び呼ばれれば（＝対象を再検出できれば）、保留中の
> クリア予約はキャンセルされ、進行中の0.65秒タイマーは途切れずに継続する。

---

## 5. 掴んだままフォルダに入る：`autoOpenFolderDuringDrag(targetFolderId)`（2672〜2763行）

これが、この機能の中で最も複雑な処理です。「掴んでいる項目を実際にそのフォルダの中へ移動させ、かつ、そのフォルダを開いた状態にして、しかもドラッグ自体は途切れさせない」という3つを同時にやろうとしています。

1. **データを実際に移動する**：フォルダなら`parentId`を、デッキなら`folderId`を書き換え、それぞれサーバーにも反映します（デッキの場合は`loadDeckCardsWithRecovery`で最新カードを取ってから移動、フォルダなら`楽観的にローカルへ反映してから`サーバーに送る、という違いがあります）。
   > **★ 2026/09/04 追記（バグ修正）**：このフォルダ側の`save_folder`へのリクエストが
   > `guild_id`/`session_token`を含めておらず、サーバー側の`require_login_json`に
   > 「missing guild_id」で常に弾かれていた（＝ローカルの見た目上は移動できたように
   > 見えても、実際にはサーバーへ一度も反映されていなかった）。前節末尾の追記も参照。
   > フォルダメニューの「移動」（[02_Cardmaker.js_その2_一覧画面とフォルダ操作.md](02_Cardmaker.js_その2_一覧画面とフォルダ操作.md)の`selectMoveTarget`）側は
   > 元々`guild_id`/`session_token`を送っていたため、この2つの経路だけ挙動が食い違っていた。
2. **今のフォルダを切り替える**：`currentFolderId = targetFolderId`とし、`renderDeckListUI()`で画面を作り直します。ただし通常は`cmListDragActive`のガードで再描画がスキップされるため、ここだけ**一時的にガードを外して**（`cmListDragActive = false`にしてから呼び、終わったら元に戻す）強制的に再描画させています。
3. **ドラッグを継続する**：フォルダを切り替えると`#deck-grid`の中身が丸ごと新しく作り直されるため、それまで掴んでいた`dragEl`（DOM要素）はもう画面から消えています。そこで、同じ`data-key`を持つ**新しく描画された要素**を探し直し（`getItems().find(...)`）、それに対してもう一度`beginDrag`を呼び直すことでドラッグを継続します。

```js
// ★ 追加：フォルダを切り替える直前の「見た目の位置」と、その時点の最新の指の位置を
//   ここで（＝各種await完了後の最新の状態で）確定させる。デッキ読み込み待ちなどの
//   非同期処理中に指が動いていた場合でも、ここで最新値を使うことでズレを防ぐ。
const oldVisualTop = dragEl.getBoundingClientRect().top;
const resumeClientX = lastClientX;
const resumeClientY = lastClientY;

// フォルダを開く
currentFolderId = targetFolderId;

// ★ ドラッグ中は renderDeckListUI() が丸ごとスキップされるため、ここだけ
//   一時的にガードを外して再描画し、開いたフォルダの中身を表示する。
const wasDragActive = cmListDragActive;
cmListDragActive = false;
renderDeckListUI();
cmListDragActive = wasDragActive;

const body = document.querySelector('#screen-list .cm-scroll-body');
if (body) body.scrollTop = 0;

// ★ 再描画で古いdragEl要素はDOMから外れてしまったので、新しく描画された
//   同じ項目（data-keyで特定）を探し直し、そのままドラッグを継続する。
const newEl = getItems().find(it => it.dataset.key === key) || null;
if (newEl) {
  const newNaturalTop = newEl.getBoundingClientRect().top;
  const initialDy = oldVisualTop - newNaturalTop;
  beginDrag(newEl, resumeClientY, initialDy);
  lastClientX = resumeClientX;
  try { newEl.setPointerCapture(pressPointerId); } catch (_) {}
} else {
  // 万一見つからなければドラッグ状態を安全に終了させる
  dragEl = null;
  cmListDragActive = false;
}
```
- 新しく描画された要素は「開いたフォルダの一覧の中の自然な位置」に置かれるだけで、指の位置とは無関係な場所に表示されます。そこで、フォルダ切り替え前の見た目の位置（`oldVisualTop`）と、切り替え後の自然な位置（`newNaturalTop`）の差を`initialDy`として`beginDrag`に渡すことで、**カードが指の位置からずれずにそのまま連続して見える**ように調整しています（3.1節で説明した`beginDrag`の`initialDy`引数がここで使われます）。

```js
try { newEl.setPointerCapture(pressPointerId); } catch (_) {}
```
- フォルダの切り替えで要素が作り直されると、それまで張っていたポインターキャプチャ（「この指の動きはこの要素が責任を持って受け取る」という設定）も外れてしまいます。コメントによれば、これを張り直さないと、指がパンくず付近（元の`#deck-grid`の外）に出た瞬間から`pointermove`/`pointerup`が届かなくなり、`endDrag()`が一切呼ばれないまま`touch-action: none`が要素に残り続けて「スクロールも何もできなくなる」不具合につながる、とのことです。

---

## 6. スクロールとの共存（2881〜2979行）

`onPointerMove`は、状況によって次の3通りのいずれかの処理に分岐します：
1. すでにドラッグ中なら、`moveDrag`を呼んでカードを動かす。
2. 長押し判定がキャンセルされたあとの「手動スクロール」中なら、スクロール量を溜めておいて次のアニメーションフレームでまとめて反映する（3.3節の自動スクロールと同じ「まとめて1回で反映する」考え方。コメントによれば、`touchmove`のたびに直接`scrollTop`を書き換えるとカクカクして見えるため、これを避けています）。
3. 長押し判定がまだ確定していない間に、指が`MOVE_CANCEL_PX`（10px）を超えて動いたら、「これはスクロールしたいのだ」と判断して長押し判定をキャンセルします（`cancelPress()`）。

```js
function onNativeTouchMove(e) {
  if (dragEl || manualScrollActive || touchActionItem) {
    e.preventDefault();
  }
}
grid.addEventListener('touchmove', onNativeTouchMove, { passive: false });
```
- 最後の保険として、素の`touchmove`イベントも監視しています。コメントによれば、`touch-action`の設定だけに頼ると、実機（特にAndroid Chrome）ではタイミングによってブラウザのジェスチャー判定に間に合わず、ネイティブのスクロールが始まってしまうことがあるためです。ドラッグ中・手動スクロール中・touch-action適用済みのいずれかであれば、`e.preventDefault()`でネイティブスクロールの発生自体を確実に止めます。逆にそれ以外（まだ様子見中）のときは何もしないことで、普通のスワイプ操作を邪魔しないようにしています。

---

## 7. 後片付けの二重の保険（2936〜2953行）

```js
function onPointerUp(e) {
  if (dragEl && e.pointerId === pressPointerId) { endDrag(); cancelPress(); resetTouchAction(); return; }
  if (e.pointerId === pressPointerId) { cancelPress(); resetTouchAction(); return; }
  if (e.pointerId === manualScrollPointerId) { resetTouchAction(); }
}
grid.addEventListener('pointerup', onPointerUp);
grid.addEventListener('pointercancel', onPointerUp);
document.addEventListener('pointerup', onPointerUp);
document.addEventListener('pointercancel', onPointerUp);
```
- `pointerup`/`pointercancel`は`#deck-grid`だけでなく、`document`全体（ページ全体）にも同じ関数を登録しています。コメントによれば、フォルダの自動オープン処理の直後は指の位置が一時的に`#deck-grid`の外（パンくず付近）にあることがあり、万一ポインターキャプチャの張り直しがうまくいかない端末があっても、「指が離れた」というイベント自体は必ずページ全体まで伝わってくるので、そこで拾って確実に後片付け（`touch-action`を元に戻す等）を行う、という二重の安全策です。

---

続きは[06_Cardmaker.js_その6_カード編集と学習データ同期.md](06_Cardmaker.js_その6_カード編集と学習データ同期.md)で、カード編集モーダル・サーバーとの学習データ同期を解説します。
