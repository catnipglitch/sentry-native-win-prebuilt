# CLAUDE.md — sentry-native-win-prebuilt

このリポジトリは `getsentry/sentry-native` の Windows プリビルドバイナリを配布する **CI 専用リポジトリ**（fork ではない独立リポジトリ）。上流ソースは本リポジトリに含めず、ビルド時のみ fresh clone する。

## 上流への送信は厳禁（最優先ルール）

このリポジトリの作業から **`getsentry/sentry-native` に対していかなる送信操作も行ってはならない**。

禁止される操作（絶対に実行しないこと）:

- `git push` で上流リポジトリへ送信（URL に `getsentry/sentry-native` を含む push は全て）
- `gh pr create` で上流を base にした Pull Request 作成
- `gh pr create --repo getsentry/sentry-native` 形式の fork PR
- `gh issue create -R getsentry/sentry-native` など Issue 作成
- `gh issue comment` / `gh pr comment` / `gh pr review` の上流リポジトリへのコメント送信
- `gh api` での上流リポジトリに対する書き込み系リクエスト（POST/PATCH/PUT/DELETE）
- 上流の Discussion / Release / Wiki への投稿

許可される操作:

- `git clone https://github.com/getsentry/sentry-native.git` など **読み取り専用**の clone / fetch
- `gh api` の GET リクエスト（upstream 最新タグ取得 等）
- `gh release list -R getsentry/sentry-native`（情報取得のみ）
- `origin`（= `catnipglitch/sentry-native-win-prebuilt`）への push, Release 作成, Issue/PR 作成, secret 登録

**Why:** このリポジトリは上流の改変提案・バグ報告を行う立場にない（単にビルドを配布しているだけ）。上流コミュニティにノイズや意図しない改変提案が飛ぶのを防ぐため。

**How to apply:**

- shell・ワークフローで push 系コマンドを書くときは常にリポジトリ URL／ref を目視確認
- 新規ワークフロー追加時、`permissions:` に `pull-requests: write` を付けない
- `gh pr create` を書く前に base リポジトリが `catnipglitch/sentry-native-win-prebuilt` であることを明示確認
- 疑わしい場合はユーザーに確認してから実行

## リポジトリ構成メモ

- 既定ブランチ: `main`
- 上流ソース: **本リポジトリにコミットしない**。`build.yml` 実行時のみ CI 上で `git clone --branch <upstream-tag>` する
- リリースタグ形式: `v<upstream-version>-win`（例: 上流 `0.13.7` → `v0.13.7-win`）
- ワークフロー（`.github/workflows/`）:
  - `sync.yml` — 月次で上流の最新タグを検出し、自リポジトリに `v<ver>-win` タグを発行
  - `build.yml` — タグ push でトリガ、上流を fresh clone してビルド、アーティファクト化
  - `release.yml` — build 成功後に GitHub Release 作成
- 自動連鎖のため `ACTIONS_PAT` secret に Fine-grained PAT を登録して `sync.yml` の tag push で使う
