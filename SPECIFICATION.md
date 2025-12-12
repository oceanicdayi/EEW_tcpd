# TCPD 地震早期預警系統模組 - 技術規格書

## 文件資訊

| 項目 | 內容 |
|------|------|
| **專案名稱** | TCPD (Taiwan Central Paleoseismic Data / 地震早期預警處理模組) |
| **版本** | 1.0 |
| **建立日期** | 2025-12-12 |
| **程式語言** | C |
| **執行平台** | Windows NT / Unix/Linux |
| **依賴框架** | Earthworm (地震監測系統框架) |

---

## 1. 系統概述

### 1.1 目的

TCPD 是一個基於 **Earthworm** 地震監測系統框架的**地震早期預警 (Earthquake Early Warning, EEW)** 處理模組。其主要功能為：

1. 從共享記憶體環 (Shared Memory Ring) 接收來自地震儀器的 P 波到達資訊
2. 基於多測站觸發進行地震定位 (Earthquake Location)
3. 計算地震規模 (Magnitude Estimation)
4. 發送地震預警報告

### 1.2 系統特點

- **即時處理**：持續監控共享記憶體，即時處理地震觸發事件
- **多站觸發驗證**：透過時間窗口與距離窗口過濾誤觸發
- **迭代定位演算法**：使用最小平方法進行震源定位
- **多種規模計算方法**：支援 Mpd (位移)、Mtc (週期) 等規模估算
- **心跳機制**：定期發送心跳訊息確保模組運作正常

---

## 2. 系統架構

### 2.1 架構圖

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   PICK_RING     │────>│      TCPD        │────>│    EEW_RING     │
│ (輸入共享記憶體) │     │  (處理程式核心)   │     │ (輸出共享記憶體) │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                        │                        │
        v                        v                        v
  TYPE_EEW 訊息           地震定位與規模計算        TYPE_EEW 預警訊息
  (P波到達資訊)                                   Type_EEW_record
```

### 2.2 模組依賴

| 模組 | 功能 |
|------|------|
| `transport.c` | 共享記憶體傳輸函式庫 |
| `kom.c` | 組態檔解析器 |
| `logit.c` | 日誌記錄函式庫 |
| `getutil.c` | Earthworm 系統參數查詢 |
| `time_ew.c` | 時間處理函式庫 |
| `chron3.c` | 時間格式轉換 |
| `lockfile.c` | 程序鎖定機制 |

---

## 3. 資料結構

### 3.1 PEEW 結構 (地震預警資料)

```c
typedef struct {
    int    flag;           // 狀態標誌: 0=空, 1=使用中-最新, 2=使用中-舊
    int    serial;         // 序列號
    char   stn_name[8];    // 測站名稱
    char   stn_Loc[8];     // 測站位置碼
    char   stn_Comp[8];    // 測站分量 (如 HHZ, HHN, HHE)
    char   stn_Net[8];     // 測站網路碼
    double latitude;       // 緯度
    double longitude;      // 經度
    double altitude;       // 海拔
    double P;              // P波到達時間 (epoch seconds)
    double Pa;             // P波加速度峰值
    double Pv;             // P波速度峰值
    double Pd[15];         // P波位移值 (多秒)
    double Tc;             // 主週期 (Tau-c)
    double dura;           // 持續時間
    double report_time;    // 報告時間
    int    npoints;        // 資料點數
    double perr;           // P波到達時間誤差
    double wei;            // 權重 (定位使用)
    int    weight;         // 拾取權重
    int    inst;           // 儀器類型 (1=HL, 2=HH, 3=HS)
    int    upd_sec;        // 更新秒數
    int    usd_sec;        // 使用秒數
    double P_S_time;       // P-S 波到時差
    int    pin;            // SCNL 識別碼
} PEEW;
```

### 3.2 HYP 結構 (震源參數)

```c
typedef struct {
    double xla0;      // 震央緯度
    double xlo0;      // 震央經度
    double depth0;    // 震源深度 (km)
    double time0;     // 發震時刻 (epoch seconds)
    int    Q;         // 定位品質指標
    double averr;     // 平均時間誤差
    double gap;       // 方位角間隙 (度)
    double avwei;     // 平均權重
} HYP;
```

### 3.3 MAG 結構 (規模參數)

```c
typedef struct {
    double xMpd;      // Mpd 規模 (位移法)
    double mtc;       // Mtc 規模 (週期法)
    double ALL_Mag;   // 綜合規模
    double Padj;      // Pa 調整係數
} MAG;
```

---

## 4. 組態參數

### 4.1 基本 Earthworm 設定

| 參數 | 型態 | 說明 | 範例 |
|------|------|------|------|
| `MyModuleId` | 字串 | 模組識別名稱 | `MOD_TCPD` |
| `RingName` | 字串 | 輸入共享記憶體環名稱 | `PICK_RING` |
| `RingName_out` | 字串 | 輸出共享記憶體環名稱 | `EEW_RING` |
| `LogFile` | 整數 | 日誌檔開關 (0=關, 1=開) | `0` |
| `HeartBeatInterval` | 整數 | 心跳間隔 (秒) | `15` |

### 4.2 規模過濾設定

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `MagMin` | 浮點數 | 最小報告規模 | `0.5` |
| `MagMax` | 浮點數 | 最大報告規模 | `10.0` |

### 4.3 觸發驗證參數

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `Trig_tm_win` | 浮點數 | 觸發時間窗口 (秒) | `15.0` |
| `Trig_dis_win` | 浮點數 | 觸發距離窗口 (公里) | `100.0` |
| `Active_parr_win` | 浮點數 | P波到達存活時間 (秒) | `80.0` |

### 4.4 拾取權重過濾

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `Ignore_weight_P` | 整數 | P波忽略權重閾值 | `2` |
| `Ignore_weight_S` | 整數 | S波忽略權重閾值 | `2` |

### 4.5 報告設定

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `Term_num` | 整數 | 最大報告次數 | `50` |
| `Show_Report` | 整數 | 顯示報告開關 (0=關, 1=開) | `1` |
| `ReportLimitNumber` | 整數 | 連續報告限制 | `1` |

### 4.6 P波速度模型

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `Boundary_P` | 浮點數 | 淺深層邊界深度 (km) | `40.0` |
| `SwP_V` | 浮點數 | 淺層初始速度 (km/s) | `5.10298` |
| `SwP_VG` | 浮點數 | 淺層速度梯度 | `0.06659` |
| `DpP_V` | 浮點數 | 深層初始速度 (km/s) | `7.80479` |
| `DpP_VG` | 浮點數 | 深層速度梯度 | `0.00457` |

### 4.7 S波速度模型

| 參數 | 型態 | 說明 | 預設值 |
|------|------|------|--------|
| `Boundary_S` | 浮點數 | 淺深層邊界深度 (km) | `50.0` |
| `SwS_V` | 浮點數 | 淺層初始速度 (km/s) | `2.9105` |
| `SwS_VG` | 浮點數 | 淺層速度梯度 | `0.0365` |
| `DpS_V` | 浮點數 | 深層初始速度 (km/s) | `4.5374` |
| `DpS_VG` | 浮點數 | 深層速度梯度 | `0.0023` |

---

## 5. 功能模組

### 5.1 主處理流程 (`main`)

```
程式啟動
    │
    ├── 讀取組態檔 (tcpd_config)
    ├── 查詢系統參數 (tcpd_lookup)
    ├── 連接輸入/輸出共享記憶體環
    │
    └── 主迴圈
         │
         ├── 發送心跳訊息 (每 HeartBeatInterval 秒)
         │
         ├── 接收 TYPE_EEW 訊息
         │   ├── 解析訊息內容
         │   ├── 過濾無效拾取 (權重、分量)
         │   └── 儲存至觸發陣列
         │
         ├── 過濾過期觸發 (超過 Active_parr_win)
         │
         ├── 多站觸發驗證
         │   ├── 計算平均位置與時間
         │   ├── 檢查距離與時間窗口
         │   └── 保留符合條件的觸發
         │
         └── 若觸發數 > 4
             └── processTrigger() 進行定位與規模計算
```

### 5.2 地震定位 (`locaeq`)

**演算法**: 最小平方迭代法 (Least Squares Iteration)

**步驟**:

1. **初始位置估計**: 使用最早 P 波到達測站的位置
2. **迭代定位** (10 次):
   - 計算各測站理論到時
   - 計算到時殘差
   - 使用加權最小平方法調整震源位置
   - 根據距離與殘差計算權重
3. **深度網格搜尋**: 在 10-100 km 範圍內搜尋最佳深度
4. **計算方位角間隙 (GAP)**

**速度模型**: 使用雙層梯度速度模型

```
P波走時 = (-1/vg) * log|tan(θ2/2) / tan(θ1/2)|

其中:
- v = v0 + vg * z (深度梯度速度)
- θ1, θ2 為射線參數
```

### 5.3 規模計算 (`Magnitude`)

**支援的規模類型**:

| 規模 | 公式 | 說明 |
|------|------|------|
| **Mpd (HH)** | `5.000 + 1.102*log10(Pd) + 1.737*log10(R)` | 高增益寬頻位移規模 |
| **Mpd (HL)** | `5.067 + 1.281*log10(Pd) + 1.760*log10(R)` | 低增益寬頻位移規模 |
| **Mpd (HS)** | `4.811 + 1.089*log10(Pd) + 1.738*log10(R)` | 短週期位移規模 |
| **Mtc** | `4.218*log10(Tc) + 6.166` | 主週期規模 |

**資料品質控制**:
- 使用 Z 分數過濾離群值 (|Z| < 1.0)
- 加權平均計算最終規模
- 避免 S 波污染 Pd 值 (使用 P_S_time 判斷)

### 5.4 報告產生 (`Report_seq`)

**報告檔案格式**: `YYYYMMDDHHMMSS_nNN.rep`

**報告內容**:
- 報告時間、定位品質參數
- 震央座標、深度、規模
- 各測站詳細資料 (位置、Pa、Pv、Pd、Tc、殘差等)

**輸出訊息格式**:
```
num_eew t_now time0 count Mpd mag lat lon dep n n_c n_m averr avwei Q gap pro_time Mark Padj
```

---

## 6. 輸入/輸出規格

### 6.1 輸入訊息格式 (TYPE_EEW)

```
STA CMP NET LOC LON LAT PA PV PD TC P_TIME WEIGHT INST UPD_SEC
```

| 欄位 | 說明 | 範例 |
|------|------|------|
| STA | 測站代碼 | `YULB` |
| CMP | 分量代碼 | `HHZ` |
| NET | 網路代碼 | `TW` |
| LOC | 位置代碼 | `01` |
| LON | 經度 | `121.297100` |
| LAT | 緯度 | `23.392400` |
| PA | P波加速度峰值 (m/s²) | `0.006514` |
| PV | P波速度峰值 (m/s) | `0.000187` |
| PD | P波位移峰值 (m) | `0.000074` |
| TC | 主週期 Tau-c (s) | `4.297026` |
| P_TIME | P波到達時間 (epoch) | `1328085502.95800` |
| WEIGHT | 拾取權重 | `0` |
| INST | 儀器類型 | `2` |
| UPD_SEC | 更新秒數 | `3` |

### 6.2 輸出訊息格式 (TYPE_EEW)

```
num_eew t_now time0 count Mpd mag lat lon dep n n_c n_m averr avwei Q gap pro_time Mark Padj
```

| 欄位 | 說明 |
|------|------|
| num_eew | 地震事件編號 |
| t_now | 當前時間 (epoch) |
| time0 | 發震時刻 (epoch) |
| count | 報告序號 |
| Mpd | 規模 (Mpd) |
| lat | 震央緯度 |
| lon | 震央經度 |
| dep | 震源深度 (km) |
| n | 總觸發站數 |
| n_c | 非共站測站數 |
| n_m | 規模計算站數 |
| averr | 平均時間誤差 |
| avwei | 平均權重 |
| Q | 定位品質指標 |
| gap | 方位角間隙 (度) |
| pro_time | 處理時間 (秒) |
| Mark | 標記字串 |
| Padj | Pa 調整係數 |

---

## 7. 編譯與安裝

### 7.1 Windows (makefile.nt)

```batch
# 設定 Earthworm 環境變數
set EW_HOME=C:\Earthworm
set EW_VERSION=v7.10

# 編譯
nmake -f makefile.nt
```

### 7.2 Unix/Linux (makefile.ux)

```bash
# 設定 Earthworm 環境變數
export EW_HOME=/home/ew
export EW_VERSION=v7.10

# 編譯
make -f makefile.ux
```

### 7.3 依賴函式庫

- `$L/kom.obj` - 組態檔解析
- `$L/getutil.obj` - 系統參數查詢
- `$L/time_ew.obj` - 時間處理
- `$L/chron3.obj` - 時間格式轉換
- `$L/logit.obj` - 日誌記錄
- `$L/transport.obj` - 共享記憶體傳輸
- `$L/sleep_ew.obj` - 休眠函式
- `$L/lockfile.obj` / `$L/lockfile_ew.obj` - 程序鎖定

---

## 8. 執行方式

```bash
tcpd tcpd.d
```

**必要條件**:
1. Earthworm 系統必須已啟動
2. `PICK_RING` 與 `EEW_RING` 必須已建立
3. `earthworm.d` 中必須定義 `TYPE_EEW` 與 `Type_EEW_record` 訊息類型

---

## 9. 錯誤處理

### 9.1 錯誤代碼

| 代碼 | 名稱 | 說明 |
|------|------|------|
| 0 | ERR_MISSMSG | 在傳輸環中遺失訊息 |
| 1 | ERR_TOOBIG | 接收訊息超過緩衝區大小 |
| 2 | ERR_NOTRACK | 接收訊息但追蹤限制已超過 |

### 9.2 常見問題

| 問題 | 原因 | 解決方案 |
|------|------|----------|
| `TYPE_EEW not defined` | earthworm.d 未定義訊息類型 | 在 earthworm.d 加入 TYPE_EEW 定義 |
| `Invalid ring name` | 共享記憶體環不存在 | 確認 Earthworm startstop 已啟動該 ring |
| `Cannot get pid` | 系統呼叫失敗 | 檢查系統資源與權限 |

---

## 10. 版本歷史

| 版本 | 日期 | 變更說明 |
|------|------|----------|
| 1.0 | 2025-12-12 | 初始規格書建立 |

---

## 11. 參考文獻

1. Earthworm Programmer's Guide
2. 吳逸民教授 地震早期預警演算法
3. Taiwan Strong Motion Instrumentation Program (TSMIP)

---

## 12. 附錄

### 附錄 A: 組態檔範例 (tcpd.d)

```
# This is template's parameter file

#  Basic Earthworm setup:
MyModuleId         MOD_TCPD
RingName           PICK_RING
RingName_out       EEW_RING

LogFile            0
HeartBeatInterval  15

MagMin 0.5
MagMax 10

Ignore_weight_P    2
Ignore_weight_S    2

Trig_tm_win       15.0
Trig_dis_win      100.0
Active_parr_win   80.0

Term_num     50
Show_Report   1

# P-wave velocity model
Boundary_P     40.0
SwP_V        5.10298
SwP_VG       0.06659
DpP_V        7.80479
DpP_VG       0.00457

# S-wave velocity model
Boundary_S     50.0
SwS_V        2.9105
SwS_VG       0.0365
DpS_V        4.5374
DpS_VG       0.0023

# Message source
GetEventsFrom  INST_WILDCARD    MOD_WILDCARD    TYPE_EEW
```

### 附錄 B: 檔案清單

| 檔案 | 說明 |
|------|------|
| `tcpd.c` | 主程式原始碼 (3009 行) |
| `tcpd.d` | 組態檔 |
| `locate.h` | 定位與規模計算相關結構與函式宣告 |
| `dayi_time.h` | 時間處理函式宣告 |
| `time_ew.h` | Earthworm 時間函式宣告 |
| `makefile.nt` | Windows 編譯腳本 |
| `makefile.ux` | Unix/Linux 編譯腳本 |
