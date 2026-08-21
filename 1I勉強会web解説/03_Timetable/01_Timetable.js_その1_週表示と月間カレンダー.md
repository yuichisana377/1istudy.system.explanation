# Timetable.js その1：ログイン・週表示・月間カレンダー（1〜778行）

[00_HTML構造とページ全体像.md](00_HTML構造とページ全体像.md)の続きです。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

---

## 1. 冒頭：ログイン・アカウント表示・定数（1〜206行）

6〜122行（`API_BASE`/`GUILD_ID`/`SESSION_KEY`/`getLoginSession`/`requireLoginOrRedirect`/`renderDrawerAccount`/`POINT_CATEGORIES`/`POINT_OPTIONS`/`NOTE_SEP`）は、[../01_index_予定管理.md](../01_index_予定管理.md)の`Plan.js`とほぼ**一字一句同じ内容**です。詳しい説明はそちらを参照してください。

1点だけ違いがあります。`Plan.js`はページを開いた瞬間に未ログインなら強制的にログイン画面へ飛ばす処理（IIFE）がありましたが、**時間割ページにはその処理がありません**。コメントに理由が書かれています：「このページ自体は閲覧にログイン不要（誰でも時間割を見られる）だが、追加・編集・削除はサーバー側もログイン必須になった」。つまり、時間割の「見るだけ」は誰でもできる仕様のため、ページを開いた時点でのログイン強制は行わず、`requireLoginOrRedirect()`（変更操作の直前に呼ぶ）だけでガードする、という設計です。

### 1.1 時間割の初期値：`DEFAULT_TIMETABLE`（130〜159行）
```js
const DEFAULT_TIMETABLE = {
  mon: [ { subject: "コンピュータリテラシ", items: ["教科書"] }, ... ],
  ...
};
```
- 曜日ごとに、時限順の「科目名」と「持ち物リスト」を並べた固定データです。コメントには「学期の時間割が未設定の期間に使うフォールバック用」とあります。前期・後期など学期ごとの時間割は、後述の`terms`（学期データ）で管理され、**どの学期にも当てはまらない期間だけ**、このデフォルト値が使われます。

### 1.2 APIエンドポイントの定数化（166〜179行）
```js
const TT_API = {
  UPDATE:         '/update_timetable',
  HOLIDAY:        '/set_holiday',
  PERIOD_HOLIDAY: '/set_period_holiday',
  DELETE:         '/delete_timetable',
  LIST:           '/list_timetable',
};
const TERM_API = { LIST: '/list_terms', SAVE: '/save_term', DELETE: '/delete_term' };
```
- サーバーとの通信先のパスを、直接文字列で書かず、意味のある名前を付けた定数としてまとめています。こうしておくと、コード中で`TT_API.HOLIDAY`のように書けて読みやすく、また実際のパスが変わったときもここ1箇所を直せば済みます。

### 1.3 グローバル状態（184〜206行）
- `weekOffset`：今週から何週間ずらして表示しているか（0が今週）。
- `ttActiveDay`：週表示で今どの曜日タブ（0=月〜4=金）を開いているか。
- `monthOffset`：月間カレンダーが今月から何ヶ月ずれているか。
- `ttHomeworks`：宿題・課題の一覧（[../01_index_予定管理.md](../01_index_予定管理.md)の`list_schedule`から取得したもの。時間割上に表示するため）。
- `ttOverrides`：休講・時限変更・曜日変更など「基本の時間割からの上書き」を、後述のキー形式で保持するオブジェクト。
- `terms`：学期ごとの基本時間割データ。
- `plans`/`channels`/`calState`/`editTarget`/`delTarget`/`selectedPoints`：予定管理機能（[04_Timetable.js_その4_予定管理モーダル共通処理.md](04_Timetable.js_その4_予定管理モーダル共通処理.md)で解説）用の状態で、`Plan.js`と同じ変数です。

---

## 2. 起動処理（211〜256行）

```js
function adjustWeekForWeekend() {
  const today = new Date().getDay(); // 0=日, 6=土
  if (today === 0 || today === 6) {
    weekOffset = 1;
    ttActiveDay = 0; // 月曜日を開く
  } else {
    weekOffset = 0;
    ttActiveDay = today - 1; // 月〜金 → 0〜4
  }
}
window.addEventListener('load', () => {
  adjustWeekForWeekend();
  loadTTHomeworks();
  loadTTOverrides();
  loadTerms();
  loadChannels();
  loadPlans();
  renderTimetable();
  prefetchOtherPages();
});
```
- `adjustWeekForWeekend()`は、ページを開いた瞬間に「今が土日なら、来週の月曜日を最初に表示する」という親切な調整をする関数です。平日に開いた場合は、今日の曜日タブを最初から開いておきます。
- ページ読み込み時に、時間割に関するデータ（課題・時間割の変更履歴・学期設定）と、予定管理機能に関するデータ（科目一覧・予定一覧）を同時に読み込み始め、`renderTimetable()`で画面を描画します。

`api(path, opts)`（246〜255行）は`Plan.js`の`api()`とほぼ同じ共通APIヘルパーです。コメントには「このページ自体（時間割の表示）は引き続き閲覧にログイン不要のままなので、未ログイン時は宿題の表示分だけ（`list_schedule`起因の部分だけ）空になる形で許容する」とあります。つまり、未ログインで時間割を見ることはできますが、その場合は課題・提出物の表示だけが（ログインが必要なAPIのため）欠けた状態になる、という仕様です。

---

## 3. 時間割データの読み込み（257〜291行）

```js
async function loadTTHomeworks() {
  try {
    const data = await api(`/list_schedule?guild_id=${GUILD_ID}`);
    ttHomeworks = (data.ok && Array.isArray(data.plans)) ? data.plans : [];
  } catch(e) { ttHomeworks = []; }
  renderTimetable();
}
async function loadTerms() {
  try {
    const res  = await api(`${TERM_API.LIST}?guild_id=${GUILD_ID}`);
    terms = (res.ok && Array.isArray(res.terms)) ? res.terms : [];
  } catch(e) { terms = []; }
  renderTimetable();
}
```
- どちらも、失敗時は空配列にフォールバックしてから、必ず`renderTimetable()`を呼んで画面を更新します。

```js
function getTimetableForDate(dateStr) {
  const term = terms.find(t => t.start_date <= dateStr && dateStr <= t.end_date);
  return (term && term.timetable) ? term.timetable : DEFAULT_TIMETABLE;
}
```
- 指定した日付が、登録されているどれかの学期の期間内（`start_date`〜`end_date`）にあれば、その学期専用の時間割を使い、どの学期にも当てはまらなければ`DEFAULT_TIMETABLE`（1.1節）にフォールバックします。コメントには「複数の学期が同じ日付に重なることは保存時にサーバー側で防いでいるため、最初に一致したものを使えばよい」とあり、データの整合性はサーバー側の保存処理が保証している前提で、ここではシンプルに最初の一致を使うだけになっています。

---

## 4. 週の日付計算（293〜348行）

```js
function getWeekDates() {
  const now = new Date();
  const day = now.getDay();
  const mon = new Date(now);
  mon.setDate(now.getDate() - (day === 0 ? 6 : day - 1) + weekOffset * 7);
  return DAY_KEYS.map((_, i) => {
    const d = new Date(mon);
    d.setDate(mon.getDate() + i);
    return d;
  });
}
```
- 今日の日付から、その週の月曜日を逆算します（`day === 0 ? 6 : day - 1`は「日曜日なら6日戻る、それ以外は`曜日番号-1`日戻る」という意味で、日曜日だけ特別扱いしているのは、JSの`getDay()`が日曜日を0として返すためです）。
- そこに`weekOffset * 7`日を足すことで、「何週間先/前の月曜日」を求め、そこから月〜金の5日分の日付を配列で返します。

`formatWeekLabel`/`getDateStr`/`getTodayDayIndex`/`dateToDayKey`/`moveWeek`/`goToday`/`ttSwitchDay`は、それぞれ「週の表示ラベルを作る」「`Date`を`YYYY-MM-DD`文字列にする（[../01_index_予定管理.md](../01_index_予定管理.md)の`todayLocalStr`と同じ考え方）」「今日が月〜金の何番目か（土日なら-1）」「日付文字列から曜日キーを逆算する」「週を移動して再描画」「今日の週・曜日に戻して再描画」「曜日タブを切り替えて再描画」という、小さな役割を持つ関数群です。

---

## 5. 時間割本体の描画：`renderTimetable()`（350〜465行）

この関数が、週表示画面全体を組み立てる中心的な処理です。

```js
const dates      = getWeekDates();
const todayIdx   = getTodayDayIndex();
const isThisWeek = weekOffset === 0;
weekLabelEl.textContent = formatWeekLabel(dates);
```
- 週の日付・今日の曜日インデックス・今表示しているのが今週かどうかを求め、ヘッダーの週ラベルを更新します。

### 5.1 「今日」バナー（363〜373行）
今週を表示していて、かつ今日が平日なら、「8月20日（木）　今日」のようなバナーを表示します。

### 5.2 その日が休校かどうか（382〜396行）
```js
const holidayKey = `holiday:${dateStr}`;
const holidayOv  = ttOverrides[holidayKey];
const basePeriods = getTimetableForDate(dateStr)[dayKey] || [];
```
- `ttOverrides`（休講・変更などの上書きデータ）は、`種類:日付` または `種類:日付:時限` という形式のキーを持つオブジェクトです（`holiday:2026-08-20`のような形）。この日付が終日休校として登録されていれば、コマの中身は描画せず、休校理由だけを表示するブロックに切り替えます。

### 5.3 1コマずつの描画（397〜443行）
休校でなければ、その曜日の基本時限（`basePeriods`）を1つずつ処理します：

```js
const periodHolidayKey = `period_holiday:${dateStr}:${periodNum}`;
const periodHolidayOv  = ttOverrides[periodHolidayKey];
if (periodHolidayOv) {
  // その時限だけ「休み」として表示して次へ
}
const changeKey = `change:${dateStr}:${periodNum}`;
const changeOv  = ttOverrides[changeKey];
const subject   = changeOv ? (changeOv.subject || p.subject) : p.subject;
const items     = changeOv ? (changeOv.items   || [])        : p.items;
const isChanged = !!changeOv;
```
- まず「その時限だけの休み」（`period_holiday:`キー）があれば、その時限だけグレーの「休み」表示にします。
- なければ「授業変更」（`change:`キー）があるかを確認し、あれば変更後の科目・持ち物で上書きして表示し、「変更」バッジを付けます。無ければ通常通り、基本時間割の内容をそのまま表示します。

```js
const hw = ttHomeworks.filter(h => h.date === dateStr && h.subject === subject);
const hwHtml = hw.map(h => {
  const { cat, text } = parsePlanContent(h.content);
  return `<div class="homework-row"><span class="tt-badge tt-badge-${escapeAttr(cat)}">${escapeAttr(cat)}</span> <span class="homework-text">${escapeAttr(text)}</span></div>`;
}).join('');
```
- その日・その科目に対応する課題（`ttHomeworks`、[../01_index_予定管理.md](../01_index_予定管理.md)の予定データと同じ形式）を絞り込み、時間割の下に一緒に表示します。`parsePlanContent`（[../01_index_予定管理.md](../01_index_予定管理.md)で解説）で予定の`content`文字列からカテゴリ・本文を取り出すロジックも、`Plan.js`とまったく同じものが使われています（後述の`escapeAttr`という名前の関数ですが、実質`esc()`と同じ働きをする関数です）。

### 5.4 全体を組み立てる（445〜465行）
最後に、曜日タブ（月〜金のボタン列）・選択中の曜日の時間割カード・月間カレンダー（次節）を1つの文字列として連結し、`main.innerHTML`に一括代入します。

---

## 6. 月間カレンダー（467〜625行）

週表示の下に表示される、もう1つの日付ナビゲーションです。

### 6.1 その日の予定を集める：`getDatePlanItems(dateStr)`（471〜489行）
```js
const pushItem = (subject, raw) => {
  const { cat, text, note } = parsePlanContent(raw);
  const key = `${subject}|${cat}|${text}`;
  if (seen.has(key)) return;
  seen.add(key);
  items.push({ subject, cat, text, note });
};
plans.filter(p => p.date === dateStr).forEach(p => pushItem(p.subject, p.content));
ttHomeworks.filter(h => h.date === dateStr).forEach(h => pushItem(h.subject, h.content));
```
- 指定した日の予定を、`plans`（予定管理機能で読み込んだデータ）と`ttHomeworks`（時間割用に読み込んだ課題データ）の**両方**から集めます。実はこの2つは同じサーバーAPI（`list_schedule`）を別々の関数（`loadPlans`と`loadTTHomeworks`）がそれぞれ独自に呼んで保持しているため、同じ予定が両方に重複して存在する可能性があります。`seen`という`Set`で「科目｜カテゴリ｜本文」の組み合わせがすでに登場していないかチェックすることで、月間カレンダー上では重複させずに1件として表示します。

### 6.2 カレンダーグリッドの組み立て：`buildMonthCalendarHtml()`（491〜544行）
```js
const firstDow  = new Date(year, month, 1).getDay();
const daysInMon = new Date(year, month + 1, 0).getDate();
let cells = '';
for (let i = 0; i < firstDow; i++) cells += `<div class="mc-day mc-empty"></div>`;
for (let d = 1; d <= daysInMon; d++) { ... }
const totalCells = firstDow + daysInMon;
const trailing = (7 - (totalCells % 7)) % 7;
for (let i = 0; i < trailing; i++) cells += `<div class="mc-day mc-empty"></div>`;
```
- [../01_index_予定管理.md](../01_index_予定管理.md)の自作カレンダー（`renderCal`）と同じ考え方で、月初の空白セル・実際の日付セルを組み立てますが、こちらはさらに**月末の空白セル**（`trailing`）も計算して埋めています。`totalCells % 7`で「7で割った余り」を求め、`7 - 余り`（ただし余りが0ならそのまま0）で、グリッドをちょうど7の倍数マスにそろえるために必要な空白の数を計算しています。これにより、カレンダーの見た目が毎月きれいな長方形になります。
- 各日付セルには、予定がある日・終日休みの日を示す小さな点（`mc-dots`）が付きます。

### 6.3 日付タップでの詳細表示：`onMonthDayClick(dateStr)`（556〜600行）
月間カレンダーの日付をタップすると、その日の休校情報・予定一覧をモーダルで表示します。休校情報も予定も無ければ「この日の予定はありません」と表示します。

### 6.4 週表示への切り替え：`jumpToDayFromMonth()`（603〜625行）
```js
const now = new Date();
const day = now.getDay();
const thisMon = new Date(now);
thisMon.setDate(now.getDate() - (day === 0 ? 6 : day - 1));
const target = new Date(monthDetailTarget + 'T00:00:00');
const tDow = target.getDay();
const targetMon = new Date(target);
targetMon.setDate(target.getDate() - (tDow === 0 ? 6 : tDow - 1));
weekOffset  = Math.round((targetMon - thisMon) / (1000*60*60*24*7));
ttActiveDay = DAY_KEYS.indexOf(dayKey);
```
- 月間カレンダーの詳細モーダルにある「この日の時間割を見る」ボタンの処理です。今週の月曜日（`thisMon`）と、タップした日が属する週の月曜日（`targetMon`）を、それぞれ4.節と同じ考え方で計算し、その差を7日単位で割ることで「今週から何週間ずれているか」（`weekOffset`）を逆算しています。`Date`オブジェクト同士の引き算はミリ秒単位の差になるため、`1000*60*60*24*7`（1週間のミリ秒数）で割ることで週数に変換しています。

---

## 7. 時間割編集の下ごしらえ（627〜778行）

### 7.1 FABの開閉（627〜640行）
`toggleTTFab`/`closeTTFab`は、右下の＋ボタンの開閉です。[../01_index_予定管理.md](../01_index_予定管理.md)の`toggleFab`/`closeFab`と同じ考え方ですが、対象の要素IDが`tt-`接頭辞付きになっている点だけが違います（時間割ページのFABは「時間割編集」「学期設定」の項目も追加で持つため、専用のID体系になっています）。

### 7.2 上書きデータの読み込み：`loadTTOverrides()`（645〜676行）
```js
let localOverrides = {};
try {
  const raw = localStorage.getItem('tt_overrides_' + GUILD_ID);
  localOverrides = raw ? JSON.parse(raw) : {};
} catch(_) {}
try {
  const res  = await fetch(`${API_BASE}${TT_API.LIST}?guild_id=${GUILD_ID}`);
  const data = await res.json();
  if (data.ok && Array.isArray(data.overrides)) {
    const serverOverrides = {};
    data.overrides.forEach(o => { serverOverrides[o.key] = o; });
    ttOverrides = { ...localOverrides, ...serverOverrides };
    saveTTOverrideLocal();
    renderTimetable();
    return;
  }
} catch(e) {}
ttOverrides = localOverrides;
renderTimetable();
```
- コメントに重要な設計意図が書かれています：「『1コマ休み』などバックエンドがまだ対応していない種類の変更は`localStorage`にしか残らないため、消さずに後でマージする」。つまり、サーバー側の対応が追いついていない一部の機能（実装時点でまだ`set_period_holiday`が本格運用されていなかった時期の名残と考えられます）は、この端末のローカル保存だけが頼りになるため、サーバーから取得した最新データで**単純に上書きするのではなく**、`{ ...localOverrides, ...serverOverrides }`という順序で合体させることで、「サーバー側にまだ無いローカルだけの変更」を消さずに残しています（オブジェクトのスプレッド構文では、後に書いた方が同じキーを上書きするので、サーバー側のデータで同じキーがあればそちらが優先され、無ければローカルの値がそのまま残ります）。

### 7.3 コマの詳細モーダル：`showTTDetail(dateStr, period)`（681〜732行）
1コマをタップしたときの詳細表示です。5.3節の週表示の描画ロジックと同じ「休みか→変更ありか→通常か」の判定を、今度は詳細モーダル用の行（日付・時限・科目・持ち物・課題・備考）として組み立てます。同じ判定ロジックが週表示の描画と詳細モーダルの2箇所に別々に書かれている点は、コードの重複が生まれている箇所と言えます。

### 7.4 詳細から編集へ：`editFromTTDetail()`（734〜778行）
```js
closeModal('tt-detail');
openTTEditModal();
calState['tt-edit'].selected = date;
...
renderCal('tt-edit');
const periodEl = document.getElementById('tt-edit-period');
if (periodEl) periodEl.value = String(period);
const holidayOv = ttOverrides[`period_holiday:${date}:${period}`];
const changeOv  = ttOverrides[`change:${date}:${period}`];
if (holidayOv) {
  switchTTMode('period-holiday');
  document.getElementById('tt-edit-period-holiday-reason').value = holidayOv.reason || '休み';
  ...
} else {
  switchTTMode('change');
  ...
}
```
- 詳細モーダルの「この時間割を変更する」ボタンから、時間割編集モーダルを開き、タップしていたコマの日付・時限・（既存の変更があれば）その内容をあらかじめ入力欄に反映しておく、「詳細→編集へのつなぎ役」です。[../01_index_予定管理.md](../01_index_予定管理.md)の`editFromDetail()`と同じ役割のパターンです。

---

続きは[02_Timetable.js_その2_時間割編集モーダル.md](02_Timetable.js_その2_時間割編集モーダル.md)で、時間割編集モーダル（授業変更・1コマ休み・曜日変更・休校設定）を解説します。
