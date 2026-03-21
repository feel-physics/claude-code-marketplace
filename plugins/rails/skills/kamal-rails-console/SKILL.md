---
name: kamal-rails-console
description: Kamalデプロイ環境の本番Railsコンソールでデータ操作を実行する。本番データの修正・確認・削除リクエスト時に使用。
---

# Kamal Rails Console操作

Kamal経由で本番RailsコンソールにRubyコードを送り、データの確認・修正・削除を行う。

## 使い方

- `/kamal-rails-console` → 対話的に操作内容を確認しながら実行
- `/kamal-rails-console 〇〇のデータを確認して` → 指定操作を実行

## 実行仕様

### 基本コマンド形式

```bash
kamal app exec 'bin/rails runner "Rubyコード"'
```

### シェルエスケープの制約と回避策

kamal経由でRubyコードを渡す際、以下の制約がある：

| 問題 | 原因 | 回避策 |
|------|------|--------|
| `!` がエスケープされる | bashの履歴展開 | `save` を使う（`save!` → `save`） |
| ダブルクォート内の文字列 | シェル展開 | `%(文字列)` リテラルを使う |
| 長すぎるワンライナー | ARG_MAX制限 | 処理を分割して複数回実行 |
| stdin経由のスクリプト | Docker execで届かない | ワンライナーで直接渡す |

### 実行手順

#### 1. 現状確認（必ず最初に行う）

データ修正の前に、必ず現在の状態を確認するクエリを実行する。

```bash
kamal app exec 'bin/rails runner "puts User.count"'
```

#### 2. 削除操作

外部キー制約を考慮し、依存先から順に削除する。

```ruby
# 例：子テーブル → 親テーブルの順
LessonMaterial.where(lesson_id: ids).delete_all  # 子を先に
Lesson.where(user: u).delete_all                  # 親を後に
```

#### 3. データ投入

データをRuby配列リテラルとして直接埋め込む。CSVファイルは本番コンテナから読めないため。

```ruby
data = [[%(江戸),1,%(江戸城炎上)], [%(戦国),2,%(火計)]]
data.each { |name,pos,title| ... }
```

#### 4. 投入後の検証（必須）

投入後に必ずカウントと内容を確認する。

```bash
kamal app exec 'bin/rails runner "puts Model.count"'
```

### コード量が多い場合の分割戦略

1回のコマンドに収まらない場合、論理的な単位で分割する：

- テーブルごと（Lesson → LessonMaterial）
- データ量ごと（前半・後半）

各分割実行後に件数を確認し、次に進む。

## 注意

- **本番環境の操作である**ことを常に意識する。実行前にユーザーに確認を取る
- **`delete_all` と `destroy_all` の使い分け**：コールバック不要なら `delete_all`（高速）、コールバック必要なら `destroy_all`
- **`find_or_create_by!` 等の `!` メソッドは使えない**。`find_by` + `save` で代替する
- **トランザクションはワンライナー単位**。分割実行の途中で失敗した場合、手動でロールバックが必要
- **`--reuse` オプション**：既存コンテナを再利用する（速い）。新コンテナが必要な場合は省略
