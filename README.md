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

GDALのシステムライブラリが必要（`rasterio`/`geopandas`が依存）。

```bash
# Ubuntu/Debian
sudo apt-get install -y gdal-bin libgdal-dev
# macOS (Homebrew)
brew install gdal
```

```bash
cd pipeline
python -m venv venv
source venv/bin/activate   # Windowsは venv\Scripts\activate
pip install -r requirements.txt
python process/ingest_inzai_data.py   # 元データが data/raw/inzai/ にある場合
```

## 新しいラスタ/ベクタデータを追加する手順（Claudeを介さずに行う場合）

Claudeに頼らずデータを追加・更新したい場合は、以下の手順をそのまま
たどればよい（このREADME自体がそのためのテンプレート）。

### 1. 元データを配置する

```
data/raw/inzai/raster/<ファイル名>.tif
data/raw/inzai/vector/<ファイル名>.shp (.gpkg等)
```

（このディレクトリは`.gitignore`対象なので、置くだけではGitに反映されない。
後述の通り、Gitに乗るのは`pipeline/process/ingest_inzai_data.py`実行後の
`data/processed/`配下の成果物のみ）

### 2. `pipeline/process/ingest_inzai_data.py` に定義を1つ追加する

`RASTER_DEFS`（ラスタの場合）または`VECTOR_DEFS`（ベクタの場合）の
リストの末尾に、以下のテンプレートをコピーして追記する。

**ラスタの場合：**

```python
{
    "id": "inzai_xxx",                 # 半角英数字・アンダースコアのみ。他と重複しないこと
    "src": "元ファイル名.tif",          # data/raw/inzai/raster/ 内のファイル名
    "name": "画面に表示する名前（日本語可）",
    "unit": "単位（例: 比率(0-1)、ランク(1-5) など）",
    "description": "画面の説明文に使われる1〜2文程度の説明",
    "resampling": Resampling.bilinear, # 連続値ならbilinear、カテゴリ/ランク値ならnearest
    # 連続値を軽量化したい場合のみ指定（省略可）。10なら値を10倍してuint8(0-255)で
    # 保存し、frontend側で10で割り戻す。小数点以下1桁を保ったまま軽量化できる
    "uint8_scale": 10,
},
```

ファイルが複数に分割されている場合（例: 都道府県ごとに分かれているなど）は、
`"src"` にリストを渡すとモザイク結合してから処理される
（`merge_rasters()` 参照。shimamotodataで京都府・大阪府にまたがる
データを結合するために追加した仕組みを流用）。

```python
"src": ["ファイルA.tif", "ファイルB.tif"],
```

**ベクタの場合：**

```python
{
    "id": "inzai_xxx",
    "src": "元ファイル名.shp",          # data/raw/inzai/vector/ 内のファイル名
    "name": "画面に表示する名前",
    "description": "説明文",
    # 以下はオプション（必要な場合のみ追加）
    "fix_mojibake": True,              # 属性の日本語が文字化けする場合に指定
    "simplify_tolerance": 0.0003,      # ジオメトリが重い場合に単純化する度数(約0.0003度=約30m)
},
```

カテゴリカルな値（例: 区分名など）を地図上で色分け表示したい場合は、
`web/index.html` 側にも凡例の仕組みを追加する必要がある
（`inzai_infiltration_potential`の実装、`INFILTRATION_CATEGORIES`が参考例）。
単に属性テーブルをポップアップ表示するだけなら、この手順2だけで完結する。

### 3. パイプラインを実行する

```bash
cd pipeline
source venv/bin/activate
python process/ingest_inzai_data.py
```

エラーなく完了したら、`data/processed/catalog.json`（ラスタ一覧）または
`data/processed/vectors_catalog.json`（ベクタ一覧）に新しいエントリが
追加されていること、対応するファイルが`data/processed/rasters/`または
`data/processed/vectors/`に生成されていることを確認する。

### 4. ローカルで表示確認する

```bash
mkdir -p web/data
cp -r data/processed/* web/data/
cd web
python -m http.server 8000
```

ブラウザで `http://localhost:8000` を開き、追加したデータが
パネルの選択肢に表示され、クリック/範囲選択で正しく集計されることを確認する。

### 5. GitHubに反映する（別環境で作業する場合）

このリポジトリを初めてcloneする環境、あるいはいつもと違うPC/CI環境で
作業する場合の、初回セットアップからpushまでの一連の流れ。

```bash
# 1. リポジトリをclone（初回のみ）
git clone https://github.com/shonadeshiko/inzaidata.git
cd inzaidata

# 2. 現在のmainの最新状態を取り込む（他の変更と衝突しないように）
git checkout main
git pull origin main

# 3. 元データを data/raw/inzai/{raster,vector}/ に配置し、
#    上記1〜4の手順で RASTER_DEFS/VECTOR_DEFS への追記・パイプライン実行・
#    ローカル確認まで行う

# 4. 変更内容を確認する（data/processed/ 以下の生成物と、
#    ingest_inzai_data.py の差分だけになっているはず）
git status
git diff pipeline/process/ingest_inzai_data.py

# 5. ステージ・コミット・push
git add pipeline/process/ingest_inzai_data.py data/processed/
git commit -m "データ追加: <追加したデータの説明>"
git push origin main
```

`git push`すると、`.github/workflows/pipeline.yml`が
`data/processed/**`の変更を検知して自動実行され、
`web/data/`へのコピー→GitHub Pagesへのデプロイまで自動で行われる
（進捗は https://github.com/shonadeshiko/inzaidata/actions で確認できる）。
数分後に https://shonadeshiko.github.io/inzaidata/ に反映される。

**認証について**：`git push`時にGitHubの認証が求められる場合は、
HTTPSならPersonal Access Token（パスワード欄にトークンを入力）、
SSHなら`git@github.com:shonadeshiko/inzaidata.git`形式のURLで
clone済みであることを確認する。初回pushで認証エラーが出た場合は、
GitHub上でリポジトリへの書き込み権限があるアカウントでログインできているか
確認すること。

**注意**：`data/raw/`配下は`.gitignore`対象のため、元データ自体は
Gitに乗らない（コミットされるのは`data/processed/`の生成物のみ）。
元データは別途バックアップ・共有しておくこと。

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

## ラスタの表示について

「② ラスタの表示」でドロップダウンから選んだラスタを、地図上に
色分け画像として重ねて表示できる。geotiff.jsで全ピクセルを読み込み、
nodataを除いた実際のmin/maxに合わせてカラーランプ(青→緑→黄→赤の
4段階、`RASTER_OVERLAY_RAMP`)で着色したcanvasを、MapLibreの
`image`ソースとして地図に貼り付ける方式（タイル化はしていない）。
透過度スライダーで調整可能。ベースマップ切り替え時もaddDataLayers()
から再適用される。

## 範囲分析について

「③ 範囲の分析」セクションで2つのモードを切り替えられる。

- **バッファモード**：地図をクリックすると、その地点を中心とした
  指定半径の円内で統計値を計算する。
- **ポリゴン描画モード**：地図上を順にクリックして頂点を打ち、
  「この範囲で計算」で確定すると、その多角形内で統計値を計算する
  （`calcStatsForAllRastersInPolygon()`がバッファ円・手描きポリゴン
  どちらでも使える共通処理になっている）。

いずれのモードも、カタログに登録されている全ラスタの平均・最小・
最大・集計セル数をブラウザ内（geotiff.js + turf.js）でオンデマンド
計算する。サーバー計算は不要。「平均」はセルの合計値(sum)を集計
セル数(count)で割った値で、比率として使いたい場合の「割合」と
同じ値になる。一度取得したラスタはブラウザ内にキャッシュされ、
2回目以降の分析で再取得しないようになっている。

計算に使った範囲(バッファ円・確定したポリゴン)は`analysis-area`
レイヤーとして地図上に薄い青で表示され、「分析範囲を消す」ボタンで
非表示にできる(結果テーブルの内容もあわせてクリアされる)。

## ベクタレイヤーの透過度について

100mメッシュ・雨水浸透機能の各ベクタレイヤーは、パネル内の
スライダーで表示中に透過度を調整できる（`fill-opacity`を
`map.setPaintProperty()`で動的に変更）。

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
