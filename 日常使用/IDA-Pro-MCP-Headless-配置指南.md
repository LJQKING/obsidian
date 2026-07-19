# IDA Pro MCP Headless 模式配置指南

## 概述

IDA Pro MCP Server 通过 MCP (Model Context Protocol) 桥接 IDA Pro 与 AI 语言模型，支持在 Claude Code 中进行无头二进制分析（vibe reversing）。

---

## 前置条件

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| Python | 3.11+ | 通过 `uv python install 3.11` 安装 |
| uv | 最新版 | 包管理器，`curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| IDA Pro | 9.x (推荐 9.3) | 仅专业版支持，免费版不可用 |
| idalib | 随 IDA Pro 安装 | 通过 Python 激活脚本启用 |

---

## 安装步骤

### 1. 安装 uv 包管理器

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. 安装 Python 3.11

```bash
uv python install 3.11
```

### 3. 添加 mrexodia marketplace

```bash
claude plugin marketplace add mrexodia/claude-marketplace
```

### 4. 安装 ida-pro-mcp 插件

```bash
claude plugin install ida-pro-mcp@mrexodia
```

### 5. 配置项目 Python 版本

```bash
uv python pin 3.11 \
  --directory ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0
```

### 6. 安装项目依赖

```bash
uv sync --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0
```

### 7. 配置 MCP Server

在 `~/.claude/mcp-configs/mcp-servers.json` 中添加：

```json
{
  "mcpServers": {
    "idalib": {
      "command": "uv",
      "args": [
        "run",
        "--project",
        "/Users/jialiu/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0",
        "idalib-mcp",
        "--stdio"
      ],
      "description": "IDA Pro MCP server for headless binary analysis via idalib"
    }
  }
}
```

### 8. 重启 Claude Code 会话

MCP 配置修改后需要新建会话才能生效。

---

## 使用方式

### 方式一：通过 Claude Code 插件自动连接（推荐）

配置完成后，Claude Code 会自动加载 idalib MCP server，可直接使用以下工具：

| 工具 | 功能 |
|------|------|
| `decompile(address)` | 反编译指定地址的函数 |
| `disasm(address)` | 反汇编指定地址 |
| `lookup_funcs(query)` | 按名称或地址查找函数 |
| `int_convert(input)` | 数字进制转换 |
| `idb_open(path, session_id)` | 打开数据库会话 |

### 方式二：手动启动 idalib-mcp

#### Stdio 模式（直接分析二进制文件）

```bash
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  idalib-mcp --stdio /path/to/binary
```

#### HTTP 服务模式（持久化连接）

```bash
# 带初始二进制
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  idalib-mcp --host 127.0.0.1 --port 8745 /path/to/binary

# 不带初始二进制（后续通过 idb_open 打开）
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  idalib-mcp --host 127.0.0.1 --port 8745
```

#### SSE 远程模式

```bash
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  ida-pro-mcp --transport http://127.0.0.1:8744/sse
```

---

## Headless 模式工作流程

```
1. idalib-mcp 启动（监听端口或 stdio）
2. 通过 MCP 工具打开数据库：
   idb_open("/path/to/binary", preferred_session_id="my_session")
3. 执行分析操作：
   decompile("main", database="my_session")
   disasm(0x401000, database="my_session")
4. 分析完成后 worker 空闲超时退出（默认 1 小时）
```

> **注意**：Headless 模式下没有 `idb_close` 工具，worker 会在空闲超时后自动退出。

---

## 测试与验证

### 验证插件安装

```bash
claude plugin list
# 应显示 ida-pro-mcp@mrexodia 状态为 enabled
```

### 验证 idalib-mcp 可用

```bash
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  idalib-mcp --help
```

### 运行内置测试

```bash
# 使用测试固件
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  ida-mcp-test tests/crackme03.elf -q

# 按模块测试
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  ida-mcp-test tests/crackme03.elf -c api_analysis
```

### 覆盖率测试

```bash
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  coverage erase
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  coverage run -m ida_pro_mcp.test tests/crackme03.elf -q
uv run --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0 \
  coverage report --show-missing
```

---

## 关键文件索引

| 文件 | 路径 |
|------|------|
| 插件缓存 | `~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0/` |
| 项目配置 | `pyproject.toml` |
| MCP 入口 | `src/ida_pro_mcp/server.py` |
| Headless 入口 | `src/ida_pro_mcp/idalib_server.py` |
| 分析 API | `src/ida_pro_mcp/ida_mcp/api_analysis.py` |
| 内存 API | `src/ida_pro_mcp/ida_mcp/api_memory.py` |
| 类型 API | `src/ida_pro_mcp/ida_mcp/api_types.py` |
| 修改 API | `src/ida_pro_mcp/ida_mcp/api_modify.py` |
| 测试固件 | `tests/crackme03.elf`, `tests/typed_fixture.elf` |

---

## 常见问题

### Q: idalib 激活失败

确保 IDA Pro 已正确安装，并运行激活脚本：

```bash
uv run "/Applications/IDA Professional 9.3.app/Contents/MacOS/idalib/python/py-activate-idalib.py"
```

### Q: Python 版本不兼容

ida-pro-mcp 要求 Python 3.11+，使用 `uv` 管理版本：

```bash
uv python install 3.11
uv python pin 3.11 --project <project_dir>
```

### Q: MCP 配置不生效

1. 检查 `~/.claude/mcp-configs/mcp-servers.json` 格式是否正确
2. 重启 Claude Code 会话（新建会话）
3. 确认 `claude plugin list` 中 ida-pro-mcp 状态为 enabled

### Q: 构建或测试失败

```bash
# 重新同步依赖
uv sync --project ~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0
```

---

## 参考链接

- GitHub 仓库：https://github.com/mrexodia/ida-pro-mcp
- MCP 协议文档：https://modelcontextprotocol.io/introduction
- IDA Pro 官网：https://www.hex-rays.com/ida-pro/
- uv 文档：https://docs.astral.sh/uv/
