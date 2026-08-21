# サービス情報ページ — 全体像と `ServiceInfo.js` 解説

対象：`bot.1istudy.web/ServiceInfo.html`（250行）、`bot.1istudy.web/ServiceInfo.js`（410行）。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

## このページの特徴

「🤖 Bot」「🌐 Web」「🗂️ 運用ログ」の3タブで構成される、更新履歴・監査ログの表示ページです。[サーバー側の運用ログ機能](../../1I勉強会bot解説/02_データ保存基盤/01_運用ログとdiff表示.md)のフロントエンド実装がここにあります。

面白い点として、**「Bot」タブと「Web」タブの更新履歴は、JSで動的に生成されるのではなく`ServiceInfo.html`に直接ベタ書きされています**（117〜221行、`<li class="service-update-item">`の並び）。これは、Botリポジトリ側の運用ルール「新しいエントリは各カード（Discord Bot / Webサイト）の`<ul class="service-update-list">`の先頭に追加する」と一致しており、コミットのたびに人（Claude Code）が手でこのHTMLに追記している、という運用になっているようです。JSが動的に取得・描画するのは**「運用ログ」タブだけ**です。

---

## 1. タブ切り替え（160〜173行）

```js
function switchServiceTab(tab) {
  document.querySelectorAll('.service-tab-btn').forEach(btn => { btn.classList.toggle('active', btn.dataset.tab === tab); });
  document.querySelectorAll('.service-tab-panel').forEach(panel => { panel.classList.toggle('active', panel.dataset.tabPanel === tab); });
  if (tab === 'log') loadSystemLog();
}
```
- コメントに「3つのカードを全部並べると縦にとても長くなるため、1つだけ表示する」とあります。他ページの`showTab`/`showScreenQ`と同じ、単純な`active`クラスの付け替えです。
- 「運用ログ」タブを開いたタイミングで**毎回**最新の内容に取り直します（キャッシュして使い回さない）。

---

## 2. 運用ログの読み込み：`loadSystemLog()`（343〜404行）

```js
const session = getLoginSession();
let url = `${API_BASE}system_log?limit=${logDisplayCount}`;
const headers = {};
if (session && session.session_token) {
  url += `&guild_id=${GUILD_ID}`;
  headers['Authorization'] = `Bearer ${session.session_token}`;
}
const res = await fetch(url, { cache: 'no-store', headers });
```
- コメントに「ログインしていなくても閲覧できる（利用者11人の小規模運用のため）。ただし、対象サーバーに参加していない制限付きアカウントでログイン中の場合だけは見せない」とあります。未ログインなら`guild_id`も`Authorization`も付けずに送り、サーバー側は誰でも見せる想定です。ログイン中なら、`guild_id`とトークンを一緒に送ることで、サーバー側が「このアカウントは対象サーバーに参加しているか」を確認できるようにしています。
- **未ログインの場合はあえて`guild_id`を送らない**という設計が特徴的です。もし未ログインでも`guild_id`だけ送ってしまうと、サーバー側の実装によっては中途半端なチェックが働いてしまう可能性があるため、「本人確認ができるときだけ、対象を絞った問い合わせにする」という意図的な使い分けだと考えられます。

```js
} catch (e) {
  ...
  li.textContent = (e && e.code === 'guild_membership_required')
    ? 'このアカウントでは運用ログを閲覧できません（対象のDiscordサーバーに参加していないため）。'
    : 'ログを読み込めませんでした。';
  listEl.appendChild(li);
}
```
- コメントに「制限付きアカウント（対象Discordサーバー未参加）でログイン中の場合はサーバー側が意図的に拒否しているだけで、通信・サーバー側の障害ではない。『読み込めませんでした』という技術的失敗の表現は誤解を招くため区別する」とあります。エラーコード（`guild_membership_required`）を見て、「あなたのアカウントでは見られません」という**意図的な制限**であることを明確に伝え、単なる通信障害と誤解させないようにしています。

### 2.1 「もっと見る」の実装（196・379〜389行）
```js
let logDisplayCount = 50; // ★「もっと見る」を押すたびに増やして再取得する（件数自体は多くないため単純な方式でよい）
...
if (data.entries.length >= logDisplayCount) {
  const more = document.createElement('button');
  more.className = 'log-load-more';
  more.textContent = 'もっと見る';
  more.onclick = () => { logDisplayCount += 50; loadSystemLog(); };
  listEl.parentElement.appendChild(more);
}
```
- [../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`のようなオフセット方式のページングではなく、**「今何件表示したいか」という数を増やして、毎回0件目から丸ごと取り直す**、より単純な方式です。コメントに「サーバー側はoffsetに対応していないため」「件数上限が300件程度の小規模運用なので、これで十分速い」とあります。データ量がそれほど多くないことが分かっているからこそ選べる、割り切ったシンプルな実装です。

---

## 3. ログ1件の描画：`renderLogEntry(entry)`（198〜277行）

```js
const detailFiles = Array.isArray(entry.detail) ? entry.detail.filter(f => f && f.diff) : [];
const hasDetail = detailFiles.length > 0;
```
- コメントに「`entry.detail`は`[{file, diff}, ...]`形式（`bot.py`側の`file_diff()`が作る。`file`は実際のデータファイルのパス、`diff`はGitHubのコミットのような+/-形式のテキスト）」とあります。これは[サーバー側の`file_diff()`](../../1I勉強会bot解説/02_データ保存基盤/01_運用ログとdiff表示.md)がそのまま受け取って表示する部分です。

```js
if (hasDetail) {
  const chevron = document.createElement('span');
  chevron.textContent = '▸';
  summary.appendChild(chevron);
}
...
if (hasDetail) {
  const detail = document.createElement('div');
  detailFiles.forEach(f => detail.appendChild(renderLogFileBlock(f)));
  body.appendChild(detail);
  li.classList.add('is-expandable');
  li.tabIndex = 0;
  li.setAttribute('role', 'button');
  li.setAttribute('aria-expanded', 'false');
  const toggle = () => { const open = li.classList.toggle('is-open'); li.setAttribute('aria-expanded', open ? 'true' : 'false'); };
  li.addEventListener('click', toggle);
  li.addEventListener('keydown', (e) => { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); toggle(); } });
}
```
- 詳細情報（`detail`）が無いエントリは、開閉矢印も付かず、タップしても何も起きません（`hasDetail`が`false`のときはこのブロック自体がスキップされます）。
- `li.tabIndex = 0`／`role="button"`／`aria-expanded`は、**キーボード操作やスクリーンリーダーでもこの行が「開閉できるボタン」だと認識されるようにする**、アクセシビリティへの配慮です。`tabIndex = 0`にすることで、マウスを使わずTabキーでこの行にフォーカスを移動できるようになり、`keydown`イベントで`Enter`キーやスペースキーでも開閉できるようにしています。これはマウスの`click`イベントだけに頼った他の多くの箇所とは一線を画す、より丁寧な作りです。

---

## 4. ファイル単位の差分表示：`renderLogFileBlock(f)`（286〜341行）

```js
if (f.file) {
  wrap.classList.add('is-collapsed'); // ★ 既定は閉じた状態
  ...
  if (f.status === 'added' || f.status === 'deleted') {
    const badge = document.createElement('span');
    badge.className = 'log-item-file-badge is-' + (f.status === 'added' ? 'add' : 'del');
    badge.textContent = f.status === 'added' ? '新規作成' : '削除';
    header.appendChild(badge);
  }
  header.addEventListener('click', (e) => {
    e.stopPropagation(); // ★ 親（ログ行全体の開閉）に伝播させない
    wrap.classList.toggle('is-collapsed');
  });
  wrap.appendChild(header);
}
```
- コメントに「GitHubのコミット画面の『変更されたファイル』表示と同じ考え方」とあります。1件のログイベントの中に複数のファイルの変更が含まれる場合（[cards_index.jsonも記録対象に含める](../../1I勉強会bot解説/02_データ保存基盤/01_運用ログとdiff表示.md)といった例）、ファイルごとに個別に折りたたみ・展開できるようにしています。
- `e.stopPropagation()`が重要です。ファイル見出しをタップしたときのクリックが、そのまま親要素（ログ行全体の開閉、3節）にも伝わってしまうと、ファイル1つを開閉したつもりが行全体まで閉じてしまう、という意図しない動作になります。これを防ぐため、ファイル見出しのクリックはそこで止めています。
- `f.file`が無い場合（コメントに「削除依頼の理由等、対象がファイル1つに対応しない場合」とあります）は、見出し自体を作らず、常に中身を表示したままにします。

```js
const lines = document.createElement('div');
String(f.diff).split('\n').forEach(line => {
  const lineEl = document.createElement('div');
  if (line.startsWith('+ ')) { lineEl.className = 'log-item-diff-line is-add'; lineEl.textContent = line.slice(2); }
  else if (line.startsWith('- ')) { lineEl.className = 'log-item-diff-line is-del'; lineEl.textContent = line.slice(2); }
  else { lineEl.className = 'log-item-diff-line'; lineEl.textContent = line; }
  lines.appendChild(lineEl);
});
```
- サーバーから送られてきた`diff`テキスト（改行区切りの、`+`/`-`から始まる行の並び）を1行ずつ分解し、`+`で始まる行は緑（追加）、`-`で始まる行は赤（削除）に色分けします。`line.slice(2)`で、色分けの目印として使った`"+ "`/`"- "`（記号＋半角スペース）自体は表示に含めず、実際の内容だけを表示します。

---

## まとめ

`ServiceInfo.js`は、これまでのページと違い、**更新履歴の本体（Bot/Webタブ）はHTMLに直接書かれており、JS側は「運用ログ」タブの取得・描画だけを担当する**という、珍しい役割分担のページでした。運用ログの表示部分は、Botリポジトリ側に記録されている長い試行錯誤（「本人にのみ表示される情報は載せない」「ファイル名・差分をGitHubのコミット風に」など）の集大成であり、キーボード操作への配慮やクリックの伝播制御など、細部まで作り込まれている点が印象的でした。
