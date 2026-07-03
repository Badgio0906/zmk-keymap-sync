# 追記レポート：修正A-C 実施結果

作成日: 2026-07-04  
対象: チェックポイント1 後の修正A〜C  
ベースコミット: 98142d3

---

## 修正一覧

| 修正 | 内容 | 結果 |
|---|---|---|
| 修正A | `buildBehIdMap()` 共通ヘルパー化、idToBeh 堅牢化 | ✅ 完了 |
| 修正B | 適用後 GetKeymap 再検証 | ✅ 完了 |
| 修正C | 適用不可時案内文の4ステップ化 | ✅ 完了 |
| 記録 | handover.md に E-3（ListAllBehaviors）追記 | ✅ 完了 |

---

## 修正A 詳細：`buildBehIdMap()` の共通ヘルパー化

### 変更前の問題
- `loadFromDevice()` と `applyGithubToDevice()` でそれぞれ異なる behIdMap 構築ロジックを持っていた
- `applyGithubToDevice()` の旧実装は「最初に現れた behavior 名→ID を記録するだけ」で、矛盾する証拠を無視し誤マップの可能性があった
- `loadFromDevice()` の旧実装は `nameToId` という別変数名で同様の問題を抱えていた

### 変更後
共通ヘルパー `buildBehIdMap(deviceBindings, layers)` を新設（1箇所のみ定義）：
- **学習条件**: GitHub の originalBindings と実機 GetKeymap の両者で `param1` AND `param2` が完全一致するキー位置のみ
- **矛盾除外**: 同一 behavior 名が複数の異なる ID に対応する場合、そのエントリを除外（1名前↔1ID の単射を強制）
- **理由付き can-apply**: `reason` フィールドで「behavior ID 未確定」「behavior ID 未確定（矛盾）」「未対応構文」を区別
- `loadFromDevice()` と `applyGithubToDevice()` の両方から呼ぶよう統一

### 影響範囲
`loadFromDevice()` の step 2/3、`writeToDevice()`、`renderDiffPanel()` を修正。
既存の GetKeymap フロー・SetLayerBinding フローは変更なし。

---

## 修正B 詳細：適用後 GetKeymap 再検証

### 実装場所
`applyGithubToDevice()` のステップ3として追加（書き込みループの直後）

### 検証フロー
1. 書き込み件数 > 0 の場合のみ GetKeymap を再実行
2. 適用した各キーについて `behaviorId + param1 + param2` を照合
3. 一致: `verifiedOk++`
4. 不一致: ログに警告 + `stillDiff` リストに追加（`devBinding` を実機の現在値に更新）
5. `S.diffItems` を `[適用不可] + [stillDiff]` で再構成
6. 差分パネルを更新（全一致なら非表示、残差分があれば表示継続）

### 検証タイムアウト時
`'⚠ 検証のGetKeymapがタイムアウト — 手動で逆読み込みを実行してください'` をログ表示し、
差分パネルは更新せず（安全側に倒す）。

---

## 修正C 詳細：適用不可時案内文

### 変更前
`適用不可 N 件は「Restore Stock Settings」後に再フラッシュが必要です`

### 変更後（4ステップ）
```
適用不可 N 件の手順:
  1. 未保存の変更がすべてGitHubに保存済みであることを確認
  2. GitHubソースからビルドし、生成されたファームウェアを手動でフラッシュ
  3. ZMK Studioまたは本アプリから Restore Stock Settings（設定リセット）を実行
     ※実機のRPC変更はすべて消えます。手順1の確認が必須です
  4. 本アプリの逆読み込みで実機とGitHubの一致を確認
```

手順1の確認を先頭に置くことで、RPC変更の消失前に GitHub 側の保存が完了しているかをユーザーが確認できる。

---

## 記録：handover.md E-3 追記

`docs/handover.md` の「将来候補タスク（E系）」セクション（新設）に
`E-3: behaviors サブシステム RPC による権威的 behavior ID 取得` を追記した。

なぜ今やらないかの理由（推論手法の安全弁で十分）も合わせて記録。

---

## Task 3 への引継ぎ事項

- `S.diffItems` は修正後、`[!canApply] + [verified失敗]` の構造を維持する
- Task 3 の SHA 競合対策は `btn-save` クリックハンドラ冒頭への追加で最小差分
- `S.edits` の `beforeunload` 判定は `Object.keys(S.edits).length > 0` で可
