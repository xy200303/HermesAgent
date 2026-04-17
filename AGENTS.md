# Hermes Agent - 开发指南

面向在 `hermes-agent` 代码库中工作的 AI 编码助手与开发者的说明。

## 开发环境

```bash
source venv/bin/activate  # 运行 Python 前务必先激活
```

## 项目结构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 —— 核心对话循环
├── model_tools.py        # 工具编排、discover_builtin_tools()、handle_function_call()
├── toolsets.py           # 工具集定义，_HERMES_CORE_TOOLS 列表
├── cli.py                # HermesCLI 类 —— 交互式 CLI 协调器
├── hermes_state.py       # SessionDB —— SQLite 会话存储（FTS5 搜索）
├── agent/                # Agent 内部模块
│   ├── prompt_builder.py     # 系统提示词组装
│   ├── context_compressor.py # 自动上下文压缩
│   ├── prompt_caching.py     # Anthropic 提示缓存
│   ├── auxiliary_client.py   # 辅助 LLM 客户端（视觉、摘要）
│   ├── model_metadata.py     # 模型上下文长度、token 估算
│   ├── models_dev.py         # models.dev 注册表集成（带 provider 感知的上下文）
│   ├── display.py            # KawaiiSpinner、工具预览格式化
│   ├── skill_commands.py     # Skill 斜杠命令（CLI/gateway 共用）
│   └── trajectory.py         # 轨迹保存辅助函数
├── hermes_cli/           # CLI 子命令与初始化
│   ├── main.py           # 入口点 —— 所有 `hermes` 子命令
│   ├── config.py         # DEFAULT_CONFIG、OPTIONAL_ENV_VARS、迁移逻辑
│   ├── commands.py       # 斜杠命令定义 + SlashCommandCompleter
│   ├── callbacks.py      # 终端回调（clarify、sudo、approval）
│   ├── setup.py          # 交互式初始化向导
│   ├── skin_engine.py    # 皮肤/主题引擎 —— CLI 视觉定制
│   ├── skills_config.py  # `hermes skills` —— 按平台启用/禁用 skills
│   ├── tools_config.py   # `hermes tools` —— 按平台启用/禁用 tools
│   ├── skills_hub.py     # `/skills` 斜杠命令（搜索、浏览、安装）
│   ├── models.py         # 模型目录、provider 模型列表
│   ├── model_switch.py   # 共用的 /model 切换流程（CLI + gateway）
│   └── auth.py           # Provider 凭证解析
├── tools/                # 工具实现（每个工具一个文件）
│   ├── registry.py       # 中央工具注册表（schema、handler、dispatch）
│   ├── approval.py       # 危险命令检测
│   ├── terminal_tool.py  # 终端编排
│   ├── process_registry.py # 后台进程管理
│   ├── file_tools.py     # 文件读写/搜索/补丁
│   ├── web_tools.py      # Web 搜索/提取（Parallel + Firecrawl）
│   ├── browser_tool.py   # Browserbase 浏览器自动化
│   ├── code_execution_tool.py # execute_code 沙箱
│   ├── delegate_tool.py  # 子代理委派
│   ├── mcp_tool.py       # MCP 客户端（约 1050 行）
│   └── environments/     # 终端后端（local、docker、ssh、modal、daytona、singularity）
├── gateway/              # 消息平台网关
│   ├── run.py            # 主循环、斜杠命令、消息分发
│   ├── session.py        # SessionStore —— 对话持久化
│   └── platforms/        # 适配器：telegram、discord、slack、whatsapp、homeassistant、signal、qqbot
├── acp_adapter/          # ACP 服务器（VS Code / Zed / JetBrains 集成）
├── cron/                 # 调度器（jobs.py、scheduler.py）
├── environments/         # RL 训练环境（Atropos）
├── tests/                # Pytest 测试套件（约 3000 个测试）
└── batch_runner.py       # 并行批处理
```

**用户配置：** `~/.hermes/config.yaml`（设置），`~/.hermes/.env`（API keys）

## 文件依赖链

```
tools/registry.py  （无依赖 —— 被所有工具文件导入）
       ↑
tools/*.py  （每个文件在导入时调用 registry.register()）
       ↑
model_tools.py  （导入 tools/registry 并触发工具自动发现）
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## AIAgent 类（run_agent.py）

```python
class AIAgent:
    def __init__(self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,           # "cli"、"telegram" 等
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        # ... 以及 provider、api_mode、callbacks、routing params 等
    ): ...

    def chat(self, message: str) -> str:
        """简单接口 —— 返回最终响应字符串。"""

    def run_conversation(self, user_message: str, system_message: str = None,
                         conversation_history: list = None, task_id: str = None) -> dict:
        """完整接口 —— 返回包含 final_response + messages 的 dict。"""
```

### Agent 循环

核心循环位于 `run_conversation()` 中，完全同步执行：

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

消息遵循 OpenAI 格式：`{"role": "system/user/assistant/tool", ...}`。推理内容存储在 `assistant_msg["reasoning"]` 中。

---

## CLI 架构（cli.py）

- 使用 **Rich** 渲染横幅/面板，使用 **prompt_toolkit** 提供带自动补全的输入
- **KawaiiSpinner**（`agent/display.py`）—— API 调用期间的动画表情，以及用于显示工具结果的 `┊` 活动流
- `cli.py` 中的 `load_cli_config()` 会合并硬编码默认值与用户 YAML 配置
- **皮肤引擎**（`hermes_cli/skin_engine.py`）—— 数据驱动的 CLI 主题系统；在启动时根据 `display.skin` 配置键初始化；皮肤可自定义横幅颜色、spinner 表情/动词/翅膀、工具前缀、响应框与品牌文本
- `process_command()` 是 `HermesCLI` 上的一个方法 —— 按中央注册表中的 `resolve_command()` 解析出的规范命令名进行分发
- Skill 斜杠命令：`agent/skill_commands.py` 会扫描 `~/.hermes/skills/`，并将内容作为 **user message** 注入（而非 system prompt），以保留提示缓存

### 斜杠命令注册表（`hermes_cli/commands.py`）

所有斜杠命令都定义在中央 `COMMAND_REGISTRY` 列表中，列表元素为 `CommandDef` 对象。所有下游使用方都会自动基于这个注册表派生：

- **CLI** —— `process_command()` 通过 `resolve_command()` 解析别名，并按规范名分发
- **Gateway** —— `GATEWAY_KNOWN_COMMANDS` 冻结集合用于 hook 发射，`resolve_command()` 用于分发
- **Gateway help** —— `gateway_help_lines()` 生成 `/help` 输出
- **Telegram** —— `telegram_bot_commands()` 生成 BotCommand 菜单
- **Slack** —— `slack_subcommand_map()` 生成 `/hermes` 子命令路由
- **Autocomplete** —— `COMMANDS` 扁平字典供 `SlashCommandCompleter` 使用
- **CLI help** —— `COMMANDS_BY_CATEGORY` 字典供 `show_help()` 使用

### 添加一个斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中添加一个 `CommandDef` 条目：
```python
CommandDef("mycommand", "它的功能描述", "Session",
           aliases=("mc",), args_hint="[arg]"),
```
2. 在 `cli.py` 的 `HermesCLI.process_command()` 中添加处理逻辑：
```python
elif canonical == "mycommand":
    self._handle_mycommand(cmd_original)
```
3. 如果该命令在 gateway 中也可用，则在 `gateway/run.py` 中添加处理逻辑：
```python
if canonical == "mycommand":
    return await self._handle_mycommand(event)
```
4. 对于持久化设置，请在 `cli.py` 中使用 `save_config_value()`

**CommandDef 字段：**
- `name` —— 不带斜杠的规范名称（例如 `"background"`）
- `description` —— 面向人类的描述
- `category` —— `"Session"`、`"Configuration"`、`"Tools & Skills"`、`"Info"`、`"Exit"` 之一
- `aliases` —— 别名元组（例如 `("bg",)`）
- `args_hint` —— 帮助信息中显示的参数占位（例如 `"<prompt>"`、`"[name]"`）
- `cli_only` —— 仅在交互式 CLI 中可用
- `gateway_only` —— 仅在消息平台中可用
- `gateway_config_gate` —— 配置点路径（例如 `"display.tool_progress_command"`）；当它设置在一个 `cli_only` 命令上时，只要对应配置值为真，该命令就会在 gateway 中可用。`GATEWAY_KNOWN_COMMANDS` 总是包含受配置门控的命令，以便 gateway 分发；帮助/菜单仅在门控打开时展示它们。

**添加一个别名** 只需要把别名加到现有 `CommandDef` 的 `aliases` 元组中即可。无需修改其他文件 —— 分发、帮助文本、Telegram 菜单、Slack 映射和自动补全都会自动更新。

---

## 添加新工具

需要修改 **2 个文件**：

**1. 创建 `tools/your_tool.py`：**
```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**2. 添加到 `toolsets.py`** —— 可以加入 `_HERMES_CORE_TOOLS`（所有平台）或新增一个 toolset。

自动发现：任何 `tools/*.py` 文件只要在顶层调用了 `registry.register()`，都会被自动导入 —— 无需手动维护导入列表。

注册表负责处理 schema 收集、分发、可用性检查与错误包装。所有 handler **必须** 返回 JSON 字符串。

**工具 schema 中的路径引用**：如果 schema 描述里提到了文件路径（例如默认输出目录），请使用 `display_hermes_home()` 让它具有 profile 感知能力。schema 在导入时生成，而这发生在 `_apply_profile_override()` 设置 `HERMES_HOME` 之后。

**状态文件**：如果工具需要存储持久状态（缓存、日志、检查点），请使用 `get_hermes_home()` 作为基础目录 —— 不要使用 `Path.home() / ".hermes"`。这样每个 profile 都会拥有自己的状态目录。

**Agent 级工具**（todo、memory）：在进入 `handle_function_call()` 之前由 `run_agent.py` 拦截。可参考 `todo_tool.py` 的模式。

---

## 添加配置

### `config.yaml` 选项：
1. 添加到 `hermes_cli/config.py` 的 `DEFAULT_CONFIG`
2. 递增 `_config_version`（当前为 5），以触发对现有用户的迁移

### `.env` 变量：
1. 在 `hermes_cli/config.py` 的 `OPTIONAL_ENV_VARS` 中添加元数据：
```python
"NEW_API_KEY": {
    "description": "它的用途",
    "prompt": "显示名称",
    "url": "https://...",
    "password": True,
    "category": "tool",  # provider、tool、messaging、setting
},
```

### 配置加载器（两套独立系统）：

| Loader | 用于 | 位置 |
|--------|------|------|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | `hermes tools`、`hermes setup` | `hermes_cli/config.py` |
| 直接加载 YAML | Gateway | `gateway/run.py` |

---

## 皮肤/主题系统

皮肤引擎（`hermes_cli/skin_engine.py`）提供数据驱动的 CLI 视觉定制。皮肤是**纯数据** —— 添加新皮肤不需要改动代码。

### 架构

```
hermes_cli/skin_engine.py    # SkinConfig 数据类、内置皮肤、YAML 加载器
~/.hermes/skins/*.yaml       # 用户安装的自定义皮肤（直接放入即可）
```

- `init_skin_from_config()` —— 在 CLI 启动时调用，从配置中读取 `display.skin`
- `get_active_skin()` —— 返回当前皮肤对应的缓存 `SkinConfig`
- `set_active_skin(name)` —— 在运行时切换皮肤（由 `/skin` 命令使用）
- `load_skin(name)` —— 先加载用户皮肤，再尝试内置皮肤，最后回退到默认皮肤
- 缺失的皮肤值会自动继承 `default` 皮肤中的对应值

### 皮肤可定制内容

| 元素 | 皮肤键 | 使用位置 |
|------|--------|----------|
| 横幅面板边框 | `colors.banner_border` | `banner.py` |
| 横幅面板标题 | `colors.banner_title` | `banner.py` |
| 横幅分节标题 | `colors.banner_accent` | `banner.py` |
| 横幅弱化文本 | `colors.banner_dim` | `banner.py` |
| 横幅正文文本 | `colors.banner_text` | `banner.py` |
| 响应框边框 | `colors.response_border` | `cli.py` |
| Spinner 表情（等待） | `spinner.waiting_faces` | `display.py` |
| Spinner 表情（思考） | `spinner.thinking_faces` | `display.py` |
| Spinner 动词 | `spinner.thinking_verbs` | `display.py` |
| Spinner 翅膀（可选） | `spinner.wings` | `display.py` |
| 工具输出前缀 | `tool_prefix` | `display.py` |
| 每个工具的 emoji | `tool_emojis` | `display.py` → `get_tool_emoji()` |
| Agent 名称 | `branding.agent_name` | `banner.py`、`cli.py` |
| 欢迎语 | `branding.welcome` | `cli.py` |
| 响应框标签 | `branding.response_label` | `cli.py` |
| 提示符符号 | `branding.prompt_symbol` | `cli.py` |

### 内置皮肤

- `default` —— 经典 Hermes 金色/kawaii 风格（当前默认外观）
- `ares` —— 深红/青铜战神主题，带自定义 spinner 翅膀
- `mono` —— 干净的灰度单色主题
- `slate` —— 冷色调、偏开发者风格的蓝色主题

### 添加内置皮肤

在 `hermes_cli/skin_engine.py` 的 `_BUILTIN_SKINS` 字典中添加：

```python
"mytheme": {
    "name": "mytheme",
    "description": "简短描述",
    "colors": { ... },
    "spinner": { ... },
    "branding": { ... },
    "tool_prefix": "┊",
},
```

### 用户皮肤（YAML）

用户可创建 `~/.hermes/skins/<name>.yaml`：

```yaml
name: cyberpunk
description: 霓虹浸染的终端主题

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  banner_accent: "#FF1493"

spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "

tool_prefix: "▏"
```

通过 `/skin cyberpunk` 或在 `config.yaml` 中设置 `display.skin: cyberpunk` 启用。

---

## 重要策略
### 提示缓存绝不能被破坏

Hermes-Agent 会确保一次对话中的缓存始终有效。**不要实现以下会导致缓存失效的改动：**
- 在对话过程中修改过去的上下文
- 在对话过程中更改 toolset
- 在对话过程中重新加载记忆或重建系统提示词

缓存失效会显著增加成本。我们**唯一**允许修改上下文的时机是进行上下文压缩时。

### 工作目录行为
- **CLI**：使用当前目录（`.` → `os.getcwd()`）
- **Messaging**：使用 `MESSAGING_CWD` 环境变量（默认是 home 目录）

### 后台进程通知（Gateway）

当使用 `terminal(background=true, notify_on_complete=true)` 时，gateway 会运行一个 watcher，
检测进程完成并触发新的一轮 agent turn。后台进程消息的详细程度由
`config.yaml` 中的 `display.background_process_notifications`
（或环境变量 `HERMES_BACKGROUND_NOTIFICATIONS`）控制：

- `all` —— 运行中的输出更新 + 最终消息（默认）
- `result` —— 只显示最终完成消息
- `error` —— 仅在退出码不为 0 时显示最终消息
- `off` —— 完全不显示 watcher 消息

---

## Profiles：多实例支持

Hermes 支持 **profiles** —— 多个完全隔离的实例，每个实例都有自己的
`HERMES_HOME` 目录（配置、API keys、memory、sessions、skills、gateway 等）。

核心机制是：`hermes_cli/main.py` 中的 `_apply_profile_override()` 会在任何模块导入之前设置
`HERMES_HOME`。这样，所有 119+ 处对 `get_hermes_home()` 的调用都会自动作用于当前活动 profile。

### 编写 profile-safe 代码的规则

1. **所有 HERMES_HOME 路径都使用 `get_hermes_home()`。** 从 `hermes_constants` 导入。
   绝对不要在读写状态的代码中硬编码 `~/.hermes` 或 `Path.home() / ".hermes"`。
   ```python
   # GOOD
   from hermes_constants import get_hermes_home
   config_path = get_hermes_home() / "config.yaml"

   # BAD —— 会破坏 profiles
   config_path = Path.home() / ".hermes" / "config.yaml"
   ```

2. **面向用户展示的路径使用 `display_hermes_home()`。** 从 `hermes_constants` 导入。
   它在默认 profile 下返回 `~/.hermes`，在其他 profile 下返回 `~/.hermes/profiles/<name>`。
   ```python
   # GOOD
   from hermes_constants import display_hermes_home
   print(f"Config saved to {display_hermes_home()}/config.yaml")

   # BAD —— 对 profiles 展示了错误路径
   print("Config saved to ~/.hermes/config.yaml")
   ```

3. **模块级常量是可以的** —— 它们会在导入时缓存 `get_hermes_home()`，
   而导入发生在 `_apply_profile_override()` 设置环境变量之后。只要使用 `get_hermes_home()`，
   不要使用 `Path.home() / ".hermes"` 即可。

4. **测试中如果 mock 了 `Path.home()`，也必须设置 `HERMES_HOME`** —— 因为现在代码使用的是
   `get_hermes_home()`（读取环境变量），而不是 `Path.home() / ".hermes"`：
   ```python
   with patch.object(Path, "home", return_value=tmp_path), \
        patch.dict(os.environ, {"HERMES_HOME": str(tmp_path / ".hermes")}):
       ...
   ```

5. **Gateway 平台适配器应使用 token 锁** —— 如果适配器使用唯一凭证（bot token、API key）连接，
   请在 `connect()`/`start()` 中调用 `gateway.status` 里的 `acquire_scoped_lock()`，
   并在 `disconnect()`/`stop()` 中调用 `release_scoped_lock()`。这样可以防止两个 profile
   同时使用同一份凭证。标准写法可参考 `gateway/platforms/telegram.py`。

6. **Profile 操作以 HOME 为锚点，而不是 HERMES_HOME** —— `_get_profiles_root()` 返回的是
   `Path.home() / ".hermes" / "profiles"`，而不是 `get_hermes_home() / "profiles"`。
   这是有意为之 —— 这样 `hermes -p coder profile list` 无论当前激活哪个 profile，
   都能看到所有 profiles。

## 已知陷阱

### 不要硬编码 `~/.hermes` 路径
代码路径请使用 `hermes_constants` 中的 `get_hermes_home()`。面向用户打印/日志中的路径请使用 `display_hermes_home()`。
硬编码 `~/.hermes` 会破坏 profiles —— 每个 profile 都有自己的 `HERMES_HOME` 目录。这正是 PR #3575 修复的 5 个 bug 的根源。

### 不要用 `simple_term_menu` 做交互式菜单
它在 tmux/iTerm2 中有渲染问题 —— 滚动时会出现 ghosting。请改用 `curses`（stdlib）。可参考 `hermes_cli/tools_config.py` 的做法。

### 不要在 spinner/display 代码中使用 `\033[K`（ANSI 擦除到行尾）
在 `prompt_toolkit` 的 `patch_stdout` 下，它会以字面量 `?[K` 泄漏到终端。请改用空格补齐：`f"\r{line}{' ' * pad}"`。

### `_last_resolved_tool_names` 是 `model_tools.py` 中的进程级全局变量
`delegate_tool.py` 中的 `_run_single_child()` 会在子代理执行前后保存并恢复这个全局变量。如果你新增代码读取这个变量，请注意在 child agent 运行期间它可能暂时处于过时状态。

### 不要在 schema 描述中硬编码跨工具引用
工具 schema 的描述中不应直接按名称提到其他 toolset 的工具（例如让 `browser_navigate` 写“优先使用 web_search”）。这些工具可能不可用（缺少 API key、toolset 被禁用），从而导致模型幻觉调用不存在的工具。如果确实需要跨工具引用，请在 `model_tools.py` 的 `get_tool_definitions()` 中动态添加 —— 可参考 `browser_navigate` / `execute_code` 的后处理逻辑。

### 测试绝不能写入 `~/.hermes/`
`tests/conftest.py` 中的 `_isolate_hermes_home` 自动 fixture 会把 `HERMES_HOME` 重定向到临时目录。测试中绝对不要硬编码 `~/.hermes/` 路径。

**Profile 测试**：当测试 profile 功能时，也要 mock `Path.home()`，这样
`_get_profiles_root()` 和 `_get_default_hermes_home()` 才会解析到临时目录中。
可参考 `tests/hermes_cli/test_profiles.py` 的写法：
```python
@pytest.fixture
def profile_env(tmp_path, monkeypatch):
    home = tmp_path / ".hermes"
    home.mkdir()
    monkeypatch.setattr(Path, "home", lambda: tmp_path)
    monkeypatch.setenv("HERMES_HOME", str(home))
    return home
```

---

## 测试

```bash
source venv/bin/activate
python -m pytest tests/ -q          # 完整测试套件（约 3000 个测试，约 3 分钟）
python -m pytest tests/test_model_tools.py -q   # toolset 解析
python -m pytest tests/test_cli_init.py -q       # CLI 配置加载
python -m pytest tests/gateway/ -q               # Gateway 测试
python -m pytest tests/tools/ -q                 # 工具级测试
```

推送改动前务必运行完整测试套件。
