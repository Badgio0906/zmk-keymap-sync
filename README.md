# zmk-keymap-syn

## 前提条件

本アプリは **ZMK Studio 対応ファームウェア**（`CONFIG_ZMK_STUDIO=y` +
`studio-rpc-usb-uart` スニペット付きでビルドされたファームウェア）が
書き込まれたキーボードでのみ動作します。実機への即時反映（RPC書き込み・
逆読み込み）は、対応ビルドが焼かれていることが前提です。

お使いのキーボードが対応ビルドかどうかは、対象リポジトリの `build.yaml` を
確認してください（該当シールドのエントリに `snippet: studio-rpc-usb-uart`
があり、かつシールドの `.conf` に `CONFIG_ZMK_STUDIO=y` が設定されているか）。
