## 📋 你安装的插件总览

|插件|状态|功能|版本|
|---|---|---|---|
|**ida-pro-mcp**|✅ 已启用|IDA Pro 逆向工程分析|0.1.0|
|**ponytail**|❌ 已禁用|最小化代码编写（YAGNI 原则）|4.8.4|
|**ecc**|❌ 已禁用|Agent 框架和工程化系统|2.0.0-rc.1|
|**claude-hud**|❌ 已禁用|Claude Code 执行情况监控|0.0.12|

---

## 🔍 各插件详细说明

### 1. **ida-pro-mcp** ✅（已启用）

**功能：** IDA Pro 逆向工程的 AI 助手

**能做什么：**

- 与 IDA Pro 集成，分析二进制文件、反汇编代码、伪代码
- 使用 `decompile()` 查看伪 C 代码
- 使用 `disasm()` 查看汇编指令
- `xrefs_to()` 找所有交叉引用
- 使用 MCP 工具进行 APK 分析、Smali 反编译等
- 自动化逆向工程工作流

**核心工具：**

```
- lookup_funcs()      # 按名字/地址找函数
- decompile()        # 反编译伪代码
- disasm()          # 反汇编
- xrefs_to()        # 交叉引用分析
- int_convert()     # 数字格式转换
- set_comments()    # 添加注释
- patch_asm()       # 修改汇编
```

**使用方式：**

```
# 在 Claude Code 中直接使用（已启用）
# 询问 Claude 进行 IDA Pro 分析：
"分析这个二进制文件的 main 函数"
```

**前置条件：**

- 安装了 IDA Pro 9.3+（Pro 版，不支持免费版）
- 激活 idalib：
    
    ```bash
    # macOSuv run "/Applications/IDA Professional 9.3.app/Contents/MacOS/idalib/python/py-activate-idalib.py"
    ```
    

---

### 2. **ponytail** ❌（已禁用）

**功能：** 写最少的代码（「最懒的高级开发」风格）

**能做什么：**

- 避免过度设计：在决策阶梯前停下
    1. 这个需要存在吗？→ 不需要：跳过（YAGNI）
    2. 代码库里已经有了？→ 复用，别重写
    3. 标准库有吗？→ 用它
    4. 原生特性有吗？→ 用它
    5. 已装的依赖？→ 用它
    6. 一行代码搞定？→ 一行
    7. 才写最小化代码

**效果：**

- 减少 **54% 代码量**
- 降低 **20% 成本**
- 快 **27%**
- 保持 100% 安全

**使用方式：**

```
# 启用后自动在每次代码生成前应用
/ponytail full        # 启用完全模式
/ponytail lite        # 启用轻量模式
/ponytail ultra       # 启用激进模式（删除更多）
/ponytail review      # 审查当前代码是否过度设计
```

**启用步骤：**

```bash
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

---

### 3. **ecc** ❌（已禁用）

**功能：** Agent 工程框架（规则、技能、钩子）

**能做什么：**

- **67 个预制 Agent**：架构师、代码审查、TDD 指导、构建错误修复等
- **268+ 技能**：编码标准、后端模式、前端模式、Django/Spring Boot、安全扫描
- **12+ 语言支持**：TypeScript、Python、Go、Rust、Java、Kotlin、C++、PHP 等
- **Hook 系统**：自动化钩子（Pre/Post 工具执行、Session 开始/结束）
- **规则引擎**：语言特定的最佳实践
- **持续学习**：自动从 Session 提取模式

**核心命令：**

```
/plan              # 功能规划
/code-review       # 代码审查
/build-fix         # 修复构建错误
/refactor-clean    # 清理死代码
/tdd-workflow      # TDD 工作流
/security-scan     # 安全审计
```

**推荐用法：**

```bash
# 启用
/plugin marketplace add https://github.com/affaan-m/ECC
/plugin install ecc@ecc

# 或手动安装规则
git clone https://github.com/affaan-m/ECC.git
mkdir -p ~/.claude/rules/ecc
cp -r ECC/rules/common ~/.claude/rules/ecc/
cp -r ECC/rules/typescript ~/.claude/rules/ecc/  # 选你需要的语言
```

---

### 4. **claude-hud** ❌（已禁用）

**功能：** Claude Code 执行情况仪表盘

**能做什么：**

- **实时监控：** Context 使用率（绿→黄→红）
- **速率限制：** 显示 5 小时/7 天用量百分比
- **工具活动：** 显示正在运行的 Tool（Read, Edit, Bash 等）
- **Agent 状态：** 显示哪个 subagent 在跑什么
- **Todo 进度：** 任务完成情况追踪
- **Git 状态：** 分支、未提交更改数

**显示示例：**

```
[Opus] │ my-project git:(main*)
Context ████░░░░░░ 45% │ Usage ██░░░░░░░░ 25% (1h 30m / 5h)
◐ Edit: auth.ts | ✓ Read ×3 | ✓ Grep ×2
◐ explore [haiku]: Finding auth code (2m 15s)
▸ Fix authentication bug (2/5)
```

**启用步骤：**

```bash
/plugin marketplace add jarrodwatts/claude-hud
/plugin install claude-hud
/reload-plugins
/claude-hud:setup
```

---

## ⚙️ 如何设置默认调用

### 方式 1：启用插件

```bash
# 进入 Claude Code，运行
/plugin install ecc@ecc           # 启用 ECC
/plugin install ponytail@ponytail # 启用 Ponytail
/plugin install claude-hud        # 启用 HUD
```

### 方式 2：编辑配置文件

编辑 `~/.claude/settings.json`：

```json
{
  "enabledPlugins": {
    "ida-pro-mcp@mrexodia": true,
    "ecc@ecc": true,
    "ponytail@ponytail": true,
    "claude-hud": true
  }
}
```

### 方式 3：环境变量控制

**Ponytail 默认强度：**

```bash
export PONYTAIL_DEFAULT_MODE=full  # lite | full | ultra | off
```

**ECC Hook 配置：**

```bash
export ECC_HOOK_PROFILE=standard   # minimal | standard | strict
export ECC_SESSION_START_CONTEXT=off  # 低上下文环境禁用
```

**Claude HUD 禁用：**

```bash
export CLAUDE_HUD_DISABLE=1  # 禁用 HUD（不删除配置）
```

---

## 🎯 推荐用法

### 针对 Android 逆向工程（你的背景）

**启用组合：**

```bash
/plugin install ida-pro-mcp@mrexodia     # 逆向分析
/plugin install ecc@ecc                  # Agent 框架 + 技能库
/plugin install claude-hud               # 监控执行
```

**工作流示例：**

```
1. 用 ida-pro-mcp decompile APK 的关键函数
2. 用 ECC 的 /security-scan 进行安全审计
3. 用 claude-hud 监控 Context 用量（防止爆满）
4. 用 ponytail 避免生成过度优化的分析脚本
```

### 日常编码（推荐）

启用这个组合：

```bash
# 核心三件套
/plugin install ecc@ecc        # 规则 + Agent
/plugin install claude-hud     # 监控
/plugin install ponytail       # 最小化代码

# 可选
/plugin install ida-pro-mcp    # 如需要逆向
```

---

## 🔗 快速参考

|需求|用哪个|命令|
|---|---|---|
|规划新功能|ECC|`/plan "..."`|
|代码审查|ECC|`/code-review`|
|安全审计|ECC|`/security-scan`|
|看 Context 用量|claude-hud|自动显示在底部|
|最小化代码|ponytail|`/ponytail full`|
|逆向 APK|ida-pro-mcp|自动加载（已启用）|
|TDD 工作流|ECC|`tdd-workflow` 技能|

---

需要我帮你启用某个插件或配置特定功能吗？