---
name: verify-local-rails-runtime
description: Use when the user asks to actually verify the local Science Planner app by running Rails in a worktree, using a port in 3001 to 3005, and checking the login flow in a browser.
---

# ローカルRails実動確認

この skill は、このプロジェクトで「実際に動くか見たい」「スマホで確認したい」「Rails を起動してログインまで確かめたい」と言われたときに使う。

この skill のゴールは、**テストが通ること**ではなく、**指定された worktree の Rails を起動し、ユーザー自身がブラウザで確認できる状態を作り、ログイン後の通常画面まで実際に確かめること**。

## 成功条件

以下をすべて満たしたら成功とみなす。

1. 対象 worktree または repo が明確である
2. `3001` から `3005` の中で使うポートが明確である
3. 対象の Rails がそのポートで起動している
4. Playwright で `/session/new` または `/` を開き、ログイン画面に到達できる
5. Playwright で `ikeda@example.com / password` でログインできる
6. ログイン後に `PendingMigrationError` や例外ページではなく通常画面が表示される
7. ユーザーへ、起動元 worktree、ポート、DB コピー有無、migration 有無、確認 URL を報告している

`curl` は補助確認であり、**必須成功条件ではない**。sandbox の都合で `curl` が失敗しても、Playwright で正常画面まで確認できれば成功扱いにしてよい。

## 固定値

### 開発用ログイン情報

- email: `ikeda@example.com`
- password: `password`

### 使用ポート候補

- `3001`
- `3002`
- `3003`
- `3004`
- `3005`

### source repo の基準パス

`/Users/tatsuroueda/Dropbox/SharedDocs/Business_documents/2-2_Flow_by_Month/17_Products/science-planner-v2`

## 実行仕様

## 1. 対象ディレクトリを決める

- ユーザーが worktree を指定したらそこを使う
- ユーザーが「この branch / この worktree を見たい」と言ったらそれを優先する
- 指定がなければ、現在作業中の `science-planner-v2` の worktree を優先し、無ければ本体 repo を使う

確認したいこと:

- どのディレクトリで起動するか
- そのディレクトリの branch は何か
- すでにその worktree 専用の Puma が動いていないか

最低限見るもの:

```bash
pwd
git branch --show-current
ls tmp/pids
```

## 2. 3001〜3005 の空きポートを探す

まず `3001` から `3005` を順に確認し、**ユーザーの指定があればそのポートを優先**し、指定がなければ**最初に空いているポート**を今回の `PORT` にする。

```bash
for port in 3001 3002 3003 3004 3005; do
  lsof -iTCP:$port -sTCP:LISTEN -n -P
done
```

ポートが埋まっていたら、次を確認する。

- どの PID が使っているか
- どの worktree / repo の Puma か
- 今回確認したい対象と同じものか

確認例:

```bash
ps -p <PID> -o pid=,ppid=,command=
lsof -p <PID> | rg 'science-planner-v2'
```

運用ルール:

- **別 worktree の Puma は勝手に止めない**
- 今回確認したい対象と別なら、次の空きポートを使う
- `3001`〜`3005` が全部埋まっていたら、勝手に kill せず報告する

ただし、**今回確認したいのと同じ worktree の古い Puma** が残っていて、再起動が必要な場合は別である。

- 同じ worktree の healthy な Puma なら再利用してよい
- 同じ worktree の stale PID / 不健康な Puma なら、停止して起動し直してよい
- 停止に escalation が必要なら、承認を取りに行く

## 3. 開発DBを安全に準備する

対象 worktree の `storage/development.sqlite3` を確認する。

```bash
ls -l storage
bin/rails runner 'puts({users: User.count, login_user: User.find_by(email: "ikeda@example.com")&.id}.inspect)'
```

以下のどれかなら、**source repo の開発DBを worktree へコピーする**。

- `development.sqlite3` が無い
- DB が極端に小さい
- `ikeda@example.com` が見つからない
- ログイン後にデータが足りない
- ユーザーが「源流の開発DBを使って」と言っている

### 重要ルール

- **symlink ではなく copy を使う**
- **copy 前に worktree 側 DB を退避する**
- **copy 後は必ず worktree 側で `db:migrate` する**

理由:

- symlink だと、worktree 側の migration や動作確認が source repo の DB を直接壊す
- source repo の DB もコードに対して古いことがある
- だから、**copy してから migrate** が安全

コピー元:

`/Users/tatsuroueda/Dropbox/SharedDocs/Business_documents/2-2_Flow_by_Month/17_Products/science-planner-v2/storage/development.sqlite3`

安全な手順:

```bash
cp storage/development.sqlite3 storage/development.sqlite3.worktree-copy
cp /Users/tatsuroueda/Dropbox/SharedDocs/Business_documents/2-2_Flow_by_Month/17_Products/science-planner-v2/storage/development.sqlite3 storage/development.sqlite3
bin/rails db:migrate
```

その後、最低限これを確認する。

```bash
bin/rails runner 'puts({users: User.count, login_user: User.find_by(email: "ikeda@example.com")&.id, schema_migrations: ActiveRecord::Base.connection.select_value("SELECT COUNT(*) FROM schema_migrations")}.inspect)'
```

期待する最低条件:

- `users` が 1 以上
- `login_user` が `nil` ではない
- `schema_migrations` が 0 ではない

## 4. server.pid と既存 Puma を扱う

`tmp/pids/server.pid` がある場合、**それだけで「正常起動中」と思い込まない**。

確認:

```bash
cat tmp/pids/server.pid
ps -p <PID> -o pid,ppid,command
```

分岐:

- PID が実在し、今回の worktree の Puma なら再利用候補
- PID が死んでいるなら stale PID なので、削除してから起動する
- PID が別 worktree の Puma なら、安易に消さない

stale PID の例:

```bash
rm -f tmp/pids/server.pid
```

実プロセスを止める必要があるなら、必要に応じて escalation を使う。

## 5. Rails を起動する

確認用では、ユーザーがそのままブラウザで触れる状態を優先する。

基本:

```bash
bin/rails server -b 0.0.0.0 -p <PORT> -d
```

起動後に見るもの:

```bash
lsof -iTCP:<PORT> -sTCP:LISTEN -n -P
cat tmp/pids/server.pid
```

もし起動が怪しいときは、foreground でログを見る。

```bash
bin/rails server -b 0.0.0.0 -p <PORT>
```

## 6. 入口確認は Playwright を正とする

今回の経験では、**`curl` は sandbox の都合で失敗しても、Playwright では普通に開ける**ことがあった。

なので判定優先順位はこうする。

1. **Playwright で開けるか**
2. 必要なら `lsof` と `server.pid`
3. `curl` は補助

最初に Playwright で、次のどちらかを開く。

- `http://localhost:<PORT>/`
- `http://localhost:<PORT>/session/new`

正常系の目安:

- `/` がログイン画面へ遷移する
- `/session/new` に email と password 入力欄がある
- 例外画面ではない

`curl` を使うなら補助確認として扱う。

```bash
curl -I http://localhost:<PORT>
curl -I http://127.0.0.1:<PORT>
```

`curl` が失敗しても、Playwright が成功していれば **sandbox 制約の可能性を優先して考える**。

## 7. Playwright でログイン確認する

Playwright で次を行う。

1. `http://localhost:<PORT>/session/new` を開く
2. `ikeda@example.com` を入力する
3. `password` を入力する
4. `Sign in` を押す
5. ログイン後の画面を snapshot で確認する

ログイン後の判定:

- ログインページに戻っていない
- `PendingMigrationError` や例外ページではない
- 通常画面の title や主要 UI が見えている

例:

- title に `実験手帳`
- 週案 / 学級別 / 月間 / 年間計画 のタブが見える

## 8. ユーザーへの報告

報告では次を必ず伝える。

- どの worktree / branch を起動したか
- どのポートを使ったか
- DB コピーをしたか
- `development.sqlite3.worktree-copy` を作ったか
- migration をしたか
- Playwright でどこまで確認したか
- 必要なら `curl` が sandbox で失敗したか
- アクセス URL

URL は少なくとも次を出す。

- `http://localhost:<PORT>`
- 必要なら `http://127.0.0.1:<PORT>`

同一 Wi-Fi 用の IP が必要ならこれで取る。

```bash
ifconfig | awk '/^[a-z0-9]/ {iface=$1} /inet / && $2 != "127.0.0.1" {print iface, $2}'
```

## よくある失敗と対処

### 1. source DB をコピーしたのに PendingMigrationError

原因:

- source repo の開発 DB 自体が今のコードに対して古い

対処:

```bash
bin/rails db:migrate
```

大事なのは、**copy した後の worktree DB に migrate を当てること**。

### 2. `curl` は失敗するのにブラウザでは開ける

原因:

- sandbox 制限

対処:

- `curl` 失敗だけでアプリ失敗と判断しない
- Playwright で `/session/new` が見えるかを優先する
- 必要なら `curl` は escalation 付きで再試行する

### 3. ログインできない

原因:

- worktree 側 DB が空
- `ikeda@example.com` が入っていない

対処:

- source repo の `development.sqlite3` を copy する
- `User.find_by(email: "ikeda@example.com")` を確認する
- その後 `db:migrate` を当てる

### 4. `server.pid` が残っていて起動できない

原因:

- 古い Puma が死んでいるのに pid ファイルだけ残っている

対処:

- `ps -p <PID>` で生死を確認する
- stale なら `tmp/pids/server.pid` を削除する
- 生きているなら、同じ worktree の Puma かどうかを確認する

### 5. 今回の worktree の古い Puma が邪魔

原因:

- 前回の確認用サーバーが残っている

対処:

- 同じ worktree の Puma なら停止してよい
- ただし kill が sandbox に弾かれることがあるので、その場合は escalation を使う
- 別 worktree の Puma は止めない

### 6. 3001〜3005 がすべて埋まっている

原因:

- 複数 worktree の Puma が並行稼働している

対処:

- 勝手に止めない
- どのポートがどの worktree に使われているかを報告する
- 必要ならユーザーに、どれを止めるか確認する

## 注意

- 「Puma が起動したはず」だけでは成功扱いにしない
- **Playwright でログイン後画面まで見てから報告する**
- `curl` は補助であって、成功条件の本体ではない
- source DB は **copy してから** 使う。symlink にしない
- DB を copy したら **毎回 migrate する**
- 古い `3001` を見て成功だと勘違いしやすいので、**必ず worktree / PID / ポートを対応づける**
