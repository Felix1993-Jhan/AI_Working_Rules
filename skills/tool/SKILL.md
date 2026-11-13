---
name: tool
description: 這台電腦(Windows 11,使用者 AQUIVIO)已安裝的開發工具清單 —— Flutter / Dart、Python 與套件(PDF、Excel、Word、numpy/scipy)、Node(nvm)、STM32CubeIDE / CubeProgrammer CLI、串口工具。要讀 PDF / Excel / Word、跑 Python 或 Node 腳本、燒錄或命令列操作 STM32、操作 git / GitHub(PR、Actions)、或打算安裝任何工具之前,先載入查有沒有現成的,不要重新查找或下載。
---

# 這台電腦的工具清單

> 查詢日期:**2026-11-03**。版本以實際查詢為準 —— 發現與下表不符(升級、移除、新裝),就更新這份,並改上面的日期。
> 重新盤點的指令在最下面。

## Flutter / Dart

| 工具 | 版本 | 位置 / 備註 |
|---|---|---|
| Flutter SDK | 3.38.5 stable | `C:\flutter\flutter` |
| Dart | 3.10.4 | 隨 Flutter |
| Visual Studio Community 2026 | 18.1 | Windows 桌面建置用(cmake 用它內建的,PATH 沒有 cmake 是正常的) |
| Android Studio + SDK | SDK 36.1 | `flutter doctor` 只警告 Android licenses 未接受;目前專案都是桌面版,不影響 |

## MCU(STM32)

| 工具 | 版本 | 位置 / 備註 |
|---|---|---|
| STM32CubeIDE | 1.19.0 | `C:\ST\STM32CubeIDE_1.19.0`,ARM 編譯器在 IDE 內(PATH 沒有 `arm-none-eabi-gcc` 是正常的),見下方 |
| arm-none-eabi-gcc | 13.3.1 | `/c/ST/STM32CubeIDE_1.19.0/STM32CubeIDE/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.13.3.rel1.win32_1.0.100.202509120712/tools/bin/`(建置 2025-05-23)。⚠️ 同目錄還有一份舊的 `…win32_1.0.0.202411081344`(建置 2024-09-26),IDE 升級沒清掉;用萬用字元找會抓到兩份,用上面這份。IDE 專案實際設定用哪份未查證 |
| STM32CubeProgrammer | 2.20.0 | CLI **不在 PATH**:`"/c/Program Files/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin/STM32_Programmer_CLI.exe"` |
| stlink_server | — | `C:\Program Files (x86)\STMicroelectronics\stlink_server` |
| PuTTY | — | `C:\Program Files\PuTTY`(注意:COM 埠獨占,App / server 握著時開不起來) |

## 通用

| 工具 | 版本 | 位置 / 備註 |
|---|---|---|
| Git + Git Bash | 2.52 | Claude 的 shell 就是 Git Bash |
| Node.js | 22.18(預設) | 由 **nvm** 管理(`~/.nvm`),另有 18.18 / 18.20 / 20.17 / 20.18 / 20.20 |
| npm 全域 | npm 10.9、yarn 1.22、corepack | 沒有其他全域套件 |
| Python | 3.14.3 | `C:\Users\AQUIVIO\AppData\Local\Python\bin\python.exe`。**沒有 `py` 啟動器** → 一律 `python -m pip ...` |
| pdftotext | 4.00 | Git for Windows 內建(`/mingw64/bin`) |

## Python 套件(共 35 個,工作相關的)

| 用途 | 套件 | 怎麼用 |
|---|---|---|
| PDF | `pymupdf` 1.28、`pypdf` 6.14、`pypdfium2` 5.11 | 文字:`pdftotext` 或 `pypdf`;要看圖表 / 版面:`pymupdf` 把頁面轉 PNG 再看(不必裝 poppler) |
| Excel | `openpyxl` 3.1 | 讀寫 .xlsx |
| Word | `python-docx` 1.2 | 讀寫 .docx |
| 數值 | `numpy` 2.4、`scipy` 1.17 | 濾波器係數驗算、訊號分析 |
| 串口 | `pyserial` 3.5 | |
| 其他 | `pillow` 12.3、`PyYAML` 6.0、`requests` 2.32 | |

> 下載或解壓的不明檔案,用 `python -I` 跑,避免載入同資料夾的同名模組。
> 中文輸出遇到 cp950 編碼錯誤:用 `python -I -X utf8`(`-I` 會忽略 `PYTHONIOENCODING`)。

## 沒有裝的(以前提過、或常被假設存在)

| 工具 | 狀況 / 替代 |
|---|---|
| `gh`(GitHub CLI) | 沒裝 → PR、Actions 到 GitHub 網頁操作 |
| `choco`、`srecord` | 都不在了 |
| `winget` | Git Bash 的 PATH 裡沒有 |
| `intelhex`、`opencv-python`、`FinMind` | 沒裝 |
| `cmake`、`java`、`ffmpeg`、ImageMagick | PATH 裡沒有(Flutter 建置用 VS 內建的 cmake,不受影響) |
| `make`、`openocd` | 沒有;STM32 建置用 CubeIDE |

**要裝新工具前先問使用者**(會動到他的系統)。裝完要更新這份。

## 環境注意事項(來自各專案記憶)

- **智慧型應用程式控制(Smart App Control)開著** → 會擋沒簽章的 DLL。Flutter debug 出現 `log reader stopped unexpectedly` 時,查 CodeIntegrity 事件 3077 看是哪個 DLL 被擋。關閉它是**不可逆**的,只說明、由使用者決定。
- Git Bash 路徑用正斜線 `/`,中文檔名用單引號。
- 不要用 shell 指令重新轉碼含中文的檔案。

## 重新盤點

```bash
flutter --version; dart --version; git --version; node -v; npm -v; python --version
python -m pip list
npm ls -g --depth=0; ls ~/.nvm/versions/node
"/c/Program Files/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin/STM32_Programmer_CLI.exe" --version
"/c/Program Files (x86)/Microsoft Visual Studio/Installer/vswhere.exe" -products '*' -format text | grep -E "^(displayName|installationVersion)"
flutter doctor
```
