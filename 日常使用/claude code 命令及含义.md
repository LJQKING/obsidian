## **核心命令分类**

### **1️⃣ 会话管理**

|命令|含义|
|---|---|
|`/init`|初始化项目，生成 `CLAUDE.md` 指南|
|`/memory`|编辑和管理内存文件|
|`/clear` / `/reset`|清空对话历史，开始新任务|
|`/resume`|返回之前的对话|
|`/branch [name]`|创建对话分支，尝试不同方向|
|`/fork [prompt]`|复制当前对话到后台会话|
|`/exit` / `/quit`|退出 CLI|

### **2️⃣ 模型与性能**

|命令|含义|
|---|---|
|`/model [model]`|切换 AI 模型|
|`/effort [level\|auto]`|调整推理努力等级（低/中/高/超高/最高）|
|`/fast [on\|off]`|切换快速模式|
|`/advisor [model\|off]`|启用/禁用顾问工具|
|`/config [key=value]`|打开设置界面或直接设置|

### **3️⃣ 上下文管理**

|命令|含义|
|---|---|
|`/context [all]`|可视化当前上下文使用情况|
|`/compact [instructions]`|压缩对话历史，释放 token 空间|
|`/cd <path>`|切换工作目录|
|`/add-dir <path>`|添加额外的工作目录|
|`/btw [question]`|问快速问题而不添加到对话历史|

### **4️⃣ 代码审查与安全**

|命令|含义|
|---|---|
|`/diff`|打开交互式 diff 查看器|
|`/code-review [level] [--fix]`|审查代码质量和 bug|
|`/security-review`|检查安全漏洞|
|`/review`|快速单轮审查 GitHub PR|

### **5️⃣ 工作流与自动化**

|命令|含义|
|---|---|
|`/plan [description]`|进入计划模式|
|`/batch <instruction>`|并行编排大规模代码变更|
|`/loop [interval] [prompt]`|定时运行提示|
|`/goal [condition\|clear]`|设置目标，让 Claude 持续工作直到达成|
|`/autofix-pr [prompt]`|自动修复 PR 中的 CI 失败|

### **6️⃣ 工具与集成**

|命令|含义|
|---|---|
|`/mcp [reconnect\|enable\|disable]`|管理 MCP 服务器连接|
|`/plugin [subcommand]`|管理插件|
|`/chrome`|配置 Chrome 扩展|
|`/ide`|管理 IDE 集成|
|`/hooks`|查看 hook 配置|

### **7️⃣ 权限与安全**

|命令|含义|
|---|---|
|`/permissions`|管理工具权限规则|
|`/design-login`|授权设计系统访问|
|`/privacy-settings`|查看隐私设置|

### **8️⃣ 诊断与调试**

|命令|含义|
|---|---|
|`/doctor` / `/checkup`|运行设置检查，诊断问题|
|`/debug [description]`|启用调试日志并诊断问题|
|`/heapdump`|导出堆快照以诊断内存问题|
|`/rewind`|回滚代码和对话到检查点|

### **9️⃣ 信息与帮助**

|命令|含义|
|---|---|
|`/help`|显示帮助和可用命令|
|`/status`|显示当前状态|
|`/tasks`|列出后台任务|
|`/usage` / `/cost`|显示 token 使用情况|
|`/release-notes`|查看更新日志|
|`/insights`|生成会话分析报告|

### **🔟 高级功能**

|命令|含义|
|---|---|
|`/background [prompt]`|将会话放到后台运行|
|`/teleport`|将 web 会话拉到终端|
|`/remote-control`|启用 Remote Control 连接|
|`/copy [N]`|复制响应到剪贴板|
|`/export [filename]`|导出对话为纯文本|

### **1️⃣1️⃣ 技能与工作流**

|命令|含义|
|---|---|
|`/claude-api`|加载 Claude API 参考|
|`/dataviz [request]`|图表设计指导|
|`/design-sync [hint]`|同步设计系统到 Claude Design|
|`/deep-research <question>`|并行网络搜索和综合报告|
|`/verify`|验证代码正确性|
|`/simplify`|简化代码|
|`/fewer-permission-prompts`|减少权限提示|

### **1️⃣2️⃣ 个人化与其他**

|命令|含义|
|---|---|
|`/color [color\|default]`|设置提示条颜色|
|`/focus`|切换焦点视图|
|`/keybindings`|打开快捷键文件|
|`/login` / `/logout`|登录/登出|
|`/mobile` / `/ios` / `/android`|下载移动应用|
|`/desktop` / `/app`|在桌面应用中继续会话|
|`/radio`|打开 Claude FM 音乐|
|`/powerup`|学习 Claude Code 功能|
|`/feedback` / `/bug` / `/share`|反馈问题或分享会话|

## **快速技巧** 💡

- 输入 `/` 后跟字母可过滤命令
- 大多数命令仅在特定时间点有用（如 `/init` 在项目初始化时）
- `/model` 和 `/effort` 可立即生效，无需等待当前响应完成
- `/goal` 可让 Claude 自主工作直到达成条件
- `/batch` 用于并行处理大规模重构

需要详细了解某个特定命令吗？直接告诉我你要用的命令名称！