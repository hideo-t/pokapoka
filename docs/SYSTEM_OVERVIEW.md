# ぽかぽか食堂 予約システム — システム概要書

> 最終更新: 2026-05-27 (Step 2 / Phase 1.5 完了時点)

## このシステムは何か

地域こども食堂向けの **オンライン予約システム**。  
保護者がスマホ/PCから予約 → 運営者が Google スプレッドシートで一元管理 → メールで自動通知。  
将来的にはマルチテナント SaaS として 100店舗運用を目指す。

| 項目 | 値 |
|---|---|
| 本番URL | <https://hideo-t.github.io/pokapoka/?tenant=DEFAULT> |
| GitHub | <https://github.com/hideo-t/pokapoka> |
| LINE 公式アカウント | `@533dgghw` |
| 想定運営コスト | 月額¥0（Phase 1段階）／ 月¥1,632〜（Workspace 移行後） |

## 主な機能

### 利用者向け
- メール OTP 認証（6桁・10分有効）
- 公開カレンダー（未認証で混雑度確認、絵文字4段階）
- 予約作成・キャンセル
- マイ予約（来場回数バッジ、過去履歴）
- 待機リスト（満席日も登録可、繰り上げメール）
- リピーター向け自動プリフィル（氏名・電話・人数・アレルギー）
- 重複予約の自動検出と対処ボタン (取消・別日選択・マイ予約)
- LINE 友だち追加 / メール問合せ

### 運営者（管理者）向け
- 管理ダッシュボード（予約数サマリ、棒グラフ、CSV）
- 予約ステータス変更（確認済 / キャンセル / 待機）
- 待機 → 確認済 への繰り上げ操作（自動通知メール）
- 利用者マスタ管理（リピーター分析、ブロックリスト）
- 整合性監査（重複・抜け・矛盾検出）
- 週間レポート自動送信（毎週月曜10時）
- 前日リマインダー自動送信（毎日18時）
- メールログ蓄積（プラン上限実数の根拠用）

## 技術構成

```
┌──────────────────────────────────────────────────────────┐
│  利用者                                                  │
│   - スマホ（LINE経由 or 直接URL）                        │
│   - PC ブラウザ                                          │
└────────────────────┬─────────────────────────────────────┘
                     │ HTTPS
┌────────────────────▼─────────────────────────────────────┐
│  GitHub Pages (フロントエンド配信)                       │
│   - index.html (シングルファイル・Vanilla JS)            │
│   - Chart.js (管理者ダッシュ用 CDN 1本のみ)              │
└────────────────────┬─────────────────────────────────────┘
                     │ fetch (POST/GET)
┌────────────────────▼─────────────────────────────────────┐
│  Google Apps Script Web App (バックエンド)               │
│   - Code.gs        ルーティング・初期化                  │
│   - Config.gs      定数（プラン・上限・シート名）        │
│   - DataAccess.gs  全DB操作の抽象化                      │
│   - Tenant.gs      テナント解決（Phase 1 は DEFAULT）    │
│   - Auth.gs        OTP 認証・セッション                  │
│   - Reservation.gs 予約 CRUD・重複防止 L1/L2/L3          │
│   - UserMaster.gs  利用者マスタ集計                      │
│   - Notify.gs      メール送信 (MailApp ラッパー)         │
│   - Admin.gs       管理者 API                            │
│   - Billing.gs     課金スタブ (Phase 2 で本実装)         │
└────────────────────┬─────────────────────────────────────┘
                     │ SpreadsheetApp.openById
┌────────────────────▼─────────────────────────────────────┐
│  Google スプレッドシート (データストア)                  │
│   ・予約        (来場日・人数・ステータス等)             │
│   ・開催日程    (受付上限・状態)                         │
│   ・利用者マスタ (リピーター集計・ブロック)              │
│   ・認証コード  (OTP 一時保存)                           │
│   ・管理者      (権限リスト)                             │
│   ・メールログ  (送信履歴 + messageId)                   │
└──────────────────────────────────────────────────────────┘
```

## データの流れ（予約作成）

```
1. 利用者: 日程選択 + 氏名・人数入力
2. フロント: createReservation API 呼出 (token 認証付)
3. GAS: LockService 取得
4. GAS: 重複チェック (L1/L2/L3)
        - L3 ブロックリスト → 拒否
        - L1 同日確認済 → 拒否（既存予約案内）
        - L2 5分以内連投 → 拒否
5. GAS: 空き判定 (dao_countReservationsFor で動的集計)
6. GAS: 予約シートに 1 行追加 (短縮UUID生成)
7. GAS: 確認メール送信 (MailApp ラッパー → メールログ記録)
8. GAS: updateUserMaster で利用者マスタ集計更新
9. GAS: LockService 解放
10. フロント: 完了画面表示
```

## セキュリティ概要

| 領域 | 対策 |
|---|---|
| 認証 | メール OTP (6桁・10分有効・5回失敗で10分ロック) |
| セッション | UUID トークン (PropertiesService に1時間保存) |
| 認可 | 全API 冒頭で `authorizeTenantRequest` (tenantId 整合性検証) |
| 二重予約 | `LockService.waitLock` で排他制御 |
| 重複登録 | L1/L2/L3 の3レベル防止 |
| 個人情報 | テナント単位でデータ分離 (Phase 2 対応済) |
| 公開API | `getPublicSchedules` は数値非開示（絵文字のみ） |
| 機密値 | LINE トークン等は PropertiesService のみ・コード/git に書かない |
| バックアップ | スプレッドシート手動コピー + 版の履歴 (Google 自動90日) |

## 関連リソース

| リソース | URL |
|---|---|
| 本番フロント | <https://hideo-t.github.io/pokapoka/?tenant=DEFAULT> |
| GitHub リポジトリ | <https://github.com/hideo-t/pokapoka> |
| GAS プロジェクト | <https://script.google.com/u/0/home/projects/1Vhj2IrWo2IhE9IUzL7PQaNEmDkcdTSNr6qhS2dlndSkehNfXW4eEj35N/edit> |
| スプレッドシート | <https://docs.google.com/spreadsheets/d/15Yw5uQejs4Y0XkKu8t_ILsB7nP-8x1ZeIda4tKGugjw/edit> |
| LINE Bot Manager | <https://manager.line.biz/account/@533dgghw> |
| 初期設定書 | [INITIAL_SETUP.md](./INITIAL_SETUP.md) |
| 運用マニュアル | [ADMIN_OPERATION_MANUAL.md](./ADMIN_OPERATION_MANUAL.md) |
| 利用者向け使い方 | [USER_GUIDE.md](./USER_GUIDE.md) |

## 開発フェーズ

| Phase | 内容 | 状態 |
|---|---|---|
| Phase 1 | 単店舗MVP（OTP・予約・通知） | ✅ 完了 |
| Phase 1.5 | 重複防止・利用者マスタ・履歴・公開カレンダー | ✅ 完了 |
| Phase 1.7 | LINE 中心UI (LIFF + Reply Bot + リッチメニュー) | 🔄 着手準備 |
| Phase 2 | マルチテナント化・中央管理SS | 設計済・未実装 |
| Phase 3 | Stripe 自動課金・セルフ申込 | 未着手 |
| Phase 4 | Social プラン (AI 相談Bot・寄付動線等) | 未着手 |
| Phase 5 | DB 移行 (Supabase)・Enterprise | 検討段階 |
