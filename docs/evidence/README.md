# docs/evidence · 测试证据与基线规则

> 本目录保存可复现的测试/测量证据。没有证据的结论不得进入验收。
> 大文件（截图、日志、APK）按 `.gitignore` 规则不入库，只入库指标 JSON 与说明；原始大文件保存在本地/对象存储并在 manifest 记录 sha256。

## 1. 目录

```
docs/evidence/
  README.md                     # 本文件
  baselines/                    # golden 图与 metrics JSON（authority 机器生成）
  run/<YYYYMMDD-HHMM>-<stage>/  # 每次运行一个目录
```

## 2. 一次运行必须包含

| 文件 | 内容 |
|---|---|
| `manifest.json` | git commit、Flutter/Dart/engine 版本、字体 sha256、设备/浏览器、产物 sha256 |
| `junit/test-*.xml` | 显式 UTF-8 编码 |
| `metrics/*.json` | 分页、首载、体积、帧率等指标 |
| `shots/<platform>-<page>-<state>-<variant>.png` | 画面语义命名（不是意图命名） |
| `logs/*.log` | 构建/测试/E2E 原始日志 |
| `REPORT.md` | 结论表、命令、输出摘要、未验证清单、与上期口径差异 |

## 3. 口径规则

- 每条指标必须写测量口径：纯算法测试 / 桌面浏览器 / Android 构建 / 模拟器 / 真机。
- 不同口径不得互相替代；模拟器数值只用于相对回归。
- `unverified` 单独成列，不计入通过率。
- 阈值变更属于判据变更，必须记录并取得用户签字。
- golden 的 authority：本机 Windows + Flutter 3.50.1 + engine `4d35714800` + 指定字体 sha；任一变化即整体重建基线。

## 4. 命名与追溯

- 用例 `TC-<域>-<n>`；质疑 `Q-<n>`；缺陷 `D-<域>-<n>`；证据 `EV-<runid>-<n>`；验收 `ACC-<域>-<n>`。
- 要求 `缺陷 → 用例 → 证据 → 修复提交` 双向可查；任一环缺失，缺陷不能关闭。

## 5. 禁止

- 只放结论截图、不记录命令与版本；
- 用开发机自验截图冒充独立验证；
- 把 `SKIPPED` / `unverified` 折算为通过；
- 修改历史运行目录（追加不覆盖）。
