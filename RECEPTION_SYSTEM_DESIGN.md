# JIDAI イベント受付システム — 設計書 v1.0

> 本ドキュメントは JIDAI クロージング受付（0728, 0729）で検証された受付システムの標準仕様です。
> 他のイベント受付ページの実装時に参考にしてください。

## 概要

参加者の来場・欠席・支払い方法を管理する、シンプルでレスポンシブなウェブベース受付システム。

- **デバイス**: スマホ・タブレット・PC 対応
- **機能**: 来場打刻・欠席マーク・支払い管理・ドタ参追加・CSV 出力
- **ストレージ**: localStorage で状態永続化
- **フレームワーク**: Vanilla HTML/CSS/JavaScript（フレームワーク不使用）

## 主要機能

### 1. 参加者リスト表示

**特徴**:
- あいうえお順（KANA_ORDER ベース）で自動ソート
- 未処理・処理済み の 2 セクション分離
- 実時間での統計表示（来場数・欠席数・当日現金額）

**実装**:
- `DATA` 配列に参加者情報を定義（no, name, kana）
- `isProcessed()` で未処理/処理済みを判定
- `render()` で UI 更新

```javascript
const DATA = [
  {"no": 1, "name": "相見高志", "kana": "あいみたかし"},
  {"no": 2, "name": "麻生雅人", "kana": "あそうまさと"},
  // ...
];
```

### 2. あいうえお Jump 機能

**特徴**:
- あ・か・さ・た・な・は・ま・や・ら・わ のボタンで該当セクションへスクロール
- 該当者がいないグループはボタン disabled

**重要**: イベントハンドラは `appendChild()` で設定する。`outerHTML` で HTML 文字列に変換してはいけない（click handlers が失われる）。

```javascript
// ✅ 正解：appendChild で直接 DOM に追加
const jumpbar = document.getElementById('jumpbar');
jumpbar.innerHTML = '';
KANA_ORDER.forEach(k => {
  const btn = document.createElement('button');
  btn.onclick = () => jumpToKana(k);
  jumpbar.appendChild(btn);
});

// ❌ 間違い：outerHTML で HTML 文字列に変換すると handlers が失われる
jumpbar.innerHTML = KANA_ORDER.map(k => {
  const btn = document.createElement('button');
  btn.onclick = () => jumpToKana(k);
  return btn.outerHTML;  // ← handlers が失われる
}).join('');
```

### 3. 来場・欠席・支払い管理

**来場打刻** (`✓`):
- `state[no].arr = true` で来場をマーク
- JST タイムスタンプ自動記録
- クリックすると灰色化・処理済みセクションに移動
- undo ボタン（↩）で取り消し可能

**欠席マーク** (`✗`):
- `state[no].noshow = true` でマーク
- 来場とは独立・同時には選択不可

**支払い方法**:
- 現金（¥）: `state[no].pay = 'c'`
- カード（カ）: `state[no].pay = 'k'`
- 免除（免）: `state[no].pay = 'e'`
- トグル式で複数選択可能

**状態保存**:
```javascript
const KEY = 'jidai-07-28-v2';  // イベントごとに一意の KEY を使う
function save() {
  localStorage.setItem(KEY, JSON.stringify(state));
  render();
}
```

### 4. ドタ参追加機能

**UI**:
- 「➕ ドタ参追加」ボタンでパネル表示
- 入力項目: 氏名、かな、支払い方法（必須）
- 「追加して来場打刻」で即座に処理済みリストに追加

**実装**:
```javascript
function addParticipant() {
  const name = document.getElementById('add-name').value.trim();
  const kana = document.getElementById('add-kana').value.trim();
  const payment = document.getElementById('add-payment').value;
  
  if (!name || !payment) {
    alert('必須項目を入力してください');
    return;
  }
  
  const nextNo = Math.max(...DATA.map(p => p.no), 0) + 1;
  DATA.push({no: nextNo, name, kana});
  const s = setState(nextNo);
  s.pay = payment;
  setArr(nextNo, true);  // 即座に来場打刻
  
  // フォームリセット
  document.getElementById('add-name').value = '';
  document.getElementById('add-kana').value = '';
  document.getElementById('add-payment').value = '';
}
```

### 5. CSV 出力

**機能**:
- 全参加者のデータを CSV で下記フォーマットで出力
- ファイル名: `JIDAI_[日付].csv`

**フォーマット**:
```
番号,氏名,来場,時刻,支払
1,相見高志,◯,15:30:45,現金
2,麻生雅人,✗,,
```

### 6. リセット機能

**説明**:
- 確認ダイアログを表示
- localStorage を削除・ページリロード
- 全参加者の状態を初期化

## 状態管理

### State 構造

```javascript
// 各参加者の state
state[no] = {
  arr: null | true | false,    // null=未処理, true=来場, false=欠席
  pay: null | 'c' | 'k' | 'e', // null=未設定, c=現金, k=カード, e=免除
  ts: "15:30:45" | null,       // 来場時刻（JSTタイムスタンプ）
  noshow: true | false         // 欠席フラグ（arr=false の時は true）
}

// グローバル state
state = {
  1: { arr: true, pay: 'c', ts: "15:30:45", noshow: false },
  2: { arr: false, pay: null, ts: null, noshow: true },
  3: { arr: null, pay: null, ts: null, noshow: false },
}
```

### 状態判定

```javascript
function isProcessed(p) {
  const s = state[p.no] || {};
  return s.arr !== null || s.noshow;
}

// true = 処理済み（来場 or 欠席）
// false = 未処理
```

**重要**: 新規参加者追加時は必ず初期化してから使用

```javascript
const nextNo = Math.max(...DATA.map(p => p.no), 0) + 1;
DATA.push({no: nextNo, name, kana});
setState(nextNo);  // ← 必須：{arr:null, pay:null, ts:null, noshow:false} を初期化
```

## あいうえお順ソート実装

### KANA_RANGES テーブル

```javascript
const KANA_ORDER = ['あ','か','さ','た','な','は','ま','や','ら','わ'];
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
```

### Jump 機能の実装パターン

```javascript
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

## スタイリング

### カラースキーム

```css
:root {
  --accent: #8B1A1A;        /* 深紅・メインカラー */
  --good: #16a34a;          /* 緑・OK 状態 */
  --muted: #94a3b8;         /* グレー・補助テキスト */
}
```

### 状態視覚化

| 状態 | CSS | 説明 |
|------|-----|------|
| 未処理 | 通常色 | デフォルト表示 |
| 来場済み | `.row.arrived` | opacity .42 + grayscale .2 |
| ボタン active | `.mini.active` | 黄色枠+光 |

## localStorage キー管理

**命名規則**: `jidai-[日付]-[バージョン]`

```javascript
const KEY = 'jidai-07-28-v2';  // イベント・日付・キャッシュバージョンで一意化
```

**理由**: 
- 同一イベントの複数開催や再テスト時のキャッシュ競合を防ぐ
- キャッシュクリア時も同じ KEY で安全に初期化可

## 実装チェックリスト（新規イベント用）

```markdown
- [ ] イベント・日付に応じた新規 HTML ファイル作成
- [ ] DATA 配列に参加者情報を all-in-one で定義
- [ ] KEY を `イベント-日付-バージョン` に設定
- [ ] あいうえお Jump ボタンは appendChild で実装（outerHTML NG）
- [ ] ドタ参追加時は setState() → setArr() 順序を守る
- [ ] CSV 出力ファイル名を `[イベント名]_[日付].csv` に統一
- [ ] GitHub Pages 反映前にローカル localStorage をクリア
- [ ] jump・来場打刻・支払い管理の動作確認
- [ ] mobile デバイスで表示・操作確認
```

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| Jump ボタンがクリックしても反応しない | `outerHTML` で HTML 文字列変換 | `appendChild()` を使用 |
| 全参加者が「処理済み」で表示される | state 初期化漏れ | `DATA.forEach(p => if (!state[p.no]) state[p.no] = {...})` |
| localStorage キャッシュが残る | キーが同じまま再テスト | KEY をバージョンアップ（e.g., v1 → v2） |
| ドタ参追加後に Jump が反応しない | 新規追加者が DOM に反映されていない | `save()` → `render()` を確認 |

## 標準化への次ステップ

1. **テンプレート HTML 作成**: 
   - `reception-template.html` として、日付・参加者数だけをカスタマイズ可能な形に

2. **設定ファイル化**:
   - イベント名・日付・参加者リスト を JSON で外部化
   - Web UI で簡単にカスタマイズ可能に

3. **機能拡張案**:
   - QR コード来場打刻
   - リアルタイム同期（複数タブ・複数デバイス）
   - 支払い忘れ通知
   - アンケート機能統合

---

**最終更新**: 2026-07-28  
**バージョン**: 1.0
