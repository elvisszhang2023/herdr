# Herdr 用户指南

## 1. Herdr 是什么

Herdr 是一个面向 AI 编程智能体的终端工作空间管理器。

它不是 AI 模型本身，也不是完整的 IDE，而是负责管理多个终端、项目和 coding agent。你可以在同一个界面里运行 Claude Code、Codex、Cursor Agent、OpenCode 等工具，并随时查看哪些任务正在运行、已经完成，或正在等待你的处理。

Herdr 的核心模型是：

```text
Workspace
├── Tab
│   ├── Pane
│   └── Pane
└── Tab
    └── Pane
```

- **Workspace（工作区）**：通常对应一个项目或任务。
- **Tab（标签页）**：工作区中的不同工作视角，例如 `agents`、`tests`、`logs`。
- **Pane（窗格）**：一个真实的终端，可以运行 shell、服务器、测试或 AI agent。

## 2. 启动 Herdr

进入项目目录后运行：

```bash
cd ~/my-project
herdr
```

Herdr 会自动启动或连接到默认后台会话。第一次使用时，如果没有工作区，它会自动创建一个。

## 3. 拆分和管理终端

Herdr 的 Pane 是真实终端，可以直接运行：

```bash
claude
codex
npm run dev
cargo test
git status
```

常用快捷键如下。默认前缀键是 `Ctrl+B`：

| 操作 | 快捷键 |
| --- | --- |
| 向右拆分 Pane | `Ctrl+B`，然后按 `V` |
| 向下拆分 Pane | `Ctrl+B`，然后按 `-` |
| 新建 Tab | `Ctrl+B`，然后按 `C` |
| 下一个 Tab | `Ctrl+B`，然后按 `N` |
| 上一个 Tab | `Ctrl+B`，然后按 `P` |
| 切换 Workspace | `Ctrl+B`，然后按 `W` |
| 新建 Workspace | `Ctrl+B`，然后按 `Shift+N` |
| 查看快捷键 | `Ctrl+B`，然后按 `?` |
| 分离当前客户端 | `Ctrl+B`，然后按 `Q` |

Herdr 也支持鼠标操作：可以点击 Pane、Tab 和 Workspace，拖动分割线调整大小，并使用右键菜单执行常用操作。

## 4. 同时运行多个 AI agent

一个典型的工作区可以这样组织：

```text
Workspace: my-project

Tab: agents
├── Claude Code：实现登录功能
├── Codex：编写测试
└── OpenCode：检查数据库迁移

Tab: dev
├── npm run dev
└── docker compose logs

Tab: review
└── git diff
```

Herdr 支持或识别的 agent 包括：

- Claude Code
- Codex
- Cursor Agent
- OpenCode
- GitHub Copilot CLI
- Pi
- Droid
- Devin CLI
- Gemini CLI
- Kimi Code
- Qwen Code
- Grok CLI
- Kilo Code
- Cline

其他命令行程序也可以照常运行，只是 Herdr 可能不会识别它们的 agent 身份和运行状态。

## 5. 查看 agent 状态

Herdr 会尝试识别每个 agent 的当前状态：

| 状态 | 含义 |
| --- | --- |
| `working` | agent 正在工作 |
| `blocked` | agent 需要确认、回答问题或授权 |
| `done` | agent 已经完成，但你还没有查看 |
| `idle` | agent 空闲，可以继续输入 |
| `unknown` | Herdr 暂时无法确定状态 |

状态会从 Pane 汇总到 Tab，再汇总到 Workspace。因此你可以先从侧边栏判断哪个项目需要关注，再跳转到具体 Pane。

需要注意：`blocked` 检测依赖 agent 的屏幕内容或集成上报。遇到不熟悉的新提示时，建议先查看 Pane，再发送输入或确认操作。

## 6. 分离和重新连接

按下：

```text
Ctrl+B Q
```

只会关闭当前 Herdr 客户端，不会停止后台服务器和 Pane 中的程序。之后重新运行：

```bash
herdr
```

即可连接回原来的会话。

这适合以下场景：

- 关闭终端窗口后让 agent 继续工作
- SSH 断线后重新连接
- 暂时离开电脑，但保留开发服务器和测试任务
- 在不同终端之间切换同一个工作会话

如果确实要停止整个会话和其中的 Pane：

```bash
herdr server stop
```

## 7. 远程开发

### 7.1 通过 SSH 运行

最简单的方式是先连接服务器，再在服务器上运行 Herdr：

```bash
ssh user@server
herdr
```

此时代码、agent、Herdr 服务器都运行在远程机器上。手机或平板也可以使用 SSH 客户端连接后运行 Herdr。

### 7.2 从本地连接远程 Herdr

也可以在本地直接连接远程机器：

```bash
herdr --remote workbox
```

这种方式下，agent 仍然运行在远程服务器，但 Herdr 的界面显示在本地终端中。本地客户端还可以桥接部分本地能力，例如剪贴板图片粘贴。

推荐先把主机写进 SSH 配置：

```text
Host workbox
  HostName server.example.com
  User user
  Port 2222
```

然后运行：

```bash
herdr --remote workbox
```

## 8. 管理多台机器

可以保存多个远程主机：

```bash
herdr machine add workbox --label "工作服务器"
herdr machine add homelab --label "家用服务器"
```

这样可以在同一个 Herdr 界面中查看本地项目和多个远程项目，每台机器的连接和会话可以独立管理。

## 9. 使用 CLI 自动化

Herdr 提供了 CLI，可以从脚本或其他 agent 控制运行中的会话：

```bash
herdr workspace list
herdr pane list
herdr pane read w1:p2 --source recent --lines 50
herdr pane split w1:p1 --direction right
herdr agent list
herdr agent wait w1:p1 --until done
```

常见用途包括：

- 创建和关闭 Workspace、Tab、Pane
- 读取 Pane 的输出
- 向 Pane 或 agent 发送输入
- 查询 agent 状态
- 等待 agent 完成
- 自动创建固定的项目布局
- 让多个 agent 按顺序协作

例如，可以让脚本等待一个 agent 完成后再启动下一个任务：

```bash
herdr agent wait w1:p1 --until done
```

## 10. Socket API 和 agent 协作

除了 CLI，Herdr 还提供本地 Socket API。它适合：

- 自定义自动化工具
- 长时间监听事件
- 外部程序控制 Workspace 和 Pane
- 让 coding agent 创建其他 Pane 或等待其他 agent

查看当前版本的 API Schema：

```bash
herdr api schema
herdr api schema --json
```

一个典型的协作流程是：

```text
Agent A 完成代码
        ↓
Agent B 运行测试
        ↓
Agent C 检查代码
        ↓
人工处理需要确认的结果
```

## 11. 集成和会话恢复

部分 agent 提供 Herdr 集成，可以让状态识别和会话恢复更加准确。例如：

```bash
herdr integration install claude
herdr integration install codex
```

Herdr 可以保存和恢复：

- Workspace 和 Tab 布局
- Pane 信息
- 部分终端历史
- 支持集成的 agent 会话

机器或服务器完全重启后，原来的进程本身不会继续存在；对于支持恢复的 agent，Herdr 会根据集成重新启动或恢复会话。

## 12. 插件

Herdr 支持插件，用来扩展工作流。插件可以实现：

- 自定义快捷操作
- 自动创建项目布局
- 打开额外 Pane
- 响应 Workspace 或 agent 事件
- 发送通知
- 处理特定链接
- 集成外部工具

插件可以使用 Bash、JavaScript、Python、Rust、Go、Lua 或其他可执行程序编写。

插件本质上是在本机以当前用户权限运行的程序，因此只应安装可信来源的插件。

## 13. 有没有 App 或 Web 版本

目前 Herdr 的官方客户端是终端里的 TUI，不是传统桌面 GUI。现在没有官方的：

- macOS、Windows、Linux 桌面 App
- iOS、Android 原生 App
- 浏览器 Web 控制台

不过可以通过手机 SSH 客户端使用 Herdr：

```bash
ssh user@server
herdr
```

`herdr.dev` 主要是官网和文档，不是 Web 管理面板。

## 14. 适合谁使用

Herdr 特别适合：

- 同时运行多个 AI 编程 agent
- 需要长时间运行远程开发任务
- 经常通过 SSH 工作
- 想集中查看多个项目的 agent 状态
- 希望断开终端后任务继续运行
- 想通过脚本自动化终端和 agent
- 不想使用 Electron 或重量级桌面 IDE

## 15. 一句话总结

Herdr 可以概括为：

> 终端负责运行程序，Herdr 负责组织终端；AI agent 负责写代码，Herdr 负责让你看见并管理它们。

