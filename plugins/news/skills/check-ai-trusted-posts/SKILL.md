---
name: check-ai-trusted-posts
description: AI分野の信頼できる発信者の最近の発信内容をチェックする。「最近のAI動向は？」「有名人は何を言ってる？」等のリクエスト時に使用。
---

# check-ai-trusted-posts

AI分野の信頼できる発信者6名の直近2週間の発信内容を検索・要約し、テキストで報告する。

## 使い方

- `/check-ai-trusted-posts` → 6名全員の直近発信をチェック
- `/check-ai-trusted-posts --format 30-80` → 発信内容の文字数を30-80字に変更

## 発信者リスト（2026-03-05調査、3モデルクロスチェック済み）

| Tier | 名前 | 主要媒体 | 検索キーワード |
|------|------|----------|--------------|
| 1 | Ethan Mollick | One Useful Thing | `Ethan Mollick One Useful Thing` |
| 1 | Simon Willison | simonwillison.net | `Simon Willison blog` |
| 2 | Andrej Karpathy | X @karpathy | `Andrej Karpathy` |
| 2 | Andrew Ng | The Batch / deeplearning.ai | `Andrew Ng The Batch` |
| 2 | Swyx | Latent Space Podcast & Substack | `Swyx Latent Space` |
| 2 | 清水亮 | note / X @shi3z | `清水亮 shi3z AI` |

リストを更新する場合はこのSKILL.mdを直接編集する。

## 実行仕様

### ステップ1：検索

各発信者の直近2週間の発信を検索する。
Tier1を先に、Tier2を後に処理する。

### ステップ2：フォーマット

各発信者について以下の形式でまとめる：

```
#### {Tier}: {名前}（{主要媒体}）
立場: {肩書・専門 10-20字}
1. {日付} {発信内容の要約 20-50字}({リンク})
```

- 立場: デフォルト10-20字（`--format` の第1引数で変更可）
- 発信内容: デフォルト20-50字（`--format` の第2引数で変更可）
- 「何を主張しているか」を書く（記事タイトルの転記ではない）
- 直近に発信がない場合「直近2週間の新規記事なし」と記録する

### ステップ3：全体トレンド

発信者間で共通するトレンド・注目トピックを3行以内でまとめる。

## 注意

- 「更新なし」も有用な情報として必ず記録する
