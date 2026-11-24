# 00 · 接管检查与项目真实状态

> 检查日期：2026-11-24 ｜ 执行：主控 Agent ｜ 方法：只读检查 + 实测统计 + 原型证据复核。
> 规则：旧报告是研究入口，不是现状。凡与代码/产物冲突，以实测为准并登记冲突。

## 1. 工作区与仓库状态

| 项 | 实测 | 结论 |
|---|---|---|
| 新仓库 `D:\Develop\Projects\novel` | 接管时**空目录、无 git**、无远程 | 仓库为全新起点；本阶段只写入方案文档 |
| 远程 `https://github.com/wwxst/novel` | 本机网络不可达（git 走失效代理 `127.0.0.1`，HTTPS 失败） | 无法 clone/push；本地 `git init` 后需解决网络再同步 |
| 既有 Android 参考工程 `D:\Develop\Projects\novel-app` | 非 git 仓库；含实现、设计规格、测试与证据；只读参考 | 不修改、不删除；迁移期间保留 |
| 既有研究/设计资料 | 页面/交互/视觉/数据/原型等资料完整 | 作为内部设计基线使用，不随新仓库发布 |

## 2. 环境实测（2026-11-24）

| 项 | 实测值 | 影响 |
|---|---|---|
| OS | Windows 11 26H2 (10.0.26300) | 无法构建/验证 iOS |
| Flutter / Dart | **3.50.1 stable / Dart 3.14.0**，engine `4d35714800`，位于 `D:\Develop\tools\flutter-sdk\flutter`（不在 PATH） | 可构建 Web；版本与既有原型一致 |
| Android SDK | `D:\Develop\tools\android-sdk`：初始仅有 platform **android-35**、build-tools 34/35；**无 emulator**；`adb devices` 为空。构建探针触发后已自动补装 **platform-36 / build-tools-36 / NDK 28.2**，并实测 `flutter build apk --debug` 成功 | 可编译打包，不能安装/真机验证；Android 构建能力已由探针证实 |
| JDK / Gradle | Temurin 21.0.12.1；Gradle 8.9 | 满足既有构建链 |
| Node | v24.20.0；playwright-core 1.65.1 + chromium 已装 | Web 自动化可本机运行 |
| 浏览器 | Edge 158.0.4370.33；**无 Chrome**（`flutter doctor` 报缺失）；无 WebKit | Web 验证限 Edge/Chromium；Safari 无法本机验证 |
| 网络 | pub.dev 可达；`maven.google.com` 检测超时；GitHub 不可达 | 依赖下载有风险；Android 构建可能被 Maven 阻断；CI 不能依赖 GitHub Actions |
| flutter doctor | Android toolchain 未配置（未指向 SDK）、Chrome 缺失、Maven 超时 | 阶段二开工首日必须先修工具链并做构建探针 |

## 3. 既有工程真实规模（实测，排除 `build/`）

| 指标 | 实测 | 说明 |
|---|---:|---|
| Gradle 模块 | 13 | 1 app + 1 app-shell + 6 core/feature 共 13 |
| Kotlin 文件 | 251 | main 167 / 单测 77 / testDebug 4 / androidTest 3 |
| 代码行（含空行） | 34,438 | main 25,524；测试 8,914 |
| 代码行（非空） | 30,386 | main **22,509**；测试 7,877（旧报告 22,627 为旧快照） |
| `@Test` 标记 | **513** | `src/test` 495 + `src/testDebug` 14 + `src/androidTest` 4 |
| 历史测试结果 | 176 个 JUnit XML / **937 条记录** / 0 failed / 0 error / 0 skipped | 去重后**唯一用例 509**；428 条差异=同一批用例在 debug/release 两个变体各跑一次；**本次未重跑** |
| Android 产物 | `app-debug.apk` 8,260,129 B（历史冻结基线） | 仅作参考，新仓库不复用该产物 |
| Mock 数据 | **182 个 JSON / 761,264 B**；12 本书 / 148 个章节文件 / 5 个榜单 tab / 8 个接口 | 正文为自写测试文本；封面/插图引用为 `mock://` 占位，仓库内无图片文件 |
| 自绘 View 类 | 47 个自定义 View/ViewGroup + 3 个 Dialog；其中 **23 个有 `onDraw`** | 旧口径"40 个自绘 View"不准确；换栈后 UI 代码 0 行可复用 |
| 已实现屏幕载体 | **11 个**（4 主 tab + 7 二级页）+ 设置页 | 旧口径"12 屏"把 MainActivity 与其 4 个 tab 重复计数 |

## 4. 旧报告口径修正（必须随引用带走）

| # | 旧口径 | 实测/更正 | 证据 |
|---|---|---|---|
| 1 | 主代码 22,627 行 | 当前 main **22,509 非空行 / 25,524 含空行** | 逐文件统计 |
| 2 | `@Test` 429 个 | **513** 个（含 testDebug 与 androidTest） | 源码扫描 |
| 3 | 840/903/919/937 "testcase 全绿" | 均为**历史记录数**，唯一用例 **509**；本次未重跑；androidTest 4 条从未执行 | JUnit XML 解析 |
| 4 | "12 屏" | **11 个载体 + 设置页**；第 2 个 tab 实为**分类**（设计基线中的"短剧 tab"未实现） | 代码 + 模块枚举 |
| 5 | 40 个自绘 View | 47 + 3，其中 23 个 `onDraw`；其余为组合式自定义 View | 类声明扫描 |
| 6 | 章标题 1.47em / 46px / 块高 203px | 现行为 **1.10em / 27px / 90px，块高 186px**；旧值会被 Web 像素差与代码同时证伪 | 排版代码 + 原型测量 |
| 7 | 内容盒 2190px、每页 15 行 | 现行为 **2187px、每页 14 行**（≤1 个段距的页可 15 行，属边界特例） | 排版常量 + 四端原型 |
| 8 | 原型"像素级一致" | 原型**未实现两端对齐**，右侧 ink 偏窄 8–11px 即由此产生；其结论不覆盖 justify | 原型代码 + Web 原型对照 |
| 9 | 文档称 `core:ui` 暴露通用组件 | 实际只有 3 个 token 文件（469 行），组件全部在各 feature 内自绘 | 目录枚举 |
| 10 | 文档称不做 TTS | 代码已有 TTS 控制器与入口（248 行），且存在降级路径 | 源码 |

## 5. 既有原型证据（可复核，不等于产品验收）

| 证据 | 实测值 | 可信度 |
|---|---|---|
| Flutter 分页金标 | 8 页 / 94 行 / `[10,0,14,14,14,14,14,14]` / 8 个页首锚点 / 每页首行字数 | 高：`flutter test`、Web release、JS 独立复算三方一致 |
| 行 pitch | 52.5714dp（=138px @2.625） | 高：布局值与 Web 截图像素双证 |
| 分页耗时 | 纯算法 1.97ms（JIT）；Web release 2.0–2.55ms | 高，但**只测算法，不含渲染/冷启** |
| Web 产物/字体 | 58.7MB 开发产物；未子集化字体 17.7MB；冷启动 3.4–10.8s（headless 桌面） | 高（测量口径已注明），**不能当作上线指标** |
| 未证明项 | 19 档字号 × 4 行距 × 5 边距、跨章/目录/进度恢复、真实图片、5 种翻页、选词、Android/iOS 真机帧率、skwasm | 未验证，必须进入阶段二补齐清单 |

## 6. 本次实际执行与未执行

- **已执行**：目录/规模/测试/数据统计；旧报告与代码交叉核对；原型 evidence 复核；工具链版本与网络探测；Web 原型产物静态分析。
- **未执行**：重跑既有 Android 测试、构建新 Flutter 工程、安装/运行 Android App、iOS 构建、Safari 验证、真实后端联调。
- **历史产物**（APK、JUnit XML、截图）只作对照，不作为本轮"已验证"依据。

## 7. 结论

1. 既有工程是一个**可运行、有测试与测量基线**的 Android 参考实现，但其账面代码对 Flutter 的复用率为 **0**；真正可迁移的是**算法行为、排版数值、数据契约、测试断言与设计规格**。
2. 旧报告存在多处口径过时（行数、测试数、屏幕数、排版参数），引用前必须按本文修正。
3. 当前环境可以支撑 **Web 开发与验证**、**Android 编译/打包**（debug 683.6s、release split APK 与 AAB 均成功），**不能**支撑 iOS 与 Android 真机验证；必须作为"未验证项"写入所有验收结论。
4. 阶段一的方案设计可以基于现有证据推进；阶段二开工前必须先完成工具链探针（见 `docs/09-roadmap.md` 的 Gate 0）。
