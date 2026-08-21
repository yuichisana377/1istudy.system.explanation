# StudyLog.js その1：ログイン・アカウント設定・データ読み込み（1〜635行）

[00_HTML構造と全体像.md](00_HTML構造と全体像.md)の続きです。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

---

## 1. 連打対策のローディング表示（14〜46行）

```js
(function injectSpinnerStyle() {
  if (document.getElementById("sl-spinner-style")) return;
  var style = document.createElement("style");
  style.id = "sl-spinner-style";
  style.textContent = "@keyframes sl-spin{...}" + ".sl-spinner{...}" + "button.sl-btn-loading{...}";
  document.head.appendChild(style);
})();
```
- 珍しい書き方として、CSSのアニメーション定義を`<style>`タグごとJSで作って`<head>`に追加しています。コメントに「HTML/CSS側の変更なしで動くよう、スピナー用CSSはJSから自前で挿入する」とあり、`StudyLog.css`ファイル自体を編集せずに、この機能をJS側だけで完結させる工夫です。`if (document.getElementById(...)) return;`で、二重に追加されないようガードしています。

```js
function setButtonLoading(btn, loading, label) {
  if (loading) {
    if (btn.dataset.origLabel === undefined) btn.dataset.origLabel = btn.textContent;
    btn.disabled = true;
    btn.classList.add("sl-btn-loading");
    btn.innerHTML = '<span class="sl-spinner"></span>' + (label || "保存中…");
  } else {
    btn.disabled = false;
    btn.classList.remove("sl-btn-loading");
    btn.textContent = (btn.dataset.origLabel !== undefined) ? btn.dataset.origLabel : btn.textContent;
    delete btn.dataset.origLabel;
  }
}
```
- [../02_Cardmaker/08_Cardmaker.js_その8_画像処理と基盤機能.md](../02_Cardmaker/08_Cardmaker.js_その8_画像処理と基盤機能.md)の`setBtnLoading`と同じ考え方の関数です。このページのボタンはこの関数で「保存中…」の見た目に切り替えられます。

---

## 2. 基本設定とログインチェック（48〜85行）

```js
const _s = getSession() || {};
const STUDENT = { id: _s.student_id, nickname: _s.nickname, color: _s.color, textColor: _s.text_color };
const SESSION_TOKEN = _s.session_token;
```
- [../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md](../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md)の`STUDENT`と同じパターンです。`SESSION_TOKEN`はログイン証明のトークンを、変数名を付けて何度も使えるようにしたものです。コメントに「サーバーはこれを使って`student_id`を特定するので、クライアントが送る`student_id`自体は（表示用途を除き）信用されない」とあります。つまり、通信のたびに「自分は誰です」と学籍番号を自己申告するのではなく、サーバー側がこのトークンから本人を特定する仕組みで、なりすましを防いでいます。

```js
(function() {
  var s = getSession();
  if (!s || !s.session_token) { location.replace("/Login.html"); }
})();
```
- このページは、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`と同じように、開いた瞬間に未ログインなら強制的にログイン画面へ飛ばします。ただし`Plan.js`と違い、戻ってくるためのURL記憶（`sessionStorage.setItem('post_login_redirect', ...)`）は行っていません。

```js
function splitContentNote(raw) {
  const stripped = String(raw).replace(/【.*?】/, "").trim();
  const [textPart, notePart] = stripped.split(NOTE_SEP);
  return { text: (textPart || "").trim(), note: (notePart || "").trim() };
}
```
- [../01_index_予定管理.md](../01_index_予定管理.md)の`parsePlanContent`と似ていますが、カテゴリ部分（`【】`の中身）自体は返さず、取り除いた残りだけを本文・備考に分ける、少し簡略化された版です（課題一覧ではカテゴリの区別が使われないため）。

---

## 3. ドロワーのアカウント表示（87〜172行）

`renderDrawerAccount()`の実装自体は[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`版とほぼ同じですが、1点大きな違いがあります：

```js
var settingsBtn = document.createElement('button');
...
settingsBtn.addEventListener('click', function(e) {
  e.stopPropagation();
  el.classList.remove('is-open');
  openAccountModal();
});
```
- 他ページでは「アカウント設定」の項目が`<a href="/StudyLog.html?openAccount=1">`という**リンク**でしたが、このページ自身にはアカウント設定モーダルの実装があるため、リンクで別ページに飛ばす必要が無く、`openAccountModal()`（4節）を**直接呼び出す**ボタンになっています。コメントにも「このページ自体にアカウント設定モーダル（`openAccountModal`）・ログアウト（`doLogout`）が既にあるので、他ページのように別ページへ飛ばさずそれらを直接呼ぶ」とあります。

---

## 4. アカウント設定モーダル（342〜512行）

このページにしか無い、Discord連携・ニックネーム変更・パスワード変更をまとめた設定画面です。他ページはここへのリンクを持つだけで、実装の本体はここにあります。

### 4.1 モーダルの組み立て：`openAccountModal()`（360〜409行）
```js
function openAccountModal() {
  closeAccountModal(); // 二重生成防止
  var overlay = document.createElement("div");
  overlay.id = "sl-acct-overlay";
  overlay.style.cssText = "position:fixed;inset:0;z-index:9999;...";
  ...
  box.innerHTML =
    '<div>...閉じるボタン...</div>' +
    '<div>学籍番号: ' + escapeHtmlSl(STUDENT.id) + '</div>' +
    '<div>ニックネーム入力欄...</div>' +
    '<div>Discord連携ボタン...</div>';
  overlay.appendChild(box);
  document.body.appendChild(overlay);
  ...
}
```
- 他ページのモーダルのように、あらかじめHTMLに`<div class="modal-bg">`を用意しておくのではなく、**このモーダル全体をJSの実行時に丸ごと`document.createElement`で作って`document.body`に追加**する、という独特な作り方をしています。コメントによれば「以前は専用の『⚙ アカウント』ボタンから開いていたが、現在はヘッダーのニックネーム／学籍番号をタップすることで開く方式に変更した」という経緯があり、その名残でHTML側に定型のモーダル要素が用意されていない、という事情がありそうです。
- `innerHTML`に直接埋め込んでいる部分（`box.innerHTML = '...'`）でも、`STUDENT.id`のような値は必ず`escapeHtmlSl()`（4.3節、他の`esc()`関数と同じ役割）を通してからテンプレート文字列に埋め込んでいます。

### 4.2 モーダルを閉じる（411〜414行）
```js
function closeAccountModal() {
  var el = document.getElementById("sl-acct-overlay");
  if (el) el.remove();
}
```
- 他ページの`open`/`close`クラスの付け外しとは違い、**要素そのものをDOMから削除**する形で閉じます。`openAccountModal()`の先頭で毎回`closeAccountModal()`を呼んでいるのは、間違って複数回開かれてモーダルが重複生成されるのを防ぐためです。

### 4.3 メッセージ表示（416〜428行）
```js
function escapeHtmlSl(s) {
  return String(s == null ? "" : s).replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
}
function setAcctMsg(id, msg, isError) {
  var el = document.getElementById(id);
  if (!el) return;
  el.innerHTML = msg;
  el.style.color = isError ? "#dc2626" : "#16a34a";
}
```
- `escapeHtmlSl`はこのファイル独自の`esc()`相当の関数（[../01_index_予定管理.md](../01_index_予定管理.md)参照）。`setAcctMsg`はモーダル内のメッセージ欄（保存成功・エラーなど）を表示する共通関数で、コメントに「`msg`は常にこのファイル内の固定文字列なので、`innerHTML`で組み立てても安全」とあります（サーバーから返ってくるエラーメッセージをそのまま使っている箇所もあるため、ここは他の慎重な箇所と比べるとやや踏み込んだ扱いです）。

### 4.4 ニックネーム変更：`submitNicknameChange()`（431〜465行）
```js
if (!nickname)            { setAcctMsg("sl-acct-nickname-msg", "ニックネームを入力してください", true); return; }
if (nickname.length > 16) { setAcctMsg("sl-acct-nickname-msg", "16文字以内で入力してください", true); return; }
...
var data = await api("/change_nickname", { method: "POST", body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN, nickname: nickname }) });
if (data && data.ok) {
  STUDENT.nickname = nickname;
  var s = getSession() || {};
  s.nickname = nickname;
  try { localStorage.setItem(SESSION_KEY, JSON.stringify(s)); } catch(e) {}
  nicknameMap[STUDENT.id] = nickname;
  applySession();
  renderAll();
  setAcctMsg(...);
} else if (data && data.error === "not_logged_in") {
  forceReLogin();
} else { ... }
```
- 成功したら、**この端末が持つあらゆる箇所のニックネーム情報**を一度に更新しています：メモリ上の`STUDENT`オブジェクト、`localStorage`に保存されているセッション情報、全ユーザーの名前対応表（`nicknameMap`）、そして画面表示（`applySession()`）とランキング等の再描画（`renderAll()`）。1箇所でも更新し忘れると「新しい名前がヘッダーには反映されたのに、ランキングでは古い名前のまま」というような不整合が起きるため、関連する場所を漏れなく更新する必要があります。
- `data.error === "not_logged_in"`のときは`forceReLogin()`（4.6節）でセッション切れとして扱われます。

### 4.5 ログアウト（468〜473行）
```js
async function doLogout() {
  const ok = await showAppConfirm({ title: "ログアウトしますか？", okLabel: "ログアウト", danger: true });
  if (!ok) return;
  localStorage.removeItem(SESSION_KEY);
  location.replace("/Login.html");
}
```
- [../01_index_予定管理.md](../01_index_予定管理.md)で紹介した`Dialog.js`共通の確認ダイアログ（`showAppConfirm`）を使っています。

### 4.6 セッション切れの強制ログイン画面誘導（478〜482行）
```js
async function forceReLogin() {
  localStorage.removeItem(SESSION_KEY);
  await showAppAlert({ title: "ログインが切れました", desc: "もう一度ログインしてください。" });
  location.replace("/Login.html");
}
```
- サーバー側が「このセッションはもう無効です」（`not_logged_in`エラー）と答えたときに、あちこちの通信処理から呼ばれる共通の後処理です。ローカルの古いセッション情報を消してから、一言案内してログイン画面に戻します。

### 4.7 Discord連携の開始：`startDiscordOAuth()`（490〜512行）
```js
var data = await api("/discord_oauth_start", { method: "POST", body: JSON.stringify({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }) });
if (data && data.ok && data.authorize_url) {
  location.href = data.authorize_url; // Discordの認可画面へ移動
} else if (data && data.error === "not_logged_in") { forceReLogin(); }
else if (data && data.error === "oauth_not_configured") { setAcctMsg(..., "現在Discord連携（OAuth）は準備中です。...", true); }
else { setAcctMsg(..., "連携の開始に失敗しました。...", true); }
```
- コメントに認証の仕組みが説明されています：「ログイン済み（`session_token`検証済み）の本人だけが呼べるAPIで一時state（合言葉のようなもの）を発行してもらい、そのstate付きでDiscordの認可画面にブラウザごと移動する。認可後はサーバー側が`state`を検証してから連携するので、他人になりすまして連携される心配はない」。この`state`は、`session_token`の持ち主（＝ログイン中の本人）に対してのみサーバーが発行するため、URLを他人に渡されても悪用できない仕組みになっています。

---

## 5. その他のデータ読み込み関数（514〜634行）

```js
async function loadUsers() {
  try {
    var data = await api("/get_users?guild_id=" + GUILD_ID);
    if (data.ok) { (data.users || []).forEach(function(u) { if (u.id && u.nickname) nicknameMap[u.id] = u.nickname; }); }
  } catch(e) {}
  nicknameMap[STUDENT.id] = STUDENT.nickname;
}
```
- 全ユーザーの「学籍番号→ニックネーム」対応表（`nicknameMap`）を組み立てます。最後に必ず自分自身の分を上書きしているのは、サーバー側のデータが多少古くても、少なくとも自分の表示名だけは今の正しい値になるようにするための保険です。

`loadCompletedTasks()`（自分の達成済み課題）／`loadAllCompletedTasks()`（全員分の達成済み課題、週間ランキング集計用）は、それぞれサーバーから取得したデータを整形して変数に保存するだけの関数です。`loadCompletedTasks`では、古いデータ形式（文字列だけのID配列）と新しいデータ形式（日付・ポイント付きのオブジェクト配列）の両方に対応する変換処理（`typeof e === "string" ? {...} : e`）が入っています。

```js
async function api(path, opts) {
  opts = opts || {};
  var headers = Object.assign({ "Content-Type": "application/json" }, SESSION_TOKEN ? { "Authorization": "Bearer " + SESSION_TOKEN } : {}, opts.headers || {});
  var res = await fetch(API_BASE + path.replace(/^\/+/, ''), Object.assign({}, opts, { headers: headers }));
  return res.json();
}
```
- このページ独自の`api()`ヘルパーです。他ページの`api()`と違い、ログインセッションを毎回`getLoginSession()`で調べ直すのではなく、**ページ読み込み時に確定した`SESSION_TOKEN`をそのまま使い回します**。コメントにある通り、このページ自体が最初からログイン必須（未ログインなら開いた瞬間にログイン画面へ飛ばされる）なので、`SESSION_TOKEN`は必ず値を持っている前提で単純化されています。

```js
async function loadLogs() {
  try {
    var data = await api("/list_study_logs?guild_id=" + GUILD_ID);
    logs = data.ok ? (data.logs || []) : [];
    logs.forEach(function(l) {
      if (l.student_id && l.nickname && !nicknameMap[l.student_id]) { nicknameMap[l.student_id] = l.nickname; }
    });
    nicknameMap[STUDENT.id] = STUDENT.nickname;
  } catch(e) { logs = []; }
}
```
- 全員分の勉強ログを取得します。取得したログに含まれるニックネーム情報で、`loadUsers()`でまだ埋まっていなかった分の`nicknameMap`を補完しています（`!nicknameMap[l.student_id]`＝まだ登録が無いIDだけ）。

```js
async function loadPoints() {
  try {
    var data = await api("/get_points?guild_id=" + GUILD_ID);
    if (data.ok) { allPoints = data.points || {}; myPoints  = allPoints[STUDENT.id] || 0; updatePointDisplay(); }
  } catch(e) { allPoints = {}; myPoints = 0; }
}
```
- 全員の累計ポイントを取得し、ヘッダーのポイントバッジ（`updatePointDisplay()`、[02_StudyLog.js_その2_ランキングと記録の描画.md](02_StudyLog.js_その2_ランキングと記録の描画.md)で説明）を更新します。

### 5.1 ログを実際に投稿する：`postLog(entry)`（601〜634行）
```js
async function postLog(entry) {
  var data;
  try {
    data = await api("/add_study_log", { method: "POST", body: JSON.stringify(Object.assign({ guild_id: GUILD_ID, session_token: SESSION_TOKEN }, entry)) });
  } catch(e) {
    return { ok: false, error: "通信エラーが発生しました。もう一度お試しください。" };
  }
  if (!data || data.ok === false) {
    if (data && data.error === "not_logged_in") { forceReLogin(); return { ok: false, error: "ログインが切れました。再度ログインしてください。" }; }
    return { ok: false, error: (data && data.error) || "記録に失敗しました。" };
  }
  var earned = (data.earned != null) ? data.earned : Math.floor(entry.minutes / 5);
  if (earned > 0) {
    allPoints[STUDENT.id] = (data.total != null) ? data.total : (allPoints[STUDENT.id] || 0) + earned;
    myPoints = allPoints[STUDENT.id];
    floatPoints("+" + earned + "pt");
    updatePointDisplay();
  }
  nicknameMap[STUDENT.id] = STUDENT.nickname;
  logs.push(entry);
  renderAll();
  return { ok: true };
}
```
- 手入力・タイマーの両方から呼ばれる、勉強ログを実際にサーバーへ送信する共通関数です。コメントに過去の設計判断の変更が記録されています：「以前は通信エラー時もローカルだけ成功扱いにしていたが、サーバー側の不正防止チェック（連続記録の制限など）を無意味にしてしまうため、失敗はきちんと失敗として扱う」。つまり、通信が失敗した場合に「見た目だけ成功したことにする」という、ユーザーに優しそうに見える実装を、あえてやめた経緯があります。サーバー側が「これは記録として認められません」と判断したものを、クライアント側の判断で勝手に成功扱いにしてしまうと、ポイント制度の不正防止の仕組みが意味を失ってしまうためです。
- 獲得ポイント（`earned`）は、サーバーが計算した値（`data.earned`）を優先して使い、無ければクライアント側でも同じ計算式（5分で1pt）を使って見た目上のフォールバックをします。
- `floatPoints("+5pt")`（[03_StudyLog.js_その3_タブ表示・手入力・課題達成.md](03_StudyLog.js_その3_タブ表示・手入力・課題達成.md)で説明）は、獲得ポイントをふわっと浮かび上がるアニメーションで表示する演出です。

---

続きは[02_StudyLog.js_その2_ランキングと記録の描画.md](02_StudyLog.js_その2_ランキングと記録の描画.md)で、週間ランキングの集計とホーム画面の描画処理を解説します。
