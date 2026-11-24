# backend/ · 服务端预留位置

> 状态：**只有契约与占位，当前不实现任何服务端业务。**
> 客户端的 Mock 数据是本地资产，不依赖本目录；在服务端可用之前，产品不提供账号、云书架、跨端同步。

## 1. 目录约定

```
backend/
├─ README.md                  # 本文件
├─ openapi/novel-v1.yaml      # 唯一契约源（待阶段四编写）
├─ migrations/                # 数据库迁移脚本（以契约为准）
├─ fixtures/                  # 与客户端 mock 同结构的样本数据
├─ .env.example               # 只有键名，没有值
└─ src/                       # 未来服务端代码（当前为空）
```

## 2. 契约面（计划）

| 能力 | 方法 | 路径 | 说明 |
|---|---|---|---|
| 匿名设备会话 | POST | `/v1/auth/anonymous` | 当前客户端不调用 |
| 书架读取 | GET | `/v1/shelf` | |
| 书架写入 | PUT | `/v1/shelf/{bookId}` | LWW（updatedAt，相同取 deviceId 字典序） |
| 书架删除 | DELETE | `/v1/shelf/{bookId}` | 墓碑 + updatedAt |
| 书架批量同步 | POST | `/v1/shelf/sync` | 幂等键 |
| 进度读取 | GET | `/v1/progress/{bookId}` | |
| 进度写入 | PUT | `/v1/progress/{bookId}` | LWW + 同设备单调不回退 |
| 进度批量 | POST | `/v1/progress/batch` | 逐条 LWW，冲突返回 409 明细 |

统一错误体 `{code,message,retryable}`；`code ∈ {UNAUTHORIZED,NOT_FOUND,CONFLICT,RATE_LIMITED,INTERNAL}`。

## 3. 边界

- 不提交真实域名、密钥、证书、账号数据或生产配置；`.env.example` 只有键名。
- 不在服务端保存未经授权的第三方正文或素材。
- 服务端技术栈未定（Dart / Node / Go 均可）；契约先行，实现后置。
- 客户端通过 `CatalogApi` 接口与 `AppSettings.dataSource` 切换，不直接依赖本目录。

## 4. 何时开始实现

阶段四、且用户明确批准账号/同步子项后。开始条件：契约评审通过、鉴权与隐私方案明确、部署与成本可接受。
