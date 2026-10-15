# 开发进度

## 当前主线

维护 V2EX provider 隔离、配额等待、临时窗口、Smart Compact 补救与上游故障重试。代码已推送，默认关闭临时窗口与上游重试。接手先读最新条目，再读 README 的安装与命令说明，以及 Smart Compact fork 的接口文档。本机路径与操作约定见本地 `AGENTS.md`。

下方 2026-10-08 / 01 的未提交状态属于旧阶段，已由后续实现取代。

## 2026-10-08 / 01：V2EX 临时上下文窗口

状态：本地开发完成，基于 `main` 的 `3e7a26a`，本轮改动尚未提交和部署。

### 本轮完成

- 新增 `/v2ex compact window <tokens>`、`on`、`off` 和 `status`。
- 使用当前会话的模型副本调整窗口，关闭、切换模型和重载时恢复或重新应用。
- 开关与目标值持久化到扩展配置，默认关闭、目标窗口 400,000 tokens。
- 读取 pi 的实际预留量与逐模型覆盖，检查 Smart Compact 的安装及全局触发门槛。
- 压缩失败后暂停临时窗口、恢复原值并停止自动任务，手动压缩后可重新启用。

- 状态栏开关段新增临时窗口：开启时显示目标（`400K`），关闭、待启用、补救失败后暂停各有不同文案。
- `/v2ex compact status` 改为报出 Smart Compact 自动压缩的实际开关和最低占比，不再让人自己去翻设置面板。
- 启用、暂停、恢复及压缩失败时会立刻重画状态栏，不等下一次轮询。

### 验证与边界

- `node tools/verify-compact-window.mjs` 通过：真实 pi 和 Smart Compact 自动生成并应用摘要，模型及 pi 设置文件保持原样。包含关闭恢复、模型切换和重载。
- 模型响应使用固定 fixture，未调用 V2EX 上游。临时 agent 目录已由验收脚本清理。
- 全量测试首次运行有 54 项通过，两个测试文件因 Node strip-only 不支持参数属性而未加载。修正语法后，仅重跑这两个文件，28 项通过。共验证 82 项测试。
- `npm test` 在当前 Bash 环境遇到 npm 命令入口路径问题，因此使用等价的 Node 命令执行测试。
- Smart Compact 仍可能回退到 pi 原生摘要。扩展在收到失败事件后暂停，不能通过当前公开接口阻止第一次原生回退。
- Smart Compact 的分支级开关由该扩展自己管理，本扩展检查全局设置。手动 Apply 设置保持原有值。
- 历史记录曾实测 V2EX 单条消息上限为 160,000 字符。本轮未重复探测上限，不能据此断言上游目前仍完全相同。

### 下一步

该阶段已由下方无人值守补救实现取代。

## 2026-10-08 / 02：无人值守压缩恢复

状态：实现与验证完成。发布目标为 V2EX `main` 和 Smart Compact fork 的 `v2ex-recovery` 分支。

### 本轮完成

- 创建 `huaxianyan/pi-smart-compact` fork，补丁基于上游 `v9.7.1`，不混入升级到最新主线。
- Smart Compact 提供公开的 `extension-api` 协议模块，由唯一加载的扩展运行既有 EESV 管线。
- 自动压缩失败后结束本轮，空闲时请求 `fast` 补救，默认时限 300 秒；更低的全局预算仍生效。
- 补救应用必须匹配本次已验证摘要的 run ID，成功后自动继续原任务，无需 Apply 确认。
- 配额用尽等待刷新，网络或超时错误有界退避。恢复计划沿用已有落盘协议，绑定原会话、模型和任务分支。
- 取消、关闭、切换任务会取消进行中的补救。恢复检查最多 `1 + maxErrorRetries` 次，包含配额复核；终止时保留原记录、恢复原窗口。
- V2EX 从公开模块读取 Smart Compact 默认占比，不再复制外部默认常量。

### 验证记录

- V2EX 全量 Node 测试 89 项通过，含状态栏窗口段与 `compact status` 的开关文案。
- Smart Compact 定向检查共 8 项通过：接口成功及超时两项，现有工具回归六项。补齐测试 fixture 的事件总线后仅重跑此前失败的一项。
- 相关源码严格 TypeScript 检查通过，库目标与既有代码对齐为 ES2024。
- 真实 pi 集成通过：设置 `SMART_COMPACT_EXTENSION` 指向 fork 的 `dist/index.js`，执行 `node tools/verify-compact-window.mjs --recovery`。覆盖初次自动压缩失败、保存计划、配额等待、重载恢复、补救摘要应用和任务续跑。
- 模型及配额响应均为 fixture，未请求真实 V2EX。模型和 pi 设置文件保持原样，验收临时目录已清理。
- Git 安装使用 fork 中预构建的入口及协议模块，不直接修改已安装 npm 包。

### 安装来源与接续

- V2EX：`git:github.com/huaxianyan/pi-v2ex-quota`。
- Smart Compact：`git:github.com/huaxianyan/pi-smart-compact@v2ex-recovery`。移除旧 npm 来源，避免加载两个实例。
- 重启或 `/reload` 后，按需执行 `/v2ex compact window 400k`。默认仍关闭，不修改 `models.json`。
- 下一步只需在真实 V2EX 长会话中观察摘要质量及上游成功率。现有 fixture 验收不替代该检查。

## 2026-10-13 / 03：核对现有交接入口

- 本轮基线：`huaxianyan/pi-v2ex-quota` 的 `main`，提交 `96ab708`。状态栏窗口段与 Smart Compact 自动开关已进入代码基线。
- 沿用本文件，补 README 交接链接，明确旧阶段状态与当前主线，移出历史验证命令中的实际机器路径。
- 历史边界：89 项 Node 测试、类型检查和真实 pi fixture 配合结果沿用前述记录。本轮未重跑，也未请求真实 V2EX。
- 安装边界：历史已经按 Git 来源部署，本轮未更新或检查安装副本，没有修改模型、压缩设置或等待计划。
- 本机托管安装的入口与依赖按本地约定定位，不能把 README 中 npm 全局安装示例当作本机事实。
- 待办：真实长会话的摘要质量与上游成功率仍需观察，凭据接入统一尚未实施。现有凭据解析不等于已经支持所有 pi 认证来源。
- CI 与发布：此次文档推送后的结果未查询，没有发版或运行集成脚本。

## 2026-10-15 / 04：空响应重试与自检脚本的假上游接管

状态：本地完成并提交，基线 `7fca034`。上一轮遗留的未提交改动（空响应识别）在本轮补齐验证与文档后一并入库。

### 本轮完成

- 上游「整轮没有可见内容」（`AI Chat returned no visible response.` / `AI Chat 没有返回可见内容`）纳入瞬时故障：`TransientErrorKind` 增加 `empty`，新增 `EMPTY_RESPONSE_PATTERN`，并入 `/v2ex retry` 的退避重试。
- 该故障的两种落法都认：`stopReason` 为 `error` 的报错形态；上游把它当正文返回、pi 以正常 `stop` 收尾的正文形态。正文形态只在「这一轮恰好一块纯文本、且没有工具调用」时才认，避免把正常回复误判成空响应。
- 修复自检脚本的假上游接管：`prepareAgentDir` 原先只改了扩展自己的 `baseUrl`，而补全请求是 pi 按 `models.json` 自己发的，于是假上游只收得到配额请求，「补全固定 522」从未生效。现在同时改写临时目录里 `models.json` 的 `v2ex.baseUrl`。
- 新增真机用例 `--case retry-empty`（`npm run verify:retry-empty`）：假上游按 openai-completions 的流式协议把空响应那句当正文返回，200 正常收尾。用例断言正文恰好一块纯文本、`stopReason` 为 `stop`、pi 自己的 `auto_retry_start` 为 0、扩展归类为「空响应」并排入 `reason=error` 的重试。

### 验证与边界

- V2EX 全量 Node 测试 92 项通过，新增 3 项覆盖报错形态、正文形态（含带工具调用与正常回复的反例）与开关关闭时的提示。
- 严格 TypeScript 检查对 `src/` 与 `test/` 全部通过 —— 但用一个声明桩替换了 `pi-smart-compact/extension-api` 的类型入口，原因见待办。
- 真机自检：`retry`、`retry-empty` 通过，`start-live` 回归通过。三条都打 127.0.0.1 假上游，未请求真实 V2EX，临时 agent 目录已清理。
- 修正一处此前被记成事实的错误结论：`retry` 用例原先并未把 522 送到 pi，pi 报的是连不上真实端点的 `Connection error.`，因此「pi 会先快速重试三次」被当成了 522 的行为。实测 522 不在 pi 的可重试清单里（清单收了 500 / 502 / 503 / 504 / 520 / 524，没有 522），pi 一次都不重试，扩展在同一个 `agent_settled` 上接手；`Connection error.` 那种措辞走的是网络那条路。README 中英两版已同步，含实测输出。
- `start` / `rearm` 仍打真实配额接口，本机需直连或设 `V2EX_VERIFY_PROXY`，本轮未跑。
- 空响应在真实长会话里的触发频率与重试成功率未观察；现有假上游验收只证明链路成立。

### 待办

- `npm run typecheck` 目前不可用：仓库没有 `tsconfig.json`，`tsc --noEmit` 只会打印帮助；补上配置后还会撞到依赖侧问题 —— `pi-smart-compact` 的 `exports["./extension-api"].types` 指向 `src/extension-api.ts`，而它 import 的 `./constants.ts` 不在该包 `files` 里，直接检查会报 TS2307。要么在 fork 里把 `types` 指向随包发布的产物，要么在本仓库用 `paths` 映射绕开。**已由下方 2026-10-15 / 05 处理。**
- 仓库没有 `package-lock.json`；`pi-smart-compact` 按分支 `v2ex-recovery` 解析，分支移动会静默换实现。是否引入锁文件待定。
- 真实长会话的摘要质量与上游成功率仍需观察。

## 2026-10-15 / 05：类型检查落地

状态：本地完成并提交，基线 `c3f10a0`。本仓库补 `tsconfig.json`，依赖侧在 fork 上修类型入口。

### 本轮完成

- 新增 `tsconfig.json`：覆盖 `src/` 与 `test/`，`strict` + `NodeNext` + `allowImportingTsExtensions` + `noEmit`，与 Node 直接跑 `.ts` 的方式对齐。`npm run typecheck` 至此真正可用，不再只是打印帮助。
- fork `huaxianyan/pi-smart-compact` 的 `v2ex-recovery` 分支：`exports["./extension-api"].types` 由 `src/extension-api.ts` 改为 `dist/extension-api.d.ts`，并按仓库既有构建步骤（`tsc -p tsconfig.json --emitDeclarationOnly`）生成该声明与 `.d.ts.map`，与 `dist/*.js` 一样强制入库（`dist/` 在 `.gitignore` 里）。声明自包含，只有一条对 `@earendil-works/pi-coding-agent` 的外部类型 import。运行时入口 `dist/extension-api.js` 未动。
- `src/extension-api.ts` 继续随包发布：source map 的目标指向它，读者也能看到实现。
- README 中英的「类型检查」一节改为 `npm run typecheck`，并记录这两处修复；「让编辑器找到 pi 的类型」一节改为以 `npm install` 的结果为主、手工软链为兜底。

### 验证与边界

- `npm run typecheck` 通过（`src/` + `test/` 全量，strict）。
- 从 git 重装依赖（删掉 `node_modules/pi-smart-compact` 后 `npm install`）确认拿到的是修复后的分支：`dist/extension-api.d.ts` 与 `.d.ts.map` 都在，`types` 指向正确；重装后 `npm run typecheck` 与 92 项 Node 测试均通过。
- fork 的声明用本机 TypeScript 5.9.3 生成（fork devDependencies 声明的是 7.0.2，本机未装 Bun 工具链）。声明内容只有常量与接口，与编译器版本无关；但严格来说不是用 fork 声明的构建命令跑出来的，记为已知偏差。
- 本轮未重跑 fork 的 Bun 构建、定向测试与 `release:check`，未改动 `dist/*.js`，未发布 npm。
- fork 包根入口的 `types: ./dist/index.d.ts` 仍然缺文件：它的声明会带上 `.ts` 后缀的相对导入，而 `files` 不发布整个 `src`，补它需要另立一项。

### 待办

- 仓库仍无 `package-lock.json`；`pi-smart-compact` 按分支解析，分支移动会静默换实现。是否引入锁文件待定。
- fork 的 `dist/index.d.ts`（包根 `types`）仍是空的，见上。
- 真实长会话的摘要质量与上游成功率仍需观察；空响应的实际触发频率也未观察。
