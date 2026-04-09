# PR #1120 集成测试用例：支持外部钱包签名的通道资金注入

> PR: https://github.com/nervosnetwork/fiber/pull/1120  
> 功能概述：新增两个 RPC —— `open_channel_with_external_funding` 和 `submit_signed_funding_tx`，允许用户使用外部钱包签名 funding 交易来开通通道，不再要求 FNN 节点持有 CKB 私钥。

---

## 环境准备

- 至少部署 2 个 FNN 节点（以下称 **Node A**、**Node B**），已连通对等方
- Node B 建议配置 `auto_accept_channel_ckb_funding_amount`（自动接受通道），简化测试流程
- 准备一个外部钱包地址（有足够 CKB 余额），其 lock script 用于 `funding_lock_script`
- 准备 `ckb-cli` 或其他签名工具，可对 CKB 交易进行签名
- 以下所有操作通过 JSON-RPC 调用完成

---

## 一、正常流程

### T-01 外部资金通道开通并获取未签名交易

**目的**：验证 `open_channel_with_external_funding` 能正确返回 channel_id 和未签名交易

**步骤**：
1. 确保 Node A 和 Node B 已互相连接（通过 `connect_peer`）
2. 准备好外部钱包的 lock script（即你控制的钱包地址对应的 lock script，可通过 `ckb-cli` 或钱包 SDK 获取）
3. 向 Node A 发送 RPC 请求：
   ```json
   {
     "jsonrpc": "2.0",
     "method": "open_channel_with_external_funding",
     "params": [{
       "pubkey": "<Node B 的 pubkey>",
       "funding_amount": "0xba43b7400",
       "public": true,
       "shutdown_script": { "code_hash": "...", "hash_type": "...", "args": "..." },
       "funding_lock_script": { "code_hash": "...", "hash_type": "...", "args": "..." }
     }]
   }
   ```
4. 检查返回结果

**预期结果**：
- 返回 `result.channel_id`：非空的 32 字节 hex 字符串（`0x` 前缀 + 64 位十六进制）
- 返回 `result.unsigned_funding_tx`：一个完整的 CKB Transaction JSON 对象
- `unsigned_funding_tx.outputs` 至少包含一个 output
- `unsigned_funding_tx.witnesses` 为占位见证（placeholder）：通常存在，但签名区为全零（未签名状态）

---

### T-02 提交签名交易，完成通道开通

**目的**：验证 `submit_signed_funding_tx` 能正确接受签名后的交易并推进通道状态

**前提**：已完成 T-01，拿到 `channel_id` 和 `unsigned_funding_tx`

**步骤**：
1. 使用外部钱包/`ckb-cli` 对 `unsigned_funding_tx` 进行签名，得到带 witnesses 的签名交易  
   - 注意：**不能修改交易的 inputs、outputs、outputs_data**，只能**替换**占位 witnesses 为真实签名 witnesses
2. 向 Node A 发送 RPC 请求：
   ```json
   {
     "jsonrpc": "2.0",
     "method": "submit_signed_funding_tx",
     "params": [{
       "channel_id": "<T-01 返回的 channel_id>",
       "signed_funding_tx": { "<签名后的完整交易 JSON>": "..." }
     }]
   }
   ```
3. 检查返回结果
4. 通过 `list_channels` 查询通道状态

**预期结果**：
- 返回 `result.channel_id`：与提交时一致
- 返回 `result.funding_tx_hash`：非空，为签名交易的 hash
- 通过 Node A 的 `list_channels` 查询，通道状态不再是 `AwaitingExternalFunding`，已推进到后续阶段

---

### T-03 通道完整生命周期：开通 → 支付 → 关闭

**目的**：验证外部资金通道从开通到关闭的完整流程

**前提**：Node B 启用 `auto_accept_channel_ckb_funding_amount`

**步骤**：
1. Node A 调用 `open_channel_with_external_funding` 开通通道，获取 `channel_id` 和 `unsigned_funding_tx`
2. 使用外部钱包签名交易
3. Node A 调用 `submit_signed_funding_tx` 提交签名交易
4. 生成若干区块（使 funding 交易上链确认）
5. 轮询 `list_channels` 等待双方通道状态变为 `ChannelReady`（建议轮询间隔 1-2 秒，最多等 60 秒）
6. Node B 调用 `new_invoice` 生成 invoice
7. Node A 调用 `send_payment` 使用该 invoice 向 Node B 支付
8. Node A 调用 `shutdown_channel` 关闭通道
9. 生成若干区块（使关闭交易上链确认）
10. 通过 `list_channels`（带 `include_closed: true`）查看通道状态

**预期结果**：
- 步骤 5：双方通道状态最终变为 `ChannelReady`
- 步骤 7：支付成功，无错误返回
- 步骤 10：通道状态变为 `Closed`
- 检查通道内余额变化：通过 `list_channels` 返回的 `local_balance` / `remote_balance` 验证支付金额转移
- 检查链上余额变化：通过 `ckb-cli` 或 CKB RPC 查询外部钱包地址余额，验证 funding amount + 手续费扣减

---

### T-04 提交签名后双方均进入就绪状态

**目的**：验证提交签名交易后，发起方和接受方都能顺利完成 commitment 握手并最终进入 ChannelReady

**前提**：Node B 启用 auto_accept

**步骤**：
1. Node A 调用 `open_channel_with_external_funding`
2. 签名 `unsigned_funding_tx`
3. Node A 调用 `submit_signed_funding_tx`
4. 生成若干区块
5. 分别在 Node A 和 Node B 上轮询 `list_channels`

**预期结果**：
- Node A 端通道状态最终变为 `AwaitingChannelReady` 或 `ChannelReady`
- Node B 端通道状态也最终变为 `AwaitingChannelReady` 或 `ChannelReady`

---

## 二、错误处理 - 交易验证

### T-05 提交 output 被篡改的交易

**目的**：验证签名交易的 outputs 与原始 unsigned tx 不一致时被拒绝

**前提**：已完成 `open_channel_with_external_funding` 获取 `channel_id` 和 `unsigned_funding_tx`

**步骤**：
1. 手动构造一个交易 JSON，**修改其中一个 output 的 capacity 或 lock script**（与 `unsigned_funding_tx` 不同）
2. 对此篡改后的交易进行签名
3. 调用 `submit_signed_funding_tx` 提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 "mismatch" 相关描述

---

### T-06 提交 input 数量不一致的交易

**目的**：验证签名交易的 input 数量与原始 unsigned tx 不一致时被拒绝

**前提**：已完成 `open_channel_with_external_funding` 获取 `channel_id` 和 `unsigned_funding_tx`

**步骤**：
1. 在 `unsigned_funding_tx` 的 `inputs` 数组中额外添加一个 input
2. 对此篡改后的交易进行签名
3. 调用 `submit_signed_funding_tx` 提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 "Input count mismatch" 相关描述

---

### T-07 提交 output_data 被篡改的交易

**目的**：验证签名交易的 outputs_data 被修改时被拒绝

**前提**：已完成 `open_channel_with_external_funding`

**步骤**：
1. 修改 `unsigned_funding_tx` 中某个 `outputs_data` 条目的内容
2. 签名并提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 "mismatch" 相关描述

---

### T-08 提交 input 的 previous_output 被篡改的交易

**目的**：验证签名交易中某个 input 的 `previous_output`（tx_hash 或 index）与原始 unsigned tx 不一致时被拒绝

**前提**：已完成 `open_channel_with_external_funding`

**步骤**：
1. 修改 `unsigned_funding_tx` 中某个 input 的 `previous_output.tx_hash` 或 `previous_output.index`
2. 签名并提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 "previous_output mismatch" 相关描述

---

### T-22 提交 output 数量不一致的交易

**目的**：验证签名交易的 output 数量与原始 unsigned tx 不一致时被拒绝（数量维度）

**前提**：已完成 `open_channel_with_external_funding` 获取 `channel_id` 和 `unsigned_funding_tx`

**步骤**：
1. 在 `unsigned_funding_tx.outputs` 中增加或删除一个 output（并同步调整 `outputs_data` 长度，避免触发其他校验）
2. 对此篡改后的交易进行签名
3. 调用 `submit_signed_funding_tx` 提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 "Output count mismatch" 或等价 mismatch 描述

---

### T-23 提交 output_data 数量不一致的交易

**目的**：验证签名交易 `outputs_data` 数组长度与 `outputs` 不匹配时被拒绝（数量维度）

**前提**：已完成 `open_channel_with_external_funding`

**步骤**：
1. 删除或新增一个 `outputs_data` 条目，使长度与 `outputs` 不一致
2. 签名并提交

**预期结果**：
- RPC 返回错误
- 错误信息包含 mismatch 相关描述

---

## 三、错误处理 - 状态检查

### T-09 对普通通道调用 submit_signed_funding_tx

**目的**：验证对非外部资金通道提交签名交易会被拒绝

**步骤**：
1. 使用普通 `open_channel` RPC 开通一个通道，获取 `channel_id`
2. 构造一个任意的交易 JSON
3. 调用 `submit_signed_funding_tx`：
   ```json
   {
     "jsonrpc": "2.0",
     "method": "submit_signed_funding_tx",
     "params": [{
       "channel_id": "<普通通道的 channel_id>",
       "signed_funding_tx": { "<任意交易>": "..." }
     }]
   }
   ```

**预期结果**：
- RPC 返回错误
- 错误信息包含 "AwaitingExternalFunding" 或 "InvalidState"

---

### T-10 重复提交签名交易

**目的**：验证对同一通道重复提交签名交易会被拒绝

**前提**：已完成外部资金通道开通并成功提交过一次签名交易

**步骤**：
1. 使用与第一次相同的签名交易，再次调用 `submit_signed_funding_tx`

**预期结果**：
- RPC 返回错误
- 错误信息包含 "already been submitted" 或 "InvalidState"

---

### T-11 对不存在的 channel_id 调用 submit_signed_funding_tx

**目的**：验证使用无效的 channel_id 时的错误处理

**步骤**：
1. 构造一个不存在的 channel_id（32 字节随机 hex）
2. 调用 `submit_signed_funding_tx`

**预期结果**：
- RPC 返回错误
- 错误信息指示通道不存在

---

## 四、参数验证

### T-12 使用过小的 tlc_expiry_delta 开通通道

**目的**：验证 `tlc_expiry_delta` 太小时 RPC 会拒绝

**步骤**：
1. 调用 `open_channel_with_external_funding`，设置 `tlc_expiry_delta` 为 `"0x1"`（远小于要求的最小值）

**预期结果**：
- RPC 返回错误
- 错误信息包含 "TLC expiry delta" 相关描述

---

### T-13 使用过小的 commitment_delay_epoch 开通通道

**目的**：验证 `commitment_delay_epoch` 为 0 或过小时 RPC 会拒绝

**步骤**：
1. 调用 `open_channel_with_external_funding`，设置 `commitment_delay_epoch` 为 `"0x0"`

**预期结果**：
- RPC 返回错误
- 错误信息包含 "commitment delay" 相关描述

---

### T-24 使用过小的 funding_amount 开通通道

**目的**：验证 `funding_amount` 为 0 或低于最小占用容量时会被拒绝

**步骤**：
1. 调用 `open_channel_with_external_funding`，设置 `funding_amount` 为 `"0x0"`（或极小值）

**预期结果**：
- RPC 返回错误
- 错误信息包含 "funding" / "capacity" 相关描述

---

### T-25 未连接 peer 时调用 open_channel_with_external_funding

**目的**：验证未建立 peer 连接时的前置条件校验

**步骤**：
1. 确保 Node A 与目标 Node B 未连接
2. 调用 `open_channel_with_external_funding`

**预期结果**：
- RPC 返回错误
- 错误信息包含 "peer" / "connect" 相关描述

---

## 五、超时与生命周期

### T-14 外部资金等待超时后通道自动中止

**目的**：验证在超时时间内未提交签名交易时，通道被自动中止

**前提**：Node A 配置 `external_funding_timeout_seconds` 为较短时间（例如 60 秒）

**步骤**：
1. Node A 调用 `open_channel_with_external_funding` 获取 `channel_id` 和 `unsigned_funding_tx`
2. **不进行签名和提交**，等待超过超时时间
3. 通过 `list_channels` 查询该通道

**预期结果**：
- 超时后通道被自动中止
- `list_channels` 中该通道不再存在，或状态显示已关闭/中止

---

### T-15 已提交签名后超时事件不影响通道

**目的**：验证已经成功提交签名交易后，即使之前调度的超时事件触发，通道不会被错误中止

**前提**：Node A 配置 `external_funding_timeout_seconds` 为较短时间（例如 60 秒）

**步骤**：
1. Node A 调用 `open_channel_with_external_funding`
2. 立即签名并调用 `submit_signed_funding_tx`（在超时时间之前）
3. 等待超过 `external_funding_timeout_seconds`
4. 通过 `list_channels` 查询通道状态

**预期结果**：
- 通道仍然存在
- 通道状态正常推进，没有被超时中止

---

### T-16 在等待签名阶段主动中止通道

**目的**：验证在 `AwaitingExternalFunding` 状态下可以通过 `abandon_channel` 主动中止通道

**步骤**：
1. Node A 调用 `open_channel_with_external_funding`，获取 `channel_id`
2. 不签名，直接调用 `abandon_channel`
3. 通过 `list_channels` 确认通道状态

**预期结果**：
- `abandon_channel` 调用成功
- 通道被中止，`list_channels` 中不再出现该通道

---

### T-17 节点重启后 AwaitingExternalFunding 状态的通道丢失

**目的**：验证在等待外部签名阶段重启节点后，该通道不会被恢复（设计上此阶段不持久化）

**步骤**：
1. Node A 调用 `open_channel_with_external_funding`，获取 `channel_id`
2. **不提交签名交易**
3. 重启 Node A
4. 通过 `list_channels` 查询

**预期结果**：
- 重启后该通道不存在于 `list_channels` 结果中
- 设计意图：`AwaitingExternalFunding` 阶段的通道不会持久化到磁盘

---

## 六、功能覆盖

### T-18 开通公共通道（public=true）

**目的**：验证外部资金通道可以设为公共通道

**步骤**：
1. 调用 `open_channel_with_external_funding`，设置 `public: true`
2. 完成签名并提交
3. 等待通道就绪后，通过 `list_channels` 查看通道信息

**预期结果**：
- 通道开通成功
- 通道信息中显示该通道为公共通道

---

### T-19 使用自定义 funding_lock_script_cell_deps

**目的**：验证 `funding_lock_script_cell_deps` 参数能被正确使用

**步骤**：
1. 准备一个需要额外 cell dep 的自定义 lock script（例如非默认钱包锁脚本）
2. 调用 `open_channel_with_external_funding`，设置 `funding_lock_script_cell_deps`
3. 检查返回的 `unsigned_funding_tx` 的 `cell_deps` 字段

**预期结果**：
- 返回的 `unsigned_funding_tx.cell_deps` 中包含了用户指定的额外 cell deps

---

### T-20 与已有 open_channel 流程的兼容性

**目的**：验证新增的外部资金功能不影响已有的普通 `open_channel` 流程

**步骤**：
1. 使用普通 `open_channel` RPC 开通一个通道
2. 等待通道就绪
3. 进行支付
4. 关闭通道

**预期结果**：
- 所有步骤正常完成，无任何行为变化
- 已有 `open_channel` 流程不受影响

---

## 七、通道状态观察

### T-21 提交签名前通过 list_channels 查看通道状态

**目的**：验证在提交签名前通道的状态展示

**步骤**：
1. Node A 调用 `open_channel_with_external_funding`
2. **不提交签名**，立即在 Node A 和 Node B 上分别调用 `list_channels`

**预期结果**：
- 两种可能结果都是正确的：
  - 通道尚未出现在 `list_channels` 结果中（因为此阶段不持久化）
  - 或通道出现但状态为 `AwaitingExternalFunding`

---

## 测试用例汇总

| 编号 | 类别 | 用例名称 | 优先级 |
|------|------|----------|--------|
| T-01 | 正常流程 | 外部资金通道开通并获取未签名交易 | P0 |
| T-02 | 正常流程 | 提交签名交易完成通道开通 | P0 |
| T-03 | 正常流程 | 完整生命周期：开通→支付→关闭 | P0 |
| T-04 | 正常流程 | 提交签名后双方均进入就绪状态 | P0 |
| T-05 | 交易验证 | 提交 output 被篡改的交易 | P1 |
| T-06 | 交易验证 | 提交 input 数量不一致的交易 | P1 |
| T-07 | 交易验证 | 提交 output_data 被篡改的交易 | P1 |
| T-08 | 交易验证 | 提交 input 的 previous_output 被篡改的交易 | P1 |
| T-09 | 状态检查 | 对普通通道调用 submit_signed_funding_tx | P1 |
| T-10 | 状态检查 | 重复提交签名交易 | P1 |
| T-11 | 状态检查 | 对不存在的 channel_id 调用 submit | P1 |
| T-12 | 参数验证 | 使用过小的 tlc_expiry_delta | P1 |
| T-13 | 参数验证 | 使用过小的 commitment_delay_epoch | P1 |
| T-14 | 超时/生命周期 | 外部资金等待超时后通道自动中止 | P1 |
| T-15 | 超时/生命周期 | 已提交签名后超时事件不影响通道 | P2 |
| T-16 | 超时/生命周期 | 等待签名阶段主动中止通道 | P1 |
| T-17 | 超时/生命周期 | 节点重启后通道丢失 | P2 |
| T-18 | 功能覆盖 | 开通公共通道 | P2 |
| T-19 | 功能覆盖 | 使用自定义 cell deps | P2 |
| T-20 | 功能覆盖 | 与已有 open_channel 流程的兼容性 | P1 |
| T-21 | 状态观察 | 提交签名前查看通道状态 | P2 |
| T-22 | 交易验证 | 提交 output 数量不一致的交易 | P1 |
| T-23 | 交易验证 | 提交 output_data 数量不一致的交易 | P1 |
| T-24 | 参数验证 | 使用过小的 funding_amount | P1 |
| T-25 | 前置条件 | 未连接 peer 时调用 | P1 |

---

## 附录：PR 单元测试与集成测试覆盖对照

PR #1120 在 `crates/fiber-lib/src/fiber/tests/channel.rs` 中新增了 13 个单元测试。以下分析每个单元测试是否已被上述集成测试用例覆盖，以及是否需要补充新的集成测试。

| 单元测试 | 测试内容 | 对应集成测试 | 是否需要补充 |
|----------|----------|-------------|-------------|
| `test_channel_state_bincode_compatibility` | 验证 ChannelState 枚举的 bincode 序列化兼容性 | 无 | **否** — 内部序列化细节，无 RPC 接口可测 |
| `test_open_channel_with_external_funding` | 开通外部资金通道，验证返回值 | **T-01** | 否 |
| `test_submit_signed_funding_tx` | 提交签名交易，验证状态推进 | **T-02** | 否 |
| `test_submit_signed_funding_tx_unblocks_acceptor_commitment_handshake` | 双方完成 commitment 握手 | **T-04** | 否 |
| `test_submit_signed_funding_tx_wrong_state` | 对普通通道提交签名交易被拒 | **T-09** | 否 |
| `test_submit_signed_funding_tx_duplicate` | 重复提交被拒 | **T-10** | 否 |
| `test_submit_signed_funding_tx_output_mismatch` | output 篡改被拒 | **T-05** | 否 |
| `test_submit_signed_funding_tx_input_count_mismatch` | input 数量不一致被拒 | **T-06** | 否 |
| `test_external_funding_invalid_tlc_expiry_delta` | 非法 tlc_expiry_delta 被拒 | **T-12** | 否 |
| `test_external_funding_invalid_commitment_delay` | 非法 commitment_delay 被拒 | **T-13** | 否 |
| `test_external_funding_timeout_abort` | 超时自动中止 | **T-14** | 否 |
| `test_external_funding_signed_submission_not_aborted_by_stale_timeout` | 已提交后超时不中止 | **T-15** | 否 |
| `test_external_funding_pending_reply_returns_error_when_channel_stops` | 通道被中止时 pending 的 RPC 调用返回错误 | 无 | **否** — 见下方说明 |

### 不需要补充的单元测试说明

**`test_channel_state_bincode_compatibility`**：验证 `ChannelState` 枚举（含新增的 `AwaitingExternalFunding`）的 bincode 序列化字节与预期一致。这是纯内部数据兼容性测试，没有对应 RPC 接口可直接验证。

**`test_external_funding_pending_reply_returns_error_when_channel_stops`**：该场景依赖 actor 内部时序（阻塞中的 pending reply + 通道中止）和内部临时 channel_id 的生命周期，集成测试通过 RPC 很难稳定复现，单元测试覆盖更合适。