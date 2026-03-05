# show-obsidian-dotfiles

Obsidianのファイルエクスプローラでドットフォルダ（`.claude` 等）を表示するためのガイドスキル。

## Problem

Obsidianは標準ではドットフォルダを非表示にし、設定変更もシンボリックリンクも効きません。

## Solution

BRAT経由で `show-hidden-files` プラグインをインストールすることで解決します。

## Install

```
/plugin marketplace add https://github.com/feel-physics/claude-code-marketplace.git
/plugin install show-obsidian-dotfiles@feel-marketplace
```

## License

MIT
