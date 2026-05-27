# こども食堂予約システム — Phase 1.7 設計書（LINE中心UI）

> Phase 1.5（重複防止・履歴）完了後、Phase 2（マルチテナント）の前に実装する独立フェーズ。  
> LINEを主導線化し、メール認証は副ルートとして残すハイブリッド設計。

---

## 目的

1. 運営者の現行ワークフロー（LINE中心）と連続性を保ちながら、プライバシー問題を解決する
2. LINE Messaging API の Reply API（無料・無制限）を最大活用し、月額¥0のLINE運用を実現
3. LIFF（LINE Front-end Framework）でメール認証スキップを可能にし、シニア層のハードルを下げる
4. v2 企画書の Standard プラン「LINE Bot基本」機能を満たす

---

## 機能仕様

### F1. LINE公式アカウント連携

- ぽかぽか食堂専用 LINE公式アカウント（フリープラン、月¥0）
- 友だち追加 = 予約システム利用開始
- Webhook URL に既存 GAS WebApp URL を設定

### F2. リッチメニュー4ボタン

```
┌─────────────────────┬─────────────────────┐
│ 📅 予約する         │ 📋 マイ予約          │
│ → LIFF で予約フォーム│ → LIFF で履歴・予定   │
├─────────────────────┼─────────────────────┤
│ ❌ キャンセル        │ ❓ お問い合わせ      │
│ → 対話形式で処理    │ → 管理者通知 + Reply │
└─────────────────────┴─────────────────────┘
```

### F3. LIFF統合

既存フロントエンド（`frontend/index.html`）を LIFF アプリとして登録する。  
LINE内ブラウザで開かれた時、`liff.getProfile()` で LINE userId / displayName を取得 → 自動で本人確認、メール認証スキップ。

```javascript
// frontend/index.html 冒頭に追加
liff.init({ liffId: LIFF_ID }).then(async () => {
  if (liff.isInClient()) {
    const profile = await liff.getProfile();
    // LINE ID で既存ユーザー確認 → セッショントークン即発行
    const result = await callApi('lineLogin', {
      lineUserId: profile.userId,
      displayName: profile.displayName,
      tenantId: TENANT_ID
    });
    if (result.success) {
      saveSession(result.token);
      goToScreen('reservation-form');
    }
  }
});
```

### F4. Reply Bot

ユーザー発信メッセージに対する自動応答。**Reply API のみ使用、Push API は使わない**（無料無制限維持）。

| キーワード | Reply内容 |
|---|---|
| 予約 / 予約したい / よやく | 「下のリッチメニューから📅予約するをタップしてください 🍙」+ LIFF URL |
| マイ予約 / 確認 / かくにん | 「下のリッチメニューから📋マイ予約をタップしてください」 |
| キャンセル / きゃんせる | 「予約番号を教えてください（例: A1B2C3D4）」→ 返信待ち |
| 次回 / いつ | 次回開催日3件を整形して返信 |
| ヘルプ / help | 使い方ガイドを返信 |
| その他 | 「すみません、わからない言葉でした 🙏 リッチメニューから操作してください」 |

### F5. LINE ID と email の紐付け

利用者マスタに `line_user_id` 列を追加（既存設計に1列追加）。

| email | line_user_id | 認証方法 |
|---|---|---|
| user@example.com | U1234... | 両方使える |
| user2@example.com | (空) | メールのみ |
| (空) | U5678... | LINEのみ |

### F6. メール認証の保持

LINE未使用層のために、既存メール認証フローは完全に残す。フロントの起動時判定：

```javascript
if (liff.isInClient()) {
  // LINE内 → LINE認証
} else {
  // 通常ブラウザ → メール認証
}
```

---

## 技術構成

### 新規ファイル：`LineBot.gs`

```javascript
/**
 * LineBot.gs
 * 責務: LINE Messaging API の Webhook 処理、Reply 生成、LIFF認証ブリッジ
 * 依存: Config.gs, DataAccess.gs, Tenant.gs, Auth.gs, Reservation.gs, UserMaster.gs
 * 呼出元: Code.gs（doPost ルーター経由、action='lineWebhook'）
 * Phase 2 移行時: tenantId 解決（LINE Bot URL に tenantId を含めるか、チャネル別に分岐）
 */
```

主な関数：

| 関数 | 役割 |
|---|---|
| `handleLineWebhook(tenantId, events)` | Webhook イベント配列を処理 |
| `handleMessageEvent(tenantId, event)` | テキストメッセージ受信時の処理 |
| `handlePostbackEvent(tenantId, event)` | ボタンタップ等の処理 |
| `handleFollowEvent(tenantId, event)` | 友だち追加時のあいさつ |
| `verifyLineSignature(body, signature)` | LINE署名検証 |
| `replyMessage(replyToken, messages)` | Reply API 呼び出し |
| `lineLogin(tenantId, lineUserId, displayName)` | LIFF認証 → セッショントークン発行 |
| `getActiveCancelDialog(lineUserId)` | キャンセル対話の状態管理 |

### Config.gs への追加

```javascript
// LINE関連の定数（トークンは PropertiesService に保管）
const LINE_CONFIG = {
  REPLY_API_URL: 'https://api.line.me/v2/bot/message/reply',
  PROFILE_API_URL: 'https://api.line.me/v2/bot/profile/',
  RICHMENU_API_URL: 'https://api.line.me/v2/bot/richmenu',
  // 認証情報は PropertiesService.getScriptProperties() で管理：
  //   - LINE_CHANNEL_ACCESS_TOKEN
  //   - LINE_CHANNEL_SECRET
  //   - LIFF_ID
};
```

### Code.gs への追加

`doPost` ルーターに新ルート追加：

```javascript
const routes = {
  // 既存（省略）
  
  // Phase 1.7 追加
  'lineWebhook': () => {
    // LINE からは action 形式ではなく直接 events 配列が来る
    const tenantId = e.parameter.tenant || 'DEFAULT';
    const body = e.postData.contents;
    const signature = e.parameter.signature || e.headers?.['x-line-signature'];
    
    if (!verifyLineSignature(body, signature)) {
      return jsonResponse({ success: false, message: 'invalid signature' });
    }
    
    const data = JSON.parse(body);
    return handleLineWebhook(tenantId, data.events);
  },
  'lineLogin': () => lineLogin(params.tenantId, params.lineUserId, params.displayName),
};
```

ただし、LINE Webhook は GET パラメータでなく POST body 全体が JSON なので、`doPost` の冒頭で LINE 由来か判定する必要あり：

```javascript
function doPost(e) {
  const body = e.postData.contents;
  const data = JSON.parse(body);
  
  // LINE Webhook は events 配列を持つ
  if (data.events && Array.isArray(data.events)) {
    return handleLineWebhook(e.parameter.tenant || 'DEFAULT', data.events, body, e);
  }
  
  // 既存のアクションベース API
  const action = data.action;
  // ... 既存ルーター
}
```

### データ構造変更

#### 利用者マスタシート（1列追加）

| 列 | 既存 | 追加 |
|---|---|---|
| A〜J（既存） | tenantId / email / 氏名 / 電話 / 通算来場 / キャンセル数 / 初回来場日 / 最終来場日 / アレルギー / 備考 / ブロック | |
| L | | **line_user_id**（LINE userId、空文字可） |
| M | | **line_display_name**（LINE 表示名、参考用） |
| N | | **line_linked_at**（紐付け日時） |

#### 認証セッションへの拡張

セッションデータに `authMethod` フィールドを追加：

```javascript
sessionData = {
  email: 'user@example.com',
  tenantId: 'DEFAULT',
  authMethod: 'line',  // 'email' or 'line' or 'both'
  lineUserId: 'U1234...',  // line認証時のみ
  expiry: 1234567890000
};
```

---

## Webhook 処理フロー

### メッセージ受信フロー

```
LINE → POST /exec?tenant=DEFAULT
  ↓
doPost(e)
  ├─ events 配列を検出 → handleLineWebhook()
  ↓
events.forEach(event)
  ├─ event.type === 'message' → handleMessageEvent()
  │   ├─ event.message.text を解析
  │   ├─ キーワード判定 → replyMessage()
  │   └─ ユーザーがキャンセル対話中なら → 予約番号として処理
  ├─ event.type === 'postback' → handlePostbackEvent()
  ├─ event.type === 'follow' → handleFollowEvent()
  └─ event.type === 'unfollow' → 利用者マスタから line_user_id 削除（任意）
```

### LIFF認証フロー

```
ユーザーがリッチメニュー「📅 予約する」をタップ
  ↓
LIFF URL（https://kodomo.er-biru.net/?liff=1）を開く
  ↓
frontend/index.html が読み込まれる
  ├─ liff.init() 実行
  ├─ liff.isInClient() === true
  ├─ liff.getProfile() で userId 取得
  └─ POST /exec { action: 'lineLogin', lineUserId, displayName, tenantId }
  ↓
lineLogin(tenantId, lineUserId, displayName)
  ├─ 利用者マスタから line_user_id で検索
  ├─ 存在すれば → セッショントークン発行 → 認証済み状態へ
  ├─ 存在しなければ → 新規ユーザー扱い、email 任意入力画面へ
  └─ メール任意紐付け → 既存ユーザーとマージ可能
```

### キャンセル対話フロー

```
ユーザー「キャンセル」と送信
  ↓
Bot Reply「予約番号を教えてください（例: A1B2C3D4）」
  ↓
ScriptProperties に { lineUserId: 'awaiting_cancel_id' } を保存（10分有効）
  ↓
ユーザー「A1B2C3D4」と送信
  ↓
ScriptProperties で対話状態を検出 → cancelReservation(tenantId, token, 'A1B2C3D4')
  ├─ 成功 → Reply「キャンセルしました ✓ 予約番号: A1B2C3D4」
  ├─ 該当なし → Reply「予約番号が見つかりません。マイ予約からも確認できます」
  └─ 他人の予約 → Reply「ご本人の予約のみキャンセル可能です」
```

---

## リッチメニュー実装

### 構成データ（JSON）

```json
{
  "size": { "width": 2500, "height": 1686 },
  "selected": true,
  "name": "kodomo-main-menu",
  "chatBarText": "メニュー",
  "areas": [
    {
      "bounds": { "x": 0, "y": 0, "width": 1250, "height": 843 },
      "action": {
        "type": "uri",
        "label": "予約する",
        "uri": "https://liff.line.me/{LIFF_ID}?screen=reservation"
      }
    },
    {
      "bounds": { "x": 1250, "y": 0, "width": 1250, "height": 843 },
      "action": {
        "type": "uri",
        "label": "マイ予約",
        "uri": "https://liff.line.me/{LIFF_ID}?screen=mypage"
      }
    },
    {
      "bounds": { "x": 0, "y": 843, "width": 1250, "height": 843 },
      "action": {
        "type": "message",
        "label": "キャンセル",
        "text": "キャンセル"
      }
    },
    {
      "bounds": { "x": 1250, "y": 843, "width": 1250, "height": 843 },
      "action": {
        "type": "message",
        "label": "お問い合わせ",
        "text": "お問い合わせ"
      }
    }
  ]
}
```

### デプロイスクリプト（GAS）

`LineBot.gs` に管理用関数：

```javascript
/**
 * リッチメニュー作成・適用（初回1回 + 変更時のみ実行）
 */
function setupRichMenu() {
  const token = PropertiesService.getScriptProperties().getProperty('LINE_CHANNEL_ACCESS_TOKEN');
  const liffId = PropertiesService.getScriptProperties().getProperty('LIFF_ID');
  
  // 1. リッチメニュー作成
  const menuData = { /* 上記 JSON */ };
  // ${LIFF_ID} を実値に置換
  // ...
  
  const createRes = UrlFetchApp.fetch('https://api.line.me/v2/bot/richmenu', {
    method: 'post',
    contentType: 'application/json',
    headers: { 'Authorization': 'Bearer ' + token },
    payload: JSON.stringify(menuData)
  });
  
  const richMenuId = JSON.parse(createRes.getContentText()).richMenuId;
  
  // 2. リッチメニュー画像アップロード（事前にDriveに配置した画像を取得）
  const imageBlob = DriveApp.getFileById(RICHMENU_IMAGE_FILE_ID).getBlob();
  UrlFetchApp.fetch(`https://api-data.line.me/v2/bot/richmenu/${richMenuId}/content`, {
    method: 'post',
    contentType: 'image/png',
    headers: { 'Authorization': 'Bearer ' + token },
    payload: imageBlob.getBytes()
  });
  
  // 3. デフォルトリッチメニューに設定
  UrlFetchApp.fetch(`https://api.line.me/v2/bot/user/all/richmenu/${richMenuId}`, {
    method: 'post',
    headers: { 'Authorization': 'Bearer ' + token }
  });
  
  Logger.log(`Rich menu created and applied: ${richMenuId}`);
}
```

リッチメニュー画像（2500×1686）は事前にデザインして Google Drive に置く。  
仮置きの画像でも動作するので、最初はテキストだけでOK（後でデザイン差し替え）。

---

## メールログへの記録拡張

LINE通信も `メールログ` シートに記録（種別を拡張）：

| 種別 | 既存 | 追加 |
|---|---|---|
| otp / confirm / cancel / reminder / promotion / weekly | ✓ | |
| **line_reply** | | LINE Reply送信時 |
| **line_login** | | LIFF認証成功時 |

これによりLINE通信量も Phase 2 のプラン上限管理時にデータとして使える。

---

## セキュリティ要件

| 項目 | 対策 |
|---|---|
| Webhook 署名検証 | LINE Channel Secret で X-Line-Signature を検証 |
| LIFF 認証検証 | アクセストークンの妥当性確認（任意・推奨） |
| LINE userId なりすまし | LIFF経由のみ受け入れ、直接の lineLogin API は内部用 |
| トークン漏洩 | Channel Access Token は PropertiesService、コードに置かない |
| Replay攻撃 | LINE側で timestamp ±5分以内チェック |
| クロステナント | tenantId と LINE チャネルの紐付けを中央台帳で管理（Phase 2） |

---

## 動作確認チェックリスト（Phase 1.7 完了基準）

実装完了時に `docs/STEP_PHASE1.7_VERIFICATION.md` として以下を発行：

### O. LINE基本連携
- [ ] LINE公式アカウントを友だち追加 → あいさつメッセージ受信
- [ ] リッチメニュー4ボタンが表示される
- [ ] 「予約」とメッセージ送信 → LIFF URL がReply される
- [ ] 「キャンセル」→ 予約番号を聞かれる対話開始
- [ ] 「次回」→ 開催日3件が整形されて返ってくる
- [ ] 「ヘルプ」→ 使い方ガイドが返ってくる
- [ ] 意味不明な文字列 → 親切なエラー返信

### P. LIFF認証
- [ ] LINE内で「📅 予約する」タップ → メール認証スキップで予約フォームへ
- [ ] 初回ユーザーは email 任意入力画面が表示される
- [ ] 既存メールユーザーが LINE 紐付け → 利用者マスタの line_user_id 列に保存
- [ ] LINE 外ブラウザでアクセス → 既存メール認証フローが動作

### Q. データ整合性
- [ ] 利用者マスタに line_user_id / line_display_name / line_linked_at 列が追加されている
- [ ] LINE経由で作成した予約が、メール経由予約と同じ「予約」シートに記録される
- [ ] LINEログイン時に「メールログ」に type=line_login で記録される
- [ ] LINE Reply 送信時に type=line_reply で記録される

### R. リグレッション
- [ ] Step 1（60項目）すべて再実行 → 全て継続通過
- [ ] Step 2（Phase 1.5）の機能（重複防止、履歴、公開モード）も継続動作

### S. セキュリティ
- [ ] 無効な署名でWebhook叩く → 401相当のエラー
- [ ] 他人の LINE userId でログイン試行 → 拒否される
- [ ] Channel Access Token がコードに残っていないこと（grep確認）

### T. メール送信枠への影響
- [ ] LINE経由予約は OTPメール送信ゼロ（LIFF認証スキップ確認）
- [ ] メール認証経由は従来通り送信
- [ ] 月間メール送信量が LINE導入前と比べて減少していること

---

## Claude Code 向け実装指示（コピペ用）

```
docs/kodomo_shokudo_phase1.7_line_integration.md を読んで、
Phase 1.7（LINE中心UI）を実装してください。

【前提条件】
- Phase 1.5（Step 2）の動作確認が完了していること
- Hideoさんが以下を準備済みであること：
  - LINE公式アカウント開設
  - LINE Developers でチャネル作成、アクセストークン取得
  - LIFF アプリ作成、LIFF ID 取得
  - これらが PropertiesService に登録済み
    - LINE_CHANNEL_ACCESS_TOKEN
    - LINE_CHANNEL_SECRET
    - LIFF_ID

【実装範囲】
1. LineBot.gs 新規作成（全関数）
2. Config.gs に LINE_CONFIG 定数追加
3. Code.gs の doPost を LINE Webhook 対応に拡張
4. 利用者マスタシートに line_user_id / line_display_name / line_linked_at 列追加
5. DataAccess.gs に LINE 紐付け関連のラッパー追加
6. frontend/index.html に LIFF SDK 統合 + LIFF 起動判定
7. リッチメニュー作成スクリプト setupRichMenu() 実装
8. セッションデータに authMethod, lineUserId フィールド追加
9. メールログの種別に line_reply, line_login 追加

【遵守すべき設計原則】
- v2 企画書（kodomo_shokudo_business_plan_v2.md）を絶対基準とする
- Phase 1 / 1.5 で確立した tenantId、ラッパー、DataAccess 設計を維持
- LINE関連処理は Reply API ベース（無料無制限）、Push API は使わない
- アクセストークン等の認証情報はコードに置かず PropertiesService 経由
- 既存メール認証フローへの影響ゼロ（リグレッションなし）

【着手前に必ず提示】
- 設計差分の README 提案
- 利用者マスタ列追加の migration スクリプト（既存データを壊さない冪等処理）
- frontend/index.html の変更前後 diff サマリ

Hideoさんの承認後に本実装開始。

【完了条件】
- docs/STEP_PHASE1.7_VERIFICATION.md として28項目のチェックリスト発行
- Step 1 / Step 2 のリグレッションがゼロであること
- LINE経由・メール経由の両方で予約完了まで動作すること

【共通ルール】
- 着手前に方針提示 → Hideo承認 → 実装
- 既存機能のリグレッション禁止
- すべての変更を論理的単位で git commit
- v2 企画書の方針に反する実装は禁止
```
