# Git Rebase 実践ガイド（チーム開発向け）

## 目次

1. [Rebase の基本概念](#1-rebase-の基本概念)
2. [チーム開発でのブランチ戦略](#2-チーム開発でのブランチ戦略)
3. [基本的な Rebase 操作](#3-基本的な-rebase-操作)
4. [Interactive Rebase（コミット整理）](#4-interactive-rebaseコミット整理)
5. [コンフリクト解消手順](#5-コンフリクト解消手順)
6. [Pull Rebase（fetch + rebase）](#6-pull-rebasefetch--rebase)
7. [業務でのよくあるシナリオ](#7-業務でのよくあるシナリオ)
8. [Rebase のチームルール（推奨）](#8-rebase-のチームルール推奨)
9. [緊急時のリカバリ手順](#9-緊急時のリカバリ手順)

---

## 1. Rebase の基本概念

### Merge vs Rebase

```
【Merge の場合】
main:    A---B---C---------M
              \           /　
feature:       D---E---F

【Rebase の場合】
main:    A---B---C
                  \
feature:           D'--E'--F'
```

| 観点 | Merge | Rebase |
|------|-------|--------|
| 履歴 | 分岐履歴がそのまま残る | 直線的な履歴になる |
| コミット | マージコミットが生成される | コミットが再作成される |
| 適用場面 | feature → main の取込 | ローカル整理・main の変更取込 |
| リスク | 低い | 共有ブランチへの使用は危険 |

> **重要：** プッシュ済みの共有ブランチ（`main`, `develop` 等）には絶対に Rebase しない。

---

## 2. チーム開発でのブランチ戦略

### 推奨ブランチ構成（Git Flow ベース）

```
main          ─── 本番リリース済みコード
develop       ─── 開発統合ブランチ
feature/*     ─── 機能開発（個人作業）
hotfix/*      ─── 本番障害対応
release/*     ─── リリース準備
```

### Rebase を使うタイミング

```
[OK] feature ブランチで develop の最新を取り込む
[OK] プッシュ前にローカルコミットを整理する
[OK] レビュー前にコミットを squash してまとめる
[NG] 他メンバーが使っている共有ブランチを rebase する
[NG] main/develop を rebase する
```

---

## 3. 基本的な Rebase 操作

### 3-1. feature ブランチを develop の最新に追従させる

```bash
# 1. リモートの最新情報を取得
git fetch origin

# 2. feature ブランチにいることを確認
git switch feature/my-feature

# 3. develop の最新へ rebase
git rebase origin/develop

# 4. コンフリクトがなければ完了。リモートへ強制プッシュ（自分のブランチのみ）
git push --force-with-lease origin feature/my-feature
```

> `--force-with-lease` は他者が同ブランチにプッシュしていた場合に失敗するため、`--force` より安全。

### 3-2. 現在の状況確認

```bash
# rebase 前後のログ確認
git log --oneline --graph origin/develop..HEAD

# リモートとのずれ確認
git status
```

---

## 4. Interactive Rebase（コミット整理）

### 4-1. プッシュ前のコミット整理（squash / reword）

作業中に細かくコミットした内容を、PR（プルリクエスト）前にまとめる。

```bash
# 直近 3 コミットを対話的に操作
git rebase -i HEAD~3
```

エディタが開いたら、各コミットの先頭のコマンドを変更する：

```
# 変更前（デフォルト）
pick abc1234 WIP: ログ追加
pick def5678 fix: バグ修正
pick ghi9012 fix: バグ修正（続き）

# 変更後（squash でまとめる）
pick abc1234 WIP: ログ追加
squash def5678 fix: バグ修正
squash ghi9012 fix: バグ修正（続き）
```

### 4-2. コマンド一覧

| コマンド | 短縮形 | 動作 |
|---------|--------|------|
| `pick`  | `p` | コミットをそのまま使用 |
| `reword`| `r` | コミットメッセージを変更 |
| `edit`  | `e` | コミット内容を修正 |
| `squash`| `s` | 直前のコミットに統合（メッセージも統合） |
| `fixup` | `f` | 直前のコミットに統合（メッセージは破棄） |
| `drop`  | `d` | コミットを削除 |

### 4-3. 実践例：PR 前のコミット整理フロー

```bash
# 1. 作業ブランチの状態を確認
git log --oneline HEAD~5..HEAD

# 例）以下のようなコミット履歴
# a1b2c3d fix: ログ追加
# e4f5g6h wip: 途中
# i7j8k9l wip: 途中2
# m0n1o2p feat: ユーザー認証機能追加

# 2. 4 コミットをまとめて整理
git rebase -i HEAD~4

# 3. エディタで整形（例）
# pick m0n1o2p feat: ユーザー認証機能追加
# fixup i7j8k9l wip: 途中2
# fixup e4f5g6h wip: 途中
# reword a1b2c3d fix: ログ追加

# 4. 完了後にプッシュ
git push --force-with-lease origin feature/my-feature
```

---

## 5. コンフリクト解消手順

### 5-1. Rebase 中のコンフリクト発生時

```bash
# rebase 実行後、コンフリクトが発生した場合
git rebase origin/develop

# 出力例：
# CONFLICT (content): Merge conflict in src/auth.js
# error: could not apply abc1234... feat: 認証ロジック追加
```

### 5-2. 解消手順

```bash
# 1. コンフリクトファイルを確認
git status

# 2. コンフリクトファイルをエディタで修正
#    <<<<<<< HEAD（現在の変更）と >>>>>>> （取り込む変更）を解消

# 3. 修正したファイルをステージング
git add src/auth.js

# 4. rebase を再開（コミットは不要）
git rebase --continue

# コンフリクトが複数コミットにまたがる場合、手順 2〜4 を繰り返す
```

### 5-3. Rebase の中断・やり直し

```bash
# rebase を全て取り消してやり直す
git rebase --abort
```

---

## 6. Pull Rebase（fetch + rebase）

### 6-1. 設定（チーム全体で統一推奨）

```bash
# git pull をデフォルトで rebase にする（個人設定）
git config --global pull.rebase true

# または プロジェクト単位で設定
git config pull.rebase true
```

### 6-2. 実行例

```bash
# マージコミットを作らずにリモートを取り込む
git pull --rebase origin develop

# エイリアスとして登録しておくと便利
git config --global alias.pr 'pull --rebase'
git pr origin develop
```

---

## 7. 業務でのよくあるシナリオ

### シナリオ A：レビュー指摘後のコミット修正

```bash
# 1. 修正を加えた新しいコミットを作成
git add .
git commit -m "fix: レビュー指摘対応"

# 2. 既存コミットに統合（interactive rebase）
git rebase -i HEAD~2

# エディタで
# pick abc1234 feat: ユーザー認証機能追加
# fixup def5678 fix: レビュー指摘対応   ← pick を fixup に変更

# 3. 強制プッシュ
git push --force-with-lease origin feature/my-feature
```

### シナリオ B：長期作業ブランチで develop が大幅に進んだ場合

```bash
# 1. リモートの最新を取得
git fetch origin

# 2. 差分を確認（どれだけ離れているか）
git log --oneline feature/my-feature..origin/develop

# 3. rebase 実行
git rebase origin/develop

# 4. コンフリクトを順番に解消しながら進める
#    （コンフリクトごとに git add → git rebase --continue）

# 5. テストを実行してから強制プッシュ
npm test
git push --force-with-lease origin feature/my-feature
```

### シナリオ C：誤ったコミットメッセージの修正（プッシュ前）

```bash
# 直前のコミットメッセージのみ修正
git commit --amend -m "feat: 正しいコミットメッセージ"

# 過去のコミットメッセージを修正（例：3 つ前）
git rebase -i HEAD~3
# エディタで対象コミットを pick → reword に変更して保存
# 次に開くエディタでメッセージを修正
```

### シナリオ D：PR レビュー中に main/develop に hotfix がマージされた

```bash
# 1. リモートの最新を取得
git fetch origin

# 2. feature ブランチを最新の develop に rebase
git rebase origin/develop

# 3. コンフリクトがあれば解消し、テスト後にプッシュ
git push --force-with-lease origin feature/my-feature

# 4. PR のレビュアーに rebase したことを連絡する
```

---

## 8. Rebase のチームルール（推奨）

### DO（やること）

- [x] feature ブランチはマージ前に `origin/develop` へ rebase する
- [x] PR 提出前にコミットを squash/fixup で整理する
- [x] `--force-with-lease` を使って強制プッシュする
- [x] rebase 後は必ずテストを通してからプッシュする
- [x] rebase したことを PR やチャットでレビュアーに伝える

### DON'T（やってはいけないこと）

- [ ] `main` / `develop` / `release` など共有ブランチを rebase しない
- [ ] `--force`（`--force-with-lease` でなく）を共有ブランチに使わない
- [ ] rebase 中にコンフリクトを適当に解消しない（必ず動作確認）
- [ ] 大量のコンフリクトが発生する場合は、一人で解決せずチームに相談する

### コミットメッセージ規約（Conventional Commits）

```
feat:     新機能の追加
fix:      バグ修正
docs:     ドキュメントのみの変更
style:    コードの意味に影響しない変更（フォーマット等）
refactor: バグ修正でも機能追加でもないコードの変更
test:     テストの追加・修正
chore:    ビルドプロセスや補助ツールの変更
```

---

## 9. 緊急時のリカバリ手順

### 9-1. rebase を間違えた場合（ローカルのみ）

```bash
# reflog で直前の状態を確認
git reflog

# 出力例：
# abc1234 HEAD@{0}: rebase (finish): ...
# def5678 HEAD@{1}: rebase (start): ...
# ghi9012 HEAD@{2}: commit: feat: 認証機能追加  ← rebase 前の状態

# rebase 前の状態に戻す
git reset --hard HEAD@{2}
```

### 9-2. force push 後に間違いに気づいた場合

```bash
# reflog でリモートプッシュ前の状態を特定
git reflog

# 巻き戻し
git reset --hard <commit-hash>

# チームに連絡してから再プッシュ
git push --force-with-lease origin feature/my-feature
```

### 9-3. rebase 中のコンフリクトで作業が止まった場合

```bash
# 現在の rebase 状態を確認
git status
cat .git/rebase-merge/msgnum   # 現在何番目のコミットか
cat .git/rebase-merge/end      # 全コミット数

# 一旦中断して仕切り直す
git rebase --abort

# 必要なら最新の develop を取得してから再実行
git fetch origin
git rebase origin/develop
```

---

## クイックリファレンス

```bash
# 最新の develop を取り込む
git fetch origin && git rebase origin/develop

# コミット整理（直近 N 件）
git rebase -i HEAD~N

# 強制プッシュ（安全）
git push --force-with-lease origin <branch>

# rebase の中断
git rebase --abort

# rebase の続行（コンフリクト解消後）
git add <file> && git rebase --continue

# リカバリ
git reflog
git reset --hard HEAD@{N}
```
