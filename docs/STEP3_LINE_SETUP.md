# Step 3 LINE & GitHub セットアップ進捗

> 機密情報（Channel Secret / Access Token）は **GAS スクリプトプロパティ専用**。
> このドキュメント・コード本体・git・memory に書き込み禁止。
> 値の確認は LINE Developers Console から都度取得すること。

---

## 取得済みの情報（2026-05-27）

### 公開情報
| 項目 | 値 |
|---|---|
| GitHub リポジトリ | <https://github.com/hideo-t/pokapoka> |
| LINE 公式アカウント Manager | <https://manager.line.biz/account/@533dgghw> |
| LINE Bot ID (友だち追加用) | `@533dgghw` |
| LIFF URL（ユーザーが開く）| <https://liff.line.me/2010207912-vQPmUWd1> |
| LIFF ID（フロントで使う） | `2010207912-vQPmUWd1` |

### 秘匿情報（このドキュメントに値は書きません）
- 🔒 **Channel Access Token** — Hideoさんが LINE Developers Console から取得・GAS スクリプトプロパティに登録（手順は下記）
- 🔒 **Channel Secret** — 同上

---

## スクリプトプロパティ登録手順（Hideoさん作業・5分）

### 1. GAS エディタを開く
<https://script.google.com/u/0/home/projects/1Vhj2IrWo2IhE9IUzL7PQaNEmDkcdTSNr6qhS2dlndSkehNfXW4eEj35N/edit>

### 2. 「プロジェクトの設定」(左メニュー ⚙️) を開く
- 「スクリプト プロパティ」セクションを展開
- 「スクリプト プロパティを編集」をクリック

### 3. 3件のプロパティを追加

| プロパティ名 | 値の取得元 |
|---|---|
| `LINE_CHANNEL_ACCESS_TOKEN` | LINE Developers Console → 該当チャネル → 「Messaging API設定」タブ → 「チャネルアクセストークン（長期）」を発行/コピー |
| `LINE_CHANNEL_SECRET` | LINE Developers Console → 該当チャネル → 「チャネル基本設定」タブ → 「チャネルシークレット」 |
| `LIFF_ID` | `2010207912-vQPmUWd1` （上記の公開情報、これは書いてOK） |

### 4. 「スクリプト プロパティを保存」をクリック

### 5. 設定確認関数 (Phase 1.7 実装時に Claude が用意)
```javascript
function checkLineCredentials() {
  var props = PropertiesService.getScriptProperties();
  Logger.log('TOKEN: ' + (props.getProperty('LINE_CHANNEL_ACCESS_TOKEN') ? '✅ 設定済 (長さ' + props.getProperty('LINE_CHANNEL_ACCESS_TOKEN').length + ')' : '❌ 未設定'));
  Logger.log('SECRET: ' + (props.getProperty('LINE_CHANNEL_SECRET') ? '✅ 設定済 (長さ' + props.getProperty('LINE_CHANNEL_SECRET').length + ')' : '❌ 未設定'));
  Logger.log('LIFF_ID: ' + (props.getProperty('LIFF_ID') || '❌ 未設定'));
}
```
GAS エディタで ▶ 実行 → 3件すべて ✅ 設定済 を確認

---

## LINE Developers コンソール 設定

### Webhook URL を GAS Web App URL に設定

1. <https://developers.line.biz/console/> → 該当プロバイダー → Messaging API チャネル
2. 「Webhook 設定」セクション
3. **Webhook URL** に貼付:
   ```
   https://script.google.com/macros/s/AKfycbx0qYEXzMT8gOVYK0POBxo86oc2OIsW3180S2CblCoKOa6frSuba7NrFdznlNfmuvSj/exec
   ```
4. 「Webhook の利用」を **オン**
5. LINE Official Account Manager 側で「応答メッセージ」「あいさつメッセージ」を**オフ**（Bot応答と競合するため）

### Webhook 検証
LINE Developers コンソールの「検証」ボタン → 200 OK が返れば疎通OK
（現状の doPost は `unknown action` を返すが、HTTPステータスは 200 なので検証は通る）

---

## GitHub リポジトリ進捗

| 状態 | 内容 |
|---|---|
| ✅ リポジトリ作成 | `hideo-t/pokapoka` (Public) |
| ⏳ プッシュ前 | frontend/ docs/ README.md .gitignore をコミット予定 |
| ⏳ Pages 設定前 | `main` ブランチ・`/frontend` フォルダで公開予定 |
| ⏳ カスタムドメイン未設定 | `kodomo.er-biru.net` (Cloudflare 経由) |

### `.gitignore` 内容（Step 3 手順書通り）
```
# GAS バックエンドは公開しない
gas/
.clasp.json
.claspignore

# Claude Code 作業ファイル
.claude/
*.log
.DS_Store
Thumbs.db

# ローカル設定
.env
.env.local

# 機密扱いの docs
docs/PRIVATE_*.md
```

`gas/` が除外されていれば、機密値がコード本体に書かれていない限り **git に流出しない設計**。

---

## 完了チェック

- [ ] GAS スクリプトプロパティに `LINE_CHANNEL_ACCESS_TOKEN` を登録
- [ ] GAS スクリプトプロパティに `LINE_CHANNEL_SECRET` を登録
- [ ] GAS スクリプトプロパティに `LIFF_ID` を登録
- [ ] `checkLineCredentials()` で 3件全て ✅ 設定済 を確認 (Phase 1.7 実装後)
- [ ] LINE Developers コンソールで Webhook URL 設定
- [ ] LINE Developers コンソールで Webhook 検証 200 OK
- [ ] LINE Official Account Manager で「応答メッセージ」「あいさつメッセージ」をオフ
- [ ] GitHub リポジトリに frontend/ をプッシュ
- [ ] GitHub Pages 公開設定
- [ ] (将来) kodomo.er-biru.net をカスタムドメインとして紐付け

---

## セキュリティ運用ルール

1. **トークン値はチャット・docs・コード・memoryに書かない**（再取得が必要な場合は LINE Developers Console から直接）
2. **コード本体 (`gas/*.gs`) には書かない**（PropertiesService 経由で取得）
3. **docs/ に書かない**（GitHub 公開リポジトリに流出するため）
4. **再生成手順**: LINE Developers コンソール → チャネル設定 → アクセストークン再発行 → 旧トークン無効化 → 新トークンをスクリプトプロパティに再設定
