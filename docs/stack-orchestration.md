# Stack Orchestration Guide / Stack 编排使用指南

This document describes how to use Stack orchestration to connect multiple Stacks, pass data between them, and enable automatic联动 (downstream auto-trigger).

本文档介绍如何使用 Stack 编排功能连接多个资源栈、在 Stack 之间传递数据，以及启用自动联动（下游自动触发）。

---

## Overview / 概述

Stack orchestration allows you to:

- **Consume upstream outputs**: Reference the output of one Stack as input to another Stack
- **Publish outputs for downstream**: Expose specific outputs so other Stacks can consume them
- **Automatic downstream triggering**: When an upstream Stack's published output changes, all downstream Stacks automatically run `terraform plan`

Stack 编排使你能够：

- **消费上游输出**：将一个 Stack 的输出引用为另一个 Stack 的输入
- **发布输出供下游消费**：暴露特定输出，让其他 Stack 可以消费
- **自动触发下游**：当上游 Stack 的已发布输出发生变更时，所有下游 Stack 自动执行 `terraform plan`

---

## Step 1: Declare Upstream Inputs / 声明上游输入

In the Stack **Deployment** file, use the `upstream_input` block to declare which upstream Stacks you want to consume.

在 Stack **Deployment** 文件中，使用 `upstream_input` 块声明要消费的上游 Stack。

```yaml
upstream_input:
  - name: network_stack        # Alias used in references / 在引用时使用的别名
    type: stack                 # Type, currently only supports "stack" / 类型，当前仅支持 "stack"
    source: "{IaCEndpoint}/{AccountId}/{StackName}"  # Upstream Stack identifier / 上游 Stack 标识
```

**Fields / 字段说明：**

| Field / 字段 | Description / 描述 | Required / 必填 | Example / 示例 |
|------|------|------|--------|
| name | Alias used to reference this upstream in `deployment.inputs` / 在 `deployment.inputs` 中引用此上游时使用的别名 | Yes / 是 | network_stack |
| type | Input type, currently only supports `stack` / 输入类型，当前仅支持 `stack` | Yes / 是 | stack |
| source | Upstream Stack identifier in format `{IaCEndpoint}/{AccountId}/{StackName}` / 上游 Stack 标识，格式为 `{IaCEndpoint}/{AccountId}/{StackName}` | Yes / 是 | iac.aliyuncs.com/123456/network-infra |

**Limits / 限制：**

- A Stack can declare at most **20** upstream inputs / 一个 Stack 最多可声明 **20** 个上游输入

**Note / 注意事项：**

- If the upstream Stack belongs to a different account, you must first set up a sharing authorization / 如果上游 Stack 属于不同账号，需先完成共享授权

---

## Step 2: Reference Upstream Outputs in Deployments / 在 Deployment 中引用上游输出

In the `deployment.inputs` block, reference upstream outputs using the format:

在 `deployment.inputs` 块中，使用以下格式引用上游输出：

```
upstream_input.<alias>.<output_name>
```

**Supported reference formats / 支持的引用格式：**

- Direct reference / 直接引用：`upstream_input.network_stack.vpc_id`
- Template reference / 模板引用：`{{ upstream_input.network_stack.vpc_id }}` or `${ upstream_input.network_stack.vpc_id }`

**Example / 示例：**

```yaml
deployment:
  - name: production
    inputs:
      vpc_id: upstream_input.network_stack.vpc_id
      subnet_id: upstream_input.network_stack.subnet_id
      region: cn-shanghai

  - name: staging
    inputs:
      vpc_id: upstream_input.network_stack.vpc_id
      region: cn-shanghai
```

Each deployment that references an upstream output will create a consumption relationship record. If the same upstream output is referenced in multiple deployments, each reference is tracked separately.

每个引用了上游输出的 deployment 都会创建一条消费关系记录。如果同一个上游输出在多个 deployment 中被引用，每条引用会单独记录。

---

## Step 3: Declare Publish Outputs / 声明发布输出

In the Stack **Deployment** file, use the `publish_output` block to declare which outputs you want to expose for downstream Stacks to consume.

在 Stack **Deployment** 文件中，使用 `publish_output` 块声明要暴露给下游 Stack 消费的输出。

```yaml
publish_output:
  - name: vpc_id
    description: "Production VPC ID"
    value: deployment.production.vpc_id

  - name: subnet_id
    description: "Production Subnet ID"
    value: deployment.production.subnet_id
```

**Fields / 字段说明：**

| Field / 字段 | Description / 描述 | Required / 必填 | Example / 示例 |
|------|------|------|--------|
| name | Unique identifier for this published output / 该发布输出的唯一标识 | Yes / 是 | vpc_id |
| description | Description of the output / 输出的描述 | No / 否 | Production VPC ID |
| value | The actual output value, referencing a deployment output / 实际输出值，引用某个 deployment 的输出 | Yes / 是 | deployment.production.vpc_id |

**Note / 注意事项：**

- Only outputs listed in `publish_output` can be consumed by downstream Stacks. Other outputs from the Stack Component remain internal.
  - 只有在 `publish_output` 中声明的输出才能被下游 Stack 消费。Stack Component 的其他输出保持内部使用。
- The `value` field supports template syntax: `deployment.<deployment_name>.<output_name>`
  - `value` 字段支持模板语法：`deployment.<deployment名称>.<输出名称>`

---

## Step 4: View Orchestration Relationships / 查看编排关系

You can query the upstream and downstream relationships of a Stack via the API.

可以通过 API 查询 Stack 的上游和下游编排关系。

**API / 接口：** `GET /stacks/{stackId}/outputConsumers`

**Query parameters / 请求参数：**

| Parameter / 参数 | Required / 必填 | Values / 可选值 | Description / 描述 |
|------|------|------|------|
| type | Yes / 是 | `upstream` / `downstream` | Query direction: `upstream` shows what this Stack consumes; `downstream` shows what Stacks consume this Stack's outputs / 查询方向：`upstream` 显示当前 Stack 消费了什么；`downstream` 显示哪些 Stack 消费当前 Stack 的输出 |
| pageNumber | No / 否 | integer | Page number, starts from 1 / 页码，从 1 开始 |
| pageSize | No / 否 | integer | Items per page / 每页条数 |
| sortBy | No / 否 | `stackName` / `publishOutputAddress` | Sort field / 排序字段 |
| sortOrder | No / 否 | `asc` / `desc` | Sort direction / 排序方向 |
| keyword | No / 否 | string | Keyword for fuzzy search / 关键字模糊搜索 |

---

## Automatic Downstream Trigger / 自动触发下游

### How it works / 工作原理

After an upstream Stack is successfully deployed, the system:

上游 Stack 成功部署后，系统会：

1. **Updates the parameter set** with the resolved `publish_output` values / 将解析后的 `publish_output` 值更新到参数集
2. **Detects changes** by comparing the new output values against the previous ones / 对比新输出值与旧值，检测是否发生变更
3. **Publishes a change event** if the output content has actually changed / 如果输出内容确实发生变更，发布变更事件
4. **Triggers `terraform plan`** for all downstream Stacks that consume this upstream / 为所有消费该上游的下游 Stack 触发 `terraform plan`

### Trigger conditions / 触发条件

- The upstream Stack must be in a **deployed terminal state** (`FINISH`) / 上游 Stack 必须处于**已部署终态**（`FINISH`）
- The published output values must have **actually changed** (new value ≠ old value) / 已发布输出值必须**实际发生变更**（新值 ≠ 旧值）
- If the output values are the same as before, no event is published and no downstream is triggered / 如果输出值与之前相同，则不会发布事件，也不会触发下游

### Idempotency / 幂等性

The system ensures that the same change event does not trigger duplicate downstream runs. If a trigger for the same upstream + version + deployment is already in progress, subsequent events are safely ignored.

系统确保同一变更事件不会重复触发下游运行。如果针对同一上游 + 版本 + deployment 的触发正在进行中，后续事件会被安全忽略。

---

## Cross-Account Orchestration / 跨账号编排

When a downstream Stack wants to consume an upstream Stack from a **different account**, a sharing authorization must be set up first.

当下游 Stack 需要消费**不同账号**的上游 Stack 时，需要先完成共享授权。

- Without authorization, the deployment configuration update will be rejected with an error: `STACK_OUTPUT_CONSUMER_UPSTREAM_STACK_NOT_AUTHORIZED`
  - 未授权时，部署配置更新将被拒绝，报错：`STACK_OUTPUT_CONSUMER_UPSTREAM_STACK_NOT_AUTHORIZED`
- Canceling the sharing authorization will soft-delete all associated consumption relationships
  - 取消共享授权将软删除所有关联的消费关系

---

## Complete Example / 完整示例

### Upstream Stack: Network Infrastructure / 上游 Stack：网络基础设施

**Stack Deployment file (`deployment.yaml`)**：

```yaml
format_version: IaCService/2021-08-06
description: Network infrastructure stack that publishes VPC outputs

publish_output:
  - name: vpc_id
    description: "Production VPC ID"
    value: deployment.production.vpc_id
  - name: subnet_id
    description: "Production Subnet ID"
    value: deployment.production.subnet_id

deployment:
  - name: production
    inputs:
      region: cn-shanghai
      vpc_cidr: "10.0.0.0/16"
```

### Downstream Stack: Application / 下游 Stack：应用部署

**Stack Deployment file (`deployment.yaml`)**：

```yaml
format_version: IaCService/2021-08-06
description: Application stack that consumes network outputs

upstream_input:
  - name: network_stack
    type: stack
    source: "iac.aliyuncs.com/123456/network-infra"

deployment:
  - name: production
    inputs:
      vpc_id: upstream_input.network_stack.vpc_id
      subnet_id: upstream_input.network_stack.subnet_id
      app_name: my-application
      app_env: production

  - name: staging
    inputs:
      vpc_id: upstream_input.network_stack.vpc_id
      app_name: my-application
      app_env: staging
```

In this example:

在此示例中：

- The Application Stack declares that it consumes outputs from the `network-infra` Stack / 应用 Stack 声明消费 `network-infra` Stack 的输出
- `vpc_id` and `subnet_id` from the upstream are referenced in both `production` and `staging` deployments / 上游的 `vpc_id` 和 `subnet_id` 在 `production` 和 `staging` 两个 deployment 中都被引用
- This creates **4 consumption relationship records** (2 outputs × 2 deployments) / 这将创建 **4 条消费关系记录**（2 个输出 × 2 个 deployment）
- When the upstream Stack redeploys and `vpc_id` or `subnet_id` changes, the Application Stack will automatically run `terraform plan` / 当上游 Stack 重新部署且 `vpc_id` 或 `subnet_id` 发生变更时，应用 Stack 将自动执行 `terraform plan`

---

## Limits / 限制

| Limit / 限制 | Value / 值 | Description / 描述 |
|------|------|------|
| Max upstream inputs / 最大上游输入数 | 20 | Maximum `upstream_input` entries per Stack / 每个 Stack 最多可声明的 `upstream_input` 条目数 |
| Max downstream Stacks / 最大下游 Stack 数 | 25 | Maximum Stacks that can consume a single upstream Stack's outputs / 最多可消费单个上游 Stack 输出的 Stack 数量 |

---

## Error Codes / 错误码

| Error Code / 错误码 | Description / 描述 | Solution / 解决方法 |
|------|------|------|
| `STACK_CONSUMERS_COUNT_EXCEED_LIMIT` | Downstream Stack count exceeds limit (max 25) / 下游 Stack 数量超过限制（最大 25） | Reduce the number of downstream Stacks / 减少下游 Stack 数量 |
| `STACK_OUTPUT_CONSUMER_UPSTREAM_STACK_STATUS_INVALID` | Upstream Stack is not in a deployed terminal state / 上游 Stack 未处于已部署终态 | Ensure the upstream Stack deployment has completed successfully / 确保上游 Stack 部署已成功完成 |
| `STACK_OUTPUT_CONSUMER_UPSTREAM_STACK_NOT_FOUND` | Upstream Stack does not exist / 上游 Stack 不存在 | Check the `source` field and ensure the upstream Stack name is correct / 检查 `source` 字段，确认上游 Stack 名称正确 |
| `STACK_OUTPUT_CONSUMER_UPSTREAM_STACK_NOT_AUTHORIZED` | Cross-account upstream Stack is not authorized / 跨账号上游 Stack 未授权 | Set up sharing authorization first / 先完成共享授权 |
| `STACK_DEPLOYMENT_CONFIG_REFERENCE_INVALID` | Referenced upstream_input alias is not declared / 引用了未声明的 upstream_input 别名 | Ensure all `upstream_input.*` references match a declared alias / 确保所有 `upstream_input.*` 引用都匹配已声明的别名 |

---

## Related Documentation / 相关文档

- [Stack Syntax Reference / 资源栈语法说明](./stack-syntax.md) — Full YAML syntax for Stack Component and Deployment / Stack Component 和 Deployment 的完整 YAML 语法
- [Secret Parameters Guide / 参数集保密值使用指南](./secret-parameters.md) — How to manage and use secret parameters in Stacks / 如何在 Stack 中管理和使用保密参数
