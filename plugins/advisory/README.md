# Advisory Plugin

Advisory dialogue skills for Claude Code / Codex CLI.

## Skills

| Skill | Description |
|-------|-------------|
| [discuss-with-adviser](skills/discuss-with-adviser/) | 迷っている判断を、主催キャラと相談相手の二者対話で整理する。反論を必ず入れ、最後に結論または判断軸を返す |

## Usage

```
/discuss-with-adviser [A について] [B と]
```

`A` は話題、`B` は相手。どちらも省略できる。省略した場合は直前の会話から話題を推定し、相手は候補を出して承認を取る。

## Optional companions

このスキルは、次のものが手元にあれば使い、無ければ使わずに進む。**どれも必須ではない。**

| 名前 | 役割 | 無いとき |
|------|------|----------|
| `*-tone` スキル | 主催キャラの口調 | 通常の丁寧な口調で進む |
| `check-trusted-posts/lists/` | 相手候補の発信者リスト | AskUserQuestion で検索方法を選ばせる |
| `find-trusted-voices` スキル | 発信者の探索 | 通常のWeb検索で代替 |
| `advisers.yml` | 人物像の台帳（スキルフォルダ直下） | 台帳なしで進む。作りたければ `advisers.yml.example` をコピーして使う |

`advisers.yml` は利用者個人のメモなので**このプラグインには同梱していません**。実在の人物を相手にするときは、本人が実際に述べたことと対話用の推測を混ぜないでください。
