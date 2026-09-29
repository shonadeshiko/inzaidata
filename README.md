# 印旛沼流域 GIS解析プラットフォーム（白井・印西 高解像度版）

千葉県印旛沼流域のうち白井市・印西市を対象に、空間演算・ラスタ解析・
ポテンシャル測定を行い、結果を静的サイトとして公開するためのリポジトリです。

姉妹リポジトリ [gisdata](https://github.com/shonadeshiko/gisdata)（印旛沼流域版）の
コード構成をコピーし、より高解像度なデータ（100mメッシュ等）を
取り込んだもの。

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
│   │   ├── ingest_inzai_data.py       # 白井・印西の高解像度実データ取り込み
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

新しいデータを追加したい場合は `pipeline/process/ingest_inzai_data.py` の
`RASTER_DEFS` / `VECTOR_DEFS` に定義を1つ追加するだけでよい。連続値のデータを
uint8で軽量化したい場合は `uint8_scale` を指定する（例: 10を指定すると
値を10倍してuint8(0-255)に丸めて保存し、frontend側で10で割り戻す）。

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
属性名を踏襲しているが、白井・印西の実メッシュデータ
(`メッシュ100m_GI統合_白井印西.gpkg`、100m解像度・22,299メッシュ)も
同じ属性名(`gi_conservation_score`/`gi_pressure_score`/`priority_rank`/
`urban_frac`/`forest_frac`等)で作られていることを確認済み。

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

## 白井・印西 実データについて

`pipeline/process/ingest_inzai_data.py` は、白井市・印西市の高解像度実データ
（水田占有率・自然的景観の多様度(千葉県全域)、HANDランク・開発圧
(2011-2022/2020-2024)・TWIランク・GI地形スコア(白井・印西域)のラスタ、
行政界+1kmバッファ・100mメッシュGI統合のベクタ）を取り込み、
EPSG:4326への再投影・COG化・GeoJSON化を行う。

元データは `data/raw/inzai/{raster,vector}/` に配置する想定（Git管理外）。

### データの注意点

- **メッシュGeoJSONのサイズ**：100m解像度・22,299メッシュのため
  約17MBある。本サイトは「ブラウザが全ファイルを丸ごとfetchする」設計
  のため、初回読み込みがやや重くなる。
- **雨水浸透機能（`inzai_infiltration_potential`）**：千葉県全域の
  地形分類ポリゴン(2,031件)で、`result`列（01最適地/02適地/03不適地/
  05判定不能/06判定対象外/07除外区域）による6区分のカテゴリカル
  データ。frontendでカテゴリ→色のマッピング(`INFILTRATION_CATEGORIES`)
  ・専用凡例・表示切り替えチェックボックスを実装済み(既定は非表示)。
  - 元のshp(`.dbf`)は文字コードが特殊で、通常の読み込みでは日本語が
    文字化けする。`VECTOR_DEFS`に`"fix_mojibake": True`を指定すると、
    文字列列をlatin1→utf-8で読み直して補正する
    (`process_vectors()`参照)。
  - 元データは地形分類由来で頂点数が非常に多く(2,031件で約50万頂点、
    未加工だと34MB近い)、`"simplify_tolerance": 0.0003`
    (約30m)でジオメトリを単純化し約8.5MBまで軽量化している。
- **未取り込みのデータ種類**：以下のデータはまだ
  `RASTER_DEFS`に未登録。ラスタのカテゴリカル値(土地被覆区分)を
  色分け表示するフロントエンド実装が必要。
  - `JAXA_HRLC土地被覆_2020_白井印西.tif` / `_2024_白井印西.tif`
    （カテゴリカル、1-15の土地被覆区分）

## 今後のTODO

- [ ] JAXA土地被覆データの取り込み（ラスタのカテゴリカル値色分けを
      新規実装。雨水浸透機能で実装したベクタ用の仕組みをラスタ用に
      応用する形になる見込み）
- [ ] メッシュのサイズ最適化（ジオメトリ簡略化 / タイル化）が必要か検討
- [ ] `ingest_gsi_elevation.py` の `ZOOM` を対象解像度に合わせて調整
- [ ] Cloudflare R2 / Pages への接続とデプロイ設定
