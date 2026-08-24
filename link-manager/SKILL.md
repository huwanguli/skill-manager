---
name: link-manager
description: Link or unlink skills between ~/.skills/ (the store) and the current project. Supports .claude, .agent, and custom platform directories. Use when the user wants to connect, disconnect, or view skills linked to their project.
---

# Link Manager

The **skill store** is `~/.skills/`. This skill manages symlinks that connect the store to project directories (e.g., `.claude/skills/`, `.agents/skills/`).

It does NOT install or remove skills from the store — use `skill-store` for that.

When the user says "链接 skill", "link skill", "项目里加个 skill", or similar, they want this skill.

## How It Works

```
~/.skills/<category>/<skill-name>/SKILL.md    ← source of truth
        ↑ symlink
<project>/.claude/skills/<skill-name>/        ← project reads from here
<project>/.agents/skills/<skill-name>/        ← or here (other AI tools)
```

## Cross-Platform Compatibility

This skill must work on Linux, macOS, and Windows (Git Bash, WSL, or native).

**Before running any operation**, detect the platform and adapt:

```bash
OS="$(uname -s)"
case "$OS" in
  Linux*)   PLATFORM="linux";;
  Darwin*)  PLATFORM="macos";;
  MINGW*|MSYS*|CYGWIN*) PLATFORM="windows-gitbash";;
  *)        PLATFORM="unknown";;
esac
```

**Symlink creation rules:**
- **Linux / macOS:** Use `ln -s <target> <link>` directly.
- **Windows (Git Bash):** `ln -s` alone silently copies instead of creating symlinks. Must set the MSYS environment variable first:
  ```bash
  export MSYS="winsymlinks:native"
  ln -s <target> <link>
  ```
  **Prerequisites:** Developer Mode must be enabled (Settings → Developer Options → Developer Mode). If it's not enabled, prompt the user to enable it.
  **Verification:** After creating, always verify with `[ -L "<link>" ]`. If it returns false, the symlink was not created.

**Symlink detection:**
- `[ -L "<path>" ]` works on all platforms.
- `readlink "<path>"` works on Linux, macOS, and Git Bash. Use it without `-f`.

**Path handling:**
- Always use `/` in scripts. Use `cygpath -w` when passing to `cmd.exe`.
- `$HOME` works everywhere.

## Operations

### link

**Triggers:** "link", "connect", "attach", "链接", "连接"

**Input:** Skill name, or a category to browse.

**Steps:**

1. **Resolve the target skill.**

   **By name:** If user provides a skill name:
   ```bash
   # Search store for the skill
   find "$HOME/.skills" -mindepth 2 -maxdepth 3 -name "SKILL.md" | while read f; do
     dir=$(dirname "$f")
     name=$(grep "^name:" "$f" | head -1 | sed 's/name: *//')
     if [ "$name" = "<skill-name>" ]; then echo "$dir"; fi
   done
   ```
   If not found → suggest `skill-store list` and ask user to install first.
   If multiple matches (same name in different categories) → list them, ask user to pick.

   **By category browse:** If user says "看看 X 类" or "show me X skills":
   ```bash
   ls -1 "$HOME/.skills/<category>/" 2>/dev/null
   ```
   List skills in that category with name + description. User picks one.

2. **Detect available platform directories** in the current project:
   ```bash
   PLATFORMS=()
   [ -d ".claude" ] && PLATFORMS+=(".claude")
   [ -d ".agents" ] && PLATFORMS+=(".agents")
   ```
   Also check for any other `*/skills/` directories in the project root.

3. **Select target platform.**
   - If only one platform directory exists → use it, confirm with user.
   - If multiple → list them, ask user to choose: "项目中有以下平台目录，链接到哪个？"
   - If none exist → ask user which to create (e.g., `.claude`), then `mkdir -p .claude/skills`.

4. **Check for conflicts.**
   - If `<project>/<platform>/skills/<skill-name>` already exists:
     - If it's a symlink → check if it points to the store already. If yes, report "已经链接了". If it points elsewhere, ask user to unlink first.
     - If it's a real directory → this is a local skill. Ask user: "同名的本地 skill 已存在，要替换为中央仓库的链接吗？" If yes, warn that the local skill will be lost (suggest `skill-store migrate` first).

5. **Create symlink** (cross-platform, see "Cross-Platform Compatibility" section):
   ```bash
   # Linux / macOS:
   ln -s "$HOME/.skills/<category>/<skill-name>" "<project>/<platform>/skills/<skill-name>"
   # Windows (Git Bash) — must set MSYS first:
   # export MSYS="winsymlinks:native"
   # ln -s "$HOME/.skills/<category>/<skill-name>" "<project>/<platform>/skills/<skill-name>"
   ```

6. **Verify.**
   ```bash
   ls -la "<project>/<platform>/skills/<skill-name>"
   ```
   Confirm the symlink resolves correctly.

7. **Report success.**
   ```
   ✅ 已链接: <skill-name>
      源: ~/.skills/<category>/<skill-name>/
      链接: <project>/<platform>/skills/<skill-name> → 源
   ```

**Batch link:** If user says "链接 X 类的所有 skill" or "链接这些 skill: a, b, c":
- Loop through the list, execute steps 3-7 for each.
- Report summary at the end.

### unlink

**Triggers:** "unlink", "disconnect", "remove link", "取消链接", "断开"

**Input:** Skill name (optional: platform to target).

**Steps:**

1. **Scan current project for symlinks** pointing to the store:
   ```bash
   for platform in .claude .agents; do
     [ -d "$platform/skills" ] || continue
     for link in "$platform/skills"/*/; do
       [ -L "${link%/}" ] || continue
       target=$(readlink "${link%/}")
       if echo "$target" | grep -q "$HOME/.skills/"; then
         name=$(basename "${link%/}")
         echo "🔗 $platform/skills/$name → $target"
       fi
     done
   done
   ```

2. **Select which to unlink.**
   - If user specified a name and it matches → proceed with that one.
   - If user specified a platform → filter to that platform only.
   - If multiple matches → list them, ask user to pick (or "all").

3. **Confirm.** "确认取消链接 <skill-name>？（不会删除中央仓库中的 skill）"

4. **Remove symlink.**
   ```bash
   rm "<project>/<platform>/skills/<skill-name>"
   ```
   ⚠️ **Use `rm` on the symlink path, NOT `rm -r`.** The symlink target (`~/.skills/...`) must NOT be touched.
   ⚠️ **On Windows:** If `rm` fails on a directory symlink, use `rmdir` instead.

5. **Report success.**
   ```
   ✅ 已取消链接: <skill-name>
      中央仓库中的 skill 仍保留在 ~/.skills/<category>/<skill-name>/
   ```

### list

**Triggers:** "list links", "show linked", "what skills", "已链接", "查看链接"

**Input:** None (operates on current project).

**Steps:**

1. **Scan all platform directories** in current project:
   ```bash
   echo "📋 当前项目的 skill 链接:"
   echo ""
   for platform in .claude .agents; do
     [ -d "$platform/skills" ] || continue
     echo "  📁 $platform/skills/"
     for item in "$platform/skills"/*/; do
       [ -e "${item%/}" ] || continue
       name=$(basename "${item%/}")
       if [ -L "${item%/}" ]; then
         target=$(readlink "${item%/}")
         if echo "$target" | grep -q "$HOME/.skills/"; then
           echo "    🔗 $name → $target"
         else
           echo "    🔗 $name → $target (外部链接)"
         fi
       else
         echo "    📦 $name (本地 skill，未链接)"
       fi
     done
   done
   ```

2. **If no platform directories found** → report "当前项目没有任何 skill 目录。使用 link-manager link 开始添加 skill。"

3. **If no skills in any platform** → report "当前项目没有链接任何 skill。使用 link-manager link 链接中央仓库中的 skill。"

4. Output the formatted list.

## Error Handling

- **Store not initialized:** If `~/.skills/` doesn't exist → "中央仓库尚未初始化。请先运行 `skill-store install` 或手动创建 ~/.skills/ 目录。"
- **Broken symlink:** If a symlink exists but target doesn't → show warning: "⚠️ 断开的链接: <path> → <target> (目标不存在). 建议取消链接后重新链接。"
- **Permission errors (symlink creation):**
  - **Linux / macOS:** Check directory permissions. The user may need `chmod` or `sudo` (rare).
  - **Windows:** If `ln -s` created a copy instead of a symlink, the MSYS environment variable was not set. Ensure `export MSYS="winsymlinks:native"` is set before `ln -s`. If it still fails, Developer Mode may be off: "请在 Windows 设置中开启开发者模式（设置 → 开发者选项 → 开发人员模式）。"
  - Always verify with `[ -L "<link>" ]` after creation.
- **Path too long (Windows):** If the full path exceeds 260 characters, symlink creation may fail. Suggest shortening the category or skill name, or enabling long paths in Windows registry.
