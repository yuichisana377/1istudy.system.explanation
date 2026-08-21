# 削除の確認ページ — 全体像と `DeleteApproval.js` 解説

対象：`bot.1istudy.web/DeleteApproval.html`（118行）、`bot.1istudy.web/DeleteApproval.js`（135行）。これがこのシリーズで扱う最後のページです。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

## このページの特徴

[../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md](../02_Cardmaker/03_Cardmaker.js_その3_デッキの読み込みと作成編集.md)・[../06_Notice/01_Notice.js_その2_削除依頼とアップロード編集.md](../06_Notice/01_Notice.js_その2_削除依頼とアップロード編集.md)で見た「作成者本人以外が削除しようとしたときの削除依頼」の、**依頼を受け取った側（作成者本人）が確認するページ**です。Discordに届くDMのリンク（`?token=...`）を開くと表示されます。

冒頭のコメントに設計方針がまとめられています：
- **ログイン不要**：「トークン自体がDMで本人にだけ届く合言葉の代わり」。学籍番号やパスワードでの本人確認を求める代わりに、Discordの本人だけが受け取れるDM内のリンク（トークン付きURL）を持っていること自体が本人確認の代わりになる、という設計です。
- **表示内容は必ずDOM APIで組み立てる**：「依頼理由・お知らせ本文・カードの問題文などは、すべて利用者が入力したテキストなので、`Notice.js`等と同じ理由で必ず`textContent`/DOM APIで組み立てる（`innerHTML`+テンプレート文字列は禁止）」。このページも[../06_Notice/00_HTML構造とその1_一覧と詳細表示.md](../06_Notice/00_HTML構造とその1_一覧と詳細表示.md)で見たXSS対策の教訓を踏まえた作りになっています。

`DeleteApproval.html`は`Login.css`を流用しており（`login-wrap`/`login-card`などのクラス名がそのまま使われています）、ログインページと似た「合言葉ページ」らしい簡素な見た目です。ドロワーは無く、`Icons.js`・`SwipeGuard.js`・`DeleteApproval.js`だけを読み込みます。

画面は3段階（`step-loading`／`step-error`／`step-body`）で、[../09_Login/00_Login解説.md](../09_Login/00_Login解説.md)と同じ`style.display`直接操作の切り替え方式です。

---

## 1. 起動と情報取得：`init()`（38〜62行）

```js
function getToken() { return new URLSearchParams(location.search).get('token') || ''; }

async function init() {
  currentToken = getToken();
  if (!currentToken) {
    showStep('error');
    qs('error-msg').textContent = 'リンクが正しくありません。DMのリンクからもう一度開いてください。';
    return;
  }
  try {
    const res = await fetch(`${API_BASE}delete_request_info?token=${encodeURIComponent(currentToken)}`, { cache: 'no-store' });
    const data = await res.json();
    if (!data.ok) {
      showStep('error');
      qs('error-msg').textContent = data.error || 'リンクが無効か、期限切れです。';
      return;
    }
    renderRequest(data);
    showStep('body');
  } catch (e) {
    showStep('error');
    qs('error-msg').textContent = 'サーバーに接続できませんでした。';
  }
}
```
- URLの`?token=...`が無ければ、そもそも不正なアクセスとしてエラー画面を表示します。
- トークンがあれば`delete_request_info`というAPIに問い合わせ、「今まさに削除されようとしている中身」を取得します。[サーバー側の実装](../../1I勉強会bot解説/12_FlaskAPI_削除依頼システム/01_承認ページのAPIと結果の反映.md)で見た通り、これは「依頼時点のスナップショットではなく閲覧時点の最新内容」を返す設計になっており、依頼が送られたあとで対象がさらに編集されていても、この確認ページには**今の実際の内容**が表示されます。
- `data.ok`が偽なら、サーバーが返してきたエラーメッセージ（期限切れ・既に処理済みなど）をそのまま表示します。

---

## 2. 依頼内容の表示：`renderRequest(data)`（64〜86行）

```js
function renderRequest(data) {
  currentCategory = data.category;
  const categoryLabel = data.category === 'deck' ? 'カードデッキ' : 'お知らせ';
  qs('da-title').textContent = `${categoryLabel}を削除してよいか確認してください`;
  qs('da-target-label').textContent = `対象の${categoryLabel}`;
  qs('da-requester').textContent = `${data.requester_nickname || '（不明）'} さん`;
  qs('da-reason').textContent = data.reason || '（理由が入力されていません）';
  qs('da-target-name').textContent = data.target_name || '（不明）';
  const detailEl = qs('da-detail');
  detailEl.textContent = '';
  const lines = data.detail_lines || [];
  if (lines.length) { detailEl.textContent = lines.join('\n'); }
  else { detailEl.textContent = '（内容を表示できませんでした）'; }
  if (data.already_gone) { qs('da-already-gone').style.display = ''; }
}
```
- `data.category`（`'deck'`または`'notice'`）に応じて、画面の文言を「カードデッキ」「お知らせ」のどちらかに出し分けます。1つのページで両方の削除依頼を扱えるようにする、シンプルな条件分岐です。
- コメントの方針通り、すべての値の代入が`textContent`によるものです（`innerHTML`は一切使われていません）。依頼理由（`data.reason`）は依頼した側が自由に書けるテキストであり、対象の名前・内容（`data.target_name`／`data.detail_lines`）も元々は誰かが入力したデータなので、この徹底ぶりは適切な安全対策です。
- `data.detail_lines`（配列）を`.join('\n')`で1つの文字列にまとめ、`textContent`に代入しています。`textContent`は改行文字（`\n`）をそのままでは見た目の改行として表示しませんが、CSS側で`white-space: pre-line`のような指定がされていれば、この`\n`が実際の改行として表示されます（`da-detail`要素のスタイルで対応していると考えられます）。
- `data.already_gone`（対象が既に削除済み、または依頼が取り下げられている）が真なら、「以下の内容は依頼された時点のものです」という注記を表示します。

---

## 3. 承諾・拒否の送信：`respond(action)`（88〜128行）

```js
async function respond(action) {
  hideActionError();
  const approveBtn = qs('btn-approve');
  const rejectBtn = qs('btn-reject');
  approveBtn.disabled = true;
  rejectBtn.disabled = true;
  const clickedBtn = action === 'approve' ? approveBtn : rejectBtn;
  const originalLabel = clickedBtn.textContent;
  clickedBtn.textContent = '処理中…';
  try {
    const res = await fetch(`${API_BASE}respond_delete_request`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token: currentToken, action }),
      signal: AbortSignal.timeout(10000),
    });
    const data = await res.json();
    if (!data.ok) {
      showActionError(data.error || '処理に失敗しました');
      approveBtn.disabled = false; rejectBtn.disabled = false;
      clickedBtn.textContent = originalLabel;
      return;
    }
    qs('da-btn-row').style.display = 'none';
    const resultEl = qs('da-result');
    resultEl.textContent = action === 'approve' ? '削除しました。ご確認ありがとうございました。' : '削除を拒否しました。依頼者にはその旨が伝わります。';
    resultEl.style.display = '';
  } catch (e) {
    showActionError('サーバーに接続できませんでした。');
    approveBtn.disabled = false; rejectBtn.disabled = false;
    clickedBtn.textContent = originalLabel;
  }
}
qs('btn-approve').addEventListener('click', () => respond('approve'));
qs('btn-reject').addEventListener('click', () => respond('reject'));
```
- 「承諾」「拒否」どちらのボタンでも同じ`respond(action)`関数を使い、`action`の値（`'approve'`/`'reject'`）だけを変えて呼び分けます。
- **両方のボタンを同時に無効化**してから送信します（`approveBtn.disabled = true; rejectBtn.disabled = true;`）。これは、片方のボタンだけを無効化すると、もう片方を連打して二重に別のアクションを送信できてしまう可能性があるためです（例えば「承諾」を送信中に、まだ押せる「拒否」を押してしまう、という事故を防ぎます）。
- 押されたボタンの方だけ、元のラベルを`originalLabel`に控えてから「処理中…」に変更します。失敗した場合は両方のボタンを再度有効化し、ラベルも元に戻して、もう一度試せるようにします。
- 成功したら、ボタン列自体を非表示にし（`qs('da-btn-row').style.display = 'none'`）、結果メッセージを表示します。**一度承諾・拒否したら、二度と操作できない画面に切り替わる**という設計です。これは[削除依頼トークンの設計](../../1I勉強会bot解説/12_FlaskAPI_削除依頼システム/00_削除依頼トークンと送信フロー.md)で見た「承認トークンはステートレス設計のため、同じDMリンクを2回押すといった厳密な二重実行防止は持たない（2回目は対象が既に無いので『ファイルが見つかりません』になるだけで実害は無い）」という仕様とも合致しており、サーバー側で完全な二重実行防止がされていなくても、フロント側でボタンを隠すことで通常の操作フローでは再送信が起きないようにしています。

---

## まとめ

`DeleteApproval.js`はこのシリーズで扱った中で最も小さいページですが、「ログイン不要でトークンだけを本人確認の代わりにする」「利用者が入力したテキストを表示する箇所は必ずDOM APIで安全に組み立てる」「削除という取り消せない操作の前に、閲覧時点の最新内容を見せる」といった、これまでのページで積み重ねられてきた設計方針が、コンパクトな形で凝縮されたページでした。

---

以上で、`bot.1istudy.web`の全10ページ・全ファイルの解説が完了しました。[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)で紹介した「web裏側」の記事と合わせて読むことで、このサイトの裏側（データ・セキュリティ・パフォーマンス）と画面側（各ページの実際のコードの中身）の両方を一通りカバーできます。
