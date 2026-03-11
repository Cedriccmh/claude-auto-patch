# claude-auto-patch

**[English](#english) | [中文](#中文)**

---

<a id="english"></a>

## English

A configurable auto-patch system for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Automatically applies binary patches on session startup and survives updates.

### Features

- **Auto-patch on startup** - SessionStart hook detects updates and re-applies patches automatically
- **Configurable** - Enable/disable individual patches via JSON config
- **Smart caching** - Skips re-scanning unchanged binaries (uses file mtime)
- **Cross-platform** - Windows (rename-swap for running exe), macOS (codesign), Linux
- **Create patches with AI** - `/create-patch` skill guides Claude through the entire patch creation workflow

### Quick Start

#### 1. Clone

```bash
git clone https://github.com/Cedriccmh/claude-auto-patch.git
```

#### 2. Install Hook

Copy the hook files to your Claude Code hooks directory:

```bash
cp hooks/auto-patch.py ~/.claude/hooks/
cp hooks/auto-patch-config.json ~/.claude/hooks/
```

Add the SessionStart hook to your `~/.claude/settings.json`:

```jsonc
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            // Adjust path to where you placed auto-patch.py
            "command": "python ~/.claude/hooks/auto-patch.py"
          }
        ]
      }
    ]
  }
}
```

#### 3. Install Skill (Optional)

To use the `/create-patch` skill for AI-assisted patch creation:

```bash
cp -r skills/create-patch ~/.claude/skills/
```

Or install as a plugin by adding this repo path to your Claude Code configuration.

### Configuration

Edit `auto-patch-config.json` to toggle patches:

```json
{
  "toolsearch": true
}
```

Set a patch to `false` to disable it. Remove the cache file to force re-evaluation:

```bash
rm ~/.claude/.auto-patch-cache.json
```

### Built-in Patches

| Patch | Description | Default |
|-------|-------------|---------|
| `toolsearch` | Remove Tool Search domain restriction | Enabled |

### Creating New Patches

#### Using the Skill (Recommended)

Use `/create-patch` in Claude Code and describe what you want to patch. The AI will:

1. Search the npm version of `cli.js` for relevant code
2. Analyze the minified JavaScript logic
3. Design an equal-length binary replacement
4. Register the patch definition and update config
5. Test the patch

#### Manual

Add a `PatchDef` to `hooks/auto-patch.py`:

```python
"your_patch": PatchDef(
    name="your_patch",
    description="What this patch does",
    target_re=re.compile(rb'regex matching original code'),
    patched_re=re.compile(rb'regex matching patched code'),
    build_replacement=lambda m: _equal_length_replace(
        b"replacement_prefix", b"replacement_suffix", m
    ),
),
```

Then add the toggle to `auto-patch-config.json`:

```json
{
  "toolsearch": true,
  "your_patch": true
}
```

### How It Works

#### Auto-Patch Flow

```
Claude Code starts
  -> SessionStart hook triggers
  -> Load config (which patches are enabled)
  -> Find claude binary / cli.js in PATH
  -> Check cache (file mtime + enabled patches)
  -> If unchanged: skip (< 1ms)
  -> If changed: read file, check each patch status
  -> Apply unpatched patches (equal-length byte replacement)
  -> Backup original, write patched version
  -> Update cache
```

#### Equal-Length Replacement

Patches must be exactly the same byte length as the original code. This is achieved by using JavaScript comments (`/* */`) as padding:

```
Original: return["api.anthropic.com"].includes(Xe)}catch{return!1}
Patched:  return!0/*                          */}catch{return!0}
```

#### Three-State Detection

Each patch has two regexes:
- `target_re` matches unpatched code -> safe to apply
- `patched_re` matches patched code -> already done, skip
- Neither matches -> version incompatible, do not touch

### Requirements

- Python 3.10+
- Claude Code (bun binary or npm install)

---

<a id="中文"></a>

## 中文

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的可配置自动补丁系统。会话启动时自动应用补丁，更新后自动恢复。

### 功能

- **启动时自动补丁** — SessionStart hook 检测更新并自动重新应用补丁
- **可配置** — 通过 JSON 配置文件启用/禁用单个补丁
- **智能缓存** — 跳过未变更的二进制文件（基于 mtime），< 1ms
- **跨平台** — Windows（rename-swap 处理运行中的 exe）、macOS（codesign）、Linux
- **AI 辅助创建补丁** — `/create-patch` skill 引导 Claude 完成补丁创建全流程

### 快速开始

#### 1. 克隆

```bash
git clone https://github.com/Cedriccmh/claude-auto-patch.git
```

#### 2. 安装 Hook

将 hook 文件复制到 Claude Code 的 hooks 目录：

```bash
cp hooks/auto-patch.py ~/.claude/hooks/
cp hooks/auto-patch-config.json ~/.claude/hooks/
```

在 `~/.claude/settings.json` 中添加 SessionStart hook：

```jsonc
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            // 调整为 auto-patch.py 的实际路径
            "command": "python ~/.claude/hooks/auto-patch.py"
          }
        ]
      }
    ]
  }
}
```

#### 3. 安装 Skill（可选）

使用 `/create-patch` skill 进行 AI 辅助补丁创建：

```bash
cp -r skills/create-patch ~/.claude/skills/
```

也可以作为 plugin 安装，将此仓库路径添加到 Claude Code 配置中。

### 配置

编辑 `auto-patch-config.json` 切换补丁开关：

```json
{
  "toolsearch": true
}
```

设为 `false` 禁用。删除缓存文件可强制重新检测：

```bash
rm ~/.claude/.auto-patch-cache.json
```

### 内置补丁

| 补丁 | 说明 | 默认 |
|------|------|------|
| `toolsearch` | 解除 Tool Search 域名限制 | 启用 |

### 创建新补丁

#### 使用 Skill（推荐）

在 Claude Code 中使用 `/create-patch`，描述你想补丁的内容。AI 将自动：

1. 在 npm 版 `cli.js` 中搜索相关代码
2. 分析 minified JavaScript 逻辑
3. 设计等长二进制替换
4. 注册补丁定义并更新配置
5. 测试补丁

#### 手动添加

在 `hooks/auto-patch.py` 中添加 `PatchDef`：

```python
"your_patch": PatchDef(
    name="your_patch",
    description="补丁说明",
    target_re=re.compile(rb'匹配原始代码的正则'),
    patched_re=re.compile(rb'匹配补丁后代码的正则'),
    build_replacement=lambda m: _equal_length_replace(
        b"替换前缀", b"替换后缀", m
    ),
),
```

然后在 `auto-patch-config.json` 中添加开关：

```json
{
  "toolsearch": true,
  "your_patch": true
}
```

### 工作原理

#### 自动补丁流程

```
Claude Code 启动
  -> 触发 SessionStart hook
  -> 加载配置（哪些补丁已启用）
  -> 在 PATH 中查找 claude 二进制或 cli.js
  -> 检查缓存（文件 mtime + 已启用补丁列表）
  -> 未变更：跳过（< 1ms）
  -> 已变更：读取文件，检查各补丁状态
  -> 应用未补丁的项（等长字节替换）
  -> 备份原始文件，写入补丁版本
  -> 更新缓存
```

#### 等长替换

补丁必须与原始代码完全等长。通过 JavaScript 注释（`/* */`）填充实现：

```
原始: return["api.anthropic.com"].includes(Xe)}catch{return!1}
补丁: return!0/*                          */}catch{return!0}
```

#### 三态检测

每个补丁有两个正则表达式：
- `target_re` 匹配未补丁代码 -> 可安全应用
- `patched_re` 匹配已补丁代码 -> 跳过
- 都不匹配 -> 版本不兼容，不操作

### 环境要求

- Python 3.10+
- Claude Code（bun 二进制或 npm 安装）

## License / 许可证

MIT
