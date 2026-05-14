# Rshell Skills

AI Agent 技能包，用于通过自然语言操控 Rshell C2 框架。

## 使用方法

在 OpenCode / Claude Code 等支持 Skills 的 AI 工具中引用此技能：

```yaml
available_skills:
  - name: rshell-c2
    description: Rshell C2 框架控制端操作指南
    location: file:///mnt/Data/Pentest/Creating/Rshell/Rshell-Skills/SKILL.md
```

或复制到 `~/.opencode/skill/rshell-c2/SKILL.md` 自动加载。

## 章节概览

| 章节 | 内容 |
|------|------|
| 第一章 | 认证与鉴权 — 登录、JWT Token、WebSocket 双 Token |
| 第二章 | 完整 API 路由参考 — 69 个路由的输入/输出格式 |
| 第三章 | 通用客户端操作流程 — 命令执行、文件操作、凭据抓取、后渗透 |
| 第四章 | 常见问题 — Token 过期、离线、超时等 |
