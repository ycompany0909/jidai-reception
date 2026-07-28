# JIDAI イベント受付システム — 標準仕様書 v1.0

> 本ドキュメントは JIDAI クロージング受付システムの完全な実装・運用ガイドです。
> 他のイベント受付ページをこの仕様に基づいて実装してください。

---

## 目次

1. [概要](#概要)
2. [要件](#要件)
3. [アーキテクチャ](#アーキテクチャ)
4. [データモデル](#データモデル)
5. [機能仕様](#機能仕様)
6. [コーディング実装ガイド](#コーディング実装ガイド)
7. [パフォーマンス・セキュリティ](#パフォーマンスセキュリティ)
8. [トラブルシューティング](#トラブルシューティング)
9. [チェックリスト](#チェックリスト)

---

## 概要

### プロダクト概要

**JIDAI イベント受付システム** は、イベント当日の参加者を効率的に管理するウェブアプリケーションです。

| 項目 | 内容 |
|------|------|
| **対象** | JIDAI（銀座）・IAPONIA（新橋）等のイベント受付 |
| **デバイス** | スマートフォン・タブレット・PC |
| **機能** | 来場打刻・欠席管理・支払い管理・ドタ参追加・CSV 出力 |
| **保存** | localStorage（デバイスローカル） |
| **ライセンス** | Vanilla JS（フレームワーク不使用） |

### コアバリュー

1. **シンプル**: フレームワークに依存しない Vanilla HTML/CSS/JS
2. **高速**: 動的ページ遷移なし、即座に UI 更新
3. **オフライン**: インターネット接続不要（ローカルストレージ）
4. **スマホ最適**: タッチ操作を想定した UI/UX

---

## 要件

### 機能要件

| 優先度 | 機能 | 説明 |
|------|------|------|
| **MUST** | 参加者表示 | あいうえお順で全参加者リスト表示 |
| **MUST** | 来場打刻 | ワンクリックで来場をマーク・時刻自動記録 |
| **MUST** | 欠席管理 | ワンクリックで欠席をマーク |
| **MUST** | 支払い管理 | 現金・カード・免除をトグル選択 |
| **MUST** | Jump 機能 | あいうえおボタンで該当セクションへスクロール |
| **SHOULD** | ドタ参追加 | 当日参加者を追加・即座に打刻 |
| **SHOULD** | CSV 出力 | 全データを CSV で ダウンロード |
| **SHOULD** | リセット | localStorage を初期化 |
| **NICE** | 検索機能 | 名前で参加者をフィルタ |

### 非機能要件

| 項目 | 要件 |
|------|------|
| **レスポンス時間** | ボタンクリック → UI 更新: <100ms |
| **ローカルストレージ** | 最大 5MB（参加者 100-200 名分で十分） |
| **ブラウザ対応** | Safari 13+, Chrome 88+, Firefox 87+ (IE11 不対応) |
| **オフライン対応** | インターネット接続不要 |
| **マルチタブ同期** | 同一 KEY で複数タブ開く場合、state が競合しない設計 |

---

## アーキテクチャ

### 全体構成図

```
┌─────────────────────────────────────────┐
│  HTML (静的マークアップ)                 │
│  - Header (イベント情報・統計表示)      │
│  - Toolbar (検索・ボタン)                │
│  - Jumpbar (あいうえおボタン)           │
│  - List (参加者リスト・セクション分離) │
│  - Add Panel (ドタ参追加フォーム)       │
└─────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│  JavaScript (イベント・状態管理)         │
│  - buildJumpbar()  (あいうえおボタン)   │
│  - render()        (UI 更新)             │
│  - setArr()        (来場/欠席マーク)    │
│  - setPay()        (支払い方法設定)     │
│  - addParticipant() (ドタ参追加)         │
│  - save()          (localStorage 保存)  │
└─────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│  localStorage (永続化)                   │
│  KEY: 'jidai-07-28-v3'                  │
│  VALUE: JSON 状態オブジェクト            │
└─────────────────────────────────────────┘
```

### データフロー

```
ユーザー操作
  ↓
イベントリスナー (onclick, addEventListener)
  ↓
状態更新関数 (setArr, setPay, addParticipant)
  ↓
save() ← localStorage に保存
  ↓
render() ← UI 再描画
  ↓
画面表示
```

---

## データモデル

### 1. DATA 配列（参加者マスタ）

参加者の**静的情報**を定義。ページロード時に固定。

```javascript
const DATA = [
  {
    "no": 1,
    "name": "相見高志",
    "kana": "あいみたかし"
  },
  {
    "no": 2,
    "name": "麻生雅人",
    "kana": "あそうまさと"
  },
  // ... 最大 200-300 名
];
```

**フィールド説明**:

| フィールド | 型 | 説明 | 例 |
|-----------|-----|------|-----|
| `no` | number | 参加者 ID（1 から順番）| 1 |
| `name` | string | 漢字氏名 | "相見高志" |
| `kana` | string | ひらがなカナ（あいうえお順ソート用）| "あいみたかし" |

**重要**:
- `kana` は必ず **ひらがな全角** を使用（カタカナ・半角 NG）
- `no` は **ユニーク**・昇順・連番が推奨（ドタ参追加時の nextNo 計算に使用）
- CSV から import する場合は、上記形式で正規化してから挿入

### 2. state オブジェクト（参加者の動的状態）

参加者の**動的な状態**（来場・支払いなど）を管理。localStorage に保存。

```javascript
// グローバル state オブジェクト
let state = {
  1: {
    arr: true,              // 来場フラグ: null | true | false
    pay: 'c',               // 支払い方法: null | 'c' | 'k' | 'e'
    ts: "15:30:45",         // 来場時刻: "HH:MM:SS" | null
    noshow: false           // 欠席フラグ: boolean
  },
  2: {
    arr: false,
    pay: null,
    ts: null,
    noshow: true            // 欠席が true の場合、arr は false
  },
  3: {
    arr: null,              // 未処理（来場も欠席もしていない）
    pay: null,
    ts: null,
    noshow: false
  }
};
```

**フィールド説明**:

| フィールド | 型 | 値 | 説明 |
|-----------|-----|------|------|
| `arr` | `null \| boolean` | `null` | 来場: true / 欠席: false / 未処理: null |
| `pay` | `null \| string` | `'c' \| 'k' \| 'e' \| null` | 現金=c / カード=k / 免除=e |
| `ts` | `string \| null` | `"HH:MM:SS"` | JST タイムスタンプ |
| `noshow` | `boolean` | `true \| false` | 欠席フラグ（arr=false 時は true） |

**状態遷移図**:

```
初期状態
  arr: null, noshow: false
    ↓ [✓来場クリック]
  arr: true, noshow: false ← 来場打刻
    ↓ [↩クリック]
  arr: null, noshow: false ← 来場取り消し
    ↓ [✗欠席クリック]
  arr: false, noshow: true ← 欠席登録
```

### 3. localStorage 永続化

```javascript
const KEY = 'jidai-07-28-v3';  // イベント・日付・バージョン一意識別子
const serialized = JSON.stringify(state);
localStorage.setItem(KEY, serialized);

// 読み込み時
let state = JSON.parse(localStorage.getItem(KEY) || '{}');
```

**重要な注意点**:

1. **KEY の一意性**: イベントごと・日付ごと・キャッシュクリア時にバージョンをインクリメント
   ```javascript
   // ❌ NG: 複数イベント・再テスト時にデータが混在
   const KEY = 'reception';
   
   // ✅ OK: バージョンで分離
   const KEY = 'jidai-07-28-v3';  // v2 → v3 で新規 state
   ```

2. **初期化タイミング**: ページロード時に全参加者の state を初期化
   ```javascript
   DATA.forEach(p => {
     if (!state[p.no]) state[p.no] = {arr:null, pay:null, ts:null, noshow:false};
   });
   ```

3. **サイズ制限**: localStorage は最大 5-10MB。参加者 100-300 名で余裕
   ```javascript
   // 概算
   1人分の state ≈ 50 bytes
   300人 × 50 = 15,000 bytes = 15KB ✅ OK
   ```

---

## 機能仕様

### 1. 参加者リスト表示

#### 要件

- 全参加者を **あいうえお順** で表示
- **未処理** と **処理済み** に 2 セクション分離
- 来場数・欠席数・現金額をリアルタイム表示
- スマホで見やすい UI

#### 実装コード

```javascript
function render() {
  const list = document.getElementById('list');
  const stats = {arr:0, abs:0, cash:0};
  
  // フィルタ：検索入力があれば該当者のみ
  const filtered = filteredRows();
  
  // セクション分離
  const pending = filtered.filter(p => !isProcessed(p));   // 未処理
  const processed = filtered.filter(p => isProcessed(p));  // 処理済み
  
  // 統計計算（全 DATA に対して）
  DATA.forEach(p => {
    const s = state[p.no] || {};
    if (s.arr === true) stats.arr++;
    if (s.noshow || (s.arr === false && !s.noshow)) stats.abs++;
    if (s.pay === 'c') stats.cash++;
  });
  
  // HTML 生成
  let html = '';
  
  // セクション 1: 未処理
  if (pending.length > 0) {
    html += pending.map(renderRow).join('');
  } else if (filtered.length > 0) {
    html += '<div style="...">未処理なし</div>';
  }
  
  // セクション 2: 処理済み
  if (processed.length > 0) {
    html += `<div style="...">処理済み (${processed.length}名)</div>`;
    html += processed.map(renderRow).join('');
  }
  
  // DOM 更新
  list.innerHTML = html;
  
  // 統計表示
  document.getElementById('stat-arr').textContent = stats.arr;
  document.getElementById('stat-abs').textContent = stats.abs;
  document.getElementById('stat-tot').textContent = DATA.length;
  document.getElementById('stat-cash').textContent = (stats.cash * 10000).toLocaleString();
}

function renderRow(p) {
  const s = state[p.no] || {};
  const isArr = s.arr === true;
  const isNoShow = s.noshow;
  const rowClass = isProcessed(p) ? 'arrived' : '';
  
  // タイムスタンプ表示
  const timeHtml = (isArr && s.ts) ? `<div class="timestamp">${s.ts}</div>` : '';
  
  // undo ボタン（来場時のみ表示）
  const undoBtn = isArr ? '<button class="mini undo" onclick="clearArr(' + p.no + ')">↩</button>' : '';
  
  return `
    <div class="row ${rowClass}" data-no="${p.no}">
      <div class="no">${p.no}</div>
      <div>
        <div class="name">${p.name}</div>
        <div class="detail">${p.kana}</div>
      </div>
      ${timeHtml}
      <div class="actions">
        <div class="line">
          <button class="mini ${isArr ? 'active' : ''} arrive" onclick="setArr(${p.no}, true)">✓</button>
          <button class="mini ${isNoShow ? 'active' : ''} absent" onclick="setNoShow(${p.no})">✗</button>
          ${undoBtn}
        </div>
        <div class="line">
          <button class="mini ${s.pay==='c'?'active':''} cash" onclick="setPay(${p.no}, 'c')">¥</button>
          <button class="mini ${s.pay==='k'?'active':''} card" onclick="setPay(${p.no}, 'k')">カ</button>
          <button class="mini ${s.pay==='e'?'active':''} exempt" onclick="setPay(${p.no}, 'e')">免</button>
        </div>
      </div>
    </div>
  `;
}

function isProcessed(p) {
  const s = state[p.no] || {};
  return s.arr !== null || s.noshow;
}
```

#### パフォーマンス最適化

- **render() の頻度**: `save()` 内からのみ呼出。不要な二重呼び出しは避ける
- **DOM 操作**: `innerHTML` を 1 回だけ設定（複数回の操作は避ける）
- **配列フィルタ**: `filter()` × 2 回は許容（データ量 < 300 なら問題なし）

---

### 2. あいうえお Jump 機能

#### 要件

- あ・か・さ・た・な・は・ま・や・ら・わ のボタンを表示
- 各ボタンをクリックで該当セクションへスクロール
- 該当者がいないグループは disabled 表示

#### コーディング上の重要ポイント

**❌ よくある間違い: `outerHTML` で HTML 文字列に変換**

```javascript
// ❌ 間違い：onclick ハンドラが失われる
const jumpbar = document.getElementById('jumpbar');
jumpbar.innerHTML = KANA_ORDER.map(k => {
  const btn = document.createElement('button');
  btn.textContent = k;
  btn.onclick = () => jumpToKana(k);  // ← ここで onclick を設定
  return btn.outerHTML;  // ← でも outerHTML で HTML 文字列に変換するから handlers が失われる
}).join('');
// result: <button>あ</button> だけが HTML になり、onclick は削除される
```

**✅ 正解: `appendChild()` で直接 DOM に追加**

```javascript
// ✅ 正解：DOM に直接追加するので onclick ハンドラが保持される
function buildJumpbar() {
  const groups = new Set();
  DATA.forEach(p => {
    const kana = getFirstKana(p.kana);
    const group = getKanaGroup(kana);
    if (group) groups.add(group);
  });

  const jumpbar = document.getElementById('jumpbar');
  jumpbar.innerHTML = '';  // 既存の子要素を削除
  
  KANA_ORDER.forEach(k => {
    const btn = document.createElement('button');
    btn.className = 'jumpbtn';
    btn.textContent = k;
    btn.onclick = () => jumpToKana(k);  // onclick を設定
    if (!groups.has(k)) btn.disabled = true;
    jumpbar.appendChild(btn);  // ← DOM に追加（ハンドラが保持される）
  });
}

function jumpToKana(group) {
  const range = KANA_RANGES[group];
  const target = DATA.find(p => {
    const kana = getFirstKana(p.kana);
    return range.includes(kana);
  });
  
  if (target) {
    const elem = document.querySelector(`[data-no="${target.no}"]`);
    if (elem) {
      elem.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  }
}
```

#### あいうえお順の実装

```javascript
const KANA_ORDER = ['あ','か','さ','た','な','は','ま','や','ら','わ'];

// あいうえおの範囲定義（各行の子音を含む）
const KANA_RANGES = {
  'あ': ['あ','い','う','え','お'],
  'か': ['か','き','く','け','こ','が','ぎ','ぐ','げ','ご'],
  'さ': ['さ','し','す','せ','そ','ざ','じ','ず','ぜ','ぞ'],
  'た': ['た','ち','つ','て','と','だ','ぢ','づ','で','ど'],
  'な': ['な','に','ぬ','ね','の'],
  'は': ['は','ひ','ふ','へ','ほ','ば','び','ぶ','べ','ぼ','ぱ','ぴ','ぷ','ぺ','ぽ'],
  'ま': ['ま','み','む','め','も'],
  'や': ['や','ゆ','よ'],
  'ら': ['ら','り','る','れ','ろ'],
  'わ': ['わ','を','ん']
};

function getFirstKana(name) {
  if (!name || name.length === 0) return null;
  return name.charAt(0);  // 最初の文字（ひらがな）を返す
}

function getKanaGroup(kana) {
  for (let group of KANA_ORDER) {
    if (KANA_RANGES[group].includes(kana)) return group;
  }
  return null;
}

// 使用例：「ししどこうじ」→ 最初の「し」→ 「さ」グループ
console.log(getKanaGroup(getFirstKana('ししどこうじ')));  // → 'さ'
```

---

### 3. 来場・欠席・支払い管理

#### 来場打刻

```javascript
function setArr(no, val) {
  const s = setState(no);
  s.arr = val;  // true or false
  
  // 来場の場合、JST タイムスタンプを自動記録
  if (val === true) {
    s.ts = getJSTTime();
    s.noshow = false;  // 来場と欠席は同時に true にならない
  }
  
  save();  // ← 必ず save() を呼ぶ
}

function clearArr(no) {
  const s = setState(no);
  s.arr = null;  // 来場状態を初期化
  s.ts = null;   // タイムスタンプもクリア
  save();
}

// JST タイムスタンプ生成（必須）
function getJSTTime() {
  const now = new Date(Date.now() + 9*3600*1000);  // ← JST offset 追加
  const h = pad(now.getUTCHours());
  const m = pad(now.getUTCMinutes());
  const s = pad(now.getUTCSeconds());
  return `${h}:${m}:${s}`;
}

function pad(n) {
  return n < 10 ? '0' + n : n;
}
```

**重要: JST タイムスタンプの理由**

```javascript
// ❌ NG: UTC で保存すると深夜 0-9 時に日付ずれ
const now = new Date();
now.toISOString();  // "2026-07-28T14:30:00.000Z" (UTC)
// → 日本時間は 23:30 だが、ISO では 2026-07-28 に見える

// ✅ OK: JST offset を加えて、日本時間の日付で保存
const now = new Date(Date.now() + 9*3600*1000);
now.getUTCHours();  // 23 (JST で 23:30)
```

#### 欠席管理

```javascript
function setNoShow(no) {
  const s = setState(no);
  s.noshow = true;
  s.arr = false;  // 欠席をマークすると来場は false（null ではない）
  save();
}

// 状態判定：来場 or 欠席 → 処理済み
function isProcessed(p) {
  const s = state[p.no] || {};
  return s.arr !== null || s.noshow;
}
```

#### 支払い方法

```javascript
function setPay(no, val) {
  const s = setState(no);
  s.pay = val;  // 'c' | 'k' | 'e' | null
  save();
}

// 支払い方法と現金額計算
const PARTICIPANT_FEE = 10000;  // ¥10,000 固定

function calculateCashAmount() {
  let cashCount = 0;
  DATA.forEach(p => {
    const s = state[p.no] || {};
    if (s.pay === 'c') cashCount++;
  });
  return cashCount * PARTICIPANT_FEE;  // ¥100,000 など
}
```

---

### 4. ドタ参追加機能

#### 要件

- 「➕ ドタ参追加」ボタンでパネル表示
- 氏名・かな・支払い方法を入力（必須）
- 追加と同時に来場打刻
- リスト末尾に追加（no は自動採番）

#### 実装コード

```javascript
function toggleAddPanel() {
  const panel = document.getElementById('add-panel');
  const btn = event.target;  // クリックされたボタン
  
  if (panel.hidden) {
    panel.hidden = false;
    btn.textContent = '➕ 閉じる';
    document.getElementById('add-name').focus();  // フォーカス移動
  } else {
    panel.hidden = true;
    btn.textContent = '➕ ドタ参追加';
  }
}

function addParticipant() {
  const name = document.getElementById('add-name').value.trim();
  const kana = document.getElementById('add-kana').value.trim();
  const payment = document.getElementById('add-payment').value;
  
  // バリデーション
  if (!name) {
    alert('氏名を入力してください');
    return;
  }
  if (!kana) {
    alert('かなを入力してください');
    return;
  }
  if (!payment) {
    alert('支払方法を選択してください');
    return;
  }
  
  // no の自動採番（最大値 + 1）
  const nextNo = Math.max(...DATA.map(p => p.no), 0) + 1;
  
  // DATA に追加
  DATA.push({no: nextNo, name, kana});
  
  // 状態初期化 + 支払方法設定
  const s = setState(nextNo);
  s.pay = payment;
  
  // 即座に来場打刻
  setArr(nextNo, true);
  
  // フォーム reset（save() → render() で UI も更新される）
  document.getElementById('add-name').value = '';
  document.getElementById('add-kana').value = '';
  document.getElementById('add-payment').value = '';
  
  toggleAddPanel();  // パネルを閉じる
}
```

**重要な実装順序**:

```javascript
// ✅ 正解：setState() → s.pay 設定 → setArr() の順序
const s = setState(nextNo);
s.pay = payment;
setArr(nextNo, true);

// ❌ NG：setArr() を先に呼ぶと setState() で初期化される可能性
setArr(nextNo, true);
const s = state[nextNo];  // 既に s.pay が null に初期化されている
s.pay = payment;  // ← この時点では遅い
```

---

### 5. CSV 出力

#### 要件

- 全参加者データを CSV フォーマットで出力
- ファイル名: `[イベント名]_[日付].csv`
- 来場・欠席・時刻・支払い方法を含む

#### 実装コード

```javascript
function downloadCSV() {
  let csv = '番号,氏名,来場,時刻,支払\n';
  
  DATA.forEach(p => {
    const s = state[p.no] || {};
    
    // 来場状態を記号で表現
    const arr = s.arr === true ? '◯' : (s.arr === false ? '✗' : '');
    
    // タイムスタンプ（来場時のみ）
    const ts = s.ts || '';
    
    // 支払い方法をテキスト化
    const paymentMap = {c: '現金', k: 'カード', e: '免除'};
    const pay = paymentMap[s.pay] || '';
    
    // CSV 行を追加（氏名にカンマが含まれる場合は " で囲む）
    csv += `${p.no},"${p.name}","${arr}","${ts}","${pay}"\n`;
  });
  
  // Blob 化して ダウンロード
  const blob = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'JIDAI_07-28.csv';  // ← ファイル名をイベント・日付で設定
  link.click();
}
```

**CSV フォーマット例**:

```csv
番号,氏名,来場,時刻,支払
1,"相見高志","◯","15:30:45","現金"
2,"麻生雅人","✗","","カード"
3,"市川治義","","",""
```

---

## コーディング実装ガイド

### Vanilla JS の設計原則

```javascript
// ✅ 推奨パターン：シンプルで読みやすい

// 1. グローバル変数は最小限
const DATA = [...];      // 静的マスタ
let state = {};          // 動的状態
const KEY = 'jidai-07-28-v3';  // localStorage キー

// 2. 関数は単一責任
function setState(no) { /* ... */ }       // 状態初期化のみ
function setArr(no, val) { /* ... */ }   // 来場マークのみ
function save() { /* ... */ }            // 保存のみ
function render() { /* ... */ }          // 描画のみ

// 3. DOM 操作は render() に集約
// ❌ NG: 各関数内で DOM を直接更新
function setArr(no, val) {
  // ...
  document.getElementById('stat-arr').textContent = newVal;  // ← NG
}

// ✅ OK: render() で一括更新
function setArr(no, val) {
  // ...
  save();  // save() 内から render() を呼ぶ
}

function render() {
  // 全ての DOM 更新をここで実行
}
```

### 初期化シーケンス

```javascript
// ページロード時の初期化順序（重要）

// 1. localStorage から state を読み込む
let state = JSON.parse(localStorage.getItem(KEY) || '{}');

// 2. 全参加者の state を初期化（既存データは上書きしない）
DATA.forEach(p => {
  if (!state[p.no]) state[p.no] = {arr:null, pay:null, ts:null, noshow:false};
});

// 3. UI を構築
function init() {
  buildJumpbar();  // あいうえおボタン
  render();        // リスト表示
}

// 4. イベントリスナー登録
document.getElementById('q').addEventListener('input', render);

// 5. 初期化実行
init();
```

### エラーハンドリング

```javascript
// ✅ 推奨：エラー時でも graceful に動作

function setState(no) {
  if (!state) state = {};  // state が undefined の可能性に対応
  if (!state[no]) state[no] = {arr:null, pay:null, ts:null, noshow:false};
  return state[no];
}

function addParticipant() {
  // バリデーション
  if (!name.trim()) {
    alert('氏名を入力してください');
    return;  // 早期終了
  }
  
  try {
    // 処理
    const nextNo = Math.max(...DATA.map(p => p.no), 0) + 1;
    DATA.push({no: nextNo, name, kana});
    save();
  } catch (e) {
    console.error('ドタ参追加に失敗:', e);
    alert('追加に失敗しました');
  }
}
```

---

## パフォーマンス・セキュリティ

### パフォーマンス最適化

#### 1. render() の最適化

```javascript
// ❌ NG：毎回全データを処理
function render() {
  for (let i = 0; i < 1000; i++) {  // 不要なループ
    // ...
  }
}

// ✅ OK：必要な部分のみ処理
function render() {
  const filtered = filteredRows();  // フィルタ済みデータのみ処理
  const pending = filtered.filter(p => !isProcessed(p));
  const processed = filtered.filter(p => isProcessed(p));
  
  // 件数が少なければ DOM 操作も高速
  const html = pending.map(renderRow).join('');
}
```

#### 2. DOM バッチ更新

```javascript
// ❌ NG：複数回の DOM 更新（リフロー多発）
document.getElementById('stat-arr').textContent = stats.arr;
document.getElementById('stat-abs').textContent = stats.abs;
document.getElementById('stat-tot').textContent = DATA.length;
document.getElementById('stat-cash').textContent = cashAmount;

// ✅ OK：1 回の innerHTML で更新
const html = `
  <div>${stats.arr}</div>
  <div>${stats.abs}</div>
  <div>${DATA.length}</div>
  <div>${cashAmount}</div>
`;
document.getElementById('stats').innerHTML = html;
```

#### 3. localStorage 操作

```javascript
// ✅ OK：save() で必ず JSON.stringify（1 回のみ）
function save() {
  const serialized = JSON.stringify(state);  // 1 回のシリアライズ
  localStorage.setItem(KEY, serialized);
  render();
}

// ❌ NG：毎回シリアライズ
state[1].arr = true;
localStorage.setItem(KEY, JSON.stringify(state));
state[1].pay = 'c';
localStorage.setItem(KEY, JSON.stringify(state));  // 2 回目のシリアライズ
```

### セキュリティ考慮

#### 1. localStorage 限定スコープ

```javascript
// ✅ OK：ローカルストレージのみ（ネットワーク通信なし）
const KEY = 'jidai-07-28-v3';
localStorage.setItem(KEY, JSON.stringify(state));

// ❌ NG：サーバーに送信する場合は HTTPS + 認証必須
fetch('/api/save', {
  method: 'POST',
  body: JSON.stringify(state)
  // ← このパターンは本仕様では不採用（オフラインで動作すべき）
});
```

#### 2. HTML インジェクション防止

```javascript
// ❌ NG：ユーザー入力を直接 HTML に埋め込む
const name = document.getElementById('add-name').value;
html += `<div class="name">${name}</div>`;  // XSS 脆弱性

// ✅ OK：textContent で自動エスケープ
const div = document.createElement('div');
div.textContent = name;  // HTML タグは文字列として表示される

// または template literal で要素生成
const div = document.createElement('div');
div.className = 'name';
div.append(name);  // textContent と同じ効果
```

#### 3. 不正な操作防止

```javascript
// ✅ OK：バリデーション必須
function addParticipant() {
  const name = document.getElementById('add-name').value.trim();
  
  // 空文字列チェック
  if (!name) {
    alert('必須');
    return;
  }
  
  // 長さチェック（UI 上は制限できないが、サーバー送信時は必須）
  if (name.length > 50) {
    alert('50文字以内で入力してください');
    return;
  }
}
```

---

## トラブルシューティング

### よくあるトラブルと対処法

#### 1. Jump ボタンがクリックしても反応しない

**症状**: あいうえおボタンを押しても何も起きない

**原因**: `outerHTML` で HTML 文字列に変換してから、onclick ハンドラが失われている

```javascript
// ❌ 原因コード
jumpbar.innerHTML = KANA_ORDER.map(k => {
  const btn = document.createElement('button');
  btn.onclick = () => jumpToKana(k);
  return btn.outerHTML;  // ← onclick が削除される
}).join('');
```

**対処**:
```javascript
// ✅ 修正：appendChild を使用
const jumpbar = document.getElementById('jumpbar');
jumpbar.innerHTML = '';
KANA_ORDER.forEach(k => {
  const btn = document.createElement('button');
  btn.onclick = () => jumpToKana(k);
  jumpbar.appendChild(btn);  // ← DOM に直接追加
});
```

#### 2. 全参加者が「処理済み」で表示される

**症状**: ブラウザを開いた直後、全員が「処理済み」セクションに表示される

**原因**: 新規参加者の state が初期化されていない。`state[no]` が undefined で、`undefined !== null` が true になる

```javascript
// ❌ 問題のあるコード
function isProcessed(p) {
  const s = state[p.no] || {};  // undefined の場合 {}
  return s.arr !== null || s.noshow;  // undefined !== null = true
}
```

**対処**: ページロード時に全参加者を初期化

```javascript
// ✅ 修正：初期化を追加
DATA.forEach(p => {
  if (!state[p.no]) state[p.no] = {arr:null, pay:null, ts:null, noshow:false};
});
```

#### 3. ドタ参追加後、支払い情報が保存されない

**症状**: ドタ参追加 → 支払い方法を設定 → ページリロード → 支払い情報が消えている

**原因**: setState() の前に setArr() を呼んでいる（初期化順序が違う）

```javascript
// ❌ 間違い
setArr(nextNo, true);  // state[nextNo] が新規初期化
const s = state[nextNo];
s.pay = payment;  // ← この時点では既に state は初期化されている
```

**対処**: setState() → s.pay 設定 → setArr() の順序を守る

```javascript
// ✅ 正解
const s = setState(nextNo);
s.pay = payment;  // 支払い方法を先に設定
setArr(nextNo, true);  // その後に来場打刻
```

#### 4. localStorage キャッシュが残って、新規テストデータが反映されない

**症状**: HTML を編集して新規参加者を追加 → ブラウザをリロード → 前のデータが表示される

**原因**: localStorage にある古い state が優先される

**対処**: 
- オプション A: KEY をバージョンアップ（v2 → v3）
- オプション B: ブラウザの DevTools で localStorage を削除

```javascript
// DevTools の Console で実行
localStorage.removeItem('jidai-07-28-v2');
location.reload();
```

#### 5. CSV 出力が文字化けして見える

**症状**: CSV をエクセルで開くと 「相見高志」が「相見é«˜å¿—」に見える

**原因**: エクセルが BOM なしの UTF-8 として認識していない

**対処**: CSV 作成時に BOM を付与

```javascript
// ✅ BOM 付きで出力
const BOM = '﻿';  // UTF-8 BOM
const csv = BOM + '番号,氏名,来場,...\n' + csvContent;
const blob = new Blob([csv], {type: 'text/csv;charset=utf-8'});
```

#### 6. スマホで操作が遅い

**症状**: スマートフォンでボタンをタップしても反応が遅い（1-2 秒かかる）

**原因**: render() が重い（参加者数が多い場合）

**対処**: 
- 参加者リストを仮想化（画面外の要素は DOM に作らない）
- または別ファイルに分割（200 名以上の場合）

```javascript
// ✅ 簡易版：表示件数を制限
const MAX_VISIBLE = 50;
const visible = filtered.slice(0, MAX_VISIBLE);
const html = visible.map(renderRow).join('');
```

---

## チェックリスト

### 新規イベント実装時

```markdown
## データ定義
- [ ] DATA 配列に全参加者を定義（no, name, kana）
- [ ] kana はひらがな全角で統一
- [ ] no は昇順・連番

## localStorage
- [ ] KEY を `イベント-日付-バージョン` に設定（例: jidai-07-28-v3）
- [ ] ページロード時に全参加者の state を初期化

## UI 実装
- [ ] 参加者リストをあいうえお順で表示
- [ ] あいうえおボタンは appendChild で実装（❌ outerHTML NG）
- [ ] 来場・欠席・支払いボタンが動作確認
- [ ] ドタ参追加パネルが表示・非表示切り替え
- [ ] CSV 出力が機能確認

## 動作確認
- [ ] PC・タブレット・スマホ全デバイスで表示確認
- [ ] Jump ボタンでスクロール動作確認
- [ ] 来場打刻 → タイムスタンプ自動記録を確認
- [ ] ドタ参追加 → 即座に「処理済み」リストに表示を確認
- [ ] ページリロード後も state が保持されているか確認
- [ ] CSV 出力で全データが含まれているか確認

## パフォーマンス
- [ ] ボタンクリック → UI 更新が <100ms であることを確認
- [ ] localStorage サイズが 1MB 以下であることを確認

## デプロイ
- [ ] GitHub Pages にプッシュ
- [ ] 本番 URL でアクセス確認
- [ ] ブラウザキャッシュクリア（Cmd+Shift+R）で再確認
```

### 定期運用時

```markdown
## イベント終了後
- [ ] CSV をダウンロード・保存
- [ ] 次回イベント用に新規 v ファイル（v4, v5...）を作成
- [ ] 過去 version の v1, v2 は archive フォルダに移動
- [ ] GitHub に push・バージョン管理

## トラブル発生時
- [ ] DevTools Console でエラーがないか確認
- [ ] localStorage の KEY を確認（正しい KEY か）
- [ ] 別ブラウザ・別デバイスで症状が再現するか確認
- [ ] 症状が再現したら CLAUDE.md に記録
```

---

## 付録

### A. 完全な HTML テンプレート

本仕様に基づいた `jidai-closing-reception-0728-v3.html` を参照。

### B. CSS カラースキーム

```css
:root {
  --accent: #8B1A1A;        /* 深紅・メインカラー */
  --accent2: #5a0f0f;       /* 濃い紫 */
  --good: #16a34a;          /* 緑・OK 状態 */
  --good2: #15803d;         /* 濃い緑 */
  --bg: #0f172a;            /* ダークグレー背景 */
  --panel: rgba(15, 23, 42, 0.88);  /* パネル背景 */
  --line: #334155;          /* 罫線 */
  --ink: #e5e7eb;           /* テキスト色 */
  --muted: #94a3b8;         /* 補助テキスト */
}
```

### C. よく使う正規表現・バリデーション

```javascript
// 氏名バリデーション（3-20 文字、日本語 or 英数字）
const nameRegex = /^[ぁ-んァ-ヴー一-龥a-zA-Z0-9\s]{3,20}$/;
if (!nameRegex.test(name)) {
  alert('氏名は 3-20 文字（日本語・英数字）で入力してください');
}

// かなバリデーション（ひらがなのみ）
const kanaRegex = /^[ぁ-ん]{2,30}$/;
if (!kanaRegex.test(kana)) {
  alert('かなはひらがなで入力してください');
}
```

---

**最終更新**: 2026-07-28  
**バージョン**: 1.0  
**作成者**: Claude Code  
**対象**: JIDAI イベント受付システム標準仕様
