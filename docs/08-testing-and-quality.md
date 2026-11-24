# 08 · 测试体系、验收门槛与证据链

> 原则：**没有证据不得宣布完成；未验证是一等公民；不许把 `SKIPPED/unverified` 折算成通过。**
> 本机测试口径已实测：513 个 `@Test` = 509 可执行 + 4 从未执行；176 个 JUnit XML / 937 条记录 / 0 failed；937 是记录数，不是用例数。

## 1. 测试分层与工具

| 层 | 工具/命令 | 覆盖 | 限制 |
|---|---|---|---|
| L1 纯 Dart 单元 | `flutter test test/unit` | 断行、装页、锚点、模型、token、进度聚合 | 默认 FlutterTest 方块字体：分页数可信，字形宽度不可信 |
| L2 Widget | `flutter test test/widget` | 组件树、交互、语义、可见性 | 不验证真机排版 |
| L3 Golden（像素） | `flutter test test/golden`；`--update-goldens` | 组件像素回归 | 仅 authority 机器（字体/引擎/OS 锁定） |
| L4 集成 | `flutter test integration_test -d <device>` | 端到端链路、返回栈、持久化 | 无设备时只能 `unverified` |
| L5 Patrol（可选） | `patrol test` | 系统对话框、权限、后台 | 版本兼容未验证，暂不纳入 |
| L6 Web E2E | Playwright（Edge/Chromium） | 路由、刷新、返回、键盘、分页 dump、截图 | 无 Safari/WebKit，不可宣称三浏览器通过 |
| L7 截图/度量 | `tools/imgdiff.py`、`measure_screenshot.py`、`compare_metrics.mjs` | 与设计基线对照、四端度量矩阵 | 阈值分档，整页差不作通过判据 |
| L8 性能 | Playwright 首载、Web 体积、AOT APK、帧追踪 | 见 `07-platform-and-performance.md` | 口径必须标注 |

**Web 金标的硬约束**：
1. 断言前必须加载显式打包字体并等待就绪，禁止依赖 CanvasKit 自动回退；
2. Web 只做**度量金标**（页数/行数/pitch/锚点），像素 golden 只在 authority 机器做；
3. authority 固定为「本机 Windows + Flutter 3.50.1 + engine `4d35714800` + 指定字体 sha」，任一变更即整体重建基线。

## 2. 旧测试迁移（513 条）

| 类别 | 判定 | 旧数量 | 新目标 |
|---|---|---:|---:|
| A 可直接改写 | 无 `import android/androidx` 的测试（455 条 / 65 文件）；若按"无 Android runner"则为 447 条 / 62 文件 | 455 或 447（谓词不同） | 440–455 |
| B 需重设计 | Robolectric 但断言是几何/测量/可见性 | ≈30 | 30–35（纯函数或 widget 断言） |
| C 丢弃并替代 | Room DAO 真 SQL、资源/主题、View 测量 | ≈32 | 0 直接迁移，必须有替代断言 |
| D 仪器化 | `AndroidJUnit4` | 4 | 4–8（integration_test，无设备标 unverified） |
| **合计** | | **513** | **474–493** |

每个 C 类必须写「丢弃理由 + 替代断言」到 `docs/evidence/test-migration.md`。**禁止静默删除用例。**

## 3. 金标 fixture 与断言

| # | fixture | 状态 |
|---|---|---|
| F1 | 金标章（34 block / 1046 字符 / 32 段 + 1 引言 + 1 图片） | 照搬，sha256 固定 |
| F2 | 图片页（1 处 image） | 补 2 张不同宽高比 |
| F3 | 空章（`blocks:[]`） | 新建，锁"本章暂无内容" |
| F4 | 长章（5 万字） | 新建，锁 100+ 页与耗时上限 |
| F5 | 跨字号 0.85 / 1.00 / 1.15 / 1.30 / 1.50 | 新建 |
| F6 | 混合中英数字 | 新建（只能用真字体断言） |
| F7 | 全角标点 + 避头尾 | 新建 |
| F8 | 页脚/顶栏几何 | 沿用 |

F1 断言（唯一权威）：

```dart
expect(pages.length, 8);
expect(pages.map((p) => p.lines.length).toList(), [10,0,14,14,14,14,14,14]);
expect(totalLines, 94);
expect(pages.map((p) => p.firstLineCharCount).toList(), [13,0,13,4,8,5,15,13]);
expect(pages.map((p) => p.topAnchor).toList(), GOLD_ANCHORS);
expect(linePitchDp, closeTo(52.5714, 1e-3));
expect(contentHeightDp, closeTo(2187/2.625, 0.01));
// 仅真渲染通道：
expect(medianLinePitchPx, closeTo(138, 1));
expect(inkBBoxDeltaPx, lessThanOrEqualTo(2));
```

## 4. 四端一致性判定

| 判据 | 纯 Dart | Android | iOS | Web | 判定 |
|---|---|---|---|---|---|
| 页数/行数/每页行数/首行字数/页首锚点 | 可比 | 可比 | 可比 | 可比 | **必须全同** |
| pitch/内容盒/段距 | 可比 | 可比 | 可比 | 可比 | 差 ≤0.01dp |
| 截图 pitch | — | ≤±1px | ≤±1px | ≤±1px | 各端对 138px |
| ink bbox | — | 参照 | ≤±3px | ≤±3px | 各端对参照 ≤3px |
| 整页 `changed_gt8` | — | — | — | — | **不作通过判据**，仅趋势 |

## 5. 三端 E2E 与 CI

| 端 | 现在可做 | 命令 | 不可做 |
|---|---|---|---|
| Android | `flutter analyze` + `flutter build apk --debug`（已实测成功） | 构建、体积 | 安装/点击/截图/真机排版 |
| Web | Edge/Chromium 全链路 | `flutter build web --release --wasm`；Playwright | Safari/WebKit |
| iOS | `flutter analyze` | — | 全部运行时 |

CI（GitHub 不可达）：
- 本地/自托管 runner：`flutter pub get --offline` → `flutter analyze` → `flutter test` → `flutter build web` → 体积/首载门禁；
- 依赖缓存：`.pub-cache`、`.dart_tool`、固定 Flutter SDK；`pubspec` 全部使用 pub.dev 版本（禁止 git 依赖）；
- golden 只在 authority 机器生成；
- 输出：JUnit XML（显式 UTF-8）、metrics JSON、截图、`REPORT.md`。**注意**：PowerShell 5.1 读 UTF-8 XML 会按 ANSI 解码丢用例，必须显式 UTF-8 读写。

## 6. 完成定义与阶段门槛

### DoD（每个功能项）
1. 功能可用，点击产生真实状态迁移（禁止假实现）。
2. 数据真实变化（DB/文件/prefs 前后有命令证据）。
3. 空/错/超时/重试/离线可达；loading 可见下限 ≥400ms；文案不截断。
4. `flutter test` 全绿；新增用例覆盖被改行为（≥被改模块旧用例数 ×0.8）。
5. 三端矩阵逐项 `PASS/FAIL/unverified`；unverified 不计通过。
6. 质疑闭环（无 open）；P0/P1 双签、accepted-risk 仅用户。

### 阶段门槛

| 阶段 | 出口 |
|---|---|
| 一 方案 | 架构/迁移/测试/计划/风险/质疑闭环；工具链 Gate 0 通过；用户批准 |
| 二 内核链路 | 纯算法 + 金标 8 页 94 行三列全同；Web 可运行；Android 可构建；核心链路真实数据流 |
| 三 阅读器完整 | 5 种翻页、设置重排、锚点恢复、目录、菜单、夜间、图片；真机或 unverified |
| 四 全站页面 | 11 屏 + 设置逐模块 DoD；三端冒烟清单 |
| 五 硬化 | P0=0/P1=0；P2 有状态；C/D 清单用户确认；真机性能重测 |

## 7. 证据留存结构

```
docs/evidence/
  README.md                 # 命名/口径/authority 声明
  baselines/                # golden PNG + metrics JSON
  run/<YYYYMMDD-HHMM>-<stage>/
    manifest.json           # commit、Flutter/engine 版本、设备、产物 sha256
    junit/test-*.xml
    shots/<platform>-<page>-<state>-<variant>.png + .json
    metrics/*.json
    logs/*.log
    REPORT.md               # 结论表 + 命令 + 未验证清单
```

- 截图命名必须描述**画面语义**，不能只写意图；
- 缺陷单：ID、级别、域、现象、可粘贴复现步骤、证据（帧 sha + 量取值）、修复前后对比、裁决、回归；
- 可追溯 ID：`TC-*` 用例、`Q-*` 质疑、`D-*` 缺陷、`EV-*` 证据、`ACC-*` 验收。

## 8. 缺陷分级与防作弊

| 级别 | 定义 |
|---|---|
| P0 | 崩溃/ANR/数据错误或丢失/核心链不可完成 |
| P1 | 主功能不可用或存在可见假实现 |
| P2 | 可用但明显偏离规格（几何/信息架构/密度） |
| P3 | 细节：文案、1–3px、边缘态、对比度 |

状态：`open / fixed / rejected(有证据) / accepted-risk(仅用户)`。

防作弊五条：
1. **判据先冻结**（写入 `judgement-<stage>.md`），事后改判据=变更请求，需用户签字；
2. `fixed` 必须附本轮新采集、可复跑证据；开发自验截图不能作为独立依据；
3. P0/P1 关闭需"实现者 + 非实现者"双签；
4. 每份报告必须写"与上期口径差异"；
5. `unverified/SKIPPED` 记为未完成，禁止折算为通过。

## 9. 与质疑专家的接口

质疑者只提可证伪问题，不给结论；测试/质量角色把问题翻译成**可执行断言 + 复现命令 + 判据数值**；独立裁决者按证据裁决。流程：质疑 → 断言化 → 执行 → 证据 → 裁决 → 修复 → 回归。`open` 不得进入下一阶段。

## 10. 风险

1. 像素 golden 对字体/引擎/OS 敏感，字体选型一旦确定变更即全红 → 以度量为主、像素为辅，基线版本化。
2. Robolectric 丢弃项可能丢掉真实回归 → 每条必须有替代断言。
3. 无设备/无 Mac 时三端验证容易"汇总成已验证" → 矩阵单列 `unverified`，质量角色签署。
4. GitHub 不可达 → CI 只能本机/自托管，golden 只认 authority 机器。

## 11. 独立质疑后的修订（v2）

### 11.1 判据分层（消除自相矛盾）
- **硬判据**：度量（页/行/每页行数/首行字数/锚点/usedHeight/右墨迹）+ 分区几何 bbox + 控件色值。
- **趋势指标**：整页 `changed_gt8` 只记录、不阻塞、不作通过依据。
- 判据在每阶段开始前写入 `docs/evidence/judgement-<stage>.md` 并冻结；修改需用户签字。
- golden authority 机器生成基线不构成独立验收；独立验收只能由未参与开发的角色用同一命令复算。

### 11.2 测试基数与阶段承诺
- 实测口径已对账：**513 个 `@Test`**；无 `import android/androidx` 的测试为 **455 条 / 65 文件**；无 Robolectric/AndroidJUnit runner 的为 **447 条 / 62 文件**；差 8 条来自 3 个"有 runner 无 import"的文件。任何引用必须写清谓词，禁止混用。
- **阶段二只承诺** `core/util` 54 条 + `core/model` 13 条全绿；其余按模块验收，不得用"改写断言迁就实现"凑数。
- 总目标调整为 **380–430 条可用用例**（原 474–493 偏乐观），每条 C 类必须有替代断言。

### 11.3 unverified 出口规则
- 三端矩阵按 **15 行状态对象**（搜索拆入口/联想/结果 3 行，阅读器正文与菜单/目录/设置面板分行，含设置）× 7 态 × 3 端 = **315 格**；报告必须输出 `unverified_cells=NNN/315` 与逐格状态。
- 无 Android 设备/模拟器或 macOS 时，**阶段五不得判定通过**；设备到位后重开对应格。
- `unverified` 不计通过，也不能用"汇总表省略分母"的方式掩盖。

### 11.4 fixture 与可达性
- 新增一本 UI 可达的测试书：含空章、长章（>2 万字）、混合中英数字、书末、含生僻字章节；边界态必须能从界面操作到达，而不是只跑单测。
- 错误/超时态若只能靠 debug 注入，报告必须标注"注入态，非真实网络"。
- manifest v2 校验只在 CI 全量跑（182 文件 + sha256），启动期不做全量校验；书架列表补性能断言（100 本首帧 ≤100ms，桌面 Web 与真机两口径）。

### 11.5 Web 专项断言
- DOM 镜像：整章文本可选；跨两页拖选复制文本与原文逐字一致（0 差异）；第 3 页 `Ctrl+F` 能命中第 1 页词并定位；镜像 span 坐标与 `RenderLine` 差 ≤0.5dp。
- 字体失败降级：模拟字体加载失败时，页面给出明确提示且不出现豆腐块/无限 loading。
- 离线：断网刷新可进入已缓存章节（自建 SW 后）。
