# logis-community skills

本仓库提供可由 [Vercel Skills CLI](https://www.skills.sh/docs/cli) 发现的 Agent Skills。首个 Skill `aliyun-yunxiao-mcp` 指导将阿里云云效 MCP Server 接入支持 MCP 的 AI 客户端，不负责代办云效授权或保存凭证。

## 安装

仓库公开后可运行：

```sh
npx skills add logis-community/skills --skill aliyun-yunxiao-mcp
```

当前仓库为**私有**。在公开前，具备仓库读取权限的成员可先克隆，再在本地仓库目录运行 `npx skills add . --skill aliyun-yunxiao-mcp`。GitHub 简写远程安装是否能读取私有仓库取决于 Skills CLI 的认证路径，不能仅凭 `gh auth` 登录推定可用。

## 适用范围

面向需要通过云效 MCP 查询组织、项目、工作项、代码库或流水线，以及提交 Bug 或需求的用户。中心站托管地址是 `https://openapi-rdc.aliyuncs.com/ai/mcp`；专有 Region 站使用本组织的 `https://<组织专属域名>/ai/mcp`。默认使用 OAuth 与托管地址；仅在客户端不支持 OAuth 时使用个人访问令牌（PAT）。本地 stdio 需要 Node.js 20 或更新版本。

提交 Bug 或需求时，Skill 会先确认云效项目，默认指派给 `dlx`，创建后获取并返回可验证的工作项链接；若当前接口仅能提供 ID，会如实说明。找不到指定项目时会提醒联系项目负责人添加权限。

详细配置及安全要求见 [`skills/aliyun-yunxiao-mcp/SKILL.md`](skills/aliyun-yunxiao-mcp/SKILL.md)。

## 验证

1. 在客户端确认 `yunxiao` MCP 服务已连接，并能看到云效工具。
2. 仅执行只读请求，例如“列出我所在的组织”。确认返回的是自己的云效组织。
3. 如需特定模块，再用对应只读操作验证；不要把写操作当作连接测试。

仓库维护者可在根目录运行 `npx skills add . --list` 检查 Skill 发现能力。

## 常见错误

| 现象 | 检查 |
| --- | --- |
| 无法连接或地址错误 | 核对中心站与 Region 站域名；托管模式需要客户端支持 Streamable HTTP。 |
| 未出现浏览器授权 | 确认客户端支持 MCP OAuth；否则改用 PAT。 |
| 401 或令牌过期 | 核对 PAT 是否有效，或重新完成 OAuth 授权；切勿把令牌贴进 Issue、聊天或日志。 |
| 403 或部分工具失败 | 检查云效账号资源权限及 PAT 的对应 API 权限；仅补足所需权限。 |
| 本地服务启动失败 | 检查 Node.js >= 20、`npx` 可用，以及客户端进程是否继承 `YUNXIAO_ACCESS_TOKEN`。 |
| Skill CLI 找不到 Skill | 检查仓库根目录下的 `skills/aliyun-yunxiao-mcp/SKILL.md`，运行 `npx skills add . --list`。 |

## 官方来源

- [阿里云：云效 MCP 工具使用说明](https://help.aliyun.com/zh/yunxiao/developer-reference/cloud-effect-mcp-tool-instructions)
- [阿里云：个人访问令牌](https://help.aliyun.com/zh/yunxiao/user-guide/personal-access-token)
- [阿里云：创建工作项](https://help.aliyun.com/zh/yunxiao/developer-reference/createworkitem)
- [阿里云：云效 MCP Server 源码](https://github.com/aliyun/alibabacloud-devops-mcp-server)
- [Vercel：Skills CLI](https://www.skills.sh/docs/cli)
