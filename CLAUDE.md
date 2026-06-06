# ZMK キーマップ同期アプリ - Claude Code 指示書

## このファイルについて

このファイルはClaude Codeがプロジェクト全体を通じて参照する指示書です。
開発の方針・制約・進め方を記載しています。

---

## プロジェクト概要

ZMKファームウェアを使用する自作キーボードのキーマップを、
WebアプリのUIから変更した際に以下を同時実行するアプリを開発する：

- **即時バイナリ**：ZMK Studio RPCプロトコル経由で実機フラッシュを直接書き換え
- **ビルドバイナリ**：GitHubにソースをpushしてGitHub Actionsでビルド

詳細は `docs/requirements.md` を参照すること。

---

## 最初にやること（T-1検証）

**まずこれだけを作る：**

HTMLファイル1枚で以下を確認する検証コードを作成する。

1. WebBluetooth経由でZMK Studio RPCプロトコルに接続できるか
2. 接続後、実機から現在のキーマップを1つ読み取れるか
3. 実機のキーを1つ変更できるか

使用するライブラリ：`@zmkfirmware/zmk-studio-ts-client`

検証結果によって以下を判断する：
- **可**：そのままWebアプリ開発へ進む
- **否**：TauriまたはElectronへの切り替えを検討する（`docs/requirements.md`のT-1参照）

---

## 技術方針

- **アプリ形態**：Webアプリ
- **対応ブラウザ**：Chrome / Edge（PC・Android）、Bluefy等（iOS）
- **接続方式**：WebSerial（USB）またはWebBluetooth（BLE）
- **GitHub連携**：GitHub OAuth + GitHub API
- **ビルド**：GitHub Actions
- **RPC通信**：`@zmkfirmware/zmk-studio-ts-client`（公式TSクライアント）

---

## 開発の進め方

1. 勝手に大きく作り始めない。段階的に確認しながら進める
2. 各フェーズの完了時に動作確認を求めること
3. 技術的な判断が必要な場面では選択肢を提示してから進める
4. 要件定義書にない機能は勝手に追加しない。追加が必要な場合は確認する

---

## フェーズ構成

### Phase 1：T-1検証（最初にやること）
- WebBluetooth/WebSerial経由でZMK Studio RPCの読み書きができるか確認
- 成果物：`test/t1_verification.html`

### Phase 2：基本構成
- GitHub OAuth認証
- リポジトリの`.keymap`読み込みとUI表示
- 実機からの逆読み込み

### Phase 3：キーマップ編集
- キーの視覚的編集UI
- 保存ボタン（即時バイナリ＋GitHubプッシュの同時実行）

### Phase 4：ビヘイビア管理
- `.conf`読み込みと未作成ビヘイビア一覧表示
- ビヘイビア専用作成画面
- 解説ページ

### Phase 5：仕上げ
- ビルドステータス表示
- エラー通知・リトライ案内
- iOS向けブラウザ案内

---

## 対象外（やらないこと）

- ボード定義・デバイスツリーの変更
- `.conf`の依存関係の自動解決
- ビルドバイナリの実機への自動フラッシュ
- QMK・Via等ZMK以外のファームウェア対応
- ZMK Studio非対応キーボードへの対応
