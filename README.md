# こども食堂 予約システム

スマホから予約 → スプレッドシートで一元管理。サーバー不要・月額ゼロで運用できる軽量予約システム。

## 構成

```
kodomo/
├── frontend/
│   └── index.html          # シングルファイル予約フォーム（LINE共有可）
├── gas/
│   ├── Code.gs             # 定数・ルーティング・初期化
│   ├── Auth.gs             # メール OTP 認証
│   ├── Reservation.gs      # 予約 CRUD・満席制御
│   ├── Notify.gs           # メール通知
│   └── Admin.gs            # 管理者向け API
└── README.md
```

## 技術スタック

| レイヤー | 採用 |
|---|---|
| フロントエンド | HTML + Vanilla JS（シングルファイル） |
| バックエンド | Google Apps Script（GAS）Web App |
| データストア | Google スプレッドシート |
| 認証 | メール OTP（6桁・10分有効） |
| 通知 | GAS MailApp（Gmail経由） |

---

## デプロイ手順

> ![スクリーンショット: 全体フロー](docs/img/00-overview.png)
> *（プレースホルダー）*

### 1. Googleスプレッドシートを新規作成

> ![スクリーンショット: スプレッドシート作成](docs/img/01-create-sheet.png)

1. <https://sheets.google.com/> で新規スプレッドシートを作成
2. URL から **スプレッドシートID** をコピー（`/d/` と `/edit` の間の英数字）

### 2. シート4枚は自動生成されます

`initializeSheets()` を後の手順で実行すると、以下のシートが自動的に作られヘッダー行が挿入されます：

| シート名 | 用途 |
|---|---|
| 予約 | 予約レコード本体 |
| 開催日程 | 開催日と上限定員 |
| 認証コード | OTPコード（一時的） |
| 管理者 | 管理者メールリスト |

### 3. Apps Script プロジェクトを作成

> ![スクリーンショット: GASエディタを開く](docs/img/03-open-gas.png)

1. スプレッドシートのメニュー → **拡張機能 → Apps Script**
2. 既定の `Code.gs` を削除し、本リポジトリの `gas/` 内 5 ファイル（`Code.gs / Auth.gs / Reservation.gs / Notify.gs / Admin.gs`）をすべて貼り付け
   - 各ファイルは Apps Script エディタの左側「ファイル ＋」ボタンで作成
   - ファイル名は **拡張子なし** で `Code`, `Auth`, `Reservation`, `Notify`, `Admin` の5つ

### 4. SPREADSHEET_ID を設定

`Code.gs` 冒頭の以下の定数を書き換え：

```javascript
var SPREADSHEET_ID = '__PASTE_SPREADSHEET_ID_HERE__';  // ← 1. でコピーしたIDに置換
var ORG_NAME = 'こども食堂';                             // ← 表示名を変更
var CONTACT_EMAIL = 'contact@example.com';              // ← お問い合わせ先メール
```

### 5. シート初期化を実行

> ![スクリーンショット: initializeSheets 実行](docs/img/05-init-sheets.png)

Apps Script エディタの関数選択プルダウンで `initializeSheets` を選び **▶ 実行**。
初回はアクセス権限の許可ダイアログが出るので承認。

### 6. 開催日程をスプレッドシートに入力

「開催日程」シートを開いて、開催する日を入力します。

| 開催日 | 受付上限（大人） | 受付上限（子ども） | 受付状態 | 備考 |
|---|---|---|---|---|
| 2026/06/01 | 20 | 30 | 受付中 | |
| 2026/06/15 | 20 | 30 | 受付中 | カレー大会 |

「現在予約数」列（D・E）は予約作成時に GAS 側で自動集計されるので、空欄でOK。

### 7. 管理者を登録

「管理者」シートに管理用メールアドレスを入力：

| メールアドレス | 権限 |
|---|---|
| admin@example.com | admin |

### 8. Web App としてデプロイ

> ![スクリーンショット: Web App デプロイ](docs/img/08-deploy.png)

1. Apps Script エディタ右上の **デプロイ → 新しいデプロイ**
2. 種類: **ウェブアプリ**
3. 設定:
   - 実行: 自分（オーナーアカウント）
   - アクセスできるユーザー: **全員**
4. **デプロイ**を押し、表示された **ウェブアプリ URL** をコピー

### 9. フロントエンドに URL を設定

`frontend/index.html` の以下の定数を書き換え：

```javascript
const GAS_URL = '__PASTE_GAS_DEPLOY_URL_HERE__';  // ← 8. でコピーした URL に置換
```

### 10. トリガーを登録

Apps Script エディタの関数選択で `setupTriggers` を選び **▶ 実行**。
これで以下が自動実行されます：

- `cleanupExpiredOTPs` : 15分おき（期限切れOTPの掃除）
- `sendDayBeforeReminder` : 毎日18時（翌日来場者へのリマインダー）

### 11. フロントエンドを公開

選べる公開方法：

**A. GitHub Pages（推奨）**
1. GitHub リポジトリの Settings → Pages
2. Source: `main` ブランチ・`/frontend` フォルダ
3. 公開された URL を LINE 等で共有

**B. ファイル直接共有**
1. `index.html` を Google Drive 等にアップロード
2. 一般公開でリンクを取得 → LINE 等で共有

**C. Cloudflare Pages / Vercel / Netlify**
1. `frontend/` をデプロイ
2. カスタムドメインを設定（必要なら）

---

## 動作確認

> ![スクリーンショット: 動作確認フロー](docs/img/11-test-flow.png)

1. フロントエンド URL にアクセス
2. メールアドレス入力 → OTP受信 → OTP入力
3. 日程選択 → 予約フォーム入力 → 確認 → 予約
4. 「予約」シートに行が追加されていること、確認メールが届いていることを確認

---

## 運営オペレーション

### 予約一覧の確認

スプレッドシートの「予約」シートを直接開くか、管理者ログイン → 管理ダッシュボードから一覧表示。

### キャンセル処理

- 利用者本人: マイ予約画面から自分でキャンセル可能
- 管理者: 管理ダッシュボードからステータスを「キャンセル」に変更

### CSV ダウンロード

管理者ダッシュボードの「📥 CSVダウンロード」ボタンから期間内予約をCSV取得（BOM付きUTF-8、Excel対応）。

### 開催日の追加・上限変更

「開催日程」シートに行を追加するか、既存の上限値を直接編集するだけで反映されます。再デプロイ不要。

---

## セキュリティ

| 項目 | 対策 |
|---|---|
| OTP総当たり | 5回失敗で約10分ロック |
| トークン管理 | `PropertiesService` に保存・1時間有効 |
| 二重予約 | `LockService.getScriptLock()` で排他制御 |
| メールインジェクション | サーバー側で改行・空白除去 |
| 管理者権限 | 「管理者」シートのメール照合 + isAdmin チェック |
| CORS | GAS WebApp は同一オリジン以外も許可しているため、API レベルでトークン検証 |

---

## メール送信枠

| Googleアカウント種別 | 1日あたりの送信枠 |
|---|---|
| 無料（@gmail.com） | 100通 |
| Google Workspace | 1,500通 |

OTPメール・予約確認メール・キャンセル通知・前日リマインダーすべて GAS の `MailApp` で送信されます。
大規模運用時は Workspace への切り替えを検討してください。

---

## トラブルシューティング

### OTPメールが届かない
- 迷惑メールフォルダを確認
- `Notify.gs` の `MailApp` 送信エラーが Apps Script の「実行履歴」に出ていないか確認
- 1日の送信枠に到達していないか確認

### 「シート〇〇が見つかりません」エラー
- `initializeSheets()` を実行
- シート名がコードの定数と一致しているか確認（全角・半角・スペース）

### 「セッションの有効期限が切れました」
- 認証から1時間経過 → 再度ログイン
- 必要なら `Code.gs` の `SESSION_EXPIRY_HOURS` を変更

### 予約が同時に同じ日に殺到した時の挙動
- `LockService.waitLock(10000)` で順次処理
- 上限を超えた予約は「受付上限を超えています」と返り、フロントでエラー表示

---

## Phase 2 / Phase 3（拡張予定）

### Phase 2
- [x] マイ予約画面 + キャンセル機能（実装済）
- [x] 前日リマインダー自動送信（実装済・setupTriggers で有効化）
- [x] 管理者ダッシュボード（実装済）

### Phase 3
- [ ] 週間集計レポート自動送信
- [ ] 待機リスト管理
- [x] CSVエクスポート（実装済）
- [ ] 空き状況グラフ表示（Chart.js を有効化）

Chart.js を使うには `frontend/index.html` 末尾のコメントアウト行を有効化してください。

---

## ライセンス

このコードはこども食堂の運営支援を目的としています。改変・再利用自由。

---

# 📐 Phase 1.5 / Phase 2 設計（実装着手前の事前提示）

> Hideoさんへの確認用ドキュメント。本実装着手前にこのセクションをレビュー → OK ならコーディング開始。
> NG 箇所があればコメント → 修正後に着手。

## ファイル構成（PascalCase 統一）

```
gas/
├── Code.gs            ルーター（doGet/doPost）・初期化・トリガー登録
├── Config.gs          ★新規 定数集約（シート名 / ステータス / 上限 / プラン）
├── DataAccess.gs      ★新規 DB操作の抽象化レイヤー（Phase 5 で Supabase 化のポイント）
├── Tenant.gs          ★新規 テナント管理（Phase 1: DEFAULT 固定 → Phase 2: 台帳引き）
├── Auth.gs            OTP認証・セッション・認可（authorizeTenantRequest）
├── Reservation.gs     予約のビジネスロジック（重複チェック / 満席制御 / 待機リスト）
├── UserMaster.gs      ★新規 利用者マスタの集計・更新（Phase 1.5）
├── Notify.gs          メール送信ラッパー + メールログ蓄積
├── Admin.gs           管理者 API（一覧 / CSV / サマリ / seedDemoData / clearAllData）
└── Billing.gs         ★新規 課金スタブ（Phase 2 で Stripe Webhook 実装）
```

### 各ファイル冒頭の責務コメント雛形（全ファイルに必須）

```javascript
/**
 * Reservation.gs
 * 責務: 予約のCRUD、重複チェック（L1/L2/L3）、満席制御、待機リスト登録
 * 依存: Config.gs, DataAccess.gs, Tenant.gs, Auth.gs, UserMaster.gs, Notify.gs
 * 呼出元: Code.gs（doPost ルーター経由）
 * Phase 2 移行時: tenantId 引数で分岐、authorizeTenantRequest による課金検証
 */
```

---

## DataAccess.gs 公開 API 仕様（実装前のインターフェース確定）

> Phase 5 (Supabase 移行) 時の差し替えポイント。  
> 戻り値は **null / 配列 / オブジェクト** で統一。例外は throw しない（呼出元で扱いやすくするため）。  
> 全関数の第1引数は `tenantId`。Phase 1 では `'DEFAULT'`、Phase 2 では台帳から解決される実 ID。

### 📋 予約 (Reservation)

| シグネチャ | 役割 |
|---|---|
| `dao_getReservation(tenantId, reservationId)` | 単一予約取得 (null=未存在) |
| `dao_listReservations(tenantId, filters)` | 一覧取得。filters: `{email?, date?, status?, fromDate?, toDate?, createdAfter?}` |
| `dao_createReservation(tenantId, data)` | 行追加。data: `{name, email, phone, date, adultCount, childCount, allergy, status, source?}`。返値: 行番号と shortUuid |
| `dao_updateReservation(tenantId, reservationId, updates)` | 部分更新。`{status?, ...}` |
| `dao_countReservationsFor(tenantId, date)` | 指定日の `{adult, child, count}` 集計（キャンセル除外） |

### 📅 開催日程 (Schedule)

| シグネチャ | 役割 |
|---|---|
| `dao_getSchedule(tenantId, date)` | 単一日程取得 (null=未存在) |
| `dao_listSchedules(tenantId, options)` | 一覧。options: `{includePast?:false, includeClosed?:true}` |
| `dao_updateScheduleStatus(tenantId, date, status)` | 受付状態変更（受付中/満席/停止） |

> 注: 数式列 D・E（現在予約数）は **使わず**、`dao_countReservationsFor` で動的集計する設計に変更。スプレッドシート編集時のラグを排除。

### 👤 利用者マスタ (UserMaster) — Phase 1.5

| シグネチャ | 役割 |
|---|---|
| `dao_getUserMaster(tenantId, email)` | 取得 (null=未存在) |
| `dao_upsertUserMaster(tenantId, email, data)` | 作成 or 更新 (`{name?, phone?, allergies?, adminNote?, blocked?}`) |
| `dao_listUserMasters(tenantId, filters?)` | 一覧。filters: `{blocked?, minVisits?}` |
| `dao_recomputeUserMaster(tenantId, email)` | 予約シート再走査して再集計 (visits/cancels/firstVisit/lastVisit) |

### 🔐 認証 (OTP)

| シグネチャ | 役割 |
|---|---|
| `dao_saveOtp(tenantId, email, code, expiry)` | OTP行追加 (失敗回数=0で初期化) |
| `dao_findActiveOtp(tenantId, email)` | 最新の未使用OTP取得 (null=なし) |
| `dao_invalidateOtp(tenantId, rowNum)` | 使用済みフラグON |
| `dao_incrementOtpFailCount(tenantId, rowNum)` | 失敗カウント+1。返値: 現在の失敗回数 |
| `dao_cleanupExpiredOtp(tenantId)` | 24h以上経過レコードを物理削除 |

### 👮 管理者 (Admin)

| シグネチャ | 役割 |
|---|---|
| `dao_isAdmin(tenantId, email)` | bool |
| `dao_listAdmins(tenantId)` | メールアドレス配列 |

### 📧 メールログ (MailLog) — 新規

| シグネチャ | 役割 |
|---|---|
| `dao_appendMailLog(tenantId, type, recipient, subject, status)` | 1行追加。type: `otp/confirm/cancel/reminder/promotion/weekly` |
| `dao_countMailsSent(tenantId, fromDate, toDate)` | 期間内送信数（プラン上限チェック用） |

### 🎟 セッション (PropertiesService 経由・テナント横断キー)

| シグネチャ | 役割 |
|---|---|
| `dao_saveSession(token, sessionData)` | `SESSION_<token>` キーで保存。expiry 含む |
| `dao_getSession(token)` | 取得 (null=未存在 or 期限切れ) |
| `dao_deleteSession(token)` | 削除 |

### 🏢 内部ユーティリティ

| シグネチャ | 役割 |
|---|---|
| `_ensureSheet(tenantId, sheetName, headers)` | シート存在保証 + ヘッダー行追加 |
| `_ensureColumns(sheet, headers)` | 不足列を末尾に追加（migration 用） |
| `_ensureTenantIdColumn(sheet)` | A列に `tenantId` 列が無ければ挿入＋既存行を 'DEFAULT' で埋め |

---

## Tenant.gs 公開 API 仕様

| シグネチャ | Phase 1 挙動 | Phase 2 挙動 |
|---|---|---|
| `getTenantSpreadsheet(tenantId)` | `SpreadsheetApp.openById(SPREADSHEET_ID)` 固定 | 中央台帳からスプレッドシートID解決して `openById` |
| `getTenantConfig(tenantId, key, fallback)` | Config.gs の値そのまま | テナント固有設定があれば上書き |
| `listAllTenants()` | `[{tenantId:'DEFAULT', ...}]` 単一返却 | 中央台帳全件 |
| `authorizeTenantRequest(tenantId, token)` | セッション検証のみ、課金OK固定 | + 課金ステータス検証 (suspended/cancelled で 402/410) |
| `updateLastAccess(tenantId)` | no-op | 中央台帳 O列更新 |

---

## Config.gs の集約定数

```javascript
const SHEET_NAMES = {
  RESERVATION: '予約',
  SCHEDULE:    '開催日程',
  OTP:         '認証コード',
  USER_MASTER: '利用者マスタ',  // Phase 1.5
  ADMIN:       '管理者',
  MAIL_LOG:    'メールログ',    // 新規
  // Phase 2:
  // TENANTS:   '_central_tenants',     (中央管理SSのみ)
  // BILLING:   '_central_billing',
};

const STATUS = {
  RESERVATION: { CONFIRMED:'確認済', CANCELLED:'キャンセル', WAITING:'待機' },
  SCHEDULE:    { OPEN:'受付中', FULL:'満席', CLOSED:'停止' },
  TENANT:      { TRIAL:'trial', ACTIVE:'active', SUSPENDED:'suspended', CANCELLED:'cancelled' },
};

const LIMITS = {
  OTP_LENGTH: 6,
  OTP_EXPIRY_MIN: 10,
  OTP_MAX_ATTEMPTS: 5,
  OTP_LOCK_MIN: 10,
  SESSION_EXPIRY_HOURS: 1,
  DUPLICATE_WINDOW_MIN: 5,         // Phase 1.5 L2 重複検出
  REGULAR_VISIT_THRESHOLD: 10,     // 常連バッジ閾値
  MAX_RESERVATION_PER_BOOKING: 20,
};

// v2 企画書 (2026-05-27) で4プラン構成に確定
const PLAN_LIMITS = {
  free: {
    label: 'Free', price: 0,
    reservations: 20, emails: 100,
    features: ['basic']
  },
  standard: {
    label: 'Standard', price: 2980,                                  // 「LINE有料版を払わずに、LINEで予約完結」
    reservations: 200, emails: 1000,
    features: ['basic','history','reminder','csv','line_basic']
  },
  social: {
    label: 'Social', price: 9800,                                    // 「こども食堂を、地域のセーフティネットの入口に」
    reservations: 500, emails: 5000,
    features: ['basic','history','reminder','csv','line_basic',
               'ai_chat','resource_nav','donation','shortage_alert',
               'multilang','study_support','food_bank']
  },
  enterprise: {
    label: 'Enterprise', price: 19800,                               // 「行政・社協・複数食堂を、AIで束ねる」
    reservations: Infinity, emails: Infinity,
    features: ['*']
  }
};
```

---

## 重要な設計確定事項（Hideoさんの回答からの反映）

| # | 確定事項 | 実装上の対応 |
|---|---|---|
| 1 | `initializeSheets()` は **冪等な migration** | 既存4シートのA列に `tenantId` を挿入し既存行を `'DEFAULT'` で埋め、不足列を末尾追加。再実行安全。Logger に「追加列：X / 埋めた行：Y」 |
| 2 | フロント tenantId は **フォールバック方式** | URL `?tenant=` 無ければ `'DEFAULT'`。`localhost` 以外で DEFAULT 使用なら console.warn |
| 3 | `Billing.gs` は **スタブ関数** | `recordBillingEvent` / `getTenantBillingStatus` / `handleStripeWebhook(throw)`。`authorizeTenantRequest` から `getTenantBillingStatus` を呼ぶ |
| 4 | `sendMail()` ラッパー + **メールログシート** | Phase 1 から `メールログ` シートに `tenantId / 種別 / 送信日時 / 宛先 / 件名 / ステータス` 蓄積 → Phase 2 プラン設計の実数根拠 |
| 5 | ファイル名 **PascalCase 統一**・各ファイル冒頭に **責務・依存・呼出元コメント** | 上記雛形参照 |
| 6 | **段階的実装**: Step 1 = Phase 1 本体 + Phase 2 仕込み、Step 2 = Phase 1.5、Step 3 = Phase 2 本格起動 | Step 1 完了後デプロイ → 実運用フィードバック → Step 2 |

## 追加要件（Hideoさん側からの追記）

| 追加 | 実装内容 |
|---|---|
| `seedDemoData()` (Admin.gs) | 1ヶ月分の毎週土曜日程 + 5名ダミー予約 + 常連リピーター履歴。本番ガードフラグ必須 |
| `clearAllData()` (Admin.gs) | 全シートの2行目以降削除。本番ガード + 確認プロンプト必須 |
| メールログシート | プラン別上限チェックの実数根拠を Phase 1 から蓄積 |

---

## 次の手順

1. **このセクションを Hideoさんがレビュー** → 認識ズレ確認
2. OK なら **Step 1 実装着手**: 既存5ファイルのリファクタ + 新規5ファイル作成 + initializeSheets 冪等化 + tenantId 列追加 + ラッパー導入
3. Step 1 完了 → clasp push → redeploy → 動作確認
4. その後 Step 2 (Phase 1.5)、Step 3 (Phase 2 本格起動) へ

> 認識ズレあれば指摘してください。問題なければ「実装スタート」と返してください。
