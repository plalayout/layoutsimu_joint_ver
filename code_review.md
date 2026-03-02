# レビュー結果（対象: `ada8b26` / cut off display out side drawing）

## 総評
- 描画対象を絞る方針自体は有効ですが、**描画最適化のための判定結果 (`skipDraw`) を入力処理（選択・右クリック・スナップ）にも使っている箇所**があり、ユーザー操作の正しさを壊すリスクがあります。
- 特にズームアウト時の制御は「軽量化」ではなく「機能停止」に近い動作が入っているため、体験劣化の影響が大きいです。

---

## 指摘事項

### 1) [High] 近傍レール抽出が端点ベースのみで、長いレールを取りこぼす
**問題**
- `getNearbyRails` は `endpoints.some(...)` で「端点が範囲内か」だけを見ています。
- そのため、レール本体が範囲内を横切っていても、端点が両方とも範囲外なら除外されます。

**影響**
- `snapToClosestEndpoint` / `snapGroupToClosestEndpoint` の対象レールが欠落し、軽減モード時にスナップ漏れが発生します。

**対象箇所**
- `getNearbyRails` の `endpoints.some(...)` 判定
- `snapToClosestEndpoint` / `snapGroupToClosestEndpoint` からの利用

**改善案**
- 端点包含判定ではなく、レールAABBと探索範囲AABBの交差判定に変更。
- `isRailInViewport_Test` で実施しているAABB算出を共通関数化して再利用。

---

### 2) [High] `skipDraw` をヒットテストに流用しており、操作不能領域が発生する
**問題**
- mousedown とコンテキストメニュー表示で `rails.filter(r => !r.skipDraw)` を使っています。
- `skipDraw` は描画最適化用の暫定フラグであり、入力可否の真理値としては不適切です。

**影響**
- 画面端付近・急速パン中など `skipDraw` の更新タイミング次第で、本来見えているレールを選択できない/メニューが出ない状態が起きます。

**対象箇所**
- mousedown 時の `checkRails`
- `showContextMenu` 内の `checkRails`

**改善案**
- ヒットテストは常に全レール対象にし、必要なら「簡易バウンディングによる早期除外」を別途導入。
- `skipDraw` は描画処理だけに閉じる。

---

### 3) [Medium] `scale < 0.5` でスナップ・選択・メニューを無効化しており、機能退行が大きい
**問題**
- 軽減モード時にズーム50%未満で、スナップ処理が return、選択が禁止、右クリックメニューも無効化されています。

**影響**
- ズームアウト中に編集操作できなくなり、ユーザーから見ると「壊れた」ように見える。

**対象箇所**
- `snapGroupToClosestEndpoint`
- `snapToClosestEndpoint`
- mousedown の選定処理
- `showContextMenu`

**改善案**
- 完全無効化ではなく、計算コストを段階的に落とす（探索半径縮小、候補上限、フレーム間引きなど）。
- どうしても無効化するならUIで明示し、ユーザーが理由を理解できるようにする。

---

### 4) [Low] `getTestViewportBounds(verbose = false)` の `verbose` 引数が未使用
**問題**
- 呼び出し側で `getTestViewportBounds(true)` としているものの、関数内で `verbose` を参照していません。

**影響**
- 「ログ制御がある」ように見えるが実態がなく、読み手に誤解を与える。

**改善案**
- 使わないなら引数を削除。
- 使うなら `if (verbose) console.debug(...)` 等を実装。

---

### 5) [Low] `updateSkipDrawFlags` の未使用変数 `bounds`
**問題**
- `const bounds = getTestViewportBounds();` が未使用です。

**影響**
- 実装意図が不明確になり、保守時のノイズになる。

**改善案**
- 不要なら削除。
- 使う想定なら `isRailInViewport_Test` に引数で渡して二重計算を避ける。

---

## 優先対応の提案（実施順）
1. **High 2件を先に修正**（入力処理の正しさに直結）。
2. その上で `scale < 0.5` の完全停止ロジックを段階的劣化へ変更。
3. 最後に未使用引数/変数を整理し、可読性を回復。
