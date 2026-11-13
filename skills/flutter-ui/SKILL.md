---
name: flutter-ui
description: Flutter 桌面版 UI 防雷清單(Windows / Linux)。寫、改、檢查任何 Flutter 畫面前先載入 —— 版面配置、overflow 溢出、捲動與捲動條、縮放、圖表(CustomPainter)、對話框;也用於交付前的版面檢查。
---

# Flutter 桌面版 UI 防雷清單

兩段:**A. 通用基礎**(一般 Flutter 知識)、**B. 桌面實戰**(使用者各專案實際踩過、修過的坑,每條附出處)。
最後是 **C. 交付前檢查表**。

> 來源:A 段整理自 `C:\flutter\Flutter_FarmwareTesterUnifiedV01\aidocs\flutter_ui_patterns.md`(17 項,含完整程式範例,要看範例去那份);
> B 段整理自 `C:\flutter\` 與 `C:\flutter_\` 底下各專案程式碼裡的註解(2026-11-03)。
> 出處以「檔名 + 函式 / 變數名」定位,括號內行號只是查詢當時的參考,對不上就搜名稱。
> 這份只寫通則;某個專案「目前還沒修」的現況寫在該專案自己的 CLAUDE.md。

---

## A. 通用基礎

| # | 問題 | 解法 |
|---|---|---|
| 1 | `Row` / `Column` 子元件超出邊界(黃黑斜紋 `RenderFlex overflowed`) | 會伸縮的子元件包 `Expanded` / `Flexible` |
| 2 | `Column` 裡放 `ListView` → `Vertical viewport was given unbounded height` | **優先** `Expanded` 包住,或外層給固定高度;**項目少**才用 `shrinkWrap: true`(它會一次建出所有項目,上千筆的日誌會變慢) |
| 3 | 文字溢出不截斷 | `maxLines` + `overflow: TextOverflow.ellipsis`;在 `Row` 裡外層還要 `Flexible` / `Expanded` 給寬度 |
| 4 | 內容需要捲動 | `SingleChildScrollView` 包 `Column`;**桌面版一律加看得見的捲動條**(見 B1~B3) |
| 5 | 內容高度不確定 | `ConstrainedBox(maxHeight:)` + 內部捲動 |
| 6 | 視窗大小改變不自適應 | `LayoutBuilder` 依父容器實際寬度切換(不是 `MediaQuery`) |
| 7 | 一排按鈕塞不下 | `Wrap(spacing:, runSpacing:)`(例外見 B7) |
| 8 | Dialog 內容撐爆螢幕 | `mainAxisSize: MainAxisSize.min` + `ConstrainedBox` + 內部 `Flexible(SingleChildScrollView)` |
| 9 | 非同步回來時 widget 已銷毀 | `await` 之後、Timer 回呼裡先 `if (!mounted) return;` |
| 10 | 資源沒清 | `dispose()` 取消所有 Timer / StreamSubscription,dispose 所有 Controller / FocusNode / ValueNotifier |
| 11 | `Stack` 子元件溢出圓角 | 父容器 `clipBehavior: Clip.antiAlias` |
| 12 | 圖片載入失敗白屏 | `errorBuilder` |
| 13 | 桌面視窗縮太小 UI 壞掉 | `window_manager` 的 `WindowOptions(minimumSize:)` |
| 14 | 點擊沒有視覺回饋 | 按鈕類用 `InkWell`;純手勢才用 `GestureDetector` |
| 15 | 只為留白用 `Container` | 純間距用 `SizedBox` |
| 16 | Padding 各寫各的 | 統一 4 / 8 / 12 / 16 / 24 |
| 17 | 顏色寫死散落各處 | 集中成顏色常數類或 `ThemeData` |

---

## B. 桌面實戰

格式:**症狀 → 原因 → 解法 → 出處**。

### 捲動

#### B1. 滑鼠拖不動
- **症狀**:水平清單用滑鼠拖不動;滾輪又只捲垂直 → 實際上捲不了。
- **原因**:Flutter 預設的 `dragDevices` **不含滑鼠**(只有觸控等)。
- **解法**:
  ```dart
  ScrollConfiguration(
    behavior: ScrollConfiguration.of(context).copyWith(dragDevices: {
      PointerDeviceKind.touch, PointerDeviceKind.mouse,
      PointerDeviceKind.trackpad, PointerDeviceKind.stylus,
    }),
    child: ...,
  )
  ```
- **出處**:Max30102Tester `k2_page.dart` 的 `_snapshotViewer()` 快照列(約 293 行);AutoCleaningTester `adc_scope_page.dart`(註解寫「K2 的教訓」)。

#### B2. 可以捲,卻沒有捲動條
- **症狀**:使用者看不出「這裡可以捲」,**以為資料不見了**(K2 快照列實測就是這樣被回報的)。
- **原因**:預設不顯示捲動條;加了 `Scrollbar` 也只在捲動時才浮現。
- **解法**:`Scrollbar(thumbVisibility: true, ...)`。日誌框、清單、可捲的面板都要。
- **出處**:Max30102Tester `k2_page.dart` 的 `_snapshotViewer()`,`Scrollbar` + `ListView.separated`(約 286-308 行)。

#### B3. `Scrollbar` 與捲動元件要共用同一個 controller
- **症狀**:例外,捲動條抓不到位置。
- **解法**:
  ```dart
  final _scroll = ScrollController();   // dispose 時記得 _scroll.dispose()
  Scrollbar(
    controller: _scroll, thumbVisibility: true,
    child: ListView(controller: _scroll, ...),
  )
  ```
- **出處**:Max30102Tester `k2_page.dart` 的 `_snapScroll` 宣告(約 128-130 行);AutoCleaningTester `adc_scope_page.dart` 的 `_snapScroll`(約 63 行)。

#### B4. 日誌自動跟到最新,但使用者往上看時要停
- **症狀**:新資料一直進來,使用者往上捲查看時被拉回底部,根本看不了。
- **解法**:「貼底」旗標 —— 離底 < 40px 才跟隨;更新後下一幀再捲。
  ```dart
  bool _stick = true;
  // initState
  _scroll.addListener(() => _stick = _atBottom(_scroll));

  bool _atBottom(ScrollController c) {
    if (!c.hasClients) return true;
    final p = c.position;
    return p.maxScrollExtent - p.pixels < 40;
  }
  void _followBottom() {
    if (!mounted || !_stick) return;
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (!mounted || !_scroll.hasClients) return;
      final p = _scroll.position;
      if (p.pixels < p.maxScrollExtent) _scroll.jumpTo(p.maxScrollExtent);
    });
  }
  ```
- **出處**:AutoCleaningTester `max30102/widgets/max30102_page.dart` 的 `_logStick` 等旗標(約 52-59 行)、`_atBottom()` / `_followBottom()`(約 134-149 行)。

#### B5. 凍結表頭 / 左欄(Excel 式)
- **解法**:資料區雙向捲動,表頭跟水平、左欄跟垂直,用 listener 連動;**同步時要 `clamp`**,否則留白處越界。
  ```dart
  _bodyH.addListener(() {
    if (!_headerH.hasClients) return;
    final t = _bodyH.offset.clamp(
        _headerH.position.minScrollExtent, _headerH.position.maxScrollExtent);
    if (_headerH.offset != t) _headerH.jumpTo(t);
  });
  ```
  四區要固定行高 / 欄寬才對得齊。
- **出處**:FarmwareTesterUnifiedV01 `shared/widgets/report_view_page.dart` 的 `_ReportViewPageState`:四個 controller(`_bodyH` / `_bodyV` / `_headerH` / `_labelV`)與 `initState()` 的連動 listener(約 87-151 行)。

#### B6. 捲動區套捲動區
- **症狀**:滾輪先捲內層,捲到底才換外層 → 使用者以為頁面卡住。
- **解法**:內層給固定高度並顯示捲動條(B2),讓人看得出邊界;能不巢狀就不巢狀。(一般經驗)

### 溢出(overflow)

#### B7. 會隨時間變長的文字
- **症狀**:剛開機正常,跑久了才爆 —— 計數器從 3 位數長到 6 位數(`進料 #39531`、`RX 79999`),
  前面是固定寬度控制項,`Spacer` 被擠成 0 → 整行 `RenderFlex overflow`。
- **解法**:那段文字包 `Flexible` + `ellipsis`。寬度夠時行為一樣,不夠時從尾巴截斷。
- **出處**:Max30102Tester `k2_page.dart` 的 `_sec1Controls()`,即時心跳計數那段 `Flexible`(約 998-1008 行)。
- **通則**:任何**數字會長大**的欄位(計數、時間、檔名)都當成長文字處理。

#### B8. 固定高度的容器裡不能用 `Wrap`
- **症狀**:AppBar 這類高度固定的地方,`Wrap` 一換行就溢出。
- **解法**:改成單行水平捲動 `SingleChildScrollView(scrollDirection: Axis.horizontal, child: Row(mainAxisSize: MainAxisSize.min, ...))`。
- **出處**:Max30102Tester `k2_page.dart` 的 `_connectionBar()`(約 764 行)。

#### B9. `VerticalDivider` 放在 `Wrap` 裡會把高度撐開
- **解法**:改用固定高度的 `SizedBox(width: 1, height: 20, child: ColoredBox(...))`,或拿掉。
- **出處**:Max30102Tester `k2_page.dart` 的 `_connectionBar()` 開頭註解(約 751 行)。

#### B10. 小格子內容放不下(格狀面板)
- **解法**:`FittedBox(fit: BoxFit.scaleDown)` 縮小;外層包 `ClipRect` 當最後一道防線;文字仍要 `maxLines` + `ellipsis`。
  ```dart
  ClipRect(child: LayoutBuilder(builder: (context, c) => FittedBox(
    fit: BoxFit.scaleDown, alignment: Alignment.centerLeft,
    child: SizedBox(width: c.maxWidth, child: Column(mainAxisSize: MainAxisSize.min, ...)),
  )))
  ```
- **出處**:AutoCleaningTester `error_report_page.dart` 的 `_buildIdCell()`(約 700-735 行)。

### 版面結構

#### B11. 塞得下就填滿,塞不下才捲
- **解法**:上方主區塊給最小高度(對應最小視窗,例:最小視窗 650 高 → `_minTopHeight = 640`);
  可用高度不足、或展開下方面板時,才出現外層捲動。
- **出處**:AutoCleaningTester `adc_scope_page.dart` 的檔頭註解與 `_minTopHeight`(約 16-17、53 行);`error_report_page.dart` 的 `build()`(約 141-184 行)。

#### B12. ⚠️ 切換「要不要捲動」時,widget 結構要固定
- **症狀**:展開 / 收合面板後,**圖表的縮放、暫停狀態莫名其妙被重置**。畫面看起來完全正常,最難查。
- **原因**:展開時才多包一層 `SingleChildScrollView`,樹的結構變了 → 上方區塊的 element 被重建,State 全部重來。
- **解法**:**永遠**包著外層捲動,只切換 `thumbVisibility` 與內容:
  ```dart
  LayoutBuilder(builder: (context, c) {
    final topHeight = math.max(c.maxHeight, _minTopHeight);
    final scrollable = _showPanel || c.maxHeight < _minTopHeight;
    return Scrollbar(
      controller: _outer, thumbVisibility: scrollable,
      child: SingleChildScrollView(
        controller: _outer,
        child: Column(children: [
          SizedBox(height: topHeight, child: _buildTopArea()),
          if (_showPanel) _buildPanel(),
        ]),
      ),
    );
  })
  ```
- **出處**:AutoCleaningTester `adc_scope_page.dart` 的 `build()`(約 156-184 行)。
- **反例長相**(自己檢查時找這個):`if (!_showPanel) return topArea;`,展開時才 `return Scrollbar(child: SingleChildScrollView(...))`。

### 縮放與繪圖

#### B13. 圖表滾輪縮放,外層頁面也跟著捲
- **解法**:向 `pointerSignalResolver` 登記,這一次滾輪就只給圖:
  ```dart
  Listener(
    onPointerSignal: (e) {
      if (e is PointerScrollEvent) {
        GestureBinding.instance.pointerSignalResolver
            .register(e, (ev) => _onScroll(ev as PointerScrollEvent));
      }
    },
    child: ...,
  )
  ```
- 拖曳平移只在**已放大時**才啟用(`onHorizontalDragUpdate: zoom <= 1 ? null : ...`),否則和頁面捲動 / 點選互搶。
- 一定要有還原方式(雙擊還原)。
- 同時設 `onDoubleTap` 與 `onTapUp` 時,單擊要等雙擊判定逾時才觸發:`kDoubleTapTimeout` = 300ms(Flutter SDK `packages/flutter/lib/src/gestures/constants.dart`)。這是手勢機制本身的行為,不是 bug;單擊要即時反應就別在同一個 `GestureDetector` 掛雙擊。
- 一般圖片 / 版面要縮放,直接用 `InteractiveViewer`。
- **出處**:Max30102Tester `k2_wave_chart.dart` 的 `build()`,`Listener` + `GestureDetector`(約 226-250 行);AutoCleaningTester `adc_scope_chart.dart` 的 `_onPointerSignal()`(約 512 行)。

#### B14. CustomPainter 切換語言不重繪
- **原因**:painter 沒有 `BuildContext`;切語言時資料可能一個 bit 都沒變,`shouldRepaint` 回 false。
- **解法**:painter 用到的**所有外部狀態**(語言、主題、縮放…)都當建構參數傳入,並放進 `shouldRepaint` 比較。
- **出處**:Max30102Tester `k2_wave_chart.dart` / `k2_hrv_chart.dart`(專案 CLAUDE.md「多語系」一節)。

#### B15. 毛玻璃對話框
- 沒包 `ClipRRect` → 模糊擴散到**整個畫面**。
- `Dialog` 沒設 `backgroundColor: Colors.transparent` → Material 白底擋在前面,模糊看不到。
  ```dart
  Dialog(
    backgroundColor: Colors.transparent, elevation: 0,
    child: ClipRRect(
      borderRadius: BorderRadius.circular(20),
      child: BackdropFilter(filter: ImageFilter.blur(sigmaX: 22, sigmaY: 22), child: ...),
    ),
  )
  ```
- **出處**:Max30102Tester `k2_field_help.dart` 的 `_glassDialog()`(約 116-130 行)、`shared/ui/glass.dart`。

### 檢查方法

#### B16. overflow 只在 debug 看得到
- 黃黑斜紋只在 debug 出現;**release 版不報錯,內容被靜靜切掉**。debug 看到一次就修,不能因為 release「看起來沒事」就放過。

#### B17. 多語系:英文比中文長
- 中文剛好塞得下的地方,切成英文就爆。按鈕、標籤、表頭最常中。
- 要在 **最小視窗 + 英文** 的組合下看過一次。

---

## C. 交付前檢查表

- [ ] 視窗縮到 `minimumSize`,每一頁看過一次
- [ ] 切成英文,同樣再看一次(B17)
- [ ] 計數 / 時間 / 檔名塞入很長的值(B7)
- [ ] 每個能捲動的地方:看得到捲動條(B2)、滑鼠拖得動(B1)
- [ ] 展開 / 收合所有面板,確認圖表縮放與其他狀態沒被重置(B12)
- [ ] 圖表上滾輪,外層頁面不會跟著捲(B13)
- [ ] 在 debug 模式下跑,沒有任何黃黑斜紋(B16)
- [ ] 每個 Controller / Timer / Subscription 都在 `dispose()` 清掉(A10)
