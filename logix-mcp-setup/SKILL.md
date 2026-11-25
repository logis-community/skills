---
name: logix-mcp-setup
description: 将 Codex 或 OpenCode 接入 Logix 智能仿真系统 MCP，查找服务地址、配置 Streamable HTTP、完成 OAuth 登录并验证连通。
---

# 接入 Logix 智能仿真系统 MCP

## 1. 确认 MCP 地址

- 同机桌面版用 `http://127.0.0.1:<实际 BFF 端口>/api/mcp`，端口从 Logix 当前环境配置或启动日志确认。
- 远程实例用部署方提供的 `https://<Logix 域名>/api/mcp`，确认客户端运行的机器可以访问该地址。

Logix 实例需要包含 `LOGIX_MCP` 的有效 Keygen 许可证和可信授权快照。

## 2. 配置客户端

把下列命令中的 `<MCP_URL>` 换成实际地址。已有同址的 `logix` 条目时直接复用。

- **Codex** 用 `codex mcp get logix` 检查，缺失时运行 `codex mcp add logix --url '<MCP_URL>'`。
- **OpenCode** 用 `opencode mcp list` 检查，缺失时运行 `opencode mcp add logix --url '<MCP_URL>'`。需要写入用户级配置时加 `--global`。

## 3. 登录

- **Codex** 运行 `codex mcp login logix`。只读接入运行 `codex mcp login logix --scopes logix:read`。
- **OpenCode** 运行 `opencode mcp auth logix`，或继续客户端已打开的登录流程。

在浏览器登录 Logix 并核对账号和 OAuth 同意页上的权限，只读接入时若出现写入或执行权限，拒绝本次请求，改用 Codex 时先按第 2 步确认 `logix` 指向同一 MCP 地址或添加该地址，再运行本步骤的 Codex 只读登录命令。

## 4. 验证连通

在客户端确认能列出 Logix 工具，再调用 `listWorkspaces({})`。调用成功即表示接入通过，返回空列表表示当前没有可见的编辑会话。

部署、授权及工作台问题见 [Logix MCP 产品文档](https://codeup.aliyun.com/691be33315dfc6c86050c687/logis/vue-mono/blob/dev/apps/logix/docs/mcp-server.md)。
