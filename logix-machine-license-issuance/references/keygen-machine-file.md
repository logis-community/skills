# Keygen MACHINE FILE 操作细节

以下请求形状来自本工作区 Keygen CE 1.7.2 与 Logix 客户端的实际签出和解析流程。每次执行前核对部署版本、policy 和产品配置；示例不包含可用凭据或固定资源 ID。

## API 与凭据

- API 基础路径：`/v1/accounts/{account-id}`。在受控环境读取已有部署配置，用短时管理员令牌执行查询和签发，结束时撤销该令牌。不要打印管理员密码、令牌或 license key。请求超时后先查询服务端是否已创建，不盲目重发写请求。
- 查询 policies、licenses、machines 时遍历分页。用目标 product 和 policy 的真实 ID，不从名称猜 ID；校验 policy 的 `maxMachines: 1`、产品关系及订阅 `duration`。
- 创建父许可证：

  ```json
  {
    "data": {
      "type": "licenses",
      "attributes": { "name": "<用户名>" },
      "relationships": {
        "policy": { "data": { "type": "policies", "id": "<已核对的policy-id>" } }
      }
    }
  }
  ```

  `POST /licenses` 后读取 `attributes.created`、`expiry`、`name` 和 policy 关系。当前“订阅版 · 1 年”policy 的 `duration` 是 `31536000`，但 CE 1.7.2 签出的实际 `expiry` 可能比创建后完整日历年略早。本次一周年要求通过 `PATCH /licenses/{id}` 将 `expiry` 设为 `created` 的下一日历年同一时刻，并从 API 回读确认。只在用户要求完整一年、且该修改符合合同条款时这样做；不要把政策的字段值本身当作到期证明。
- 激活机器：`POST /machines`，以该 license key 授权，`data.type` 为 `machines`，`attributes.fingerprint` 为台账指纹，`relationships.license.data.id` 为父许可证 ID。已有相同指纹时先核对所属许可证和目标 policy；不要直接当作成功，也不要重复创建。
- 导出机器文件：`POST /machines/{machine-id}/actions/check-out/`，请求体为 `{"meta":{"ttl":<秒>,"encrypt":false,"include":["license"]}}`。证书文本位于响应 `data.attributes.certificate`。缺少 `include: ["license"]` 时，Logix 客户端无法读取父许可证的产品、状态和期限。

## 一年期限与文件验收

Keygen CE 1.7.2 对 `ttl` 使用日历时长解释。本次同一年期历史样本的参数为 `31556952`；直接传 `31536000` 虽然接近 365 天，实际 `meta.expiry` 会提前。传参后仍需检查文件 `meta.issued`、`meta.ttl` 和 `meta.expiry`，并按用户承诺的完整日历年核对实际截止日。父许可证期限与文件期限是两个独立约束；客户端以较早者为准。闰年边界以实际日历日期核对。

Logix 直接导入的文件需满足：

1. PEM 封套是 `BEGIN/END MACHINE FILE`，原始文件不超过客户端 64 KiB 上限；封套体是 base64 JSON，`alg` 为 `base64+ed25519`。
2. 账户 Ed25519 公钥验证 `sig`，签名输入为 UTF-8 `"machine/" + enc`；`enc` 解码后的数据集含 `data`、`included` 和 `meta`。不要从证书本身取公钥作为信任锚。
3. `data.type == "machines"`，机器指纹与台账完全一致；`included` 中父 license 的 ID、`name`、policy 及 product 关系符合预期，状态为 `ACTIVE`，`maxMachines == 1`，期限未过。分别检查 `meta.issued`/`expiry` 和父 license 的 `expiry`。

若修改已签发证书的用户名或 policy，签名内容不会自动更新，必须重新 checkout 并复验文件。改变服务端状态不会改写已分发文件中的签名状态；替换时提醒接收方弃用旧文件。台账回写只在新文件复验成功后进行。
