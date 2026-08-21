# ログインページ — 全体像と `Login.js` 解説

対象：`bot.1istudy.web/Login.html`（137行）、`bot.1istudy.web/Login.js`（289行）。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

## このページの特徴

他ページからログインが必要になったときに誘導される、ログイン専用ページです。冒頭のコメントに全体のフローがまとめられています：

1. `localStorage`に有効なセッション（`session_token`付き）があれば自動ログイン。
2. `loginWithDiscord()` → Discordの認可画面へ → 認可後サーバー側で分岐：
   - 登録済み → `session_token`をクエリパラメータで受け取りそのままログイン。
   - 初めて／未登録 → `?discord_reg=<dtoken>`付きで戻ってきて登録ステップへ。
3. 遷移先：`sessionStorage`に`post_login_redirect`が保存されていればログイン後にそのページへ戻る（例：`Cardmaker.html`から来た場合）。無ければ通常通り`StudyLog.html`へ遷移する。

`Login.html`は他ページと違い**ドロワーメニューを持ちません**（ログイン前なのでメニューを見せる意味が無いため）。読み込むJSも`Icons.js`と`Login.js`だけで、`Dialog.js`も無く（このページ独自の簡易なエラー表示があるため）、`PendingDeleteCheck.js`もありません（ログイン前なので削除依頼の確認は関係ないため）。

画面は3段階（`login-step`）で、`style.display`の`none`/`空文字`で切り替えます（`open`/`active`クラスではなく、このページだけ直接`display`を操作する方式です）：
- `step-loading`：自動ログイン確認中
- `step-id`：「Discordでログイン」ボタンだけのシンプルな画面
- `step-discord-reg`：初回登録用（学籍番号・ニックネーム入力）

---

## 1. アバターの色パレット（26〜36・96〜101行）

```js
const AVATAR_COLORS = [
  { color: "#dbeafe", text: "#1e40af" },
  { color: "#dcfce7", text: "#166534" },
  { color: "#fce7f3", text: "#9d174d" },
  { color: "#ffedd5", text: "#9a3412" },
  { color: "#fef9c3", text: "#854d0e" },
  { color: "#ede9fe", text: "#6d28d9" },
  { color: "#fee2e2", text: "#991b1b" },
  { color: "#f0fdf4", text: "#15803d" },
];
// ID（文字列）から常に同じ色を選ぶ（新規登録時のインデックスに依存しないように）
function paletteFor(id) {
  let hash = 0;
  for (let i = 0; i < id.length; i++) hash = (hash * 31 + id.charCodeAt(i)) >>> 0;
  return AVATAR_COLORS[hash % AVATAR_COLORS.length];
}
```
- ドロワーやヘッダーに表示される丸いアバターの背景色・文字色は、ここで決まって以降ずっと同じ組み合わせで表示され続けます。`paletteFor(id)`は、[../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md](../02_Cardmaker/01_Cardmaker.js_その1_ログインとデータ管理.md)の`hashStr`と同じ考え方の簡易ハッシュ関数で、**学籍番号の文字列から必ず同じ色が選ばれる**ようにしています。コメントに「新規登録時のインデックスに依存しないように」とあり、「登録した順番」のような変わりうる情報ではなく、学籍番号という不変の情報から色を決めることで、毎回同じ人には同じ色が表示されることを保証しています。
- `>>> 0`は、計算結果を「符号なし32ビット整数」に変換する演算子です。ハッシュの掛け算・足し算を繰り返していくと、JSの数値は非常に大きくなったりマイナスになったりすることがありますが、これを挟むことで常に0以上の扱いやすい整数に収め、`% AVATAR_COLORS.length`（8で割った余り）で0〜7のインデックスを安全に取り出せるようにしています。

---

## 2. 起動処理（38〜73行）

```js
window.addEventListener("load", () => {
  const params = new URLSearchParams(location.search);

  // ★ 既にDiscordログイン登録済みの場合：APIサーバー（別ドメイン）側では
  //   localStorageを共有できないため、session_tokenをURLのクエリパラメータで
  //   受け取り、ここ（フロントエンドのドメイン上）でlocalStorageに保存する。
  const discordToken = params.get("discord_session_token");
  if (discordToken) {
    const studentId = params.get("student_id") || "";
    const nickname  = params.get("nickname") || studentId;
    const palette   = paletteFor(studentId);
    saveSession({ id: studentId, nickname }, discordToken, palette);
    history.replaceState(null, "", location.pathname); // URLからトークンを消す
    location.href = getRedirectTarget();
    return;
  }

  // ★ Discordログインで初回登録が必要な場合、/discord_login_start →
  //   Discord認可 → コールバック経由で ?discord_reg=<dtoken> 付きで
  //   このページに戻ってくる。最優先でそちらを処理する。
  const dtoken = params.get("discord_reg");
  if (dtoken) {
    openDiscordRegisterStep(dtoken);
    return;
  }

  // 自動ログイン（localStorage に session_token 付きの保存済みセッションがある場合のみ）
  const saved = getSession();
  if (saved && saved.session_token) {
    autoLogin(saved);
    return;
  }
  localStorage.removeItem(SESSION_KEY); // session_tokenの無い旧形式セッションは破棄
  showStep("step-id");
});
```
- コメントに「APIサーバー（別ドメイン）側では`localStorage`を共有できないため、`session_token`をURLのクエリパラメータで受け取り、ここ（フロントエンドのドメイン上）で`localStorage`に保存する」とあります。これは[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)で触れた「Cloudflare Pagesのフロント」と「Tailscale経由のバックエンド」がそれぞれ別のドメイン（オリジン）である、という構成に由来する制約です。`localStorage`はドメインごとに独立しているため、バックエンド側でセッションを発行しても、それをフロント側の`localStorage`に直接書き込むことはできません。そこでDiscordの認可が終わったサーバー側は、ログイン情報を**URLのクエリパラメータに載せてこのページに戻す**ことで、フロントのドメイン上で動くこのJSが受け取って`localStorage`に保存する、という橋渡しをしています。
- 受け取ったら`history.replaceState`でURLからトークンを消し（ブラウザ履歴やアクセスログに残さないため）、目的地へ遷移します。
- `discord_reg`パラメータがあれば初回登録が必要な場合の分岐です（4節で詳しく解説）。
- 上記どちらの特別なURLパラメータも無ければ、既存のセッションが保存されていないか確認します。あれば自動ログイン（3節）、無ければ`step-id`（ログインボタンだけの画面）を表示します。`session_token`の無い古い形式のセッションが残っていた場合は、この機会に消しておきます（パスワード方式が廃止された経緯にある、過去の名残データの掃除です）。

---

## 3. セッションの保存と自動ログイン（75〜124行）

```js
function saveSession(user, sessionToken, colorPalette) {
  const session = {
    student_id: user.id, nickname: user.nickname,
    color: colorPalette.color, text_color: colorPalette.text,
    session_token: sessionToken, logged_in_at: new Date().toISOString(),
  };
  try { localStorage.setItem(SESSION_KEY, JSON.stringify(session)); } catch {}
  return session;
}
```
- 他ページで何度も見た`sl_session`という`localStorage`キーの中身が、実際にここで組み立てられていることが分かります。

```js
function getRedirectTarget() {
  const savedRedirect = sessionStorage.getItem('post_login_redirect');
  if (savedRedirect) { sessionStorage.removeItem('post_login_redirect'); return savedRedirect; }
  return REDIRECT_PATH;
}
```
- 他ページ（[../01_index_予定管理.md](../01_index_予定管理.md)の`requireLoginOrRedirect`など）が「未ログインならログイン画面へ」と誘導する直前に控えておいた`post_login_redirect`（元々見ていたページのURL）を、ここで**受け取る側**として使います。あればそこへ戻し、無ければデフォルトの`StudyLog.html`へ。

```js
async function autoLogin(session) {
  showStep("step-loading");
  setLoadingMsg("ログイン情報を確認中…");
  location.href = getRedirectTarget();
}
```
- コメントに「`session_token`の有効性はサーバー側でしか判定できない（改ざん・期限切れ等）ので、軽いAPIを叩いて通信自体が生きているかだけ確認し、トークンの正当性チェック自体は`StudyLog.html`側の各APIコールに任せる（そちらで`not_logged_in`が返ってくれば自動的にログイン画面へ戻される）」とあります。この関数自体は実際にはトークンの検証をせず、「読み込み中…」を一瞬見せてからすぐに遷移先へ飛ばすだけです。もしトークンが実際には無効だったとしても、遷移先のページ（[../04_StudyLog/*](../04_StudyLog/00_HTML構造と全体像.md)など）が独自に`not_logged_in`エラーを検知して、改めてこのログインページへ戻す仕組みになっているため、ここで二重にチェックする必要が無い、という設計です。

---

## 4. Discordログインと初回登録（126〜260行）

```js
function loginWithDiscord() {
  location.href = `${API_BASE}discord_login_start?guild_id=${GUILD_ID}`;
}
```
- Discordの認可を開始するAPIへ、ブラウザごと移動します。コメントに、認可後の3通りの分岐がまとめられています：①既に登録済みならそのままセッションが発行されて`StudyLog.html`へ自動遷移（このページには戻らない）、②初めて／未登録なら`?discord_reg=<dtoken>`付きでこのページに戻る、③対象サーバーのメンバーでなければサーバー側がエラーページを表示する（このページには戻らない）。

### 4.1 登録画面を開く：`openDiscordRegisterStep(dtoken)`（188〜209行）
```js
async function openDiscordRegisterStep(dtoken) {
  document.getElementById("inp-dtoken").value = dtoken;
  document.getElementById("discord-reg-err").style.display = "none";

  showStep("step-discord-reg");

  try {
    const info = await fetchDiscordRegInfo(dtoken);
    if (info.ok && info.discord_username) {
      // Discordの表示名をニックネームの初期値として提案する（そのまま使うかは本人の自由）
      const nickEl = document.getElementById("inp-discord-nickname");
      if (nickEl && !nickEl.value) nickEl.value = info.discord_username.slice(0, 16);
    } else if (!info.ok) {
      showDiscordRegErr("このリンクの有効期限が切れています。もう一度「Discordでログイン」からやり直してください。");
    }
  } catch {
    // 参考情報の取得に失敗しても、登録フォーム自体は使えるので致命的ではない
  }

  // URLに残った ?discord_reg=... を消しておく（再読み込みで壊れないように）
  history.replaceState(null, "", location.pathname);
}
```
- `dtoken`（Discord認可直後に発行される、登録専用の一時トークン）を隠しフィールドに保存しつつ、Discordの表示名を`discord_reg_info`というAPIから取得し、ニックネームの初期値として提案します（「そのまま使うかは本人の自由」とコメントにあり、あくまで入力の手間を減らす親切機能です）。この取得に失敗しても、登録フォーム自体（学籍番号・ニックネームの手入力）は変わらず使えるため、致命的なエラーとしては扱いません。

### 4.2 登録の送信：`submitDiscordRegister()`（211〜245行）
```js
async function submitDiscordRegister() {
  const dtoken   = document.getElementById("inp-dtoken").value;
  const id       = document.getElementById("inp-discord-student-id").value.trim().toUpperCase();
  const nickname = document.getElementById("inp-discord-nickname").value.trim();
  const btnEl    = document.getElementById("btn-discord-reg-submit");

  if (!validateDiscordId(id)) return;

  setBtn(btnEl, true, "登録中…");

  try {
    const result = await completeDiscordRegistration(dtoken, id, nickname);

    if (result.ok) {
      const palette = paletteFor(result.student.id);
      saveSession(result.student, result.session_token, palette);
      location.href = getRedirectTarget();
      return;
    }

    if (result.error === "nickname_required") {
      showDiscordRegErr("ニックネームを入力してください");
      return;
    }
    if (result.error === "reg_token_invalid") {
      showDiscordRegErr("このリンクの有効期限が切れています。もう一度「Discordでログイン」からやり直してください。");
      return;
    }
    showDiscordRegErr("登録に失敗しました。時間をおいて再試行してください。");
  } catch {
    showDiscordRegErr("サーバーに接続できません。時間をおいて再試行してください。");
  } finally {
    setBtn(btnEl, false, "登録してログイン " + Icons.html('check', {size:15}));
  }
}
```
- 学籍番号は`.toUpperCase()`で大文字に統一してから送信します。
- コメントに「すでに登録済みの学籍番号を入力した場合は、そのままDiscordアカウントに紐付けてログインします」とある通り、このフォームは「新規登録」と「既存アカウントへのDiscord連携」の両方を兼ねています。サーバー側が学籍番号の存在を見て、新規作成するか既存アカウントに連携するかを判断していると考えられます。
- `dtoken`が期限切れの場合（`reg_token_invalid`）は、もう一度「Discordでログイン」からやり直すよう案内します。この一時トークンには有効期限があり、認可からあまり時間が経ちすぎると使えなくなる、という設計です。

```js
function validateDiscordId(raw) {
  if (!raw) { showDiscordRegErr("学籍番号を入力してください"); return false; }
  if (!/^[A-Z0-9]{2,20}$/.test(raw)) { showDiscordRegErr("半角英数字で入力してください（例: 1I001）"); return false; }
  return true;
}
```
- 学籍番号の形式チェックです。半角英数字2〜20文字という制限を、送信前にクライアント側でも確認しています（もちろんサーバー側でも同様のチェックがあると考えられます）。

---

## 5. ステップ切り替えとユーティリティ（150〜288行）

```js
function showStep(id) {
  document.querySelectorAll(".login-step").forEach(el => { el.style.display = el.id === id ? "" : "none"; });
  if (id === "step-discord-reg") {
    setTimeout(() => document.getElementById("inp-discord-student-id")?.focus(), 60);
  }
}
```
- 登録ステップに切り替わったときは、少し遅らせて（60ミリ秒後）学籍番号の入力欄に自動でフォーカスを当てます。画面が切り替わった直後だとフォーカスが正しく当たらないブラウザがあるための、ごく短い遅延だと考えられます。

```js
document.addEventListener("keydown", e => {
  if (e.key !== "Enter") return;
  const active = [...document.querySelectorAll(".login-step")].find(el => el.style.display !== "none");
  if (!active) return;
  if (active.id === "step-discord-reg") submitDiscordRegister();
});
```
- 今**表示されているステップ**を`style.display !== "none"`という条件で見つけ出し、それが登録ステップなら、Enterキーで送信できるようにしています。フォームの`<form>`タグによる標準的なEnter送信ではなく、ページ全体のキーボードイベントを監視して、今どのステップを見ているかで振る舞いを変える、という実装です。

```js
function setBtn(el, disabled, label) {
  if (!el) return;
  el.disabled     = disabled;
  el.innerHTML    = label;
}

function escHtml(s) {
  return String(s).replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
}
```
`setBtn(el, disabled, label)`（267〜271行）と`escHtml(s)`（273〜275行）は、他ページで何度も見たパターンの小さなヘルパーです。

---

## まとめ

`Login.js`は、他ページと比べて機能自体はシンプル（ログインボタンと初回登録フォームだけ）ですが、**別ドメイン間でのセッション情報の受け渡し**（URLクエリパラメータ経由）や、**登録と連携を1つのフォームで兼ねる**設計など、認証まわり特有の工夫が凝縮されたページでした。トークンの正当性チェックを自分では行わず、遷移先のページに委ねるという役割分担も、無駄な二重チェックを避けるための合理的な設計だと言えます。

---

続きは[../10_DeleteApproval/00_DeleteApproval解説.md](../10_DeleteApproval/00_DeleteApproval解説.md)で、削除承認ページ（`DeleteApproval.html`/`DeleteApproval.js`）を解説します。これで`bot.1istudy.web`の全ページの解説が完了します。
