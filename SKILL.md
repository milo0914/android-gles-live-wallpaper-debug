# Android GLES Live Wallpaper — 開發方法、坑清單與驗證腳本

從零開發 Android OpenGL ES 動態桌布的完整方法論。提煉自 RippleLiveWallpaper
任務（mission-20261031-103134，v1.0.0 → v2.7.0，十三個版本、十層疊加 bug
逐層剝開，OPPO Reno6 Z / Android 13 / ColorOS 13 實機驗收通過）。

**參考實作**（完整可運作的 Kotlin/GLSL，路徑相對 workspace root）：

| 檔案 | 內容 |
|---|---|
| `app/src/main/java/com/milo0914/ripplewallpaper/WallpaperGlRunner.kt` | raw EGL 渲染執行緒（含 lifecycle/可見性/pacing/錯誤重試） |
| `app/src/main/java/com/milo0914/ripplewallpaper/RippleRenderer.kt` | 三緩衝波動模擬＋雙 composite（modern/classic）＋魚池＋背景來源（asset/gallery/solid）＋EXIF |
| `app/src/main/assets/shaders/wave_sim.frag` | 穩定 Elias 水波 scheme（aspect 校正觸控注入） |
| `app/src/main/assets/shaders/composite.frag` | Modern 疊加式合成（背景為主角＋魚池＋clarity） |
| `app/src/main/assets/shaders/composite_classic.frag` | Classic Deep Ocean 合成（v1.1.4 原文＋V 翻轉＋光暈） |
| `app/src/main/java/com/milo0914/ripplewallpaper/MainActivity.kt` | 啟動頁（繞過 ColorOS 封鎖） |

---

## 執行（Execution）

本 skill 附一鍵驗證 batch `scripts/verify-build.json`（建置→lint→badging→dex
探測→md5）。改完 code 後**直接呼叫 batch**，不要逐步手打：

```
run_tool_batch(
  file_path="<本 skill 目錄>/scripts/verify-build.json",
  args={
    "repo_dir": "<workspace>/",
    "dex_probe": "loadGalleryTexture"
  }
)
```

### Batch Parameters

- `repo_dir`：Android 專案根（含 `gradlew`、`app/`、`settings.gradle.kts`）。
  預設＝workspace root；範例：`/run/csi/.../workspaces/default`
- `dex_probe`：要 grep 的新修復標記（類別名/函數名/shader 片段），用來確認
  新 code 真的進了 APK 的哪個 dex。範例：`loadGalleryTexture`（v1.1.4 的
  gallery 載入）、`idxPrevPrev`（三緩衝）、`Elias`（shader 改版）

判讀：`assemble_exit=0`＋`BUILD SUCCESSFUL`＋lint `0 errors`＋badging 的
`versionCode/versionName` 上升＋probe 命中 → 驗證通過。lint 偶發
`daemon disappeared`（容器記憶體緊）→ 直接重跑。

---

## 一、架構 Runbook（動態桌布的標準渲染架構）

1. **絕不用 GLSurfaceView**。它在 WallpaperService.Engine 中不會 attach 到
   視窗、holder 收不到 surface 事件 → `onDrawFrame` 一幀都不會跑。
   **用 raw EGL14/GLES30**：專門渲染執行緒持有 display/context/surface，
   Engine 的 `onSurfaceCreated/Changed/Destroyed/Visibility` 只負責餵狀態。
2. Engine 必須 `setTouchEventsEnabled(true)`，否則 `onTouchEvent` 永遠收不到。
3. 不要 `holder.setFixedSize(w,h)`——會把桌布表面縮小。
4. 渲染迴圈職責順序：等 surface 有效且 visible → 建/換 eglSurface →
   makeCurrent → 首次 init（shader/geometry/FBO/背景）→ sizeDirty 時 resize →
   draw → swap → 按 target frame time 睡眠。
5. **每個 init/resize/draw 都要 try-catch**：記 log、重置狀態、sleep 重試。
   讓任何單幀失敗都不會殺死執行緒（殺了=永久黑）。
6. `onSurfaceDestroyed` 內 join 執行緒（給 2.5s timeout）再返回。
7. 可見性：Engine.onVisibilityChanged → runner.setVisible；畫面不可見時
   釋放 EGL surface 並睡眠（省電）。

## 二、十層 Bug 坑表（按症狀定位）

| # | 症狀 | 根因 | 修法 |
|---|---|---|---|
| 1 | 桌面無圖示；ColorOS 桌布選擇器看不到 | 缺 MAIN/LAUNCHER；ColorOS 隱藏第三方動態桌布 | 加 launcher activity＋按鈕走 `ACTION_CHANGE_LIVE_WALLPAPER`（帶 service component） |
| 2 | 桌布完全靜止、永遠不畫 | GLSurfaceView（見架構 #1） | raw EGL 執行緒 |
| 3 | 畫面全黑；log 有 shader/FBO 錯誤重複拋 | prefs 型別錯配等每幀例外殺迴圈 | 見坑表 #5；執行緒錯誤重試 |
| 4 | 波紋永遠不出現（靜態圖） | ping-pong **feedback loop**（渲染目標同時被採樣，GLES UB；Adreno 讀 0） | **三緩衝輪替**：idxPrev/idxPrevPrev/idxTarget，每幀輪替，目標永不作輸入 |
| 5 | 改過數值設定後永久黑 | `EditTextPreference` 存**字串**，`getFloat()` 每幀拋 ClassCastException | 型別無關讀取：`prefs.all[key]` 接受 Number/String，壞值退預設 |
| 6 | 白→黑一次→永久黑 | 連續形式 laplacian（÷texelSize²）違反 CFL 條件 → 爆 Inf/NaN，NaN 永久污染緩衝 | **Elias scheme**：`h_new=(L+R+U+D)/2−h_prevPrev`，×damping；`clamp(h,±上限)` |
| 7 | 背景永遠全白（換紋理都一樣） | fresnel 用 `dot(-viewDir, normal)`——向下視角對朝上法線恆 ≤0 → fresnel 恆最大 → 水層 RGB 飽和成白 | 改 `dot(viewDir, normal)`；合成改**疊加式**：平靜處=純背景圖，波動處才混主題色（alpha≤0.7） |
| 8 | 上傳圖片上下顛倒 | `GLUtils.texImage2D` 頂列落在 v=0，quad 的 v=0 在螢幕底 | shader 取樣背景 `v' = 1 − v`（抽象漸層背景會掩蓋此 bug 多年） |
| 9 | 相機照片轉向/鏡像 | `decodeStream` 忽略 EXIF | 解碼後依 EXIF 重套旋轉/翻轉（Glide 對照表：90/180/270/FLIP_H/V/TRANSPOSE/TRANSVERSE） |
| 10 | 照片顏色洗白、看起來糊、暗部出現網格 | ①二次 gamma（貼圖已是 sRGB 又 pow 1/2.2）②無 mipmap 縮小鋸齒 ③焦散 sin×sin 網格在暗部被 gamma 放大 8 倍 | ①移除 gamma（管線全程 sRGB）②`GL_LINEAR_MIPMAP_LINEAR`＋`glGenerateMipmap` ③焦散閘控 `×motion`（只在波動處） |

**通用鐵律**：
- `glTexImage2D(..., null)` **不清零**（內容未定義）——配 `glClear` 或上傳真實零資料
- 均勻場是 Elias 的固點（2h−h=h）——初始垃圾會自我維持，必須確定性清零
- NaN 進了緩衝就是永久污染——夾限是唯一防線
- shader `pow(x<0, y)` = NaN——先 max(x, 0)
- sRGB 貼圖＋sRGB 調色盤＝管線不要再 gamma（二次 gamma=說不出哪裡不對的色偏）
- 裝飾雜訊（焦散/顆粒）在暗部會被感知放大——閘控或提供開關

## 三、症狀定位流程

1. **先確認渲染執行緒活著**：裝置裝 Logcat Reader，過濾 app tag；無 log 且
   全黑 → init 失敗迴圈或執行緒死亡；有畫面但錯誤 → 依症狀查坑表
2. **一次只改一層**：上層修好下層才會浮現（本任務六層 bug 層層掩護）——
   每版只動一個懷疑點，讓用戶的症狀回報成為下一層的診斷線索
3. **版本戳進 UI**：launcher 顯示 versionName，側載機上一眼分辨版本
4. **數學先驗**：波動/物理格式先算 CFL 等穩定條件再上 GPU；fresnel 這類
   光照公式按幾何意義驗符號（dot(-V,N) 在向下視角恆 ≤0）

## 四、建置與驗證（一鍵 batch）

見上方「執行（Execution）」——把本次重複 8 次的驗證鏈自動化：
`assembleDebug` → `lintDebug` → aapt badging（版本/launchable）→
dex 內容確認（新修復類別/字串真的進了 APK）→ APK 大小與 md5。

**每次改版 versionCode 必須 +1**（側載更新判定），versionName 顯示在
launcher UI 上（側載機上一眼分辨版本）。

## 五、交付（ColorOS 安裝說明模板）

APK 給用戶後，附這份說明（本任務反覆使用的定稿）：

1. 下載 `app-debug.apk`，用 LINE／雲端硬碟／USB 傳到手機
2. 點開 APK → 允許「安裝未知應用」→ 再點一次 → 安裝
   （同簽名直接蓋過舊版，設定保留；versionCode 需遞增）
3. App 列表點開「App 名稱」→ 核對啟動頁上的版本號
4. 點「Set Wallpaper」→ 系統預覽 →「設為桌布／套用」
   ——走系統標準預覽流程，繞過 ColorOS 隱藏第三方動態桌布的限制
5. 「Open Settings」調整（全部即時生效）

需求：Android 10+、OpenGL ES 3.0。聲音反應模式另需麥克風權限（執行期請求）。

## 六、實機除錯備援

模擬器在低記憶體容器（<4GB）會被 OOM 殺死，別硬撐——直接請用戶裝
「Logcat Reader」（免 root），過濾 app tag／`AndroidRuntime`，重現問題後
把 Exception 堆疊貼回來。有堆疊就能精準定位，勝過隔空推測。

## 七、漣漪效應雙模式（Modern / Classic Shimmer）— 心得、程式、算法

設定：Ripple Physics →「Ripple Effect」（`pref_ripple_mode`）：
`modern`（乾淨清晰，預設）／`classic`（v1.1.4 深海層次閃爍重現）。

### 心得：v1.1.4 美感之謎與重現

用戶記憶中 v1.1.4 的「光彩持久閃亮、漣漪波紋層次豐富」深海模式，來自兩個
後來被「修掉」的缺陷組合——重現它們（受控版本）就是 Classic 模式：

1. **高度場未初始化**（`glTexImage2D(null)` 內容未定義）：未定義記憶體恰形成
   大片穩定高原——均勻高度是波動格式的數學不動點（`sum/2 − prev = h`），
   永不衰減 → 層次持久。之後的「確定性清零」修掉垃圾初始化＝平靜水面。
2. **舊合成管線**：fresnel 恆最大（`dot(-V,N)` 符號 bug）、未夾限的
   signed-height alpha（負高度→背景增亮，正高度→水色主導）、
   `specular×(1+2h)` 波峰增益 → 明亮、高對比、層次豐富的發光水面。
   v1.1.5+ 的疊加式合成（平靜=純背景）乾淨但暗淡。

**教訓**：修 bug 時「垃圾行為」可能正是用戶喜愛的外觀——先確認用戶想保留
什麼，再把缺陷轉成受控功能（deterministic 重現），而不是直接刪除。
**診斷**：用戶提供的舊版 APK 是 ground truth——提取 assets/shaders 直接
比對，勝過隔空重建（本次隔空重建未命中，附件到手後一次命中）。

### 程式（關鍵實作）

**雙 composite program**（renderer 同時編譯 modern + classic，依模式切換）：

- `composite.frag`（modern）：疊加式合成＋fish pond＋clarity
- `composite_classic.frag`（classic）：v1.1.4 原文（用戶提供的 APK 提取，
  md5 6baa2bde 驗證）＋ V 翻轉（gallery 需要）＋ touch glow 附加

RippleRenderer 關鍵（模式切換與上傳）：

```kotlin
val classic = DesignTokens.rippleMode == "classic"
GLES30.glUseProgram(if (classic) compositeClassicProgram else compositeProgram)
// ... sampler/解析度/顏色依模式上傳到對應 location 集 ...
GLES30.glUniform1f(uLegacyLoc /*classic 備用*/, ...)
```

**Classic 播種**（seedClassicPlateaus）：小正方格（cell = height/24 ≈ 97px）
scissored glClear，每格隨機高度 ±0.35 → 三個 sim buffer 各自獨立隨機
（prev/prevPrev 不同 → 邊界立即產生波動）→ 開場細微紋理、數秒內融入。

**Classic 正圓**：v1.1.4 的觸控注入用 UV 距離（長螢幕上橢圓）——套用
v2.1.0 的 aspect 校正（`delta.x *= width/height`）。

### 算法

**Plateau 不動點**（Elias scheme）：均勻高度 h 的場：
`h_new = (4h)/2 − h = h` → 永不衰減。不同高度的 plateau 邊界產生階梯波，
慢慢均化。這是「層次持久」的數學基礎（也是 v1.1.4 白洗 bug 的機制——
垃圾高度自我維持）。

**Legacy 合成公式**（classic 模式的水面）：

```glsl
float depthL = 1.0 - exp(-height * 3.0);          // signed height!
vec4 water = waterColor + specular + fresnelColor; // fresnel ≡ max（符號 bug 原樣保留）
water.a = depthL * 0.85 + 0.15;                    // 未夾限：負 h → 背景增亮層
fragColor = background * (1.0 - water.a) + water;  // + 未閘控焦散 + gamma
```

**Clarity 雙模式通用**（v2.7.0）：classic 的常數洗白（waterColor+fresnel）
與 tint alpha 皆 ÷clarity → 自選背景色透出；高度調變閃爍（specular、caustics）
全強保留——classic 個性不變。

## 八、其他已驗證功能模組（v2.x 累積）

- **魚池**（v2.2.0）：程序化錦鯉（composite shader 錐形身體+擺尾+深度色偏+
  漣漪折射晃動）＋CPU 魚群 AI（漫遊/邊界轉向/閃避手指）；Fish Pond 設定分類
- **雨聲音效**（v2.2.0）：numpy 程序化合成 11.4s 無縫雨聲（低通噪音床+泊松
  雨滴脈衝+等功率交叉淡化）＋RainSoundPlayer（可見性閘控+音量 0-100）
- **通知漣漪**（v2.2.0）：NotificationListenerService→sink→spawnRandomRipple
  （跳 ongoing＋節流 1.2s）；設定頁含授權引導
- **Solid 自訂顏色**（v2.1.0）：pref_solid_color HEX→1×1 貼圖；**色域選色器**
  （v2.4.0）：ColorGamutView（hue bar＋SV 方塊，純 Canvas 漸層零依賴）
  →AlertDialog 即時預覽色塊/HEX
- **觸控光暈**（v1.3.0/加大 v1.3.1）：shader 高斯光暈（aspect 校正正圓、
  exp 1.2s 衰減、fresnel 主題色）——共用觸控槽位（雨滴/通知也發光）
- **Ripple Spread/Clarity**（v2.3.0）：spread=damping^(1/spread) 控制擴散距離；
  clarity=水霧÷clarity＋折射/高光×clarity——**v2.7.0 起 Classic 也生效**

## 九、已知未解

- **ColorOS 殺背景進程**：白名單治本（用戶端設定，非代碼）——處方見「症狀
  定位流程」與 memory/2026-11-08.md（聲音生死判別→時機→OEM 白名單）
- **release 更新直覆**：已驗證（同簽名 versionCode 遞增）；首次 debug→release
  遷移需解除安裝（簽名不符）
