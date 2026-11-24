# 03 · Flutter 三端统一架构（方案，待批准）

> 技术路线已定：Flutter 单代码库覆盖 Android / iOS / Web。本文只回答"怎么组织代码"，不重新论证选型。
> 目标：个人可维护、边界清楚、纯算法可独立测试、三端差异只出现在适配层。

## 1. 核心裁决

| # | 裁决 | 结论 | 理由 |
|---|---|---|---|
| D1 | 单包 vs 多包 | **单 pubspec 包**，不引入 melos/多包 | 单交付物、单人维护，包切分对编译产物与 tree-shaking 无收益；边界由静态检查脚本保证，而非包边界 |
| D2 | 状态管理 | **Riverpod 3.x**（手写 Notifier） | 需要：阅读设置的细粒度选择、重排可取消、进度生命周期钩子、搜索状态机、异步三态；`ChangeNotifier` 无法提供取消/生命周期；Bloc 样板成本过高 |
| D3 | 路由 | **go_router 18.x** + path URL | Web 深链/刷新/前进后退、tab 独立返回栈、Android 返回拦截、iOS 边缘手势配置 |
| D4 | 本地存储 | **drift + shared_preferences**（先做 Web 探针） | 结构化查询（排序/聚合/事务）与 Room 语义最接近；KV 保留设置项。Web 体积/初始化超门限则回退 sembast |
| D5 | 代码生成 | 仅 `drift_dev` + `build_runner`；**不引 freezed**；8 个 DTO 手写 `fromJson`（必要时再加 json_serializable）；riverpod_generator 不引 | 模型约 500 行，手写 copyWith/equals 成本可控；少一条 analyzer 版本锁（freezed 与生成链升级冲突） |
| D6 | 后端 | 仓库预留 `backend/`，本期只有 OpenAPI 契约与说明 | 本阶段无服务端，不得伪造同步能力 |
| D7 | 平台差异 | 全部收敛到 `lib/platform/` 的条件导入接口 | 业务代码禁止散落 `Platform.isXxx` |

## 2. 目录结构

```
novel/
├─ lib/
│  ├─ main.dart                  # 字体/偏好就绪 → runApp
│  ├─ bootstrap/                 # AppGraph（ProviderContainer）唯一装配点、启动门
│  ├─ app/                       # MaterialApp.router、主题、外壳、底部导航
│  ├─ router/                    # 路由表、返回策略、深链
│  ├─ design/                    # tokens.g.dart（生成，禁改）+ 主题/皮肤
│  ├─ widgets/                   # 跨 feature 组件
│  ├─ domain/                    # 模型 + 仓储接口（零平台依赖）
│  ├─ core/util/                 # 时间、缓存 TTL、日志、结果类型
│  ├─ data/
│  │  ├─ api/                    # CatalogApi / MockApi / RemoteApi + DTO
│  │  ├─ repositories/           # 9 个仓储实现
│  │  ├─ db/                     # drift 表、DAO、迁移
│  │  ├─ kv/                     # shared_preferences 封装
│  │  └─ blobstore/              # 章节正文文件/IndexedDB
│  ├─ reader/
│  │  ├─ model/ layout/ progress/ nav/   # 纯 Dart 算法（禁 flutter/dart:ui）
│  │  └─ ui/                     # TextPainter 断行、CustomPainter、翻页、面板
│  ├─ features/                  # home / category / shelf / search / detail / mine / history / messages / settings
│  └─ platform/                  # 字体、文件、返回手势、系统 UI、Web 文本镜像
├─ assets/mock/                  # 182 份自写 JSON + LICENSES.md
├─ design/tokens/                # ui.tokens.json / reader.themes.json / shell.skin.json / breakpoints.json
├─ backend/                      # 预留：openapi/ + README + .env.example，本期无实现
├─ docs/                         # 本方案、规格、证据、验收
├─ tools/                        # arch_check / gen_tokens / mock_verify / capture / measure
└─ test/                         # unit / widget / golden / integration
```

## 3. 分层与依赖规则（可机器检查）

| 层 | 允许依赖 | 硬禁止 |
|---|---|---|
| shell（app/router/bootstrap） | 全部 | `dart:io` |
| features | design/widgets/domain/data 接口/reader 公开 API | 直接 import drift、shared_preferences、`dart:io` |
| reader-ui | reader 纯层、platform、design | drift、`dart:io` |
| reader-core | domain/model、core/util | **package:flutter、dart:ui**、其他层 |
| data | domain/model、core/util、platform(blob) | features、widgets、design |
| domain/core/design | 仅 `dart:*` + collection/meta | 其他层 |

门禁命令：

```powershell
dart run tool/arch_check.dart          # 层表 + 循环 import 检测，违规退出码 1
flutter analyze --fatal-infos
flutter test test/unit --reporter expanded
```

`arch_check` 解析 import 图，禁止反向边与环；另检查 `lib/` 中除 `design/tokens.g.dart` 与 `reader/ui/painters/` 外不得出现裸色值与裸 dp 常量。

## 4. 状态管理

| 场景 | 方案 |
|---|---|
| 阅读设置（字号/行距/边距/主题/翻页） | 每个字段一个 `Notifier` + `select` 精确重建；快照为派生 provider |
| 阅读重排 | `AsyncNotifier` + generation token，旧分页结果自动作废 |
| 阅读进度自动保存 | provider 生命周期 + `AppLifecycleListener`，20s 聚合，切章/退后台强制刷 |
| tab 保活 | `StatefulShellRoute.indexedStack`，不靠全局 provider |
| 搜索 6 态 | `Notifier<SearchUiState>` 保留 sealed 状态机，`autoDispose` |
| Mock 异步 | `FutureProvider` + `AsyncValue`；`MockScenario` 用 provider override 注入 |
| 按钮瞬态/动画 | `ChangeNotifier`/`AnimationController`，不进跨页数据流 |

## 5. 路由表（深链与返回语义）

| # | 路径 | 页面 | URL 参数 | 返回/刷新语义 |
|---|---|---|---|---|
| 1 | `/` | 重定向 `/home` | — | — |
| 2 | `/home` | 书城 | `?channel=recommend\|novel` | tab 根，返回退出 |
| 3 | `/category` | 分类 | — | tab 根 |
| 4 | `/shelf` | 书架 | `?tab=books\|history\|favorites\|circle` | tab 根 |
| 5 | `/mine` | 我的 | — | tab 根 |
| 6 | `/search` | 搜索入口 | — | 刷新回入口 |
| 7 | `/search/results` | 结果/空/错 | `?q=` | 刷新按 q 重跑 |
| 8 | `/rank/:tabId` | 完整榜单 | path `tabId`，`?category=` | 刷新可直出；分类/类型栏切换 |
| 9 | `/book/:bookId` | 详情 | `?from=` | 可直出 |
| 10 | `/read/:bookId/cover` | 书封页 | `?from=` | 可直出 |
| 11 | `/read/:bookId` | 正文 | `?chapter=&para=` | 按段落锚点重定位；**页码不进 URL** |
| 12 | `/settings` | 设置 | — | 可直出 |
| 13 | `/history` | 阅读历史 | — | 可直出 |
| 14 | `/messages` | 消息 | — | 可直出 |
| 15 | `/404` | 兜底 | — | 托管侧 SPA rewrite |

要点：
- 书封页与正文是两个 URL，避免"同一页两个初态"导致 Web 返回语义混乱。
- `/book/:bookId`（独立详情页）与 `/read/:bookId/cover`（阅读器扉页）是**两个不同 UI**：前者是书籍信息页，后者属于阅读器沉浸态；返回栈分别回到来源页。
- 返回语义：先逐层关闭覆盖层（评论→设置→目录→菜单）；在正文根状态**退出阅读器**（不做"返回=上一页"，避免与浏览器/系统手势冲突）。Web 浏览器后退永远离开阅读器，**不劫持 history**。
- iOS 二级页保留边缘返回；正文页是否关闭平台边缘手势取决于单击语义拍板（见 `docs/10-risks-and-decisions.md`），关闭后返回只走菜单/顶栏。
- Web 用 path URL 策略，部署需 SPA rewrite；`--base-href` 与部署子路径一致。

## 6. 数据/网络接口与错误模型

- 9 个仓储接口：Book / Chapter / Shelf / Progress / Search / ReaderSettings / AppSettings / Follow / ReadingTime。**不预写同步空实现**：同步只保留契约文档，接服务端时新增 `SyncService` 与 `sync_outbox`（见 `docs/06-data-and-backend.md`）。
- 命令返回 `Future`，观察流返回 `Stream`；UI 只依赖接口。
- 统一错误：`DataError { network, notFound, parse, storage, conflict, unsupported }`，UI 侧单一文案映射表。
- Mock 契约 8 方法：`getBookList / getBookDetail / getCatalog / getChapter / getRankTabs / getRank / search / getHotWords`。
- Mock 与真实后端通过 `CatalogApi` 接口 + `AppSettings.dataSource` + `--dart-define apiBaseUrl` 三处切换；默认 mock，仓库不放真实地址。
- 布局指纹使用自实现 FNV-1a（同步、无依赖），避免 Web Crypto 异步接口。

## 7. 平台适配层（只保留真实差异）

| 接缝 | 接口 | Android/iOS | Web |
|---|---|---|---|
| 字体 | `ensureReaderFont()` / `bodyFamily` | 打包字体 + 系统字体 | 打包子集字体 + `FontLoader`，失败显式降级 |
| 正文文件 | `ChapterBlobStore`（read/write/delete/size） | `path_provider` + File | IndexedDB / OPFS |
| Web 文本选择 | `SelectionBridge` | 空实现（系统选择） | Canvas 上叠加**整章** DOM 文本镜像 |

返回手势与系统 UI **不另建接口**：直接在壳层使用 `PopScope`、`CupertinoPage`、`SystemChrome`，避免为同一 API 包一层。

条件导入示例：`import 'blobstore_io.dart' if (dart.library.js_interop) 'blobstore_web.dart';`

## 8. 构建配置

| 项 | 取值 | 说明 |
|---|---|---|
| applicationId / bundleId | **待拍板**（建议自有域名反向，如 `app.<domain>.novel`） | 占位符不可上架；禁用旧参考工程的包名 |
| minSdk | **24**（建议；可覆盖 Android 7.0+） | Flutter 3.50 默认；如沿用旧基线 26 需记录取舍（见决策 U13） |
| compileSdk / targetSdk | **36** | Flutter 3.50 默认；构建探针已在本机自动安装 platform-36 / build-tools-36 / NDK 28.2，实测可构建 |
| 签名 | `android/key.properties`（gitignore）+ `key.properties.example` | 缺配置时 release 构建 fail-fast；仓库无密钥 |
| iOS | `IPHONEOS_DEPLOYMENT_TARGET=15.0` | Windows 只交代码，不验证 |
| Web | CanvasKit 首期 + `--no-web-resources-cdn`（自托管） | skwasm 二期冒烟；path URL + SPA rewrite |
| 环境 | 无 git 依赖；版本锁定；lockfile 入库 | 网络受限环境可离线构建 |

## 9. 依赖清单（阶段一候选，批准后锁定）

| 包 | 版本 | 用途 | 风险 |
|---|---|---|---|
| `material_ui` | ^1.1.0 | Material 组件（Flutter 3.50 起 `package:flutter/material.dart` 已弃用） | 新拆分包；`flutter create` 模板已默认依赖 |
| `cupertino_ui` | ^1.1.1（传递） | Cupertino 组件/手势 | 由 material_ui/go_router 传递 |
| flutter_riverpod | 3.5.0 | 状态/依赖注入 | 3.x 与 2.x API 差异，示例多为 2.x |
| go_router | 18.1.4 | 路由/深链 | 传递引入 material_ui/cupertino_ui；**体积归因应为字体 > 引擎 > 代码**，不设"自写 Router"预案 |
| drift / drift_dev | 2.35.4（候选，Gate 0 探针后锁定） | 本地 SQL | Web 需 sqlite3 wasm，体积/初始化必须探针（否则切 sembast） |
| sqlite3 | 3.9.2 | drift 后端 | 同上 |
| shared_preferences | 2.5.6 | KV 设置 | 无 |
| path_provider | 2.1.6 | 文件目录（io） | Web 无实现，接口已隔离 |
| build_runner | 2.17.1 | drift 代码生成 | 生成物入库，diff 噪声 |
| flutter_lints | 6.0.0 | lint | 无 |
| wakelock_plus | 1.8.2 | 阅读常亮 | Web 依赖 Wake Lock API，需实测 |
| **不引** | — | freezed / json_serializable（首期）/ provider / bloc / crypto / http | 无真实网络；http 仅测试需要时加入 |

**依赖前置门（S0）**：9 个直接依赖 + 传递依赖必须先预拉进 pub cache，并证明 `flutter pub get --offline` 可完成、lockfile 入库；否则不得以"版本已兼容"作为排期前提。代码生成物（`*.g.dart`）入库，保证离线可构建；CI 追加"重新生成后 diff 为空"检查。

## 10. 后端预留

```
backend/
├─ README.md              # 定位与边界：当前不实现
├─ openapi/novel-v1.yaml  # 账号(匿名)/书架/进度/设置的契约
├─ migrations/0001_init.sql
├─ fixtures/              # 与 mock 同结构样本
└─ .env.example           # 只有键名，无值
```

客户端当前**不调用**任何远程服务；`apiBaseUrl` 默认空串，UI 不出现"已同步/云书架"。

## 11. 方案待确认项（真正开放的决策）

1. 包名 / applicationId / 显示名称。
2. Web 是否纳入 Safari（决定首屏门限与渲染器矩阵）。
3. 是否提供 Android 真机与 macOS 环境（决定真机/iOS 验证能否离开 `unverified`）。
4. 字体选型与家族数（v1 建议 1 个打包字体 + 系统字体）。
5. TTS / 划词笔记 / 离线下载是否进入 v1（当前建议：TTS 与离线做最小路径，划词笔记延后并书面登记）。
6. 后端技术栈（本阶段只放契约，不锁实现）。
