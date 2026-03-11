# claude-auto-patch

A configurable auto-patch system for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Automatically applies binary patches on session startup and survives updates.

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的可配置自动补丁系统。会话启动时自动应用补丁，更新后自动恢复。

## Features / 功能

- **Auto-patch on startup / 启动时自动补丁** - SessionStart hook detects updates and re-applies patches automatically / SessionStart hook 检测更新并自动重新应用补丁
- **Configurable / 可配置** - Enable/disable individual patches via JSON config / 通过 JSON 配置文件启用/禁用单个补丁
- **Smart caching / 智能缓存** - Skips re-scanning unchanged binaries (uses file mtime) / 跳过未变更的二进制文件（基于 mtime）
- **Cross-platform / 跨平台** - Windows (rename-swap for running exe), macOS (codesign), Linux
- **Create patches with AI / AI 辅助创建补丁** - `/create-patch` skill guides Claude through the entire patch creation workflow / `/create-patch` skill 引导 Claude 完成补丁创建全流程

## Quick Start / 快速开始

### 1. Clone / 克隆

```bash
git clone https://github.com/Cedriccmh/claude-auto-patch.git
```

### 2. Install Hook / 安装 Hook

Copy the hook files to your Claude Code hooks directory:

将 hook 文件复制到 Claude Code 的 hooks 目录：

```bash
cp hooks/auto-patch.py ~/.claude/hooks/
cp hooks/auto-patch-config.json ~/.claude/hooks/
```

Add the SessionStart hook to your `~/.claude/settings.json`:

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
            // Adjust path to where you placed auto-patch.py
            // 调整为 auto-patch.py 的实际路径
            "command": "python ~/.claude/hooks/auto-patch.py"
          }
        ]
      }
    ]
  }
}
```

### 3. Install Skill (Optional) / 安装 Skill（可选）

To use the `/create-patch` skill for AI-assisted patch creation:

使用 `/create-patch` skill 进行 AI 辅助补丁创建：

```bash
cp -r skills/create-patch ~/.claude/skills/
```

Or install as a plugin by adding this repo path to your Claude Code configuration.

也可以作为 plugin 安装，将此仓库路径添加到 Claude Code 配置中。

## Configuration / 配置

Edit `auto-patch-config.json` to toggle patches:

编辑 `auto-patch-config.json` 切换补丁开关：

```json
{
  "toolsearch": true
}
```

Set a patch to `false` to disable it. Remove the cache file to force re-evaluation:

设为 `false` 禁用。删除缓存文件可强制重新检测：

```bash
rm ~/.claude/.auto-patch-cache.json
```

## Built-in Patches / 内置补丁

| Patch / 补丁 | Description / 说明 | Default / 默认 |
|-------|-------------|---------|
| `toolsearch` | Remove Tool Search domain restriction / 解除 Tool Search 域名限制 | Enabled / 启用 |

## Creating New Patches / 创建新补丁

### Using the Skill (Recommended) / 使用 Skill（推荐）

Use `/create-patch` in Claude Code and describe what you want to patch. The AI will:

在 Claude Code 中使用 `/create-patch`，描述你想补丁的内容。AI 将自动：

1. Search the npm version of `cli.js` for relevant code / 在 npm 版 `cli.js` 中搜索相关代码
2. Analyze the minified JavaScript logic / 分析 minified JavaScript 逻辑
3. Design an equal-length binary replacement / 设计等长二进制替换
4. Register the patch definition and update config / 注册补丁定义并更新配置
5. Test the patch / 测试补丁

### Manual / 手动添加

Add a `PatchDef` to `hooks/auto-patch.py`:

在 `hooks/auto-patch.py` 中添加 `PatchDef`：

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

然后在 `auto-patch-config.json` 中添加开关：

```json
{
  "toolsearch": true,
  "your_patch": true
}
```

## How It Works / 工作原理

### Auto-Patch Flow / 自动补丁流程

```
Claude Code starts / 启动
  -> SessionStart hook triggers / 触发 SessionStart hook
  -> Load config (which patches are enabled) / 加载配置（哪些补丁已启用）
  -> Find claude binary / cli.js in PATH / 在 PATH 中查找 claude 二进制或 cli.js
  -> Check cache (file mtime + enabled patches) / 检查缓存（文件 mtime + 已启用补丁列表）
  -> If unchanged: skip (< 1ms) / 未变更：跳过（< 1ms）
  -> If changed: read file, check each patch status / 已变更：读取文件，检查各补丁状态
  -> Apply unpatched patches (equal-length byte replacement) / 应用未补丁的项（等长字节替换）
  -> Backup original, write patched version / 备份原始文件，写入补丁版本
  -> Update cache / 更新缓存
```

### Equal-Length Replacement / 等长替换

Patches must be exactly the same byte length as the original code. This is achieved by using JavaScript comments (`/* */`) as padding:

补丁必须与原始代码完全等长。通过 JavaScript 注释（`/* */`）填充实现：

```
Original: return["api.anthropic.com"].includes(Xe)}catch{return!1}
Patched:  return!0/*                          */}catch{return!0}
```

### Three-State Detection / 三态检测

Each patch has two regexes / 每个补丁有两个正则表达式：

- `target_re` matches unpatched code -> safe to apply / 匹配未补丁代码 -> 可安全应用
- `patched_re` matches patched code -> already done, skip / 匹配已补丁代码 -> 跳过
- Neither matches -> version incompatible, do not touch / 都不匹配 -> 版本不兼容，不操作

## Requirements / 环境要求

- Python 3.10+
- Claude Code (bun binary or npm install)

## License / 许可证

MIT
