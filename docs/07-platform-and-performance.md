# 07 · 三端兼容性、性能与构建可行性

> 口径铁律：**桌面 Web / Android 构建 / 真机 / 模拟器 / iOS 的数字不得互相替代**。未测的写 `unverified`，不得计入通过。
> 本机实测日期：2026-11-24（Flutter 3.50.1 / Dart 3.14.0 / engine `4d35714800`）。

## 1. 今天这台机器能验证什么

| 端 | 已验证 | 可验证 | 不可验证（必须标 unverified） |
|---|---|---|---|
| Web | 构建、产物构成、首访下载估算、桌面 Edge 端到端 | 分页度量、路由/刷新/返回、键盘、截图 | 手机浏览器、Safari、真机性能、长时间阅读内存 |
| Android | **构建成功**（debug 683.6s；release split APK；AAB）；体积 | 编译期静态检查、APK 体积门禁 | 安装/启动/帧率/内存/后台恢复/返回手势/不同 Android 版本 |
| iOS | 仅 Dart 静态分析与代码交付 | 无 | 编译、签名、运行、性能、安全区、手势 —— 全部 |

## 2. Web 现状（实测）

- **构建目录 58.68MB ≠ 用户下载量**。40 个文件中真正首访下载的是 1 个 CanvasKit 变体 + 1 份字体 + 主包。
- 首访实测（Edge/Chromium、默认 CDN、未子集字体）：**≈24.7MB raw / ≈13.8MB gzip**；其中字体 gzip **11.3MB（占 82%）**。
- 未子集字体 `NotoSansSC-VF.ttf` = 17,773,244 B（TTF gzip 后仍 11,297,204 B）。
- CanvasKit 默认从 gstatic CDN 加载；Chromium 用 `canvaskit/chromium`（5.46MB raw / 2.08MB gzip），WebKit/Gecko 用 full（7.32MB / 2.93MB）。
- `--wasm` 构建：skwasm 默认仅 Blink 启用；`main.dart.wasm` + `skwasm.wasm` gzip 约 1.97MB，比 CanvasKit 少约 0.5MB；Safari 回退 dart2js。
- 内置 `flutter_service_worker.js` 是"注销自身并 reload"的 stub，**默认零离线缓存**，PWA 必须自建。
- 冷启动 3.4–10.8s 的原始数据缺失（只有 1 次 `dataReadyMs=4407ms` + 若干缓存 reload 334–439ms），口径为桌面 headless，**不得与手机/原生比较**。
- `*.symbols` 调试符号 8.6MB（占构建目录 14.7%），**不得部署**。

## 3. Web 方案与预算

### 3.1 渲染器
- 基线：`flutter build web --release --wasm`（含 dart2js 回退）；Gate 0 对比 `--wasm` 与默认 CanvasKit 的首访、分页、浏览器兼容，再冻结。
- 保留 CanvasKit 作为保底构建；关闭 CDN，使用 `--no-web-resources-cdn` 自托管。
- 不采用 `preferWebParagraph`：正文断行必须自控。

### 3.2 字体（Web 第一瓶颈）
- **禁止**依赖 CanvasKit 自动回退（会得到错误分页或豆腐块）。
- 推荐 **B：单一 CJK 子集 + woff2**，常用 3,500–6,763 字，目标 ≤1.2MB（gzip）；App 端使用**同一子集**，保证三端分页一致。
- 生僻字兜底：系统字体回退 + 缺字提示；不允许静默改变分页。
- 子集化在 Gate 0 完成（需要 fontTools；本机未装，先确认可用工具链，体积为推断）。

### 3.3 首屏预算（gzip，硬门禁）

| 指标 | 桌面目标 | 中端手机 4G 目标 | 失败阈值 |
|---|---|---|---|
| 首访总下载 | ≤3.2MB | ≤3.5MB | >4MB |
| 主 JS/wasm 入口 | ≤450KB | 同 | >600KB |
| 字体子集 | ≤1.2MB | 同 | >2MB |
| TTFB | ≤200ms | ≤400ms | >600ms |
| 可交互 | ≤2.0s | ≤4.0s | >3s / >6s |
| 进阅读器字体就绪 | ≤300ms | ≤800ms | >1.5s |

### 3.4 加载与缓存
- 首期**不做**路由级 deferred（与 `--web-content-hash` 不兼容，弱网进阅读器会白屏）；Phase 5 再评估。
- 自建 Service Worker：precache 入口/wasm/字体；章节 JSON `stale-while-revalidate`；封面 cache-first + 版本化。
- 静态资源内容哈希 + `immutable`；`index.html` 与 bootstrap `no-cache`。
- 图片 WebP/AVIF + 懒加载，禁止大图入包。

### 3.5 已知风险
- WebKit 无 `Intl.v8BreakIterator` → 固定 full CanvasKit（多 0.85MB gzip、更高内存）。
- iOS Safari 的 WebGL 上下文回收、页面冻结、字体限制可能导致白屏/重排（本机无法验证）。
- 移动浏览器地址栏收起触发 `resize` → 必须"分页与视口解耦 + 防抖"，只在宽度变化时重排。

## 4. Android

- **实测可构建**：`flutter build apk --debug` 成功（683.6s，首次自动安装 platform-36 / build-tools-36 / NDK 28.2）；`flutter build apk --release --split-per-abi` 成功（arm64 APK **25,092,783 B**）；`flutter build appbundle --release` 成功（AAB 52,109,438 B）。
- 版本：Flutter 3.50 默认 `compileSdk/targetSdk 36`、`minSdk 24`。建议 **minSdk 24**（覆盖 Android 7+，面板/兼容成本低）；如坚持 26 需记录取舍。`targetSdk 36`，注意 Android 15+ edge-to-edge。
- 体积：arm64 release 25.1MB，字体压缩后约 11.8MB（47%）；子集字体后预计 ≤14MB。
- 签名：keystore 走 CI secret 或本地 `key.properties`；启用 Play App Signing；仓库无密钥。
- 构建耗时：冷环境约 11 分钟（Gradle 9.5 + SDK 组件 + NDK），热环境 release 约 95s；**CI 必须预热缓存**。
- 交互：`SafeArea` + `SystemChrome`；阅读器沉浸 `immersiveSticky`；返回统一 `PopScope`；翻页手势让出边缘 24dp 或正文页禁用边缘返回（真机验证）。
- AOT release；禁止用 debug 包做性能结论。

## 5. iOS（Windows 下）

- 能做：全部 Dart 代码、`ios/Runner` 配置、Podfile、签名/Fastlane 脚本、CI workflow、静态审查、`flutter analyze`。
- 不能做：编译、签名、模拟器/真机、性能、TestFlight。
- 最小验证路径（需 macOS）：GitHub Actions `macos-latest` → `flutter build ios --release --no-codesign` → 模拟器 `integration_test`（启动、分页金标、翻页耗时）→ `.app` 体积门禁。当前 GitHub 不可达，该路径**暂不可用**。
- 无 macOS 时 iOS 只能标 `unverified`。

## 6. 三端性能预算

| 指标 | 目标 | 方法 | 口径 |
|---|---|---|---|
| 首访下载 | ≤3.2MB gzip | 产物体积脚本 | 桌面 Web |
| 可交互 | ≤2.0s（桌面）/ ≤4.0s（手机） | Playwright `dataReadyMs` | 后者未验证 |
| 分页单章 | ≤8ms | 应用内计时（当前 2.0–2.55ms） | 桌面 Web/纯算法；不可跨端比 |
| 翻页帧率 | 60fps，p99 ≤32ms | DevTools/`integration_test` trace | Android 真机 |
| 阅读器内存 | PSS ≤220MB | `dumpsys meminfo` | Android 真机 |
| 冷启动到可读 | ≤1.2s | `am start -W` + 首帧打点 | Android 真机 |
| App 下载体积 | arm64 ≤14MB（子集字体后） | split APK / 商店报告 | 商店口径 |
| 分页金标 | 8 页/94 行/138px | 金标 JSON 对比 | 各端自测，跨端不互证 |

## 7. 自动化门禁（与测试体系对接）

- `tool/perf/check_web_size.mjs`：按 `budget.json` 对 `build/web` gzip 求和，超限 exit 1。
- `tool/perf/check_web_firstload.mjs`：Playwright headless 测 TTFB/数据就绪/首帧，输出 JSON。
- `tool/perf/check_apk_size`：split APK/AAB 字节数门禁。
- `budget.json` 是唯一阈值源；阈值变更需用户签字并记录。
- 性能基线（分页哈希 + 首载 JSON + 体积 JSON）入库，diff 即回归。

## 8. 风险与待拍板

**风险**：字体子集工具链未验证；Safari/移动 Web 未验证；无 Android 设备与 Mac 导致真机口径全空；Android 首次构建依赖公网（11 分钟、240MB+）；GitHub 不可达导致 macOS CI 暂不可用。

**待拍板**：
1. App 是否内置与 Web 相同的字体子集（+1–2MB 包体，换跨端分页一致）。
2. Web 是否全资源自托管（含 COOP/COEP，换取 skwasm 多线程）。
3. 是否投入 Android 真机与 macOS CI 预算（否则真机/iOS 永远未验证）。
4. 首访 ≤3.2MB gzip 是否作为发布硬门禁。
5. Web 范围：先"移动端可读 + 桌面可用"，还是必须完整公开发布（决定是否为 Safari 做双渲染器验证）。

## 9. 独立质疑后的修订（v2）

### 9.1 首屏门限分浏览器，且先分解再定门限
原"≤3.2MB / ≤2s"在算术上不成立：按实测分项相加，Chromium 路径约 3.69MB、WebKit 路径约 4.55MB（均含字体门限 1.2MB），而唯一一次冷加载实测 4,407ms。修订：

| 浏览器路径 | 首访 gzip 预算（候选） | 说明 |
|---|---:|---|
| Blink + skwasm（若采用 wasm 构建） | ≤3.4MB | 余量小，需计入 bootstrap JS |
| Blink + CanvasKit（v1 基线） | ≤4.2MB | Chromium 小变体 |
| WebKit + full CanvasKit | ≤5.2MB | 若纳入 Safari，必须接受该量级 |

- **门限暂标"候选"**，在 Gate 0 完成四段耗时分解（engine init / 主包下载 / 字体就绪 / 首次分页）后再进 DoD。
- 可交互目标改为：桌面 ≤3.0s、手机 4G ≤5.0s（候选）；当前 4.4s 冷加载作为基线记录，不承诺 2s。
- 若用户不接受 Safari 的量级，明确把 Safari 列为"不支持/未验证"，不做双渲染器承诺。

### 9.2 渲染器：v1 固定 dart2js + CanvasKit 自托管
`--wasm` 会把验证矩阵翻倍（Blink wasm / WebKit 回退 / dart2js），且 Safari 回退路径恰是超预算路径。v1 固定 **dart2js + CanvasKit + 自托管**；wasm 作为独立分支，只有在首载/分页/兼容三项指标通过后才合入。

### 9.3 字体：先定字表，再谈体积
- 子集字表：GB2312 常用字 + 产品文案用字（约 6,763 字），目标 ≤1.2MB woff2（**待实测**）。
- 缺字策略：缺字时用回退字体并**重新分页**（不是只换字形），并给"排版可能不同"提示；新增"含生僻字章节"负向金标。
- 子集后必须复跑 8 页/94 行金标；字体 sha 写入基线元数据。
- App 端与 Web 端使用同一子集，避免跨端分页差异。

### 9.4 离线与 Service Worker 单列任务
Flutter 3.50 生成的 `flutter_service_worker.js` 会注销自身，默认零离线缓存。新增任务（8–12 人日）：
- 自建 SW：precache 入口/wasm/字体；章节 JSON stale-while-revalidate；封面 cache-first + 版本化。
- 验收：断网刷新仍能进入已缓存章节；缓存版本升级不白屏；`--web-content-hash` 与 deferred 二选一（v1 不做 deferred）。
- 离线章节最小集：最近 N 章 + 已读章节（具体 N 待定）。

### 9.5 Android 构建前置
首次构建需下载 Gradle/SDK/NDK（实测 683.6s、240MB+）。CI 必须预热缓存或自建镜像；`compileSdk/targetSdk 36`、`minSdk 24` 已由实测构建验证；Gradle 版本与依赖锁定入库。

### 9.6 记忆口径统一
- 阅读器总内存：Android 真机 PSS ≤220MB（唯一总口径）。
- 页位图缓存：按可用内存与单页字节动态计算，不设"3 张/40MB"固定值；DPR 3.5 设备单页约 17.9MiB，必须按字节淘汰。
- 分页耗时：目标 8ms/章、失败线 16ms，单一源 `tool/perf/budget.json`。
