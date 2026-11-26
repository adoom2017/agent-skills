# Agent Skills

个人可复用的 AI Coding Agent Skills 仓库（兼容 Antigravity、Claude Code、Codex、`npx skills`）。

## Skills 列表

| Skill | 路径 | 说明 |
|---|---|---|
| `rust-development` | [SKILL.md](./skills/rust-development/SKILL.md) | 基于 *The Rust Programming Language*（稳定版全书 Ch 1–21）、Rust 2024 Edition、Rust API Guidelines 与主流 `rust-skills` 提炼的 Rust 开发、编译错误诊断与代码审查 Skill |

## 安装与使用

### 1. 使用 `npx skills` CLI 安装

```bash
npx skills add adoom2017/agent-skills --skill rust-development -g
```

### 2. 手动克隆并软链接

链接到全局 `~/.agents/skills/` 或具体项目的 `.agents/skills/`：

```bash
git clone https://github.com/adoom2017/agent-skills.git
ln -s "$(pwd)/agent-skills/skills/rust-development" ~/.agents/skills/rust-development
```

## 新增 Skill 目录规范

```text
skills/<skill-name>/
├── SKILL.md       # 必填：YAML frontmatter (name, description) + 核心契约与路由表
├── references/    # 选填：按需加载的深度参考文档（渐进式披露）
├── resources/     # 选填：配置模板与静态资源
└── scripts/       # 选填：自动化校验或辅助脚本
```
