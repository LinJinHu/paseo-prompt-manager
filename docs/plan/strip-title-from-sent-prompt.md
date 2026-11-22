# 实施方案：送出去的提示词不带标题

关联意图：本次对话确认（三个发送入口都去掉开头标题，只去掉正文最开头那一行）。

## 背景

插件没有单独的标题字段。列表里的标题来自正文第一个 Markdown 标题行（`server/library.ts:310`）。插入和发送时，插件把整个正文都送出去，标题行也在里面。预览列表已经跳过标题行（`shared/markdown.ts:157` 的 `previewLines`），插入和发送没有跳过。

## 目标

送出去的文字 = 正文去掉最开头那一行标题（`#` 到 `######`），以及标题后面的空行。

- 只处理正文最开头。正文中间的标题行不动。
- 如果去掉标题后没有内容，保持原文不变。
- 文件本身不改。标题还在文件里，列表照常显示名字。

## 范围

三个入口，行为一致：

1. 输入框上方的 Prompts 按钮（插入）
2. 面板里的 Send to agent（发送）
3. `/prompt 名字` 命令（发送）

不动：

- 附件里的 Saved prompt（仍然是文件原样）
- 工作区顶部的"复制"按钮（仍然复制全文）

## 工程原则选择

| 原则 | 触发依据与使用原因 | 执行的行动 |
|---|---|---|
| 少做一点 `principle-laziness-protocol` | 同一规则要用于三处，别在三处各写一遍 | 在 `shared/markdown.ts` 加一个函数，三处调用它，不新增模块或抽象层 |
| 基础先行 `principle-foundational-thinking` | 先定函数放哪、边界怎么算，再接三个入口 | 函数与 `previewLines` 放同一文件；边界：开头标题、后续空行、去掉后为空则返回原文 |
| 按可验证单元推进 `principle-sequence-verifiable-units` | 多步骤改动，需要能检查的进度 | 分 4 个阶段，每阶段有检查点 |
| 测试行为 `principle-test-behavior-not-implementation` | 要加单测和改端到端断言 | 单测断言函数返回值；端到端断言"输入框/agent 收到的文字"，不锁内部写法 |
| 证明它能工作 `principle-prove-it-works` | 最后要宣布完成 | 跑 check + 端到端 + 真机 reload 点一遍 |

## 阶段 1：共享函数（先写测试）

- 在 `server/markdown.test.ts` 先加用例，覆盖：
  - 开头一级标题：去掉标题和后面的空行。
  - 开头二级到六级标题：同样去掉。
  - 正文中间的标题：保留。
  - 第一行不是标题：原样返回。
  - 只有标题没有正文：原样返回。
  - CRLF 换行：同样处理。
- 在 `shared/markdown.ts` 加 `promptBody(content: string): string`。标题判断复用和服务端一致的写法（`^ {0,3}#{1,6}\s+...$`）。
- 先运行测试确认失败，再实现到通过。

检查点：`npm test` 通过，新用例覆盖以上 6 种情况。

## 阶段 2：接入三个入口

- `client/pill.tsx`：`choose()` 里，对 `item.text` 先过一次 `promptBody`，再做变量解析和填充。
- `client/prompts.tsx`：`sendToAgent` 和变量表单提交处，对 `saved.content` 先过 `promptBody`。
- `index.client.tsx`：`/prompt` 命令里，对 `prompt.content` 先过 `promptBody`，再解析必填变量。

注意：变量解析必须和最终送出的文字用同一份内容。所以先去掉标题，再解析变量。

检查点：`npm run typecheck` 通过。

## 阶段 3：更新端到端断言

- `e2e/specs/agent.spec.ts`：
  - 输入框应等于"正文"，不再是 `CODE_REVIEW` 全文。为此加正文常量，例如 `CODE_BODY`。
  - `/prompt`、`/prompt` 带默认值、面板发送：agent 收到的都是正文。
  - 大括号用例：插入的是去掉 `# React` 后的正文。
  - 附件用例不变（附件仍是全文）。
  - 复制按钮用例不变（仍复制全文）。
- `e2e/specs/compact.spec.ts`：输入框断言改为 `# Step 12` 之后的正文。
- 移动端流程：现有断言找的是正文文字，预期不用改；跑之前再核对一遍。

检查点：`npm run check` 全绿；web 宽屏与手机布局的相关用例通过。

## 阶段 4：文档与真机验证

- `README.md`、`OVERVIEW.md`：说明标题只用于列表显示，插入和发送时去掉。
- `npm run check`、相关端到端全绿。
- `paseo plugin reload paseo-prompt-manager`，在真实界面点三个入口：
  1. Prompts 按钮插入 → 输入框只有正文。
  2. 面板 Send to agent → agent 收到正文。
  3. `/prompt 名字` → agent 收到正文。
  4. 正文中间带标题的提示词 → 中间标题保留。

检查点：以上四项在真实界面通过。

## 已知边界

- 判断标题只看第一行（允许前面有空行）。正文中间的标题不动。
- 只有标题、没有正文的提示词，保持原样送出。
- 附件和"复制"不受这次改动影响。
