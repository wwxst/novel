# 06 · 数据层、Mock 契约与后端预留

> 原则：业务逻辑不散落在页面；UI 只依赖仓储接口；当前没有服务端，**不得伪造账号/同步能力**。

## 1. 数据模型（字段级以代码为准，此处列关键结构）

| 模型 | 关键字段 | 来源/决策 |
|---|---|---|
| Book | bookId(TEXT), name, shortName, author, authorId, abstraction, category, tags[], serialCount, lastSerialCount, isFinish, wordNumber, score(String), readCount, coverUrl, smallCoverUrl, colorDominate, lastChapterTitle, lastChapterUpdateTime, platform, source, isEbook, updatedAt | 既有 4 个纯 Kotlin 模型文件；合并两套定义 |
| CategorySchema | categoryId, name, routeSchema | 保留 |
| BookDetail | book, categorySchema[], catalogSummary | 保留 |
| Catalog / ChapterMeta | bookId, totalChapters, chapters[]{chapterId, chapterIndex, title, wordCount, updateTime} | 保留；正文不入库 |
| ChapterContent | chapterId, bookId, chapterIndex, title, blocks[] | 保留 |
| ContentBlock | Title / Paragraph / Quote / Image(src, alt, licenseNote) | 保留 4 型；Image 增许可注记 |
| BookshelfItem | bookId, addType, addTimeMs, isPinned, pinnedTime, isAsterisked, isChasedUpdates, booklistId, sortIndex, lastReadTime | 保留；时间统一 ms |
| ReadingHistory | bookId, bookName, authorName, coverUrl, bookType, resource, pageInfoJson, readTime, isDelete | 保留冗余字段 |
| ReadingProgress | bookId, chapterId, chapterIndex, pageIndex, paragraphIndex, lineInParagraphOffset, charOffsetInParagraph, charOffsetInChapter, bookPageProgress, chapterProgressRate, layoutFingerprint, bookType, genreType, updateState, msTimestamp, deviceId(新增) | 锚点列是数据资产，迁移不得丢 |
| ChapterProgress | bookId, chapterId, chapterProgressRate, isFinished, updatedAt | 保留 |
| SearchHistory | keyword, updateTime, hitCount（上限 20） | 保留 |
| RankTab/RankItem/RankList | tabId/name；rank/bookId/heat/reviewTag/score/book；page/pageSize/pages | 保留 |
| ReaderSettings | fontSizeDp(24), fontName, lineSpacingMode(1), pageMarginMode(1), localLineSpace, localParagraphSpace, themeId(2), dayThemeId(2), darkThemeId(5), intelligentTheme(true), pageTurnMode(2), volumeKeyPageTurn, oneHandTurnPage, lockScreenTime, progressType | 保留；含取值域校验 |
| AppSettings | skinMode, fontScaleLevel, bootGateDone, dataSource("mock"), mockScenario, apiBaseUrl("") | 保留 + 切换点 |
| 排版派生参数 | 19 档字号、行 pitch 表、段距表、边距表 | 与阅读器共享，单一源 |

## 2. 仓储接口（9 + 1 预留）

| 接口 | 关键方法 |
|---|---|
| BookRepository | `list(FeedKind)` / `detail(bookId)` / `rankTabs()` / `rank(tabId)` / `cached(bookId)` / `cachedMany(ids)` |
| ChapterRepository | `catalog(bookId,{force})` / `chapter(bookId,chapterId,{force})` / `clearBookCache` / `cacheSizeBytes` / `evictLruIfNeeded` |
| ShelfRepository | `watchShelf()` / `shelf()` / `add` / `remove` / `contains` / `setPinned` / `touchLastRead` / `history` / `recordHistory` / `clearHistory` |
| ProgressRepository | `watchCurrent()` / `load` / `update` / `flush` / `flushIfDue` / `saveNow` / `chapterProgress(es)` / `lastChapterId` / `startAutoFlush(20s)` |
| SearchRepository | `search` / `hotWords` / `history` / `saveKeyword` / `removeKeyword` / `clearHistory` |
| ReaderSettingsStore | `watch()` + 各字段 setter（非法值返回 false） |
| AppSettingsStore | skinMode / fontScaleLevel / bootGateDone / dataSource |
| FollowRepository | isFollowing / setFollowing / followedBookIds（≤200） |
| ReadingTimeRepository | addSeconds / todaySeconds |
| 同步（不实现） | 只在本文 §7 记录契约；客户端**不写** `ProgressSyncRepository` / `sync_outbox` 代码与表 |

错误模型与文案表见 `docs/03-architecture.md §6`。

## 3. Mock 契约（8 接口 · v1）

| # | asset 路径 | 返回 | 规模 |
|---|---|---|---|
| 1 | `books/index.json` | Book[] | 12 本；书架投影前 5 |
| 2 | `books/<bookId>.json` | BookDetail | 单本 |
| 3 | `chapters/<bookId>/index.json` | Catalog | 30/12/6/2/0 章分档 |
| 4 | `chapters/<bookId>/<chapterId>.json` | ChapterContent | 每章 800–2500 字 |
| 5 | `ranks/tabs.json` | RankTab[] | 5 tab |
| 6 | `ranks/<tabId>.json` | RankList | pageSize 8，2 页 |
| 7 | `search/index.json` + `books/index.json` | SearchResult | 内存倒排 + 字段兜底 |
| 8 | `search/hot_words.json` | String[] | 20 条 |

**升级 manifest v2**（必须）：`version / schema_version / files[]{path,sha256,bytes} / license / source_note`；启动时 debug 校验，CI 全量校验；替代现有"字符串包含"式完整性测试。

## 4. 本地存储选型

| 候选 | Web 支持 | 查询能力 | 迁移成本 | 裁决 |
|---|---|---|---|---|
| **drift 2.35.4** | web + wasm + dart3 | 完整 SQL/事务 | 低（Room 语义接近） | **主选**（待 Web 探针） |
| sembast | web + wasm + dart3 | NoSQL 手写索引 | 高 | 备选 |
| hive_ce | web + wasm + dart3 | 最弱 | 最高 | 仅作 KV |
| sqflite_common_ffi_web | 仅 web 驱动 | 无 ORM | 中 | 否决（三端不自洽） |
| shared_preferences | web + wasm + dart3 | KV | 低 | **KV 主选** |

**定稿门限**：Web 产物增量 ≤1.5 MB（gzip）且存储初始化 ≤300 ms；任一超标切 sembast。探针在阶段二第一天执行，结果回填本文。

表结构：`books / chapters / chapter_content_cache / bookshelf / reading_history / reading_progress / chapter_progress / search_history` + 预留 `sync_outbox / sync_state / feed_cache`。列名保持 snake_case，便于与既有 schema 对拍。

## 5. 设置项迁移（键名等价）

保留旧键名语义并加命名空间，例如：
`reader.font_dp_size → reader.font_dp_size`、`reader_lib_theme → reader.theme_id`、`key_page_space_setting → reader.page_margin_mode`（默认 **1**，以代码为准而非旧文档的 0）、`key_skin_mode → app.skin_mode`、`key_font_scale_level → app.font_scale_level`、`key_mock_scenario → debug.mock_scenario`。

新仓库只读写新键，**不写旧键回退分支**（旧参考包的 SharedPreferences/Room 位于其应用私有目录，新包不可达，回退永远不执行）。非法值回退默认。**不导入旧数据**（需用户确认，见 `docs/10-risks-and-decisions.md`）。

## 6. 进度、缓存与生命周期

| 项 | 规则 |
|---|---|
| 翻页 | 只更新内存 + 置脏，不落库 |
| 聚合 | 20,000ms 窗口；切章/退后台/退出阅读器强刷 |
| 落库 | 事务同时写 `reading_progress` + `chapter_progress` |
| 正文 TTL | 24h；过期文件保留但读路径作废 |
| 目录 TTL | 24h |
| 缓存上限 | 100MB，按最近访问 LRU |
| 失效 | 布局指纹变化 → 忽略 pageIndex，按段/字符锚点重定位 |
| 页位图 | 按字节上限（如 48MB）+ 生命周期清理，禁止按张数 |

## 7. 同步预留（只写契约，不写代码）

只在 `backend/openapi/novel-v1.yaml` 与本文定义：匿名设备令牌、书架 CRUD/批量同步、进度 upsert/批量。冲突策略：书架 LWW（`updatedAt`，相同取 deviceId 字典序）；进度 LWW + 同设备单调不回退；批量幂等键。

**客户端不预写** `sync_outbox` / `sync_state` 表、`ProgressSyncRepository` 或任何"已同步"状态；接服务端时再新增（避免永不执行的死代码）。UI 不出现同步状态。

## 8. 数据合规与素材处置

| 项 | 结论 |
|---|---|
| 外链/令牌/真实账号 | 全量扫描 182 份 JSON：0 命中 |
| 正文 | 4,410 个文本块、仅 51 种自写文本轮转，非第三方作品正文 |
| 书名/作者/评论/热词 | 自写虚构；1 本书名含知名作品名片段 → **改名**（3 处引用同改） |
| 封面/插图 | 24 处 `mock://` 引用、仓库无文件 → 提交自绘占位资源并登记许可，或去引用改纯色块 |
| manifest | 补许可证与来源声明 |
| CI 禁项 | 真实域名/URL、token/secret、真实账号字段、第三方正文 |

## 9. 数据层测试

| 层 | 手段 |
|---|---|
| 模型/纯逻辑 | 纯函数断言：默认值、19 档、20s 窗口、哈希确定性 |
| 仓储 | 内存库 + FakeCatalogApi：TTL、强制刷新、LRU、去重、事务 |
| SQL 契约 | DAO 直测：排序、`trimTo(20)`、聚合 |
| Mock 契约 | 复制 182 JSON 到测试夹具，按 manifest v2 校验 sha256 + schema |
| 迁移 | drift schema 快照 + 版本升级测试 |
| Web | `flutter test --platform chrome`（本机用 Edge 作为 `CHROME_EXECUTABLE`） |

## 10. 接入真实后端的切换点

1. `CatalogApi` 接口不变，新增 `RemoteApi` 实现；
2. `AppSettings.dataSource` 从 `mock` 切 `remote`（debug 开关）；
3. `apiBaseUrl` 由构建注入，默认空；
4. 契约版本写入请求头与 manifest；
5. 仓储层无需改动。

以上四层确保"换后端不改 UI"，但在后端真正实现前，**不得对外宣称已支持账号/同步/云书架**。
