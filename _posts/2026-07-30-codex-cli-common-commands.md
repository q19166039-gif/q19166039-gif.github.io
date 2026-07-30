---
layout: post
title: "Codex CLI 常用命令手册"
date: 2026-07-30
excerpt: "面向终端用户的 Codex CLI 命令参考，涵盖交互式启动、非交互执行、代码审查、会话管理、MCP/插件及安全边界。"
categories: [技术, Codex]
tags: [AI, Codex, CLI, 教程]
---

## 快速开始

Codex CLI 是 OpenAI 推出的终端 AI 编程助手，运行在本地终端中，支持交互式聊天和非交互式脚本两种使用模式。

```bash
# 安装完成后查看版本
codex --version

# 生成 shell 补全（以 zsh 为例）
codex completion zsh > "${fpath[1]}/_codex"

# 检查安装和认证状态
codex doctor --summary
codex login status
```

首次使用需要先登录：`codex login` 默认打开浏览器走 ChatGPT OAuth 流程；在无浏览器的环境可用 `--device-auth` 或 `--with-api-key`。

## 交互式启动

不带子命令直接运行 `codex` 进入交互终端（TUI）：

```bash
# 直接启动，可选附带初始提示
codex "分析这个仓库的依赖结构"

# 带图片附件启动
codex -i screenshot.png "根据截图实现这个组件"

# 指定工作目录
codex -C /path/to/project "初始化一个新模块"

# 指定模型
codex -m <MODEL_NAME>

# 启用实时网络搜索（默认使用缓存模式）
codex --search "查询最新 React 文档"
```

启动后你可以输入自然语言指令，Codex 会自主执行命令、读写文件、调用工具。TUI 底部状态栏显示当前模型、上下文用量、Git 分支等信息，可用 `/statusline` 自定义。

## 常用启动参数

| 参数 | 作用 |
|------|------|
| `-m, --model <NAME>` | 覆盖配置中的模型 |
| `-C, --cd <PATH>` | 设置工作目录 |
| `-c key=value` | 临时覆盖配置项（TOML 格式） |
| `-i, --image <PATH>` | 附图片到初始提示 |
| `--search` | 启用实时网络搜索 |
| `--profile, -p <NAME>` | 加载 `$CODEX_HOME/<name>.config.toml` |
| `--oss` | 使用本地开源模型（需配置 Ollama/LM Studio） |
| `--sandbox, -s <MODE>` | 沙箱策略：`read-only` / `workspace-write` / `danger-full-access` |
| `--remote <ADDR>` | 连接远程 app-server（`ws://` / `wss://` / `unix://`） |
| `--no-alt-screen` | 禁用 TUI 备用屏幕模式 |
| `--dangerously-bypass-approvals-and-sandbox` | **极高风险**：跳过所有审批和沙箱 |

`--sandbox workspace-write` 配合 `--ask-for-approval on-request` 是日常开发中比较实用的组合：允许在工作区内写入，并由 Codex 在确有必要时请求批准。

## 会话管理

Codex 自动保存交互会话，可按 ID 或名称管理：

```bash
# 恢复最近的会话（限当前目录）
codex resume --last

# 按 ID 恢复
codex resume <SESSION_ID>

# fork 最近的会话到新聊天
codex fork --last

# 归档会话（隐藏，不删除）
codex archive <SESSION_ID>

# 恢复归档
codex unarchive <SESSION_ID>

# 永久删除
codex delete <SESSION_ID>
codex delete <UUID> --force   # 跳过确认（仅 UUID）
```

`codex resume --last` 只搜索当前工作目录中的会话，加 `--all` 则搜索所有目录。如果当前目录与会话保存时的目录不同，Codex 会询问使用哪个目录，也可通过配置 `tui.resume_cwd` 预设行为。

`codex exec` 也支持恢复非交互会话（见下一节）。

## 非交互 codex exec

`codex exec`（别名 `codex e`）用于脚本、CI 管道等无需人工介入的场景。输出策略：进度信息走 stderr，最终结果走 stdout。

```bash
# 基本用法
codex exec "总结仓库结构，列出前 5 个风险区域"

# 管道输入：指令作为参数，stdin 内容作为上下文
npm test 2>&1 | codex exec "总结失败测试并建议最可能修复" > test-summary.md

# stdin 即提示（使用 `-`）
cat prompt.txt | codex exec -

# 不持久化会话文件
codex exec --ephemeral "检查这个仓库的健康状况"

# 输出最终消息到文件
codex exec -o summary.md "生成发布说明"

# JSON Lines 输出供脚本消费
codex exec --json "分析性能问题" | jq

# 结构化输出：指定 JSON Schema
codex exec "提取项目元数据" --output-schema ./schema.json -o meta.json

# 覆盖沙箱权限
codex exec --sandbox workspace-write "修复测试失败"

# 跳过 Git 仓库检查（仅限安全环境）
codex exec --skip-git-repo-check "生成配置模板"

# 指定模型
codex exec -m <MODEL_NAME> "写一个快速排序"

# 使用本地模型
codex exec --oss --local-provider ollama "解释这段代码"
```

### 恢复非交互会话

```bash
codex exec "评审变更中的竞态条件"
codex exec resume --last "修复你找到的竞态条件"
codex exec resume <SESSION_ID> "继续之前的工作"
```

### 自动化认证

`codex exec` 默认复用已保存的认证。在 CI 中：

```bash
# 单次调用注入 API key
CODEX_API_KEY=<key> codex exec --json "分类待办工单"
```

**安全提醒**：`CODEX_API_KEY` 仅在 `codex exec` 中生效。在 GitHub Actions 中优先使用 [openai/codex-action](https://github.com/openai/codex-action)，避免将 API key 暴露给工作流中的任意代码。

## 代码审查 codex review

`codex review` 用于非交互式代码审查，支持四种审查目标，**互斥使用**：

```bash
# 审查未提交的变更（staged + unstaged + untracked）
codex review --uncommitted

# 对比某个分支
codex review --base main

# 审查某个 commit
codex review --commit <SHA>
codex review --commit <SHA> --title "提交标题"  # 显示友好标题

# 自定义审查指令（最灵活）
codex review "重点关注性能热点和内存泄漏"

# 从 stdin 读取审查指令
echo "检查安全漏洞" | codex review -
```

`--uncommitted`、`--base`、`--commit` 三者互斥，`--title` 仅与 `--commit` 联用。审查结果包含行为变更分析和测试覆盖建议。

## 登录与诊断

```bash
# 浏览器 OAuth 登录
codex login

# 无浏览器环境：设备码流程
codex login --device-auth

# 通过 API key 登录
printenv OPENAI_API_KEY | codex login --with-api-key

# 通过 access token 登录
printenv CODEX_ACCESS_TOKEN | codex login --with-access-token

# 查询登录状态（可用于脚本：登录时 exit 0）
codex login status

# 登出清除凭证
codex logout

# 诊断报告（排查问题首选）
codex doctor               # 完整报告
codex doctor --summary      # 仅汇总
codex doctor --json         # 机器可读
codex doctor --ascii        # ASCII（无颜色）
codex doctor --all          # 展开长列表
```

`codex doctor` 检查的内容包括：安装完整性、配置文件、认证状态、运行时、Git 集成、终端兼容性、app-server 状态和会话清单。

## MCP / 插件

### MCP（Model Context Protocol）

```bash
# 列出已配置的 MCP 服务器
codex mcp list

# 查看某个服务器的配置详情
codex mcp get <NAME>
codex mcp get <NAME> --json

# 添加 stdio 型 MCP 服务器
codex mcp add my-tool -- npx @my/mcp-server
codex mcp add my-tool --env KEY=VALUE -- npx @my/mcp-server

# 添加流式 HTTP MCP 服务器
codex mcp add my-tool --url https://example.com/mcp
codex mcp add my-tool --url https://example.com/mcp --oauth-client-id <ID>

# OAuth 登录/登出（仅流式 HTTP 服务器支持）
codex mcp login <NAME> --scopes read,write
codex mcp logout <NAME>

# 删除 MCP 服务器
codex mcp remove <NAME>
```

如果在 `config.toml` 中将某个 MCP 服务器标记为 `required = true`，初始化失败时 `codex exec` 会直接报错退出，而非忽略。

### 插件

```bash
# 列出已安装插件
codex plugin list
codex plugin list --json
codex plugin list --json --available   # 包含市场可用的

# 安装插件
codex plugin add <NAME>
codex plugin add <NAME> --marketplace <MARKET>
codex plugin add <NAME> --json

# 删除插件
codex plugin remove <NAME>

# 管理插件市场
codex plugin marketplace list
codex plugin marketplace add owner/repo
codex plugin marketplace add https://github.com/owner/repo.git --ref v1.0
codex plugin marketplace add /path/to/local/marketplace
codex plugin marketplace upgrade           # 刷新所有 Git 市场
codex plugin marketplace upgrade <NAME>    # 刷新指定一个
codex plugin marketplace remove <NAME>
```

## 常用斜杠命令

以下命令在交互会话的 composer 中以 `/` 开头输入：

| 斜杠命令 | 作用 |
|---------|------|
| `/model` | 切换当前会话模型 |
| `/fast` | 切换 Fast 服务层级（仅模型支持时可见） |
| `/personality` | 设置沟通风格（`friendly` / `pragmatic` / `none`） |
| `/permissions` | 设置审批策略（Auto / Read Only / 自定义配置） |
| `/approve` | 批准一次自动审查拒绝，允许重试 |
| `/plan` | 切换到计划模式，输出方案再实现 |
| `/goal` | 设置/查看/暂停/恢复/清除任务目标 |
| `/compact` | 压缩上下文，总结早期对话释放 token |
| `/new` | 在同一 CLI 会话中启动新聊天 |
| `/clear` | 清屏并启动新聊天 |
| `/rename <NAME>` | 重命名当前会话 |
| `/archive` | 归档当前会话并退出 |
| `/delete` | 永久删除当前会话并退出 |
| `/fork` | 分叉当前聊天为新会话 |
| `/resume` | 从已保存列表中恢复会话 |
| `/side` | 启动临时旁路聊天，完成后回到主聊天 |
| `/status` | 查看当前配置和 token 用量 |
| `/usage` | 查看账户 token 用量或申请速率重置 |
| `/diff` | 查看当前 Git 工作树差异 |
| `/review` | 在当前会话内进行工作树审查 |
| `/mention <FILE>` | 附加文件到聊天上下文 |
| `/copy` | 复制最近一次输出 |
| `/mcp` | 列出可用 MCP 工具 |
| `/plugins` | 浏览已安装和可发现的插件 |
| `/apps` | 浏览应用（connector）并插入到提示 |
| `/skills` | 浏览并使用本地技能 |
| `/memories` | 配置记忆注入和生成 |
| `/init` | 生成 `AGENTS.md` 脚手架 |
| `/import` | 导入 Claude Code 配置和最近聊天 |
| `/hooks` | 查看和管理生命周期钩子 |
| `/agent, /subagents` | 切换子代理线程 |
| `/ps` | 查看后台终端及其最近输出 |
| `/stop` | 停止所有后台终端 |
| `/experimental` | 开关实验性功能 |
| `/keymap` | 查看/修改 TUI 快捷键绑定 |
| `/vim` | 切换 Vim 模式 |
| `/theme` | 选择语法高亮主题 |
| `/statusline` | 自定义底部状态栏项目 |
| `/title` | 自定义终端标题 |
| `/pets` | 选择终端宠物 |
| `/logout` | 登出并清除凭证 |
| `/quit, /exit` | 退出 CLI |
| `/feedback` | 发送日志给维护者 |
| `/raw` | 切换原始滚动模式（复制更方便） |

部分命令在任务运行中不可用，如 `/plan`、`/compact`、`/archive`、`/delete`、`/import`、`/clear`。斜杠命令支持 Tab 预输入（在任务运行时输入并按 Tab，等当前轮次结束后执行）。

## 推荐工作流

### 日常开发

```bash
codex --sandbox workspace-write -a on-request
```

交互式启动，允许在工作区内写入，并在需要扩大权限时请求批准。适合大多数本地项目。

### 代码审查闭环

```bash
codex review --uncommitted   # 先由 AI 审查
# 基于审查结果修改
codex exec "根据上一步的审查建议修改"   # 自动修复
```

### CI 自动修复管道

在 GitHub Actions 中使用 `openai/codex-action`，配置 `contents: read` 权限的 job 执行 Codex，将生成的 patch 作为 artifact 上传，由另一个有写权限的 job 创建 PR（具体示例见[官方文档](https://developers.openai.com/codex/noninteractive/)）。

### 日志排查

```bash
tail -n 200 app.log | codex exec "定位根因，列出最关键的 3 个错误，建议接下来的调试步骤" > triage.md
```

### 使用 MCP 工具扩展能力

```bash
codex mcp add db-tools -- npx @my/db-mcp
# 然后进入交互会话直接让 Codex 调用数据库工具
codex
```

## 安全边界

Codex CLI 的安全模型围绕两个概念：**沙箱**和**审批**。

### 沙箱策略

- `read-only`：模型生成的命令只能读，不能写。适合只审查不修改的场景。
- `workspace-write`：允许在工作目录内读写。日常开发推荐。
- `danger-full-access`：无限制的文件系统访问。仅在受控隔离环境使用。

### 极高风险：绕过安全机制

```bash
# 跳过所有审批提示和沙箱隔离
codex --dangerously-bypass-approvals-and-sandbox
codex exec --dangerously-bypass-approvals-and-sandbox
```

**`--dangerously-bypass-approvals-and-sandbox` 会跳过所有审批和沙箱，让模型生成的命令直接执行。** 仅在外部硬化的隔离环境（如一次性 CI 容器）中使用。**绝不要在普通本机或不受信任的仓库中使用此选项。**

类似地，`--dangerously-bypass-hook-trust` 跳过生命周期钩子的信任检查，仅适用于已验证钩子来源的自动化场景。

### ExecPolicy 规则

部分版本提供实验性的 `codex execpolicy`，可在保存规则前检查命令将被允许、要求确认还是阻止。是否可用以本机 `codex --help` 为准：

```bash
codex execpolicy check --rules ~/.codex/rules/default.rules --pretty -- git status
```

### CI 安全要点

- 优先使用 `openai/codex-action`，而非直接传递 API key。
- 在 GitHub Actions 中，执行 Codex 的 job 仅授予 `contents: read` 权限。
- 不要将 `OPENAI_API_KEY` 或 `CODEX_API_KEY` 设为 job 级别的环境变量。

## 命令速查表

| 子命令 | 说明 | 成熟度 |
|--------|------|--------|
| `codex` | 启动交互 TUI | stable |
| `codex exec` | 非交互执行 | stable |
| `codex review` | 非交互代码审查 | stable |
| `codex login` | 登录认证 | stable |
| `codex logout` | 清除凭证 | stable |
| `codex login status` | 查询登录状态 | stable |
| `codex doctor` | 诊断检查 | stable |
| `codex mcp` | 管理 MCP 服务器 | stable |
| `codex mcp-server` | 以 MCP 服务端模式启动 | stable |
| `codex plugin` | 管理插件 | stable |
| `codex resume` | 恢复交互会话 | stable |
| `codex fork` | 分叉会话 | stable |
| `codex archive` / `unarchive` | 归档/恢复会话 | stable |
| `codex delete` | 删除会话 | stable |
| `codex apply` | 将 Codex 最近生成的 diff 应用到本地工作树 | stable |
| `codex features` | 管理特性开关 | stable |
| `codex completion` | 生成 shell 补全 | stable |
| `codex update` | 检查并更新 | stable |
| `codex sandbox` | 在沙箱中执行命令 | stable |
| `codex app` | 启动桌面应用 | stable |
| `codex app-server` | 启动应用服务 | experimental |
| `codex remote-control` | 远程控制 | experimental |
| `codex cloud` | 云聊天管理 | experimental |
| `codex execpolicy` | 执行策略检查 | experimental |

## 官方资料链接

- CLI 参考文档：<https://developers.openai.com/codex/cli/reference/>
- 斜杠命令列表：<https://developers.openai.com/codex/cli/slash-commands/>
- 非交互模式：<https://developers.openai.com/codex/noninteractive/>
- Config 配置参考：<https://learn.chatgpt.com/docs/config-file/config-reference>
- GitHub Action：<https://github.com/openai/codex-action>
- CI/CD 认证指南：<https://learn.chatgpt.com/docs/auth/ci-cd-auth>
- 特性成熟度说明：<https://learn.chatgpt.com/docs/feature-maturity>
