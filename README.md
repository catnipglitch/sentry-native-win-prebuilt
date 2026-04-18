# sentry-native-win-prebuilt

[getsentry/sentry-native](https://github.com/getsentry/sentry-native) を **Windows 11 x64 / MSVC 2022** でビルド済みにして GitHub Releases で配布している、非公式のプリビルドリポジトリです。

C/C++ プロジェクト（特にゲーム開発）で Sentry を組み込みたいとき、上流を clone して CMake を走らせて…という手間を省けます。ZIP を解凍するだけで `find_package(sentry)` が通る状態になります。

## こんなときに便利

- MSVC / Windows SDK のセットアップが整っていない CI 環境で、素早く Sentry を導入したいとき
- CMake・crashpad・winhttp の組み合わせを自分で検証する時間を節約したいとき
- ゲームエンジン本体のビルドとは独立に、Sentry のバージョンを固定したいとき

## リリース形式

上流 `getsentry/sentry-native` の semver リリース（例: `0.13.7`）に追従して、
`v<upstream>-win` 形式のタグで Release を発行しています（例: [`v0.13.7-win`](https://github.com/catnipglitch/sentry-native-win-prebuilt/releases/tag/v0.13.7-win)）。

配布バイナリの構成は以下で固定しています:

| 項目 | 値 |
|---|---|
| OS / アーキテクチャ | Windows 11 x64 |
| コンパイラ | MSVC 2022 (Visual Studio 17.14+, toolset v143) |
| ビルド構成 | Release |
| C ランタイム | `/MD`（動的 CRT） |
| ライブラリ形式 | 静的ライブラリ（`sentry.lib`） |
| バックエンド | crashpad |
| トランスポート | winhttp |

他の構成（`/MT`、Debug ビルド、32-bit など）は今のところ配布していません。
もし必要な構成があれば、Issue で相談してもらえると嬉しいです。

## ダウンロード

[Releases ページ](https://github.com/catnipglitch/sentry-native-win-prebuilt/releases) に
`sentry-native_v<version>-win_win11_x64_Release_MD_static.zip`（約 3 MB）を置いています。

ZIP の中身:

| パス | 内容 |
|---|---|
| `include/sentry.h` | 公開ヘッダ |
| `lib/sentry.lib` | 静的ライブラリ（crashpad + winhttp 同梱） |
| `lib/cmake/sentry/` | `find_package(sentry)` 用の CMake config |
| `bin/crashpad_handler.exe` | 別プロセスのクラッシュハンドラ（アプリと同梱配布が必要） |
| `manifest.json` | ビルドメタ情報（upstream SHA, VS バージョン, 有効オプション等） |
| `SHA256SUMS.txt` | 全ファイルの SHA256 ハッシュ |

## 使いかた（最小例）

1. ZIP を任意のディレクトリに解凍する
2. `CMakeLists.txt` で `find_package(sentry)` を呼ぶ

   ```cmake
   list(APPEND CMAKE_PREFIX_PATH "path/to/unzipped")
   find_package(sentry REQUIRED)

   target_link_libraries(my_game PRIVATE sentry::sentry)
   ```

3. 配布するときは `bin/crashpad_handler.exe` を実行ファイルと同じディレクトリに置く
   — crashpad は別プロセスでクラッシュを捕まえる仕組みなので、ハンドラの同梱配布が必要になります

Sentry SDK 自体の初期化方法や DSN の設定は [上流ドキュメント](https://docs.sentry.io/platforms/native/) に案内があります。本リポジトリはビルド済みバイナリを提供するだけで、ランタイム API は上流とまったく同じです。

## これは上流の fork ではありません

このリポジトリには上流 `getsentry/sentry-native` の **ソースコードは含まれていません**。
CI がタグ発行のたびに上流を fresh clone してビルドしています。

そのため:

- 上流への Pull Request・Issue は [本家 getsentry/sentry-native](https://github.com/getsentry/sentry-native) へどうぞ（このリポジトリは配布インフラなので、上流の変更提案は扱っていません）
- ソースコードの変更履歴は上流リポジトリ側にあります
- ここは「上流をビルドして配布するだけ」の CI インフラという位置づけです

## ライセンス

配布バイナリは上流と同じ **MIT License** です（[LICENSE](LICENSE) 参照）。
本リポジトリのワークフローとドキュメントも同じく MIT で提供しています。

## 運用・自動化の仕組み

月初に上流の最新 semver タグを自動検出して、タグが新しければ自リポジトリに `v<version>-win` タグを発行 → ビルド → Release 作成、という流れで連鎖します。
詳細な仕組みと手動リカバリ手順は [`docs/operations.md`](docs/operations.md) にまとめています。
