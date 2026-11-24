# 测试迁移基数与分类（阶段一对账）

> 目的：消除"447 / 455 / 513 / 509 / 937"多个数字的口径混乱；本文给出可复现的谓词与分类。
> 数据来源：既有参考工程 `implementation/android` 下 `src/test`、`src/testDebug`、`src/androidTest`（排除 `build/`）。

## 1. 谓词对账（必须带谓词引用）

| 谓词 | 文件 | 用例 | 说明 |
|---|---:|---:|---|
| `@Test` 静态标记 | 80 | **513** | 495 `src/test` + 14 `src/testDebug` + 4 `src/androidTest` |
| 无 `import android/androidx` | **65** | **455** | 含 3 个"有 Robolectric runner 但无 import"的文件（`CrossModuleProgressTest` 2、`ShellAssemblyTest` 3、`ReaderProgressBridgeTest` 3） |
| 无 Robolectric/AndroidJUnit runner | **62** | **447** | 纯 JVM 测试；即"可 1:1 改写"的主体 |
| Robolectric | 15 | 62 | 与 447/455 的差集关系见上 |
| AndroidJUnit4 | 3 | 4 | 从未执行（无设备） |
| JUnit XML 记录数 | 176 XML | 937 | 同一批用例在 debug/release 两个变体各跑一次；去重后 **509**（= 495 + 14），4 条 androidTest 无 XML |

**结论**：引用"447 条纯 JVM"时必须写"无 Robolectric/AndroidJUnit runner"；写"无 Android import"时应为 455 条 / 65 文件。禁止再用"714 / 634 / 937 全绿"作为用例数。

## 2. 分类统计（按模块）

| 模块 | 文件 | 用例 | A 纯 JVM | B/C Robolectric | D 仪器化 |
|---|---:|---:|---:|---:|---:|
| app | 2 | 5 | 0 | 5 | 0 |
| app-shell | 6 | 29 | 22 | 7 | 0 |
| core（model/util/net/data/ui） | 24 | 176 | 170 | 6 | 0 |
| feature:home | 9 | 54 | 50 | 4 | 0 |
| feature:detail | 6 | 31 | 31 | 0 | 0 |
| feature:reader | 29 | 143 | 99 | 40 | 4 |
| feature:shelf | 3 | 33 | 33 | 0 | 0 |
| feature:search | 3 | 23 | 23 | 0 | 0 |
| feature:mine | 2 | 19 | 19 | 0 | 0 |
| **合计** | **84** | **513** | **447** | **62** | **4** |

注：A 类 66 个文件中含 4 个 0 用例辅助文件（`Fakes.kt`、`FileMockAssetSource.kt`、`TestData.kt`、`ReaderTestFixtures.kt`），所以有效 A 类文件为 62。

## 3. B/C/D 类逐文件（必须逐条给出迁移或替代）

| 模块 | 文件 | 用例 | 类别 | 迁移处置 |
|---|---|---:|---|---|
| app | CrossModuleProgressTest | 2 | Robolectric | 跨模块进度纯逻辑 → Dart unit |
| app | ShellAssemblyTest | 3 | Robolectric | 装配语义 → Dart 依赖注入测试 |
| app-shell | AppSkinUiTest | 4 | Robolectric | 资源/主题 → token 断言 + widget test |
| app-shell | ContinueReadingBallMeasureTest | 3 | Robolectric | 测量 → widget 几何断言 |
| core:data | ReplicaDatabaseDaoTest | 6 | Robolectric | Room DAO → drift 内存库测试 |
| feature:home | HomeStateViewsTest | 4 | Robolectric | 状态视图 → widget test |
| feature:reader | AndroidLineMeasurerTest | 2 | Robolectric | TextPainter 测宽 → Dart 适配层测试 |
| feature:reader | PrefsReaderSettingsStoreTest | 4 | Robolectric | SharedPreferences → fake KV 测试 |
| feature:reader | ReaderAddToShelfTest | 4 | Robolectric | 写库链路 → 仓储 + 内存库测试 |
| feature:reader | ReaderBarsThemeTest | 3 | Robolectric | 主题颜色 → token 断言 |
| feature:reader | ReaderMenuActionsTest | 3 | Robolectric | 菜单动作 → 状态机测试 |
| feature:reader | ReaderMenuOpenStateTest | 3 | Robolectric | 面板可见性 → widget test |
| feature:reader | ReaderProgressBridgeTest | 3 | Robolectric | 进度接线 → 纯逻辑测试 |
| feature:reader | ReaderStatusOverlayTest | 13 | Robolectric | 状态层可见性 → widget + 语义测试 |
| feature:reader | ReaderTtsBallViewTest | 5 | Robolectric | 自绘球 → widget 几何测试 |
| feature:reader | ReaderChromeInstrumentedTest | 1 | AndroidJUnit4 | integration_test（无设备标 unverified） |
| feature:reader | ReaderGeometryInstrumentedTest | 2 | AndroidJUnit4 | integration_test（无设备标 unverified） |
| feature:reader | ReaderTextJustifyInstrumentedTest | 1 | AndroidJUnit4 | integration_test（无设备标 unverified） |

> 原则：B/C 类不得静默删除，每条必须有"替代断言"；D 类在无设备时产出 `unverified`，不计通过。

## 4. A 类迁移策略

- 62 个有效文件 / 447 条：按 1:1 语义改写为 `test/unit/**`，断言值照抄（金标、常量、状态机）。
- 优先级：`core:util`（54）+ `core:model`（13）→ 阅读器规则 → `core:data`（73）→ feature 契约。
- 阶段二只承诺 `core:util` + `core:model` 全绿；其余按模块验收，禁止用"改写断言迁就实现"。
- 逐文件 Dart 目标名与断言清单在 S0 由脚本生成 `docs/evidence/test-migration.csv` 并入库；本文给分类与处置。

## 5. 迁移目标

| 类别 | 旧数量 | 新目标 | 备注 |
|---|---:|---:|---|
| A 纯 JVM | 447 | 440–455 | 1:1 改写 |
| B/C 重设计 | 62 | 30–35 | 每条有替代断言 |
| D 仪器化 | 4 | 4–8 | integration_test |
| 合计 | 513 | **380–430 可用用例** | 另新增金标/边界/混排/Web 镜像用例 |
