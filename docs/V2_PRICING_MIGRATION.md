# v2 価格テーブル移行レポート

> 作成日: 2026-05-27
> 対応指示: 「Claude Code 向け v2 企画書 実装指示集」内 **指示1**
> v2 企画書: 価格構造 1プラン → 4プラン（Free / Standard / Social / Enterprise）

---

## 1. 修正したファイル一覧

| ファイル | 修正内容 | デプロイ反映 |
|---|---|---|
| `gas/Config.gs` | `PLAN_LIMITS` を3プラン (free/standard/pro) → 4プラン (free/standard/**social**/**enterprise**) に変更。`label` / `price` / `features` を拡張 | **v6 デプロイ済み** (2026-05-27 9:49) |
| `README.md` | 設計セクションのサンプル `PLAN_LIMITS` を v2 仕様 4プランに差替 | (ドキュメントのみ) |

## 2. 修正前 → 修正後 diff サマリ

### gas/Config.gs（コード本体）

**修正前** (3プラン・価格情報なし):
```js
var PLAN_LIMITS = {
  free:     { reservations: 30,       emails: 100,  features: ['basic'] },
  standard: { reservations: 300,      emails: 1000, features: ['basic','history','reminder','csv'] },
  pro:      { reservations: Infinity, emails: 5000, features: ['*'] }
};
```

**修正後** (4プラン・label/price/features 拡張):
```js
var PLAN_LIMITS = {
  free: {
    label: 'Free', price: 0,
    reservations: 20, emails: 100,
    features: ['basic']
  },
  standard: {
    label: 'Standard', price: 2980,
    reservations: 200, emails: 1000,
    features: ['basic','history','reminder','csv','line_basic']
  },
  social: {
    label: 'Social', price: 9800,
    reservations: 500, emails: 5000,
    features: ['basic','history','reminder','csv','line_basic',
               'ai_chat','resource_nav','donation','shortage_alert',
               'multilang','study_support','food_bank']
  },
  enterprise: {
    label: 'Enterprise', price: 19800,
    reservations: Infinity, emails: Infinity,
    features: ['*'],
    note: 'カスタム開発・複数店舗統合・行政連携API・補助金導入支援を含む'
  }
};
```

### README.md
Phase 2 設計セクション内のサンプルコードを上記と同構造に差替え（コード本体の `Config.gs` と一致）。

## 3. 旧価格表記 grep 結果（全文書スキャン）

| パターン | 検索結果 | 状態 |
|---|---|---|
| `pro:` / `'pro'` (旧プラン名) | コード本体: 0件 / README.md: 1件（修正済） | ✅ 解消 |
| `¥1,000` / `¥1000` (旧 v1 価格) | 0件 | ✅ なし |
| `pro.*3,000` (旧 v1 想定価格) | 0件 | ✅ なし |
| `monthlyPrice` (指示書サンプル形式) | 0件 | ✅ コードは `price` 命名で統一 |

## 4. 未反映の場所 / 指摘

指示書中で参照されている以下のドキュメントは、本リポジトリの `D:\data\kodomo\docs\` には存在しません（チャット内コンテンツとして提供されたもの）：

- `docs/kodomo_shokudo_business_plan_v2.md` — v2 企画書本体
- `docs/kodomo_shokudo_phase2_saas_extension.md` — Phase 2 設計書
- `docs/kodomo_shokudo_step2_instructions.md` — Step 2 指示書
- `docs/kodomo_shokudo_phase1.5_history_and_duplicate.md` — Phase 1.5 仕様
- `docs/kodomo_shokudo_step1_to_production.md` — 本番移行手順
- `docs/kodomo_shokudo_phase1.7_line_integration.md` — Phase 1.7 設計書（Claude 発行予定）

**推奨**: 上記ドキュメントを `docs/` 配下にファイルとして保存しておくと、今後の Claude Code セッション間で参照が安定します（チャットコンテキストが切れても残る）。

## 5. Step 2 / Phase 2 への影響

| 範囲 | 影響 | 対応 |
|---|---|---|
| **Step 2 (Phase 1.5)** 実装 | なし | 4プラン化は構造変更のみ、Phase 1.5 機能（履歴・重複防止・公開モード）に直接影響しない |
| **Phase 2 マルチテナント** | あり（軽微） | `authorizeTenantRequest` 内のプラン判定で `'pro'` 参照がないか要確認 → grep 済・0件 |
| **Phase 3 Stripe** | あり（大きい） | 価格 ¥2,980 / ¥9,800 / ¥19,800 を Stripe 商品として事前登録要 |
| **Phase 4 Social プラン** | あり（重要） | `features: ['ai_chat', 'resource_nav', 'donation', ...]` をゲーティングの根拠とする |
| **`Billing.gs` のスタブ** | なし | `getTenantBillingStatus()` は `plan: 'standard'` 固定で返すため、新プランへの分岐は Phase 2 で実装 |

## 6. デプロイ状態

| 環境 | バージョン | 状態 |
|---|---|---|
| GAS Web App | v6 | ✅ 4プラン PLAN_LIMITS で稼働中 |
| URL | `https://script.google.com/macros/s/AKfycbx0qYEXzMT8gOVYK0POBxo86oc2OIsW3180S2CblCoKOa6frSuba7NrFdznlNfmuvSj/exec` | 不変 |

## 7. 完了条件チェック

- [x] Config.gs が4プラン構成に修正されている
- [x] 既存ドキュメントの旧価格表記が消えている (README.md 修正済)
- [x] 移行レポートが `docs/V2_PRICING_MIGRATION.md` として存在する
- [x] 機能テスト不要（PLAN_LIMITS は Phase 2 以降で参照、現在は未使用）

## 8. 次のアクション

指示書フローに従い、以下の順で進行：

1. **Step 2 (Phase 1.5) 実装** ← 次の Claude Code 着手案件
2. ぽかぽか食堂 PRODUCTION_GUARD ON (Hideo / Step 2 完了後)
3. LINE公式アカウント開設 (Hideo / 今週中)
4. Phase 1.7 設計書発行 (Claude / 来週前半)
5. Phase 1.7 実装着手 (Claude Code / Step 2 完了後)
