# 意图：Prompts 按钮选中后填入输入框，不直接发送

## 背景

Paseo Prompt Manager 在 agent 输入框上方有个 Prompts 按钮，点开搜索并选中一条提示词后，会直接发给 agent（代码里调 `paseo.agents.ref(agentId).send(text)`）。用户容易误发，希望改成"填进输入框、由用户自己点发送"。

## 目标

- 选中一条提示词后，把文字插进输入框**光标处**（追加、不覆盖已有内容），光标放回输入框，**不发送**。
- 带 `{{变量}}` 的提示词照旧先弹表单，填完后同样"填进输入框"，不发送。

## 范围

只改这一个入口：agent 输入框上方的 Prompts 按钮（`client/pill.tsx` 里的 `PromptPicker`）。

不改：
- 面板里的 Send to agent 按钮
- `/prompt 名字` 命令
- 附件里的 Saved prompt
- 工作区顶部按钮的"复制到剪贴板"行为

## 实现要点

- 用 Paseo 的输入框对象（composer）的能力：`insertText`（插入光标处）、`focus`（聚焦）。`setPrompt`（整体替换）仅作备用，本次不用。
- 通过 `useComposers({ agentId })` 拿到当前 agent 的输入框对象。
- 把 `choose()` 里无变量分支的 `send()` 换成"插入文字 + 聚焦"；带变量表单的提交（`onSubmit`）也从 `send` 改成"插入"。

## 验收标准

1. agent 输入框点 Prompts → 选一条（无变量）→ 文字出现在输入框光标处，未发送。
2. 选一条带变量的提示词 → 弹表单 → 填完 → 文字填进输入框，未发送。
3. 输入框已有文字时，新选的提示词追加在光标处，不覆盖。
4. 其余入口行为不变。
