---
title: Harness、Agent Loop 与项目级 Harness：使用 Claude Code / Codex 时需要自己设计吗？
date: 2026-07-20 22:05:00
tags:
  - AI
  - Agent
  - Claude Code
  - Codex
---

在讨论 Claude Code、Codex、Skills、MCP 或多 Agent 工作流时，经常会遇到两个容易混淆的概念：**Harness** 和 **Agent Loop**。

它们有关联，但并不是同一个层级的东西。

> **Agent Loop 是 Agent 不断“思考—行动—观察—再思考”的执行循环；Harness 是承载、约束并扩展这个循环的完整运行系统。**

对于日常项目开发，Claude Code 和 Codex 已经提供了产品级 Harness，我们通常不需要从零实现底层循环。但为了让编码 Agent 更稳定地理解、修改和验证项目，仍然值得建设一套轻量的**项目级 Harness**。

![Harness、Agent Loop 与项目级 Harness 完整知识总览](/articleImgs/Harness-Agent-Loop-与项目级-Harness/overview.webp)

------

## 一、什么是 Agent Loop？

Agent Loop 是 Agent 的核心控制流程，它回答的是：

> 模型完成当前一步以后，接下来应该做什么？

一个最简化的 Agent Loop 大致如下：

```text
读取用户目标
  ↓
模型分析并决定下一步
  ↓
调用工具
  ↓
执行操作并获得结果
  ↓
将结果放回上下文
  ↓
继续分析，直到任务完成
```

![Agent Loop 从用户目标到持续迭代的执行流程](/articleImgs/Harness-Agent-Loop-与项目级-Harness/agent-loop.svg)

对应的伪代码可以写成：

```ts
while (!done) {
  const response = await model.generate(context)

  if (response.toolCall) {
    const result = await executeTool(response.toolCall)
    context.push(response, result)
    continue
  }

  done = true
  return response.text
}
```

真实的 Agent Loop 会复杂得多，还需要处理：

- 一次返回多个工具调用时如何调度
- 工具失败后是否重试
- 上下文过长时如何压缩
- 如何判断任务已经完成
- 如何避免无限循环
- 用户中途补充要求时如何继续
- 是否允许启动子 Agent

但无论实现多复杂，它的核心仍然是：

```text
Reason → Act → Observe → Reason
```

也就是“推理、行动、观察，再继续推理”。

------

## 二、什么是 Harness？

Harness 原意有“控制装置、线束、挽具”的含义。在 Agent 领域，可以把它理解为：

> **把模型、Agent Loop、工具和运行环境连接起来，并保证它们能够安全、稳定运行的系统。**

一个典型的 Harness 可能包含：

```text
Harness
├─ Agent Loop
├─ 模型调用与模型选择
├─ Prompt / 上下文构造
├─ 工具注册与工具执行
├─ 文件系统和 Shell
├─ 权限审批与沙箱
├─ 会话状态持久化
├─ 上下文压缩
├─ 超时、重试和预算控制
├─ 日志与可观测性
├─ MCP / Skills / Hooks
└─ 子 Agent 与任务编排
```

![Harness 由 LLM、Agent Loop、工具、上下文和沙箱组成](/articleImgs/Harness-Agent-Loop-与项目级-Harness/harness.svg)

需要注意，业界对 Harness 的边界没有完全统一。

OpenAI 在介绍 Codex 时，有时会把 Codex Harness 描述为支撑不同产品形态的 Agent Loop 与执行逻辑；在更宽泛的工程语境中，Harness 通常还包括工具、沙箱、状态、权限和扩展机制。

本文采用更宽泛的定义：

> Agent Loop 表示执行流程；Harness 表示承载这套流程的完整运行系统。

------

## 三、Harness 和 Agent Loop 的核心区别

| 对比维度 | Agent Loop | Harness |
| --- | --- | --- |
| 关注范围 | Agent 的循环执行逻辑 | Agent 的完整运行系统 |
| 核心问题 | 下一步做什么，何时结束 | 以什么工具、权限、上下文和规则运行 |
| 工具调用 | 决定何时调用工具 | 注册、执行、限制并记录工具调用 |
| 上下文 | 消费并更新上下文 | 构造、裁剪、压缩和持久化上下文 |
| 权限与沙箱 | 通常不直接负责 | 通常负责 |
| 错误处理 | 决定是否继续 | 提供超时、重试、恢复和降级机制 |
| 二者关系 | Harness 的核心组成部分 | 包含 Agent Loop |

可以用机器人来类比：

- **LLM**：大脑
- **Agent Loop**：思考、行动、观察、再次思考的循环
- **Tools**：手、眼睛、终端和编辑器
- **Harness**：整个身体、控制系统和安全装置
- **Context**：工作记忆
- **Sandbox / Permission**：安全护栏

一句话概括：

> **Agent Loop 决定执行顺序，Harness 决定执行能力和运行边界。**

------

## 四、以“修复一个 Bug”为例

假设我们让 Codex 修复一个前端 Bug：

```text
读取用户描述
  ↓
搜索相关代码
  ↓
分析根因
  ↓
修改代码
  ↓
运行测试
  ↓
根据测试结果继续修改
  ↓
测试通过并输出结果
```

其中下面这段属于 Agent Loop：

```text
搜索 → 分析 → 修改 → 测试 → 观察结果 → 再次分析
```

而下面这些属于 Harness：

- 如何搜索和读取文件
- 如何应用代码补丁
- Shell 命令在哪个目录执行
- 哪些命令需要用户审批
- Agent 是否允许访问网络
- 测试超时以后如何处理
- 上下文太长时如何压缩
- 会话中断以后如何恢复
- 如何加载 `AGENTS.md`、`CLAUDE.md`、Skills 和 MCP

------

## 五、使用 Claude Code 或 Codex，需要自己实现 Harness 吗？

通常不需要从零实现。

Claude Code 和 Codex 本身已经是成熟的编码 Agent Harness，它们已经处理了大量底层工作，例如：

- 模型调用与 Agent Loop
- 文件搜索和编辑
- Shell 命令执行
- 工具调用结果回填
- 上下文管理与压缩
- 权限控制和沙箱
- 会话保存与恢复
- MCP、Skills 或其他扩展机制

因此，日常使用时没有必要再写一个这样的循环：

```ts
while (!done) {
  const response = await callLLM(context)
  const action = parseAction(response)
  const result = await execute(action)
  context.push(result)
}
```

除非你的目标是开发一个类似 Claude Code、Codex 的 Agent 产品，或者需要完全自定义模型、工具、状态与调度方式。

但这不代表项目什么都不需要做。

------

## 六、真正需要建设的是“项目级 Harness”

可以把 Harness 分成三个层级：

1. **产品级 Harness**：由 Claude Code、Codex 提供，负责 Agent Loop、工具、权限、上下文和会话。
2. **项目级 Harness**：由项目团队维护，负责项目规则、架构边界、运行方式和验收标准。
3. **任务级 Harness**：面向复杂任务临时创建，负责任务拆分、并行执行、独立复核与结果汇总。

![产品级、项目级和任务级 Harness 的三层结构](/articleImgs/Harness-Agent-Loop-与项目级-Harness/harness-layers.webp)

### 1. 产品级 Harness

这是 Claude Code 和 Codex 已经提供的部分，一般不需要项目团队重新实现。

### 2. 项目级 Harness

项目级 Harness 的作用，是让 Agent 能回答下面这些问题：

- 项目怎么启动？
- 应该使用 npm、pnpm 还是 yarn？
- 一个功能应该放在哪个模块？
- 哪些目录不能修改？
- 修改业务逻辑后应该运行哪些测试？
- 什么条件才算任务完成？

它通常不是一个独立框架，而是一组工程设施：

```text
项目级 Harness
├─ CLAUDE.md / AGENTS.md
├─ README 和架构文档
├─ package.json scripts
├─ lint / typecheck / test / build
├─ Docker 和环境初始化脚本
├─ .env.example
├─ Hooks
├─ Skills
├─ MCP
└─ Definition of Done
```

### 3. 任务级 Harness

任务级 Harness 用于大规模迁移、安全审计、复杂研究、多方案验证等任务。

它可能临时组织出这样的结构：

```text
规划 Agent
   ↓
多个执行 Agent 并行处理
   ↓
Reviewer Agent 独立检查
   ↓
测试 Agent 验证
   ↓
汇总 Agent 生成最终结果
```

Anthropic 的 Dynamic Workflows 就属于这类能力：Claude Code 可以根据具体任务动态生成编排逻辑，启动多个隔离的子 Agent，并进行交叉验证与结果汇总。

------

## 七、项目级 Harness 最少应该包含什么？

对于普通 TypeScript、React、Next.js 或 Node.js 项目，不需要一开始就设计复杂的多 Agent 系统。下面这些基础设施通常已经足够。

### 1. 明确的项目指令

Claude Code 通常使用 `CLAUDE.md`，Codex 可以使用 `AGENTS.md`。

例如：

```md
# Project Overview

This is a pnpm monorepo using Next.js and TypeScript.

## Commands

- Install: `pnpm install`
- Type check: `pnpm typecheck`
- Lint: `pnpm lint`
- Unit tests: `pnpm test`
- Build: `pnpm build`

## Architecture

- Shared UI components belong in `packages/ui`
- Business state belongs in stores
- API requests belong in services
- React components must not call backend APIs directly

## Working Rules

- Do not use npm or yarn
- Do not add production dependencies without a clear reason
- Avoid modifying unrelated files
- Run typecheck after changing TypeScript code
- Update tests after changing business logic

## Definition of Done

A task is complete only when:

1. TypeScript passes
2. Relevant tests pass
3. No unrelated files are changed
4. The final response explains the root cause and validation
```

### 2. 确定性的验证命令

不要只告诉 Agent“确保代码没有问题”，而应该提供可以执行的验证方式：

```json
{
  "scripts": {
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "build": "next build",
    "verify": "pnpm lint && pnpm typecheck && pnpm test"
  }
}
```

验证命令越明确，Agent 越容易形成稳定闭环：

```text
修改代码 → 运行验证 → 观察错误 → 继续修复 → 验证通过
```

### 3. 可重复的开发环境

最好让新开发者和 Agent 都能通过少量命令启动项目：

```bash
pnpm install
docker compose up -d
pnpm dev
```

同时准备 `.env.example`、`docker-compose.yml`、`README.md` 和初始化脚本，不要让 Agent 猜测数据库、Redis、Node 版本、服务启动顺序或测试账号的配置方式。

### 4. 合理使用 Hooks、Skills 和 MCP

它们解决的是不同问题：

- **Hooks**：在固定时机确定性执行动作，例如格式化、检查危险命令或运行验证
- **Skills**：沉淀可复用的多步骤流程，例如性能排查、代码审查或模块生成
- **MCP**：接入项目外部工具和数据，例如 GitHub、监控平台、设计稿或数据库

它们不是越多越好。只有当某个流程高频重复、容易出错或确实需要外部系统时，才值得沉淀。

------

## 八、什么时候需要任务级 Harness？

下面这些任务通常适合更复杂的任务编排：

- 大规模架构迁移
- 跨几十个模块的重构
- 全仓库安全审计
- 大量 Issue 的自动分类和处理
- 多方案并行实现与评估
- 需要独立 Reviewer 进行交叉验证
- 工作量无法提前确定，需要循环到没有新问题为止

但普通 Bug 修复、单个功能开发或小规模重构，通常不需要专门创建多 Agent Harness。

原因也很直接：子 Agent 会增加 Token 消耗，并行任务会增加协调与合并成本，过度设计反而会让简单任务变慢。

------

## 九、如何判断自己需要哪一层？

### 普通单次任务

修一个 Bug、改一个组件、补一个接口，直接使用 Claude Code 或 Codex 的默认 Harness 即可。

### Agent 经常违反相同的项目规则

总是用错包管理器、放错目录、漏跑测试，就补充项目级 Harness：`CLAUDE.md`、`AGENTS.md`、验证脚本和架构约束。

### 存在高频重复流程

每次都要按相同步骤排查性能、生成模块或审查 PR，就将流程沉淀成 Skill、脚本或 Hook。

### 任务规模很大，需要拆分和独立复核

全仓迁移、安全审计、批量修复或复杂研究，再考虑任务级 Harness、Dynamic Workflow 或自定义多 Agent 编排。

------

## 十、最终结论

Harness 和 Agent Loop 最准确的关系是：

```text
Agent Loop = Agent 的执行循环
Harness = 承载和管理执行循环的完整运行系统
```

在 Claude Code 和 Codex 的使用场景中：

1. **不需要从零实现底层 Agent Loop**，产品已经提供了成熟的编码 Agent Harness。
2. **正式项目应该建设轻量的项目级 Harness**，让项目更容易被理解、操作和验证。
3. **复杂任务才需要任务级 Harness**，例如多 Agent、Worktree、并行执行和独立复核。

一句话总结：

> Claude Code 和 Codex 已经解决了“Agent 如何运行”；项目团队需要解决的是“Agent 如何在这个项目里正确地工作”。

------

## 参考资料

- [OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [OpenAI：Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)
- [Anthropic：A harness for every task — dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
