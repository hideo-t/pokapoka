# 初期設定書（システム導入手順）

> 新規にこのシステムを導入する場合の **0→1 セットアップ手順**。  
> 所要時間：合計 約2〜3時間。

---

## 前提：必要なもの

- **Google アカウント**（Workspace 推奨、無料 Gmail でも可）
- **GitHub アカウント**（フロント公開用）
- **LINE Developers アカウント**（Phase 1.7 で必要、後でも可）
- **PC**: Windows / Mac / Linux いずれも可、Node.js 環境推奨（clasp 使用）

---

## ステップ1: Google スプレッドシート作成（5分）

1. <https://sheets.google.com/> で新規スプレッドシートを作成
2. ファイル名を運営名に変更（例: `ぽかぽか食堂_予約データ`）
3. URL から **スプレッドシートID** をコピー
   - URL 例: `https://docs.google.com/spreadsheets/d/15Yw5uQejs4Y0XkKu8t_ILsB7nP-8x1ZeIda4tKGugjw/edit`
   - ID 部分: `15Yw5uQejs4Y0XkKu8t_ILsB7nP-8x1ZeIda4tKGugjw`
4. シート4枚は **後で自動生成** されるので、ここでは空のまま

---

## ステップ2: Apps Script プロジェクト作成（10分）

### 方法A: clasp を使う（推奨）

```bash
# Node.js 必要
npm install -g @google/clasp
clasp login              # ブラウザ認証
clasp create --type standalone --title "ぽかぽか食堂予約システム"
# 出力されたスクリプトID をメモ
```

その後、本リポジトリの `gas/` ディレクトリを clasp プロジェクト直下に配置：

```bash
git clone https://github.com/hideo-t/pokapoka.git
cd pokapoka
# gas/ ディレクトリと .clasp.json をローカルに作成
# .clasp.json: { "scriptId": "上記でメモしたID", "rootDir": "gas" }
clasp push -f
```

### 方法B: 手動コピペ

1. 上記スプレッドシートを開く → **拡張機能 → Apps Script**
2. 既定の `Code.gs` を削除
3. 本リポジトリの `gas/` 内 10 ファイル（`Code.gs / Config.gs / DataAccess.gs / Tenant.gs / Billing.gs / Auth.gs / Reservation.gs / UserMaster.gs / Notify.gs / Admin.gs`）と `appsscript.json` を、Apps Script エディタの「ファイル ＋」で同名で作成・貼付

---

## ステップ3: 環境定数の書き換え（5分）

`Config.gs` 冒頭の以下を編集：

```javascript
var SPREADSHEET_ID = '__ステップ1でコピーしたID__';
var ORG_NAME      = '__食堂名__';            // 例: 'ぽかぽか食堂'
var CONTACT_EMAIL = '__連絡先メール__';     // 例: 'contact@example.com'
var DEFAULT_TENANT_ID = 'DEFAULT';          // Phase 1 はこのまま
```

clasp を使った場合は `clasp push -f` で反映。手動コピペの場合は GAS エディタで保存。

---

## ステップ4: シート初期化（2分）

1. GAS エディタを **F5 リロード**
2. 関数選択プルダウンで **`initializeSheets`** を選び **▶ 実行**
3. 初回はアクセス権限の許可ダイアログ → 「許可」（Google アカウント承認）
4. 実行ログに以下が出れば成功：
   ```
   Created: 予約, 開催日程, 認証コード, 利用者マスタ, 管理者, メールログ
   Updated: ...
   ```
5. スプレッドシートを再読み込みすると6シートが自動生成されている

---

## ステップ5: 管理者と開催日程の初期データ投入（10分）

### 「管理者」シート（2行目に）
| tenantId | メールアドレス | 権限 |
|---|---|---|
| DEFAULT | あなたのメール | admin |

### 「開催日程」シート（2行目以降）
| tenantId | 開催日 | 受付上限（大人）| 受付上限（子ども）| 受付状態 | 備考 |
|---|---|---|---|---|---|
| DEFAULT | 2026/06/01 | 20 | 30 | 受付中 | |
| DEFAULT | 2026/06/15 | 20 | 30 | 受付中 | カレー大会 |

「現在予約数」列はありません（v2 で廃止、動的集計に変更）。

---

## ステップ6: GAS Web App デプロイ（5分）

1. Apps Script エディタ右上 **デプロイ → 新しいデプロイ**
2. 種類: **ウェブアプリ**
3. 設定:
   - 実行: **自分**（オーナーアカウント）
   - アクセスできるユーザー: **全員**
4. **デプロイ** → 表示された **ウェブアプリ URL** をコピー

### 疎通確認
```
https://script.google.com/macros/s/____/exec?action=health&tenant=DEFAULT
```
ブラウザで開いて `{"ok":true,"message":"こども食堂予約システム稼働中","tenantId":"DEFAULT"}` が返ればOK。

---

## ステップ7: 定期トリガー登録（2分）

GAS エディタで関数選択 → **`setupTriggers`** → **▶ 実行**

以下3つが自動登録されます：
- `cleanupExpiredOTPs` — 15分おき (OTP 掃除)
- `sendDayBeforeReminder` — 毎日18時 (前日リマインダー)
- `sendWeeklyReport` — 月曜10時 (週間レポート → 管理者宛)

左メニュー「⏰ トリガー」で3件登録されていることを確認。

---

## ステップ8: フロントエンドの GAS_URL 書き換え（2分）

`index.html` 冒頭の：

```javascript
const GAS_URL = '__ステップ6でコピーしたWebApp URL__';
```

を書き換え、保存。

---

## ステップ9: GitHub Pages 公開（15分）

```bash
cd /path/to/pokapoka
echo "gas/
.clasp.json
.claspignore
appsscript.json
.env
docs/PRIVATE_*.md
docs/STEP1_VERIFICATION.md" > .gitignore

git init -b main
git config user.email "you@example.com"
git config user.name "Your Name"
git add index.html docs/ README.md .gitignore
git commit -m "initial release"
git remote add origin https://github.com/yourname/yourrepo.git
git push -u origin main
```

GitHub 画面で：
1. **Settings → Pages**
2. **Source**: `Deploy from a branch`
3. **Branch**: `main` / `/(root)`
4. **Save**

1〜3分後、本番URL公開：`https://yourname.github.io/yourrepo/?tenant=DEFAULT`

---

## ステップ10: LINE 公式アカウント設定（オプション・Phase 1.7 着手前）

1. **LINE 公式アカウント** 作成: <https://manager.line.biz/>
2. **LINE Developers** で Messaging API チャネル作成: <https://developers.line.biz/>
3. 取得する値：
   - Channel Access Token (長期)
   - Channel Secret
   - LIFF ID（LIFF アプリ作成後）
4. **GAS スクリプトプロパティに登録**（手順は [STEP3_LINE_SETUP.md](./STEP3_LINE_SETUP.md)）
5. **Webhook URL** を GAS Web App URL に設定
6. LIFF Endpoint URL を GitHub Pages の本番URL に設定

---

## ステップ11: 動作テスト（10分）

1. 本番URLをスマホで開く（強制リロード Ctrl+Shift+R）
2. 自分のメールアドレスでログイン → OTP メール受信 → 認証
3. 日程選択 → 氏名・人数入力 → 確認 → 予約
4. 確認メール受信
5. スプレッドシート「予約」シートに行追加されていることを確認
6. 管理者の場合 ☰ メニューから「かんりダッシュボード」が見えること
7. CSV ダウンロード動作

---

## ステップ12: 本番ガード ON（運用開始時）

`Config.gs` で：
```javascript
var PRODUCTION_GUARD = true;
```
に変更 → `clasp push -f` または手動保存。

これで誤って `clearAllData` / `seedDemoData` が実行されてもブロックされます。

---

## 完了チェックリスト

- [ ] スプレッドシート作成 + ID 取得
- [ ] Apps Script プロジェクト作成
- [ ] Config.gs の SPREADSHEET_ID / ORG_NAME / CONTACT_EMAIL 設定
- [ ] `initializeSheets` 実行 → 6シート自動生成
- [ ] 管理者シートに自分のメール登録
- [ ] 開催日程シートに初期データ
- [ ] Web App デプロイ → URL取得
- [ ] `setupTriggers` で定期トリガー登録
- [ ] フロントの GAS_URL 書き換え
- [ ] GitHub Pages 公開
- [ ] スマホで予約フロー動作確認
- [ ] PRODUCTION_GUARD = true (本番稼働時)
- [ ] (任意) LINE 連携完了

完了したら → [ADMIN_OPERATION_MANUAL.md](./ADMIN_OPERATION_MANUAL.md) を読んで日常運用に入る。
