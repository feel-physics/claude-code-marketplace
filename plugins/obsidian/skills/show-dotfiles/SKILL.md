---
name: show-dotfiles
description: Use when dotfiles or dotfolders like .claude are invisible in Obsidian file explorer, or when needing to browse/edit .claude skills and commands from within Obsidian.
---

# Obsidianでドットフォルダを表示する

**Core principle:** Obsidianはドットフォルダを標準で非表示にし、設定変更もsymlinkも効かない。BRAT経由で `show-hidden-files` プラグインを導入するのが唯一の解決策。

## When to Use

- `.claude` フォルダがObsidianファイルエクスプローラに見えない
- スキルやコマンドをObsidian上で編集したい
- ドットフォルダ全般（`.git` 等）をObsidianで確認したい

**When NOT to use:**
- 既に `show-hidden-files` プラグインが導入済み

## Quick Reference

| 試みる方法 | 結果 |
|---|---|
| `Detect all file extensions` ON | ❌ ドットフォルダは表示されない |
| シンボリックリンク作成 | ❌ Obsidianが無視する |
| 実体フォルダ＋内部symlink | ❌ symlink先は無視される |
| **BRAT → show-hidden-files** | ✅ 唯一の解決策 |

## Implementation

### 1. BRATの確認

```bash
grep -q "obsidian42-brat" "$VAULT/.obsidian/community-plugins.json"
```

ない場合 → ユーザーに案内：
> `Cmd + ,` → Community plugins → Browse → 「BRAT」検索 → Install → Enable

### 2. プラグインインストール案内

> 1. `Cmd + P` → 「BRAT: Add a beta plugin」
> 2. 貼り付け： `polyipseity/obsidian-show-hidden-files`
> 3. Add Plugin → **Show Hidden Files** を Enable
> 4. `Cmd + R` でリロード

### 3. 検証

ユーザーに `.claude` フォルダの表示を確認してもらう。

## Common Mistakes

| ミス | 対策 |
|---|---|
| CLIから `.obsidian/plugins/` に直接配置する | community-plugins.jsonとの不整合が起きうる。BRAT経由が安全 |
| symlinkで代替しようとする | Obsidianはsymlinkを無視する。プラグイン導入が必須 |
| `.obsidian` フォルダが見えないと報告 | プラグインの仕様で常に非表示。変更不可 |
