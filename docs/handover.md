# ZMK Keymap Sync — プロジェクト引き継ぎ書

作成日: 2026-07-03  
リポジトリ: https://github.com/Badgio0906/zmk-keymap-sync  
対象キーボード（デフォルト）: **roBa**（Badgio0906/zmk-config-roBa）

---

## 1. プロジェクト概要

ZMKファームウェアを使う自作キーボードのキーマップを、WebアプリUIから変更する。  
変更時に以下を**同時実行**して「結果的同期」の状態を保つ：

- **即時バイナリ**：ZMK Studio RPCプロトコル（WebSerial/WebBluetooth）経由で実機フラッシュを直接書き換え
- **ビルドバイナリ**：GitHubに `.keymap` を push → GitHub Actions でビルド（実機には焼かない・ソース正確性の担保目的）

### 背景

| ツール | できること | できないこと |
|---|---|---|
| Keymap Editor | 複雑なビヘイビア設定 | 実機への即時反映 |
| ZMK Studio | 実機への即時反映 | 複雑なビヘイビア設定 |

このアプリは両者の欠点を補完する。

---

## 2. 技術スタック

- **形態**: 純粋 Webアプリ（バックエンドなし）、単一 HTML ファイル
- **ファイル**: `app/index.html`（全実装を1ファイルに収容）
- **対応ブラウザ**: Chrome / Edge（PC・Android）、Bluefy等（iOS）
- **接続**: WebSerial（USB）／WebBluetooth（BLE）
- **GitHub連携**: Personal Access Token（PAT）＋ GitHub REST API v3
- **RPC通信**: 独自バイトスタッフィング ＋ Protobuf（手書きエンコーダ/デコーダ）
- **状態管理**: `localStorage`（PAT、キーボード設定）＋ グローバル `S` オブジェクト
- **依存ライブラリ**: なし（外部ライブラリ未使用）

### ZMK Studio RPCプロトコル（重要）

`@zmkfirmware/zmk-studio-ts-client` は**使用していない**。  
独自の最小実装で直接プロトコルを実装している。

**フレーミング（独自バイトスタッフィング）:**
```
0xAB = Start of Frame (SOF)
0xAC = Escape byte (ESC)
0xAD = End of Frame (EOF)
データ中に0xAB/0xAC/0xADが現れたら [0xAC, byte] にエスケープ
```

**Protobuf フィールド番号:**
- Request: field1=requestId, field5=keymap(KeymapRequest)
- KeymapRequest: field1=getKeymap(bool), field2=setLayerBinding
- SetLayerBindingRequest: field1=layerId, field2=keyPosition, field3=binding(BehaviorBinding)
- BehaviorBinding: field1=behaviorId(sint32/zigzag), field2=param1, field3=param2
- RpcResponse: field1(wire2) = RequestResponse { field1=requestId, field5=KeymapResponse }
- KeymapResponse: field1=GetKeymapResponse { field1=layers(repeated) }

**GetKeymap リクエスト例:**
```
送信: [ab 08 01 2a 02 08 01 ad]
応答から roBa_R の全レイヤー・全キーの behaviorId/param1/param2 が得られる
```

---

## 3. 現在の実装状態（`app/index.html`）

### 3-1. 実装済み機能

#### GitHub 連携
- PAT（Personal Access Token）による認証（localStorage 保存）
- GitHub API 経由で `.keymap` と レイアウト JSON を取得
- PAT 未設定時は設定モーダルを自動表示
- GitHub に `.keymap` を PUT（コミットメッセージ付き）
- ファイルの SHA 管理（競合防止）

#### キーボード管理
- 複数キーボードの切り替え（localStorage に設定を保存）
- キーボードの追加・編集・削除モーダル
- GitHub URL 貼り付けで owner/repo/branch を自動解析
- 「自動検出」ボタンで `.keymap` と `.json` のパスを GitHub から自動取得

#### キーマップ表示・編集
- `.keymap` ファイルのパース（`keymap { }` → レイヤー → `bindings = < >` 抽出）
- KLE 形式の `レイアウト.json` を使ったキーボード可視化（絶対配置、回転キー対応）
- レイヤータブ（dirty 表示 `•` 付き）
- キーをクリックしてバインディング変更モーダルを開く
- 変更済みキーのハイライト表示

#### キー編集モーダル
対応ビヘイビア（すべて実機書き込み対応）:
- `&kp` — ビジュアルキーボードでキー選択
- `&lt` — レイヤー番号＋タップキー選択
- `&mt` — モディファイア＋タップキー選択
- `&none` / `&trans` / `&bootloader` — パラメータなし
- `&mkp` — マウスボタン（MB1〜MB5）
- `&bt` — Bluetooth操作（BT_SEL/NXT/PRV/CLR/CLR_ALL）

各ビヘイビアの説明パネル（右側）に解説・例・「⚡ 実機書き込み対応」バッジを表示。  
プレビュー入力欄で直接テキスト編集も可能。

#### 保存フロー
```
[保存ボタン] 
  ↓ 編集あり
  1. reconstructKeymap() で .keymap テキスト再構築
  2. GitHub API PUT でプッシュ → S.edits を S.githubSaved に移行
  ↓ デバイス接続中
  3. GetKeymap RPC で実機の behaviorId を取得
  4. GitHub バインディングと照合して behavior名→ID マップ構築
  5. SetLayerBinding RPC で各変更キーを書き込み
  6. 完了
```

デバイス未接続時: GitHub push のみ実行し、接続後に「再度保存ボタン」で実機書き込み。

#### デバイス接続（WebSerial）
- 「⚡ USB 接続」ボタンでポート選択
- 切断ボタン
- 接続状態インジケーター（緑/灰ドット）

#### 実機からの逆読み込み（GetLayerBindings 相当）
- 「📥 実機」ボタンで GetKeymap RPC 送信
- 全レイヤー・全キーの behaviorId/param1/param2 を取得
- GitHub の binding 文字列と値レベルで照合（一致すれば GitHub 表記を保持）
- 差分があれば `S.edits` に記録し、「保存 (GitHub + 実機)」で反映可能
- 実機と GitHub の一致/不一致をログに表示

#### コンボ編集
- `.keymap` 内の `combos { }` セクションをパース
- コンボの追加・編集・削除 UI
- `reconstructWithCombos()` で keymap に反映 → 保存ボタンで GitHub push
- **注意**: コンボは GitHub push のみ（実機への即時書き込みは未対応）

#### キーマップの再構築
- `reconstructKeymap()`: 変更済みレイヤーの bindings を元のテキストに置換
- 1行1バインディング形式でフォーマット
- sensor-bindings は誤判定しないよう除外

#### RPC 実装詳細
- `encodeFrame()` / `FrameDecoder` — フレームエンコード・デコード
- `buildSetLayerBinding()` — 手書き Protobuf エンコーダ
- `buildGetKeymapRequest()` — GetKeymap リクエスト構築
- `parseGetKeymapResponse()` — 3層ネスト Protobuf デコーダ
- `parseZmkKeycode()` — `LC(X)` 等のネスト修飾子対応キーコードパーサ
- `parseBindingForRpc()` — バインディング文字列 → {beh, p1, p2} 変換
- `deviceBindingToString()` — 実機の {behaviorId, p1, p2} → ZMK binding 文字列変換
- ステール応答検出（requestId の単調増加管理）

### 3-2. 未実装・残課題

#### Phase 2 残り
- **WebBluetooth 接続** — WebSerial のみ実装済み。BLE対応なし
- **GitHub Actions ビルドステータス表示** — push 後の Actions 結果をポーリングして UI に表示する機能なし（ログに URL を出すだけ）
- **OAuth 認証** — PAT 方式で代替中。OAuth App 未登録のまま

#### Phase 3（未着手）
- キーの視覚的編集UI は基本実装済みだが、Phase 3 の要件として残るもの:
  - `&mo`, `&to`, `&tog` ビヘイビアのモーダル内 UI（現在 `&lt` で代替可能だが専用UIなし）
  - hold-tap の詳細パラメータ設定（`tapping-term-ms` 等の per-binding 設定）

#### Phase 4（未着手）
- `.conf` ファイルの読み込みと未作成ビヘイビア一覧表示
- ビヘイビア専用作成画面（`.conf` への追記）
- ビヘイビア解説ページ

#### Phase 5（未着手）
- ビルドステータス表示（GitHub Actions 結果のリアルタイム表示）
- エラー通知・リトライ案内
- iOS 向けブラウザ案内

#### 技術的な既知の制約
- `&mo`, `&to`, `&tog` はキー編集モーダルにボタンがない（直接テキスト入力欄で入力は可能）
- GetKeymap の Protobuf レスポンスは `wire 1/5`（64bit/32bit）に未対応（停止する）
- コンボの実機即時書き込みは未対応（GitHub push のみ）
- BLE（WebBluetooth）接続は未実装
- `behaviors` Protobuf フィールド（field4）は未使用

---

## 4. ファイル構成

```
zmk-keymap-sync/
├── CLAUDE.md              # Claude Code 指示書（開発ルール）
├── README.md
├── docs/
│   ├── requirements.md    # 要件定義書
│   └── handover.md        # 本ファイル
├── app/
│   └── index.html         # 本番アプリ（全実装を1ファイルに収容）
└── test/
    └── t1_verification.html  # T-1検証用（参考用・変更不要）
```

---

## 5. 起動方法

```powershell
# app/ ディレクトリに静的サーバーを立てる
npx serve app
# → http://localhost:3000/index.html にアクセス
# または VS Code の Live Server 拡張でも可
```

**注意**: `file://` では WebSerial が動作しない。必ず HTTP サーバー経由でアクセスする。  
**注意**: ZMK Studio と同じシリアルポートは同時使用不可（排他的）。

---

## 6. 作業ルール（CLAUDE.md より）

- ビルドエラー自動修正の上限: 最大3回
- 作業完了後の必須手順: `git add → commit → push → gh run list --limit 3` でビルド確認
- ビルド確認方法: `gh run list --limit 3` のみ（`gh run watch` は使用禁止）
- push 後 5分待機してからビルド確認
- 段階的に確認しながら進める（勝手に大きく作り始めない）

---

## 7. 次の優先タスク（提案）

1. **`&mo` / `&to` / `&tog` のモーダル UI 追加** — よく使うビヘイビアなので優先度高
2. **GitHub Actions ビルドステータス表示** — push 後に Actions の結果をポーリングして表示
3. **WebBluetooth 接続対応** — iOS 利用のため必要
4. **Phase 4: `.conf` 読み込みと未作成ビヘイビア表示**

---

## 8. 将来候補タスク（E系）

現行スコープ外だが、技術的に価値がある改善候補。

| # | 内容 | 優先度 |
|---|---|---|
| E-1 | 複雑なビヘイビア対応（tap-dance、macro 等の詳細パラメータ設定） | 中 |
| E-2 | 複数キーボードの管理（複数リポジトリ・複数デバイスの切り替え対応） | 低 |
| E-3 | **behaviors サブシステム RPC による権威的 behavior ID 取得** | 中 |

### E-3 詳細：ListAllBehaviors RPC への移行

**現状の課題:**
`buildBehIdMap()` は「GitHub の originalBindings と実機の GetKeymap 結果を位置照合し、
param1/param2 が完全一致するキーのみから behavior 名→ID マップを学習する」という推論的手法を採用している。
照合できるキーが少ない（差分が多い初期状態）場合や、全キーが差分になった場合は
マップが空になり適用不可が増える。

**解決策:**
ZMK Studio RPC の `behaviors` サブシステム（`ListAllBehaviors` 等）を使うと、
デバイスが保持する behavior 名と ID の対応を権威的に取得できる。
これにより推論を廃止し、確実な mapping が得られる。

**なぜ今やらないか:**
behaviors サブシステムの RPC フォーマットは未実装・未調査であり、
実装コストが高い。現行の推論的手法は「param まで一致する位置のみ学習」
「矛盾する名前は除外（1名前↔1ID の単射）」という安全弁を持っており、
実用上は十分と判断している。
