# ZMK Keymap Sync — プロジェクト引き継ぎ書

作成日: 2026-07-03  
更新日: 2026-07-04（Task 0〜4・修正A〜E 完了後）  
リポジトリ: https://github.com/Badgio0906/zmk-keymap-sync  
対象キーボード（デフォルト）: **roBa**（Badgio0906/zmk-config-roBa）

---

## 1. プロジェクト概要

ZMKファームウェアを使う自作キーボードのキーマップを、WebアプリUIから変更する。  
変更時に以下を**同時実行**して「結果的同期」の状態を保つ：

- **即時バイナリ**：ZMK Studio RPCプロトコル（WebSerial/WebBluetooth）経由で実機の **settingsパーティション** を直接書き換え
- **ビルドバイナリ**：GitHubに `.keymap` を push → GitHub Actions でビルド（実機には焼かない・ソース正確性の担保目的）

### 重要: settingsパーティションの性質

ZMK Studio RPC による変更は **settingsパーティション（NVS領域）** に保存される。
- ファームウェアを手動で再フラッシュしても、RPC変更は消えない（`Restore Stock Settings` 実行時のみリセット）
- よって **GitHub側の変更を実機に反映する唯一の正式経路は「RPC適用」**（手動フラッシュで解決できない）
- 「ビルドバイナリを焼けば実機がGitHubと一致する」という前提は誤り

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
- **接続**: WebSerial（USB）／WebBluetooth（BLE・未実装）
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
データ中に 0xAB/0xAC/0xAD が現れたら [0xAC, byte] にエスケープ
```

**Protobuf フィールド番号:**
- Request: field1=requestId, field5=keymap(KeymapRequest)
- KeymapRequest: field1=getKeymap(bool), field2=setLayerBinding
- SetLayerBindingRequest: field1=layerId, field2=keyPosition, field3=binding(BehaviorBinding)
- BehaviorBinding: field1=behaviorId(sint32/zigzag), field2=param1, field3=param2
- RpcResponse: field1(wire2) = RequestResponse { field1=requestId, field5=KeymapResponse }
- KeymapResponse: field1=GetKeymapResponse { field1=layers(repeated) }

**wire type 対応状況（Task 1 修正後）:**
- wire 0（varint）: 読み取り ✓
- wire 2（length-delimited）: 読み取り ✓
- wire 1（64-bit）: 8バイトスキップして継続 ✓（旧実装は停止していた）
- wire 5（32-bit）: 4バイトスキップして継続 ✓（旧実装は停止していた）
- wire 3/4 等: ログ警告を出してデコード中断

---

## 3. グローバル状態 `S` オブジェクト

```javascript
const S = {
  pat:            '',          // GitHub PAT（localStorage）
  kb:             {},          // 現在選択中のキーボード設定
  layoutKeys:     [],          // KLEレイアウト
  layers:         [],          // { name, bindings[], originalBindings[], dirty }[]
  keymapRaw:      null,        // .keymap の生テキスト
  keymapSha:      null,        // GitHub ファイル SHA（競合検出用）
  currentLayer:   0,
  edits:          {},          // 未push の変更 { "layerIdx:keyIdx": binding }
  githubSaved:    {},          // GitHub push済み・実機未書き込みの変更
  transport:      null,        // WebSerial ポートオブジェクト
  diffItems:      [],          // 実機/GitHub 差分一覧（差分パネル用）
  idToBeh:        {},          // 最後の GetKeymap から得た behaviorId→名前マップ
  building:       false,       // GitHub Actions ビルド中ロック状態
  buildPollToken: 0,           // インクリメントで実行中ポーラーをキャンセル
};
```

---

## 4. 現在の実装状態（`app/index.html`）

### 4-1. 実装済み機能

#### GitHub 連携
- PAT（Personal Access Token）による認証（localStorage 保存）
- GitHub API 経由で `.keymap` とレイアウト JSON を取得
- PAT 未設定時は設定モーダルを自動表示
- GitHub に `.keymap` を PUT（コミットメッセージ付き）
- **SHA競合対策（Task 3・修正D）**:
  - push 前に `ghGetFile` で最新 SHA を再取得
  - 差異あり → SHA競合モーダルを表示（件数付き警告・リロードボタン・キャンセルボタン）
  - 強制上書きオプションなし

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

#### 保存フロー
```
[保存ボタン]
  ↓ 編集あり
  1. GitHub から最新 SHA を再取得（競合チェック）
     → 差異あり: SHA競合モーダル表示 → ユーザー選択（リロードorキャンセル）
  2. reconstructKeymap() で .keymap テキスト再構築
  3. GitHub API PUT でプッシュ → S.edits を S.githubSaved に移行
  4. pollBuildStatus(commitSha) をバックグラウンドで起動（Actions監視）
  ↓ デバイス接続中
  5. GetKeymap RPC で実機の behaviorId を取得
  6. buildBehIdMap() で behavior名→ID マップ構築（param完全一致・矛盾除外）
  7. SetLayerBinding RPC で各変更キーを書き込み
  8. 完了
```

デバイス未接続時: GitHub push のみ実行し、接続後に「再度保存ボタン」で実機書き込み。

#### デバイス接続（WebSerial）
- 「⚡ USB 接続」ボタンでポート選択
- 切断ボタン
- 接続状態インジケーター（緑/灰ドット）

#### 実機からの逆読み込み（Task 2）
- 「📥 実機」ボタンで GetKeymap RPC 送信
- 全レイヤー・全キーの behaviorId/param1/param2 を取得
- GitHub の binding 文字列と値レベルで照合
- 差分があれば **差分パネル** を表示（レイヤー・キー位置・GitHub値・実機値・適用可否）

**差分パネルの操作:**
- 「⬇ GitHubを実機に適用」→ `applyGithubToDevice()`:
  1. GetKeymap で最新の behaviorId を再取得
  2. `buildBehIdMap()` で behavior名→ID マップ構築
  3. 適用可能な差分キーに SetLayerBinding で GitHub 値を書き込み
  4. 適用後 GetKeymap で再検証 → 不一致キーはログ警告＋差分パネルに残す
  5. 適用不可キーには4ステップの案内を表示
- 「⬆ 実機をGitHubに反映」→ S.edits に記録し、保存ボタンで GitHub push 可能
- 「閉じる」

**`applyGithubToDevice` 適用不可の案内（修正C）:**
```
1. 未保存の変更がすべてGitHubに保存済みであることを確認
2. GitHubソースからビルドし、生成されたファームウェアを手動でフラッシュ
3. ZMK Studioまたは本アプリから Restore Stock Settings（設定リセット）を実行
   ※実機のRPC変更はすべて消えます。手順1の確認が必須です
4. 本アプリの逆読み込みで実機とGitHubの一致を確認
```

#### behavior名→IDマップ: `buildBehIdMap()`（修正A）
- **学習条件**: GitHub の originalBindings と実機 GetKeymap の両者で `param1` AND `param2` が完全一致するキー位置のみ
- **矛盾除外**: 同一 behavior 名が複数の異なる ID に対応する場合はそのエントリを除外（1名前↔1ID の単射を強制）
- **reason フィールド**: `canApply=false` のキーに「behavior ID 未確定」「behavior ID 未確定（矛盾）」「未対応構文」を区別表示
- `loadFromDevice()`・`writeToDevice()`・`applyGithubToDevice()` の3箇所で共通使用

#### GitHub Actions ビルド監視（Task 4・修正E）

**ポーリング設計:**
- `pollBuildStatus(headSha)`: push後に呼ぶ。`head_sha` 指定で run 検索（10秒×10回=最大100秒）→ 20秒間隔ポーリング（最大15分）
- `_pollRunLoop(runId, pollOwner, pollRepo, token)`: 単一 run をポーリングする内部ループ（`owner`/`repo` をキャプチャ）
- `checkAndResumeBuild()`: KB切り替え後に最新 run を1回確認 → 進行中なら追跡再開

**キャンセルトークン（修正E）:**
- `S.buildPollToken` を整数カウンター（インクリメントで全実行中ポーラーを一括無効化）
- KB 切り替え時: `++S.buildPollToken` → 旧ポーラーが次の await チェックで即終了 → `checkAndResumeBuild()` で新 KB のビルド状態を確認

**ビルドロック:**
- `setBuildLock(locked)`: `btn-save`/`btn-write-device` を `disabled`、`keyboard-canvas` を `pointer-events:none + opacity:0.65`
- Actions 未設定リポジトリ: run 未検出でロック自動解除＋「ビルド設定なし」ログ

**ステータスバッジ（action bar 内）:**
- `⏳ ビルド開始待ち` / `⏳ ビルド待機中`（グレー）
- `⚙ ビルド中`（ブルー）
- `✓ ビルド成功`（グリーン）
- `✗ ビルド失敗`（レッド）

**ビルド失敗時:** Actions ログ URL をログ表示＋「再保存でリトライ」案内（自動リトライなし）

#### コンボ編集
- `.keymap` 内の `combos { }` セクションをパース
- コンボの追加・編集・削除 UI
- `reconstructWithCombos()` で keymap に反映 → 保存ボタンで GitHub push
- **注意**: コンボは GitHub push のみ（実機への即時書き込みは RPC 非対応）

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

---

## 5. ファイル構成

```
zmk-keymap-sync/
├── CLAUDE.md                          # Claude Code 指示書（開発ルール）
├── README.md
├── docs/
│   ├── requirements.md                # 要件定義書（Task 0 で更新済み）
│   ├── handover.md                    # 本ファイル
│   └── reports/
│       ├── report_20260704.md         # チェックポイント1（Task 0-2）
│       ├── report_20260704_addendum.md # 修正A-C 追記レポート
│       ├── report_20260704_cp2.md     # チェックポイント2（Task 3-4）
│       └── report_20260704_de.md      # 修正D・E 追記レポート
├── app/
│   └── index.html                     # 本番アプリ（全実装を1ファイルに収容）
└── test/
    └── t1_verification.html           # T-1検証用（参考用・変更不要）
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

---

## 8. コミット履歴（2026-07-04 時点）

```
df3c4d9 docs: 修正D・E 追記レポートを追加
4f0501f feat(修正D+E): SHA競合モーダル・ビルドポーラーのKB切り替え対応
0ccf03c docs: チェックポイント2レポートを追加 (Task 3-4 完了)
5b76d42 feat(Task 4): GitHub Actionsビルドステータス表示・ビルド中編集ロックを実装
4502eb9 feat(Task 3): SHA競合対策・beforeunload 離脱警告を追加
1561e99 fix(修正A-C): behavior IDマップ堅牢化・適用後再検証・案内文4ステップ化
795f8e9 docs: チェックポイント1レポートを追加 (Task 0-2 完了)
98142d3 feat(Task 2): GitHub to 実機の差分RPC適用UI を実装
ee5be66 fix(Task 1): Protobufデコーダに未知フィールドのスキップ処理を追加
474c2ea docs(Task 0): requirements.md にRPC/settingsパーティションの仕様を反映
```

---

## 9. 将来候補タスク（E系）

現行スコープ外だが、技術的に価値がある改善候補。

| # | 内容 | 優先度 |
|---|---|---|
| E-1 | 複雑なビヘイビア対応（tap-dance、macro 等の詳細パラメータ設定） | 中 |
| E-2 | 複数キーボードの管理（複数リポジトリ・複数デバイスの切り替え対応） | 低 |
| E-3 | **behaviors サブシステム RPC による権威的 behavior ID 取得** | 中 |

### E-3 詳細：ListAllBehaviors RPC への移行

**現状の課題:**
`buildBehIdMap()` は「GitHub の originalBindings と実機の GetKeymap 結果を位置照合し、
param1/param2 が完全一致するキーのみから behavior 名→ID マップを学習する」という推論的手法。
差分が多い初期状態では照合できるキーが少なく、適用不可が増える可能性がある。

**解決策:**
ZMK Studio RPC の `behaviors` サブシステム（`ListAllBehaviors` 等）を使うと、
デバイスが保持する behavior 名と ID の対応を権威的に取得できる。
これにより推論を廃止し、確実な mapping が得られる。

**なぜ今やらないか:**
behaviors サブシステムの RPC フォーマットは未実装・未調査であり、実装コストが高い。
現行の推論的手法は「param まで一致する位置のみ学習」「矛盾する名前は除外（1名前↔1ID の単射）」
という安全弁を持っており、実用上は十分と判断している。
