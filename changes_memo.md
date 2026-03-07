# v50.23 変更内容メモ
## Pull Request 用

---

## 1. グループ移動のパフォーマンス改善（ゴースト描画方式）

**課題:** まとめ移動でレール数が多いと重く、スマホでは落ちる可能性があった

**変更内容:**
- ドラッグ中は `moveGroup()` を呼ばず `ghostOffset`（オフセット値）のみ保持
- 描画は `drawAllWithGhost()` で元位置に薄く残しつつゴーストを半透明で重ね描き
- `mouseup` / `touchend` 時に初めて `moveGroup()` で実座標を確定 → `drawAll()` は1回のみ

**技術詳細:**
- `ghostBaseX/Y` でドラッグ開始位置を記録し、絶対値でオフセット計算（累積誤差なし）
- `drawAllWithGhost()` 内で `ctx.setTransform` を明示的に再設定（`drawAll()` が `ctx.restore()` するため）
- ゴースト描画時は `globalAlpha = 0.6` で半透明表示

---

## 2. スナップ処理の最適化

**課題:** 確定時（mouseup）のスナップ処理が重く、大きなグループで顕著に遅かった

**変更内容:**
- `group.includes()` → `Set.has()` に変更（O(n)→O(1)）
- 接続済み端点を事前フィルタして総当たり数を削減
- `moveGroup()` 前にキャッシュを更新し正確な近傍レールのみを対象に
- 距離比較を `Math.sqrt` なしの二乗距離に変更
- `angleDiff ≈ 0` のとき `rotateGroup()` をスキップ（直線移動時に全ComplexRailの `_sync()` が走っていた問題を解消）

---

## 3. URL保存にビューポート情報を追加

**変更内容:**
- URL に `&view=scale,cx,cy` パラメータを追加
- `cx/cy` はワールド座標の画面中心点として保存（デバイス間で再現性あり）
- ロード時に `scale` と `canvasOffset` を復元

---

## 4. 軽減モードのデフォルトをONに変更

**変更内容:**
- `usePerformanceMode = false` → `true`
- ボタン初期表示を `軽減: OFF`（赤）→ `軽減: ON`（緑）に変更

---

## 5. Facilities描画の更新（複数クラス）

preview3 で調整した内容を本体に反映

| クラス | 内容 |
|---|---|
| DoubleTrackTurnoutRail | drawFacilities 置き換え・直線長を120→100に修正 |
| DoubleCurveRail | drawFacilities 置き換え |
| DoubleTrackCrossoverRail | drawFacilities 置き換え |
| R13PointRail | drawFacilities 置き換え |
| R15WidePointRail | A型/B型の枕木描画（B型は1-tで上下反転） |
| R22YPointRail | drawFacilities 新規追加（元々なかった） |
| R30TrianglePointRail | drawFacilities 置き換え |

---

## 6. 開発用テンプレートの追加

各レール種別ごとの `drawFacilities` 開発環境を追加

| ファイル | 対象 |
|---|---|
| `plarail_dev_straight.html` | StraightRail（直線・1/2直線） |
| `plarail_dev_curve.html` | Curve（左右カーブ） |
| `plarail_dev_turnout.html` | TurnoutRail（R型/L型 × 左右） |
| `plarail_dev_tor.html` | DoubleTrackTurnoutRail（表/裏 × 左右） |

各テンプレートの機能:
- クリックで座標取得・複数点管理・ドラッグ編集
- `ctx.lineTo()` コード自動生成
- `DevRail` クラスで開発 → 完成後にクラス名一括置換してコピー

---

## 変更ファイル

- `index.html`（旧 plarail_v50_23.html）
- `plarail_engine.js`
- `plarail_engine_ext.js`
- `plarail_dev_straight.html`（新規）
- `plarail_dev_curve.html`（新規）
- `plarail_dev_turnout.html`（新規）
- `plarail_dev_tor.html`（新規）
