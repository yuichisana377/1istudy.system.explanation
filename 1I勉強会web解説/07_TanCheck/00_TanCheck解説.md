# 単位チェッカーページ — 全体像と `TanCheck.js` 解説

対象：`bot.1istudy.web/TanCheck.html`（145行）、`bot.1istudy.web/TanCheck.js`（564行）。用語は[../01_index_予定管理.md](../01_index_予定管理.md)の「0. ミニ用語辞典」も参照してください。

## このページの特徴

[../00_HTML構造とページ全体像.md](../01_index_予定管理.md)でも紹介した通り、このページは他の機能と毛色が違います。**サーバーには一切データを送らず、入力した点数はこの端末の`localStorage`だけに保存されます**。ログイン自体は必要ですが（情報工学科1年生だけに公開するための出入り口として）、点数データそのものはサーバー側のどこにも記録されません。

`TanCheck.html`の構造は他ページとほぼ共通（ドロワー・ヘッダー・保険オーバーレイ）ですが、メインコンテンツは「前期・専門科目」「前期・一般科目」「後期・専門科目」「後期・一般科目」という4つの科目リストの入れ物（`#tc-list-*`）だけが空で用意されており、中身はすべて`TanCheck.js`がJSで組み立てます。JS読み込みは`Icons.js → SwipeGuard.js → Dialog.js → TanCheck.js → PendingDeleteCheck.js`で、`Dropdown.js`は`<select>`を使っていないため読み込まれません。

---

## 1. 評価割合のデータ（1〜84行）

冒頭のコメントに、このページの正確さに関する重要な前提が詳しく書かれています：

- 評価割合（`SUBJECTS`定数）は、高専機構の公開シラバス検索システムから、**前期・後期それぞれ実際にシラバスを確認して**転記したもの。同じ科目でも前期と後期で評価割合が異なるケース（例：公共は前期「定期65/課題5/小テスト30」、後期「定期60/課題20/小テスト20」）も、実際の違いをそのまま反映しています。
- A/B/Cの点数基準（85点以上=A、70点以上=B、60点以上=C）はシラバスに記載が無いため、シラバスとは別に「1I勉強会から教えてもらった基準」を使っています。
- 「シラバス自体の変更・担当教員による運用の違いは反映されない」ため、あくまで参考値であり、正式な成績は学校の発表で確認する必要がある、という免責が明記されています。

```js
// ── A/B/C 判定基準（1I勉強会から確認済み） ──────────────
const GRADE_THRESHOLDS = [
  { grade: 'A', min: 85 },
  { grade: 'B', min: 70 },
  { grade: 'C', min: 60 },
];

// ── 「小テスト」「課題」系は複数回・合算方式にする ───────
function isMultiLabel(label) {
  return label.includes('課題') || label.includes('小テスト');
}

// ── 科目データ（情報工学科 1年・2026年度シラバスより、前期・後期とも確認済み） ──
// items: [評価項目名, 割合(%)] の配列。合計は必ず100になるようにしてある。
const SUBJECTS = {
  zenkiSpecialized: [
    { name: 'コンピュータリテラシ', code: '31111', items: [['課題', 100]] },
    { name: '情報技術概論',         code: '31112', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '情報工学ゼミⅠ',        code: '31113', items: [['課題', 100]] },
    { name: '情報基礎',             code: '31114', items: [['定期試験', 40], ['課題', 60]] },
  ],
  zenkiGeneral: [
    { name: '国語Ⅰ',                 code: '01121', items: [['中間試験', 30], ['定期試験', 45], ['課題', 15], ['小テスト', 10]] },
    { name: '地理',                   code: '01124', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '基礎解析Ⅰ',              code: '01125', items: [['定期試験', 40], ['課題', 20], ['小テスト', 40]] },
    { name: '線形数学Ⅰ',              code: '01126', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '物理Ⅰ',                  code: '01127', items: [['定期試験', 50], ['課題', 20], ['小テスト', 30]] },
    { name: '化学Ⅰ',                  code: '01128', items: [['定期試験', 50], ['課題', 20], ['小テスト', 30]] },
    { name: '英語会話',                code: '01131', items: [['会話演習', 50], ['多読', 50]] },
    { name: '保健体育Ⅰ',              code: '01134', items: [['スポーツテスト', 10], ['水泳', 15], ['実技課題', 55], ['保健', 20]] },
    { name: '英語コミュニケーションⅠ', code: '01137', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '英語表現基礎',            code: '01138', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '公共',                    code: '01139', items: [['定期試験', 65], ['課題', 5], ['小テスト', 30]] },
  ],
  kokiSpecialized: [
    { name: 'プログラミングⅠ',      code: '31211', items: [['定期試験', 50], ['小テスト', 20], ['プログラミング演習課題', 30]] },
    { name: '数理工学演習Ⅰ',        code: '31213', items: [['定期試験', 40], ['課題', 10], ['小テスト', 50]] },
  ],
  kokiGeneral: [
    { name: '国語Ⅰ',                 code: '01221', items: [['中間試験', 30], ['定期試験', 45], ['課題', 15], ['小テスト', 10]] },
    { name: '地理',                   code: '01224', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '基礎解析Ⅰ',              code: '01225', items: [['定期試験', 40], ['課題', 20], ['小テスト', 40]] },
    { name: '線形数学Ⅰ',              code: '01226', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '物理Ⅰ',                  code: '01227', items: [['定期試験', 50], ['課題', 20], ['小テスト', 30]] },
    { name: '化学Ⅰ',                  code: '01228', items: [['定期試験', 50], ['課題', 20], ['小テスト', 30]] },
    { name: '英語会話',                code: '01231', items: [['会話演習', 50], ['多読', 50]] },
    { name: '保健体育Ⅰ',              code: '01233', items: [['持久走', 15], ['実技課題', 65], ['保健', 20]] },
    { name: '英語コミュニケーションⅠ', code: '01236', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '英語表現基礎',            code: '01237', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
    { name: '公共',                    code: '01238', items: [['定期試験', 60], ['課題', 20], ['小テスト', 20]] },
    { name: '総合理科',                code: '01234', items: [['中間試験', 30], ['定期試験', 50], ['課題', 20]] },
  ],
};
```
- 各科目は`{ name, code, items }`という形で、`items`は`[評価項目名, 割合(%)]`のペアの配列です。コメントに「合計は必ず100になるようにしてある」とあります。
- `isMultiLabel(label)`：項目名に「課題」または「小テスト」という文字列が含まれていれば、複数回・合算方式（3節）として扱います。単純な文字列の部分一致で判定しているため、例えば「小テスト」を含む項目名なら自動的にこの方式になります。

---

## 2. 保存・判定・計算ロジック（86〜134行）

```js
function loadScores() { try { return JSON.parse(localStorage.getItem(TC_STORAGE_KEY)) || {}; } catch { return {}; } }
function saveScores(all) { try { localStorage.setItem(TC_STORAGE_KEY, JSON.stringify(all)); } catch {} }
```
- 全科目・全項目の入力値を、1つの`localStorage`キー（`tancheck_scores_v2`）にまとめて保存します。データ構造は「科目コード→（単一項目なら点数、複数回項目なら`{s:得点, m:満点}`の配列）」という2段階のオブジェクトです。

```js
function gradeOf(score) {
  for (const t of GRADE_THRESHOLDS) if (score >= t.min) return t.grade;
  return 'F';
}
```
- `GRADE_THRESHOLDS`を上から順に見て、最初に条件を満たした等級を返します。配列が`A`→`B`→`C`の順（高い基準から低い基準へ）に並んでいるからこそ、この単純な「最初に一致したもの」という判定で正しく動きます。どれにも該当しなければ`'F'`（不可）です。

### 2.1 達成度の計算：`computeResult(items, entered)`（104〜134行）
```js
let enteredWeight = 0, enteredSum = 0, remainingWeight = 0;
items.forEach(([label, weight]) => {
  const v = entered[label];
  if (Array.isArray(v)) {
    // 複数回（課題・小テスト）: 得点合計 ÷ 満点合計
    let sSum = 0, mSum = 0;
    v.forEach(row => {
      const s = row.s;
      const m = (row.m === undefined || row.m === null || row.m === '') ? 100 : Number(row.m);
      if (s !== undefined && s !== null && s !== '' && !isNaN(s) && m > 0) { sSum += Number(s); mSum += m; }
    });
    if (mSum > 0) { enteredWeight += weight; enteredSum += weight * (sSum / mSum); }
    else { remainingWeight += weight; }
  } else if (v === undefined || v === null || v === '' || isNaN(v)) {
    remainingWeight += weight;
  } else {
    enteredWeight += weight;
    enteredSum += weight * Number(v) / 100;
  }
});
return { enteredWeight, enteredSum, remainingWeight };
```
- 各評価項目（試験・課題など）を1つずつ処理し、**入力済みの割合（`enteredWeight`）**、**入力済み分から計算できる得点（`enteredSum`）**、**まだ何も入力されていない割合（`remainingWeight`）**の3つを積み上げます。
- 複数回項目（課題・小テスト）は、`得点合計 ÷ 満点合計`で達成率を出し、その項目の割合（`weight`）に掛けて得点に変換します（`weight * (sSum / mSum)`）。満点（`m`）が空欄なら100点満点として扱う、という仕様がここに実装されています（コメントの「満点を空欄のままにした場合は100点満点として扱う」）。1件も有効な行が無ければ（`mSum > 0`でなければ）、その項目全体が「未入力」（`remainingWeight`に加算）扱いになります。
- 単一項目は、入力された点数（0〜100点の素点）をそのまま`weight * v / 100`で、その項目の配点に応じた得点に変換します。

---

## 3. 科目カードの描画（136〜372行）

### 3.1 保存値の書き換え：`persistEntered(code, mutate)`（139〜145行）
```js
function persistEntered(code, mutate) {
  const all = loadScores();
  const cur = all[code] || {};
  mutate(cur);
  all[code] = cur;
  saveScores(all);
}
```
- 「今の全データを読み込む→対象科目の部分だけ`mutate`関数で書き換えさせる→全体を保存し直す」という、入力があるたびに毎回呼ばれる共通の保存ヘルパーです。`mutate`という**関数を引数として渡す**ことで、呼び出し側は「保存」そのものの手続き（読み込み→保存）を意識せず、「この科目のデータをどう書き換えたいか」だけを書けばよくなっています。

### 3.2 単一項目の入力欄：`renderSingleItem(...)`（147〜183行）
```js
input.addEventListener('input', () => {
  persistEntered(subject.code, cur => {
    if (input.value === '') { delete cur[label]; }
    else {
      let v = Number(input.value);
      if (!isNaN(v)) cur[label] = Math.max(0, Math.min(100, v));
    }
  });
  onChange();
});
```
- 入力欄に何か打つたびに保存し、`onChange()`（このカードの判定結果を再計算する、3.4節）を呼びます。`Math.max(0, Math.min(100, v))`で、入力値を必ず0〜100点の範囲に収めます（範囲外の値が入力されても、保存される値はクランプ＝切り詰められます）。

### 3.3 複数回項目の入力欄：`renderMultiItem(...)`（185〜295行）
```js
function getRows() {
  const all = loadScores();
  const cur = all[subject.code] || {};
  if (!Array.isArray(cur[label])) cur[label] = [];
  return cur[label];
}

function renderRows() {
  rowsWrap.innerHTML = '';
  const rows = getRows();
  rows.forEach((row, idx) => {
    const r = document.createElement('div');
    r.className = 'tc-multi-row';

    const sInput = document.createElement('input');
    sInput.type = 'number';
    sInput.inputMode = 'decimal';
    sInput.className = 'tc-item-input tc-multi-score';
    sInput.placeholder = '得点';
    if (row.s !== undefined && row.s !== null) sInput.value = row.s;
    sInput.addEventListener('input', () => {
      persistEntered(subject.code, cur => {
        const arr = cur[label] || [];
        arr[idx] = arr[idx] || {};
        arr[idx].s = sInput.value === '' ? undefined : Number(sInput.value);
        cur[label] = arr;
      });
      onChange();
    });

    const slash = document.createElement('span');
    slash.className = 'tc-multi-slash';
    slash.textContent = '/';

    const mInput = document.createElement('input');
    mInput.type = 'number';
    mInput.inputMode = 'decimal';
    mInput.className = 'tc-item-input tc-multi-max';
    mInput.placeholder = '100';
    if (row.m !== undefined && row.m !== null) mInput.value = row.m;
    mInput.addEventListener('input', () => {
      persistEntered(subject.code, cur => {
        const arr = cur[label] || [];
        arr[idx] = arr[idx] || {};
        arr[idx].m = mInput.value === '' ? undefined : Number(mInput.value);
        cur[label] = arr;
      });
      onChange();
    });

    const rm = document.createElement('button');
    rm.type = 'button';
    rm.className = 'tc-multi-remove';
    rm.setAttribute('aria-label', 'この回を削除');
    rm.textContent = '×';
    rm.addEventListener('click', () => {
      persistEntered(subject.code, cur => {
        const arr = cur[label] || [];
        arr.splice(idx, 1);
        cur[label] = arr;
      });
      renderRows();
      onChange();
    });

    r.appendChild(sInput);
    r.appendChild(slash);
    r.appendChild(mInput);
    r.appendChild(document.createTextNode('点'));
    r.appendChild(rm);
    rowsWrap.appendChild(r);
  });
}
renderRows();

const addBtn = document.createElement('button');
addBtn.textContent = '＋ ' + label + 'を1回分追加';
addBtn.addEventListener('click', () => {
  persistEntered(subject.code, cur => { const arr = cur[label] || []; arr.push({}); cur[label] = arr; });
  renderRows();
  onChange();
});
```
- 「＋〇〇を1回分追加」ボタンで、何回分でも「得点／満点」の行を追加できます。各行には削除ボタン（`×`）も付いており、押すと`arr.splice(idx, 1)`でその行だけを配列から取り除いて再描画します。
- 得点・満点それぞれの入力欄には別々に`input`イベントリスナーが付いており、`arr[idx] = arr[idx] || {}`で「まだそのインデックスにオブジェクトが無ければ作る」という安全なアクセスをしてから、`.s`（得点）または`.m`（満点）だけを更新します。

### 3.4 科目カード本体：`renderSubjectCard(subject, allScores)`（297〜372行）
```js
function onChange() { updateCardResult(subject, badge, resultBox); }

subject.items.forEach(([label, weight]) => {
  if (isMultiLabel(label)) {
    renderMultiItem(body, subject, label, weight, entered, onChange);
  } else {
    renderSingleItem(body, subject, label, weight, entered, onChange);
  }
});
```
- 科目ごとに、評価項目を1つずつ、複数回方式か単一方式かで描画を出し分けます。すべての入力欄が同じ`onChange`関数（このカードのバッジと結果表示を更新する）を共有しているので、どの項目を編集しても即座に判定結果が更新されます。

```js
const resetBtn = document.createElement('button');
resetBtn.type = 'button';
resetBtn.className = 'tc-card-reset';
resetBtn.textContent = 'この科目の入力をリセット';
resetBtn.addEventListener('click', (e) => {
  e.stopPropagation();
  const all = loadScores();
  delete all[subject.code];
  saveScores(all);
  body.innerHTML = '';
  subject.items.forEach(([label, weight]) => {
    if (isMultiLabel(label)) {
      renderMultiItem(body, subject, label, weight, {}, onChange);
    } else {
      renderSingleItem(body, subject, label, weight, {}, onChange);
    }
  });
  body.appendChild(resultBox);
  body.appendChild(resetBtn);
  onChange();
});
body.appendChild(resetBtn);
```
- 「この科目の入力をリセット」ボタンは、保存データから該当科目を削除し、入力欄自体もすべて空の状態で作り直します。`e.stopPropagation()`で、このボタンのクリックがカード全体の開閉（`head`のクリックリスナー）に伝わらないようにしています。

```js
head.addEventListener('click', () => { card.classList.toggle('is-open'); });
```
- カードのヘッダーをタップすると、詳細（入力欄一式）が開閉します。

### 3.5 判定結果の表示：`updateCardResult(subject, badgeEl, resultBox)`（374〜434行）

3段階の表示パターンがあります：

1. **何も入力されていない**（`enteredWeight === 0`）：バッジは`−`、「点数を入力すると、この場で判定を確認できます」とだけ表示。
2. **全項目入力済み**（`remainingWeight === 0`）：確定した得点と等級を表示。
3. **一部だけ入力済み**：ここが最も情報量の多い表示です。
   ```js
   const provisionalGrade = gradeOf(enteredSum);
   badgeEl.textContent = gradeLabel(provisionalGrade) + '?';
   ```
   - 「残り項目を0点とした場合」の暫定評価を、`?`付きのバッジ（例：`B?`）で示します。
   ```js
   const maxScore = enteredSum + remainingWeight;
   maxLine.textContent = '残り項目で満点なら：最大 ' + fmt(maxScore) + '点';
   GRADE_THRESHOLDS.forEach(t => {
     if (enteredSum >= t.min) {
       line.textContent = `${t.grade}（${t.min}点以上）：残りが何点でも達成見込みです`;
     } else {
       const needAvg = (t.min - enteredSum) * 100 / remainingWeight;
       if (needAvg > 100) {
         line.textContent = `${t.grade}（${t.min}点以上）：残り全項目が満点でも届きません`;
       } else {
         line.textContent = `${t.grade}（${t.min}点以上）まで：残りの項目で平均 ${fmt(needAvg)}点 必要`;
       }
     }
   });
   ```
   - A・B・Cそれぞれの基準について、「残りの項目で平均何点取れば届くか」を逆算します。`(目標点 - 今の得点) ÷ 残り割合 × 100`という計算で、「残りの配点をすべて100点満点だとみなしたときの、必要な平均点」を求めています。
   - この必要平均点が既に100点を超えていれば「残り全項目が満点でも届きません」、逆に今の得点だけで既に基準を超えていれば「残りが何点でも達成見込みです」と、3通りの文言を出し分けます。これが、このページのタイトルにもある「Aまであとどれくらい必要か」を実際に計算している中心部分です。

---

## 4. 起動とドロワー（436〜564行）

```js
function renderGroup(listEl, subjects) {
  const allScores = loadScores();
  subjects.forEach(subject => { listEl.appendChild(renderSubjectCard(subject, allScores)); });
}
function renderAllSubjects() {
  renderGroup(document.getElementById('tc-list-zenki-specialized'), SUBJECTS.zenkiSpecialized);
  renderGroup(document.getElementById('tc-list-zenki-general'), SUBJECTS.zenkiGeneral);
  renderGroup(document.getElementById('tc-list-koki-specialized'), SUBJECTS.kokiSpecialized);
  renderGroup(document.getElementById('tc-list-koki-general'), SUBJECTS.kokiGeneral);
}
renderAllSubjects();
```
- ページ読み込み時に、4つの科目グループをそれぞれの入れ物に描画します。他ページのように`window.addEventListener('load', ...)`で包まず、スクリプトの実行順にそのまま呼び出しているのは、このページがサーバーとの通信を一切必要とせず、`localStorage`から即座に読み込めるため、`load`イベントを待つ理由が無いからだと考えられます。

`getLoginSession`/`renderDrawerAccount`/`openDrawer`/`closeDrawer`とドロワーリンクのクリック処理・`pageshow`イベント処理（451〜561行）は、他ページと同じ構造です。コメントに「このページ自体はログイン不要（入力は`localStorage`のみで完結する）」とありますが、実際には`TanCheck.html`へのアクセス自体をログイン中のユーザーだけに絞る仕組みが別途（サーバー側かBotの案内経路で）働いていると考えられ、このJSファイル自体には強制ログインのチェック（`(function(){ ... location.replace(...) })()`のようなもの）が見当たりません。ドロワーのアカウント表示は他ページと同じ体裁を保つために用意されています。

---

## まとめ

`TanCheck.js`は、このシリーズで見てきた他のページと違い、**サーバー通信が一切登場しない**という点で特殊なページでした。`localStorage`だけで完結するシンプルさを優先した設計であることが、冒頭のコメントに明記されている通りです。一方で、判定ロジック自体（複数回項目の合算、残り項目からの逆算）は数値計算として作り込まれており、単純な点数入力フォーム以上の実用性を持たせている点が特徴的でした。
