# sentry-native-win-prebuilt

`getsentry/sentry-native` の Windows 向けプリビルドバイナリを **GitHub Releases** で配布するための CI 専用リポジトリ。

## これは何か

- **目的**: ゲーム等の C/C++ 開発プロジェクトにすぐ drop-in できる `sentry.lib` を提供
- **配布物**: Windows 11 x64 / MSVC 2022 Release / `/MD` ランタイム / crashpad backend / winhttp transport の **静的ライブラリ**
- **リリース形式**: 上流の各 semver リリース (`0.13.7` 等) に対応して `v<upstream>-win` タグを発行（例: `v0.13.7-win`）
- **ソースコードは本リポジトリに含まれない**。各ビルドは CI 上で上流 `getsentry/sentry-native` の該当タグを `git clone` して作る

## ダウンロード

[Releases ページ](https://github.com/catnipglitch/sentry-native-win-prebuilt/releases) から対応バージョンの ZIP を取得。

ZIP 内容:

| パス | 内容 |
|---|---|
| `include/sentry.h` | 公開ヘッダ |
| `lib/sentry.lib` | 静的ライブラリ（crashpad + winhttp 同梱） |
| `lib/cmake/sentry/` | CMake の `find_package(sentry)` 用 config |
| `bin/crashpad_handler.exe` | 別プロセスクラッシュハンドラ（アプリと同梱配布が必要） |
| `manifest.json` | ビルドメタ（upstream SHA, VS バージョン, オプション等） |
| `SHA256SUMS.txt` | 全ファイルの SHA256 ハッシュ |

## 自分のプロジェクトで使う（最小例）

```cmake
# CMakeLists.txt
list(APPEND CMAKE_PREFIX_PATH "path/to/unzipped")
find_package(sentry REQUIRED)
target_link_libraries(my_game PRIVATE sentry::sentry)
# 配布時は bin/crashpad_handler.exe を my_game.exe と同じディレクトリにコピー
```

## ライセンス

本リポジトリで配布するバイナリは上流 `getsentry/sentry-native` と同じ **MIT License**（[LICENSE](LICENSE) 参照）。
本リポジトリ自体のビルドスクリプト・ドキュメントも同 MIT で提供する。

## 運用ドキュメント

リリース自動化の仕組み・手動リカバリ手順は [`docs/operations.md`](docs/operations.md) を参照。
