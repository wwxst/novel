# 04 · 阅读器核心迁移与实现方案

> 阅读器是产品核心竞争力，也是迁移风险最高模块。原则：**断行/装页/锚点逻辑三端共用；渲染按平台只做薄适配；正确性优先于复用率。**
> 既有实现已验证的行为必须通过金标测试对拍；已确认缺陷不迁移。

## 1. 设计不变量（不可妥协）

1. **页码不是权威**：进度用 `(chapterId, blockIndex, charOffsetInBlock)` + 布局指纹；换字号/字体/屏幕后重新定位。
2. **行位置由自己摆**：`TextPainter` 只负责断行与逐行绘制；行 y = `linePitch × 行序 + 段距累加`，禁止用 `TextStyle.height`/strut 的行盒堆叠（SkParagraph 会把 52.5714dp 取整为 53dp，14 行累计 +6dp 会改变换页点）。
3. **字体必须先就绪**：Web 端在字体加载完成后再分页，否则得到错误行数与豆腐块；iOS/Android 使用打包字体。
4. **分页可取消、可重入**：设置变化/旋屏时旧任务作废，结果按 generation 丢弃。
5. **两端对齐必须实现且单独验收**：非段末行均摊字距，末行左对齐。既有 Flutter 原型**没有实现 justify**（`lib` 目录 0 处 letterSpacing），所以"像素级一致"的结论不覆盖该项。S2 必须先做 1 页 PoC：用 `TextStyle.letterSpacing` 逐行均摊，跑"右墨迹 ≤2px 且页数不变"双断言，通过后才允许进入阅读器 UI；预算单列 ≥6 人日。

## 2. 数据流与核心接口

```
chapter.json → ChapterContent.fromJson → List<ContentBlock>{Title|Paragraph|Quote|Image}
  → Paginator.buildComposedBlocks（合成标题块/缩进/块起始偏移）
    → LineBreaker.breakBlock(text, width, indent)      ← CharWidthMeasurer 注入
      → PageComposer.compose(blocks, isActive)          → PageSlice[]
        → RenderPlanner.plan()                          → RenderPage / RenderLine{top, indent, justify}
          → CustomPainter（逐行 TextPainter，行盒居中基线）
```

| 接口/类 | 职责 |
|---|---|
| `CharWidthMeasurer` | `double widthOf(int rune)`；CJK 约定 1em；`FakeCharWidthMeasurer(63)` 用于纯算法金标 |
| `LineBreaker` | 返回每行 `charStart/charEnd/lineIndex/width`；实现 A=`GreedyCjkBreaker`（纯 Dart），实现 B=`TextPainterLineBreaker`（Flutter） |
| `PageComposer` | 装页：标题块独占页顶、页首不加段距、跨段加 gap、超界换页、图片独占页、`isActive` 每行检查 |
| `ReaderLayoutSpec` | 由设置 + 视口 + DPR 计算全部几何与 `layoutFingerprint` |
| `ReaderPaginator` | 章节 → `PaginatedChapter`（含分页缓存键） |
| `RenderPlanner` | 页 → 逐行绘制指令（含 justify 位移、基线偏移） |
| `PositionRestorer` | 锚点恢复：指纹快路径 → 段/字符 → 章内字符 → 章首 → 页 0 |
| `ProgressMapper` | 页 ↔ 进度快照；800ms 防抖 + 20s 聚合落库 |
| `ChapterNavigator` | 页内/跨章/书末三态 |

## 3. 纯算法层（零 Flutter 依赖，可快速测试）

`lib/reader/{model,layout,progress,nav}` 禁止 import `package:flutter` 与 `dart:ui`，由 `tool/arch_check.dart` 强制。

- 纯层文件：`chapter.dart`、`pagination_types.dart`、`typography.dart`（只算数值）、`cjk_chars.dart`、`greedy_cjk_breaker.dart`、`page_composer.dart`、`render_planner.dart`、`layout_fingerprint.dart`、`position_restorer.dart`、`chapter_navigator.dart`。
- 依赖 `TextPainter` 的文件：`text_painter_line_breaker.dart`、`ui/page_painter.dart`、`ui/pager_view.dart`、`web/text_mirror.dart`。
- 金标既可纯算法跑（fake measurer），也可 Flutter 环境跑（真实字体）；两套结果必须一致（当前纯 CJK fixture 已验证）。

## 4. 参数模型（数值唯一来源）

| 参数 | 取值 |
|---|---|
| 字号 | 19 档：13,15,17,18,19,20,21,22,23,24(默认),25,26,27,28,29,32,36,40,44 dp |
| 行距 | 4 档：pitch 113 / 129 / 138(默认) / 153 px@24dp；段距 41 / 49 / 50 / 63 |
| 边距 | 5 档：20 / 24(默认) / 28 / 32 / 智能=24 dp |
| 主题 | 7 档纸色 `#F6F6F6 #E8E3CE(默认) #E4F0D8 #D8E4F0 #0E0E0E #1A1A1A #262626`；主题切换不重排、不进指纹；夜间默认 id=5 |
| 翻页 | 5 模式：仿真 / 覆盖(默认) / 平移 / 上下 / 无动画 |
| 字体 | **v1：1 个打包字体子集（三端同一份）+ 系统字体**；黑体/宋体/楷体延后；切换打包字体=切换金标版本（记录字体 sha） |
| 首行缩进 | 2em（48dp@24）；引言块不缩进 |
| 章标题 | 1.10em / 上间距 27px / 距正文 90px / 块高 186px |
| 内容盒 | 954×2187px（363.43×833.14dp） |
| 非 24dp 档位 | 现行 `round(表值 × size/24 × density/2.625)` 为线性近似，**未实测**；阶段二须抽 6 组交叉实测替换 |

## 5. 锚点、重排与幂等

恢复顺序：

1. 指纹相同且 `pageIndex` 命中 → 快路径；
2. 否则按 `(blockIndex, charOffsetInBlock)` 在页集合定位；
3. 段落越界 → 回退章内字符偏移；
4. 仍失败 → 章首（**必须给可见提示**，避免"莫名回到开头"）；
5. 空排版 → 页 0。

指纹字段：字号、字体族 + 字体度量摘要、行距档、边距档、视口宽高、DPR、标题块高、对齐开关、内容版本。**不含 themeId。**

三条不变式（金标测试）：
- 恢复后的页必须包含锚点字符；
- `charOffsetInChapter` 跨重排稳定；
- 24→28→24 字号往返后页码回原页 ±1。

既有缺陷与修法：指纹缺字段（补）、恢复线性扫描（改二分）、快照进度率被冻结（持久化进度率与排版版本）、图片页高度恒 0、超高标题空页（显式失败 + 测试）。

## 6. 缓存、性能与长章节

| 项 | 方案 |
|---|---|
| 分页缓存 | **v1 不做**（实测整章 1.97–2.55ms，收益不足）；先修指纹 D1/D2。只有真机实测单章 >50ms 时才引入 `(chapterId, fingerprint)` LRU |
| 章节内容缓存 | 正文 TTL 24h、总 100MB LRU（沿用既有规则） |
| 页位图 | 按**可用内存 × 单页字节**动态定（当前页 + 预取 ±1 + 动画页），并设单页尺寸上限；禁止固定"3 张" |
| 离线 | 最近 N 章 + 已读章节缓存；断网可读已缓存章节，未缓存时显示明确离线提示（Tier B） |
| 分页执行 | 单一规则：单章 >20,000 字时按块分片并让出事件循环；否则一次分完。原"8 块/8ms"双阈值作废（实测 2ms 下永不触发）。isolate 可行性**未验证**，如要做先 30 分钟 PoC，不写进承诺 |
| 长章节 | 20k 字以下整章分页；超长章分片 + 进度反馈（实现与否需拍板） |
| 渲染 | 页位图离屏渲染，预取 ±1 页；翻页动画只操作位图，不在动画帧排版 |
| 预算 | 分页 1 章目标 ≤8ms、失败线 16ms（单一源 `tool/perf/budget.json`）；首帧（已缓存）≤300ms；章节切换 ≤400ms；页渲染 ≤8ms；翻页 ≥55fps（真机口径待测）；阅读器总内存以 PSS ≤220MB 为准，不用"增量 40MB"作为唯一口径 |

**测量口径必须分列**：纯算法测试（JIT）/ Web release（桌面 headless）/ Android 真机 / iOS 真机。当前只有前两种，后两种标 `unverified`。

## 7. 翻页、手势与滚动

| 模式 | 实现 |
|---|---|
| 无动画 | 换页 + `RepaintBoundary` |
| 覆盖（默认） | 自绘：旧页静止 + 新页平移；`AnimationController` |
| 平移 | 两页反向位移 |
| 上下 | 纵向位移；滚动阅读用 `ListView.builder`（每页一个 `CustomPaint`） |
| 仿真 | `clipPath` 折角 + 渐变阴影（首版近似，后续迭代） |

- 阈值沿用：位移 >18% 或速度 >600px/s；回弹 160ms；动画 250–350ms。
- 点击分区：左 28% 上一页 / 中 44% 菜单 / 右 28% 下一页（**待拍板**，见 §10）。
- 返回逐层关闭：评论 → 设置 → 目录 → 菜单 → 退出。
- 长按选词：移动端首版不做，登记为范围差异；Web 通过 DOM 镜像获得系统选择。

## 8. Web 文字选择、键盘与读屏（必须做，不能后置）

CanvasKit 正文不在 DOM，直接后果：不可选中/复制、`Ctrl+F` 搜不到、读屏空白、SEO 无正文。

**推荐方案 A：DOM 文本镜像层（镜像整章，而非仅可见页）**
- 在 canvas 上叠加绝对定位的逐行 `<span>`（`white-space: nowrap`、`color: transparent`），`top/left` 直接取分页输出的 `RenderLine.top/indent`；
- **镜像范围必须覆盖整章**（或至少当前章全部页），否则跨页 `Ctrl+F` 找不到、浏览器查找高亮会与 canvas 错位；
- 必须定义 `DOM character offset ↔ (blockIndex, charOffset)` 映射，把选区起止转成锚点（供复制/划线/笔记复用）；
- Canvas 仍是唯一视觉真相；镜像层只负责选择/复制/查找/读屏语义；
- 验收：① 镜像 span 的 top/left 与 `RenderLine` 差 ≤0.5dp；② 跨两页拖选 → 复制文本与章节原文逐字一致（0 差异）；③ 在第 3 页 `Ctrl+F` 搜第 1 页的词能命中并定位；④ 翻页/重排时镜像同步更新；
- 成本 3–5 人日（含映射与跨页选择），是分享/划线/笔记/读屏的地基。

**字体加载失败降级**：显式提示「字体不可用，已使用系统字体，排版可能与标准不一致」，不允许无限 loading 或豆腐块。

备选：自研 hit-test 选择（5–8 人日）——不解决 `Ctrl+F`/读屏，只建议用于将来的划线标注。

键盘：`←/→`、`PgUp/PgDn`、`Space` 翻页，`Esc` 关层，`Ctrl+F` 交给浏览器查找镜像层；正文需获得 primary focus。

Web 细节：
- 地址栏收放/视口高度变化不得触发重排；只在**宽度档**变化时重排（分页结果与视口解耦 + 防抖）。
- 滚轮翻页 300–500ms 节流，避免一次滑动翻多页。
- 读屏只保留一条 aria 通道（DOM 镜像或 Flutter 语义二选一），避免双重播报。
- 字体整体加载失败 → 回退系统字体 + 明确提示"排版可能与标准不一致"，不允许无限 loading/豆腐块。

可接受降级（需拍板）：首版整页文本可选 + 复制 + 键盘翻页 + 按段播报；不做跨页拖选与自定义选择手柄（移动浏览器仍用系统默认）。

## 9. 测试与金标

| 断言 | 期望值 |
|---|---|
| 页数 | 8 |
| 正文行数 | 94 |
| 每页行数 | `[10,0,14,14,14,14,14,14]` |
| 每页首行字数 | `[13,0,13,4,8,5,15,13]` |
| 8 个页首锚点 | `(null,0) (4,0) (5,0) (9,28) (14,28) (19,28) (24,13) (29,0)` |
| 逐块逐行字数 | 与金标 JSON 全等（94 行） |
| 行 pitch | `52.5714dp ±0.001` |
| usedHeight | `[653.83,0,812.19,831.24,831.24,831.24,812.19,812.19]` dp |
| 像素（仅真渲染） | pitch 中位 138px ±1；ink bbox ≤2–3px |

必须新增的 fixture：空章、5 万字长章、0.85/1.00/1.15/1.30/1.50 字号、混合中英数字、全角标点与避头尾、不同宽高比图片、两套真字体；另需"书末/章末/空章"三种边界文案与可达路径（UI 可达的测试书），锚点越界时必须给可见提示而不是静默回章首。

## 10. 待拍板事项

1. Web 文字选择：A（DOM 镜像整章，推荐）还是 C（自研选择）；是否接受首版"整页可选、无跨页拖选手柄"（**跨页 Ctrl+F 必须支持，不算降级**）。
2. 阅读器单击语义：保留 28/44/28 分区，还是"单击=菜单"（影响 iOS 边缘手势；推荐后者）。
3. 页脚页码：章内页（现状）还是全书页序（需全书预分页，成本高）。
4. 金标唯一口径：确认 8 页/94 行/186px/2187px，并授权重写旧的 2190/203/15 行断言；金标绑定字体 sha。
5. 超长章分块（>20k 字）是否排入本期；分页缓存是否等真机 >50ms 证据再议。
6. 划词/划线/笔记/段评：做最小集（复制+划线）还是用户书面接受范围外（accepted-risk）。
