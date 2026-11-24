# 02 · 可复用成果清单与迁移映射

> 原则：**换栈后可直接复用的代码为 0 行，但可复用的是"已被测试与测量固化的行为规格"**。
> 迁移口径：算法保行为、数值照抄、数据契约照搬、测试断言改写；已确认的错误实现不迁移。

## 1. 资产总览

| # | 资产 | 位置（既有参考工程） | 规模 | 验证状态 | 新仓库落点 | 迁移方式 |
|---|---|---|---:|---|---|---|
| 1 | 断行 + 装页引擎 | `core/util/pagination/`（CjkTextBreaker / PageComposer / PaginationTypes） | 533 行（含空行） | 单测 + 四端金标 | `lib/reader/layout/` | 纯 Dart 重写，行为逐项对齐 |
| 2 | 分页编排 / 渲染计划 | `feature/reader/engine/`（ReaderPaginator / ReaderRenderPlanner） | 365 行 | 单测 | `lib/reader/layout/` | 重写；补分页缓存 |
| 3 | 进度锚点 / 布局指纹 | `core/util/progress/PositionRestorer.kt` | 191 行 | 单测（幂等 24→28→24） | `lib/reader/progress/` | 保留算法，修指纹缺陷 |
| 4 | 排版常量 | `ReaderTypography.kt` | 198 行 | 像素实测标定 | `lib/reader/layout/typography.dart` | 数值照抄（见 §3） |
| 5 | 章节导航/切分 | `ChapterNavigator / ChapterSplitter` | 95 行 | 单测 | `lib/reader/nav/` | 保留 |
| 6 | 设计 token | `Tokens.kt / AppSkin.kt / ReaderTheme.kt` | 469 行 | 与资源 XML 互锁测试 | `design/tokens/*.json` → `lib/design/` | 数值照抄，JSON 单一源 |
| 7 | 数据模型 | `core/model/` 4 文件 | 505 行 | 单测 | `lib/domain/model/` | 转写，补同步预留字段 |
| 8 | Mock API + DTO + 182 JSON | `core/net/` | 496 行 + 761,264 B | 完整性测试（字符串级） | `lib/data/api/` + `assets/mock/` | 8 接口契约；JSON 原样搬入，manifest 升 v2 |
| 9 | 数据层纯逻辑 | `core/data/` 21 个无 Android import 文件 | ≈1,671 行 | 单测（含 Robolectric） | `lib/data/` | Room→drift，仓储接口重排 |
| 10 | 业务规则（首页/详情/搜索/书架/我的） | 各 feature 的 `*Contracts / *UiState / *Catalog` | ≈1,900 行 | 部分单测 | `lib/features/*/` | 规则转写，UI 重写 |
| 11 | 阅读器其余规则 | 阅读器 model / progress / settings / TTS | ≈715 行 | 部分单测 | `lib/reader/**` | 转写，TTS 降级保留 |
| 12 | 测试断言 | 513 个 `@Test` | 509 可执行 | 历史全绿（未重跑） | `test/` | 447 条可 1:1 改写；62 条 Robolectric 重设计 |
| 13 | 设计规格 | 视觉规范 + 11 份页面规格 + 阅读器规格 | 约 3,000 行 | 像素测量 | `docs/05-ui-system.md`、`docs/04-reader-engine.md` | 数值与状态表迁入 |
| 14 | 像素/度量基线 | 对照截图、测量脚本、金标 JSON | 大量 | 已有测量 | `docs/evidence/baselines/` + `tools/` | 方法与金标复用；旧图不入库 |
| 15 | 素材合规结论 | 182 JSON 抽查 | 182 文件 | 已扫描 | `assets/mock/LICENSES.md` | 自写素材保留，3 项处置 |
| 16 | 构建与工具脚本 | 签名、apk 安装、截图、diff、度量 | 约 30 个脚本 | 已运行 | `tools/` | 按新栈重写/复用 |

## 2. 阅读器算法资产（最关键的迁移对象）

| 算法 | 行为要点 | 保留/修 |
|---|---|---|
| 断行（CJK 逐字测宽） | 逐字累计宽度；避头尾禁则；返回每行 `charStart/charEnd` | 保留；修 UTF-16 代理对（emoji/扩展 B 汉字）按码点处理 |
| 装页 | 标题块独占页顶；页首段不加段距；跨段加 gap；超界换页；段落可跨页；图片块独占页；`isActive` 可取消 | 保留；修图片页 `usedHeight` 与超高标题边界 |
| 两端对齐 | 非段末行均摊字间距，末行左对齐，保留 0.5px 防二次换行 | **必须迁移**；既有 Flutter 原型未实现，是本轮新增验证项 |
| 行位置 | 行 pitch 固定 52.5714dp；基线按"行盒居中"公式摆放 | 保留；禁止用 `TextStyle.height` 堆行 |
| 锚点恢复 | 指纹相同走快路径 → 段/字符定位 → 章内字符 → 章首 → 页 0 | 保留；指纹补字体度量摘要、密度、标题块高、对齐开关 |
| 重排 | 字号/行距/边距/字体/旋屏触发；120ms 防抖；可取消；幂等（24→28→24 回原页 ±1） | 保留；补分页缓存 |
| 进度 | 页码非权威；20s 聚合落库；切章/退后台强刷 | 保留；页脚口径需拍板 |
| 翻页 | 5 模式（仿真/覆盖/平移/上下/无动画）；阈值 18% 或 600px/s；回弹 160ms | 重写为 `AnimationController` + CustomPainter |
| 章导航 | 页内/跨章/书末三态 | 保留 |

## 3. 必须照抄的排版数值（唯一基线）

| 参数 | 值（@24dp / 2.625dpr） | 单位 |
|---|---|---|
| 正文字号 | 24（19 档：13–44） | dp |
| 行 pitch | 52.5714（=138px） | dp |
| 段间额外 | 19.0476（=50px） | dp |
| 首行缩进 | 2em = 48（=126px） | dp |
| 左右边距 | 24（=63px） | dp |
| 顶/底 inset | 44.19 / 36.95 | dp |
| 内容盒 | 363.43 × 833.14（=954×2187px） | dp |
| 章标题 | 1.10em（26.4）＋上间距 10.29（27px）＋距正文 34.29（90px），块高 70.97（186px） | dp |
| 每页行数 | 14（≤1 个段距的页可 15 行） | 行 |
| 7 档主题 | 底 `#F6F6F6 / #E8E3CE / #E4F0D8 / #D8E4F0 / #0E0E0E / #1A1A1A / #262626` | — |
| 5 档边距 | 20 / 24 / 28 / 32 / 智能=24 | dp |
| 4 档行距 pitch | 113 / 129 / 138 / 153 | px@24dp |

## 4. 不迁移的内容（重新实现或丢弃）

| 内容 | 原因 |
|---|---|
| 40+ 自绘 View、XML 壳、ViewBinding | Android 专属，Flutter 需重写；但类职责作为组件规格保留 |
| Robolectric / instrumentation 测试设施 | 无等价运行时；改用纯 Dart/widget/integration 测试 |
| Room 实体/DAO 生成的实现 | 换 drift；仅保留表结构与 SQL 语义 |
| Activity/Intent 导航与反射装配 | 换 go_router；仅保留路由语义 |
| 平台 TTS 直接调用 | 换 Flutter 适配层；保留文本切分规则 |
| 页位图缓存实现（`LruCache<Int, Bitmap>`） | 换 `ui.Image` 缓存，容量按字节 |
| 像素级整页相似度作为验收判据 | 内容/素材差异不可控，改为分区 + 度量判据 |

## 5. 数据与素材复用

| 项 | 结论 |
|---|---|
| 182 份 JSON（12 书 / 148 章 / 5 榜单 / 8 接口） | **原样复用**，唯一 100% 跨栈资产；补 `manifest` v2（文件清单 + sha256 + 许可） |
| 正文文本 | 自写测试文本（4,410 块仅 51 种文本轮转），无第三方正文；保留 |
| 评分/热词/书评 | 自写虚构；保留并登记来源 |
| 封面/插图 | 24 处 `mock://` 引用但无文件 → 二选一：随仓库提交自绘占位图，或去引用只留纯色块（需拍板） |
| 书名 | 1 本书名含知名作品名片段（3 处引用）→ 改名 |
| 旧设备数据导入 | 不承诺；只保证键名/列名等价，后续可加迁移器（需拍板） |

## 6. 测试资产的迁移

| 类别 | 旧数量 | 新目标 | 说明 |
|---|---:|---:|---|
| 纯 JVM 单测 | 447（62 文件） | 440–450 | 1:1 改写为 `flutter test test/unit` |
| Robolectric 可重设计 | ≈30（共 62） | 30–35 | 几何/可见性抽纯函数或 widget 断言 |
| Robolectric 专属丢弃 | ≈32（共 62） | 0 | 必须写"丢弃理由 + 替代断言"，不得静默删除 |
| androidTest | 4 | 4–8 | 改为 `integration_test`；无设备时标 `unverified` |
| **合计** | **513** | **474–493** | 另新增金标/长章/混排/Web 镜像等用例 |

## 7. Kotlin 模块 → Dart 目标与工作量

| 模块 | 规模（含空行） | Dart 目标 | 处置 | 人日 |
|---|---:|---|---|---:|
| app | 103 | `lib/main.dart`、`bootstrap/` | 重写 | 1.5 |
| app-shell | 1,460 | `lib/app/`、`lib/router/` | 重写 | 22 |
| core:model | 505 | `lib/domain/model/` | 转写 | 2.5 |
| core:util | 679 | `lib/core/util/`、`lib/reader/{layout,progress,nav}` | 转写算法 | 5 |
| core:net | 496 + 182 JSON | `lib/data/api/`、`assets/mock/` | 转写 + 数据复用 | 3 |
| core:data | 2,385 | `lib/data/{repo,db,kv,blobstore}` | 重写（Room→drift） | 20 |
| core:ui | 469 | `lib/design/`、`lib/widgets/` | 常量转写/组件新建 | 5 |
| feature:home | 3,829 | `lib/features/home/` | 重写 | 56 |
| feature:detail | 1,361 | `lib/features/detail/` | 重写 | 19 |
| feature:reader | 6,714 | `lib/reader/ui/`、`lib/features/reader/` | 引擎转写 + UI 重写 | 95 |
| feature:shelf | 1,158 | `lib/features/shelf/` | 重写 | 17 |
| feature:search | 1,896 | `lib/features/search/` | 重写 | 26 |
| feature:mine | 1,572 | `lib/features/mine/` | 重写 | 23 |
| Flutter 专有增量 | — | 路由深链、Web 文本镜像、字体/构建脚本 | 新增 | 11 |
| **合计** | **22,509 非空行** | — | — | **≈306** |

> 口径：含空行行数、1 人日=6 小时、工程师熟悉本仓库，不含测试重建（30–55）与视觉基线校准（30–50）。**306 是纯代码迁移的乐观下界**；计入 Web 专有（镜像/字体/SW/双渲染器）、justify 标定、测试重建、真机修复与 15% 缓冲后，独立质疑复核区间为 **420–640（P50≈500）**。该数字用于排期，不是承诺；阶段二 MVP 只占其中一小部分（见 `docs/09-roadmap.md`）。

## 8. 迁移时必须修复的既有缺陷（不得原样带入）

| # | 缺陷 | 修法 |
|---|---|---|
| 1 | 布局指纹不含字体度量/密度/标题块高/对齐开关 | 指纹纳入所有影响断行装页的字段 |
| 2 | 锚点恢复线性扫描 O(页×行) | 行首偏移有序数组 + 二分 |
| 3 | 页码/进度率在快照中冻结 | 持久化写入时的排版口径或进度率 |
| 4 | 图片页 `usedHeight` 恒 0、超高标题产出空页 | 显式边界与测试 |
| 5 | 无分页结果缓存 | `(chapterId, fingerprint)` LRU，按字数加权 |
| 6 | 书架 Flow 逐行查询（N+1） | JOIN 一次取回 |
| 7 | 内存无降级，页位图缓存按张不按字节 | 字节上限 + 生命周期清理 |
| 8 | 空章 `1/0` 空白 | 明确空态与文案 |
| 9 | 错误文案不区分网络/解析/分页 | 统一 `DataError` + 文案表 |
| 10 | 进度 tick 常驻耗电 | 事件驱动 + 页面不可见时停表 |
| 11 | 测试与实现口径分裂（15 行 vs 14 行、2190 vs 2187） | 单一金标口径，旧断言重写 |
| 12 | mock manifest 无校验和 | v2：文件清单 + sha256 |
| 13 | 素材悬空引用 | 自绘资源随仓库提交或去引用 |
| 14 | 夜间空态按钮对比度 | token 级对比度校验 |
