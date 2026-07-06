# ZMK Keymap Sync — プロジェクト引き継ぎ書

作成日: 2026-07-03  
更新日: 2026-07-06（Task R〜U・修正S 完了後）  
リポジトリ: https://github.com/Badgio0906/zmk-keymap-sync  
対象キーボード（デフォルト）: **roBa**（Badgio0906/zmk-config-roBa）

---

## 1. プロジェクト概要

ZMKファームウェアを使う自作キーボードのキーマップを、WebアプリUIから変更する。  
変更時に以下を**同時実行**して「結果的同期」の状態を保つ：

- **即時バイナリ**：ZMK Studio RPCプロトコル（WebSerial）経由で実機の **settingsパーティション** を直接書き換え
- **ビルドバイナリ**：GitHubに `.keymap` を push → GitHub Actions でビルド（実機には焼かない・ソース正確性の担保目的）

### 重要: settingsパーティションの性質

ZMK Studio RPC による変更は **settingsパーティション（NVS領域）** に保存される。
- ファームウェアを手動で再フラッシュしても、RPC変更は消えない（`Restore Stock Settings` 実行時のみリセット）
- よって **GitHub側の変更を実機に反映する唯一の正式経路は「RPC適用」**（手動フラッシュで解決できない）
- 「ビルドバイナリを焼けば実機がGitHubと一致する」という前提は誤り
- tap-dance 等のカスタムビヘイビアは firmware 定義のため、焼き込み後に RPC で利用可能になる

### 背景

| ツール | できること | できないこと |
|---|---|---|
| Keymap Editor | 複雑なビヘイビア設定 | 実機への即時反映 |
| ZMK Studio | 実機への即時反映 | 複雑なビヘイビア設定 |

このアプリは両者の欠点を補完する。

---

## 2. 技術スタック

- **形態**: 純粋 Webアプリ（バックエンドなし）、単一 HTML ファイル
- **ファイル**: `app/index.html`（全実装を1ファイルに収容、外部ライブラリ禁止）
- **対応ブラウザ**: Chrome / Edge（PC・Android）、Bluefy等（iOS）
- **接続**: WebSerial（USB）／BLE は未実装
- **GitHub連携**: Personal Access Token（PAT）＋ GitHub REST API v3
- **RPC通信**: 独自バイトスタッフィング ＋ Protobuf（手書きエンコーダ/デコーダ）
- **状態管理**: `localStorage`（PAT、キーボード設定）＋ グローバル `S` オブジェクト
- **依存ライブラリ**: なし

### ZMK Studio RPCプロトコル

`@zmkfirmware/zmk-studio-ts-client` は**使用していない**。独自の最小実装。

**フレーミング（バイトスタッフィング）:**
```
0xAB = Start of Frame (SOF)
0xAC = Escape byte (ESC)
0xAD = End of Frame (EOF)
データ中に 0xAB/0xAC/0xAD が現れたら [0xAC, byte] にエスケープ
```

**Protobuf フィールド番号:**
- Request: field1=requestId, field5=keymap(KeymapRequest)
- KeymapRequest: field1=getKeymap(bool), field2=setLayerBinding
- SetLayerBindingRequest: field1=layerId, field2=keyPosition, field3=binding(BehaviorBinding)
- BehaviorBinding: field1=behaviorId(sint32/zigzag), field2=param1, field3=param2
- RpcResponse: field1(wire2) = RequestResponse { field1=requestId, field5=KeymapResponse }
- KeymapResponse: field1=GetKeymapResponse { field1=layers(repeated) }

**wire type 対応状況:**
- wire 0（varint）: 読み取り ✓
- wire 2（length-delimited）: 読み取り ✓
- wire 1（64-bit）: 8バイトスキップして継続 ✓
- wire 5（32-bit）: 4バイトスキップして継続 ✓
- wire 3/4 等: ログ警告を出してデコード中断

### ZMK キーコードエンコーディング（修正K）

```
bits 31-24 = modifier flags (LC=0x01 LS=0x02 LA=0x04 LG=0x08 RC=0x10 RS=0x20 RA=0x40 RG=0x80)
bits 23-16 = HID ページ (0x07 = keyboard, 0x0C = consumer, 等)
bits 7-0   = HID Usage ID
```

modifier bits が bits 7-8 にあると誤って実装していた（修正K で修正済み）。

### &bt コマンド番号（修正L）

ZMK の dt-bindings に合わせた正しいマッピング:
```
BT_CLR=0, BT_NXT=1, BT_PRV=2, BT_SEL=3, BT_CLR_ALL=4
```

### BLE（Studio Transport BLE）について（調査I）

`studio-rpc-usb-uart` スニペットは `CONFIG_ZMK_STUDIO_TRANSPORT_BLE=n` を設定するため、
USB/UART 専用となる。BLE 経由の Studio RPC は現状不可（WebBluetooth 実装も未着手）。

---

## 3. グローバル状態 `S` オブジェクト

```javascript
const S = {
  pat:          '',          // GitHub PAT（localStorage）
  kb:           {},          // 現在選択中のキーボード設定
  layoutKeys:   [],          // KLEレイアウト
  layers:       [],          // { name, bindings[], originalBindings[], dirty }[]
  keymapRaw:    null,        // .keymap の生テキスト
  keymapSha:    null,        // GitHub ファイル SHA（競合検出用）
  currentLayer: 0,
  edits:        {},          // 未 GitHub push の変更 { "layerIdx:keyIdx": binding }
  transport:    null,        // WebSerial ポートオブジェクト
  diffItems:    [],          // 実機/GitHub 差分一覧（差分パネル用）
  unparsableItems: [],       // GitHub側構文が解析不能なキー一覧
  idToBeh:      {},          // 最後の GetKeymap から得た behaviorId→名前マップ
  behIdMap:     {},          // 学習済み behavior 名→ID マップ（フラッシュ待ち判定用）
  building:     false,       // GitHub Actions ビルド中ロック状態
  buildPollToken: 0,         // インクリメントで実行中ポーラーをキャンセル
  dataSource:   null,        // 現在表示中のデータ源 { type:'github'|'device', at?:Date }
};
```

**旧 `handover.md` との差分:**
- `githubSaved` は削除済み（現在の実装にない）
- `unparsableItems`, `behIdMap`, `dataSource` を追加（Task M・P・R で追加）

---

## 4. 現在の実装状態（`app/index.html`）

### 4-1. 実装済み機能（Task 0〜U）

#### GitHub 連携
- PAT（Personal Access Token）による認証（localStorage 保存）
- GitHub API 経由で `.keymap` とレイアウト JSON を取得
- PAT 未設定時は設定モーダルを自動表示
- GitHub に `.keymap` を PUT（コミットメッセージ付き）
- **SHA競合対策（Task 3・修正D）**:
  - push 前に `ghGetFile` で最新 SHA を再取得
  - 差異あり → SHA競合モーダルを表示（件数付き警告・リロードボタン・キャンセルボタン）

#### キーボード管理
- 複数キーボードの切り替え（localStorage に設定を保存）
- キーボードの追加・編集・削除モーダル
- GitHub URL 貼り付けで owner/repo/branch を自動解析
- 「自動検出」ボタンで `.keymap` と `.json` のパスを GitHub から自動取得

#### キーマップ表示・編集
- `.keymap` ファイルのパース（`keymap { }` → レイヤー → `bindings = < >` 抽出）
- KLE 形式の `レイアウト.json` を使ったキーボード可視化（絶対配置、回転キー対応）
- キャンバスは横幅に合わせて 1.0x〜1.5x の範囲で自動スケール（`KEY_SCALE=72`）
- レイヤータブ（dirty 表示 `•` 付き）
- キーをクリックしてバインディング変更モーダルを開く
- 変更済みキーのハイライト表示
- **beforeunload 離脱警告（Task 3）**: `S.edits` に未保存変更がある間はブラウザ離脱確認を表示

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

**カスタムビヘイビア（Task U-1）:**  
`.keymap` の `behaviors {}` に tap-dance が定義されている場合、モーダル下部に「カスタムビヘイビア」
セクションを表示。ボタンをクリックすると右パネルに TD 詳細（タップ数・各動作・フラッシュ状態）を表示。
「キーに割り当て」でキーピッカーモードに入り、キーボードをクリックして直接割り当てられる。

#### カスタムビヘイビアの比較（Task P）

`.keymap` 中の `behaviors {}` ブロックに定義されたカスタムビヘイビアのパース:
- `&name` でパラメータなし → `{ beh:name, p1:0, p2:0, isCustom:true }`
- `buildBehIdMap()` の位置照合でカスタムビヘイビアの ID も学習される

#### 保存フロー
```
[保存ボタン]
  ↓ 編集あり
  1. GitHub から最新 SHA を再取得（競合チェック）
     → 差異あり: SHA競合モーダル表示
  2. validateKeymap() で構文チェック（N-3 再設計版）
     → エラーあり: 保存をブロック
  3. reconstructKeymap() で .keymap テキスト再構築
  4. GitHub API PUT でプッシュ
  5. pollBuildStatus(commitSha) をバックグラウンドで起動（Actions監視）
  ↓ デバイス接続中
  6. GetKeymap RPC で実機の behaviorId を取得
  7. buildBehIdMap() で behavior名→ID マップ構築
  8. SetLayerBinding RPC で各変更キーを書き込み
```

デバイス未接続時: GitHub push のみ実行し、接続後に「実機に書き込み」ボタンで実機書き込み。

#### バリデーション（修正N-3: S タスクで全面再設計）

**旧実装の問題**: グローバルな `bindings = <` 数と `>;` 数を比較していた。
`behaviors {}` / `combos {}` の `#binding-cells = <0>;`, `tapping-term-ms = <200>;`, `key-positions = <...>;`
等の `>;` が全文から検出されるため、常に差分が生じて正常なキーマップでも誤エラーを出していた。

**修正後の設計**:
1. `extractBlock(text, 'keymap')` でブレースカウントにより `keymap {}` ブロックのみを抽出
2. 抽出したブロック内だけで `bindings = < >` の正規表現マッチを実行（ブロック単位）
3. 各ブロックのバインディング数を `originalBindings.length` と比較
4. `selfTestValidate()` を `loadData()` 後に呼び、raw + reconstruct の両方で 0 件を確認

#### デバイス接続（WebSerial）
- 「⚡ USB 接続」ボタンでポート選択
- 切断ボタン
- 接続状態インジケーター（緑/灰ドット）
- 接続成功時に自動で `loadFromDevice()` を実行（キーマップ読み込み済みの場合）

#### 実機からの逆読み込み（Task 2）
- 「📥 実機」ボタン（USB 接続後に表示）で GetKeymap RPC 送信
- 全レイヤー・全キーの behaviorId/param1/param2 を取得
- GitHub の binding 文字列と値レベルで照合
- 差分があれば **差分パネル** を表示

**差分パネルの操作:**
- 「⬇ GitHubを実機に適用」→ `applyGithubToDevice()`:
  1. GetKeymap で最新の behaviorId を再取得
  2. `buildBehIdMap()` で behavior名→ID マップ構築
  3. 適用可能な差分キーに SetLayerBinding で GitHub 値を書き込み
  4. 適用後 GetKeymap で再検証 → 不一致キーはログ警告＋差分パネルに残す
  5. 適用不可キーには4ステップの案内を表示
- 「⬆ 実機をGitHubに反映」→ S.edits に記録し、保存ボタンで GitHub push 可能
- 「閉じる」

#### behavior名→IDマップ: `buildBehIdMap()`（修正A）
- **学習条件**: GitHub の originalBindings と実機 GetKeymap の両者で `param1` AND `param2` が完全一致するキー位置のみ
- **矛盾除外**: 同一 behavior 名が複数の異なる ID に対応する場合はそのエントリを除外（1名前↔1ID の単射を強制）
- **reason フィールド**: `canApply=false` のキーに「behavior ID 未確定」「behavior ID 未確定（矛盾）」「適用不可（フラッシュ待ち）」「未対応構文」を区別表示
- `S.behIdMap` に保存（フラッシュ待ち判定用・Task R で追加）
- `loadFromDevice()`・`writeToDevice()`・`applyGithubToDevice()` の3箇所で共通使用

#### GitHub Actions ビルド監視（Task 4・修正E）

**ポーリング設計:**
- `pollBuildStatus(headSha)`: push後に呼ぶ。`head_sha` 指定で run 検索（10秒×10回）→ 20秒間隔ポーリング（最大15分）
- `_pollRunLoop(runId, pollOwner, pollRepo, token)`: 単一 run をポーリングする内部ループ
- `checkAndResumeBuild()`: KB切り替え後に最新 run を1回確認 → 進行中なら追跡再開

**キャンセルトークン（修正E）:**
- `S.buildPollToken` を整数カウンター（インクリメントで全実行中ポーラーを一括無効化）
- KB 切り替え時: `++S.buildPollToken` → 旧ポーラーが次の await チェックで即終了 → `checkAndResumeBuild()` で新 KB のビルド状態を確認

**ビルドロック（U-2改）:**
- `setBuildLock(locked)`: `btn-save` を `disabled`、`keyboard-canvas` を `pointer-events:none + opacity:0.65`
- **`btn-write-device` はビルドロックしない**（ビルドと独立して使用可能）
- ビルド中/待機中は `build-guidance` 要素に「実機に書き込みはビルドと独立して使用できます」を表示

**ステータスバッジ:**
- `⏳ ビルド開始待ち` / `⏳ ビルド待機中`（グレー）
- `⚙ ビルド中`（ブルー）
- `✓ ビルド成功`（グリーン）
- `✗ ビルド失敗`（レッド）

#### データソース表示・同期ステータス（Task M）

- **M-1**: ステータスパネルの「表示中データ」バッジ。`S.dataSource` を表示（`github`/`device`）
- **M-2**: 同期状態バッジ（⊘ 実機未接続 / ✓ 同期済み / △ 差分あり）。差分ありは差分パネルを開くボタンとして機能
- **M-3**: USB 接続成功時にキーマップ読み込み済みなら自動で `loadFromDevice()` を実行

#### UI カスタマイズ（Task O）

- **O-2**: テーマ切替ボタン（ダーク/ウォームダーク/ライト）
- **O-3**: 文字サイズ切替ボタン（sm=14px / md=16px / lg=18px）
- **O-4**: ログパネル高さをドラッグリサイズ
- **O-5**: キーボードキャンバスの `scale-to-fit`（横幅に合わせて 1.0x〜1.5x 自動スケール）

テーマとサイズは `localStorage` に保存。

#### ステータスパネル（Task Q）

ページ右側の aside パネル。セクション構成:
1. 同期状態バッジ（`#sync-badge`）
2. USB接続ドット（`#dev-dot` / `#dev-status`）
3. 表示中データ（`#datasource-badge`）
4. ビルド（`#sp-build-section` - ビルド中のみ表示）
5. フラッシュ待ち TD（`#sp-flash-section` - TD 保存後に表示）

#### コンボ編集
- `.keymap` 内の `combos { }` セクションをパース
- コンボの追加・編集・削除 UI
- `reconstructWithCombos()` で keymap に反映 → 保存ボタンで GitHub push
- **注意**: コンボは GitHub push のみ（実機への即時書き込みは RPC 非対応）
- USB 接続中のみ「🔗 コンボ編集」ボタンが有効化される

#### tap-dance ビヘイビア管理（Task R）

アクションバーの「🎵 ビヘイビア管理」ボタン（キーマップ読み込み後に有効化、USB 接続不要）。

**DTS フォーマット（behaviors {} ブロックへの出力）:**
```dts
name: name {
    compatible = "zmk,behavior-tap-dance";
    #binding-cells = <0>;
    bindings = <&kp A>, <&kp B>;
    tapping-term-ms = <200>;
};
```

機能一覧:
- **R-1**: TD 一覧表示（タップ数・各動作・割り当てキー数）
- **R-2**: TD 新規作成フォーム（名前・動作・tapping-term-ms）+ GitHub push
- **R-3**: TD 編集フォーム（既存 TD の変更）
- **R-4**: フラッシュ待ち判定（`S.behIdMap` に TD 名が未登録 → フラッシュ待ち）
  - フラッシュ待ち TD はキー編集モーダルでグレーアウト・無効化
  - `getFlashWaitTds()` = `parseTapDances().filter(td => !(td.name in S.behIdMap))`

**TD 削除（Task U-3）:**  
削除確認パネルが展開し、割り当て済みキー一覧を表示。確認後:
1. `behaviors {}` から TD 定義を削除（`removeTdFromBehaviors()`）
2. 割り当てキーをすべて `&none` に変更
3. GitHub push

#### キーピッカーモード（Task T・Task U-1）

TD 作成/編集フォームから「キーボードから選ぶ」ボタン、またはキー編集モーダルから「キーに割り当て」でキーピッカーモードに入る。

実装:
```javascript
let keyPickMode = false;
let _keyPickForTd = null; // null = T-1 フォーム入力モード, string = U-1 直接割り当てモード

startKeyPick(tdAssignName = null)  // td-modal を隠してバナーを表示
endKeyPick(layerIdx, keyIdx)       // 選択後の処理を分岐
```

- `_keyPickForTd === null`: T-1 モード — フォームの「キー位置」欄を更新
- `_keyPickForTd === string`: U-1 モード — `S.layers[layerIdx].bindings[keyIdx]` を直接書き換えて `S.edits` に記録

キャンバスはクロスヘアカーソル（`.key-pick-mode .key { cursor: crosshair }`）。  
編集中の TD に割り当てられているキーは紫色でハイライト（`.key.td-assigned`）。

#### ファビコン（Task U-4）

```html
<link rel="icon" href="data:image/svg+xml,...">
```
SVG インライン（背景 #818cf8・⌨ 絵文字）。外部リソース不使用。

---

### 4-2. 未実装・残課題

#### Phase 4（未着手）
- `.conf` ファイルの読み込みと未作成ビヘイビア一覧表示
- ビヘイビア専用作成画面（`.conf` への追記）
- ビヘイビア解説ページ

#### 技術的な既知の制約
- `&mo`, `&to`, `&tog` はキー編集モーダルにボタンがない（直接テキスト入力欄で入力は可能）
- コンボの実機即時書き込みは未対応（GitHub push のみ）
- BLE（WebBluetooth）接続は未実装
- `behaviors` Protobuf サブシステム（`ListAllBehaviors`等）は未実装（→ E-3 参照）
- アプリ初回ロード時（ページリロード後）はビルド状態を復元しない
- `checkAndResumeBuild` は `per_page=1`（最新1件）取得のため、別ワークフロー（lint等）の run を誤追跡する可能性がある
- キーボードスケールは手動変更不可（`scale-to-fit` 自動のみ）

---

## 5. ファイル構成

```
zmk-keymap-sync/
├── CLAUDE.md                              # Claude Code 指示書（開発ルール）
├── README.md
├── docs/
│   ├── requirements.md                    # 要件定義書
│   ├── usage_spec.md                      # 利用仕様書（ユーザー向け）
│   ├── handover.md                        # 本ファイル（技術引き継ぎ）
│   └── reports/
│       ├── report_20260704.md             # チェックポイント1（Task 0-2）
│       ├── report_20260704_addendum.md    # 修正A-C 追記レポート
│       ├── report_20260704_cp2.md         # チェックポイント2（Task 3-4）
│       ├── report_20260704_de.md          # 修正D・E 追記レポート
│       └── report_20260705_i.md           # 調査I (Studio BLE) レポート
├── app/
│   └── index.html                         # 本番アプリ（全実装を1ファイルに収容）
└── test/
    └── t1_verification.html               # T-1検証用（参考用・変更不要）
```

---

## 6. 起動方法

```powershell
# app/ ディレクトリに静的サーバーを立てる
npx serve app
# → http://localhost:3000/index.html にアクセス
# または VS Code の Live Server 拡張でも可
```

**注意**: `file://` では WebSerial が動作しない。必ず HTTP サーバー経由でアクセスする。  
**注意**: ZMK Studio と同じシリアルポートは同時使用不可（排他的）。

---

## 7. 作業ルール（CLAUDE.md より）

- `app/index.html` の単一ファイル構成を維持（外部ライブラリ追加禁止）
- 既存機能を壊さない。大規模リファクタリング禁止。差分は最小限に
- ビルドエラー自動修正の上限: 最大3回
- 各タスク完了ごとに個別コミット（メッセージにタスク番号を含める）
- 作業完了後の必須手順: `git add → commit → push → gh run list --limit 3` でビルド確認
- ビルド確認方法: `gh run list --limit 3` のみ（`gh run watch` は使用禁止）
- push 後 5分待機してからビルド確認
- 段階的に確認しながら進める（勝手に大きく作り始めない）
- git fetch で最新を確認してから作業する

---

## 8. コミット履歴（2026-07-06 時点）

```
af94b43 fix(U-1): カスタムビヘイビア選択時に右パネルへTD情報を表示
fc113ff feat(U): 操作性の小改善4件 (U-1〜U-4)
1619c55 feat(T): TD作成フォームのキー位置直接選択 (T-1〜T-4)
b1fb3c8 fix(S): N-3バリデーション全面再設計 (S-1〜S-5)
335313d feat(R): tap-dance behavior manager (R-1 to R-4)
6011901 feat(Q+O6改): status panel + diff direction labels
f7d11a7 feat(O1+P): contrast fix + custom behavior comparison support
ef24e26 feat(O): UI customization — themes, font sizes, log resize, keyboard scale-to-fit
1c6a13f 修正N-2/N-3: バインディング入力バリデーション + 保存前構文チェック
2ad68fb Task M: データソース表示・自動再接続・同期ステータス (M-1/M-2/M-3)
2a800d9 fix(修正L): &bt コマンド番号を ZMK dt-bindings に合わせて修正
bfb01bf fix(調査K): ZMK modifier bits位置を bits31-24 に修正
7e3f773 feat(Task F-2): ダークテーマUIリフレッシュ
7718b4a docs(調査I): Studio BLEトランスポートの検証結果を記録
bd53576 docs(小修正2件): CLAUDE.mdにgit fetchルール追記、前提条件を明記
0dbc0ce fix(修正H): キーコード対応拡充と値レベル照合の徹底
6c11f6f docs: G-1改・G-3 実装レポートを追加
ad236da fix(G-1+G-3): RPC送受信hexダンプ診断・書き込み結果の正確な表示
d35a639 fix(rpc): serialRpc/切断ハンドラのリソースリーク修正（調査G-2追記対応）
ba0b69d debug: serialRpc タイムアウト時に受信バイト数を診断ログに出力
be95962 fix(ui): 実機に書き込み・コンボ編集ボタンを常時表示・USB接続前はグレーアウト
4d207c0 fix(Task F-1+修正F): 実機書き込みを差分アプローチに変更・保存順序修正・ビルドロック粒度変更
f8e451a docs: handover.md を Task 0-4・修正A-E 完了後の状態に全面更新
df3c4d9 docs: 修正D・E 追記レポートを追加
4f0501f feat(修正D+E): SHA競合モーダル・ビルドポーラーのKB切り替え対応
0ccf03c docs: チェックポイント2レポートを追加 (Task 3-4 完了)
5b76d42 feat(Task 4): GitHub Actionsビルドステータス表示・ビルド中編集ロックを実装
4502eb9 feat(Task 3): SHA競合対策・beforeunload 離脱警告を追加
1561e99 fix(修正A-C): behavior IDマップ堅牢化・適用後再検証・案内文4ステップ化
```

---

## 9. 将来候補タスク（E系）

| # | 内容 | 優先度 |
|---|---|---|
| E-1 | macro 等のカスタムビヘイビア詳細 UI | 中 |
| E-2 | 複数キーボード管理の強化 | 低 |
| E-3 | **behaviors サブシステム RPC による権威的 behavior ID 取得** | 中 |

### E-3 詳細：ListAllBehaviors RPC への移行

**現状の課題:**  
`buildBehIdMap()` は GitHub の originalBindings と実機 GetKeymap 結果の位置照合で推論する手法。
差分が多い初期状態では照合できるキーが少なく、適用不可が増える可能性がある。

**解決策:**  
ZMK Studio RPC の `behaviors` サブシステム（`ListAllBehaviors` 等）を使うと、
デバイスが保持する behavior 名と ID の対応を権威的に取得できる。

**なぜ今やらないか:**  
behaviors サブシステムの RPC フォーマットは未実装・未調査であり実装コストが高い。
現行の推論的手法は「param まで一致する位置のみ学習」「矛盾する名前は除外（単射強制）」
という安全弁を持っており、実用上は十分。
