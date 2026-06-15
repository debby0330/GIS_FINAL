# GIS_FINAL：花蓮災害風險與避難所適宜性分析專案

## 專案主題

```text
風雨夾擊：樺加沙颱風期間花蓮避難所的綜合風險評估
```

本專案以花蓮縣為研究區，針對 2025 年 9 月 21–23 日樺加沙颱風期間可能造成的淹水、崩塌、土石流與避難收容風險進行綜合空間分析。專案整合 GIS、遙測、空間插值、機器學習與 WebGIS 技術，建立既有避難所與候選學校的災害風險評估流程，作為防災規劃、避難據點檢核與新增避難所選址的輔助參考。

---

## 專案簡介

本專案整合多源資料，包括既有避難收容處所、學校位置、DEM 地形資料、坡度資料、土石流潛勢溪流、雨量測站資料、Sentinel-2 光學影像、Sentinel-1 SAR 雷達影像、村里高齡人口資料、鄉鎮市區界線與村里界線，建立花蓮縣避難據點災害風險分析流程。

分析重點包括：

1. 建立花蓮縣既有避難所的多因子風險評估。
2. 以 Sentinel-2 與 Sentinel-1 進行多源遙測融合，判釋可能淹水、崩塌與地表變化訊號。
3. 使用 Ordinary Kriging 與 Random Forest 輔助推估雨量空間分布與不確定性。
4. 將土石流潛勢、地形條件、坡度條件、淹水 / 崩塌暴露與高齡人口脆弱度納入避難所風險計算。
5. 評估學校作為新增避難據點的可能性，並使用 Gemini 輔助挑選候選學校。
6. 建立互動式 WebGIS 地圖，整合避難所、學校、遙測災害結果、雨量、地形與人口脆弱度資訊。
7. 依據課堂建議修正風險公式權重，降低單一雨量因子對結果的支配性。

---

## 本次版本修正說明

本次版本主要依據老師對「原始風險分數權重可能過度偏向雨量」的建議進行修正。原先 `hazard_score` 中雨量因子權重較高，導致最終風險分布可能與雨量 Ordinary Kriging 內插結果過度相似，使避難所風險評估容易被單一短延時雨量因子主導。為改善此問題，本次重新調整 Hazard score 權重，使風險分數更強調淹水與崩塌暴露，同時保留土石流潛勢、地形高程、坡度條件與資料不確定性。

### 修正摘要

| 修正類型 | 修正前問題 | 本次修正內容 | 修正目的 |
|---|---|---|---|
| Hazard score 權重調整 | 雨量權重偏高，結果容易與雨量內插圖相似 | 將 `risk_rain` 權重調整為 0.15，將 `risk_collapse` 權重提高為 0.30 | 避免分析結果受單一雨量因子主導，強化淹水與崩塌暴露資訊 |
| 遙測融合結果角色強化 | 原始公式較偏重降雨觸發條件 | 提高由 `fusion_class` 轉換而來的 `risk_collapse` 權重 | 讓 Sentinel-1 / Sentinel-2 融合判釋結果更直接影響避難所風險 |
| 地形與土石流條件維持中等權重 | 避難風險不應只看降雨與遙測暴露 | `risk_debris_stream`、`risk_max_elevation`、`risk_slope_level` 各維持 0.15 | 同時反映土石流潛勢、地形高程與坡度條件 |
| 不確定性納入 | 雨量推估與遙測資料可能有誤差 | `risk_uncertainty` 維持 0.10 | 將資料品質與插值不確定性納入保守風險判斷 |
| 社會脆弱度架構維持 | 高齡人口不是災害發生原因，但會提高避難需求 | 維持 `vulnerability_multiplier = 0.70 + 0.30 × risk_elderly` | 將高齡人口作為風險後果與避難需求的放大因子 |
| 結果合理性檢查 | 權重改變後需確認風險空間分布是否仍合理 | 重新檢視鄉鎮市區風險比例 | 調整後卓溪鄉與玉里鎮高風險避難所比例仍較高，顯示結果並非單純由雨量造成 |

### 更新後風險公式

```python
df["hazard_score"] = (
    0.15 * df["risk_rain"] +
    0.30 * df["risk_collapse"] +
    0.15 * df["risk_debris_stream"] +
    0.15 * df["risk_max_elevation"] +
    0.15 * df["risk_slope_level"] +
    0.10 * df["risk_uncertainty"]
).clip(0, 1)

# ------------------------------------------------------------
# 7. 社會脆弱度放大係數
# ------------------------------------------------------------

df["vulnerability_multiplier"] = (
    0.70 + 0.30 * df["risk_elderly"]
)
```

### 權重調整理由

本次修正的核心考量是使風險分數更接近「多因子災害暴露與避難脆弱度」的綜合判斷，而不是單純再現雨量內插結果。雨量仍然是颱風與短延時強降雨災害的重要觸發條件，因此仍保留於 Hazard score 中；然而，若雨量權重過高，則風險分布會高度接近 `rain_1hr` 的空間插值圖，導致土石流潛勢、崩塌或淹水暴露、地形與坡度條件的影響被削弱。

因此，本次將 `risk_rain` 權重調整為 0.15，降低其對總分的主導性；同時將 `risk_collapse` 權重提高至 0.30，使 Sentinel-1 SAR 與 Sentinel-2 光學影像融合後得到的 `fusion_class` 在風險評估中扮演更關鍵角色。此設計代表本研究更重視「是否已經呈現淹水或崩塌暴露訊號」，而不只是「雨量是否較高」。

`risk_debris_stream`、`risk_max_elevation` 與 `risk_slope_level` 各設定為 0.15，代表土石流潛勢溪流、地形高程與坡度條件皆為避難所安全評估的重要背景因素。這些因素雖不一定單獨造成災害，但會影響避難所周邊的暴露程度、疏散可及性與災後救援難度。`risk_uncertainty` 則維持 0.10，用於反映雨量推估或資料品質的不確定性，避免在資訊不穩定區域給出過度明確的低風險判斷。

### 調整後結果說明

經過權重調整後，花蓮縣卓溪鄉與玉里鎮之高風險避難所比例仍然相對較高。此結果表示這些地區的風險並非單純由雨量內插造成，而是由多項因素共同形成，包括淹水或崩塌暴露訊號、土石流潛勢、地形高程、坡度條件、雨量不確定性與高齡人口脆弱度。因此，卓溪鄉與玉里鎮仍應被視為後續避難所檢核與防災資源配置的優先關注區域。

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
│   └── ai_shelter_site_selection_new/
│
├── script/
│   ├── final.ipynb
│   ├── hw_phase4_ndvi_5.ipynb
│   ├── merge_new.ipynb
│   ├── plot_new.ipynb
│   ├── school_feature_collect_new.ipynb
│   └── shelter_analysis_new.ipynb
│
├── .env
├── .gitignore
├── .gitattributes
└── README.md
```

---

## 使用資料

| 資料類型 | 用途 |
|---|---|
| 避難收容處所資料 | 建立既有避難所點位，進行災害風險評估 |
| 學校點位資料 | 評估學校作為新增避難據點的可能性 |
| DEM 高程資料 | 計算坡度、高程與地形風險 |
| 坡度圖資 | 判斷避難所與學校周邊坡地風險 |
| 土石流潛勢溪流資料 | 建立 500 m、1000 m、1500 m 緩衝區，評估土石流暴露程度 |
| 雨量測站資料 | 使用 Ordinary Kriging 與 Random Forest 推估避難所與學校附近雨量 |
| Sentinel-2 光學影像 | 計算 NDVI、NDWI、BSI，偵測植被破壞與新增水體 |
| Sentinel-1 SAR 影像 | 偵測災後可能淹水區域，補足光學影像受雲層影響的限制 |
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
- 讀取 DEM 並建立花蓮縣坡度圖。
- 針對每個避難所建立 300 m 緩衝區。
- 計算避難所周邊平均坡度與最大高程。
- 依坡度條件給予坡度風險等級。
- 建立土石流潛勢溪流 500 m、1000 m、1500 m 緩衝區。
- 使用雨量測站資料進行雨量空間推估。
- 匯出避難所初步分析結果。

主要輸出：

```text
data/slope_hualien.tif
output/shelter_gdf_twd97.csv
```

---

### 2. ARIA v7.0 多源遙測融合災害分析

對應檔案：

```text
script/hw_phase4_ndvi_5.ipynb
```

此部分使用 Sentinel-2 光學影像與 Sentinel-1 SAR 雷達影像進行多源遙測融合，針對樺加沙颱風期間花蓮縣可能淹水、崩塌與地表變化進行判釋。此流程參考 ARIA v7.0 的分析概念，將光學影像、SAR 影像與地形校正結果整合為 30 m 災害風險網格。

#### 分析架構

| Phase | 說明 |
|---|---|
| 4A | Sentinel-2 STAC 搜尋與 TCI 預覽 |
| 4B | 多波段載入，包括 B02、B03、B04、B08、B11、B12，並使用 SCL 進行雲遮罩 |
| 4C | NDVI、NDWI、BSI 指標計算與災前災後變化偵測 |
| 4D | Sentinel-1 SAR 水體偵測，補足光學影像受雲層影響的限制 |
| 4E | 光學與 SAR 多源融合，建立淹水信心度分類 |
| 4F | DEM 坡度校正，降低高坡度區域被誤判為水體的可能 |
| 4G | 崩塌、水體遮罩與避難所進行空間疊合 |
| 4H | 輸出最終視覺化圖層、避難所淹水信心度與 30 m 風險網格 |

#### 遙測指標說明

| 指標 | 公式 | 災害意義 |
|---|---|---|
| NDVI | `(NIR - Red) / (NIR + Red)` | 反映植被狀態；災後 NDVI 明顯下降可作為崩塌或植被破壞潛勢 |
| NDWI | `(Green - NIR) / (Green + NIR)` | 反映水體分布；災後 NDWI 上升可作為新增水體或淹水訊號 |
| BSI | `((Red + SWIR1) - (NIR + Blue)) / ((Red + SWIR1) + (NIR + Blue))` | 反映裸露土壤或地表擾動，可輔助判斷崩塌或沖刷區域 |

#### 主要門檻參數

| 參數 | 數值 | 說明 |
|---|---:|---|
| `DNDVI_THRESH` | -0.2 | ΔNDVI 低於此值判定為植被破壞或崩塌潛勢 |
| `DNDWI_THRESH` | 0.2 | ΔNDWI 高於此值判定為新增水體或淹水可能 |
| `SAR_FLOOD_DB` | -18 dB | SAR VV 低於此值為一般水體 |
| `SAR_TURB_DB` | -14 dB | SAR VV 低於此值為濁水或高含沙水體 |
| `SAR_DELTA_THRESH` | -3 dB | SAR 差值低於此值判定為新增淹水 |
| `SLOPE_MAX_DEG` | 25° | 坡度超過此值排除假水體，降低山區誤判 |

#### 融合分類定義

| fusion_class | fusion_label | flood_confidence | 條件 | 意義 |
|---:|---|---|---|---|
| 1 | 兩者皆有淹水 | high | 光學水體與 SAR 水體皆成立 | 雙重證據確認淹水 |
| 2 | 僅光學偵測 | mid | 光學水體成立，但 SAR 非水體或 SAR 無資料 | 光學單源偵測，仍納入分析 |
| 3 | 僅 SAR 偵測 | mid | 光學無水體，但 SAR 偵測水體 | SAR 補充光學盲區 |
| 0 | 無淹水 | low | 無水體訊號且 SAR 有資料 | 無明顯淹水偵測 |
| 9 | 無淹水（SAR 無資料） | low | SAR 未覆蓋且光學亦無水體 | 資訊不足，排除淹水判斷 |
| -1 | 無淹水（範圍外） | low | 位於分析範圍外或未納入判釋 | 不作為淹水暴露區 |

主要輸出：

```text
output/tci_candidates_pre.png
output/tci_candidates_post.png
output/landslide_mask.png
output/optical_water_mask.png
output/optical_change_panel.png
output/sar_flood_detection.png
output/fusion_flood_shelter_map.png
output/topographic_correction.png
output/aria_v7_final_map.png
output/shelters_flood_confidence.csv
output/shelters_risk_v7.csv
data/risk_grid_30m.gpkg
```

---

### 3. 避難所風險分數計算

對應檔案：

```text
script/merge_new.ipynb
```

此部分將既有避難所資料、地形資料、土石流資料、雨量推估結果、遙測災害融合結果與村里高齡人口資料進行合併，並計算每個避難所的綜合風險分數。

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
- `village_A65UP_CNT`

#### 風險因子轉換方式

| 風險因子 | 來源欄位 | 轉換方式 | 風險意義 |
|---|---|---|---|
| `risk_rain` | `rain_1hr` | Min-Max 標準化 | 雨量越大，短延時降雨風險越高 |
| `risk_collapse` | `fusion_class` | 依融合分類轉換 | 反映淹水或崩塌暴露程度 |
| `risk_debris_stream` | `debris_stream_level` | Safe / Low / Medium / High 轉為 0–1 | 土石流潛勢越高，風險越高 |
| `risk_max_elevation` | `Max_elevation` | Min-Max 標準化 | 高程越高，地形與避難可及性風險越高 |
| `risk_slope_level` | `Slope_risk_level` | Low / Medium / High 轉為 0–1 | 坡度風險越高，地形不穩定性越高 |
| `risk_uncertainty` | `sigma_1hr` | Min-Max 標準化 | 雨量推估不確定性越高，決策風險越高 |
| `risk_elderly` | `village_A65UP_CNT` | `log1p` 後 Min-Max 標準化 | 高齡人口越多，避難需求與社會脆弱度越高 |

#### 更新後風險分數計算邏輯

```text
hazard_score =
    0.15 × risk_rain
  + 0.30 × risk_collapse
  + 0.15 × risk_debris_stream
  + 0.15 × risk_max_elevation
  + 0.15 × risk_slope_level
  + 0.10 × risk_uncertainty

vulnerability_multiplier =
    0.70 + 0.30 × risk_elderly

risk_raw =
    hazard_score × vulnerability_multiplier
```

其中，`risk_raw` 進一步使用第 5 與第 95 百分位數進行 robust rescale，轉換為 0–100 的相對風險分數：

```text
shelter_risk_score = robust_rescale_0_100(risk_raw)
```

最後依分數分為四個風險等級：

| 分數範圍 | 風險等級 |
|---:|---|
| 0–25 | Low |
| 25–50 | Medium |
| 50–75 | High |
| 75–100 | Very High |

主要輸出：

```text
output/shelter_gdf_twd97_score.csv
output/shelter_gdf_twd97_group.csv
```

---

### 4. 學校點位蒐集與候選避難據點分析

對應檔案：

```text
script/shelter_analysis_new.ipynb
script/school_feature_collect_new.ipynb
```

此部分先從 OpenStreetMap 擷取研究範圍內的學校資料，再針對每個學校計算災害風險因子，評估學校是否適合作為候選避難據點。

主要工作包括：

- 擷取研究範圍內學校點位。
- 加入學校所在鄉鎮市區與村里資訊。
- 計算學校與土石流潛勢溪流緩衝區的關係。
- 計算學校周邊 300 m 平均坡度與最大高程。
- 使用 Ordinary Kriging 推估學校位置雨量。
- 使用 Random Forest 推估學校位置雨量。
- 加入村里 65 歲以上人口資料。
- 從 30 m 災害風險網格提取 `fusion_class` 與 `fusion_label`。
- 使用與避難所一致的更新權重公式計算學校綜合風險分數。
- 篩選 Low 與 Medium 風險學校作為候選據點。
- 串接 Gemini，輔助挑選 5 個較適合作為避難據點的學校。

學校風險分數計算邏輯：

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
output/ai_shelter_site_selection_new/
```

---

### 5. AI 輔助避難學校選址

Gemini 模型僅使用程式提供的 `schools_gdf` 欄位進行判斷，不額外假設學校實際設施條件。

輸入 Gemini 的主要欄位包括：

- `school_name`
- `school_risk_level`
- `longitude`
- `latitude`
- `Max_elevation`
- `Slope_risk_level`
- `rain_1hr`
- `sigma_1hr`
- `village_A65UP_CNT`
- `fusion_class`
- `fusion_label`
- `debris_stream_level`

AI 評估原則包括：

1. 優先選擇 Low 或 Medium 風險學校。
2. 同時考量 5 個選址的空間分散性，避免過度集中。
3. 不自行假設學校實際已具備操場、體育館、無障礙設施、飲水設備、廁所容量或備援電力。
4. 所有設施建議均以「建議設置」或「後續需確認」方式描述。
5. 回傳表格需包含排名、學校名稱、school_risk_level、longitude、latitude、適合作為理由、分布意義、弱點與建議設施。

主要輸出：

```text
output/ai_shelter_site_selection_new/gemini_shelter_site_selection_response.md
output/school_selected.csv
output/school_selected.gpkg
```

---

### 6. WebGIS 互動式地圖建置

對應檔案：

```text
script/plot_new.ipynb
```

此部分使用 Folium 建立互動式地圖，整合本專案主要成果。地圖以 OpenStreetMap 作為底圖，並提供圖層控制功能，使使用者可依需求單獨開啟或關閉不同資料圖層。

地圖圖層包括：

- OpenStreetMap 底圖
- 花蓮縣行政區邊界
- 高程圖
- 坡度圖
- 雨量 Ordinary Kriging 插值圖
- 村里 65 歲以上人口圖層
- 土石流潛勢溪流
- 土石流潛勢溪流 500 m、1000 m、1500 m 緩衝區
- 既有避難所點位，依 `shelter_risk_level` 顯示不同顏色
- 學校點位，依 `school_risk_level` 顯示不同顏色
- AI 推薦之候選避難學校，以紅色星號顯示
- 右下角圖例與圖層控制功能

主要輸出：

```text
output/hualien_disaster_shelter_interactive_map.html
```

此 HTML 檔可直接以瀏覽器開啟，檢視互動式 WebGIS 成果。

---

## 鄉鎮市區風險結果說明

依據調整後的分數架構，花蓮縣卓溪鄉與玉里鎮之高風險避難所比例仍然相對較高。此結果代表即使降低雨量權重，這兩個地區仍因淹水或崩塌暴露、地形高程、坡度條件、土石流潛勢與社會脆弱度等因素而呈現較高風險。

因此，卓溪鄉與玉里鎮應被視為後續避難所檢核與防災資源配置的優先關注區域。後續可進一步針對兩地區檢查：

- 避難所是否位於坡地或潛勢溪流影響範圍附近。
- 避難所周邊道路是否具有災時可及性。
- 是否需增設備援避難據點或臨時收容場所。
- 是否需針對高齡人口較多的村里加強無障礙設施與醫療支援。
- 是否需增加即時雨量監測、警戒標示與撤離預案。

---

## Notebook 執行順序建議

建議依照以下順序執行：

```text
1. shelter_analysis_new.ipynb
   → 擷取學校點位，輸出 schools_list.csv

2. final.ipynb
   → 整理避難所資料、建立土石流緩衝區、計算坡度與雨量風險

3. hw_phase4_ndvi_5.ipynb
   → 進行 Sentinel-2 / Sentinel-1 多源遙測災害偵測

4. merge_new.ipynb
   → 合併避難所與遙測災害風險，計算避難所風險分數

5. school_feature_collect_new.ipynb
   → 計算學校風險因子與學校風險分數，挑選候選避難學校

6. plot_new.ipynb
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

中文字體支援建議安裝 Noto CJK 或 Microsoft JhengHei，避免地圖與圖表中文標題顯示為方框。

---

## API Key 設定

若需執行 Gemini 輔助評估學校適宜性的部分，請在專案根目錄建立 `.env` 檔案，並填入 API Key：

```text
AI_API_KEY=your_api_key_here
```

`.env` 不應上傳到 GitHub。建議在 `.gitignore` 中加入：

```gitignore
.env
*.env
__pycache__/
.ipynb_checkpoints/
```

若使用 VS Code，可確認以下設定，使 Python Terminal 自動讀取 `.env`：

```json
{
  "python.terminal.useEnvFile": true
}
```

Notebook 中也可使用：

```python
from dotenv import load_dotenv
from pathlib import Path

load_dotenv(Path("../.env"), override=True)
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
7. 建立 ARIA v7.0 多源遙測融合分析流程。
8. 建立 30 m 災害風險網格。
9. 依更新後的多因子權重計算避難所與學校的綜合風險分數。
10. 挑選較適合作為新增避難據點的候選學校。
11. 建立互動式 WebGIS 地圖，整合災害風險、人口脆弱度與避難據點資訊。

---

## 重要輸出檔案說明

| 檔案 | 說明 |
|---|---|
| `output/shelter_gdf_twd97.csv` | 避難所基礎分析結果 |
| `output/shelter_gdf_twd97_score.csv` | 加入更新風險分數後的避難所資料 |
| `output/shelter_gdf_twd97_group.csv` | 依鄉鎮市區統計避難所風險等級 |
| `output/shelters_risk_v7.csv` | 遙測融合災害風險與避難所疊合結果 |
| `output/shelters_flood_confidence.csv` | 各避難所淹水信心度評估結果 |
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
slope_hualien.tif
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
3. Sentinel-2 光學影像容易受到雲層影響，因此需搭配 SCL 雲遮罩與 SAR 影像補充。
4. 遙測災害偵測結果屬於模型與指標推估結果，仍需搭配實地調查或官方災害資料驗證。
5. 風險分數為多因子加權結果，權重設定會影響最終排序；本次已依課堂建議降低雨量主導性並提高災害暴露因子權重。
6. Gemini 推薦結果為輔助決策，不應取代專業防災規劃判斷。
7. 若要將成果上傳至 GitHub，建議大型空間資料使用 Git LFS 管理；若資料量過大，也可改以雲端硬碟或 Zenodo 提供資料下載連結。

---

## 專案限制

本專案仍有以下限制：

- 避難所與學校資料的完整性會影響分析結果。
- 雨量推估受測站分布、地形與插值方法影響。
- Sentinel-2 光學影像容易受到雲層影響。
- SAR 淹水偵測在山區、坡地或粗糙地表可能產生誤判。
- 高齡人口僅作為社會脆弱度之一，尚未納入其他弱勢族群、避難容量與交通可及性資料。
- 風險分數權重目前為研究設計設定，後續可透過專家問卷、AHP 方法或歷史災害資料進一步校正。
- Gemini 選址結果依據輸入欄位生成，仍需現地調查確認學校實際避難空間、設施容量與管理條件。

---

## 後續改進方向

未來可進一步擴充：

1. 納入道路中斷、橋梁位置與交通可及性分析。
2. 加入歷史災害點位作為模型驗證資料。
3. 使用更細緻的人口脆弱度資料，例如身障人口、獨居老人、低收入戶等。
4. 將避難所容量、設施條件與物資供應能力納入評估。
5. 建立自動化流程，將資料前處理、風險計算與地圖輸出整合為單一 pipeline。
6. 建立網頁版儀表板，支援互動查詢與即時災害決策輔助。
7. 比較不同權重組合對鄉鎮市區風險比例與高風險據點分布的敏感度。

---

## 授權與資料使用提醒

本專案主要用於課程作業與研究展示。若需公開發布或延伸應用，應確認各項資料來源之授權條件，特別是官方圖資、遙測資料、避難收容處所資料與人口統計資料。若使用 Gemini API 或其他外部模型服務，需避免上傳個人敏感資料或未授權資料。
