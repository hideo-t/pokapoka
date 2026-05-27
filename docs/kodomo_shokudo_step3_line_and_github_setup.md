# Step 3：GitHub Pages 公開 + LINE 公式アカウント開設 — Hideoさん作業手順書

> 上から順に実施。所要時間：合計 約2〜3時間。  
> 完了するとぽかぽか食堂が「kodomo.er-biru.net」で公開され、LINE 友だち追加で予約可能になる。

---

# Part A: GitHub Pages で本番公開（45分）

## A-1. .gitignore で機密ファイルを除外（5分）

`D:\data\kodomo\.gitignore` を作成：

```
# GAS バックエンドは公開しない
gas/
.clasp.json
.claspignore
appsscript.json

# Claude Code 作業ファイル
.claude/
*.log
.DS_Store
Thumbs.db

# ローカル設定
.env
.env.local

# docs内の検証メモは含めるが、機密扱いのものは除外
docs/PRIVATE_*.md
```

## A-2. フロントエンドの機密チェック（5分）

PowerShell で実行：

```powershell
cd D:\data\kodomo\frontend
# 管理者メールが混入していないか
Select-String -Path "index.html" -Pattern "hdxrope" 
# SPREADSHEET_ID が混入していないか
Select-String -Path "index.html" -Pattern "SPREADSHEET_ID|1-mV-l"
# 何か謎のトークン文字列がないか
Select-String -Path "index.html" -Pattern "AKfyc[a-zA-Z0-9_-]{30,}"
```

- GAS WebApp URL（`AKfyc...`）は出てきてOK（公開されても認証で守られている）
- 管理者メール・SPREADSHEET_ID が出たら、Config.gs 側に分離する処理を Claude Code に依頼してから次へ

## A-3. リポジトリ作成 & GitHub プッシュ（10分）

```bash
cd D:\data\kodomo

git init
git add frontend/ docs/ README.md .gitignore
git commit -m "initial: kodomo-shokudo frontend"

# GitHub に公開リポジトリ作成 & プッシュ
gh repo create kodomo-shokudo --public --source=. --push --description "こども食堂予約システム"
```

完了したら `https://github.com/hideo-t/kodomo-shokudo` でリポジトリが見えるはず。

## A-4. GitHub Pages 有効化（5分）

```bash
gh repo edit --enable-pages --pages-branch main --pages-path /frontend
```

数分後に `https://hideo-t.github.io/kodomo-shokudo/` でアクセス可能になる。  
**この時点でテストアクセスして動作確認しておくこと。** GAS API は CORS で呼べるはず。

## A-5. カスタムドメイン設定（10分）

### Cloudflare 側

1. Cloudflare ダッシュボード → `er-biru.net` を選択
2. DNS → Add record
3. 設定：
   - **Type**: CNAME
   - **Name**: `kodomo`
   - **Target**: `hideo-t.github.io`
   - **Proxy status**: 🟠 Proxied
   - **TTL**: Auto

### GitHub 側

```bash
cd D:\data\kodomo\frontend
echo "kodomo.er-biru.net" > CNAME

cd D:\data\kodomo
git add frontend/CNAME
git commit -m "add custom domain: kodomo.er-biru.net"
git push
```

### 確認

数分後（最大15分）、`https://kodomo.er-biru.net/?tenant=DEFAULT` でアクセス可能に。  
HTTPS自動発行されているはず。

## A-6. ぽかぽか食堂の本番URL確定（5分）

最終的な本番URL：

```
予約フォーム:
https://kodomo.er-biru.net/?tenant=DEFAULT

公開カレンダー（Phase 1.5 完成後）:
https://kodomo.er-biru.net/?tenant=DEFAULT#calendar

LIFF用URL（Phase 1.7 完成後）:
https://kodomo.er-biru.net/?tenant=DEFAULT&liff=1
```

このURLをLINE案内テンプレに反映 → 「Part C」で使う。

---

# Part B: LINE 公式アカウント開設（45分）

## B-1. LINE Official Account Manager で開設（10分）

1. <https://manager.line.biz/> にアクセス
2. アカウント作成 → 個人事業主としてHideoさんで作成
3. アカウント名：**「ぽかぽか食堂」**
4. プロフィール画像：店舗ロゴ or 仮アイコン
5. プランは **コミュニケーション（無料）** のまま

完了すると、友だち追加用 QRコード が発行される。**これは後でLINE案内に使う**。

## B-2. LINE Developers でチャネル作成（10分）

1. <https://developers.line.biz/> にアクセス
2. プロバイダー作成（仮称：「AIの町医者」or「PAPA」）
3. プロバイダー内に **Messaging API** チャネル作成
4. チャネル設定：
   - チャネル名：ぽかぽか食堂予約Bot
   - チャネル説明：こども食堂の予約管理
   - 大業種：レストラン・飲食店
   - 小業種：その他

## B-3. チャネル情報の取得（5分）

「Messaging API設定」タブで取得：

| 項目 | 取得場所 | 用途 |
|---|---|---|
| Channel Access Token | 「チャネルアクセストークン」セクション → 発行 | GAS で API 呼び出し時に使用 |
| Channel Secret | 「基本設定」タブ → チャネルシークレット | Webhook 署名検証 |

**これらは秘密情報**。LINE Developers 画面以外には絶対書かないこと。

## B-4. Webhook URL 設定（5分）

「Messaging API設定」タブで：

1. **Webhook URL**：
   ```
   https://script.google.com/macros/s/AKfycbx0qY.../exec?tenant=DEFAULT
   ```
   ※ 既存 GAS WebApp URL に `?tenant=DEFAULT` を付ける

2. **Webhookの利用**：ON

3. **応答メッセージ**：OFF（Bot側で全て処理）

4. **あいさつメッセージ**：OFF（Bot側で送信）

## B-5. LIFF アプリ作成（10分）

1. LINE Developers → 同じチャネルで「LIFF」タブ
2. 「追加」をクリック
3. 設定：
   - **LIFFアプリ名**：ぽかぽか食堂予約フォーム
   - **サイズ**：Full
   - **エンドポイントURL**：`https://kodomo.er-biru.net/?tenant=DEFAULT&liff=1`
   - **Scope**：profile, openid
   - **ボットリンク機能**：On (Aggressive)

4. 作成後の **LIFF ID** をコピー（形式：`1234567890-AbCdEfGh`）

## B-6. GAS PropertiesService に登録（5分）

GAS エディタを開いて、エディタの左メニュー「プロジェクトの設定」→「スクリプトプロパティ」で以下3つを追加：

| プロパティ名 | 値 |
|---|---|
| LINE_CHANNEL_ACCESS_TOKEN | （B-3 で取得） |
| LINE_CHANNEL_SECRET | （B-3 で取得） |
| LIFF_ID | （B-5 で取得） |

「スクリプトプロパティを保存」を必ず押す。

---

# Part C: 動作確認（30分）

## C-1. LINE 友だち追加（5分）

1. LINE Official Account Manager → 友だち追加用 QRコード を取得
2. 自分のLINEでスキャン → 友だち追加
3. **何もメッセージが返ってこなくてOK**（Phase 1.7 実装前なので応答ロジックがまだない）

## C-2. Webhook の到達確認（10分）

1. LINEから「テスト」と送信
2. GASエディタ → 左メニュー「実行履歴」を開く
3. doPost が実行されているはず → ステータス「完了」を確認

実行されていなければ：
- Webhook URL が正しいか確認
- GAS デプロイが「全員アクセス可」になっているか
- 「Webhookの利用」がONになっているか

## C-3. LIFFアプリ動作確認（10分）

1. LINE Developers → LIFF タブ → 作成したLIFFの「LIFF URL」（`https://liff.line.me/1234567890-AbCdEfGh`）をコピー
2. このURLを自分のLINEに送信 → タップで起動
3. LINE内ブラウザで `kodomo.er-biru.net` のフォームが開くはず
4. 開かない / エラーの場合：
   - エンドポイントURLが正しいか確認
   - GitHub Pages が公開されているか確認
   - 「Scope」が正しく設定されているか確認

## C-4. 既存メール認証フローの確認（5分）

通常のブラウザで `https://kodomo.er-biru.net/?tenant=DEFAULT` にアクセス →  
メール認証 → 予約完了まで一気通貫で動作することを確認。  
**LINE導入による既存機能リグレッションがゼロ**であることを担保。

---

# Part D: Phase 1.7 実装着手前の準備完了確認

以下が全て ✓ になったら、Claude Code に Phase 1.7 実装指示を出してOK。

- [ ] `https://kodomo.er-biru.net/?tenant=DEFAULT` で予約フォーム表示
- [ ] HTTPS 自動発行されている
- [ ] LINE公式アカウント「ぽかぽか食堂」開設済み
- [ ] 友だち追加用 QRコード 取得済み
- [ ] LINE Developers チャネル作成済み
- [ ] LINE_CHANNEL_ACCESS_TOKEN が PropertiesService に登録済み
- [ ] LINE_CHANNEL_SECRET が PropertiesService に登録済み
- [ ] LIFF_ID が PropertiesService に登録済み
- [ ] Webhook URL 設定済み、テスト送信で doPost が実行されることを確認
- [ ] LIFF URL で `kodomo.er-biru.net` が LINE内ブラウザで開くことを確認
- [ ] 既存メール認証フローのリグレッションなし

---

# Part E: Phase 1.7 実装後の最終確認チェックリスト

Claude Code が Phase 1.7 を実装完了したら、以下を実施：

## E-1. LINE Bot 動作（10分）
- [ ] 「予約」とLINE送信 → LIFF URLがReply
- [ ] 「キャンセル」→ 予約番号を聞かれる対話開始
- [ ] 「次回」→ 開催日3件が整形されて返信
- [ ] 「ヘルプ」→ 使い方ガイド返信
- [ ] 意味不明な文字列 → エラー返信

## E-2. LIFF 認証フロー（10分）
- [ ] リッチメニュー「📅予約する」タップ → 予約フォームが開く
- [ ] メール認証スキップで予約完了できる
- [ ] 利用者マスタに line_user_id が登録される

## E-3. 一気通貫テスト（10分）

実際の家族 or 知人に協力してもらい：

1. LINE友だち追加
2. 「予約」と送信 → リプライ
3. リッチメニューから予約 → 完了
4. 翌日：リマインダーメール受信
5. 「マイ予約」タップ → 履歴確認
6. キャンセル対話で取り消し

これが全部スムーズに進めば、**ぽかぽか食堂は世界の中で最も洗練されたこども食堂予約システムを持つ食堂** になります。

---

# Part F: LINE案内テンプレ（更新版）

GitHub Pages + LINE 公開後、LINEグループ等で配布する案内：

```
🍙 ぽかぽか食堂 6月の予約はじまりました 🍙

📱 LINEで友だち追加するだけで予約OK！
→ QRコードはこちら：[QR画像]

または、ブラウザから直接予約：
https://kodomo.er-biru.net/?tenant=DEFAULT

📝 予約のながれ
①LINEで友だち追加
②リッチメニュー「📅予約する」をタップ
③日にち・人数をえらぶ
④かんりょう！（確認メールが届きます）

📅 6月の開催日
・6/7(日) ふつうメニュー
・6/14(日) ふつうメニュー
・6/21(日) カレーの日 🍛
・6/28(日) ふつうメニュー

⏰ 17:00〜19:00
📍 ぽかぽか食堂

ご家族みなさまでどうぞ😊
他の方には予約内容は見えませんので、安心してお申し込みください。
```

最後の一文「**他の方には予約内容は見えません**」が、グループLINE運用からの卒業を象徴する一文。利用者にも運営者にも刺さる文言です。

---

# トラブルシューティング

## GitHub Pages が反映されない
- リポジトリの Settings → Pages で `Source` が `main` ブランチ・`/frontend` フォルダになっているか
- CNAME ファイルが `frontend/` 配下にあるか（リポジトリルートではない）

## Cloudflare で CNAME が動かない
- Cloudflare の Proxy status が `Proxied`（オレンジ雲）になっているか
- DNS反映待ち（最大15分）
- GitHub Pages の Custom domain 欄に `kodomo.er-biru.net` が設定されているか

## LINE Webhook が動かない
- Webhook URL が正しいか
- GAS デプロイ設定で「全員アクセス可」になっているか
- LINE Developers の「Webhookの利用」がONか
- 「応答メッセージ」がOFFか（ONだとLINE側で先に応答してBotに来ない）

## LIFF アプリが開かない
- エンドポイントURL が正しいか
- HTTPSになっているか（HTTPはNG）
- LIFFアプリが「公開済み」になっているか
- Scope が `profile, openid` になっているか
