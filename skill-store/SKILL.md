---
name: skill-store
description: Manage the skill store at ~/.skills/. Install skills from GitHub, migrate local skills, uninstall, list, and browse. Use when the user wants to add, remove, or view skills in ~/.skills/. The store is the single source of truth; projects link to it via symlinks.
---

# Skill Store

The **skill store** is the directory `~/.skills/` on the user's machine. It holds all skills in one place. Projects don't copy skills — they symlink to this store (managed by `link-manager`).

When the user says "中央仓库", "skill store", "the store", or "~/.skills/", they mean this directory.

## Store Structure

```
~/.skills/
├── .git/
├── management/
│   ├── skill-store/      ← this skill
│   └── link-manager/     ← companion skill
├── <category>/
│   └── <skill-name>/
│       └── SKILL.md
└── .skill-meta.json      ← registry of installed skills
```

## Cross-Platform Compatibility

This skill must work on Linux, macOS, and Windows (Git Bash, WSL, or native).

**Before running any operation**, detect the platform and adapt:

```bash
# Detect OS
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
  **Verification:** After creating, always verify with `[ -L "<link>" ]`. If it returns false, the symlink was not created (likely Developer Mode is off).

**Path handling:**
- Always use `/` (forward slash) in scripts — Git Bash and WSL handle this correctly.
- When passing paths to `cmd.exe`, convert with `cygpath -w`.
- `$HOME` works on all platforms (Linux, macOS, Git Bash, WSL).

**Symlink detection:**
- `[ -L "<path>" ]` works on all platforms to test if a path is a symlink.
- `readlink "<path>"` works on Linux, macOS, and Git Bash to read the symlink target.
- On macOS, `readlink` does NOT support `-f`. If you need to resolve a chain of symlinks, use `cd "$(dirname "$path")" && cd "$(readlink "$(basename "$path")")" && pwd` or install `coreutils` for `greadlink -f`.

## Operations

Determine the operation from the user's request. The keywords below are hints, not rigid commands—use judgment.

### install

**Triggers:** "install", "add", "download", "获取", "安装"

**Input:** A GitHub source. Accepts:
- Full URL: `https://github.com/owner/repo`
- Short form: `owner/repo`
- With skill path: `owner/repo/skills/grill-me` or `owner/repo:skills/grill-me`

**Steps:**

1. **Parse the source.** Extract `owner/repo` and optional sub-path from the input.

2. **Clone to temp directory.**
   ```bash
   TMPDIR=$(mktemp -d)
   git clone --depth 1 "https://github.com/$OWNER/$REPO.git" "$TMPDIR"
   ```

3. **Detect repo structure and locate SKILL.md files:**
   - If a sub-path was given and it contains `SKILL.md` → use that directly.
   - If root has `SKILL.md` → single skill repo (structure C).
   - If `skills/` directory exists → scan `skills/*/SKILL.md` (structure A).
   - Otherwise → recursively scan for all `SKILL.md` files.

4. **Select skill(s).**
   - If the user specified a skill name and it was found → proceed with that one.
   - If multiple candidates found → list them with name + description, ask user to pick.
   - If one candidate → confirm with user.

5. **Read the skill's SKILL.md** to extract `name` and `description` from frontmatter.

6. **Determine category.**
   - List existing directories in `~/.skills/` (excluding `management` and `.git`).
   - Read the skill's name and description, recommend the best matching existing category.
   - Present: "推荐分类: `<category>` (已有的分类: `cat1`, `cat2`, ...). 确认或输入新分类名?"
   - User confirms or provides a new name.

7. **Check for conflicts.**
   - If `~/.skills/<category>/<skill-name>/` already exists → refuse, ask user to uninstall first or choose a different name.

8. **Install.**
   ```bash
   mkdir -p "$HOME/.skills/<category>"
   cp -r "$TMPDIR/<skill-path>" "$HOME/.skills/<category>/<skill-name>"
   ```

9. **Update registry.** Read or create `~/.skills/.skill-meta.json`:
   ```json
   {
     "version": 1,
     "skills": {
       "<skill-name>": {
         "source": "owner/repo",
         "sourceType": "github",
         "skillPath": "<path-in-repo>",
         "category": "<category>",
         "installedAt": "<ISO timestamp>"
       }
     }
   }
   ```
   Merge into existing entries (don't overwrite others).

10. **Git commit.**
    ```bash
    cd "$HOME/.skills"
    git add -A
    git commit -m "install: <skill-name>"
    ```

11. **Clean up.** `rm -rf "$TMPDIR"`

12. **Report success.** Show: skill name, category, path, and suggest running `link-manager link <skill-name>` to use it in a project.

### migrate

**Triggers:** "migrate", "move", "import", "迁移", "导入"

**Input:** A local path. It can be:
- A single skill directory (contains `SKILL.md`)
- A project directory (scan for skills inside)

**Steps:**

**Single skill migration** (path contains `SKILL.md` directly):

1. Read `SKILL.md` to get name and description.
2. Check `~/.skills/` for conflicts (same name in any category → refuse).
3. Determine category (same flow as install: recommend + user confirm).
4. Move the directory:
   ```bash
   mv "<source-path>" "$HOME/.skills/<category>/<skill-name>"
   ```
5. Create symlink at original location (cross-platform, see "Cross-Platform Compatibility" section):
   ```bash
   # Linux / macOS:
   ln -s "$HOME/.skills/<category>/<skill-name>" "<source-path>"
   # Windows (Git Bash) — must set MSYS first:
   # export MSYS="winsymlinks:native"
   # ln -s "$HOME/.skills/<category>/<skill-name>" "<source-path>"
   ```
6. Update `~/.skills/.skill-meta.json` with migrate entry.
7. Git commit: `"migrate: <skill-name>"`
8. Report: skill name, category, old path → new path, symlink created.

**Batch migration** (project directory):

1. Scan for skills in these locations:
   - `<project>/.claude/skills/*/SKILL.md`
   - `<project>/.agents/skills/*/SKILL.md`
   - Any other `<project>/*/<skills>/*/SKILL.md` patterns
2. List all found skills with name + description + current path.
3. Ask user: "找到 N 个 skill，哪些要迁移到中央仓库？" with a numbered list.
4. User selects (all, none, or specific numbers).
5. For each selected skill, execute single skill migration flow (steps 1-7 above).
6. Report summary: migrated count, skipped count, details.

### uninstall

**Triggers:** "uninstall", "remove", "delete", "卸载", "删除"

**Input:** Skill name.

**Steps:**

1. Find the skill in `~/.skills/`:
   ```bash
   find "$HOME/.skills" -mindepth 2 -maxdepth 3 -name "SKILL.md" | while read f; do
     dir=$(dirname "$f")
     name=$(grep "^name:" "$f" | head -1 | sed 's/name: *//')
     if [ "$name" = "<skill-name>" ]; then echo "$dir"; fi
   done
   ```

2. If not found → report error, suggest `skill-store list` to see available skills.

3. **Check for active symlinks.** Scan known project locations:
   ```bash
   # Check common project locations for symlinks pointing to this skill
   # Avoid scanning all of $HOME — too slow and may hit permission issues on Windows
   for PROJECT_DIR in "$HOME/projects" "$HOME/code" "$HOME/dev" "$HOME/Desktop" "$HOME/Documents"; do
     [ -d "$PROJECT_DIR" ] || continue
     find "$PROJECT_DIR" -path "*/skills/<skill-name>" -type l 2>/dev/null | while read link; do
       target=$(readlink "$link")
       if echo "$target" | grep -q "$HOME/.skills/"; then
         echo "⚠️  $link → $target"
       fi
     done
   done
   ```
   Also check `.claude/skills/` and `.agents/skills/` in the current project directory if inside one.
   **Note:** On Windows, avoid scanning `$HOME` broadly — use known project roots or the current project only.

4. If active symlinks found → **refuse to uninstall**. List all affected projects:
   ```
   ❌ Cannot uninstall "<skill-name>": the following projects still link to it:
      - /path/to/project-a/.claude/skills/<skill-name>
      - /path/to/project-b/.agents/skills/<skill-name>
   Please unlink these projects first (use link-manager unlink).
   ```

5. If no active symlinks → confirm with user: "确认删除 <skill-name> (<category>/)? 此操作不可撤销。"

6. Delete:
   ```bash
   rm -rf "$HOME/.skills/<category>/<skill-name>"
   ```

7. Remove from `~/.skills/.skill-meta.json`.

8. Git commit: `"uninstall: <skill-name>"`

9. Report success.

### list

**Triggers:** "list", "show", "browse", "ls", "列表", "查看"

**Input:** Optional category filter.

**Steps:**

1. Scan `~/.skills/` for skill directories:
   ```bash
   for dir in "$HOME/.skills"/*/; do
     category=$(basename "$dir")
     [ "$category" = "management" ] || [ "$category" = ".git" ] && continue
     echo "📂 $category"
     for skill_dir in "$dir"*/; do
       [ -f "$skill_dir/SKILL.md" ] || continue
       name=$(grep "^name:" "$skill_dir/SKILL.md" | head -1 | sed 's/name: *//')
       desc=$(grep "^description:" "$skill_dir/SKILL.md" | head -1 | sed 's/description: *//')
       echo "   📦 $name — $desc"
     done
   done
   ```

2. If category filter provided → only show that category.

3. If store is empty → suggest running `skill-store install <source>`.

4. Output a clean formatted list.

### info

**Triggers:** "info", "detail", "about", "详情"

**Input:** Skill name.

**Steps:**

1. Find the skill in `~/.skills/` (same search as uninstall step 1).
2. If not found → report error.
3. Read and display:
   - Skill name and description (from frontmatter)
   - Category
   - Full path in store
   - Source info from `.skill-meta.json` (if available)
   - Full SKILL.md content
4. Also show which projects currently link to this skill (scan for symlinks).

## Platform Awareness

When the user asks to link a skill after installing, suggest: "使用 `link-manager link <skill-name>` 将此 skill 链接到你的项目。支持 .claude、.agent 等平台目录。"
