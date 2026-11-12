# Keygen MACHINE FILE 操作细节

以下请求形状来自本工作区 Keygen CE 1.7.2 与 Logix 客户端的实际签出和解析流程。每次执行前核对部署版本、policy 和产品配置；示例不包含可用凭据或固定资源 ID。

## API 与凭据

- API 基础路径：`/v1/accounts/{account-id}`。在受控环境读取已有部署配置，用短时管理员令牌执行查询和签发，结束时撤销该令牌。不要打印管理员密码、令牌或 license key。请求超时后先查询服务端是否已创建，不盲目重发写请求。
- 查询 policies、licenses、machines 时遍历分页。用目标 product 和 policy 的真实 ID，不从名称猜 ID；校验 policy 的 `maxMachines: 1`、产品关系及订阅 `duration`。
- CE 1.7.2 的许可证创建接口支持 `data.attributes.metadata`（键值对象）及部分额度覆盖字段，例如 `maxCores`、`maxUses`；`data.relationships.owner`、`group` 需要已存在的资源 ID。该版本的 `permissions` 属性在 EE 条件分支内，不作为 CE 可选授权。`entitlements` **不是** `POST /licenses` 的 attributes 或 relationships 输入；它是独立资源，通过 policy 继承或单独关联到 license。别把 `include` 误认为授权开关：它只决定签出快照包含什么。依据：[CE 1.7.2 licenses controller](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/app/controllers/api/v1/licenses_controller.rb)、[machine checkout service](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/app/services/machine_checkout_service.rb)。在其他版本或部署中先核对实际接口与客户端解析能力，再承诺可签发字段。
- 签发前只读列出目标 policy 的 entitlements、相关 license 的有效 entitlements 及已存在的 entitlement 资源，遍历分页并核对 ID、`code`、`name`、`metadata`。policy 授权会自动传给其 license；`GET /licenses/{id}/entitlements` 返回 policy 继承项和 license 专属项的合并结果。用户要求“无 entitlements”而 policy 已继承非空项时，不可能靠省略请求或 checkout 的 include 达成；先澄清或选择经确认的其他 policy。不要为单个用户的键值去修改共享 entitlement 的 `metadata`；每证个性化键值用确认过的 license `metadata`。依据：[entitlements API](https://keygen.sh/docs/api/entitlements/)、[licenses API](https://keygen.sh/docs/api/licenses/)。
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

  仅在用户已逐项确认后，才在 `attributes` 增加确认过的 `metadata` 对象或支持的额度覆盖字段，或在 `relationships` 增加已核对的 owner/group ID；确认“无”就不传这些可选项。不要传 `entitlements` 字段。`POST /licenses` 后读取 `attributes.created`、`expiry`、`name`、`metadata`、已确认的可选属性和 policy 关系；Keygen 会将 metadata 键序列化为 lower camel case，按 API 回读后的实际键和值复核，避免拼写转换导致误发。当前“订阅版 · 1 年”policy 的 `duration` 是 `31536000`，但 CE 1.7.2 签出的实际 `expiry` 可能比创建后完整日历年略早。当本次已确认的许可证有效期为一年（Skill 默认值）时，通过 `PATCH /licenses/{id}` 将 `expiry` 设为 `created` 的下一日历年同一时刻，并从 API 回读确认；若用户确认了其他期限，则按该期限设置并回读核验。不要把 policy 的 `duration` 当作许可证有效期证明。
- `entitlements` 需先按用户确认的 code 查到准确资源 ID。若资源不存在，先核对创建账户级 entitlement 的授权及其 `name`、`code`、`metadata`，不能默默新建或改写已有资源。policy 继承项无需重复关联；license 专属项在创建 license 后、checkout 前以 `POST /licenses/{license-id}/entitlements` 关联（需 `license.entitlements.attach` 权限）：

  ```json
  { "data": [{ "type": "entitlements", "id": "<已核对的entitlement-id>" }] }
  ```

  完整路径在 `/v1/accounts/{account-id}` 下。关联后用 `GET /licenses/{license-id}/entitlements` 遍历全部页，按 ID、code、`metadata` 核对有效集合与确认结果完全相等；额外项或缺项都应停止。接口形状见 [CE 1.7.2 entitlement relationship controller](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/app/controllers/api/v1/licenses/relationships/entitlements_controller.rb)。
- 激活机器：`POST /machines`，以该 license key 授权，`data.type` 为 `machines`，`attributes.fingerprint` 为台账指纹，`relationships.license.data.id` 为父许可证 ID。已有相同指纹时先核对所属许可证和目标 policy；不要直接当作成功，也不要重复创建。
- 导出机器文件：`POST /machines/{machine-id}/actions/check-out/`，请求体为 `{"meta":{"ttl":<秒>,"encrypt":false,"include":["license","license.entitlements"]}}`。证书文本位于响应 `data.attributes.certificate`；同时确认响应的 `data.attributes.includes` 含这两个路径。无 entitlements 时仍按此请求，以签名内的空集合证明“无”。缺少 `license` 时 Logix 客户端无法读取父许可证的产品、状态和期限；缺少 `license.entitlements` 时不能凭文件中没有 entitlement 断言确实没有授权。CE 1.7.2 支持该 include 路径，见 [checkout controller](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/app/controllers/api/v1/machines/actions/checkouts_controller.rb)、[include 列表](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/app/services/machine_checkout_service.rb)。

## 一年期限与文件验收

Keygen CE 1.7.2 对 `ttl` 使用日历时长解释。本次同一年期历史样本的参数为 `31556952`；直接传 `31536000` 虽然接近 365 天，实际 `meta.expiry` 会提前。传参后仍需检查文件 `meta.issued`、`meta.ttl` 和 `meta.expiry`，并按用户承诺的完整日历年核对实际截止日。父许可证期限与文件期限是两个独立约束；客户端以较早者为准。闰年边界以实际日历日期核对。

Logix 直接导入的文件需满足：

1. PEM 封套是 `BEGIN/END MACHINE FILE`，原始文件不超过客户端 64 KiB 上限；封套体是 base64 JSON，`alg` 为 `base64+ed25519`。
2. 账户 Ed25519 公钥验证 `sig`，签名输入为 UTF-8 `"machine/" + enc`；`enc` 解码后的数据集含 `data`、`included` 和 `meta`。不要从证书本身取公钥作为信任锚。
3. `data.type == "machines"`，机器指纹与台账完全一致；`included` 中父 license 的 ID、`name`、policy 及 product 关系符合预期，状态为 `ACTIVE`，`maxMachines == 1`，期限未过。分别检查 `meta.issued`/`expiry` 和父 license 的 `expiry`。
4. 在**已验签的**数据集内，检查父 license `attributes.metadata` 的完整键和值（含值类型）及已确认的额度覆盖等属性、owner/group 关系；从 `included` 中取所有 `type == "entitlements"` 的条目，按 ID、`attributes.code`、`name`、`metadata` 与确认的有效集合逐项比较，既不缺少也不多出。确认“无”时必须是空集合，且 API 的有效集合也为空；如果缺少应有条目、发现额外条目，或签名文件没有可复验的确认字段，不交付。`license.entitlements` 会将 policy 继承和 license 专属授权一并纳入签出数据，见 [CE 1.7.2 checkout 场景](https://github.com/keygen-sh/keygen-api/blob/v1.7.2/features/api/v1/machines/actions/checkouts.feature)。

若修改已签发证书的用户名、policy、entitlements、metadata 或其他已确认条款，签名内容不会自动更新，必须重新 checkout 并复验文件。只有服务端值相符仍不足以复用旧文件，需验证磁盘文件的签名快照也相符。改变服务端状态不会改写已分发文件中的签名状态；替换时提醒接收方弃用旧文件。台账回写只在新文件复验成功后进行。
