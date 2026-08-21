# 予定管理ページ（`index.html` + `Plan.js`）— コード解説

対象ファイル：`bot.1istudy.web/index.html`（329行）、`bot.1istudy.web/Plan.js`（1216行）。
このページはドロワーメニューの一番上「予定管理」で、宿題・提出・テストなどの**予定の一覧・追加・編集・削除**と、**変更履歴（ログ）タブ**を持っています。

プログラミングの前提知識が無くても読めるように書いています。専門用語が出てくるところは、その場で簡単な説明を添えるか、下の「0. ミニ用語辞典」を先に読んでおくと迷いにくいです。コードそのものも引用しているので、実物と照らし合わせながら読めます。

---

## 0. 読む前に：ミニ用語辞典

このページ以降、繰り返し出てくる言葉をあらかじめまとめておきます。「知ってるところは飛ばして、知らない言葉が出てきたらここに戻ってくる」という読み方がおすすめです。

- **JS（JavaScript）**：ブラウザ上で動くプログラム言語。「ボタンを押したら何が起きるか」「画面に何を表示するか」を全部これで書く。
- **HTML**：画面の骨組み（ボタン・入力欄・見出しなど）を書くための言語。`<button>ボタン</button>`のように「タグ」で囲んで部品を作る。
- **CSS**：HTMLの部品に色・大きさ・配置などの見た目を付ける言語。
- **関数（function）**：「まとまった処理に名前を付けたもの」。例えば`loadPlans()`は「予定を読み込む」という一連の処理ひとまとまりに`loadPlans`という名前を付けたもの。名前を書けば（＝**呼び出せば**）中の処理が実行される。料理のレシピに名前を付けて「〇〇作って」で済ませるようなイメージ。
- **変数・定数**：値を入れておく箱。`let`で作ったものは後で中身を書き換えられる（変数）、`const`で作ったものは書き換えられない（定数）。
- **配列**：値を順番に並べたリスト。`[1, 2, 3]`のように書く。「1件目」「2件目」のように番号でアクセスできる。
- **オブジェクト**：`{名前: 値, 名前: 値}`のように、名前（キー）と値のペアをまとめたもの。「日付は2026-08-20、科目は数学」のような1件のデータを表すのによく使う。
- **JSON**：オブジェクトや配列を文字列として保存・通信するための共通フォーマット。このサイトのサーバーとのやり取りは基本的に全部JSON。
- **DOM／要素**：画面に表示されているHTMLの部品1つ1つをJSから見たときの呼び方。`document.getElementById('foo')`は「id="foo"が付いた部品を画面から探してくる」という意味。
- **id / class**：HTMLの部品に付けられる名札。`id`はページ内で1つだけの部品を指すための名前、`class`は複数の部品に共通で付けて「まとめて同じ見た目・同じ扱いにする」ための名前。JSは`getElementById`で`id`から、`querySelectorAll`で`class`などから部品を探す。
- **イベント／`onclick`**：「クリックされた」「ページが読み込まれた」のような「何かが起きたタイミング」のこと。`onclick="関数名()"`と書いておくと、そのボタンがクリックされた瞬間に指定した関数が実行される。
- **API**：サーバー側が用意している「こう聞けばこう答える」という窓口。例えば「`/list_schedule`というAPIに聞けば予定の一覧を返してくれる」といった具合。
- **`fetch`**：JSからAPI（サーバー）に「これ送るので、答えをください」とお願いする命令。ネットワーク越しの問い合わせなので、答えが返ってくるまで少し時間がかかる。
- **非同期処理／`async`・`await`**：`fetch`のようにサーバーからの返事を待つ処理は、待っている間ページ全体が固まらないよう「裏で待っておいて、返事が来たら続きをやる」という書き方をする。`async function`は「この関数の中には待つ処理がありますよ」という印、`await`は「ここで実際に返事が来るまで（この関数の中だけ）待つ」という意味。難しく感じたら、「サーバーに聞きに行って、答えが返ってくるまで待つ場所」くらいの理解でいったん進んで大丈夫。
- **`try { } catch(e) { }`**：「`try`の中でエラーが起きても、プログラム全体が止まらないように`catch`の中で拾って処理を続ける」という仕組み。通信エラーなど「起こりうる失敗」に備えるためによく使われる。
- **`localStorage`／`sessionStorage`**：ブラウザがその端末に保存できる、ちょっとしたデータ置き場。`localStorage`はブラウザを閉じても消えない（ログイン情報の保存に使用）、`sessionStorage`はそのタブを閉じると消える（一時的な情報の受け渡しに使用）。
- **テンプレート文字列 `` `${変数}` ``**：バッククォート `` ` `` で囲んだ文字列の中に`${ }`で変数を埋め込める書き方。`` `こんにちは${name}さん` ``なら`name`の中身がそのまま文字列に差し込まれる。
- **エスケープ／XSS**：生徒が自由入力した文字列をそのまま画面に表示すると、その中に紛れ込ませた悪意あるプログラムが実行されてしまう危険がある（これを**XSS**という攻撃と呼ぶ）。それを防ぐため、表示前に`<`や`>`のような特殊な記号を無害な文字に変換する処理を「**エスケープ**」と呼ぶ。
- **`?.`（オプショナルチェーン）**：`a?.b`は「`a`が存在すればその`b`を、`a`が無ければエラーにせず`undefined`（値が無いことを表す値）を返す」という書き方。安全策として使われる。
- **`? :`（三項演算子）**：`条件 ? Aのとき の値 : Bのとき の値`という1行で書ける if文の短縮形。
- **正規表現**：「文字列の中からこういうパターンを探す/置き換える」というルールを記号で書く仕組み。例えば`/^【(.+?)】/`は「文字列の先頭にある『【〜】』の部分」を表す。
- **ハッシュ値**：文字列やデータから計算される「指紋」のような短い値。同じ中身なら必ず同じハッシュ値になり、中身が1文字でも変わればハッシュ値も変わるので、「2つのデータが同じ内容かどうか」を高速に比較するのに使われる。
- **SSE（Server-Sent Events）／`EventSource`**：サーバー側からブラウザへ「変化があったよ」と一方的に知らせてもらうための、接続しっぱなしの通信方式。普通の`fetch`は「聞きに行かないと答えが来ない」が、SSEは「向こうから声をかけてくれる」イメージ。

---

## 1. `index.html` の構造（ざっくり）

HTMLの細かいタグは全部は追わず、「どこに何のブロックがあるか」だけ表にまとめます。

| 行 | ブロック | 役割 |
|---|---|---|
| 4-9 | `<head>` | 文字コード・画面サイズの設定（iPhoneのノッチに画面がめり込まないための設定含む）・タイトル・アイコン・デザイン用CSSの読み込み |
| 12-32 | `#js-fail-fallback` | JSが壊れていても画面が真っ白/操作不能にならないための保険（詳細は1.1節） |
| 54-90 | `.drawer` / `.drawer-overlay` | 左から出てくるハンバーガーメニュー本体。各ページへのリンク＋一番下に「今ログインしている人」の表示欄 |
| 96-105 | `.header` | ハンバーガーボタン・タイトル・「予定一覧/ログ」タブ切り替え・「今日」ボタン・絞り込みボタン |
| 108-139 | `.plan-main` | 「予定一覧＋絞り込みバー」の画面と「ログのタイムライン」の画面、2つの切り替え式画面 |
| 144-159 | FAB（右下の丸いボタン） | 押すと「追加/編集/削除」の3つの小さいボタンが飛び出す |
| 161-219 | `#modal-add` | 予定追加用のポップアップ（日付・科目・カテゴリ・内容・ポイント・備考の入力欄） |
| 221-283 | `#modal-edit` | 予定編集用のポップアップ（対象を一覧から選び、変えたい項目だけ入力） |
| 285-303 | `#modal-delete` | 予定削除用のポップアップ（対象を一覧から選び、確認してから削除） |
| 305-319 | `#modal-detail` | 予定をタップしたときに出る詳細表示のポップアップ（編集・削除ボタン付き） |
| 321-326 | JSファイルの読み込み | `Icons.js → SwipeGuard.js → Dialog.js → Dropdown.js → Plan.js（本体） → PendingDeleteCheck.js`の順で読み込む |

入力欄はほとんどが`id`という名札だけ付いた素のHTML部品（`<select>`や`<input>`）で、その中身を実際に作ったり、入力された値を取り出したりするのは全部後述の`Plan.js`がJSで行っています。ボタンには`onclick="関数名()"`という書き方でJSの関数が直接紐付けられています（少し古典的なやり方ですが、「このボタンを押すとこの関数が呼ばれる」とHTMLを見ただけで分かりやすい書き方です）。

### 1.1 JSが壊れたときの保険（12-49行）

```html
<div id="js-fail-fallback" ...>「読み込み中です…」＋ 4秒後に出る「そのまま操作する」ボタン ＋ フォーム</div>
<script>
(function () {
  setTimeout(function () { ... btn.style.display = ''; }, 4000);
  setTimeout(function () { ... el.remove(); }, 10000);
})();
</script>
```
- 普段は`Plan.js`の一番最後（1215行目）にある`hideLoadingFallback()`という処理が、「JSがちゃんと最後まで動きました」という合図としてこの「読み込み中…」表示をすぐに消します。
- もし何かの理由でJSファイルが読み込めなかったり、途中でエラーが起きて止まってしまった場合、その`hideLoadingFallback()`自体が実行されないままになってしまいます。そのための保険として、この`<script>`（他のJSファイルに一切頼らない、このページに直接書かれたプログラム）が、4秒後に「そのまま操作する」ボタンを出し、10秒後には表示を強制的に消します。
- `setTimeout(処理, 4000)`は「4000ミリ秒＝4秒後にこの処理を実行する」という命令です。
- 中の`<form method="POST" action="/api/report_problem">`は、JSを一切使わない**ブラウザ標準の機能だけで動くフォーム**です。JSが1行も動いていない状態でも、送信ボタンを押せば管理者に問題報告が届きます。

---

## 2. `Plan.js` 冒頭：ログインチェックとアカウント表示（1〜122行）

### 2.1 基本の設定値（6〜7行）
```js
const API_BASE = "/api/";
const GUILD_ID = "1509880344806162544";
```
- `API_BASE`：サーバーに問い合わせるときの宛先の先頭部分。すべての通信がこの`/api/`から始まる（実際のバックエンドの場所を隠す仕組みについては[[web裏側の記事]]を参照）。
- `GUILD_ID`：このBotが動いているDiscordサーバーを識別する番号。今は1つのサーバーだけの運用なので、決め打ちの値になっている。

### 2.2 ログイン情報の読み取り（13〜17行）
```js
const SESSION_KEY = 'sl_session';
const LOGIN_PATH = '/Login.html';
function getLoginSession() {
  try { return JSON.parse(localStorage.getItem(SESSION_KEY)); } catch { return null; }
}
```
- ログイン情報は、この端末の`localStorage`に`sl_session`という名前で保存されている想定です。
- `getLoginSession()`はそれを取り出してJSONとして読み取る関数。もし壊れていたり保存されていなかったりで読み取りに失敗しても（`try/catch`で）エラーにはせず、`null`（＝情報なし）を返すだけにしています。

### 2.3 ページを開いた瞬間のログイン確認（18〜24行）
```js
(function() {
  var s = getLoginSession();
  if (!s || !s.session_token) {
    sessionStorage.setItem('post_login_redirect', location.href);
    location.replace(LOGIN_PATH);
  }
})();
```
- `(function() { ... })();`という書き方は、「その場ですぐ1回だけ実行される処理」という意味（IIFEと呼ばれる書き方ですが、名前は覚えなくてOK）。`Plan.js`が読み込まれた瞬間に真っ先に実行されます。
- ログイン情報が無い、または`session_token`（ログイン中であることを証明する文字列）が無ければ、**今開いていたページのURLを一時的に控えてから**、ログインページへ強制的に移動させます。
- URLを控えておくのは、ログインし終わったあとに元のページへ自動で戻ってこられるようにするためです。
- `location.href = ...`ではなく`location.replace(...)`を使っているのがポイントで、これだとブラウザの「戻る」ボタンを押しても、この未ログインの状態には戻らずに済みます（履歴に残らない移動）。

### 2.4 変更操作の直前チェック（29〜37行）
```js
function requireLoginOrRedirect() {
  const s = getLoginSession();
  if (!s || !s.session_token) {
    sessionStorage.setItem('post_login_redirect', location.href);
    location.href = LOGIN_PATH;
    return null;
  }
  return s;
}
```
- 2.3節と似ていますが、こちらは「予定を追加・編集・削除するボタンを押した瞬間」にもう一度呼ばれる関数です（`submitAdd`/`submitEdit`/`submitDelete`という関数の一番最初で呼ばれています）。
- ページを開いたときはログインできていても、その後時間が経ってセッションが切れている可能性があるため、実際に変更を送る直前にもう一度確認する二重チェックです。
- ログイン中ならログイン情報をそのまま返し、呼び出し元はそれを使って以降の通信を組み立てます。ログインしていなければ`null`を返し、呼び出し元はそこで処理を中断します。

### 2.5 ドロワー下部のアカウント表示（41〜117行）
`renderDrawerAccount()`は、ドロワーメニューの一番下にある「今ログインしている人」の表示を作る関数です。ここでは`innerHTML = "<div>...</div>"`のような文字列の組み立てではなく、`document.createElement`という**部品を1つずつJSで作って組み立てる方法**を使っています（あとで出てくるXSS対策と同じ理由です。ユーザーが入力した文字列を扱うときは、この作り方の方が安全）。

- 42〜45行：表示先の部品（`#drawer-account`）を探す。無ければ何もしない。前回の中身と開閉状態をいったんリセット。
- 46〜53行：ログイン情報（`session_token`と`nickname`）が両方揃っていなければ、「ログインしていません」というリンクだけを表示して終わり。
- 56〜65行：ログイン中なら、丸いアバターの部品を作る。中身はニックネームの最初の2文字を大文字にしたもの。アカウントごとに割り当てられた色があれば、それを背景色・文字色として使う。
- 67〜79行：ニックネームと（あれば）学籍番号を並べた部品を作る。
- 81〜84行：開閉を示す矢印（`›`）を追加。
- 86〜89行：このボタン全体をクリックすると、メニューの開閉状態が反転する。`e.stopPropagation()`は「このクリックを、後で説明する『外側クリックで閉じる』処理には伝えない」という命令（そうしないと、開いた直後に自分自身のクリックで即座に閉じてしまう）。
- 91〜100行：ミニメニュー内の「⚙️ アカウント設定」リンク。実際の設定画面はこのページには無く、`/StudyLog.html?openAccount=1`という別ページのURLに飛ぶことで、勉強ログページ側に用意されている設定画面を開く仕組みになっている。
- 102〜113行：「🚪 ログアウト」ボタン。押すと確認ダイアログ（`showAppConfirm`、自作のOKキャンセル確認画面）を出し、OKが押されたら`localStorage`からログイン情報を消してログインページに移動する。
- 115〜116行：ここまで作った部品を画面に追加する。
- 118〜121行：画面のどこかがクリックされるたびに、「そのクリックがこのアカウント表示欄の外側で起きたか」をチェックし、外側なら開いているミニメニューを閉じる（よくある「ポップアップの外をクリックしたら閉じる」という動き）。
- 122行：ページが開いたタイミングで、この関数を1回実行する。

### 2.6 表示を安全にするための関数（124〜132行）
```js
function esc(s) {
  return String(s == null ? '' : s)
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}
```
- `esc`は「エスケープする」の略で、文字列の中にある`&`・`<`・`>`・`"`という記号を、画面上ではそのまま見えるけれどプログラムとしては実行されない別の書き方に置き換える関数です。
- `&`を最初に変換しているのには理由があります。後から`<`を`&lt;`に変換したときに、その中の`&`をもう一度エスケープしてしまうと表示がおかしくなるため、必ず`&`を最初に処理する必要があります。
- なぜこれが必要かというと、生徒が自由に入力できる予定の内容（`content`）を、もし何のエスケープもせず直接画面に差し込んでしまうと、`<img src=x onerror="悪いプログラム">`のような文字列を1件予定として登録するだけで、その予定を見た全員のブラウザで勝手にプログラムが実行されてしまう、という深刻な問題（**保存型XSS**と呼ばれる攻撃）が実際に過去に起きていたためです。以降、画面に予定の内容を表示する箇所は必ずこの`esc()`を通しています。

---

## 3. 決まった値と、状態を覚えておくための変数（134〜210行）

```js
const POINT_CATEGORIES = ['提出', '宿題'];
const POINT_OPTIONS = [3, 5, 10, 15];
const NOTE_SEP = '\n📝備考：';
```
- `POINT_CATEGORIES`：この2つのカテゴリ（提出・宿題）を選んだときだけ、ポイントの入力が必須になる。
- `POINT_OPTIONS`：ポイントの選択肢（3・5・10・15pt）。
- `NOTE_SEP`：予定の「本文」と「備考」を1本の文字列としてつなげるときの目印。Discord Botがこの文字列をそのままDiscordに投稿する仕組みなので、別々の項目に分けず、人が読んでも分かる区切り文字（`\n📝備考：`＝改行して「📝備考：」と書く）でくっつけている。

```js
let plans = [], channels = [], calState = {}, editTarget = null, delTarget = null, detailTarget = null;
```
これらは「今のところ何が起きているか」を覚えておくための変数です。
- `plans`：サーバーから取ってきた予定のリスト（画面に表示する本体データ）。
- `channels`：科目（Discordのチャンネル）の一覧。
- `calState`：自作カレンダーが「今どの年月を表示しているか」「どの日が選ばれているか」を覚えておく場所。
- `editTarget`／`delTarget`：編集・削除の対象として今選ばれている予定を指す文字列。
- `detailTarget`：詳細画面で今表示中の予定を指す文字列。

**164〜172行：段階的に読み込むための変数（ページング）**
```js
const PAST_PLANS_PAGE_SIZE = 50;
const LOGS_PAGE_SIZE = 50;
let pastPlansOffset = 0, pastPlansHasMore = false, pastPlansLoading = false;
let logsData = [], logsOffset = 0, logsHasMore = false, logsLoading = false;
```
- 予定は自動的には消えない仕組みになっていて、時間が経つほど過去分がどんどん増えていきます。ページを開くたびに全部読み込んでいると年々遅くなってしまうため、「未来の予定は全部先に読み込み、過去の予定は50件ずつ、必要になった分だけ追加で読み込む」という段階読み込みの方式にしています。
- そのために「今どこまで読み込んだか（オフセット）」「まだ続きがあるか」「今読み込み中か（二重に読み込みが走らないようにするための印）」を覚えておく変数です。変更履歴（ログ）タブも同じ考え方で50件ずつ読み込みます。

**174〜179行：選択中のチップの状態**
```js
let selectedPoints = { add: null, edit: null };
let filterSubject = 'all', filterCat = 'all';
```
- `selectedPoints`：追加・編集それぞれの画面で、今どのポイント（3/5/10/15pt）が選ばれているか。
- `filterSubject`／`filterCat`：絞り込みバーで今選ばれている教科・カテゴリ（`'all'`は「絞り込みなし」を意味する）。

---

## 4. 日付の扱いと、重複を防ぐ処理（181〜209行）

```js
function todayLocalStr() {
  const now = new Date();
  const y = now.getFullYear();
  const m = String(now.getMonth() + 1).padStart(2, '0');
  const d = String(now.getDate()).padStart(2, '0');
  return `${y}-${m}-${d}`;
}
```
- 今日の日付を`2026-08-20`のような文字列にして返す関数です。
- JSには日付から自動でこの文字列を作る`toISOString()`という命令もあるのですが、それは世界標準時（イギリス基準）で計算されるため、日本時間の深夜0時前後だと日付が1日ずれてしまうことがあります。それを避けるため、あえて`getFullYear()`/`getMonth()`/`getDate()`という「今この端末が設定している時刻（＝日本時間）」を基準にした部品を組み合わせて自分で文字列を作っています。
- `padStart(2, '0')`は「2桁になるよう、足りない分の先頭に`0`を詰める」という命令（例：`8`→`08`）。

```js
function dedupePlans(list) {
  const seen = new Set();
  const out = [];
  for (const p of list) {
    const key = `${p.date}/${p.subject}${p.content}`;
    if (seen.has(key)) continue;
    seen.add(key);
    out.push(p);
  }
  return out;
}
```
- `Set`は「同じ値を2回入れても1つしか残らない」特殊な入れ物です。
- 予定1件ごとに「日付／科目内容」という文字列を作り、それを`Set`に記録しながら、「まだ見たことがなければ結果に追加する」という処理を繰り返すことで、**重複した予定を取り除いています**（先に来たものを残す方式）。
- なぜこれが必要かというと、もしサーバー側の仕組みがうまく動いていない状態（例えば新しいプログラムがまだ全部反映されていないタイミング）だと、「未来の予定」と「過去の予定」をそれぞれ問い合わせたときに、両方とも本来の分担を超えて全件を返してしまうことがあります。それをそのまま画面に足し合わせると、同じ予定が2回表示されてしまう事故が起きるため、そういうデータを合体させる箇所では必ずこの重複除去を通すようにしています。

---

## 5. ページを開いたときの起動処理（211〜218行）

```js
window.addEventListener('load', () => {
  loadChannels();
  loadPlans();
  prefetchOtherPages();
});
```
- `window.addEventListener('load', 処理)`は「ページの読み込みが（画像なども含めて）全部完了したら、この処理を実行する」という意味です。
- ここでは3つのことを一気に始めます：科目一覧の取得、予定一覧の取得、他ページのファイルを裏で先読みしておく処理（22節で説明）。3つとも「終わるのを待たずに」同時にスタートします。

---

## 6. 「予定一覧」と「ログ」タブの切り替え（220〜262行）

```js
function switchPlanView(v) {
  document.querySelectorAll('.view-btn').forEach((b, i) => {
    b.classList.toggle('active', ['plan','log'][i] === v);
  });
  document.getElementById('plan-sub-plan').classList.toggle('active', v === 'plan');
  document.getElementById('plan-sub-log').classList.toggle('active',  v === 'log');
  // （中略：絞り込みバーを閉じる処理など）
}
```
- ヘッダーの「予定一覧」「ログ」という2つのタブボタンのうち、今選ばれている方だけに`active`（アクティブ＝選択中を示すデザイン用の目印）を付けます。`classList.toggle('active', 条件)`は「条件が本当（true）なら`active`という目印を付ける、嘘（false）なら外す」という命令です。
- 対応する画面（予定一覧の画面／ログの画面）についても同様に、選ばれている方だけ表示状態にします。
- 231〜238行：ログタブに切り替えるときは、絞り込みバー（ログには関係ない機能なので）を強制的に閉じます。
- 240〜242行：ログタブに切り替わった瞬間、ログのデータを毎回サーバーから読み直します（一度読んだ内容を使い回さない設計）。
- 244〜246行：予定タブに戻ってきたときは、画面が描き終わるのを少しだけ待ってから（0.05秒後）、今日の予定の位置まで自動でスクロールします。

```js
function scrollLogsTop() { /* ログの先頭までスクロール */ }
function onTodayButton() {
  const isLog = document.getElementById('plan-sub-log').classList.contains('active');
  if (isLog) scrollLogsTop(); else scrollToToday();
}
```
- ヘッダーの「今日」ボタンは、今どちらのタブを見ているかによって挙動が変わる共通ボタンです：ログタブを見ているならログの先頭へ、予定タブを見ているなら今日の日付の位置へスクロールします。

---

## 7. サーバーと通信するための共通の窓口（264〜281行）

```js
async function api(path, opts = {}) {
  const session = getLoginSession();
  const headers = Object.assign(
    { "Content-Type": "application/json" },
    (session && session.session_token) ? { "Authorization": "Bearer " + session.session_token } : {},
    opts.headers || {}
  );
  const res = await fetch(API_BASE + path.replace(/^\/+/, ''), { ...opts, headers });
  return res.json();
}
```
- このページのサーバーとのやり取りは、すべてこの`api()`という関数を経由します。呼び出し側は「どのAPIに」「どんな内容で」聞きたいかだけを渡せば、あとはこの関数がまとめて面倒を見てくれます。
- `headers`（通信の「宛名書き」のようなもの）を組み立てる部分では、「基本の設定（JSON形式で送ります、という宣言）」→「ログイン中ならログイン証明（`Authorization`）を自動で追加」→「呼び出し側が個別に指定したものがあればそれで上書き」という順番で合体させています。
- ログイン中は**自動で**このログイン証明を付ける設計になっているのがポイントです。以前は各呼び出し箇所が個別に付け忘れることがあり、本来ログインしないと見られないはずの予定データが、うっかり誰でも見られてしまう抜け穴になっていたことがある、という経緯がコメントに書かれています。
- 最後に`fetch`でサーバーに実際の通信を送り、返ってきた答えをJSONとして読み取って（`res.json()`）呼び出し元に返します。

---

## 8. 科目（チャンネル）一覧の読み込み（283〜293行）

```js
async function loadChannels() {
  try {
    const data = await api(`/channels?guild_id=${GUILD_ID}`);
    channels = data.ok ? data.channels : [];
  } catch(e) { channels = []; }
  renderChannelOptions();
  renderSubjectFilterChips();
}
```
- サーバーに「科目の一覧をください」と問い合わせ、成功していれば結果を`channels`に保存します。通信に失敗した場合や、サーバーが「うまくいかなかった」（`data.ok`が偽）と答えた場合は、画面が壊れないよう空のリストにしておきます。
- 取得した科目一覧を使って、追加・編集モーダルの科目選択欄（`renderChannelOptions`）と、絞り込みバーの教科チップ（`renderSubjectFilterChips`）の両方を作り直します。

---

## 9. 予定一覧を少しずつ読み込む仕組み（295〜367行）

```js
async function loadPlans() {
  document.getElementById('plan-loading').style.display = 'block';
  document.getElementById('plan-content').innerHTML = '';
  plans = []; pastPlansOffset = 0; pastPlansHasMore = false;

  try {
    const data = await api(`/list_schedule?guild_id=${GUILD_ID}&scope=future`);
    plans = data.ok ? dedupePlans(data.plans) : [];
  } catch(e) { plans = []; }
  document.getElementById('plan-loading').style.display = 'none';
  renderPlans();
  scrollToToday();

  await loadMorePastPlans();
}
```
- 「読み込み中…」の表示を出し、これまでのデータをいったんリセット。
- まず「これからの予定」（`scope=future`というオプションを付けて、未来分だけ）を先にサーバーから取得し、届いたらすぐ画面に表示・今日の位置までスクロールします。これは「全部そろうまで待たせず、まず見える分だけ早く見せる」という体感速度重視の作りです。
- そのあとで`loadMorePastPlans()`（下記）を呼び、過去分の1ページ目（50件）を自動で追加取得します。

```js
async function loadMorePastPlans() {
  if (pastPlansLoading) return;
  pastPlansLoading = true;
  const btn = document.getElementById('plan-load-more-btn');
  if (btn) setLoading(btn, '読み込み中…', true);

  try {
    const data = await api(`/list_schedule?guild_id=${GUILD_ID}&scope=past&offset=${pastPlansOffset}&limit=${PAST_PLANS_PAGE_SIZE}`);
    if (data.ok) {
      plans = dedupePlans(plans.concat(data.plans));
      pastPlansOffset += data.plans.length;
      pastPlansHasMore = !!data.has_more;
    } else { pastPlansHasMore = false; }
  } catch(e) { pastPlansHasMore = false; }
  pastPlansLoading = false;
  renderPlans();
}
```
- すでに読み込み中なら何もしない（同時に2回動いてデータが変になるのを防ぐ）。「もっと読み込む」ボタンがあれば、押せない・読み込み中の見た目に変える。
- `scope=past`（過去分）と、「どこから」「何件」を指定してサーバーに問い合わせ、届いたデータを今の一覧に足し合わせます（`dedupePlans`で重複除去も同時に行う）。
- 次にどこから読み込めばいいかの位置（`pastPlansOffset`）を進め、サーバーが「まだ続きがある」と教えてくれたかどうか（`has_more`）を覚えておきます。
- 最後に画面を再描画（この中で「もっと読み込む」ボタンを出すかどうかも判断されます）。

```js
function scrollToToday() {
  // 今表示されている予定の中から、スクロール先の日付を決める
  // ①今日ぴったりの予定があればそこ
  // ②無ければ一番近い未来の日
  // ③未来の予定が1件も無ければ、一番新しい過去の日
  // 該当する日付のかたまりを画面上で探し、ヘッダーに隠れない位置までスクロールする
}
```
- 実際のコードでは、`Array`の`filter`（条件に合うものだけ残す）や`sort`（並べ替え）といった命令を組み合わせて上記の判定をしています。

---

## 10. 変更履歴（ログ）タブの読み込み（369〜414行）

`loadLogs()`と`loadMoreLogs()`は9節の予定版とほぼ同じ考え方の関数で、問い合わせ先が`/list_logs`、保存先の変数が`logsData`に変わっただけです。

- `loadLogs()`：状態をリセット →「新しい方から50件」を取得 → 画面に描画。
- `loadMoreLogs()`：二重に走らないようにチェック → 続きの50件を`logsOffset`（今どこまで読んだか）から取得して足し合わせる → 画面に再描画。

（予定と違って「未来/過去」の分け方は無く、単純に新しい順から必要な分だけ取っていく方式です。）

---

## 11. 絞り込み（フィルタ）の仕組み（416〜468行）

```js
function getFilteredPlans() {
  return plans.filter(p => {
    if (filterSubject !== 'all' && p.subject !== filterSubject) return false;
    if (filterCat !== 'all') {
      const { cat } = parsePlanContent(p.content);
      if (filterCat === 'hw') { if (cat !== '提出' && cat !== '宿題') return false; }
      else { if (cat !== filterCat) return false; }
    }
    return true;
  });
}
```
- `plans.filter(条件)`は「配列の中から、条件に一致するものだけを残した新しい配列を作る」という命令です。
- ここでは、「今選ばれている教科（`filterSubject`）に一致するか」「今選ばれているカテゴリ（`filterCat`）に一致するか」の両方をチェックして、両方通ったものだけを残します。
- カテゴリの絞り込みで「提出・宿題」という1つのチップ（内部的には`'hw'`という値）が選ばれているときだけ特別扱いで、「提出」でも「宿題」でもどちらでも通す、というOR条件（どちらか一方でよい、という条件）になっています。

```js
function renderSubjectFilterChips() {
  const wrap = document.getElementById('filter-subject-chips');
  if (!wrap) return;
  const allBtn = `<button class="chip chip-active" data-subj="all" onclick="toggleSubjFilter(this)">すべて</button>`;
  const chs = channels.map(c =>
    `<button class="chip" data-subj="${esc(c.name)}" onclick="toggleSubjFilter(this)">${esc(c.name)}</button>`
  ).join('');
  wrap.innerHTML = allBtn + chs;
}
```
- 「すべて」ボタン＋科目一覧（`channels`）からボタン（チップ）を1つずつ文字列として組み立てて表示します。ここは`innerHTML`（HTMLの文字列をそのまま流し込む方法）を使っていますが、科目名（`c.name`）は必ず`esc()`（2.6節）を通しているので、危険な文字列が混ざっていても安全に表示されます。

```js
function toggleSubjFilter(btn) {
  filterSubject = btn.dataset.subj;
  btn.closest('.filter-chips').querySelectorAll('.chip').forEach(c => c.classList.remove('chip-active'));
  btn.classList.add('chip-active');
  renderPlans();
}
function toggleCatFilter(btn) { /* 同じ考え方で filterCat を更新する */ }
```
- チップがクリックされたら、そのチップが持っている`data-subj`（HTML部品に埋め込んでおいた属性値）を新しい絞り込み条件として保存し、同じグループの他のチップから「選択中」の見た目を外して、自分だけに付け直します（＝一度に1つしか選べないようにする）。最後に一覧を再描画します。

---

## 12. 時間割の順番で並べ替える（470〜522行）

```js
const TIMETABLE = { mon: [...], tue: [...], wed: [...], thu: [...], fri: [...] };
const WDAY_KEYS = ['sun','mon','tue','wed','thu','fri','sat'];
```
- 曜日ごとに「時間割上、何番目にどの科目があるか」をあらかじめプログラムの中に書き込んである一覧です（Timetableページのデータとは別に、このページ専用にここへ直接書かれています＝1年間このままという前提のため）。

```js
function timetableOrderIndex(dateStr, subject) {
  const d = new Date(dateStr + 'T00:00:00');
  const key = WDAY_KEYS[d.getDay()];
  const list = TIMETABLE[key];
  if (!list) return Infinity;
  const idx = list.findIndex(item => item.subject === subject);
  return idx === -1 ? Infinity : idx;
}
```
- 日付の文字列から曜日を計算し、その曜日の時間割の中で指定した科目が何番目にあるかを返します。
- 土日など時間割が無い曜日や、時間割に載っていない科目は、`Infinity`（無限大＝「一番最後」として扱われる特別な値）を返します。

```js
function sortByTimetable(dateStr, dayPlans) {
  return dayPlans
    .map((p, i) => ({ p, i, order: timetableOrderIndex(dateStr, p.subject) }))
    .sort((a, b) => (a.order - b.order) || (a.i - b.i))
    .map(x => x.p);
}
```
- その日の予定それぞれに「元の並び順（`i`）」と「時間割上の順番（`order`）」という情報をいったん付け足してから、時間割の順番で並べ替え（同じ順番のものは元の並び順を保つ）、最後に付け足した情報を外して予定そのものだけを返す、という3段階の処理です。

---

## 13. 予定一覧を画面に描画する（524〜588行）

```js
function parsePlanContent(raw) {
  const cat  = raw.match(/^【(.+?)】/)?.[1] || '';
  const rest = raw.replace(/^【.+?】/, '');
  const [textPart, notePart] = rest.split(NOTE_SEP);
  return { cat, text: (textPart || '').trim(), note: (notePart || '').trim() };
}
```
- 予定の中身（例：`【宿題】p.10 問1〜5\n📝備考：来週まで延長`）を、「カテゴリ」「本文」「備考」の3つに分解する関数です。
- `raw.match(/^【(.+?)】/)`は「文字列の先頭にある『【〜】』の部分を探す」という正規表現の命令で、見つかった中身（カテゴリ名）を取り出します。`?.[1] || ''`は「見つからなければ空文字にしておく」という安全策です。
- `rest.split(NOTE_SEP)`で、区切り文字（`NOTE_SEP`）の前後を「本文」と「備考」に分けます。

`renderPlans()`（536〜588行）は、予定一覧の描画の中心となる関数です：
- 542〜544行：まだ読み込んでいない過去分があれば「さらに過去の予定を読み込む」ボタンを用意（無ければ何も用意しない）。
- 546〜551行：絞り込んだ結果が0件なら、「そもそも予定が無い」のか「絞り込みの結果0件になった」のかでメッセージを出し分けて処理を終える。
- 553〜555行：今日の日付を取得し、予定を日付ごとのグループにまとめる。
- 557行以降：日付を古い順に並べながら、日付ごとに1つのまとまり（`<div class="date-group">`）を作る。
  - 「2026年8月20日（木）」のような日本語の日付表示を作成。
  - 今日かどうか・過去かどうかを判定（`YYYY-MM-DD`という形式の文字列は、そのまま文字として比較しても日付の前後関係と一致するので、特別な計算をせず単純な比較で済んでいます）。
  - その日の予定を12節の関数で時間割順に並べ替え。
  - 予定1件ごとの行を組み立てる。カテゴリ・本文・備考に分解し、ポイントがあればバッジ、備考があればアイコンを付け足します。**表示する値はすべて`esc()`（2.6節）を通してから埋め込みます。** 行全体には後で見分けるための目印（`data-label`）を付け、クリックすると詳細表示（`showPlanDetail`）が開くようにしています。
  - 日付のまとまりには`data-date`という目印を付けておき（`scrollToToday`が「この日付のまとまりはどこ？」と探すときに使う）、今日なら「今日」というタグを追加。
- 最後に、組み立てたHTML文字列をまとめて画面に流し込みます。

このページの一覧表示は、基本的に文字列を組み立てて`innerHTML`に流し込む方式ですが、ユーザーが入力した値は必ず`esc()`を通しているため、他のページ（お知らせページなど）で過去に見つかった「表示前のエスケープ漏れ」と同じ問題は起きない作りになっています。

---

## 14. 予定の詳細表示ポップアップ（590〜652行）

```js
function showPlanDetail(el) {
  const label = el.dataset.label;
  const plan = plans.find(p => `${p.date}/${p.subject}${p.content}` === label);
  if (!plan) return;
  detailTarget = label;
  const { cat, text, note } = parsePlanContent(plan.content);
  // ...詳細欄のHTMLを組み立てる...
  document.getElementById('detail-content').innerHTML = rowsHtml.join('');
  document.getElementById('modal-detail').classList.add('open');
}
```
- 一覧の行がクリックされると、その行に付けておいた目印（`data-label`）を使って、`plans`（読み込んだ予定の一覧）の中から該当する予定を探します。
- 見つかった予定を`detailTarget`という変数に覚えておきます（あとで「編集する」「削除する」ボタンがこれを参照します）。
- 日付・科目・カテゴリ・内容・（あれば）ポイント・（あれば）備考を、それぞれエスケープ済みの表示にして詳細欄に流し込み、ポップアップを開きます。

```js
function selectPlanByLabel(label, mode) { /* 一覧の中から同じ目印を持つ項目を探して選択状態にする */ }
function editFromDetail() {
  if (!detailTarget) return;
  closeModal('detail'); openModal('edit'); selectPlanByLabel(detailTarget, 'edit');
}
function deleteFromDetail() {
  if (!detailTarget) return;
  closeModal('detail'); openModal('delete'); selectPlanByLabel(detailTarget, 'delete');
}
```
- 詳細ポップアップの「編集する」「削除する」ボタンは、対象のポップアップ（編集用/削除用）を開いたうえで、選択リストの中から同じ予定を自動的に選んであげる、という「詳細画面から編集・削除画面へのつなぎ役」になっています。

---

## 15. 変更履歴（ログ）の描画（654〜674行）

```js
const TYPE_LABEL = { add:'追加', edit:'編集', delete:'削除', cleanup:'自動削除' };
function renderLogs() {
  const el = document.getElementById('log-content');
  if (!logsData.length) { el.innerHTML = '<div class="empty-msg">ログはありません</div>'; return; }
  const loadMoreHtml = logsHasMore
    ? `<button type="button" id="log-load-more-btn" class="load-more-btn" onclick="loadMoreLogs()">もっと読み込む</button>`
    : '';
  el.innerHTML = logsData.map(l => `
    <div class="tl-item">
      <div class="tl-dot dot-${esc(l.type)}"></div>
      <div class="tl-time">${esc(l.time)}</div>
      <div class="tl-card">
        <div class="tl-type type-${esc(l.type)}">${esc(TYPE_LABEL[l.type] || l.type)}</div>
        <div class="tl-detail">${esc(l.detail)}</div>
      </div>
    </div>`).join('') + loadMoreHtml;
}
```
- `logsData`（読み込んだログ）が0件なら「ログはありません」で終了。
- それぞれのログを、種類ごとに色分けした丸印・時刻・種類（追加/編集/削除/自動削除という日本語に変換したラベル）・詳細のテキストを並べたタイムライン形式で表示します。「もっと読み込む」ボタンは一番下（ログは新しい順に並んでいるので、続きは末尾に来る）に付けます。

---

## 16. 科目選択欄と、対象を選ぶ一覧（676〜738行）

```js
function renderChannelOptions() {
  const opts = channels.map(c => `<option value="${esc(c.name)}">${esc(c.name)}</option>`).join('');
  document.getElementById('add-subject').innerHTML  = opts || '<option value="">（なし）</option>';
  document.getElementById('edit-subject').innerHTML = '<option value="">— 変更しない —</option>' + opts;
}
```
- 科目一覧（`channels`）から、`<select>`の選択肢（`<option>`）を作って、追加用（科目が1つも無ければ「（なし）」）と編集用（先頭に「変更しない」という空の選択肢）にそれぞれ入れます。

```js
function renderSelectList(containerId, mode) {
  // plans全件から、編集・削除で使う「対象を選ぶ一覧」を組み立てる
}
```
- 編集・削除ポップアップに表示する「どの予定を対象にするか選ぶ一覧」を作る共通の関数です。`mode`によって、クリックしたときの動きが編集用・削除用で変わります。

```js
function selectPlan(el, mode) {
  el.closest('.sel-list').querySelectorAll('.sel-item').forEach(i => i.classList.remove('selected'));
  el.classList.add('selected');
  if (mode === 'edit') {
    editTarget = el.dataset.label;
    const plan = plans.find(p => `${p.date}/${p.subject}${p.content}` === el.dataset.label);
    // 見つかった予定の今の内容を、編集フォームの入力欄にあらかじめ入れておく
    // （何も変更せずそのまま保存すれば、内容が変わらないようにするため）
  } else {
    delTarget = el.dataset.label;
    document.getElementById('del-label').textContent = el.dataset.label;
    document.getElementById('del-confirm').style.display = 'block';
  }
}
```
- 一覧の中で今選んでいる項目に「選択中」の見た目を付け直します。
- 編集モードのときは、選んだ予定の今の内容（本文・備考・ポイント）をあらかじめ入力欄に入れておきます。これにより、変えたい項目だけを書き換えて保存すれば、それ以外は元のまま維持されます。
- 削除モードのときは、選んだ予定の内容を確認文言として表示するだけです。

---

## 17. 右下の＋ボタンと、ポップアップの開閉（740〜780行）

```js
function toggleFab() {
  const open = !document.getElementById('fab-actions').classList.contains('open');
  document.getElementById('fab-actions').classList.toggle('open', open);
  document.getElementById('fab-main').classList.toggle('open', open);
  document.getElementById('fab-overlay').classList.toggle('open', open);
}
function closeFab() { /* 上と同じ3つの部品から「開いている」印を外すだけ */ }
```
- 右下の＋ボタン（FAB）を押すたびに、今の開閉状態を反転させて、関係する3つの部品（小さいボタン群・ボタン自身のアニメーション・背景の暗いオーバーレイ）に同じ状態を反映します。

```js
function openModal(name) {
  closeFab();
  document.getElementById('modal-' + name).classList.add('open');
  if (name === 'add')  { /* 追加用の初期化 */ }
  if (name === 'edit') { /* 編集用の初期化（過去の日付も選べるようにする等） */ }
  if (name === 'delete') { /* 削除用の初期化 */ }
}
function closeModal(name) {
  document.getElementById('modal-' + name).classList.remove('open');
  document.querySelectorAll('.cal-pop').forEach(p => p.classList.remove('open'));
}
function onBgClick(e, name) {
  if (e.target === document.getElementById('modal-' + name)) closeModal(name);
}
```
- `openModal`：まず＋ボタンのメニューを閉じ、指定されたポップアップを開きます。ポップアップの種類ごとに、カレンダーの初期化や選択状態のリセットといった準備を行います。編集ポップアップだけ「過去の日付にも変更できる」設定になっています（新規追加では過去の日付を選べません）。
- `closeModal`：ポップアップを閉じ、開いていたカレンダーも一緒に閉じます。
- `onBgClick`：ポップアップの外側の暗い背景がクリックされたときだけ閉じます。ポップアップの中身をクリックしたときはこの条件に一致しないので、誤って閉じることはありません。

---

## 18. ポイント選択チップの表示（782〜828行）

```js
function updatePointsVisibility(prefix) {
  const cat = getCatValue(prefix);
  const wrap = document.getElementById(prefix + '-points-wrap');
  if (!wrap) return;
  wrap.style.display = cat ? 'block' : 'none';
  if (cat) renderPointsChips(prefix);
}
```
- カテゴリが（自由入力も含めて）何か決まっていればポイント選択欄を表示し、決まっていなければ欄ごと隠します。

```js
function renderPointsChips(prefix) {
  // 3・5・10・15ptのボタンを、現在選ばれている値だけ「選択中」の見た目にして作る
  // 提出・宿題カテゴリで、追加時に未選択なら自動的に5ptを選んでおく
}
function pickPoints(prefix, val) {
  // 任意カテゴリのときだけ、同じボタンをもう一度押すと選択解除できる
}
```
- `prefix`は「追加画面（`add`）か、編集画面（`edit`）か」を表す文字列です。同じ関数を両方の画面で使い回せるようにするための工夫です。
- 必須カテゴリ（提出・宿題）では常にどれかを選ぶ必要がありますが、それ以外の任意カテゴリでは「選ばない」という状態も選べるように、同じボタンをもう一度押すと選択を取り消せるようになっています。

---

## 19. 追加・編集・削除の送信処理（830〜946行）

### 19.1 `submitAdd()`（833〜879行）— 予定を追加する
1. まずログインしているか確認し、していなければ処理を中断します。
2. カレンダーで選ばれた日付・選ばれた科目・カテゴリを取り出します。カテゴリが空、または日付/科目/内容のどれかが空なら、エラーメッセージを表示して処理を中断します。
3. 備考が入力されていれば、区切り文字でつなげて1本の文字列にします。
4. サーバーに送るデータのかたまり（日付・科目・カテゴリ・内容・ニックネームなど）を組み立てます。
5. カテゴリが「提出」「宿題」ならポイントの選択が必須（未選択ならエラー）、それ以外のカテゴリでも選ばれていればポイントを一緒に送ります。
6. 送信ボタンを「送信中…」の見た目に変え、サーバーへ追加のお願いを送ります。
7. 成功したら：成功メッセージを表示、入力欄を空にする、ポイント欄を隠す、カレンダーをリセットし、一覧を読み込み直します。
8. 失敗したら（サーバーがエラーを返した場合、または通信自体が失敗した場合）：ボタンを元に戻し、エラーメッセージを表示します。

### 19.2 `submitEdit()`（881〜920行）— 予定を編集する
- 基本的な流れは追加と同じですが、大きな違いは**入力・選択されている項目だけをサーバーに送る**点です。日付欄が空欄のままなら日付は送らない、科目欄が空欄のままなら科目は送らない、というように、「変更しなかった項目には触れない」形でデータを組み立てます。これは、サーバー側が「送られてきた項目だけを書き換える」という仕組みになっているためです。
- 成功したら、選択状態や入力欄をすべてリセットし、一覧と選択リストの両方を最新の内容に更新します。

### 19.3 `submitDelete()`（922〜946行）— 予定を削除する
- 削除対象が選ばれていなければ何もしません。選ばれていれば、確認済みという前提でそのままサーバーに削除をお願いします。
- 成功したら、確認欄を隠し、選択状態をクリアし、一覧と選択リストを更新します。

---

## 20. 画面の見た目を助ける小さな関数たち（948〜983行）

```js
function setLoading(btn, label, dark = false) {
  btn.disabled = true;
  btn.innerHTML = `<span class="spinner${dark ? ' spinner-dark' : ''}"></span>${label}`;
}
function resetLoading(btn, label) { btn.disabled = false; btn.textContent = label; }
```
- `setLoading`：ボタンを「押せない状態」にして、くるくる回るスピナー（読み込み中を示すアイコン）と文言を表示します。
- `resetLoading`：ボタンを元の押せる状態・元の文言に戻します。

```js
function showOk(id) { /* 成功メッセージを表示して、3秒後に自動で消す */ }
function showErr(id, msg) { /* エラーメッセージを表示して、4秒後に自動で消す（メッセージはエスケープ済み） */ }
```
- `showErr`のメッセージも`esc()`を通しています。サーバーから返ってきたエラーメッセージであっても、念のためそのまま信用せずにエスケープしてから表示する、という慎重な作りです。

```js
function getCatValue(prefix) {
  const sel = document.getElementById(prefix + '-category-sel');
  if (sel.value === '__custom__') return document.getElementById(prefix + '-category-inp').value.trim();
  return sel.value;
}
function onCatSel(prefix) {
  // 「自由入力…」が選ばれたらテキスト入力欄を表示、それ以外なら隠す
  // カテゴリが変わるたびにポイント欄の表示も更新する
}
```
- `getCatValue`：カテゴリの選択欄で「自由入力…」が選ばれている場合はテキスト入力欄の値を、そうでなければ選択欄の値そのものを返す、共通の取り出し関数です。

---

## 21. 自分で作ったカレンダー（985〜1064行）

ブラウザにもとから用意されている日付入力欄（`<input type="date">`）は使わず、見た目をサイトのデザインに合わせるため、カレンダーを完全に自分たちで作っています。

```js
function initCal(id, allowPast) {
  const now = new Date();
  calState[id] = { year: now.getFullYear(), month: now.getMonth(), selected: null, allowPast };
  renderCal(id);
}
```
- `id`は「追加用」か「編集用」かを表す文字列。今日の年月を初期表示にして状態を作り、カレンダーを描画します。`allowPast`が`false`（追加のとき）だと過去の日付を選べないようにします。

```js
function renderCal(id) {
  // 今月の1日が何曜日か、今月が何日まであるかを計算し、
  // 曜日の見出し・空白セル・日付セルを順番に組み立てて表示する
  // 過去の日付・今日・選択中の日付には、それぞれ違う見た目のクラスを付ける
}
```
- 月の1日が何曜日かを調べることで、カレンダーの先頭にいくつ空白を入れればよいかが分かります。月の最終日は、「翌月の0日目」という少し変わった計算方法（JSではよく使われる書き方）で求めています。
- 過去の日付でクリックできないようにしたいときは、その日のセルにはクリック用の命令自体を付けないことで実現しています。

```js
function moveCal(e, id, dir) {
  // 前月/翌月ボタン。月が0未満・12以上になったら年をまたいで繰り上げ・繰り下げる
}
function pickDate(e, id, ds) {
  // 日付をタップしたときの確定処理。選んだ日付を状態に保存し、表示を更新する
}
function toggleCal(e, id) {
  // 日付欄をタップしてカレンダーを開閉する
  // 開いたときにカレンダーが画面の下からはみ出していたら、少しスクロールして見えるようにする
}
```
- 外側（`.date-wrap`の外）がクリックされたときは、開いているカレンダーをすべて閉じる処理も別途用意されています（2.5節と同じ「外側クリックで閉じる」パターン）。

---

## 22. ドロワーメニューの開閉と、他ページの先読み（1066〜1122行）

```js
function openDrawer() {
  document.getElementById('drawer').classList.add('open');
  document.getElementById('drawer-overlay').classList.add('open');
  prefetchOtherPages();
}
```
- ハンバーガーボタンでメニューを開くとき、**開いた瞬間に**他ページのファイルの先読みも一緒に始めます（実際にどのページへ行くか決まる前から準備しておく、という考え方）。

```js
let _didPrefetchOtherPages = false;
function prefetchOtherPages() {
  if (_didPrefetchOtherPages) return;
  _didPrefetchOtherPages = true;
  ['/Timetable.js', '/Cardmaker.js', '/Cardmaker.css', '/StudyLog.js', '/StudyLog.css', '/Notice.js', '/ServiceInfo.js']
    .forEach(href => {
      const link = document.createElement('link');
      link.rel = 'prefetch';
      link.href = href;
      document.head.appendChild(link);
    });
}
```
- 一度実行したら、二度と実行しないように印（`_didPrefetchOtherPages`）を立てます（メニューを何度開いても、無駄に同じ先読みを繰り返さないため）。
- `<link rel="prefetch">`という特別なHTML部品をJSで動的に作って画面の裏側に追加すると、ブラウザがそのファイルをバックグラウンドでこっそりダウンロードしておいてくれます。実際にリンクをクリックする頃には、もうブラウザの中に用意ができている、という仕組みです。

```js
document.querySelectorAll('.drawer-item[href]').forEach(a => {
  a.addEventListener('click', (e) => {
    if (a.classList.contains('active')) { e.preventDefault(); closeDrawer(); return; }
    const overlay = document.getElementById('page-nav-loading');
    if (overlay) overlay.classList.add('show');
  });
});
window.addEventListener('pageshow', () => {
  const overlay = document.getElementById('page-nav-loading');
  if (overlay) overlay.classList.remove('show');
});
```
- メニューの各リンクにクリック時の処理を付けます。**今まさに開いているページ自身へのリンク**（＝ここでは「予定管理」自身）をタップした場合は、ページ遷移そのものを取り消して、メニューを閉じるだけにします（同じページへの無駄な再読み込みを防ぐため）。
- それ以外のページへのリンクなら、実際のページ移動はそのまま行いつつ、移動中であることが分かるローディング表示だけすぐに出します。
- `pageshow`というイベントは、ブラウザの「戻る」ボタンでページが（サーバーに聞き直さず、ブラウザに保存されていた状態のまま）復元されたときにも発生します。そのタイミングでもこのローディング表示は必ず消しておくようにしています。これをやっておかないと、「戻る」で前のページに戻ったときにローディング表示が画面に張り付いたまま消えなくなる、という不具合が起きるためです。

---

## 23. 絞り込みバーの開閉（1124〜1133行）

```js
function toggleFilterBar() {
  const bar = document.getElementById('filter-bar');
  const btn = document.getElementById('filter-toggle-btn');
  const isOpen = bar.classList.contains('open');
  bar.classList.toggle('open', !isOpen);
  btn.classList.toggle('filter-toggle-active', !isOpen);
}
```
- ヘッダーの虫眼鏡アイコンを押すと、絞り込みバーの開閉と、ボタン自体の見た目（押されている感じ）を同時に切り替えます。

---

## 24. 他の人の変更をリアルタイムに反映する仕組み（1135〜1211行）

```js
let lastScheduleHash = null;
async function digestMessage(message) {
  const msgUint8 = new TextEncoder().encode(message);
  const hashBuffer = await crypto.subtle.digest("SHA-256", msgUint8);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, "0")).join("");
}
```
- 文字列から「ハッシュ値」（ミニ用語辞典参照）を計算する関数です。ブラウザに標準で備わっている暗号関連の機能（Web Crypto API）を使っています。

```js
async function checkScheduleUpdate() {
  try {
    const session = getLoginSession();
    const res = await fetch(`${API_BASE}list_schedule?guild_id=${GUILD_ID}&scope=future`, { headers: /* ログイン情報 */ });
    const txt = await res.text();
    const hash = await digestMessage(txt);

    if (lastScheduleHash === null) { lastScheduleHash = hash; return; }
    if (hash === lastScheduleHash) return;
    lastScheduleHash = hash;

    // ここまで来た＝内容が本当に変わっていた、ということなので、
    // 未来分だけ新しいデータに差し替え、過去分（読み込み済み）はそのまま維持して合体させる
    const scrollY = window.scrollY;
    renderPlans();
    window.scrollTo(0, scrollY);

    // 編集・削除ポップアップが開いていれば、そちらの一覧も最新化する
  } catch(e) {}
}
```
- 「これからの予定」を毎回サーバーに問い合わせるのですが、その**返ってきた内容全体のハッシュ値**を前回のものと比較します。ハッシュ値が同じなら「中身は何も変わっていない」ということなので、そのあとの重い処理（データの読み取り・画面の再描画）を一切せずに終わります。これにより、変化が無いときの無駄な処理を減らしています。
- 内容が変わっていたと分かったときだけ、実際にデータを読み取り直して画面を更新します。このとき、画面のスクロール位置を一度覚えておいて、更新が終わったら同じ位置に戻すことで、**見ていた場所がジャンプしてしまわないように**しています。
- もし編集・削除のポップアップが開いている最中にこの更新が来た場合は、その中の選択リストも一緒に最新化します（古いデータのまま操作してしまう事故を防ぐため）。
- 全体を「エラーが起きても止まらない」処理（`try/catch`）で囲んでいるので、通信エラーが1回起きても、この定期チェック自体は次回もちゃんと動き続けます。

```js
function startRealtimeUpdates() {
  try {
    const es = new EventSource(`${API_BASE}events?guild_id=${GUILD_ID}`);
    es.onmessage = () => { checkScheduleUpdate(); };
  } catch (e) {}
}
startRealtimeUpdates();
setInterval(checkScheduleUpdate, 10000);
```
- `EventSource`という仕組みで、サーバーからの「何か変わったよ」という通知を待ち受けます。通知が来るたびに（内容の中身は見ずに）先ほどの`checkScheduleUpdate()`を呼び、実際に変わっていたかどうかを確認しに行きます。
- もしこの通知の仕組み自体が使えない環境だった場合に備えて、`setInterval(処理, 10000)`＝「10秒ごとに繰り返しこの処理を実行する」という命令で、定期的にチェックする仕組みも並行して動かしています（通知が正常に機能していても、こちらは保険として動き続けます）。

---

## 25. 締めくくり（1213〜1216行）

```js
hideLoadingFallback();
```
- ここまでのプログラムがエラーなく最後まで実行できた、という合図として、1.1節で説明した「読み込み中…」の保険表示を消します。もしこの行にたどり着けないままだった場合（途中でエラーが起きて止まっていた場合）は、代わりに1.1節のインライン`<script>`が数秒後にボタンを出す、という2段構えの安全策になっています。

---


