---
name: aliyun-yunxiao-mcp
description: 配置阿里云云效 MCP Server 供 Qoder、Claude Code 或其他 MCP 客户端使用；适用于云效组织、工作项、Codeup 和流水线的接入与排错。
---

# 配置云效 MCP

帮助用户选择云效站点、认证方式和客户端配置，并用只读请求验证。先确认所用客户端及其支持的 MCP 传输与认证；不要索取或读取真实令牌，不把凭证写入仓库、Issue、聊天、终端日志或可提交的配置文件。

## 选择接入方式

1. **优先 OAuth + 托管 Streamable HTTP**：阿里云文档确认 Qoder/QoderCLI、Claude Code 等支持 MCP OAuth 的客户端可只填服务地址，随后在浏览器登录并授权。中心站通用配置示例：

   ```json
   {
     "mcpServers": {
       "yunxiao": {
         "url": "https://openapi-rdc.aliyuncs.com/ai/mcp"
       }
     }
   }
   ```

   将这段放入**客户端指定的 MCP 配置位置**；不同客户端的配置入口可能不同。若使用云效 Region 站，把 URL 换成实际组织专属域名加 `/ai/mcp`，不要照搬中心站地址。客户端若要求明确选择传输类型，选 Streamable HTTP。OAuth 无需预先创建 PAT，也无需手填 `client_id` 或 `client_secret`。

2. **客户端不支持 OAuth 时使用 PAT**：在云效工作台的头像 → 个人设置 → 个人访问令牌中新建令牌。仅授予当前要用的模块和 API 的权限，优先只读，并设置适当有效期。云效 MCP 快速入门文档给出了全模块读写权限的宽泛前提，但个人访问令牌文档要求按需最小授权；若某个具体操作返回 403，再核对该操作对应的权限。不要为“连通性测试”授予全部读写权限。

   托管模式首选在客户端**安全凭证存储**中设置 `Authorization: Bearer <PAT>` 请求头。下面的结构仅用于认清字段；`<PAT_FROM_CLIENT_SECRET_STORE>` 是占位符，不能原样提交或当作通用变量插值语法：

   ```json
   {
     "mcpServers": {
       "yunxiao": {
         "url": "https://openapi-rdc.aliyuncs.com/ai/mcp",
         "headers": {
           "Authorization": "Bearer <PAT_FROM_CLIENT_SECRET_STORE>"
         }
       }
     }
   }
   ```

   仅在客户端能够安全地注入该值时使用此方式。不要把 PAT 放在 URL Query String 中；阿里云文档指出它可能进入访问日志。不要假定 `${YUNXIAO_ACCESS_TOKEN}` 一类写法在所有客户端 JSON 中都会展开。

3. **本地 stdio 备选**：需要 Node.js >= 20。官方包的启动命令为 `npx -y alibabacloud-devops-mcp-server`，并从环境变量 `YUNXIAO_ACCESS_TOKEN` 读取 PAT。可在启动客户端的同一 shell 中静默输入并导出变量：

   ```sh
   read -rsp 'Yunxiao PAT: ' YUNXIAO_ACCESS_TOKEN; printf '\n'
   export YUNXIAO_ACCESS_TOKEN
   # 从这个 shell 启动你的 MCP 客户端
   ```

   配置客户端启动服务器时使用以下命令和参数，不在配置文件里填 PAT：

   ```json
   {
     "mcpServers": {
       "yunxiao": {
         "command": "npx",
         "args": ["-y", "alibabacloud-devops-mcp-server"]
       }
     }
   }
   ```

   确认客户端进程会继承该环境变量；若客户端不继承环境变量，使用它提供的安全凭证注入功能。Region 站的 stdio 模式还需将 `YUNXIAO_API_BASE_URL` 设置为组织域名（不含 `/ai/mcp`）。

## 验证与排错

- 在客户端检查 `yunxiao` 已连接且工具已列出；再请求“列出我所在的组织”这类只读操作。需要特定模块时，先验证该模块的只读工具。
- 无法连接：检查站点域名与 Streamable HTTP 支持；本地模式检查 Node 版本和 `npx`。
- 401：检查 OAuth 会话或 PAT 是否有效、是否过期；只检查状态，不打印令牌。
- 403：检查账号本身能否访问资源，再检查 PAT 的对应权限；按操作补齐，避免直接扩大到所有 API。
- 工具过多：按[官方工具集筛选说明](https://help.aliyun.com/zh/yunxiao/developer-reference/cloud-effect-mcp-tool-instructions)限定 `toolsets`；可选值以该文档为准。

## 官方依据

- [云效 MCP 工具使用说明](https://help.aliyun.com/zh/yunxiao/developer-reference/cloud-effect-mcp-tool-instructions)：服务地址、托管/stdio 配置、认证、Region 站和验证示例。
- [个人访问令牌](https://help.aliyun.com/zh/yunxiao/user-guide/personal-access-token)：创建入口、有效期与最小权限。
- [云效 MCP Server 官方源码](https://github.com/aliyun/alibabacloud-devops-mcp-server)。
