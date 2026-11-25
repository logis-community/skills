# logis-community skills

本仓库提供可由 [Vercel Skills CLI](https://www.skills.sh/docs/cli) 安装的 `logis` 技能包。`aliyun-yunxiao-mcp` 指导接入阿里云云效 MCP；`logix-machine-license-issuance` 指导从台账签发与重签 Logix Keygen 单机离线证书。仓库不保存凭据、证书或个人记录。

用 [`logix-mcp-setup`](logix-mcp-setup/SKILL.md) 将外部 AI 客户端接入 Logix 智能仿真系统，并验证 OAuth 授权和工作台连接。

云效子 Skill 会识别当前 AI 客户端并配置云效 MCP；需要 OAuth 登录时先征得用户同意，再由 agent 发起登录，用户在浏览器完成授权。

## 安装

一条命令将整个技能包安装到 Codex 的用户级目录：

```sh
npx skills add logis-community/skills -g -a codex
```

在 Codex 中，目录如下；新增的 Logis 子技能放在仓库根目录的子文件夹下，可随技能包一起安装：

```text
~/.agents/skills/logis/
├── SKILL.md
├── aliyun-yunxiao-mcp/
│   └── SKILL.md
├── logix-mcp-setup/
│   └── SKILL.md
└── logix-machine-license-issuance/
    ├── SKILL.md
    └── references/keygen-machine-file.md
```

根目录 `SKILL.md` 的 `name: logis` 决定安装目录名；仓库名不会决定它。`-g` 指用户级安装，`-a codex` 只安装给 Codex，避免 CLI 尝试其他 Agent。只安装某个子 Skill 时，可运行 `npx skills add logis-community/skills --full-depth --skill <子Skill名> -g -a codex`。Codex 会扫描用户级 Skill 目录；若新 Skill 未立即出现，可重启 Codex。

当前仓库为**私有**。具备仓库读取权限且本机 Git 认证可用的成员可使用上述命令；也可先克隆，再在仓库根目录运行 `npx skills add . -g -a codex`。GitHub 简写能否读取私有仓库取决于 Git 认证路径。

## 适用范围

云效子 Skill 面向需要通过云效 MCP 查询组织、项目、工作项、代码库或流水线，以及提交 Bug 或需求的用户。中心站托管地址是 `https://openapi-rdc.aliyuncs.com/ai/mcp`；专有 Region 站使用本组织的 `https://<组织专属域名>/ai/mcp`。默认使用 OAuth 与托管地址；仅在客户端不支持 OAuth 时使用个人访问令牌（PAT）。本地 stdio 需要 Node.js 20 或更新版本。

提交 Bug 或需求时，Skill 会先确认云效项目，默认指派给 `dlx`，创建后获取并返回可验证的工作项链接；若当前接口仅能提供 ID，会如实说明。找不到指定项目时会提醒联系项目负责人添加权限。当前 Qoder CLI 没有独立的 MCP OAuth 登录子命令；无法由 agent 发起的授权入口会如实说明。

详细配置及安全要求见 [`aliyun-yunxiao-mcp/SKILL.md`](aliyun-yunxiao-mcp/SKILL.md)。Logix 单机证书的签发、验签和台账回写流程见 [`logix-machine-license-issuance/SKILL.md`](logix-machine-license-issuance/SKILL.md)。

## 验证

1. 在客户端确认 `yunxiao` MCP 服务已连接，并能看到云效工具。
2. 仅执行只读请求，例如“列出我所在的组织”。确认返回的是自己的云效组织。
3. 如需特定模块，再用对应只读操作验证；不要把写操作当作连接测试。

仓库维护者可在根目录运行 `npx skills add . --list` 检查 `logis` 入口；运行 `npx skills add . --list --full-depth` 可检查所有子 Skill。

## 常见错误

| 现象 | 检查 |
| --- | --- |
| 无法连接或地址错误 | 核对中心站与 Region 站域名；托管模式需要客户端支持 Streamable HTTP。 |
| 未出现浏览器授权 | 确认客户端支持 MCP OAuth；否则改用 PAT。 |
| 401 或令牌过期 | 核对 PAT 是否有效，或重新完成 OAuth 授权；切勿把令牌贴进 Issue、聊天或日志。 |
| 403 或部分工具失败 | 检查云效账号资源权限及 PAT 的对应 API 权限；仅补足所需权限。 |
| 本地服务启动失败 | 检查 Node.js >= 20、`npx` 可用，以及客户端进程是否继承 `YUNXIAO_ACCESS_TOKEN`。 |
| Skill CLI 找不到 Skill | 检查根目录 `SKILL.md`，运行 `npx skills add . --list`；查找子 Skill 时加 `--full-depth`。 |

## 官方来源

- [阿里云：云效 MCP 工具使用说明](https://help.aliyun.com/zh/yunxiao/developer-reference/cloud-effect-mcp-tool-instructions)
- [阿里云：个人访问令牌](https://help.aliyun.com/zh/yunxiao/user-guide/personal-access-token)
- [阿里云：创建工作项](https://help.aliyun.com/zh/yunxiao/developer-reference/createworkitem)
- [阿里云：云效 MCP Server 源码](https://github.com/aliyun/alibabacloud-devops-mcp-server)
- [Vercel：Skills CLI](https://www.skills.sh/docs/cli)
- [OpenAI：Codex 的 Skill 目录与发现](https://learn.chatgpt.com/docs/build-skills)
