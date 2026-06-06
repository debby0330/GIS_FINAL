# GIS_FINAL：花蓮災害風險與避難所適宜性分析專案

## 專案簡介

本專案以花蓮縣為研究區，針對颱風與災害事件下的避難收容需求進行空間分析。專案整合多源資料，包括避難所位置、學校位置、土石流潛勢溪流、DEM 地形資料、雨量測站資料、Sentinel-1 SAR、Sentinel-2 光學影像、村里高齡人口資料等，建立避難所與候選學校的災害風險評估流程。

主要目標包括：

1. 建立花蓮縣既有避難所的多因子風險評估。
2. 整合遙測影像分析淹水與崩塌潛勢。
3. 使用空間插值與機器學習推估雨量風險。
4. 評估學校作為新增避難據點的適宜性。
5. 建立互動式 WebGIS 地圖，呈現避難所、學校、災害風險與人口脆弱度資訊。

---

## 專案架構

```text
GIS_FINAL/
│
├── data/
│   ├── dem_20m_hualien.tif
│   ├── slope_hualien.tif
│   ├── risk_grid_30m.gpkg
│   ├── school_selected.gpkg
│   ├── schools_gdf_with_risk_factors.gpkg
│   ├── debrisstream_hualien/
│   ├── ragasa_shp/
│   ├── 鄉鎮市區界線/
│   ├── 村里界線/
│   └── 其他原始與中介資料
│
├── output/
│   ├── schools_list.csv
│   ├── shelter_gdf_twd97.csv
│   ├── shelter_gdf_twd97_score.csv
│   ├── shelter_gdf_twd97_group.csv
│   ├── shelters_risk_v7.csv
│   ├── school_selected.csv
│   ├── schools_score.csv
│   ├── hualien_disaster_shelter_interactive_map.html
│   ├── four_interpolation_methods_comparison.png
│   └── ai_shelter_site_selection/
│
├── script/
│   ├── final.ipynb
│   ├── hw_phase4_ndvi_4.ipynb
│   ├── merge.ipynb
│   ├── plot.ipynb
│   ├── school_feature_collect.ipynb
│   └── shelter_analysis.ipynb
│
├── .gitignore
└── README.md
```

---

## 使用資料

本專案使用的主要資料如下：

| 資料類型 | 用途 |
|---|---|
| 避難收容處所資料 | 建立既有避難所點位，進行災害風險評估 |
| 學校點位資料 | 評估學校作為新增避難據點的可能性 |
| DEM 高程資料 | 計算坡度、高程與地形風險 |
| 土石流潛勢溪流資料 | 建立 500 m、1000 m、1500 m 緩衝區，評估土石流暴露程度 |
| 雨量測站資料 | 使用 Ordinary Kriging 與 Random Forest 推估避難所與學校附近雨量 |
| Sentinel-2 光學影像 | 計算 NDVI、NDWI、BSI，偵測植被破壞與新增水體 |
| Sentinel-1 SAR 影像 | 偵測災後可能淹水區域 |
| 村里高齡人口資料 | 評估社會脆弱度與避難需求 |
| 鄉鎮市區、村里界線 | 進行行政區統計與空間疊合分析 |

---

## 分析流程

### 1. 避難所資料整理與地形風險分析

對應檔案：

```text
script/final.ipynb
```

主要工作包括：

- 讀取花蓮縣避難所資料。
- 將避難所點位轉換為 TWD97 坐標系統。
- 讀取 DEM，計算花蓮縣坡度圖。
- 針對每個避難所建立 300 m 緩衝區。
- 計算避難所周邊平均坡度與最大高程。
- 依坡度條件給予坡度風險等級。
- 建立土石流潛勢溪流緩衝區。
- 使用雨量測站資料進行雨量空間推估。
- 匯出避難所初步分析結果。

主要輸出：

```text
data/slope_hualien.tif
data/shelters_with_analysis_results.csv
output/shelter_gdf_twd97.csv
```

---

### 2. 多源遙測融合災害分析

對應檔案：

```text
script/hw_phase4_ndvi_4.ipynb
```

此部分使用 Sentinel-2 與 Sentinel-1 進行災害偵測，建立 ARIA v7.0 多源遙測融合分析流程。

主要分析內容包括：

- 使用 Microsoft Planetary Computer 搜尋 Sentinel-2 L2A 影像。
- 設定災前與災後影像時間區間。
- 載入 Sentinel-2 多光譜波段。
- 使用 SCL 進行雲遮罩處理。
- 計算 NDVI、NDWI、BSI。
- 使用 NDVI 變化偵測植被破壞或崩塌潛勢。
- 使用 NDWI 變化偵測新增水體。
- 搜尋 Sentinel-1 SAR VV 影像。
- 使用 SAR 變化偵測淹水區域。
- 整合光學與 SAR 結果，建立融合災害分類。
- 以 DEM 坡度進行地形校正，降低山區誤判淹水的可能。
- 輸出 30 m 網格災害風險資料。

遙測災害判斷條件：

| 指標 | 判斷方式 | 代表意義 |
|---|---|---|
| ΔNDVI | ΔNDVI < -0.2 | 植被破壞或崩塌潛勢 |
| ΔNDWI | ΔNDWI > 0.2 | 新增水體或淹水可能 |
| SAR VV 變化 | 災前災後雷達後向散射差異 | 全天候淹水偵測 |
| 坡度校正 | 排除高坡度區域 | 降低山區假水體誤判 |

融合分類概念：

| fusion_class | 說明 |
|---|---|
| 0 | 無明顯災害偵測 |
| 1 | 光學與 SAR 皆偵測到，屬高信心災害區 |
| 2 | 僅光學影像偵測到 |
| 3 | 僅 SAR 影像偵測到 |
| 9 | SAR 無資料或未覆蓋 |

主要輸出：

```text
output/shelters_risk_v7.csv
data/risk_grid_30m.gpkg
```

---

### 3. 避難所風險分數計算

對應檔案：

```text
script/merge.ipynb
```

此部分將既有避難所資料與遙測災害風險資料進行合併，並計算每個避難所的綜合風險分數。

整合欄位包括：

- `rain_1hr`
- `fusion_class`
- `fusion_label`
- `flood_confidence`
- `debris_stream_level`
- `Avg_slope`
- `Max_elevation`
- `Slope_risk_level`
- `sigma_1hr`
- `A65UP_CNT`

避難所風險分數主要由以下構面組成：

| 風險構面 | 欄位 | 權重概念 |
|---|---|---|
| 雨量風險 | `rain_1hr` | 雨量越大風險越高 |
| 崩塌或淹水暴露 | `fusion_class` | 是否位於災害偵測區 |
| 土石流暴露 | `debris_stream_level` | 越接近潛勢溪流風險越高 |
| 高程條件 | `Max_elevation` | 地形條件造成避難與災害暴露差異 |
| 坡度條件 | `Slope_risk_level` | 坡度越高地形風險越高 |
| 插值不確定性 | `sigma_1hr` | 推估不確定性越高越需注意 |
| 高齡人口 | `A65UP_CNT` | 作為社會脆弱度加權因子 |

風險分數計算邏輯：

```text
hazard_score =
    0.30 × risk_rain
  + 0.20 × risk_collapse
  + 0.15 × risk_debris_stream
  + 0.15 × risk_max_elevation
  + 0.10 × risk_slope_level
  + 0.10 × risk_uncertainty

vulnerability_multiplier =
    0.70 + 0.30 × risk_elderly

risk_raw =
    hazard_score × vulnerability_multiplier
```

再將分數拉伸至 0–100，並分為四個風險等級：

| 分數範圍 | 風險等級 |
|---|---|
| 0–25 | Low |
| 25–50 | Medium |
| 50–75 | High |
| 75–100 | Very High |

主要輸出：

```text
output/shelter_gdf_twd97_with_risk.csv
output/shelter_gdf_twd97_score.csv
output/shelter_gdf_twd97_group.csv
```

---

### 4. 學校點位蒐集與候選避難據點分析

對應檔案：

```text
script/shelter_analysis.ipynb
script/school_feature_collect.ipynb
```

此部分先從 OpenStreetMap 擷取花蓮縣吉安鄉範圍內的學校資料，再針對每個學校計算災害風險因子。

主要工作包括：

- 擷取研究範圍內學校點位。
- 加入學校所在鄉鎮市區與村里資訊。
- 計算學校與土石流潛勢溪流緩衝區的關係。
- 計算學校周邊 300 m 平均坡度與最大高程。
- 使用 Ordinary Kriging 推估學校位置雨量。
- 使用 Random Forest 推估學校位置雨量。
- 加入村里 65 歲以上人口資料。
- 從 30 m 災害風險網格提取 `fusion_class` 與 `fusion_label`。
- 計算每個學校的綜合風險分數。
- 篩選 Low 與 Medium 風險學校作為候選據點。
- 串接 Gemini，輔助挑選 5 個較適合作為避難據點的學校。

學校風險分數使用與避難所類似的架構：

```text
school_risk_score =
    hazard_score × vulnerability_multiplier × 100
```

主要輸出：

```text
output/schools_list.csv
output/schools_gdf_with_risk_factors.csv
output/schools_gdf_with_risk_factors.gpkg
output/schools_score.csv
output/school_selected.csv
output/school_selected.gpkg
output/ai_shelter_site_selection/
```

---

### 5. WebGIS 互動式地圖建置

對應檔案：

```text
script/plot.ipynb
```

此部分使用 Folium 建立互動式地圖，整合本專案主要成果。

地圖圖層包括：

- 花蓮縣行政區邊界
- 高程圖
- 坡度圖
- 雨量 Ordinary Kriging 插值圖
- 村里 65 歲以上人口圖層
- 土石流潛勢溪流
- 土石流潛勢溪流 500 m、1000 m、1500 m 緩衝區
- 既有避難所點位
- 學校點位
- AI 推薦之候選避難學校
- 避難所風險等級與風險分數

主要輸出：

```text
output/hualien_disaster_shelter_interactive_map.html
```

此 HTML 檔可直接以瀏覽器開啟，檢視互動式 WebGIS 成果。

---

## Notebook 執行順序建議

建議依照以下順序執行：

```text
1. shelter_analysis.ipynb
   → 擷取學校點位，輸出 schools_list.csv

2. final.ipynb
   → 整理避難所資料、建立土石流緩衝區、計算坡度與雨量風險

3. hw_phase4_ndvi_4.ipynb
   → 進行 Sentinel-2 / Sentinel-1 多源遙測災害偵測

4. merge.ipynb
   → 合併避難所與遙測災害風險，計算避難所風險分數

5. school_feature_collect.ipynb
   → 計算學校風險因子與學校風險分數，挑選候選避難學校

6. plot.ipynb
   → 建立最終互動式 WebGIS 地圖
```

---

## 環境需求

建議使用 Python 3.10 以上版本。

主要套件包括：

```text
pandas
numpy
geopandas
shapely
fiona
pyproj
rasterio
rioxarray
xarray
matplotlib
folium
branca
scikit-learn
scipy
pykrige
osmnx
pystac-client
stackstac
planetary-computer
python-dotenv
google-genai
```

可使用以下方式安裝常用套件：

```bash
pip install pandas numpy geopandas shapely fiona pyproj rasterio rioxarray xarray matplotlib folium branca scikit-learn scipy pykrige osmnx pystac-client stackstac planetary-computer python-dotenv google-genai
```

---

## API Key 設定

若需執行 Gemini 輔助評估學校適宜性的部分，請在專案根目錄建立 `.env` 檔案，並填入 API Key：

```text
AI_API_KEY=your_api_key_here
```

注意：

```text
.env 不應上傳到 GitHub。
```

建議在 `.gitignore` 中加入：

```gitignore
.env
__pycache__/
.ipynb_checkpoints/
```

---

## 主要成果

本專案完成以下成果：

1. 建立花蓮縣既有避難所災害風險評估資料表。
2. 建立土石流潛勢溪流多尺度緩衝區。
3. 產製花蓮縣坡度圖與避難所周邊地形風險指標。
4. 以 Ordinary Kriging 與 Random Forest 推估避難所與學校位置雨量。
5. 使用 Sentinel-2 偵測植被破壞與新增水體。
6. 使用 Sentinel-1 SAR 輔助偵測淹水區域。
7. 建立 30 m 災害風險網格。
8. 計算避難所與學校的綜合風險分數。
9. 挑選較適合作為新增避難據點的候選學校。
10. 建立互動式 WebGIS 地圖，整合災害風險、人口脆弱度與避難據點資訊。

---

## 重要輸出檔案說明

| 檔案 | 說明 |
|---|---|
| `output/shelter_gdf_twd97.csv` | 避難所基礎分析結果 |
| `output/shelter_gdf_twd97_score.csv` | 加入風險分數後的避難所資料 |
| `output/shelter_gdf_twd97_group.csv` | 依鄉鎮市區統計避難所風險等級 |
| `output/shelters_risk_v7.csv` | 遙測融合災害風險與避難所疊合結果 |
| `data/risk_grid_30m.gpkg` | 30 m 災害風險網格 |
| `output/schools_list.csv` | 從 OSM 擷取之學校清單 |
| `output/schools_score.csv` | 學校風險分數結果 |
| `output/school_selected.csv` | 推薦作為候選避難據點的學校 |
| `output/hualien_disaster_shelter_interactive_map.html` | 最終互動式 WebGIS 地圖 |

---

## 使用方式

### 1. 下載或複製專案

```bash
git clone <repository-url>
cd GIS_FINAL
```

### 2. 安裝套件

```bash
pip install -r requirements.txt
```

若專案尚未建立 `requirements.txt`，可依照 README 中的環境需求手動安裝套件。

### 3. 放置資料

請確認 `data/` 資料夾內包含必要圖資與資料，例如：

```text
dem_20m_hualien.tif
risk_grid_30m.gpkg
鄉鎮市區界線/
村里界線/
ragasa_shp/
debrisstream_hualien/
```

### 4. 執行 Notebook

依照建議順序執行 `script/` 資料夾中的 notebook。

### 5. 開啟互動式地圖

執行完成後，開啟：

```text
output/hualien_disaster_shelter_interactive_map.html
```

即可查看互動式地圖成果。

---

## 注意事項

1. 部分遙測影像需透過 Microsoft Planetary Computer 下載或串接，因此執行時需要網路連線。
2. Sentinel-1 SAR 影像可能因軌道與覆蓋範圍限制，導致部分區域為無資料。
3. 遙測災害偵測結果屬於模型與指標推估結果，仍需搭配實地調查或官方災害資料驗證。
4. 風險分數為多因子加權結果，權重設定會影響最終排序，後續可依研究需求調整。
5. Gemini 推薦結果為輔助決策，不應取代專業防災規劃判斷。
6. 若要將成果上傳至 GitHub，建議大型空間資料使用 Git LFS 管理。

---

## 專案限制

本專案仍有以下限制：

- 避難所與學校資料的完整性會影響分析結果。
- 雨量推估受測站分布與地形影響。
- Sentinel-2 光學影像容易受到雲層影響。
- SAR 淹水偵測在山區、坡地或粗糙地表可能產生誤判。
- 高齡人口僅作為社會脆弱度之一，尚未納入其他弱勢族群或交通可及性資料。
- 風險分數權重目前為研究設計設定，後續可透過專家問卷或 AHP 方法進一步校正。

---

## 後續改進方向

未來可進一步擴充：

1. 納入道路中斷、橋梁位置與交通可及性分析。
2. 加入歷史災害點位作為模型驗證資料。
3. 使用更細緻的人口脆弱度資料，例如身障人口、獨居老人、低收入戶等。
4. 將避難所容量、設施條件與物資供應能力納入評估。
5. 建立自動化流程，將資料前處理、風險計算與地圖輸出整合為單一 pipeline。
6. 建立網頁版儀表板，支援互動查詢與即時災害決策輔助。

---

## 專案主題

```text
風雨夾擊：樺加沙颱風期間花蓮避難所的綜合風險評估
```

本專案透過 GIS、遙測、空間插值、機器學習與 WebGIS 技術，建立花蓮縣避難所與候選學校的災害風險分析流程，作為防災規劃與避難據點選址的輔助參考。
