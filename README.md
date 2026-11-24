# Novel · Flutter 三端小说阅读器

> 状态：**阶段一（方案评审中）**。本仓库当前只包含接管审计、架构方案与迁移计划，**尚未开始产品编码**。
> 方案经仓库所有者批准后，才进入阶段二实现。未经批准不新增产品代码与依赖。

## 1. 项目定位

一套 Flutter 代码同时构建 **Android / iOS / Web** 三端的小说阅读产品，面向个人长期维护：

- 一套 UI、状态管理、业务逻辑与阅读器核心，三端共享；平台差异只放在明确的适配层。
- 阅读器是核心资产：中文断行、分页、进度锚点、重排、翻页动画与长时间阅读稳定性优先于其他指标。
- 结构简单、边界清楚：不引入只为"看起来企业级"而存在的抽象。
- 产品目标是完整小说产品，不是演示页面集合；分阶段扩展，每个阶段有可验收的闭合链路。
- 仓库为未来服务端预留 `backend/` 位置（当前只有契约与说明，不实现服务端）。

**非目标（本阶段）**：重写既有 Android 参考工程、复制任何第三方正文/封面/评论素材、接入真实账号与支付、伪造"已同步/已上线"能力。

## 2. 技术基线（待方案批准后冻结）

| 项 | 选择 | 状态 |
|---|---|---|
| 框架 | Flutter 3.50.1 / Dart 3.14.0 | 本机已具备（`D:\Develop\tools\flutter-sdk`） |
| 状态管理 | Riverpod（手写 Notifier，首期不引代码生成） | 方案中，待批准 |
| 路由 | go_router（path URL、深链、tab 保活） | 方案中，待批准 |
| 本地存储 | drift + shared_preferences（Web 端先做体积/初始化探针） | 方案中，待批准 |
| 阅读器 | 自研排版引擎：逐段断行 → 固定行距装页 → 自绘页面 | 方案已详设 |
| 后端 | 仅 OpenAPI 契约与目录占位 | 未实现 |

依赖与版本以 `docs/03-architecture.md` 批准后的版本为准；本阶段不提交 `pubspec.yaml`。

## 3. 仓库结构（目标形态）

```
novel/
├─ lib/                 # Flutter 应用（单包，按目录分层）
│  ├─ bootstrap/        # 启动装配、启动门
│  ├─ app/              # 应用壳、主题、tab
│  ├─ router/           # 路由表与返回策略
│  ├─ design/           # 设计 token（生成物，禁改）
│  ├─ domain/           # 数据模型与仓储接口
│  ├─ data/             # Mock/远端 API、仓储实现、本地库、文件缓存
│  ├─ reader/           # 阅读器纯算法层 + UI
│  ├─ features/         # 各业务页
│  └─ platform/         # 平台适配接缝
├─ assets/mock/         # 自写测试数据（182 份 JSON）
├─ backend/             # 后端预留：契约 + 说明，当前不含实现
├─ design/tokens/       # token 单一源（JSON）
├─ docs/                # 方案、规格、证据、验收记录
├─ tools/               # 架构检查、数据校验、截图度量、token 生成
└─ test/                # 纯逻辑 / widget / golden / 集成测试
```

## 4. 文档索引

| 文档 | 内容 |
|---|---|
| [docs/00-takeover-and-status.md](docs/00-takeover-and-status.md) | 接管检查：环境实测、既有工程真实规模、旧报告口径修正 |
| [docs/01-scope-and-feature-inventory.md](docs/01-scope-and-feature-inventory.md) | 已完成页面清单、完整产品功能域与优先级、已知问题基线 |
| [docs/02-reusable-assets.md](docs/02-reusable-assets.md) | 可复用资产清单与 Kotlin→Dart 迁移映射 |
| [docs/03-architecture.md](docs/03-architecture.md) | Flutter 三端统一架构、状态/路由/数据层、后端预留 |
| [docs/04-reader-engine.md](docs/04-reader-engine.md) | 阅读器排版/分页/锚点/翻页/缓存/Web 选择方案 |
| [docs/05-ui-system.md](docs/05-ui-system.md) | 设计 token、组件体系、状态矩阵、三端响应式与交互 |
| [docs/06-data-and-backend.md](docs/06-data-and-backend.md) | 数据模型、Mock 契约、存储选型、同步预留与素材合规 |
| [docs/07-platform-and-performance.md](docs/07-platform-and-performance.md) | 三端性能预算、Web 优化、构建与验证边界 |
| [docs/08-testing-and-quality.md](docs/08-testing-and-quality.md) | 测试体系、金标、验收门槛、证据链 |
| [docs/09-roadmap.md](docs/09-roadmap.md) | 阶段计划、里程碑、并行与依赖 |
| [docs/10-risks-and-decisions.md](docs/10-risks-and-decisions.md) | 风险清单与需要用户拍板的事项 |
| [docs/11-adversarial-review.md](docs/11-adversarial-review.md) | 独立质疑记录、裁决与修订结果 |
| [docs/evidence/judgement-stage1.md](docs/evidence/judgement-stage1.md) | 阶段验收判据（冻结版，待签字） |
| [docs/evidence/test-migration.md](docs/evidence/test-migration.md) | 测试基数对账与逐类迁移处置 |

## 5. 开发环境要求

- Flutter 3.50.1 stable（Dart 3.14.0）、JDK 21、Android SDK（platform 35+）。
- Web：Edge/Chrome（本机无 Chrome 时用 `CHROME_EXECUTABLE` 指向 Edge）。
- iOS：需要 macOS + Xcode；当前 Windows 环境**无法验证 iOS**，所有 iOS 结论必须标记未验证。
- 网络：依赖 pub.dev；当前环境访问部分构建仓库受限，CI 计划见测试文档。

## 6. 合规与素材

- 代码与设计文档为本项目自研；不包含、不复制任何第三方应用的受版权保护正文、封面、评论或品牌素材。
- 测试数据（`assets/mock/`）由项目自行编写的生成脚本产出，正文为自写测试文本；封面使用纯色块占位或自绘素材，授权逐项登记。
- 字体只使用可再分发的开源字体（OFL 等），随附许可证；不使用来源不明的商业字库。
- 仓库内不提交任何真实服务地址、密钥、令牌、账号或个人信息。

## 7. 协作规则

- 任何结论必须附证据（文件/命令/截图），未验证项显式标记，禁止把"编译通过"写成"功能通过"。
- 重要模块必须经过独立质疑与独立裁决后才能标记完成，流程见 [AGENTS.md](AGENTS.md) 与 `docs/08-testing-and-quality.md`。
- 分支与提交：`main` 保护；功能分支 `feat/<模块>`；提交信息 `type(scope): summary`。
- 当前阶段：仅方案与文档变更；批准前不进行代码迁移。
