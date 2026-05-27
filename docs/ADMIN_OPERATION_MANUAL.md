# 運用マニュアル（システム担当・管理者向け）

> 日常運用・トラブル対応・月次タスクをこのファイル1枚で。

---

## 🌟 日常運用（週次〜月次）

### ① 開催日程を追加する
スプレッドシート → 「開催日程」シート → 2行目以降に追加：

| tenantId | 開催日 | 受付上限（大人）| 受付上限（子ども）| 受付状態 | 備考 |
|---|---|---|---|---|---|
| DEFAULT | 2026/07/01 | 20 | 30 | 受付中 | カレー大会 |

- **日付形式**: `YYYY/MM/DD`
- **受付状態**: `受付中` / `満席` / `停止` のいずれか
- **備考**: 利用者の予約フォーム・公開カレンダーに表示される

再デプロイ・ボタン操作は不要。次回フロント読み込み時に反映。

### ② 受付状態を変更する
- 通常の予約締切は **受付上限到達で自動で「満席」扱い**（動的集計）
- 急に休む場合: 「受付状態」列を `停止` に変更 → 利用者の予約フォームでも「うけつけしてい」と表示
- 受付再開: `受付中` に戻す

### ③ 予約一覧の確認
- 簡易版: スプレッドシート「予約」シートを直接眺める
- 管理者ビュー: 本番URL <https://hideo-t.github.io/pokapoka/?tenant=DEFAULT&screen=admin> → グラフ + 一覧 + 操作可能

### ④ ステータス変更（待機 → 確認済 への繰り上げ）
管理ダッシュボードで該当行のプルダウン:
- `確認済` / `キャンセル` / `待機` を選択 → confirm → 保存
- **待機 → 確認済** に変更すると、対象者に **自動で繰り上げメール** 送信
- メール内容: `「キャンセル待ちから繰り上がりました」`

### ⑤ CSV ダウンロード
管理ダッシュボード → 「📥 CSVダウンロード」  
BOM 付き UTF-8 で出力 → Excel で開いても文字化けしない。  
週次バックアップや会計監査に活用。

### ⑥ 利用者マスタの確認
スプレッドシート「利用者マスタ」シートで以下が見える：
- 通算来場回数 / キャンセル回数 / 初回来場日 / 最終来場日
- アレルギー履歴（累積）
- ブロックフラグ（K列）
- 備考（運営メモ・管理者だけが書く欄）

### ⑦ ブロックリスト管理（迷惑行為対応）
「利用者マスタ」シートの該当行で：
- **K列「ブロックフラグ」を TRUE に変更** → 即座にその利用者の新規予約が L3 ブロック
- 備考列に理由を記録（推奨）

API でも操作可能（curl 例）:
```bash
curl -sL --post301 --post302 --post303 -X POST \
  "https://script.google.com/macros/s/AKfycbx0qYEXzMT8gOVYK0POBxo86oc2OIsW3180S2CblCoKOa6frSuba7NrFdznlNfmuvSj/exec" \
  -H 'Content-Type: text/plain;charset=utf-8' \
  -d '{"action":"adminBlockUser","tenant":"DEFAULT","token":"<管理者token>","email":"target@example.com","blocked":true,"adminNote":"迷惑行為のためブロック"}'
```

---

## 📅 月次タスク

### 第1月曜
- 「メールログ」シートを開き、先月の送信数を確認
- 100通/日に近づいているなら Workspace 移行を検討

### 毎週月曜10時（自動）
- `sendWeeklyReport` が管理者宛にメール送信
- 新規予約・キャンセル・今後7日間の予約サマリが届く

### 毎月1日
- スプレッドシートを **手動バックアップ** (詳細: [BACKUP.md](./BACKUP.md))
  - ファイル → コピーを作成 → `予約データ_backup_YYYY-MM-DD`

---

## 🆘 トラブル対応

### 「予約が同日に2件登録されている」
重複防止（L1）が効くはずだが、稀に：
- 利用者マスタの更新タイミングでズレ
- L1 が「確認済 or 待機」だけ判定なので「キャンセル」を経由した再予約は通る（仕様）

→ 「予約」シートの該当行を直接削除 or キャンセルステータスに変更。

### 「同じ人がL1ブロックされて困っている」
ユーザーが過去予約をキャンセルしていない場合に発生。  
画面の「❌ きそんよやくをキャンセルして あたらしくよやく」ボタンで自動キャンセル→再予約できる。  
それでも解決しない場合は、管理ダッシュからステータス変更でサポート。

### 「OTP メールが届かない」
1. 利用者の迷惑メールフォルダ確認を案内
2. GAS 「実行ログ」で `sendOTPMail` のエラー有無確認
3. 1日のメール送信枠 (Gmail 無料 = 100通/日) に到達していないか確認
4. メールアドレスが正しいか確認 → 必要なら手動で別アドレスで再送信案内

### 「予約が反映されない」
1. ブラウザで本番URLを **強制リロード** (Ctrl+Shift+R)
2. GAS Web App の **デプロイ状態確認** (`clasp deployments`)
3. 「予約」シート直接確認 → ある場合はフロント表示バグ
4. ない場合は GAS の「実行ログ」でエラー確認

### 「整合性チェックで警告が出る」
GAS エディタで `auditPayPeriodIntegrity` 関数を ▶ 実行（※こちらは tesou3 用、本システムでは未実装）  
本システムでの整合性確認は管理ダッシュの予約一覧で目視。

### 「フロントが古いまま」
GitHub Pages のキャッシュが残っている可能性。
- 利用者へ「強制リロード (Ctrl+Shift+R)」を案内
- URL末尾に `?v=1` 等のクエリを付けて開いてもらう

---

## 🛠 デプロイ管理

### 現在のデプロイ一覧
```bash
cd D:\data\kodomo
clasp deployments
```

### コード更新フロー
```bash
# 1. コード編集後
clasp push -f

# 2. 既存 Web App の URL を維持したまま新コードに更新
clasp redeploy <deploymentId> -d "変更内容のメモ"
```

### ロールバック手順
```bash
clasp deployments        # 過去デプロイ一覧
clasp redeploy <旧deploymentId> -d "rollback to vN"
```

`-d` は説明文。後で履歴追跡しやすいよう必ず書く。

---

## 🔐 セキュリティ運用

### 管理者を追加
スプレッドシート「管理者」シートに行追加:
| tenantId | メールアドレス | 権限 |
|---|---|---|
| DEFAULT | new-admin@example.com | admin |

`viewer` 権限は将来用（現状は `admin` でのみ管理ダッシュ表示可）。

### LINE 秘匿情報の管理
**Channel Access Token / Channel Secret は絶対に：**
- コード本体 (`gas/*.gs`) に書かない
- `docs/` に書かない（GitHub 公開される）
- memory / チャットに残さない

GAS スクリプトプロパティ専用。詳細: [STEP3_LINE_SETUP.md](./STEP3_LINE_SETUP.md)

### トークン再生成（漏洩時の対応）
1. LINE Developers Console → 該当チャネル → アクセストークン **再発行**
2. 旧トークンを **無効化**
3. GAS スクリプトプロパティの `LINE_CHANNEL_ACCESS_TOKEN` を新トークンに更新
4. `checkLineCredentials()` で確認

### PRODUCTION_GUARD
本番運用中は `Config.gs` の以下を必ず `true`：
```javascript
var PRODUCTION_GUARD = true;
```
これで `clearAllData` / `seedDemoData` / `resetTestData` の事故実行が拒否される。

開発・テスト時のみ `false` に戻し、終わったら必ず `true` に戻す。

---

## 📂 ファイル構成（運用者が触る範囲）

| ファイル | いつ触るか |
|---|---|
| スプレッドシート「開催日程」 | 月1〜2回（新しい開催日追加） |
| スプレッドシート「管理者」 | 管理者の入退時 |
| スプレッドシート「利用者マスタ」 | ブロックリスト操作・運営メモ追記 |
| `Config.gs` の `PRODUCTION_GUARD` | 本番投入・テスト切替時 |
| GAS スクリプトプロパティ | LINE 連携情報の追加・再生成時 |

その他のファイル (`*.gs`, `index.html`) は通常触らない。改修時のみ clasp 経由で更新。

---

## 📞 困った時の連絡先

| 連絡先 | 用途 |
|---|---|
| システム担当: hdxrope@gmail.com | コード関連・デプロイ問題 |
| LINE Developers サポート | LINE API 障害 |
| Google Workspace サポート | GAS 制限・送信枠相談 |

---

## 関連ドキュメント

- [INITIAL_SETUP.md](./INITIAL_SETUP.md) - 0→1 セットアップ
- [SYSTEM_OVERVIEW.md](./SYSTEM_OVERVIEW.md) - 仕組みの理解
- [USER_GUIDE.md](./USER_GUIDE.md) - 利用者向けマニュアル（運営者として読んでおく）
- [BACKUP.md](./BACKUP.md) - バックアップ運用
- [STEP3_LINE_SETUP.md](./STEP3_LINE_SETUP.md) - LINE 連携設定
- [STEP2_VERIFICATION.md](./STEP2_VERIFICATION.md) - リリース前の動作確認
