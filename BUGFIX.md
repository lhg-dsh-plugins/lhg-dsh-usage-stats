# BUGFIX

## 标题

OpenCode Go 用量窗口的百分比被误当作 0..1 比例缩放：真实已用 ≤1% 的窗口被报成已用 100%、剩余 0%（`lib/subscriptions.js` 的 `goWindowFromObject`）。

版本：上游 0.3.6 → 本版 0.3.7（patch +1）。

## 现象与根因

`goWindowFromObject()`（OpenCode Go 的 API 路径 `parseOpenCodeGoApi` 与 dashboard 路径 `parseOpenCodeGo` 共用）在取到百分比后有一段缩放：

```js
// The dashboard embeds usagePercent as a 0..1 ratio. ...
if (usedPercent <= 1 && usedPercent >= 0 && value.percent === void 0 && percentSource !== void 0) usedPercent *= 100;
```

即：凡是百分比来自 `usagePercent` 等非 `percent` 字段、且数值落在 (0, 1] 的窗口，都被乘以 100。但这个前提是错的：

- OpenCode 官方 console 的 `analyzeRollingUsage` / `analyzeWeeklyUsage`（packages/console/core/src/subscription.ts）计算的是 `usagePercent = Math.floor(Math.min(100, usage / limit * 100))`——**0..100 的整数百分比**，页面上也直接以 `{usagePercent}%` 渲染。dashboard 的 `usagePercent` 从来不是 0..1 比例。
- Bearer API 的 `percent` 同样是 0..100；仓库自己的测试也断言「API percentages below 1 are not ratios」（`percent: 0.5` 保持 0.5）。同一约定理应适用于 `usagePercent`。

后果：一个几乎空闲的 Go 账户，五小时窗口真实已用 1% 时，卡片显示该窗口**已用 100%、剩余 0%**（并连带把订阅告警/折叠徽标的最低剩余拉到 0）。已用 >1% 的窗口不受影响，所以该缺陷只在低用量时出现，不易被察觉。

## 证据链

1. 最小复现（离线，假 fetch 返回与仓库自身 dashboard fixture 同形状的 JSON：rolling 1% / weekly 0% / monthly 2%）：
   - 修复前输出：`windows: [["session",100,0],["weekly",0,100],["monthly",2,98]]`，脚本 exit 1 —— 1% 被报成 100。
   - 修复后输出：`windows: [["session",1,99],["weekly",0,100],["monthly",2,98]]`，exit 0。
2. 回归测试互锁：在 `scripts/test-subscriptions.mjs` 新增两个用例（dashboard 形状、API `usagePercent` 形状）。仅回退 `lib/subscriptions.js` 时套件失败，actual 为 `[['session', 100, 0], ...]` vs expected `[['session', 1, 99], ...]`；带修复时通过。
3. 外部 ground truth：OpenCode console 源码 `usagePercent: Math.floor(Math.min(100, (usage / limit) * 100))`，且模板以 `{usagePercent}%` 直接显示。

## 改动文件

- `lib/subscriptions.js`：删除 `goWindowFromObject` 的比例缩放分支，注释订正为「百分比字段均为 0..100，(0, 1] 是真实的不足 1% 用量」。这是唯一的逻辑改动。
- `scripts/test-subscriptions.mjs`：新增上述两个回归用例（+48 行）。
- 版本同步（仓库原生 `npm run release:sync -- 0.3.7` + 发行说明）：`package.json`（version、files 增加 release-notes 条目）、`package-lock.json`、`catalog/v1/plugins.json`、`README.md` 版本标记均同步为 0.3.7；新增 `docs/release-notes-v0.3.7.md`。
- `package.json` 的 `repository.url` 按 fork 落点改为 https://github.com/lhg-dsh-plugins/lhg-dsh-usage-stats ；README 顶部追加中文 fork 头（上游原文一字未删）；新增 `UPSTREAM.md`、本文件。

## 复验结果

- 复现脚本：修复前 exit 1（100/0），修复后 exit 0（1/99）。
- `npm run test:subscriptions`：通过（含两个新回归用例）。
- `npm test` 全量 13 个套件（bundle/client/overlay/server/export/provider-identity/orcarouter/pricing/billing/balance/subscriptions/accounts/codearts-credits/install）：**全部通过**，与修复前基线一致（基线同样全绿；基线运行期间 `test:accounts`/`test:balance` 曾因沙箱高负载下子进程 5 秒 spawnSync 超时偶发失败，空载复跑均通过，与本次改动无关，改动后全量一次通过）。
- `npm run check` 的全文件 `node --check` 语法链通过；`check:release` 仅在「repository metadata drifted」一处断言失败——该断言是上游用硬编码仓库地址做的身份守卫，fork 按落点改 repository 后必然触发，属预期的 fork 身份差异；其余版本一致性（package/lock/catalog/README 版本、release notes 存在性）已逐项核对同步。

## 验证边界

- 只修了百分比缩放这一处；未触碰窗口发现、凭据解析、重置时间等其他逻辑。
- GitHub issue #148（徽标/告警取所有窗口最低剩余，用户希望按月度窗口）经核查是仓库**有意且有测试锁定**的设计（client.js 注释与 smoke 测试明确写明 "lowest remaining percent wins"），不属于代码缺陷，本版未改动。
- issue #140（OpenCode V2 凭据存储变化导致显示未配置）仓库文档 `docs/opencode-usage-sources.md` 已声明为已知限制（不扫描 V2 凭据库），本版未处理。
- 未使用真实 OpenCode Go 账户联网验收；dashboard/API 两种形状均以与仓库 fixture 同构的离线假响应验证。
