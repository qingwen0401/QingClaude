# QingClaude

> 🚀 基于 [KamaClaude](https://github.com/youngyangyang04/KamaClaude) 的个人学习、源码阅读、实验与持续优化项目。
>
> 沿着项目的 Stage 路线，**从环境搭建 → 协议 → Agent Loop → 工具 → 事件 → 会话 → 安全 → 上下文 → MCP，
> 一步一步理解一个 Agent Runtime 是如何构建出来的。**

<p align="center">

![Python](https://img.shields.io/badge/Python-3.12-blue)
![Platform](https://img.shields.io/badge/Platform-Windows%20%2B%20WSL2-orange)
![Stage](https://img.shields.io/badge/Current%20Stage-S0-success)
![Learning](https://img.shields.io/badge/Learning-In%20Progress-yellow)

</p>

---
<div align="center">

<a href="#">
  <img src="https://img.shields.io/badge/%E7%8E%B0%E5%B7%B2%E6%9B%B4%E6%96%B0-windows%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%8E%92%E5%9D%91%E6%8C%87%E5%8D%97%E3%80%81s0%E5%85%A5%E9%97%A8%E5%AD%A6%E4%B9%A0%E6%96%87%E6%A1%A3-ff0000?style=for-the-badge&amp;labelColor=ffff00" alt="现已更新：windows环境配置排坑指南、s0入门学习文档">
</a>

</div>

## 项目定位

这是我的 **KamaClaude 源码学习仓库**。

我会：

- 阅读源码
- 运行项目
- 分析架构
- 绘制执行链路
- 编写学习笔记
- 做最小实验
- 编写 / 运行测试
- 记录遇到的问题
- 对发现的问题进行修复或优化
- 将自己的学习过程沉淀到仓库中

## KamaClaude：Windows + WSL2 配置与排坑总结

> **核心原则：先定位故障层级，再决定是否修改源码。**

### 一、环境搭建

**推荐路径**：将项目放在 WSL Ubuntu 的 Linux 文件系统中，避免同时引入 Windows 挂载路径带来的变量。

```bash
cd ~/Projects/KamaClaude
uv sync
cp .env.example .env
uv run kama --version
```

`.env` 关键配置：

```dotenv
KAMA_HOST=127.0.0.1
KAMA_PORT=7437
ANTHROPIC_API_KEY=你的_API_KEY
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
KAMA_LLM_DEFAULT_MODEL=deepseek-v4-flash
KAMA_MAX_STEPS=20
```

> **安全提醒**：API Key 不得提交到 Git、README 或 GitHub；如已泄露，应立即撤销或轮换。

### 二、主要问题与结论

| 环节 | 现象 | 排查结论 / 处理思路 |
| --- | --- | --- |
| WSL 代理 | 提示「NAT 模式下的 WSL 不支持 localhost 代理」 | 属于 Windows ↔ WSL 的网络/代理链路问题，优先检查代理可达性。 |
| 安装 `uv` | 安装脚本报 `curl: (35) TLS ... unexpected eof`；直连 GitHub Release 下载停在 `0 bytes` | 重点排查 WSL 代理、HTTPS/TLS 和外部下载源，不能直接归因于 `uv` 或项目源码。 |
| 项目目录 | 最初尝试放在 Windows D 盘 | 改用 `~/Projects/KamaClaude`，减少跨文件系统干扰。 |
| 单元测试 | 全量运行 `256 passed, 6 failed`，失败集中于 `test_compactor.py` | 单独运行该文件 `6 passed`，疑似测试顺序、事件循环生命周期或测试间状态影响；**尚不能认定为 Compactor 源码 Bug**。 |

### 三、Core / CLI 运行与验证

KamaClaude S0 采用 **CLI → TCP → Core** 的双进程模式；通信链路为 **TCP → NDJSON → JSON-RPC 2.0 → Handler**。

```bash
# 终端 1：启动 Core
uv run kama-core

# 终端 2：验证 CLI 与 Core 通信
uv run kama ping
```

测试命令：

```bash
uv run pytest tests/unit -v
uv run pytest tests/unit/test_compactor.py -v
```

曾出现的关键错误：

```text
RuntimeError: There is no current event loop in thread 'MainThread'.
```

**当前状态**：尚未通过修改源码修复该问题；由于单测单独执行通过，应先检查 pytest 的执行顺序、`asyncio` event loop 的创建/销毁，以及用例间共享状态。

### 四、排障顺序

1. **网络层**：WSL 代理、DNS、HTTPS/TLS、下载源。
2. **环境层**：Linux 文件系统、`uv` 依赖、Python 运行环境。
3. **配置层**：`.env`、API Key、服务地址和模型名。
4. **通信层**：Core 是否启动、CLI 是否通过 TCP / JSON-RPC 连通。
5. **测试/代码层**：先复现并隔离失败，再分析测试污染或实际源码缺陷。

**一句话总结**：网络、环境、配置、进程通信与测试问题要分层定位，不能把所有报错都当作 KamaClaude 代码问题。
