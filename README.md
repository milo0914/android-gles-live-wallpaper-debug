# android-gles-live-wallpaper-debug

Android OpenGL ES 動態桌布的除錯方法論、十層 bug 坑表、雙漣漪效應模式
（Modern / Classic Deep Ocean shimmer）與一鍵建置驗證腳本。

提煉自 **Ripple Live Wallpaper** 專案（v1.0.0 → v2.7.0，十三個版本，
OPPO Reno6 Z / Android 13 / ColorOS 13 實機驗收）：raw EGL 渲染執行緒、
Elias 水波格式、三緩衝輪替、雙 composite program（Modern 疊加式／
Classic 深海層次閃爍）、魚池、雨聲音效、通知漣漪、色域選色器、
ColorOS 安裝繞道與電池白名單處方。

## 想要完整版？DeepRipple Pro（$19，一次性）

本 repo 是免費 Lite 版：架構 runbook＋十層 bug 坑表摘要。完整版再多這些：

- 完整 shader 原文：Modern（v2.4.0、v1.2.0）＋ Classic 深海閃爍（v2.6.0/v2.7.0）
- 症狀→原因診斷流程圖＋上線前檢查清單
- 可攜式一鍵驗證腳本 `scripts/verify-build.sh`（不用 QwenPaw，assembleDebug → lint → aapt badging → dex probe）
- 完整 Modern/Classic composite 公式、plateau seeding、clarity 規則

購買連結：

- Gumroad：https://hemohuang.gumroad.com/l/deepripple-pro-android-gles-live-wallpaper-debugging-playbook
- Agensi：https://www.agensi.io/skills/deepripple-pro-android-gles-live-wallpaper-debugging

完整版不含 RippleLiveWallpaper 完整 app 原始碼、APK 或圖片／音訊素材——它是除錯方法論，不是 app 本身。授權為個人、不可轉讓，不可轉發或轉售。

## 內容

| 檔案 | 說明 |
|---|---|
| `SKILL.md` | 完整方法論：架構 runbook、十層 bug 坑表、症狀定位流程、Shimmer 模式心得/程式/算法、ColorOS 交付模板 |
| `scripts/verify-build.json` | 一鍵驗證 batch（assembleDebug → lint → aapt badging → dex probe → md5），搭配 QwenPaw `run_tool_batch` |

## 由來

本 skill 由 AI 協作開發流程沉澱：十層疊加 bug（GLSurfaceView 架構錯誤→
feedback loop→型別崩潰→CFL 爆炸→fresnel 符號→貼圖翻轉→EXIF→二次 gamma→
mipmap→焦散閘控）逐層剝離，每層由用戶實機症狀回報定位。
