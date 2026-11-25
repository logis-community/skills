---
name: logix-mcp-setup
description: 将外部 AI 客户端接入 Logix 智能仿真系统 MCP，完成 Streamable HTTP 配置、OAuth 登录和只读验证，或排查连接、授权和工作台会话问题。
---

# 接入 Logix 智能仿真系统 MCP

## 确认目标实例

1. 确认用户要连接的 Logix 实例，以及当前或指定的 AI 客户端。实例需要有效的 Keygen 许可证和可信授权快照，且包含 `LOGIX_MCP`。常规桌面版由许可证桥提供快照，普通独立 Web 构建默认不可用。
2. 同机桌面版使用 `http://127.0.0.1:<实际 BFF 端口>/api/mcp`。从桌面版当前环境配置或启动日志确认端口。远程客户端使用部署方提供的可达 `https://<Logix 域名>/api/mcp`。以客户端进程所在机器判断可达性，云端客户端的 loopback 指向云端自身。离线账号授权仅限本机连接。
3. 地址仍不明确时，向用户确认实际 BFF 端口或部署地址。验证时可请求同一 origin 的 `/.well-known/oauth-protected-resource/api/mcp`，核对响应的 `resource` 是该 MCP URL，`authorization_servers` 包含同源的 `/oauth`。`/mcp/workspace-socket` 是 Logix 编辑页面的内部连接。

## 授权前确认权限

- **只读接入**只请求 `logix:read` 这一项业务权限。它足以验证 `listWorkspaces` 和 `getWorkspaceTools`，实际可读数据仍取决于账号业务权限。复用已有授权前确认其业务 scope。
- **授予 `logix:write`** 允许客户端在账号业务权限内直接调用顶层写工具。例如顶层 `updateCase` 通过 scope 和业务权限校验后直接更新已保存案例，无需 Logix 逐次审批。逐次审批适用于经 `callWorkspaceTool` 发起的工作台写入或执行。
- **浏览器同意页**按客户端请求的 scope 整体同意或拒绝。请用户核对实例、账号、数据环境与所请求的权限后亲自决定。如果只读接入的请求包含 `logix:write` 或 `logix:execute`，拒绝该次授权，改用下方 Codex 的显式只读命令。

## 配置客户端并授权

1. 检查目标客户端现有配置。已有同址的 `logix` 条目时复用，地址不同时先确认目标实例。缺失时按用户要求的配置层级添加 Streamable HTTP 服务，保留其他条目。确认目标实例后，由 agent 使用该客户端的 MCP 管理入口完成配置。
2. 使用客户端原生 OAuth。服务端支持 CIMD 和动态客户端注册（DCR），兼容客户端通常可自动注册公开客户端并使用 Authorization Code + S256 PKCE。登录页使用 Logix 的 CFNet 或本机离线账号。若收到 `invalid_scope`，确认至少请求了一个 Logix 业务 scope。认证材料保留在客户端的凭据存储。
3. 按当前版本的 `--help` 选原生命令。客户端若提示需要重载，重载后再验证。登录等待用户完成时保留客户端的登录流程，以客户端认证结果为准。

   - **Codex**：用 `codex mcp get logix` 检查，使用 `codex mcp add logix --url <MCP_URL>` 添加。只读登录用 `codex mcp login logix --scopes logix:read`。若添加配置时自动弹出超出所需范围的授权，先拒绝，再检查条目并运行显式登录命令。需要写入或执行时，用 `--scopes` 指定任务所需的权限。若服务端无法访问 Codex 的 CIMD 文档，可在支持的版本上为登录命令加 `--oauth-client-registration dcr`，保留显式 scope。
   - **OpenCode**：用 `opencode mcp list` 检查，使用 `opencode mcp add logix --url <MCP_URL>` 添加。需写入用户配置时可加 `--global`（以当前版本为准）。`opencode mcp auth logix` 在 2.1.2 的命令帮助中没有 scope 参数，该版本的只读配置路径仍待验证。需要只读授权时，使用上面的 Codex 命令，或使用已确认能显式请求 `logix:read` 的客户端版本。

## 只读验证与使用

1. 在客户端确认 OAuth 已连接并列出工具，再调用 `listWorkspaces({})`。成功返回空列表表示账号已连接，当前无可见编辑会话。授权可在打开编辑页前完成。交付时说明配置作用域、OAuth 状态和实际验证结果。
2. 若任务涉及当前工作台，请用户在同一账号和数据环境中打开目标 Logix 页面。待页面工具加载并连接后重新调用 `listWorkspaces`，按页面选择会话，再用 `getWorkspaceTools({ workspaceId })` 读取实际工具、参数 Schema 和 scope。工作台验证以能找到目标会话并读取其工具为准。页面切换或工具更新后重新获取 workspace ID。
3. 按需用 `callWorkspaceTool({ workspaceId, name, arguments })` 调用目标会话工具。已保存案例可优先使用服务端只读工具 `getCaseSummary`，未保存状态使用工作台的画布工具。工作台审批在支持 MCP Apps 的客户端内展示，其余客户端在目标 Logix 页面展示。操作超时或断连后先读状态，再判断是否需要重试。

## 排查与部署

- **请求无响应或发现内容不匹配**。核对实际 BFF 端口、客户端网络位置、固定 origin 与 `/api/mcp` 路径。发现端点也返回许可相关 403 时，检查 `LOGIX_MCP`、授权有效性与可信快照。设备或 Relay 授权补加 entitlement 后需重新签发或 checkout，在线授权需刷新。
- **401 或 `-32003`**。检查 OAuth 是否失效或撤销，按需重新授权。CFNet 的服务端业务工具若报告页面登录会话不可用，先打开并登录 Logix 页面恢复凭据租约；若刚切换数据环境，核对授权目标。
- **`-32004` 或工具缺失**。检查已授予的 OAuth scope、账号业务权限和页面工具注册情况。工作台为空或切页后会话消失时，检查页面顶栏连接，重新列出会话。代理部署还需检查 WebSocket Upgrade。
- **用户负责部署时**。桌面默认使用应用数据目录的 `.mcp-oauth`。部署覆盖项以 Logix 源仓库的 `apps/logix/.env.example` 与 `apps/logix/docs/mcp-server.md` 为准。`LOGIX_MCP_ORIGIN` 使用无路径和尾斜杠的固定 origin，HTTP 仅限 loopback。`LOGIX_MCP_STATE_DIR` 使用私有绝对路径。`LOGIX_MCP_CLIENTS_FILE` 仅用于按客户端实际回调预注册公开客户端。反向代理保留固定 Host，按需设置精确的 `LOGIX_MCP_UI_ORIGINS`，仅当代理清理并重写转发头且直连受限时设置 `LOGIX_MCP_TRUST_PROXY=1`。变更 origin 或状态目录后，客户端可能需要重新授权。
