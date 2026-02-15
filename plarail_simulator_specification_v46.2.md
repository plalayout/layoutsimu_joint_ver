# プラレールシミュレータ 技術仕様書 v46.2

## 📌 概要
プラレール（TOMY製玩具鉄道）のレイアウトシミュレータ。
Web技術（HTML5 Canvas + JavaScript）で実装。

**最新バージョン:** v46.2 (2026-02-15)
**ファイル:** plarail_simulator_final_v46.2.html

---

## 🎯 座標系・角度の仕様

### 座標系
```
        Y↑
        |
        |
        |
   -----+----→ X
        |
        |
```

- **ワールド座標**: 数学的座標系（Y軸上向き）
- **Canvas内部座標**: 左上原点、Y軸下向き（描画時に変換）
- **原点**: Canvas中央

### 角度の定義
```
      0° (Y軸正方向 = 上)
       |
       |
270°---+---90°
       |
       |
     180°
```

- **0度**: Y軸正方向（画面上向き）
- **回転方向**: 反時計回り = 正
- **単位**: 度（degree）、内部計算はラジアン変換

### 座標変換関数
```javascript
// スクリーン座標 → ワールド座標
function screenToWorld(screenX, screenY) {
    const scaleX = canvas.width / rect.width;
    const scaleY = canvas.height / rect.height;
    const canvasX = screenX * scaleX;
    const canvasY = screenY * scaleY;
    const worldX = (canvasX - canvasOffsetX) / scale;
    const worldY = (canvH - canvasY - canvasOffsetY) / scale;
    return { x: worldX, y: worldY };
}

// ワールド座標 → スクリーン座標
function worldToScreen(worldX, worldY) {
    const screenX = worldX * scale + canvasOffsetX;
    const screenY = canvH - (worldY * scale + canvasOffsetY);
    return { x: screenX, y: screenY };
}
```

---

## 🔌 端点（ジョイント）構造

### 端点タイプ
- **凹 (concave)**: 接続を受け入れる側（メス）
- **凸 (convex)**: 接続する側（オス）

### 端点ID命名規則
```
レールID-端点番号
例: "S1-1", "L5-2"

端点番号:
- 奇数 (1, 3, 5...): 凹
- 偶数 (2, 4, 6...): 凸
```

### 端点オブジェクト構造
```javascript
{
    x: number,        // X座標
    y: number,        // Y座標
    angle: number,    // 角度（度）
    id: string,       // "レールID-端点番号"
    type: string      // 'concave' または 'convex'
}
```

### 実在製品の端点構造
```
直線レール:
  起点(1): 凹 → 終点(2): 凸

カーブレール:
  起点(1): 凹 → 終点(2): 凸

ポイントレール（分岐）:
  起点(1): 凹 → 直線終点(2): 凸
                  曲線終点(4): 凸
```

---

## 🚂 レールクラス構造

### 基本レールクラス

#### StraightRail（直線レール）
```javascript
class StraightRail {
    constructor(x, y, id, length = RAIL_LENGTH, options = {}) {
        this.x = x;              // 起点X座標
        this.y = y;              // 起点Y座標
        this.id = id;            // レールID
        this.angle = 0;          // 角度（度）
        this.length = length;    // 長さ（デフォルト: 80）
        this.width = RAIL_WIDTH; // 幅（14）
        this.color = COLORS.STRAIGHT.base;
        this.highlightColor = COLORS.STRAIGHT.highlight;
        this.startType = 'concave';  // 起点タイプ
        this.endType = 'convex';     // 終点タイプ
    }
    
    draw() { /* リアル/シンプル切替 */ }
    drawWithWidth(width, color) { /* 指定幅・色で描画 */ }
    contains(mx, my) { /* 当たり判定 */ }
    getEndpoints() { /* 端点取得 */ }
}
```

#### Curve（カーブレール）
```javascript
class Curve {
    constructor(x, y, id, direction = DIRECTION_LEFT, radius = RAIL_LENGTH) {
        this.x = x;
        this.y = y;
        this.id = id;
        this.angle = 0;
        this.direction = direction;  // 'left' or 'right'
        this.radius = radius;        // 半径（デフォルト: 80）
        this.turnAngle = 45;         // 曲がる角度（度）
        this.color = COLORS.CURVE.base;
        this.highlightColor = COLORS.CURVE.highlight;
    }
    
    draw() { /* リアル/シンプル切替 */ }
    drawWithWidth(width, color) { /* 円弧描画 */ }
    contains(mx, my) { /* 円弧上の当たり判定 */ }
    getEndpoints() { /* 端点取得 */ }
}
```

#### Slide（スライドレール）
```javascript
class Slide {
    constructor(x, y, id, slide, length = RAIL_LENGTH) {
        this.x = x;
        this.y = y;
        this.id = id;
        this.angle = 0;
        this.slide = slide;     // 横方向オフセット（±22）
        this.length = length;   // 長さ
        this.color = COLORS.SLIDE.base;
        this.highlightColor = COLORS.SLIDE.highlight;
    }
    
    draw() { /* ベジェ曲線 */ }
    drawWithWidth(width, color) { /* ベジェ曲線描画 */ }
    contains(mx, my) { /* ベジェ曲線上の当たり判定 */ }
    getEndpoints() { /* 端点取得 */ }
}
```

### ポイントレールクラス（組み合わせ型）

#### 基本構造
```javascript
class TurnoutRail {
    constructor(x, y, id, turnoutType = 'R', direction = 'left') {
        this.x = x;
        this.y = y;
        this.id = id;
        this.angle = 0;
        this.turnoutType = turnoutType;  // 'R' or 'L'
        this.direction = direction;      // 'left' or 'right'
        this.components = [];            // 基本レールの配列
        this.color = COLORS.TURNOUT.base;
        this.highlightColor = COLORS.TURNOUT.highlight;
    }
    
    _buildComponents() {
        // StraightRail + Curve を組み合わせ
        const straight = new StraightRail(this.x, this.y, this.id + '-S');
        const curve = new Curve(this.x, this.y, this.id + '-C', this.direction);
        // 色をコピー
        straight.color = this.color;
        straight.highlightColor = this.highlightColor;
        curve.color = this.color;
        curve.highlightColor = this.highlightColor;
        this.components.push(straight, curve);
    }
    
    _sync() {
        this._buildComponents();
    }
    
    draw() {
        this._sync();
        
        if (useRealisticRendering) {
            // リアル描画: 層ごと全体描画
            this.components.forEach(c => c.drawWithWidth(14, 'black'));
            this.components.forEach(c => c.drawWithWidth(12, this.color));
            this.components.forEach(c => c.drawWithWidth(6, this.highlightColor));
            this.components.forEach(c => c.drawWithWidth(4, this.color));
        } else {
            // シンプル描画
            this.components.forEach(c => c.draw());
        }
    }
    
    contains(mx, my) {
        this._sync();
        return this.components.some(c => c.contains(mx, my));
    }
    
    getEndpoints() {
        this._sync();
        // 各コンポーネントの端点を集約
        // ...
    }
}
```

#### ポイントレール一覧
- **TurnoutRail**: ターンアウト（直線+カーブ分岐）
- **R13PointRail**: 単複ポイント（直線+スライド）
- **R22YPointRail**: Y字ポイント（スライド組み合わせ）
- **Figure8PointRail**: 8の字ポイント（カーブ2本）
- **CrossPointRail**: 十字ポイント（カーブ4本＋装飾2本）
- **R30TrianglePointRail**: 三角ポイント（直線+カーブ）
- **UTurnRail**: Uターンレール（カーブ3本＋装飾1本）
- **DoubleTrackTurnoutRail**: 複線ターンアウト
- **DoubleTrackCrossRail**: 複線渡り

---

## 🎨 描画システム

### 表示モード（useDetailedEndpoints, useRealisticRendering）

#### 詳細モード（デフォルト）
- **端点表示**: 線分（実際のジョイント形状）
- **レール描画**: 4層リアル描画

#### 簡易モード
- **端点表示**: 〇（シンプル）
- **レール描画**: 単色

### リアル描画（4層構造）
```
第1層: lineWidth 14px, 黒色 (ベース)
第2層: lineWidth 12px, レール色 (サイド)
第3層: lineWidth  6px, ハイライト色 (明るい部分)
第4層: lineWidth  4px, レール色 (中央)

視覚効果:
━━━━━━━━━━━━━━  ← 14px 黒
  ━━━━━━━━━━━━    ← 12px レール色（両端1pxずつ黒見える）
    ━━━━━━━━      ← 6px ハイライト（両端3pxずつレール色）
      ━━━━        ← 4px レール色（両端1pxずつハイライト）
```

### 色定数（COLORS）
```javascript
const COLORS = {
    STRAIGHT: { base: 'deepskyblue', highlight: 'lightskyblue' },
    CURVE: { base: 'pink', highlight: 'lightpink' },
    SLIDE: { base: 'pink', highlight: 'lightpink' },
    TURNOUT: { base: 'lawngreen', highlight: 'lightgreen' },
    FIGURE8: { base: 'plum', highlight: 'violet' },
    R13: { base: 'orange', highlight: 'lightsalmon' },
    R22: { base: 'yellowgreen', highlight: 'lightgreen' },
    R15: { base: 'mediumseagreen', highlight: 'aquamarine' },
    R30: { base: 'mediumseagreen', highlight: 'aquamarine' },
    CROSS: { base: 'coral', highlight: 'lightsalmon' },
    UTURN: { base: 'red', highlight: 'lightcoral' },
    DOUBLE_CURVE: { base: 'violet', highlight: 'plum' },
    OUTER_CURVE: { base: 'purple', highlight: 'mediumorchid' },
    DOUBLE_TRACK_STRAIGHT: { base: 'cyan', highlight: 'lightcyan' },
    DOUBLE_TRACK_CURVE: { base: 'lightcyan', highlight: 'azure' },
    DOUBLE_TRACK_CROSS: { base: 'turquoise', highlight: 'paleturquoise' },
    DOUBLE_TRACK_TURNOUT: { base: 'lightseagreen', highlight: 'aquamarine' },
    QUARTER: { base: 'skyblue', highlight: 'lightskyblue' },
    SIXTH: { base: 'skyblue', highlight: 'lightskyblue' },
    HALF_STRAIGHT: { base: 'lightblue', highlight: 'lightcyan' },
    DOUBLE_STRAIGHT: { base: 'navy', highlight: 'royalblue' }
};
```

### drawWithWidth()メソッド実装例
```javascript
// 直線
drawWithWidth(width, color) {
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.rotate(this.angle * Math.PI / 180);
    ctx.fillStyle = color;
    ctx.fillRect(-width / 2, 0, width, this.length);
    ctx.restore();
}

// カーブ
drawWithWidth(width, color) {
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.rotate(this.angle * Math.PI / 180);
    ctx.strokeStyle = color;
    ctx.lineWidth = width;
    ctx.lineCap = 'butt';
    ctx.beginPath();
    if (this.direction === DIRECTION_LEFT) {
        ctx.arc(-this.radius, 0, this.radius, 0, this.turnAngle * Math.PI / 180);
    } else {
        ctx.arc(this.radius, 0, this.radius, Math.PI - this.turnAngle * Math.PI / 180, Math.PI);
    }
    ctx.stroke();
    ctx.restore();
}
```

### 装飾パーツの扱い
一部のポイントレール（CrossPointRail, UTurnRail）には走行路でない装飾パーツが含まれる。

```javascript
// 装飾パーツ: 2層のみ（黒+レール色）
this.components.slice(-2).forEach(c => c.drawWithWidth(14, 'black'));
this.components.slice(-2).forEach(c => c.drawWithWidth(12, this.color));

// 走行路部分: 4層
this.components.slice(0, 4).forEach(c => c.drawWithWidth(14, 'black'));
this.components.slice(0, 4).forEach(c => c.drawWithWidth(12, this.color));
this.components.slice(0, 4).forEach(c => c.drawWithWidth(6, this.highlightColor));
this.components.slice(0, 4).forEach(c => c.drawWithWidth(4, this.color));
```

---

## 🖱️ インタラクション

### Canvas設定
```javascript
Canvas解像度: 2400 × 1350 (16:9)
初期スケール: 3.0
```

### マウスイベント
```javascript
// mousedown: レール選択 or パン開始
// mousemove: レール移動 or パン
// mouseup: 移動終了、スナップ処理
// wheel: ズーム（マウス位置中心）
// contextmenu: コンテキストメニュー
```

### タッチイベント
```javascript
// touchstart: レール選択 or パン開始、長押し検出開始
// touchmove: レール移動 or パン、2本指でピンチズーム
// touchend: 移動終了、スナップ処理、タップ検出
```

### 座標変換の重要性
```javascript
// パンニング処理では必ず内部座標に変換
const rect = canvas.getBoundingClientRect();
const scaleX = canvas.width / rect.width;   // 2400 / 表示幅
const scaleY = canvas.height / rect.height; // 1350 / 表示高さ

const dx = (e.offsetX - startX) * scaleX;
const dy = (e.offsetY - startY) * scaleY;

canvasOffsetX += dx;
canvasOffsetY -= dy;
```

---

## 🔧 主要機能

### スナップ機能
```javascript
function snapToClosestEndpoint(shape) {
    // 最も近い端点を検索
    // 距離 < 20 の場合:
    //   1. 角度を合わせる
    //   2. 位置を合わせる
}
```

### グループ移動
```javascript
// グリップ検出: 端点から20px以内をクリック
// 接続されたレール群を一括移動
// BFS（幅優先探索）で接続グラフを構築
```

### 進行モード
```javascript
// selectedEndpoint: 最後に選択した端点
// 新しいレールを追加時、この端点から接続
// 自動スナップ
```

### レール数カウント
```javascript
function countRails() {
    // クラス名、長さ、半径、端点タイプで分類
    // 直線、1/2直線、2倍直線
    // カーブ、外カーブ、倍曲線
    // ポイント各種（A型/B型/R型/L型など）
}
```

### レイアウト保存/読込
```javascript
// テキスト形式（セミコロン区切り）
// 形式: クラス名,x,y,angle,direction,その他パラメータ;
function saveLayoutToText() { /* ... */ }
function loadLayoutFromText(text) { /* ... */ }
```

---

## 📐 定数

```javascript
const RAIL_LENGTH = 80;        // 基本レール長さ
const RAIL_WIDTH = 14;         // レール幅
const D_TRACK_WIDTH = 22;      // 複線幅
const JOINT_WIDTH = 4;         // ジョイント幅
const JOINT_LENGTH = 4;        // ジョイント長さ
const DIRECTION_LEFT = 'left';
const DIRECTION_RIGHT = 'right';
```

---

## 🎛️ グローバル変数

```javascript
const rails = [];              // 全レール配列
let dragging = null;           // ドラッグ中のレール
let draggingGroup = null;      // ドラッグ中のグループ
let selectedShape = null;      // 選択中のレール
let selectedEndpoint = null;   // 選択中の端点（進行モード用）
let scale = 3.0;               // ズーム倍率
let canvasOffsetX = canvW/2;   // パンオフセットX
let canvasOffsetY = canvH/2;   // パンオフセットY
let isPanning = false;         // パン中フラグ
let groupMoveEnabled = false;  // グループ移動モード

// 表示モード
let useDetailedEndpoints = true;   // 詳細端点表示（デフォルト: true）
let useRealisticRendering = true;  // リアル描画（デフォルト: true）
let isNeedJointConsole = false;    // デバッグログ（デフォルト: false）
```

---

## 📝 命名規則

### レールID
```
S: 直線 (Straight)
L: 左カーブ (Left curve)
R: 右カーブ (Right curve)
Sl: スライド (Slide)
T: ターンアウト (Turnout)
F: 8の字 (Figure 8)
U: Uターン (U-turn)
C: 十字 (Cross)

例: S1, L5, R3, T2-S (ターンアウトの直線部分)
```

### 端点ID
```
形式: レールID-端点番号
例: S1-1 (直線レールS1の起点・凹)
    S1-2 (直線レールS1の終点・凸)
    T2-4 (ターンアウトT2のカーブ終点・凸)
```

---

## 🐛 既知の制約・注意点

### 1. 角度の正規化
```javascript
// 360度を超える角度は正規化
angle = (angle + 360) % 360;
```

### 2. 端点検出の優先順位
```javascript
// 1. 凸端点（グリップ）: 半径8px
// 2. 凹端点: 半径4px
```

### 3. Canvas解像度とパンニング
```javascript
// 必ず scaleX/scaleY で変換
const scaleX = canvas.width / rect.width;
```

### 4. ポイントレールの_sync()
```javascript
// draw(), contains(), getEndpoints() の前に必ず呼ぶ
// components配列を毎回再構築
```

### 5. 色のコピー
```javascript
// ポイントレールの各コンポーネントに色をコピー
straight.color = this.color;
straight.highlightColor = this.highlightColor;
```

---

## 🔍 デバッグ

### コンソールログ制御
```javascript
let isNeedJointConsole = false;  // true で端点関連ログ出力

if (isNeedJointConsole) {
    console.log('Endpoint gripped:', endpoint.id);
}
```

### ブラウザ開発者ツール
```javascript
// Canvasの状態確認
console.log('Rails:', rails.length);
console.log('Scale:', scale);
console.log('Offset:', canvasOffsetX, canvasOffsetY);
console.log('Selected:', selectedShape?.id);
```

---

## 📚 参考情報

### トランスクリプト
- `/mnt/transcripts/2026-02-15-05-59-34-plarail-v42-v44-joint-canvas-zoom-rendering.txt`
- 詳細な開発履歴・仕様議論

### バージョン履歴（抜粋）
- **v42**: ジョイント描画実装
- **v43-44**: Canvas拡大、マウス位置中心ズーム
- **v45**: リアル描画（4層）実装
- **v46**: 表示モード統合、16:9対応、ドラッグ修正

---

## 🚀 新スレッドでの作業開始方法

```
プラレールシミュレータ v46.2の続きです。

最新ファイル: plarail_simulator_final_v46.2.html
仕様書: plarail_simulator_specification_v46.2.md
トランスクリプト: /mnt/transcripts/2026-02-15-05-59-34-plarail-v42-v44-joint-canvas-zoom-rendering.txt

[作業内容を記載]
```

新Claudeは必要に応じて:
1. この仕様書を参照
2. トランスクリプトで詳細確認
3. 実装ファイルを直接確認

---

**仕様書バージョン:** v46.2  
**作成日:** 2026-02-15  
**対象ファイル:** plarail_simulator_final_v46.2.html
