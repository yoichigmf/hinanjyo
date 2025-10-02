# FlatGeobufファイルをMapLibre GL JSで表示するWebページの作成過程

## プロジェクト概要
ローカルフォルダにある2つのFlatGeobuf（.fgb）ファイルをMapLibre GL JSを使用してWeb上に表示する地図アプリケーションを作成しました。

## 使用データ
- `siteihinanjyo.fgb` - 指定避難所データ（34.95MB）
- `siteihinanbasyo.fgb` - 指定緊急避難場所データ（21.32MB）

## 技術スタック
- **MapLibre GL JS v4.7.1** - オープンソースの地図ライブラリ
- **FlatGeobuf v3.31.0** - 効率的な地理空間データフォーマット
- **地理院タイル** - 背景地図として使用

## 実装手順

### 1. 初期実装
基本的なHTMLページを作成し、以下の機能を実装：

- MapLibre GL JSと FlatGeobuf ライブラリの読み込み
- 地理院タイルを背景地図として設定
- 2つのFlatGeobufファイルを非同期で読み込み
- GeoJSONに変換してマップソースとして追加
- 円形マーカーでポイントを表示
  - 指定避難所：赤色（#FF6B6B）
  - 指定緊急避難場所：水色（#4ECDC4）
- 全データが表示されるように自動的にズーム調整
- クリック時のポップアップ表示（属性情報）
- マウスホバー時のカーソル変更

### 2. レイヤー表示/非表示機能の追加
ユーザーからの要望により、各レイヤーの表示/非表示を切り替える機能を追加：

#### UIの改善
- 凡例の各項目にIDを付与（`hinanjyo-toggle`, `hinanbasyo-toggle`）
- クリック可能なスタイルを追加（カーソルポインター、選択不可）
- 非表示状態を視覚的に表現（透明度40%、灰色表示）

#### JavaScript機能
- 各凡例項目にクリックイベントリスナーを追加
- `map.getLayoutProperty()` でレイヤーの現在の表示状態を取得
- `map.setLayoutProperty()` でレイヤーの visibility プロパティを切り替え
- CSSクラスの追加/削除で視覚的フィードバックを提供

### 3. 領域フィルタリング読み込みと位置情報対応
パフォーマンス改善とユーザビリティ向上のため、データ読み込み方式を変更：

#### 現在位置取得機能
- Geolocation APIを使用してブラウザの位置情報を取得
- 位置情報が取得できない場合は立川駅（139.4073, 35.6976）を初期位置として設定
- 初期ズームレベル12で表示

#### 領域フィルタリング読み込み
- 全データを一度に読み込む方式から、表示領域のデータのみを読み込む方式に変更
- `flatgeobuf.deserialize()`の第2引数にbbox（境界ボックス）を渡すことで空間フィルタリング
- 地図の移動・ズーム時（`moveend`イベント）に表示領域のデータを動的に再読み込み
- 大容量データ（合計56MB）でも軽快に動作

#### 実装の利点
- 初期読み込み時間の大幅短縮
- メモリ使用量の削減
- ユーザーの現在位置周辺の避難所を即座に表示
- ネットワーク帯域の効率的な使用

## コードの主要部分

### 現在位置取得とマップ初期化
```javascript
async function initializeMap() {
    let initialCenter = TACHIKAWA_STATION;

    // Geolocation APIで現在位置を取得
    if (navigator.geolocation) {
        try {
            const position = await new Promise((resolve, reject) => {
                navigator.geolocation.getCurrentPosition(resolve, reject, {
                    timeout: 5000,
                    maximumAge: 0
                });
            });
            initialCenter = [position.coords.longitude, position.coords.latitude];
        } catch (error) {
            console.log('位置情報取得失敗、立川駅を中心に表示します:', error.message);
        }
    }

    map = new maplibregl.Map({
        container: 'map',
        center: initialCenter,
        zoom: 12
        // ... style定義
    });
}
```

### 領域フィルタリングでのFlatGeobuf読み込み
```javascript
async function loadVisibleFeatures() {
    const bounds = map.getBounds();
    const bbox = {
        minX: bounds.getWest(),
        minY: bounds.getSouth(),
        maxX: bounds.getEast(),
        maxY: bounds.getNorth()
    };

    // bboxを指定して表示領域内のデータのみ取得
    const response = await fetch('./siteihinanjyo.fgb');
    const iterator = flatgeobuf.deserialize(response.body, bbox);
    const features = [];
    for await (const feature of iterator) {
        features.push(feature);
    }

    map.getSource('hinanjyo').setData({
        type: 'FeatureCollection',
        features: features
    });
}

// 地図移動時に再読み込み
map.on('moveend', loadVisibleFeatures);
```

### レイヤー表示切り替え
```javascript
hinanjyoToggle.addEventListener('click', () => {
    hinanjyoLayerVisible = !hinanjyoLayerVisible;
    if (hinanjyoLayerVisible) {
        map.setLayoutProperty('hinanjyo-layer', 'visibility', 'visible');
        hinanjyoToggle.classList.remove('disabled');
    } else {
        map.setLayoutProperty('hinanjyo-layer', 'visibility', 'none');
        hinanjyoToggle.classList.add('disabled');
    }
});
```

## 機能一覧

### 地図表示機能
- ✅ FlatGeobufファイルの領域フィルタリング読み込み
- ✅ 地理院タイルを背景地図として表示
- ✅ 2種類のポイントデータを異なる色で表示
- ✅ ブラウザの位置情報による初期位置設定（フォールバック：立川駅）
- ✅ 初期ズームレベル12での表示
- ✅ 地図移動時の動的データ再読み込み

### インタラクション機能
- ✅ ポイントクリックで属性情報をポップアップ表示
- ✅ マウスホバーでカーソルがポインターに変化
- ✅ 凡例クリックでレイヤーの表示/非表示切り替え
- ✅ 非表示時の視覚的フィードバック

## 使用方法

### ローカルサーバーの起動
FlatGeobufファイルはローカルから読み込むため、HTTPサーバーが必要です：

```bash
# Python 3の場合
python -m http.server

# Node.jsの場合
npx serve

# PHPの場合
php -S localhost:8000
```

### ブラウザでアクセス
サーバー起動後、`http://localhost:8000/index.html` にアクセス

## ファイル構成
```
D:\work\2025避難所\upload\
├── index.html              # メインのHTMLファイル
├── siteihinanjyo.fgb       # 指定避難所データ
├── siteihinanbasyo.fgb     # 指定緊急避難場所データ
└── CLAUDE.md               # このドキュメント
```

## 今後の拡張可能性
- フィルタリング機能（避難所の種類別）
- 検索機能（名称や住所で検索）
- ルート案内機能
- データのCSVエクスポート
- レスポンシブデザインの改善
