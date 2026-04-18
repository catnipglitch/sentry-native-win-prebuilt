# operations — 運用ドキュメント

## このリポジトリの位置づけ

`catnipglitch/sentry-native-win-prebuilt` は `getsentry/sentry-native` の **Windows 向けプリビルドバイナリを配布するための CI 専用リポジトリ**。fork ではなく独立リポジトリ。上流ソースはここにはなく、各ビルド時に CI 上で fresh clone する。

## ブランチ・タグ設計

| 項目 | 値 |
|---|---|
| 既定ブランチ | `main` |
| リリースタグ | `v<upstream-version>-win`（例: 上流 `0.13.7` → `v0.13.7-win`） |
| タグ採番 | 上流最新 semver に追従（中間バージョンは基本スキップ） |

## ワークフロー全体像

```
  ┌──────────────────────────────────┐
  │ schedule (毎月1日 03:00 UTC)       │
  │ または手動 workflow_dispatch      │
  └──────────────┬───────────────────┘
                 ▼
      sync.yml
        ・上流 getsentry/sentry-native の最新 semver タグを検出
        ・自リポジトリに v<ver>-win タグが無ければ作成
        ・PAT (ACTIONS_PAT) で tag push
                 ▼
            （tag push イベント）
                 ▼
      build.yml
        ・タグから upstream バージョンを抽出
        ・upstream を fresh clone（submodules 含む）
        ・CMake + MSBuild で Release ビルド
        ・manifest.json / SHA256SUMS.txt を同梱
        ・ZIP をアーティファクトとして upload
                 ▼
         （workflow_run: completed）
                 ▼
      release.yml
        ・アーティファクトを download
        ・GitHub Release を作成（ZIP 添付）
```

## 各ワークフロー詳細

### `sync.yml`

- **トリガ**: `schedule` (毎月1日 03:00 UTC) / `workflow_dispatch`
- **使用 secret**: `ACTIONS_PAT`（Fine-grained PAT, Contents: write, Actions: read）
- **処理**:
  1. PAT を使って自リポジトリを checkout
  2. `gh release list -R getsentry/sentry-native` で上流 semver タグ一覧を取得、最新を抽出
  3. `v<ver>-win` が自リポジトリに既にあれば何もせず終了
  4. 無ければ annotated tag を作り PAT で push（PAT push なら `push: tags` が発火する）

### `build.yml`

- **トリガ**: `push: tags: ['v*-win']` / `workflow_dispatch`（`upstream_ref` 入力）
- **ランナー**: `windows-latest`
- **処理**:
  1. 自リポジトリを checkout（ほぼ workflow ファイルだけ）
  2. タグ名から upstream バージョンを抽出（例: `v0.13.7-win` → `0.13.7`）
  3. `git clone --depth 1 --branch <ver> --recurse-submodules --shallow-submodules https://github.com/getsentry/sentry-native.git upstream-src`
  4. CMake configure（下記オプション）→ `cmake --build` → `cmake --install --prefix ../install`
  5. `install/manifest.json` にビルドメタを書く
  6. `install/SHA256SUMS.txt` に全ファイルの SHA256 を書く
  7. `install/` を `sentry-native_v<ver>-win_win11_x64_Release_MD_static.zip` に圧縮
  8. `actions/upload-artifact@v4` でアップロード（保持 14 日）

- **CMake オプション**
  | オプション | 値 |
  |---|---|
  | Generator | `Visual Studio 17 2022` |
  | `CMAKE_BUILD_TYPE` | `Release` |
  | `BUILD_SHARED_LIBS` | `OFF` |
  | `SENTRY_BUILD_SHARED_LIBS` | `OFF` |
  | `SENTRY_BUILD_RUNTIMESTATIC` | `OFF` (= `/MD`) |
  | `SENTRY_BACKEND` | `crashpad` |
  | `SENTRY_TRANSPORT` | `winhttp` |
  | `SENTRY_BUILD_TESTS` | `OFF` |
  | `SENTRY_BUILD_EXAMPLES` | `OFF` |

### `release.yml`

- **トリガ**: `workflow_run` が `"Build Windows prebuilt"` 完了時
- **発火条件**
  ```yaml
  if: >
    github.event.workflow_run.conclusion == 'success' &&
    github.event.workflow_run.event == 'push' &&
    startsWith(github.event.workflow_run.head_branch, 'v')
  ```
  `event == 'push'` 条件により、**手動 (`workflow_dispatch`) Build では発火しない**。自動チェーンは「タグ push で Build が起動したとき」のみ機能する。
- **処理**: artifact を download → `softprops/action-gh-release@v2` で Release 作成（`generate_release_notes: true`）

## PAT セットアップ

自動連鎖には Personal Access Token が必要（`GITHUB_TOKEN` による tag push は他 workflow を発火させない仕様のため）。

1. GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → *Generate new token*
2. 設定:
   - Token name: `sentry-native-win-prebuilt-ci`
   - Expiration: 90 日（切れる前に更新）
   - Repository access: **Only select repositories** → `catnipglitch/sentry-native-win-prebuilt`
   - Permissions:
     - Contents: **Read and write**
     - Actions: **Read**
     - Metadata: Read-only（自動付与）
3. 発行された token を secret に登録:
   ```bash
   gh secret set ACTIONS_PAT -R catnipglitch/sentry-native-win-prebuilt --body "<TOKEN>"
   ```
4. token は一度しか表示されないので控えるか、パスワードマネージャに保存

**PAT 期限切れ時**: sync ワークフローが checkout 失敗して exit する。更新して同名 secret を再設定するだけ。

## 運用 Runbook

### 正常系（毎月自動）

1. 毎月 1 日 03:00 UTC に `sync.yml` が起動
2. 上流の新バージョンがあれば `v<ver>-win` タグを発行
3. 即時 `build.yml` 発火 → Windows ビルド
4. 成功で `release.yml` 発火 → Release 作成
5. 翌日 Releases ページを見れば最新の ZIP が出ている

### 手動実行

**上流監視から**
```bash
gh workflow run sync.yml -R catnipglitch/sentry-native-win-prebuilt
```

**任意バージョンのビルド（上流に存在するタグなら可）**
```bash
gh workflow run build.yml -R catnipglitch/sentry-native-win-prebuilt \
  -f upstream_ref=0.12.8
```
※ workflow_dispatch で起動した build では release.yml は自動発火しない。手動で release 作成が必要。

**手動 Release 作成（手動 build の後）**
```bash
# run_id は gh run list で確認
gh run download <RUN_ID> -R catnipglitch/sentry-native-win-prebuilt -D /tmp/rel
gh release create v0.12.8-win -R catnipglitch/sentry-native-win-prebuilt \
  --title "v0.12.8-win" --generate-notes \
  /tmp/rel/sentry-native_*/sentry-native_*_win11_x64_Release_MD_static.zip
```

## 上流送信禁止ルール

詳細は [`../CLAUDE.md`](../CLAUDE.md) 参照。要約: このリポジトリの作業から `getsentry/sentry-native` への push / PR / Issue / コメント / API 書き込みは一切禁止。`git clone` / `gh release list` / `gh api` GET のみ許可。

## トラブルシューティング

### sync.yml が 401 で失敗する
- 原因: `ACTIONS_PAT` 期限切れまたは権限不足
- 対応: PAT を新規発行し `gh secret set ACTIONS_PAT ... --body "<TOKEN>"` で再登録

### build.yml の CMake configure で失敗
- 原因候補: 上流が CMake 要件を上げた / crashpad サブモジュール初期化失敗 / 必要なツールが runner に無い
- 対応: `gh run view <RUN_ID> --log-failed` でログ確認。上流の CI 設定を参照して差分を build.yml に反映

### build は成功したが release ができない
- 原因: workflow_dispatch で起動した build は release.yml の条件 (`event == 'push'`) を満たさない
- 対応: Runbook の「手動 Release 作成」に従う

### upstream の tag が semver じゃない（rc など）
- 現 sync.yml は `^[0-9]+\.[0-9]+\.[0-9]+$` で厳密フィルタする。RC や pre-release はスキップされる設計
- 必要なら `build.yml` の workflow_dispatch で明示的に ref を指定して手動ビルド

## ファイル配置

```
.
├── CLAUDE.md                       # 上流送信禁止ルール
├── README.md                       # 利用者向け概要
├── LICENSE                         # MIT (上流と同じ)
├── .gitignore
├── docs/
│   └── operations.md               # このファイル
└── .github/workflows/
    ├── sync.yml
    ├── build.yml
    └── release.yml
```
