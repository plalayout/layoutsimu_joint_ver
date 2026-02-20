# プラレールシミュレータ 技術仕様書 v47

## 📌 概要
プラレール（TOMY製玩具鉄道）のレイアウトシミュレータ。
Web技術（HTML5 Canvas + JavaScript）で実装。

**最新バージョン:** v47.11 (2026-02-20)
**ファイル:** plarail_simulator_v47_11_debug.html

**v47の主な変更:**
- **クラスタベース保存システム**: 配置履歴に依存せず、現在の接続状態のみを見てテキスト化
- **凸凹対応チェック**: 凸→凸、凹→凹の誤接続を防止
- **全レールタイプ対応**: 座標指定で全56種類のレールを正しく復元

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

---

## 🔌 端点（ジョイント）構造

### 端点タイプ
- **凹 (concave)**: 接続を受け入れる側（メス）
- **凸 (convex)**: 接続する側（オス）

### 接続の基本原則（v47で厳格化）
```
凸 → 凹: ✅ 接続可能
凸 → 凸: ❌ 接続不可
凹 → 凹: ❌ 接続不可
```

**重要**: v47では`isEndpointsConnected`関数で凸凹の対応をチェック
```javascript
if (ep1.type === ep2.type) {
    return false; // 同じタイプ同士は接続できない
}
```

### 端点ID命名規則
```
レールID-端点番号
例: "S1-1", "L5-2"

端点番号:
- 0, 1: 基本端点（通常は0=凹、1=凸）
- 2以降: 分岐端点（ポイントレール等）
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

---

## 💾 ショートカット保存システム v47

### v46からの変更点

#### v46（配置履歴ベース）
```javascript
// 問題: ドラッグ移動後、配置履歴が古い接続情報を保持
railPlacementHistory.push({
    id: rail.id,
    connectedFrom: { railId: prevRail.id, epsIndex: 1 },
    isReversed: false
});
// → ドラッグで動かしても「接続」として保存されてしまう
```

#### v47（接続状態ベース）
```javascript
// 解決: 現在の実際の接続状態のみを見る
// 配置履歴は使用しない
function saveLayoutAsShortcut() {
    const clusters = buildAllClusters(); // 接続グループを構築
    // 各クラスタを辿ってテキスト化
}
```

### 保存アルゴリズム

#### フェーズ1: クラスタ構築
```javascript
// 1. rails[]から順に未処理レールを取得
// 2. 幅優先探索（BFS）で接続されているレール全体を収集
// 3. rails[]の順序でソート（重要！）
function buildAllClusters() {
    const remainingRails = [...rails];
    const clusters = [];
    
    while (remainingRails.length > 0) {
        const startRail = remainingRails[0];
        const cluster = buildSingleCluster(startRail, remainingRails);
        
        // ★rails配列の順序でソート
        cluster.sort((a, b) => {
            const idxA = rails.findIndex(r => r.id === a.id);
            const idxB = rails.findIndex(r => r.id === b.id);
            return idxA - idxB;
        });
        
        clusters.push(cluster);
    }
    return clusters;
}
```

#### フェーズ2: 開始点決定
```javascript
function determineStartPoint(cluster) {
    const { unconnectedConcave, unconnectedConvex } = findUnconnectedEndpoints(cluster);
    
    if (unconnectedConcave.length > 0) {
        // ケースA: 凹端点あり → 通常進行
        return { ...unconnectedConcave[0], isReversed: false };
        
    } else if (unconnectedConvex.length > 0) {
        // ケースB: 凸端点のみ → 凹進行（vフラグ）
        return { ...unconnectedConvex[0], isReversed: true };
        
    } else {
        // ケースC: ループ → 先頭レールの凹端点から通常進行
        const firstRail = cluster[0];
        const eps = firstRail.getEndpoints();
        for (let i = 0; i < eps.length; i++) {
            if (eps[i].type === 'concave') {
                return { rail: firstRail, epsIndex: i, isReversed: false };
            }
        }
        return { rail: firstRail, epsIndex: 0, isReversed: false };
    }
}
```

#### フェーズ3: DFSで辿る
```javascript
function serializeOneSegment(cluster) {
    // 座標指定で開始
    result = `@${x},${y},${angle}${vFlag}:${shortcut}`;
    
    // DFS（深さ優先探索）でスタックを使って辿る
    const stack = [startRail];
    
    while (stack.length > 0) {
        const currentRail = stack[stack.length - 1];
        const nextConnection = findNextConnection(currentRail, cluster, visited);
        
        if (!nextConnection) {
            stack.pop(); // この方向は終了
            continue;
        }
        
        const { rail: nextRail, epsIndex: nextEpsIndex } = nextConnection;
        
        if (nextEpsIndex >= 2) {
            // eps[2]以降への接続 → 分岐として中断
            // nextRailはclusterに残す（次のセグメントで処理）
            stack.pop();
            continue;
        }
        
        // 通常接続: eps[0]から入るならショートカットのみ
        const addText = nextEpsIndex === 0 ? shortcut : `[${nextEpsIndex}]${shortcut}`;
        result += addText;
        
        cluster.remove(nextRail); // ★破壊的削除
        visited.add(nextRail.id);
        stack.push(nextRail);
    }
    
    return result;
}
```

### 接続判定の競合解決

#### 問題: 同一位置に複数端点
```
S-R1-R2-R3-R4-R5-R6-R7-R8-L
^                           ^
折り返しで同じ座標
```

#### 解決: 配列順序で優先判定
```javascript
function isEndpointsConnected(ep1, ep2, rail1, rail2, cluster) {
    // 座標・角度チェック
    if (dx >= 0.5 || dy >= 0.5 || da >= 5) return false;
    
    // ★凸凹対応チェック（v47で追加）
    if (ep1.type === ep2.type) return false;
    
    // 競合レールをチェック
    const competitorsAtEp1 = findCompetingRails(ep1, rail1, cluster, 0.5, 5);
    if (competitorsAtEp1.length > 0) {
        if (!isClosestNeighbor(rail1, rail2, competitorsAtEp1, cluster)) {
            return false; // より近い競合者がいる
        }
    }
    
    return true;
}

// rails配列での距離で判定（v47で修正）
function isClosestNeighbor(baseRail, targetRail, competitors, cluster) {
    const baseIdx = rails.findIndex(r => r.id === baseRail.id);
    const targetIdx = rails.findIndex(r => r.id === targetRail.id);
    const targetDist = Math.abs(baseIdx - targetIdx);
    
    for (let competitor of competitors) {
        const competitorIdx = rails.findIndex(r => r.id === competitor.id);
        const competitorDist = Math.abs(baseIdx - competitorIdx);
        
        if (competitorDist < targetDist) {
            return false; // より近い競合者がいる
        }
    }
    return true;
}
```

### ショートカット表記

#### 基本構文
```
@x,y,angle[v]:ショートカット連続

例:
@0.0,0.0,0:SRL         # 座標指定 → S, R, L連続
@100.0,200.0,45v:LR    # 凹進行フラグ付き
POl.R[2]R              # ポイント → R → eps[2]からR
```

#### フラグ
- `v`: 凹進行フラグ（凸端点から開始）
- `[n]`: 分岐番号（eps[n]から接続）

#### 接続表記
```
R        # eps[0]（凹）から入る = 通常接続
[1]R     # eps[1]（凸）から入る
[2]R     # eps[2]から入る（分岐）
```

---

## 🔧 座標指定レール配置

### addRailAtPosition関数（v47で全レール対応）

```javascript
function addRailAtPosition(x, y, angle, railType, options, isReversed = false) {
    const id = `${railType[0].toUpperCase()}${rails.length + 1}`;
    let rail = null;
    
    // 全56種類のレールタイプに対応
    switch(railType) {
        case 'straight':
            rail = new StraightRail(x, y, id);
            break;
        case 'curve':
            rail = new Curve(x, y, id, options.direction || 'right');
            break;
        case 'turnout':
            rail = new TurnoutRail(x, y, id, options.type || 'R', options.direction || 'left');
            break;
        case 'quarter':
            rail = new QuarterRail(x, y, id, options.type || 'CC', options.reversed || false);
            break;
        // ... 他のレールタイプ
        default:
            console.error('未対応のレールタイプ:', railType);
            return;
    }
    
    if (rail) {
        rail.angle = angle; // ★角度を正しく設定
        rails.push(rail);
        updateProgressionAfterAdd(rail, false);
        
        if (isReversed) {
            railProgression.isReversed = true;
            railProgression.endpointIndex = 0;
        }
        
        railPlacementHistory.push({ 
            id: rail.id, 
            connectedFrom: null, 
            isReversed: isReversed
        });
        
        drawAll(); // ★必須: レールを描画
    }
}
```

### 対応レールタイプ（v47）
- ✅ `straight`, `curve`, `turnout`
- ✅ `quarter`, `sixth`, `eighth`
- ✅ `slope`, `slide`
- ✅ `joint`, `uturn`
- ✅ `doubleStraight`, `doubleCurve`
- ✅ `figure8Point`, `threeway`
- ✅ `r13Point`, `r22YPoint`
- ✅ `autoPoint`, `autoDoublePoint`
- ✅ `bridge`, `stop`, `bumper`

---

## 🐛 既知の問題（v47.11時点）

### 1. ポイントレール類が単独配置で消える
**症状**: ポイントレールを座標指定で単独配置すると、画面に表示されない

**デバッグ状況**:
- v47.11でデバッグログ追加済み
- `addRailAtPosition`でのレール生成・`rails.push`・`drawAll()`呼び出しを確認中

**確認すべき点**:
1. コンソールでレール生成成功ログが出るか
2. `TurnoutRail`コンストラクタがエラーを出していないか
3. `drawAll()`後にCanvasに描画されているか

---

## 🔍 デバッグ

### v47のデバッグログ
```javascript
// 保存時
console.log('=== saveLayoutAsShortcut 開始 ===');
console.log('クラスタ数:', clusters.length);
console.log('レールID:', cluster.map(r => r.id).join(', '));
console.log('未接続凹:', unconnectedConcave.length);
console.log('開始: S1 → @0.0,0.0,0:S');
console.log('[1] S1 → R2[0] : R');

// 読み込み時
console.log('addRailAtPosition:', { x, y, angle, railType, options });
console.log('Creating TurnoutRail:', { x, y, id, type, direction });
console.log('Rail created successfully:', rail.id);
```

### ブラウザ開発者ツール（F12）
```javascript
// 状態確認
console.log('Rails:', rails.length);
console.log('Selected:', selectedShape?.id);
console.log('Cluster:', cluster.map(r => r.id));
```

---

## 📚 参考情報

### トランスクリプト
- `/mnt/transcripts/2026-02-19-12-56-29-v46-9-to-v46-16-connection-tracking-fixes.txt`
  - v46.9-16: 配置履歴構造変更、接続情報記録
- 本セッション: v47クラスタベース保存システム実装

### バージョン履歴（v47系）
- **v47.0**: クラスタベース保存システム導入
- **v47.1**: 連続配置の修正（savedSetチェック削除）
- **v47.2**: serializeCluster完全修正
- **v47.3**: クラスタベース保存実装完了
- **v47.4**: 開始点判定修正（凹→通常、凸→凹進行）
- **v47.5**: 通常接続判定修正（eps[0]が通常）
- **v47.6**: 凸凹対応チェック追加
- **v47.7**: 距離判定をrails配列で行うように修正
- **v47.8**: クラスタをrails配列順でソート
- **v47.9**: 全レールタイプ対応（addRailAtPosition拡張）
- **v47.10**: drawAll()呼び出し追加
- **v47.11**: デバッグログ追加（ポイントレール問題調査中）

---

## 🚀 新スレッドでの作業開始方法

```
プラレールシミュレータ v47の続きです。

最新ファイル: plarail_simulator_v47_11_debug.html
仕様書: plarail_simulator_specification_v47.md
トランスクリプト: /mnt/transcripts/[対応するファイル]

[作業内容を記載]
```

新Claudeは必要に応じて:
1. この仕様書を参照
2. トランスクリプトで詳細確認
3. 実装ファイルを直接確認

---

**仕様書バージョン:** v47  
**作成日:** 2026-02-20  
**対象ファイル:** plarail_simulator_v47_11_debug.html
