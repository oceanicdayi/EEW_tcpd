# EEW_tcpd

## 地震早期預警處理模組 (Earthquake Early Warning Processing Module)

TCPD 是一個基於 **Earthworm** 地震監測系統框架的**地震早期預警 (EEW)** 處理模組。

### 主要功能

- 從共享記憶體環接收 P 波到達資訊
- 多測站觸發驗證與地震定位
- 規模計算 (Mpd/Mtc)
- 發送地震預警報告

### 檔案說明

| 檔案 | 說明 |
|------|------|
| `tcpd.c` | 主程式原始碼 |
| `tcpd.d` | 組態檔 |
| `locate.h` | 定位與規模計算相關宣告 |
| `dayi_time.h` | 時間處理函式宣告 |
| `time_ew.h` | Earthworm 時間函式宣告 |
| `makefile.nt` | Windows 編譯腳本 |
| `makefile.ux` | Unix/Linux 編譯腳本 |
| `SPECIFICATION.md` | 📋 技術規格書 |
| `CODE_REVIEW.md` | 🔍 程式碼審查報告 |

### 編譯方式

**Windows:**
```batch
nmake -f makefile.nt
```

**Unix/Linux:**
```bash
make -f makefile.ux
```

### 執行方式

```bash
tcpd tcpd.d
```

### 依賴

- Earthworm 地震監測系統框架

### 文件

- 詳細規格請參閱 [SPECIFICATION.md](SPECIFICATION.md)
- 程式碼審查報告請參閱 [CODE_REVIEW.md](CODE_REVIEW.md)

### License

MIT License
