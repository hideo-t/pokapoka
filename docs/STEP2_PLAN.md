# Step 2 (Phase 1.5) 実装計画

> 共通ルール: 着手前に方針を提示 → Hideoさんの承認後に本実装。
> v2 企画書 / Phase 1.5 機能仕様 / Step 2 指示書（チャット内提供）を統合した着地点。

---

## スコープ

| # | 機能 | 影響範囲 |
|---|---|---|
| 1 | 利用者マスタの自動更新 | Reservation.gs / Admin.gs / UserMaster.gs |
| 2 | 重複登録防止 L1/L2/L3 | Reservation.gs / Frontend |
| 3 | 履歴表示UI（来場回数・初回/前回・アレルギー履歴）| Frontend / UserMaster.gs |
| 4 | 紹介元入力欄 | Frontend / Reservation.gs |
| 5 | 公開カレンダー（未認証・絵文字3段階） | Reservation.gs / Code.gs / Frontend |
| 6 | リピーター画面分岐 | Frontend |

## 設計判断（実装前に確定）

### 1. 利用者マスタの自動更新
- `createReservation` / `cancelReservation` / `adminUpdateStatus` の **return 直前** で `updateUserMaster(tenantId, email)` を必ず呼ぶ
- 失敗してもメイン処理を止めない（try/catch + Logger.log）
- Step 1 で実装済みの `updateUserMaster()` をそのまま使う（UserMaster.gs）

### 2. 重複登録防止 L1/L2/L3
**LockService 内で実行**（チェック後のレース条件防止）。

| Level | 条件 | 応答 |
|---|---|---|
| **L1** | 同一メール × 同一日付 × 確認済/待機 状態の予約あり | `{ok:false, duplicate:'L1', existingReservationId, message:"..."}` |
| **L2** | 同一メールから 5 分以内に同じ日付の予約申込 | `{ok:false, duplicate:'L2', message:"..."}` |
| **L3** | 利用者マスタの blocked = true | `{ok:false, duplicate:'L3', message:"お電話でお問合せください"}` |

新規関数: `_checkDuplicate(tenantId, email, date)` (Reservation.gs 内 private)
- L3 を最初に判定 → L1 → L2 の順
- 既存の `dao_listReservations` を `filters.email + filters.createdAfter` で活用

### 3. 履歴 UI（マイ予約画面の刷新）
バックエンドAPI: `getUserHistory(tenantId, token, email)` (UserMaster.gs に既存)
- 返す: `{master:{totalVisits, firstVisit, lastVisit, allergies, isRegular}, reservations:[...]}`

フロント UI:
- マイ予約画面の冒頭に「来場 N 回目 🎉」バッジ
- 初回来場日 / 前回来場日 / 登録アレルギー履歴
- 「これからの予約」(date >= today)・「これまでの利用履歴」(date < today) で分離
- **直近5件表示 + 「もっと見る」遅延ロード** (指示書要件)

### 4. 紹介元入力欄
- 予約フォームに `<select>` 追加: `口コミ / SNS / チラシ / その他 / 選択しない`
- 利用者マスタの **通算来場回数=0** (初回) の場合のみ強調表示・必須化（リピーターは省略可能）
- バックエンドは `data.source` を受け取り、予約シート O列「紹介元」に保存

### 5. 公開カレンダー（未認証OK）
**新規GAS関数**: `getPublicSchedules(tenantId)` (Reservation.gs)
- 認証**不要**（`authorizeTenantRequest` 呼ばない）
- 残数は **絶対に数字で返さない** ← プライバシー設計
- 4段階の絵文字で混雑度を表現:
  - 🟢 余裕あり (満室率 < 50%)
  - 🟡 残りわずか (50% ≤ 満室率 < 80%)
  - 🟠 ほぼ満席 (80% ≤ 満室率 < 100%)
  - 🔴 満席 (100%)
- 返却: `{date, status, indicator, remark, isFull, isClosed}` (具体数値なし)

**Code.gs ルーター**: doGet に `getPublicSchedules` 追加（token 不要）

**フロント**: ログイン画面に「📅 にっていをみる（ログイン不要）」リンク追加 → 公開カレンダー画面で日程確認 → 予約したくなったらメール認証へ

### 6. リピーター画面分岐
v2 案C ベース。`verifyOTP` の戻り値に `isFirstTime` を追加せず、認証後に `getUserHistory` を1回呼ぶことで判定（追加APIなし）。

- 認証直後にマイページ予約フォーム上部に表示:
  - 初回 → `🌸 はじめてのご利用、ようこそ！` + 紹介元欄を強調
  - リピーター → `🎉 〇〇さん、来場 N 回目 ／ 前回: YYYY/MM/DD ／ アレルギー: ...`
  - リピーター画面に「**前回と同じ条件で予約**」ボタン (人数・アレルギーをプリフィル)

---

## データモデル変更

| シート | 変更内容 |
|---|---|
| **予約** | 既存列 O「紹介元」にデータが書き込まれるようになる（列追加なし）|
| **利用者マスタ** | `updateUserMaster()` 経由で初めて行が作成・更新される（列追加なし）|

スキーマ変更ゼロ → migration 不要 → **データバックアップは念のため取るが、危険度は低い**。

---

## 影響範囲（ファイル別）

| ファイル | 変更タイプ | 主な追加 |
|---|---|---|
| `Reservation.gs` | 拡張 | `_checkDuplicate` / `getPublicSchedules` / source 受取 / updateUserMaster 呼出 |
| `Admin.gs` | 軽微 | `adminUpdateStatus` 末尾に updateUserMaster 呼出 / `adminListUserMasters` 追加 |
| `Code.gs` | ルーター追加 | doGet に `getPublicSchedules` / `adminListUserMasters` |
| `UserMaster.gs` | なし | 既存 `getUserHistory` / `updateUserMaster` をそのまま使う |
| `frontend/index.html` | 大幅 | 公開カレンダー画面追加 / マイ予約刷新 / 紹介元欄 / 重複エラー UI / リピーターバッジ |

---

## 受入基準（STEP2_VERIFICATION.md で詳細化予定）

- [ ] 初回ユーザーが予約すると利用者マスタに自動作成される
- [ ] 同じユーザーが同日に2回予約しようとすると L1 ブロック
- [ ] 5分以内に同じ日付で2回送信すると L2 ブロック
- [ ] 管理者が利用者マスタの `blocked` を true にすると L3 ブロック
- [ ] マイ予約画面で「来場 N 回目」バッジ表示
- [ ] 初回ユーザーのみ紹介元欄が強調表示
- [ ] 未認証で公開カレンダーが見られる（絵文字3段階）
- [ ] 公開カレンダーに数字（残席数・予約者数）が一切出ない
- [ ] リピーターは「前回と同じ条件で予約」ボタンで人数プリフィル

---

## リスクと対応

| リスク | 対応 |
|---|---|
| `updateUserMaster` 失敗で予約処理ロールバックされる | try/catch で Logger.log のみ・メイン処理は継続 |
| 重複チェックの誤検知 | L1 で「キャンセル/取り直し」ボタンを案内、ユーザー判断に委ねる |
| 公開カレンダー悪用（混雑度から特定推測） | 4段階の粗い指標 + 残数の具体値を絶対出さない |
| 紹介元欄を強制化して離脱率上昇 | リピーターは省略可・初回も「選択しない」OK |
| `seedDemoData` の利用者マスタ整合性 | seedDemoData 末尾で `rebuildAllUserMasters()` を呼ぶ |

---

## 実装順序（提案）

1. **バックエンド先行** (15分): Reservation.gs / Admin.gs / Code.gs
2. **clasp push → redeploy v7**
3. **動作確認** (curl で getPublicSchedules / 重複チェック検証)
4. **フロントエンド** (30分): 公開カレンダー / マイ予約刷新 / 紹介元 / 重複UI / リピーターバッジ
5. **動作確認** (ブラウザで全フロー)
6. **STEP2_VERIFICATION.md 作成**

---

## 承認確認

以下に問題なければ「**実装開始**」と返してください。

- データモデル変更ゼロ・migration 不要
- 重複チェックは LockService 内で実行
- 公開カレンダーに数値を出さない（4段階絵文字のみ）
- 紹介元欄は初回強調・リピーター省略可
- L3 ブロックは利用者マスタの blocked フラグ参照

修正希望があれば指摘してください。承認後にバックエンドから着手します。
