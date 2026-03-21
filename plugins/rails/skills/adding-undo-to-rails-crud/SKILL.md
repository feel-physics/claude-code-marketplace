---
name: adding-undo-to-rails-crud
description: Use when adding a latest-first undo feature to a Rails CRUD-style app, especially for destructive edits, grouped slot-like changes, or move operations that should be reversible with snapshot-based history and real UI verification.
---

# Rails CRUD アプリに Undo を付ける

Rails の CRUD 系アプリで undo を付けるときは、**まず snapshot 方式を前提にする**。この skill の目的は、軽い latest-first undo を安全に足すことであり、event sourcing や完全監査を最初から作ることではない。

## いつ使うか

- 「元に戻す」「undo」「直前の操作を取り消す」を入れたいとき
- create / update / destroy だけでなく、`move` のように **1 回で複数レコードが変わる** 操作があるとき
- まずは **最新 1〜20 回** の巻き戻しで十分なとき
- 実装の単純さを優先したいとき

使わないほうがよい場面:

- redo まで最初から必要なとき
- 全変更の監査ログが必要なとき
- 非同期ジョブや外部 API まで巻き戻したいとき

## コアパターン

**Before**

- controller / service がそのまま対象レコードを更新する
- 戻すには逆操作ロジックを個別に書く

**After**

- `*_actions` に「1 回の操作」を記録する
- `*_snapshots` に「その操作で変わる各レコードの before 状態」を記録する
- undo は最新 action を 1 件取り、snapshot を transaction で復元する

最小原則:

- **1 UI 操作 = 1 action**
- **変わったレコードごとに 1 snapshot**
- **undo は before 状態へ戻す**

## クイックリファレンス

| 状況 | 基本方針 |
| --- | --- |
| 単純な update | `action 1件 + snapshot 1件` |
| move / swap / 上書き | `action 1件 + snapshot 2〜3件` |
| sparse な表 | locator と `existed` を持つ |
| 子レコードも戻す | before の集合を JSON で持ち、undo 時に全置換 |
| 最小構成 | redo なし、latest-first、最新 20 件 |

## 実装手順

1. **更新入口を洗い出す**  
   `rg` で controller / service / model を見て、どこが本当に `create/update/destroy` しているかを特定する。単純ケースと複数変更ケースを分ける。

2. **境界を先に固定する**  
   何を undo 対象にし、何を対象外にするかを決める。最初は広げすぎない。  
   このプロジェクトでは、まず `design.md` と `tasklist.md` を作り、`requirements.md` は省く。

3. **snapshot 方式で設計する**  
   基本は親 `*_actions` と子 `*_snapshots`。  
   snapshot には少なくとも次を持たせる:
   - locator
   - `existed`
   - 戻す属性
   - 必要なら子レコード集合の JSON

4. **記録は callback ではなく use case 入口で行う**  
   更新後ではなく**更新前**に snapshot を採る。`record_*_action` のような共通入口で、変更対象 location を明示して束ねる。

5. **undo service は単純に保つ**  
   最新 action を 1 件取得し、transaction で snapshot を復元する。  
   `existed=true` なら戻す。`existed=false` なら現在の row を消す。  
   子レコードは差分計算より**全置換**を優先する。

6. **単純ケースから積み上げる**  
   まず add / remove / update を通す。  
   そのあと move のような複数 snapshot ケースを載せる。

7. **最後は実画面で確認する**  
   テストが通っても終わりにしない。実際に UI から操作して、undo が期待どおり戻ることを確かめる。  
   このリポジトリでは必要に応じて [verify-local-rails-runtime](../verify-local-rails-runtime/SKILL.md) を使う。

## よくある失敗

- **event sourcing に飛びつく**  
  最初は重すぎる。まず snapshot で足りるかを見る。

- **更新後に履歴を取る**  
  undo の材料が失われる。必ず before を取る。

- **1 レコード単位でしか考えない**  
  move のような複数変更を 1 action に束ねないと、undo が壊れる。

- **子レコードの戻し方を曖昧にする**  
  差分計算より before 集合の全置換を優先する。

- **テストだけで完了扱いにする**  
  最後は UI から実際に「戻せる」ことを確認する。

## 成功の形

よい undo 実装は、次の状態になっている。

- 実装が単純で説明しやすい
- 単純ケースが先に通っている
- 複数変更ケースも同じ stack に載る
- UI から実際に戻せる
- 「どこまで戻すか」が design.md と tasklist.md に明記されている
