# Skill Manager

AI 编程工具的 skill 集中管理方案。所有 skill 统一存放在 **skill store**（`~/.skills/`），项目通过 symlink 引用，不重复、始终同步。

支持 Claude Code、OpenAI Codex 等所有基于目录读取 skill 的 AI 工具。

## 包含什么

| Skill | 功能 |
|-------|------|
| **skill-store** | 管理 skill store（`~/.skills/`）。从 GitHub 安装、迁移本地 skill、卸载、列表浏览。 |
| **link-manager** | 将 store 中的 skill 链接到项目。链接、取消链接、查看项目已绑定的 skill。 |

## 工作原理

```
~/.skills/                              ← skill store（git 仓库）
├── management/
│   ├── skill-store/                    ← 仓库管理 skill
│   └── link-manager/                   ← 项目绑定 skill
├── <分类>/
│   └── <skill名>/
│       └── SKILL.md
└── .skill-meta.json

~/.claude/skills/                       ← 项目从这里读取 skill
├── skill-store  → ~/.skills/management/skill-store   (symlink)
└── link-manager → ~/.skills/management/link-manager  (symlink)
```

## 安装

### 快速开始（3 条命令）

```bash
# 1. 克隆仓库到 skill store
git clone https://github.com/huwanguli/skill-manager.git ~/.skills

# 2. 将管理 skill 链接到你的 AI 工具
ln -s ~/.skills/management/skill-store ~/.claude/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.claude/skills/link-manager

# 3. 完成。用 skill-store 安装更多 skill。
```

### 其他 AI 工具

把 `.claude` 换成对应工具的 skill 目录即可：

```bash
# OpenAI Codex / .agent 平台
ln -s ~/.skills/management/skill-store ~/.agent/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.agent/skills/link-manager
```

### Windows 注意事项

Windows (Git Bash) 上 `ln -s` 默认会复制目录而不是创建 symlink，需要先设置环境变量：

```bash
export MSYS="winsymlinks:native"
ln -s ~/.skills/management/skill-store ~/.claude/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.claude/skills/link-manager
```

**前提条件：** 需要开启开发者模式（设置 → 开发者选项 → 开发人员模式）。

## 使用方式

用自然语言和 AI 对话即可，不需要记命令。

### 从 GitHub 安装 skill

```
用户：从 mattpocock/skills 安装 grill-me

skill-store：
1. 克隆仓库
2. 读取 SKILL.md，推荐分类：productivity
3. 复制到 ~/.skills/productivity/grill-me/
4. Git 提交
5. 提示：用 link-manager link grill-me 链接到项目
```

### 迁移本地 skill 到 store

```
用户：把 .claude/skills/build-your-own-x 迁移到中央仓库

skill-store：
1. 读取 SKILL.md，推荐分类：learning
2. 移动到 ~/.skills/learning/build-your-own-x/
3. 在原位置创建 symlink
4. Git 提交
```

### 链接 skill 到项目

```
用户：把 grill-me 链接到当前项目

link-manager：
1. 在 store 中找到：~/.skills/productivity/grill-me/
2. 检测平台目录：.claude
3. 创建 symlink：.claude/skills/grill-me → ~/.skills/productivity/grill-me/
```

### 浏览和取消链接

```
用户：我中央仓库里有哪些 skill
用户：看看当前项目链接了哪些 skill
用户：把 grill-me 从项目里去掉
```

## 跨平台支持

| 平台 | 创建 symlink 方式 | 注意事项 |
|------|-------------------|----------|
| Linux | `ln -s` | 直接可用 |
| macOS | `ln -s` | 直接可用。`readlink -f` 不支持，需要用 `greadlink` |
| Windows (Git Bash) | `MSYS="winsymlinks:native" ln -s` | 需要开启开发者模式。不设置 MSYS 会静默复制 |
| Windows (WSL) | `ln -s` | 直接可用 |

## 目录结构

```
skill-manager/
├── README.md
├── .gitignore
├── skill-store/
│   └── SKILL.md          ← 仓库管理
└── link-manager/
    └── SKILL.md          ← 项目绑定
```
