# Git Rebase 実践問題（チーム開発シミュレーション）

> **前提：** ローカル環境で安全に練習するため、専用の練習リポジトリを使います。  
> 本番リポジトリでは絶対に練習しないでください。

---

## 環境セットアップ

```bash
# 練習用リポジトリを作成
mkdir git-rebase-practice
cd git-rebase-practice
git init
git config user.name "Your Name"
git config user.email "you@example.com"

# 初期コミット（main ブランチ）
echo "# プロジェクト" > README.md
git add README.md
git commit -m "feat: initial commit"

# develop ブランチを作成
git switch -c develop
echo "console.log('app start');" > app.js
git add app.js
git commit -m "feat: アプリ起動処理を追加"
```

---

## 問題 1：feature ブランチを develop に追従させる（基礎）

### シナリオ

あなたは `feature/login` ブランチで認証機能を開発中です。  
その間にチームメンバーが `develop` ブランチに新しいコミットをプッシュしました。  
`feature/login` を最新の `develop` に追従させてください。

### 手順

```bash
# --- チームメンバーの作業をシミュレート ---
git switch develop
echo "function validateInput(v) { return !!v; }" >> app.js
git add app.js
git commit -m "feat: 入力バリデーション関数を追加"

# --- あなたの作業ブランチを作成（develop が進む前のポイントから） ---
git switch -c feature/login HEAD~1
echo "function login(user, pass) {}" >> app.js
git add app.js
git commit -m "feat: ログイン関数の雛形を追加"
```

### 問題

① 現在のブランチと develop の差分をグラフで確認してください。

② `feature/login` を最新の `develop` に rebase してください。

③ rebase 後のログをグラフ付きで確認し、直線的な履歴になっていることを確認してください。

### 答え合わせ

<details>
<summary>解答を見る</summary>

```bash
# ① ログ確認
git log --oneline --graph --all

# ② rebase 実行
git switch feature/login
git rebase develop

# ③ 結果確認
git log --oneline --graph --all
# feature/login のコミットが develop の先端に移動していればOK
```

</details>

---

## 問題 2：Interactive Rebase でコミット整理（PR 提出前）

### シナリオ

`feature/user-profile` ブランチで作業中、以下のような雑なコミット履歴になってしまいました。  
PR 提出前に、意味のある 2 つのコミットにまとめてください。

### 手順

```bash
git switch develop
git switch -c feature/user-profile

# 細かすぎるコミットを再現
echo "function getUser() {}" >> app.js
git add app.js
git commit -m "wip"

echo "function getUser(id) { return id; }" >> app.js
git add app.js
git commit -m "wip2"

echo "function updateUser(id, data) {}" >> app.js
git add app.js
git commit -m "feat: ユーザー更新"

echo "function updateUser(id, data) { return data; }" >> app.js
git add app.js
git commit -m "fix"

echo "// user profile module" >> app.js
git add app.js
git commit -m "docs: コメント追加"
```

この時点で履歴は以下のようになっています：
```
docs: コメント追加
fix
feat: ユーザー更新
wip2
wip
```

### 問題

① 直近 5 つのコミットを interactive rebase で開いてください。

② 以下の 2 つのコミットにまとめてください：
   - `feat: ユーザー取得機能を追加`（wip + wip2 を統合）
   - `feat: ユーザー更新機能を追加`（feat: ユーザー更新 + fix + docs を統合）

③ 整理後のログを確認してください。

### 答え合わせ

<details>
<summary>解答を見る</summary>

```bash
# ① interactive rebase を開く
git rebase -i HEAD~5

# ② エディタで以下のように変更して保存
# pick <hash> wip
# squash <hash> wip2
# reword <hash> feat: ユーザー更新   ← このコミットで新メッセージを入力
# fixup <hash> fix
# fixup <hash> docs: コメント追加

# squash のメッセージ入力画面で：
#   "feat: ユーザー取得機能を追加" と入力して保存

# reword のメッセージ入力画面で：
#   "feat: ユーザー更新機能を追加" と入力して保存

# ③ 確認
git log --oneline
# 2 コミットになっていればOK
```

</details>

---

## 問題 3：Rebase 中のコンフリクト解消

### シナリオ

`feature/payment` ブランチで支払い処理を開発中に、  
`develop` に同じファイルを編集するコミットが追加されました。  
コンフリクトを解消して rebase を完了させてください。

### 手順

```bash
# develop に決済ライブラリの設定が追加されたとする
git switch develop
cat > payment.js << 'EOF'
// 決済モジュール
const stripe = require('stripe');

function processPayment(amount) {
  return stripe.charge(amount);
}

module.exports = { processPayment };
EOF
git add payment.js
git commit -m "feat: Stripe 決済処理を追加"

# develop が進んだ後に feature ブランチを作成（1つ前から）
git switch -c feature/payment HEAD~1
cat > payment.js << 'EOF'
// 決済モジュール
const paypal = require('paypal');

function processPayment(amount) {
  return paypal.pay(amount);
}

module.exports = { processPayment };
EOF
git add payment.js
git commit -m "feat: PayPal 決済処理を追加"
```

### 問題

① `feature/payment` を `develop` に rebase して、コンフリクトを発生させてください。

② コンフリクト内容を確認し、**両方の決済手段を残す**形で解消してください。

③ rebase を完了させ、最終的なファイル内容を確認してください。

### 期待する最終的な payment.js の内容

```javascript
// 決済モジュール
const stripe = require('stripe');
const paypal = require('paypal');

function processPayment(amount, method = 'stripe') {
  if (method === 'paypal') return paypal.pay(amount);
  return stripe.charge(amount);
}

module.exports = { processPayment };
```

### 答え合わせ

<details>
<summary>解答を見る</summary>

```bash
# ① rebase 実行（コンフリクトが発生する）
git switch feature/payment
git rebase develop

# コンフリクト確認
git status
# → payment.js がコンフリクト状態

# ② payment.js をエディタで開いて編集
# <<<<<<< HEAD や ======= などのマーカーを取り除き、
# 期待するファイル内容に書き換える

# ③ ステージして続行
git add payment.js
git rebase --continue

# 完了後確認
cat payment.js
git log --oneline --graph
```

</details>

---

## 問題 4：レビュー指摘対応後のコミット統合

### シナリオ

PR のコードレビューで「バリデーションが不足している」と指摘されました。  
修正コミットを既存のコミットに統合して、PR の履歴をきれいに保ってください。

### 手順

```bash
git switch develop
git switch -c feature/register

# 元の実装
cat > register.js << 'EOF'
function register(email, password) {
  // TODO: バリデーション追加
  saveUser(email, password);
}
EOF
git add register.js
git commit -m "feat: ユーザー登録機能を追加"

# レビュー指摘後の修正コミット
cat > register.js << 'EOF'
function register(email, password) {
  if (!email || !password) throw new Error('入力が不正です');
  if (password.length < 8) throw new Error('パスワードは8文字以上');
  saveUser(email, password);
}
EOF
git add register.js
git commit -m "fix: レビュー指摘対応 - バリデーション追加"
```

### 問題

① 現在のコミット履歴を確認してください（2 コミットある状態）。

② `fix: レビュー指摘対応 - バリデーション追加` を `feat: ユーザー登録機能を追加` に **fixup** で統合してください。

③ 最終的に 1 コミットになり、メッセージが `feat: ユーザー登録機能を追加` のままであることを確認してください。

④ 強制プッシュのコマンドを答えてください（実際にはリモートがないので确認のみ）。

### 答え合わせ

<details>
<summary>解答を見る</summary>

```bash
# ① 確認
git log --oneline

# ② interactive rebase
git rebase -i HEAD~2

# エディタ：
# pick <hash> feat: ユーザー登録機能を追加
# fixup <hash> fix: レビュー指摘対応 - バリデーション追加
# ↑ pick を fixup に変更して保存

# ③ 確認
git log --oneline
# 1 コミットになっていればOK

# ④ 強制プッシュコマンド（--force-with-lease を使うこと）
git push --force-with-lease origin feature/register
```

</details>

---

## 問題 5：総合問題 ─ 長期ブランチの追従とコミット整理

### シナリオ

あなたは 3 日間かけて `feature/dashboard` を開発しました。  
その間 `develop` には 3 つのコミットが追加されました。  
PR 提出に向けて以下を全て実施してください。

1. `develop` の最新を `feature/dashboard` に取り込む（rebase）
2. 作業中の細かいコミットを整理して 2 コミットにまとめる
3. 強制プッシュ用コマンドを確認する

### 手順（環境構築）

```bash
# develop に 3 コミット追加
git switch develop
echo "function auth() {}" >> app.js && git add . && git commit -m "feat: 認証ミドルウェア"
echo "function logger() {}" >> app.js && git add . && git commit -m "feat: ロガー追加"
echo "function errorHandler() {}" >> app.js && git add . && git commit -m "feat: エラーハンドラー追加"

# feature ブランチを 3 コミット前から開始
git switch -c feature/dashboard HEAD~3

# ダッシュボード開発（雑なコミット）
echo "// dashboard" > dashboard.js && git add . && git commit -m "wip: ダッシュボード作業開始"
echo "function renderChart() {}" >> dashboard.js && git add . && git commit -m "wip: グラフ描画"
echo "function renderTable() {}" >> dashboard.js && git add . && git commit -m "feat: テーブル表示"
echo "function renderChart(data) { return data; }" >> dashboard.js && git add . && git commit -m "fix: グラフ修正"
echo "// dashboard module v1" >> dashboard.js && git add . && git commit -m "docs: コメント"
```

### 問題

① 現在の全ブランチのログをグラフで表示し、状況を把握してください。

② `feature/dashboard` を最新の `develop` に rebase してください。

③ rebase 後、`feature/dashboard` の 5 コミットを以下の 2 つに整理してください：
   - `feat: ダッシュボードのグラフ・テーブル表示を追加`
   - `docs: ダッシュボードモジュールにコメントを追加`

④ 最終的なログをグラフで確認し、期待する状態になっているか確認してください。

⑤ この後チームメンバーへ PR を出す前にすべき確認事項を 3 つ挙げてください。

### 答え合わせ

<details>
<summary>解答を見る</summary>

```bash
# ① ログ確認
git log --oneline --graph --all

# ② rebase
git switch feature/dashboard
git rebase develop
# コンフリクトが発生した場合は解消 → git add → git rebase --continue

# ③ コミット整理
git rebase -i HEAD~5

# エディタ：
# pick <hash> wip: ダッシュボード作業開始
# squash <hash> wip: グラフ描画
# squash <hash> feat: テーブル表示
# fixup <hash> fix: グラフ修正
# reword <hash> docs: コメント   ← または pick にして次のステップで修正

# squash のメッセージ → "feat: ダッシュボードのグラフ・テーブル表示を追加"
# reword のメッセージ → "docs: ダッシュボードモジュールにコメントを追加"

# ④ 最終確認
git log --oneline --graph --all
# develop から直線的に 2 コミット伸びていればOK

# ⑤ PR 前の確認事項（例）
# 1. テスト・ビルドが通ること（npm test 等）
# 2. コードレビュー依頼前に diff を自分で確認する
# 3. 強制プッシュ後にレビュアーへ「rebase しました」と連絡する
```

</details>

---

## チェックリスト（練習完了の確認）

練習が終わったら、以下を全てチェックできているか確認してください。

- [ ] `git rebase <branch>` で他ブランチの変更を取り込める
- [ ] `git rebase -i HEAD~N` でコミットを対話的に整理できる
- [ ] `squash` / `fixup` / `reword` / `drop` の違いが説明できる
- [ ] rebase 中のコンフリクトを `git add` + `git rebase --continue` で解消できる
- [ ] `git rebase --abort` で rebase を安全にキャンセルできる
- [ ] `git push --force-with-lease` と `--force` の違いが説明できる
- [ ] `git reflog` + `git reset --hard` でリカバリできる
- [ ] 共有ブランチ（main/develop）を rebase してはいけない理由が説明できる
