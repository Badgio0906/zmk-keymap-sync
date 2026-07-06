# ZMK Keymap Sync

ZMKファームウェアを使う自作キーボードのキーマップを、Webブラウザから変更するアプリ。
実機への即時RPC書き込みとGitHubへのソース同期を同時に行う。

## 使い方

### 公開URL（推奨）

**https://badgio0906.github.io/zmk-keymap-sync/**

インストール不要。Chrome / Edge でアクセスするだけで利用できる。
HTTPS 配信のため WebSerial（USB 接続）も制限なく動作する。

### ローカル起動（開発者向け）

```powershell
npx serve app
# → http://localhost:3000/index.html
```

`file://` では WebSerial が動作しないため、必ず HTTP サーバー経由でアクセスすること。

## 前提条件

本アプリは **ZMK Studio 対応ファームウェア**が書き込まれたキーボードでのみ動作する。

- `CONFIG_ZMK_STUDIO=y` でビルドされていること
- `studio-rpc-usb-uart` スニペットが適用されていること

対象リポジトリの `build.yaml` で該当シールドのエントリに `snippet: studio-rpc-usb-uart`
があり、かつシールドの `.conf` に `CONFIG_ZMK_STUDIO=y` が設定されているかを確認する。

詳細な利用仕様は [docs/usage_spec.md](docs/usage_spec.md) を参照。

## 技術スタック

- 単一 HTML ファイル（`app/index.html`）、外部ライブラリなし
- WebSerial（USB）経由で ZMK Studio RPC プロトコルを直接実装
- GitHub REST API v3 で `.keymap` ファイルを読み書き
- 対応ブラウザ: Chrome / Edge（PC・Android）
