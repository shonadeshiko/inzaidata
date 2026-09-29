# 印旛沼流域 GIS解析プラットフォーム（高解像度版）

千葉県印旛沼流域を対象に、空間演算・ラスタ解析・ポテンシャル測定を行い、
結果を静的サイトとして公開するためのリポジトリです。

姉妹リポジトリ [gisdata](https://github.com/shonadeshiko/gisdata)（印旛沼流域版）の
コード構成をコピーし、より高解像度なデータへの置き換えを想定した雛形です。
**実データはまだ投入されていません**（後日取り込み予定）。

## 設計方針

- **原則として静的配信**。サーバーを常時起動させず、コストを最小化する
- **重い処理（ラスタ解析・流域解析など）は事前バッチで計算**し、結果を
  Cloud Optimized GeoTIFF（COG）として書き出す
- **軽い計算（バッファ集計など）はブラウザ内**（turf.js + geotiff.js）で
  オンデマンド実行する
- データが継続的に増えても、パイプラインを再実行するだけで
  成果物を再生成できるようにする（メンテナンス性）

## 構成

```
inzaidata/
├── pipeline/
│   ├── process/
│   │   ├── ingest_inzai_data.py       # 印旛沼流域の実データ取り込み(雛形、RASTER_DEFS等は空)
│   │   └── ingest_gsi_elevation.py    # 国土地理院 標高タイル取得
│   └── requirements.txt
├── data/
│   ├── raw/inzai/{raster,vector}/     # 元データの置き場（.gitignore対象）
│   └── processed/                     # 生成物（COG, GeoJSON）
├── web/
│   └── index.html                     # フロントエンド（MapLibre GL + turf.js + geotiff.js）
├── .github/workflows/
│   ├── pipeline.yml                   # ビルド・GitHub Pages公開
│   └── ingest-elevation.yml           # 標高データ取得（手動実行）
└── README.md
```

## セットアップ（ローカルでパイプラインを試す）

```bash
cd pipeline
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python process/ingest_inzai_data.py   # 元データが data/raw/inzai/ にある場合
```

`pipeline/process/ingest_inzai_data.py` の `RASTER_DEFS` /
`VECTOR_DEFS` は現在空（`[]`）。実データが届いたら、
gisdataの `ingest_chiba_data.py` の書き方を参考に定義を追加していく。
連続値のデータをuint8で軽量化したい場合は `uint8_scale` を指定する
（例: 10を指定すると値を10倍してuint8(0-255)に丸めて保存し、
frontend側で10で割り戻す）。

`RASTER_DEFS` の `"src"` にファイル名のリストを渡すと、複数ファイル
（例: 分割されたデータ）をモザイク結合してから処理する
（`merge_rasters()` 参照。shimamotodataで京都府・大阪府にまたがる
データを結合するために追加した仕組みを流用）。

## フロントエンドの確認

```bash
mkdir -p web/data
cp -r data/processed/* web/data/
cd web
python -m http.server 8000
```

（GitHub Actionsでは、この処理が自動的に行われる）

## ベースマップについて

デフォルトは国土地理院の淡色地図（`gsi_pale`）。`web/index.html` 内の
`BASE_LAYERS` 配列にエントリを1つ追加すると、画面上部のセレクタに
選択肢が増える仕組みになっている。

「旧版地形図」は、gisdata(千葉県版)と同じ印旛沼流域が対象のため、
同じ関東地方(kanto)の今昔マップ
（`https://ktgis.net/kjmapw/kjtilemap/kanto/00/{z}/{x}/{y}.png`）
を使用（TMS方式のため `scheme: "tms"` を指定）。

## メッシュの色分けについて

`web/index.html` の `MESH_SCORES` は、gisdata側の500mメッシュ
(GI保全スコア・GI開発圧スコア・優先度ランク・市街地率・森林率)の
属性名をそのまま踏襲した雛形。印旛沼流域の高解像度メッシュデータが
同じ属性名で作られる前提になっているため、**実データが届いたら
フィールド名・domain(色分けの値range)・凡例が合っているか確認すること**。

## 標高データ（国土地理院 標高タイル）について

`pipeline/process/ingest_gsi_elevation.py` は、国土地理院の標高タイル
(DEM10B, 10mメッシュ)を印旛沼流域周辺の範囲だけダウンロードし、
モザイク結合・EPSG:4326への再投影・COG化を行う。BBOXはgisdataと
同じ印旛沼流域の範囲を設定済み。より高解像度なデータに置き換える
場合は、スクリプト内の `ZOOM` の見直しを検討すること
（zoom14相当だとファイルサイズが80MBを超える点に注意）。

外部ネットワークへの多数アクセスが必要で時間もかかるため、通常の
pushでは実行せず、GitHub Actionsの `Ingest GSI Elevation Data`
ワークフロー（手動実行 = workflow_dispatch）でのみ実行する。
実行結果（COGファイルとカタログ更新）はリポジトリにコミットされ、
以後は固定データとして配信される。

## バッファ解析について

地図をクリックすると、その地点を中心とした指定半径のバッファ内で、
カタログに登録されている全ラスタの平均・最小・最大値をブラウザ内
（geotiff.js + turf.js）でオンデマンド計算する。サーバー計算は不要。
一度取得したラスタはブラウザ内にキャッシュされ、2回目以降のクリックで
再取得しないようになっている。

## 今後のTODO

- [ ] 印旛沼流域の高解像度な実データ（ラスタ・ベクタ）を
      `data/raw/inzai/` に配置し、`ingest_inzai_data.py` の
      `RASTER_DEFS` / `VECTOR_DEFS` に登録
- [ ] メッシュデータの属性名がgisdata版と異なる場合、`MESH_SCORES` を調整
- [ ] 高解像度化に伴うファイルサイズの検証（shimamotodataで京都・大阪
      広域化した際、メッシュGeoJSONが約26MBまで増えた前例あり。
      必要に応じてジオメトリ簡略化・タイル化を検討）
- [ ] `ingest_gsi_elevation.py` の `ZOOM` を対象解像度に合わせて調整
- [ ] Cloudflare R2 / Pages への接続とデプロイ設定
