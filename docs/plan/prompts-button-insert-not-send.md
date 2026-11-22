# 实施方案：Prompts 按钮选中后填入输入框，不直接发送

关联意图：[../intent/prompts-button-insert-not-send.md](../intent/prompts-button-insert-not-send.md)

## 前置环境

- Node v24.21.0（插件要求 ≥24）
- npm 11.19.0
- Playwright Chromium 已装好，可直接跑端到端测试
- 本机 shell 可能带着 `PASEO_WORKSPACE_ID`，跑端到端前要清掉：`env -u PASEO_WORKSPACE_ID -u PASEO_AGENT_ID -u PASEO_AGENT_CWD -u PASEO_HOME`
- 本机 `NODE_ENV=production`，装依赖要加 `--include=dev`，否则开发依赖会被跳过

## 版本升级（阶段 0 发现）

"填进输入框"依赖 Paseo 的输入框接口（`composers` / `insertText` / `focus`），0.15 及以前的 SDK 没有。实测 0.16.0-beta.1 起才有。

- `paseo-plugin.json`：`requirements.paseo` 从 `>=0.11.0-beta.3` 改为 `>=0.16.0`。
- `package.json`：`@getpaseo/plugin` 和 `@getpaseo/cli` 从 `0.11.0-beta.3` / `0.11.1` 改为 `0.16.0`（按仓库惯例，依赖固定为最低支持版本，让 typecheck 校验下限）。

## 工程原则选择

| 原则 | 触发依据与使用原因 | 执行的行动 |
|---|---|---|
| 少做一点 `principle-laziness-protocol` | 一处小改动，要控制范围 | 只改 `client/pill.tsx` 一个文件的核心逻辑和文案，不拆组件、不加抽象层 |
| 基础先行 `principle-foundational-thinking` | 写逻辑前要先定"怎么拿到输入框对象" | 先定用 `client.composers({agentId})` 注入进点击处理，避开 hook 的条件调用问题 |
| 按可验证单元推进 `principle-sequence-verifiable-units` | 多步骤修改，需要可检查的进度 | 分 4 个阶段，每阶段结束有能跑的检查，绿了再进下一阶段 |
| 测试行为 `principle-test-behavior-not-implementation` | 要改端到端测试 | 断言"输入框里有文字、agent 没收到消息"这类可观察结果，不锁内部写法 |
| 证明它能工作 `principle-prove-it-works` | 最后要宣布完成 | 跑 typecheck+单测+e2e，再 reload 插件在真实界面点一遍 |

## 阶段 0：建立基线

- 装依赖（`npm ci --include=dev`）。
- `npm run check`（类型检查 + 单测）确认全绿：77 个单测通过。
- 升 Paseo SDK 到 0.16.0，再跑 typecheck + 单测：仍 77 个通过。
- 跑 web 端到端里跟 composer pill 相关的 3 个用例，记录当前"点选即发送"的行为作对照：3 个通过。

检查点：check 全绿、相关 e2e 用例当前通过。（已达成）

## 阶段 1：核心行为改动（`client/pill.tsx` + `client/variables.tsx`）

- 定机制：把 `client.composers` 注进 `PromptPicker`，agent 分支用 `composers({agentId})[0].insertText(文字)` + `focus()`；workspace 的"复制"分支不动。
- 兜底：`composers({agentId})` 为空（拿不到输入框对象）时，提示用户并停下，不静默无效果。
- 改两处逻辑：`choose()` 里无变量的分支、变量表单的 `onSubmit`，都从 `send()` 换成"插入 + 聚焦"。
- `client/variables.tsx`：表单按钮的 `action` 类型加 "Insert"，图标判断跟着加一档。
- 同步改这个入口的文案：按钮标题 "Send a saved prompt" → "Insert a saved prompt"、行标签 "Send X" → "Insert X"、成功提示 "Sent X" → "Inserted X…"、变量表单按钮 "Send" → "Insert"。（具体用词可再调）

检查点：`npm run typecheck` 通过。

## 阶段 2：更新测试

文案 "Send a saved prompt" / "Send X" 会波及 8 个文件，全部要跟着改：

- `e2e/specs/agent.spec.ts`：3 个用例（composer pill 发送、变量表单、转义大括号），断言从"agent 收到文本"改成"输入框里有文本、agent 没收到"。
- `e2e/specs/compact.spec.ts`：弹窗打开用例里的按钮/行标签。
- `e2e/mobile/flows/agent-pill.yaml`、`agent-variables.yaml`、`mobile-picker-open.yaml`、`mobile-picker-last.yaml`：选提示词的流程断言。
- `e2e/mobile/flows/lib/open-agent.yaml`、`open-agent-link.yaml`：这两处把 "Send a saved prompt" 当"agent 已打开"的信号，标题改后要同步换。

不动的用例：workspace 复制、`/prompt`、`/prompts` 面板、附件。

检查点：`npm run check` 全绿，`npm run e2e:web` 相关用例通过。

## 阶段 3：端到端验证

- `npm run check` + `npm run e2e:web` 全绿。
- 把本地这份装进 Paseo（或 reload），在真实界面点一遍：选提示词 → 出现在输入框、不发送 → 自己点发送；带变量的走一遍表单。
- 补两个放行门槛的实测：
  1. 空闲 agent、运行中 agent 各点一次，确认都能拿到输入框对象（拿不到时弹兜底提示）。
  2. 确认插入后光标在输入框里，能直接接着打字。

检查点：e2e 全绿 + 上述两项实测通过 + 界面实测符合意图文档的验收标准。
