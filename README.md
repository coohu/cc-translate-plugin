https://code.claude.com/docs/en/plugin-marketplaces 
参考这个文档，开发一个claude code 插件，它能根据~/.claude/settings.json 种的lang 字段设置，实时翻译其它插件的描述信息为该语言。例如，"lang":"zh-cn" 时，运行 /plugin 时， Discover tab 中显示的插件列表中，每个插件的用途描述都用简体中文显示

根据你提供的官方文档 `https://code.claude.com/docs/en/plugin-marketplaces` 和 Claude Code 的插件架构参考，目前 Claude Code 的插件系统（Plugin System）主要支持**自定义命令（Commands）**、**Agent**、**Hooks** 和 **MCP Servers**。

**技术限制说明**：
目前的官方文档（尤其是 Hooks reference 部分）显示，插件主要用于拦截工具调用（Tool Use）或生命周期事件（如 SessionStart），**并没有提供直接修改内置命令（如 `/plugin`）UI 渲染（即直接修改 Discover tab 显示内容）的钩子（Hook）**。

因此，要实现“根据 `lang` 设置翻译插件描述”的需求，最可行的“官方”方案是开发一个**包含自定义命令的插件**（例如 `/discover-i18n` 或 `/pdiscover`），该命令会读取配置、获取插件列表、翻译并展示。

下面是一个完整的插件开发方案，名为 `claude-code-i18n-discover`。

### 1. 插件目录结构

首先，创建一个文件夹 `claude-code-i18n-discover`，结构如下：

```text
claude-code-i18n-discover/
├── .claude-plugin/
│   └── plugin.json          # 插件清单文件
├── commands/
│   └── discover-i18n.md     # 自定义命令定义
├── scripts/
│   └── list_and_translate.py # (可选) 实际执行逻辑的 Python 脚本
└── README.md

```

### 2. 编写插件清单 (`.claude-plugin/plugin.json`)

这是插件的入口文件，定义插件名称和命令。

```json
{
  "name": "i18n-discover",
  "version": "1.0.0",
  "description": "以本地语言展示插件市场 (Translate plugin marketplace descriptions based on settings)",
  "author": {
    "name": "Developer"
  },
  "commands": [
    "./commands/discover-i18n.md"
  ],
  "permissions": {
    "filesystem": ["read"] 
  }
}

```

### 3. 实现自定义命令 (`commands/discover-i18n.md`)

Claude Code 的命令是由 Markdown 文件定义的，你可以直接在 Prompt 中指示 Claude 完成任务（Agentic 方式），或者让 Claude 执行一个 Python 脚本来获取更精确的控制。

#### 方案 A：纯 Prompt 方式（利用 Claude 自身的翻译能力）

这种方式最简单，不需要额外的 Python 依赖，直接利用 Claude 的理解力。

**文件：`commands/discover-i18n.md**`

```markdown
---
description: 查看翻译后的插件市场列表
---

# Localized Plugin Discovery

请执行以下步骤来帮助用户发现插件：

1.  **读取配置**：读取用户主目录下的 `~/.claude/settings.json` 文件，查找 `lang` 字段的值（如果不存在，默认使用 "zh-cn" 即简体中文）。
2.  **获取数据**：列出当前官方或已配置的 Marketplace 中的热门插件（你可以通过运行内部工具或读取 `~/.claude/` 下的相关缓存文件来获取插件列表）。
3.  **翻译与展示**：
    * 提取每个插件的 `name` (名称) 和 `description` (描述)。
    * 将 `description` 翻译成步骤 1 中获取的目标语言（例如简体中文）。
    * 以 Markdown 表格的形式输出结果，包含两列：**插件名称** 和 **功能描述（已翻译）**。

请确保翻译准确、通顺，并符合技术术语习惯。

```

#### 方案 B：Python 脚本方式（更稳定、可编程）

如果你希望对输出格式有更严格的控制，可以让命令去运行一个 Python 脚本。

**步骤 B1：修改 `commands/discover-i18n.md**`

```markdown
---
description: 查看翻译后的插件市场列表 (Script based)
---

# Localized Plugin Discovery

请运行当前插件目录下的脚本 `scripts/list_and_translate.py`。
该脚本会读取 `~/.claude/settings.json` 中的 `lang` 配置，并输出翻译后的插件列表。
请直接向用户展示脚本的输出结果。

```

**步骤 B2：编写 `scripts/list_and_translate.py**`

*注意：实际环境中你需要根据 Claude Code 存储 Marketplace 缓存的具体路径（通常在 `~/.claude/` 下的某个 JSON 或 SQLite）来调整读取逻辑。以下代码为模拟演示。*

```python
#!/usr/bin/env python3
import json
import os
import sys

# 1. 获取用户设置的语言
def get_target_lang():
    settings_path = os.path.expanduser("~/.claude/settings.json")
    target_lang = "zh-cn" # 默认
    if os.path.exists(settings_path):
        try:
            with open(settings_path, 'r', encoding='utf-8') as f:
                data = json.load(f)
                # 假设 lang 字段在根目录或特定结构下
                target_lang = data.get("lang", "zh-cn")
        except Exception:
            pass
    return target_lang

# 2. 模拟获取插件列表 (实际开发中应读取 ~/.claude/marketplace_cache.json 或类似文件)
def get_plugins():
    # 这里为了演示，硬编码了一些示例数据。
    # 实际开发中，你可以尝试解析 `claude plugin list` 的输出或查找缓存文件。
    return [
        {"name": "google-search", "description": "Search the web for relevant information."},
        {"name": "git-tools", "description": "Execute git commands and manage repositories."},
        {"name": "code-review", "description": "Analyze code and provide improvement suggestions."}
    ]

# 3. 模拟翻译功能
# 注意：在没有 API Key 的脚本中，为了演示我们使用简单的映射或伪翻译。
# 实际可以使用 `googletrans` 库或调用外部 API。
def translate_text(text, lang):
    if lang == "zh-cn":
        # 简单示例字典
        translations = {
            "Search the web for relevant information.": "搜索网络以获取相关信息。",
            "Execute git commands and manage repositories.": "执行 Git 命令并管理代码仓库。",
            "Analyze code and provide improvement suggestions.": "分析代码并提供改进建议。"
        }
        return translations.get(text, f"[翻译中] {text}")
    return text

def main():
    lang = get_target_lang()
    plugins = get_plugins()
    
    print(f"## 插件市场 (语言: {lang})\n")
    print(f"| 插件名称 | 功能描述 |")
    print(f"| :--- | :--- |")
    
    for p in plugins:
        translated_desc = translate_text(p['description'], lang)
        print(f"| **{p['name']}** | {translated_desc} |")

if __name__ == "__main__":
    main()

```

### 4. 安装与使用

1. **安装插件**：
在终端中，进入到插件的上级目录，运行以下命令安装你的本地插件：
```bash
claude plugin install ./claude-code-i18n-discover

```


2. **配置语言**：
确保你的 `~/.claude/settings.json` 中有 `lang` 字段：
```json
{
  "lang": "zh-cn"
}

```


3. **运行命令**：
在 Claude Code 中运行：
```text
/discover-i18n

```


(注意：由于 `plugin.json` 中定义的 `name` 是 `i18n-discover`，命令前缀可能是 `/i18n-discover:discover-i18n`，具体取决于 Claude Code 的命名空间规则，通常你可以在命令文件中定义别名)。

### 总结

由于 Claude Code 目前不允许插件直接 Hook 进 `/plugin` 的 UI 渲染层，**方案 A (Prompt Agent)** 是最符合 "Claude Code" 风格的做法。它利用了 Claude 的大模型能力，能够动态读取配置、查找文件并进行高质量翻译，而无需编写复杂的 Python 爬虫或翻译脚本。