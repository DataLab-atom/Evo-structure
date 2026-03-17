# Evo-structure Plugin — Git-Native MCTS 结构化代码搜索引擎

Evo-structure 是基于论文 **"[论文标题待更新]"** 的工程实现。它将 Monte Carlo Tree Search（MCTS）Beam Search 改造为 **git-native 异步搜索引擎**，通过 LLM 驱动的**结构算子**，在任意 git 仓库上自动搜索最优的代码结构变体，追求更优的 benchmark 表现。

> **论文引用：** *(链接待更新)*

## 核心特性

与 Evo-anything 对标的结构演化引擎，专注于**代码结构空间**的 MCTS 搜索：

| 特性 | 说明 |
|------|------|
| **git-native 搜索节点** | 每个搜索节点 = git branch，`ancestor_patches` 消失，变成 `git log` |
| **持久化状态机** | mcts-engine MCP Server，崩溃可恢复，进程无关 |
| **异步 Human Gate** | 同步阻塞 → 消息推送 + cron 超时 auto-continue，手机上决策 |
| **跨 run 记忆系统** | 跨 run op 经验，避免重复失败方向 |
| **9 种结构算子** | insert / merge / decouple / split / extract / parallelize / pipeline / stratify / cache |

## 安装

### 前置条件

**必需：**
- Python >= 3.11
- Git

**可选（安装后自动启用增强能力）：**
- `oracle` CLI — MapAgent 整仓库上下文分析（`npm install -g oracle`）
- `claude` CLI — ComboAgent 复杂结构重写，用 Claude Code 代替直接 edit
- `codex` CLI — ComboAgent 复杂变体生成的备选
- `lobster` CLI — 原子化 setup 工作流 + PR approval gate
- `tmux` — 长时间 benchmark 非阻塞后台执行
- `pyflakes` — 变体提交前 import/name 静态检查（`pip install pyflakes`）
- OpenClaw skills: `canvas`、`session-logs`（通过 `clawhub install <slug>` 安装）

### 方式一：npm 一键安装（推荐）

```bash
npm install -g evo-structure
```

安装过程中会自动调用 `pip install` 完成 Python MCP server 的安装。

安装完成后，运行 setup 配置你的 AI IDE：

```bash
# 配置所有支持的平台（Claude Code、Cursor、Windsurf、OpenClaw）
npx evo-structure setup

# 或只配置指定平台
npx evo-structure setup --platform claude
npx evo-structure setup --platform cursor
npx evo-structure setup --platform windsurf
npx evo-structure setup --platform openclaw
```

---

### 方式二：手动安装

#### 通用步骤：安装 mcts-engine

无论使用哪个平台，都需要先安装 MCP server：

```bash
git clone https://github.com/DataLab-atom/Evo-structure.git
cd Evo-structure/plugin/mcts-engine
pip install .
```

---

### OpenClaw

<details>
<summary>CLI 一键安装（推荐）</summary>

```bash
openclaw plugins install openclaw-evo-structure
openclaw gateway restart
openclaw plugins doctor   # 验证
```

</details>

<details>
<summary>本地开发模式</summary>

```bash
openclaw plugins install -l ./plugin
openclaw gateway restart
```

</details>

<details>
<summary>手动安装</summary>

将插件复制到扩展目录，并在 `~/.openclaw/openclaw.json` 中注册：

```bash
cp -r plugin/ ~/.openclaw/extensions/openclaw-evo-structure/
```

```json
{
  "plugins": {
    "entries": {
      "openclaw-evo-structure": {
        "enabled": true,
        "config": {}
      }
    }
  },
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "args": [],
      "env": {}
    }
  }
}
```

```bash
openclaw gateway restart
```

</details>

**验证：** 对话中输入 `/mcts-status`，看到 "Search not initialized" 即安装成功。

---

### Claude Code

在项目根目录或全局 `.claude/settings.json` 中添加 MCP server：

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

将 skills 链接到 Claude Code：

```bash
ln -s $(pwd)/plugin/skills/* ~/.claude/skills/
```

重启 Claude Code 即可使用。

---

### Cursor

在项目根目录的 `.cursor/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

Cursor 会自动发现 MCP tools（`mcts_init`、`mcts_step` 等）。将 Agent 协议导入 Cursor Rules：

```bash
cp plugin/AGENTS.md .cursor/rules/evo-structure-agents.md
```

---

### Windsurf

在全局 `~/.codeium/windsurf/mcp_config.json` 中添加：

```json
{
  "mcpServers": {
    "mcts-engine": {
      "command": "mcts-engine",
      "type": "stdio"
    }
  }
}
```

---

### 其它 MCP 兼容客户端

Evo-structure 的核心是一个标准 [MCP](https://modelcontextprotocol.io) server。任何支持 MCP stdio 传输的客户端都可以接入：

```bash
# 直接启动 server（stdio 模式）
mcts-engine
```

提供的 MCP tools：`mcts_init`、`mcts_register_targets`、`mcts_step`、`mcts_check_cache`、`mcts_get_lineage`、`mcts_get_status`、`mcts_freeze_branch`、`mcts_boost_branch`、`mcts_record_synergy`。

---

### 可选配置

搜索状态默认存储在 `~/.openclaw/mcts-state/`，可通过环境变量自定义：

```bash
export COG_STATE_DIR=/path/to/your/state
```

或在 OpenClaw 中通过 `openclaw.json` 配置：

```json
{
  "plugins": {
    "entries": {
      "openclaw-evo-structure": {
        "enabled": true,
        "config": {
          "statePath": "/path/to/your/state"
        }
      }
    }
  }
}
```

## Quick Start

```
你在 Telegram 发：帮我优化这个特征提取模块的速度
         ↓
  /search project_root "python main.py" 触发
         ↓
  MapAgent 分析仓库 → 识别优化目标函数
         ↓
  mcts_init → 跑基线确认能跑 → tag seed-baseline
         ↓
  MCTS 搜索循环：每代生成 beam_width 个结构变体
    ├── ComboAgent × N（并行）
    │     Critic → Engineer → AST check → git worktree → benchmark
    ├── UCB 选择最优节点
    ├── 每 gate_interval 代：推送进度到你手机
    │     （30 分钟无响应自动 continue）
    └── ReflectAgent 写记忆
         ↓
  结束后推最优分支 + 发搜索报告
```

## 工作原理

Evo-structure 将传统 MCTS Beam Search 的每个**搜索节点**映射为一个 **git branch**，彻底消除内存状态依赖：

| MCTS 概念 | git 映射 | 优势 |
|----------|---------|------|
| search node | git branch + commit | 天然持久化，崩溃可恢复 |
| root node | `seed-baseline` tag | 不可变基准点 |
| ancestor_patches | `git log`（commit 链 diff）| 自动记录，无需手工维护 |
| score | commit message | 随 commit 持久化 |
| tried_combos | branch 命名（可枚举）| `git branch --list 'mcts/*/gen-N/*'` |
| rollback | `git checkout` 父 branch | 天然支持，无副作用 |
| apply best | `git checkout best-overall` | 幂等、可 undo |
| tree structure | `git log --graph --all` | 完整谱系图 |
| 并行评估 | git worktree | 真正隔离 |

### 9 种结构算子

| 类型 | 算子 | 语义 |
|------|------|------|
| 二元 | `insert` | 在 A→B 间插入新节点 C |
| 二元 | `merge` | 合并 A→B 为单节点 |
| 二元 | `decouple` | 断开 A→B 的直接依赖 |
| 二元 | `split` | 将 B 拆分为 B1、B2 |
| 二元 | `extract` | 从 A→B 提取公共组件 C |
| 多元 | `parallelize` | 串行中间节点改并发 |
| 多元 | `pipeline` | 链路改流式处理 |
| 多元 | `stratify` | 跨层依赖重整为有序层级 |
| 单元 | `cache` | 对热路径引入缓存层 |

### 搜索循环

每代 MCTS 循环包含六个阶段：

1. **规划** — `mcts_step("begin_generation")`，UCB 选出 frontier，分配 beam_width 个 op 任务
2. **生成** — ComboAgent 并行：Critic → Engineer → 静态检查 → git commit
3. **评估** — git worktree 隔离 → 跑 benchmark → 上报 fitness
4. **选择** — `mcts_step("select")`，UCB 保留 top-K，清理落选分支
5. **人工 Gate** — 每 N 代推送搜索树快照到手机，等待决策（超时自动 continue）
6. **反思** — ReflectAgent 提取本代经验写入结构化记忆

### 与 Evo-anything 的对比

| 维度 | Evo-anything | Evo-structure |
|------|-------------|---------------|
| 搜索范围 | 全局多目标演化 | 树形 MCTS 搜索 |
| 核心映射 | individual → git branch | search node → git branch |
| 状态机 | evo-engine MCP Server | mcts-engine MCP Server |
| 选择策略 | 种群选择 + Synergy | UCB + Beam Search |
| 人机交互 | 进度通知 | **Async Human Gate**（每代决策） |
| 记忆系统 | per-target + global | per-op + per-project |

**关键差异**：Async Human Gate 是 Evo-structure 独有的——让 MCTS 从"必须人守着跑"变成"手机上睡前看一眼决定"。

## Skills

| 命令 | 说明 |
|------|------|
| `/search <project_path> <benchmark_cmd>` | 完整搜索入口：初始化 + 跑基线 + 启动 MCTS 循环 |
| `/mcts-status` | 查看当前搜索进度、树状态、best score |
| `/mcts-report` | 生成搜索报告（谱系 + op 分析 + 改动 diff）|
| `/mcts-stop` | 停止搜索，应用当前最优节点 |
| `/mcts-rollback` | 回退到上一代 frontier |
| `/boost <branch>` | 提升指定分支的 UCB 优先级 |
| `/freeze <branch>` | 冻结指定分支，停止从它扩展 |

## 目录结构

```
Evo-structure/
├── LICENSE
├── README.md
├── README_EN.md
├── research/                      # 设计文档与分析
│   └── mcts-engine/
│       ├── README.md              # 文档索引
│       ├── 01_current_analysis.md # 原 MCTS 实现深度分析
│       ├── 02_platform_mapping.md # 平台能力映射
│       └── DESIGN.md              # 完整设计文档
└── plugin/
    ├── openclaw.plugin.json       # 插件定义
    ├── AGENTS.md                  # 搜索协议（核心循环）
    ├── SOUL.md                    # Agent 人格设定
    ├── TOOLS.md                   # 工具使用约定
    ├── agents/                    # 各 Agent 行为说明
    │   ├── orchestrator.md        # OrchestratorAgent（含 canvas 可视化）
    │   ├── combo_agent.md         # ComboAgent（含静态检查、tmux、coding-agent）
    │   ├── gate_agent.md          # GateAgent（异步 Human Gate）
    │   ├── policy_agent.md        # PolicyAgent
    │   ├── reflect_agent.md       # ReflectAgent（含跨-run 记忆）
    │   └── map_agent.md           # MapAgent（含 oracle 整仓库分析）
    ├── mcts-engine/               # 搜索引擎（MCP server）
    │   ├── server.py              # MCP 工具接口 + 状态机
    │   ├── models.py              # 数据模型
    │   └── selection.py           # UCB 选择算法
    ├── skills/                    # 用户可调用的技能
    │   ├── search/                # 启动搜索
    │   ├── mcts-status/           # 查看进度
    │   ├── mcts-report/           # 生成报告
    │   ├── mcts-stop/             # 停止搜索
    │   ├── mcts-rollback/         # 回退
    │   ├── boost/                 # 提升优先级
    │   └── freeze/                # 冻结节点
    └── workflows/                 # Lobster 声明式工作流
        ├── mcts-setup.lobster     # 原子化 setup（validate→baseline→tag→mkdir）
        └── mcts-finish.lobster    # 结束流程（tag→push→approval gate→PR）
```

## 搜索记忆

Evo-structure 在目标仓库中维护结构化记忆，避免重复探索失败方向：

```
memory/
├── global/
│   ├── long_term.md              # 跨项目经验
│   └── op_lessons.md            # 跨项目 op 经验汇总
├── projects/{project_hash}/
│   ├── long_term.md              # 该项目累积经验
│   └── runs/{run_id}/
│       ├── gen_{N}.md            # 每代反思
│       ├── tree_final.md         # 最终搜索树快照
│       └── best_diff.md          # 最优节点 vs baseline diff 摘要
└── ops/
    ├── insert.md                 # insert 全局成功/失败记录
    ├── merge.md
    └── ...（其余算子）
```

## 分支命名

```
mcts/{run_id}/gen-{N}/{op}-{uuid8}           # 标准搜索节点
mcts/{run_id}/synergy/{opA}+{opB}-{uuid8}   # 跨算子组合（Synergy）
```

Tags: `seed-baseline`, `best-gen-{N}`, `best-mcts-{run_id}`, `best-overall`

Commit message 格式（机器可解析）：
```
mcts(score=0.8423,op=insert,gen=3,run=a3f9b2): insert cache layer between loader→extractor
```
