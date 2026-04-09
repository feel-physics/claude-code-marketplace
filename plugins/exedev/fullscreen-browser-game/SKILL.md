---

name: fullscreen-browser-game

description: ブラウザゲームを全画面化する。iframe方式HTMLラッパーでFullscreen APIを使いスマホで全画面プレイを実現。

---
# ブラウザゲームを全画面化する
## 方式
iframe方式。ゲームHTMLをiframe埋め込みするindex.htmlを生成。
## 構成
- `templates/wrapper.html` — `{{PYXEL_HTML_SRC}}`をiframe srcに置換
- `index.html` — 生成ラッパー。ルートURLで配信
- 元HTML — iframe内で動作。直リンク維持
## 全画面ボタン
- タッチデバイスのみ（`ontouchstart`/`maxTouchPoints`）
- ひらがな文言（例:「ぜんがめんであそぶ」）
- `fixed; bottom:20px; left:50%` 半透明
- 5秒後フェードアウト。iframe外タップで再表示
## Fullscreen API
- 対象: `document.documentElement`
- `requestFullscreen`→`webkitRequestFullscreen`順
- `fullscreenchange`/`webkitfullscreenchange`でボタン制御
- 非対応時: ひらがなで「ホームがめんについか」案内
## CSS
- iframe: `100vw/100vh; border:none`
- body: `margin:0; background:#000; overflow:hidden`
- アスペクト比維持、余白黒
## ビルド
ビルドスクリプト後処理でテンプレート→変数置換→index.html出力。