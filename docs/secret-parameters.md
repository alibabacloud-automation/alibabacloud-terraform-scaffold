# Secret Parameters Guide / 参数集保密值使用指南

IacService supports setting secret attributes for sensitive data (such as passwords, API Tokens, access keys) in parameter sets. Secret parameter values are automatically encrypted and masked in the console UI, execution logs, and API responses, preventing sensitive information leakage.

自动化服务台支持为参数集中的敏感数据（如密码、API Token、访问密钥等）设置保密属性。标记为保密的参数值会自动加密存储，并在控制台界面、运行日志和接口返回中自动隐藏，防止敏感信息泄露。

---

## Encryption Key Management / 加密密钥管理

Before using secret parameters, you need to configure an encryption key for the current Alibaba Cloud account. IacService leverages Alibaba Cloud Key Management Service (KMS) to encrypt and protect secret values.

在使用保密值功能前，需要先为当前阿里云主账号配置加密密钥。自动化服务台通过阿里云密钥管理服务（KMS）对保密值进行加密保护。

### Set Up Encryption Key / 设置加密密钥

Navigate to **System Settings > Encryption Settings** to configure an encryption key. On first configuration, the system will guide you through Service-Linked Role (SLR) authorization to allow IacService to access your KMS resources.

进入**系统设置 > 加密设置**，为当前主账号配置加密密钥。首次配置时，系统会引导您完成服务关联角色（SLR）授权，允许自动化服务台访问您的 KMS 资源。

Two key types are available / 密钥类型分为两种：

- **Service Key / 服务密钥**: Automatically created and managed by IacService. No manual maintenance required, suitable for quick start. / 由自动化服务台自动创建和管理的默认密钥。不指定密钥时系统会自动为您创建，无需手动维护，适合快速上手。
- **User-Managed Key / 用户主密钥**: A key you created in KMS or shared from another account, with full control. Suitable for scenarios requiring key lifecycle management. / 您在 KMS 中自行创建或从其他账号共享的密钥，拥有完全控制权，适用于需要自行管理密钥生命周期的场景。

Once configured, all secret values in parameter sets under this account will be encrypted with the selected key.

设置完成后，该账号下所有参数集的保密值将使用所配置的密钥进行加密。

### Key Rotation / 更换加密密钥（密钥轮转）

Key replacement is supported. After replacement, new secret parameters are encrypted with the new key, while historical data encrypted with the old key remains unaffected and can still be decrypted normally.

支持更换加密密钥。更换后，新写入的保密参数将使用新密钥加密；已使用旧密钥加密的历史数据不受影响，仍通过旧密钥正常解密。

**Steps / 更换步骤：**

1. Navigate to **System Settings > Encryption Settings**, select a new KMS key and save. / 进入**系统设置 > 加密设置**，重新选择新的 KMS 密钥并保存。
2. After saving, new secret parameters will automatically use the new key. / 保存后，新的保密参数将自动使用新密钥加密。

**Important / 重要**

After key replacement, the old key must remain enabled in KMS. Do not delete or disable it, otherwise historical secret data encrypted with the old key will become undecryptable.

更换密钥后，旧密钥必须保持在 KMS 中处于启用状态，不可删除或禁用，否则使用旧密钥加密的历史保密数据将无法解密。

### Key Deletion / 密钥删除注意事项

Deleting or disabling the active encryption key will cause all secret parameters encrypted with it to become undecryptable, resulting in parameter sets becoming unavailable and Stack executions failing.

删除或禁用当前使用的加密密钥会导致所有依赖该密钥加密的保密参数无法解密，进而导致参数集不可用、引用该参数集的 Stack 运行失败。

Before deleting a key in the KMS console, the system automatically validates whether the key is in use by IacService. If associated encrypted resources exist (such as parameter sets, Stacks, Jobs, Tasks, etc.), the system will block the deletion and list the affected resources.

在 KMS 控制台删除密钥前，系统会自动校验该密钥是否被自动化服务台使用。若存在关联的加密资源（如参数集、Stack、Job、Task 等），系统将阻止删除并列出受影响的资源。

**Important / 重要**

Data loss caused by user-initiated key invalidation is the user's responsibility. It is recommended to confirm the key's associated usage before any operation and back up critical data in advance.

因用户主动操作导致密钥失效而引起的数据不可恢复损失，由用户自行承担。建议在操作前确认密钥的关联使用情况，并提前备份关键数据。

---

## Configure Secret Parameters / 设置保密参数

### Create Secret Parameters / 创建保密参数

On the parameter set list page, create a new parameter set or edit an existing one:

在参数集列表页面，创建新参数集或编辑已有参数集：

1. Click **Add Parameter**, fill in the parameter name and value. / 点击 **新增参数**，填写参数名称和参数值。
2. Check the **Secret** option. / 勾选 **保密** 选项。
3. Save the parameter set. / 点击保存。

The system automatically encrypts secret parameter values. After saving, values are displayed as masks and plaintext can no longer be viewed.

系统会自动对保密参数值进行加密存储。保存完成后，参数值在界面上以掩码形式展示，无法查看明文。

**Important / 重要**

- The secret attribute cannot be revoked once set. To change, delete the parameter and recreate it. / 保密属性一旦设置不可取消。如需调整，请删除该参数后重新创建。
- The plaintext value cannot be viewed after saving. Keep your original data safe. / 保密参数的明文值在保存后将不可再被查看，请妥善保管原始数据。

### Modify Secret Parameter Values / 修改保密参数值

Secret parameters support value modification. When editing a parameter set, enter a new value directly and save. The system re-encrypts the new value with the current active encryption key.

保密参数支持修改值。编辑参数集时，直接输入新的参数值并保存即可，系统会使用当前活跃的加密密钥对新值重新加密。

Since plaintext is not viewable, the old value is not required when modifying — new values overwrite directly.

由于明文值不可查看，修改时无需提供旧值，新值将直接覆盖。

### Visibility / 保密参数的可见性

Secret parameters are automatically masked in the following scenarios / 保密参数在以下场景中均会自动隐藏，不展示明文值：

- Parameter set list and detail pages / 参数集列表和详情页面
- Stack creation and configuration pages / Stack 创建和配置页面
- Stack Deployment input/output display / Stack Deployment 的输入、输出展示
- Execution logs and run records / 运行日志和执行记录
- API response data / API 接口返回数据
- Sensitive fields in Terraform State file display / Terraform State 文件中的敏感字段展示

---

## Use Secret Values in Stacks / 在 Stack 中使用保密值

### Associate Parameter Set When Creating a Stack / 创建 Stack 时关联参数集

When creating a Stack, select the parameter set in the **Associate Parameter Set** step. During deployment, the Stack automatically reads variable values from the parameter set, with secret values decrypted and injected into the runtime environment automatically.

创建 Stack 时，在 **关联参数集** 步骤选择已创建的参数集。Stack 部署时会自动从参数集中读取变量值，保密值在运行时自动解密并注入执行环境。

### Reference Parameter Set in Deployment Configuration / 在 Deployment 配置中引用参数集

Use the `store` block to reference parameter sets in Stack Deployment configuration:

通过 `store` 块可以在 Stack Deployment 配置中引用已创建的参数集：

```yaml
store:
  - type: "varset"
    store_name: "credentials"
    name: "<parameter_set_name>"
    id: "<parameter_set_id>"
    category: "terraform"

deployment:
  - name: "main"
    inputs:
      db_password: store.varset.credentials.db_password
      api_token: store.varset.credentials.api_token
```

**Fields / 字段说明：**

| Field / 字段 | Description / 描述 | Required / 必填 |
|------|------|------|
| type | Store type, currently only `varset` / 存储类型，当前仅支持 `varset` | Yes / 是 |
| store_name | Reference name within the Stack / 在 Stack 中自定义的引用名称 | Yes / 是 |
| name | Parameter set name (either `name` or `id`) / 参数集名称（与 `id` 二选一） | No / 否 |
| id | Parameter set ID (either `name` or `id`) / 参数集 ID（与 `name` 二选一） | No / 否 |
| category | Category, currently only `terraform` / 类别，当前仅支持 `terraform` | No / 否 |

**Reference format / 引用格式：** `store.<STORE_TYPE>.<STORE_NAME>.<VARIABLE_NAME>`

### Declare Sensitive Variables in Component / 组件变量声明

Component variables that receive secret parameter values must declare `sensitive: true`, otherwise the system will return an error.

若 Stack 的组件变量需要接收保密参数值，必须在变量定义中声明 `sensitive: true`，否则系统会返回错误。

```yaml
variable:
  - name: db_password
    type: string
    sensitive: true
    description: "Database password / 数据库密码"
```

Secret values are automatically decrypted during Stack execution, but remain masked in the console UI and logs.

保密值在 Stack 执行过程中会自动解密，但在控制台界面和日志中仍以掩码形式展示。

---

## How Encryption Works / 加密与隐藏原理简介

### Encrypted Storage / 加密存储

Secret values are protected via the envelope encryption mechanism of Alibaba Cloud KMS. The system generates an independent Data Encryption Key (DEK) for each resource (e.g., parameter set), and encrypts the DEK with your configured KMS master key. The secret value itself is encrypted by the DEK and stored as ciphertext, ensuring plaintext is inaccessible even if the storage layer is compromised.

保密值基于阿里云 KMS 的信封加密机制进行保护。系统为每个资源（如参数集）生成独立的数据加密密钥（DEK），并使用您配置的 KMS 主密钥对 DEK 进行加密保管。保密值本身由 DEK 加密后以密文形式存储，确保即使存储层被访问，也无法获取明文。

### Automatic Masking / 自动隐藏

The system uses multiple layers to ensure secret values are not leaked in unauthorized scenarios. In UI display, secret values are uniformly replaced with mask placeholders; in execution log output, secret fields are automatically identified and replaced; in API responses, secret fields are nulled and not sent to callers. Sensitive attributes in Terraform State files are also replaced with placeholders during display.

系统通过多层机制确保保密值不会在非授权场景中泄露。在界面展示时，保密值统一替换为掩码占位符；在运行日志输出时，自动识别并替换保密字段；在 API 接口返回时，保密字段会被置空，不会下发至调用方。Terraform State 文件中标记为 sensitive 的属性在展示时也会被替换为占位符。

### Runtime Decryption / 运行时解密

Secret values are decrypted only during the parameter merge phase before Stack execution. The decrypted plaintext is used briefly in memory and passed to the Terraform runtime, then released immediately.

保密值仅在 Stack 执行前的参数合并阶段进行解密，解密后的明文仅在内存中短暂使用，传递给 Terraform 运行时后即释放。

---

## FAQ / 常见问题

### What if I forget the original value of a secret parameter? / 如果忘记保密参数的原始值怎么办？

The plaintext value cannot be viewed after saving. If you forget the original value, the only option is to edit the parameter set and enter a new value to overwrite.

保密参数的明文值在保存后无法再被查看。如果忘记原始值，只能通过编辑参数集重新输入新值来覆盖。

### Can the secret attribute be revoked? / 可以取消参数的保密属性吗？

No. Once set, the secret attribute is permanent and cannot be reverted to a non-secret parameter. To use a non-secret parameter, delete it and recreate as a regular parameter.

不可以。保密属性一旦设置即为永久生效，不支持改回普通参数。如需使用非保密的参数，请删除该参数后重新创建为普通参数。

### Can historical secret data still be used after key rotation? / 更换加密密钥后，历史保密数据还能正常使用吗？

Yes. After key rotation, new secret data uses the new key for encryption, while historical data is still decrypted with the old key. Ensure the old key remains enabled in KMS.

可以。更换密钥后，新保密数据使用新密钥加密，历史保密数据仍通过旧密钥解密。请确保旧密钥在 KMS 中保持启用状态。

### What happens if a KMS key is disabled or deleted? / 如果禁用或删除了 KMS 密钥会怎样？

After disabling or deleting the encryption key, all secret parameters encrypted with it will become undecryptable. This will cause parameter sets to become unusable, Stack executions referencing those parameter sets to fail, and data may be unrecoverable. Always confirm the key's associated usage before performing any operation.

禁用或删除加密密钥后，所有使用该密钥加密的保密参数将无法解密。这将导致参数集无法正常使用，引用该参数集的 Stack 运行失败，且数据可能不可恢复。建议在操作前务必确认密钥的关联使用情况。

---

## Related Documentation / 相关文档

- For details on KMS keys, see [KMS Key Management Overview](https://help.aliyun.com/zh/kms/key-management-service/getting-started/getting-started-with-key-management). / 关于 KMS 密钥的详细介绍，请参考 [KMS 密钥管理概述](https://help.aliyun.com/zh/kms/key-management-service/getting-started/getting-started-with-key-management)。
