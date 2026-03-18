# PR #1120 测试分析：支持外部钱包签名的通道资金注入

> PR: https://github.com/nervosnetwork/fiber/pull/1120
> 状态：Open（截至 2026-03-18）
> 目标分支：`develop`，里程碑：v0.8
> 功能概述：Fiber 当前依赖 CKB 私钥来初始化和签名交易，但大多数 Web 钱包不支持导出私钥。此 PR 增加了通过外部签名的 funding 交易来开通通道的支持，新增两个 RPC：`open_channel_with_external_funding` 和 `submit_signed_funding_tx`。

---

## 一、变更范围总结

### 1.1 新增核心类型与状态

| 变更 | 文件 | 说明 |
|------|------|------|
| `ChannelState::AwaitingExternalFunding` | `crates/fiber-types/src/channel.rs` | 新增通道状态枚举变体，放在末尾以保持 bincode 序列化兼容 |
| `ChannelFlags::EXTERNAL_FUNDING` | `crates/fiber-types/src/channel.rs` | 新增通道标志位 `1 << 2` |
| `ExternalFundingRuntime` | `crates/fiber-lib/src/fiber/channel.rs` | 运行时状态结构：`enabled`、`signed_submitted`、`unsigned_funding_tx`、`started_at` |
| `OpenChannelWithExternalFundingParameter` | `crates/fiber-lib/src/fiber/channel.rs` | 通道初始化参数 |
| `ExternalFundingTxBuilder` | `crates/fiber-lib/src/ckb/funding/funding_tx.rs` | 新增外部资金交易构建器 |
| `ExternalFundingContext` | `crates/fiber-lib/src/ckb/funding/funding_tx.rs` | 外部资金上下文（lock_script、cell_deps） |

### 1.2 新增 RPC

| RPC 方法 | 参数类型 | 返回类型 | 说明 |
|----------|----------|----------|------|
| `open_channel_with_external_funding` | `OpenChannelWithExternalFundingParams` | `OpenChannelWithExternalFundingResult` | 返回 `channel_id` 和 `unsigned_funding_tx` |
| `submit_signed_funding_tx` | `SubmitSignedFundingTxParams` | `SubmitSignedFundingTxResult` | 提交签名后的 funding 交易 |

### 1.3 核心状态转换流程

```
OpenChannelWithExternalFunding
    │
    ↓
NegotiatingFunding → (协商完成)
    │
    ↓
CollaboratingFundingTx → (TX协作完成, 生成 unsigned_funding_tx)
    │
    ↓
AwaitingExternalFunding ← 用户需要在此阶段签名
    │
    ├─ submit_signed_funding_tx → 验证签名交易 → install → TxUpdate → SigningCommitment → ...
    │
    ├─ 如果对端先发 CommitmentSigned → SigningCommitment(THEIR_COMMITMENT_SIGNED_SENT)
    │   └─ submit_signed_funding_tx → install (preserve_signing_state) → handle_commitment_signed_command
    │
    └─ 超时 → CheckFundingTimeout → AbortFunding
```

### 1.4 配置变更

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `external_funding_timeout_seconds` | 300 (5分钟) | 等待用户提交签名交易的超时时间 |

---

## 二、测试用例清单

### 2.1 bincode 序列化兼容性测试

| 编号 | 测试用例 | 目的 | 验证点 |
|------|----------|------|--------|
| T-01 | `test_channel_state_bincode_compatibility` | 确保新增 `AwaitingExternalFunding` 状态不破坏已有状态的 bincode 编码 | 所有 `ChannelState` 变体的 bincode 序列化结果与预期字节序列一致，特别是 `AwaitingExternalFunding` 的 discriminant 为 8 |

**具体验证项**：
- `NegotiatingFunding(empty)` → `[0, 0, 0, 0, 0, 0, 0, 0]`
- `CollaboratingFundingTx(empty)` → `[1, 0, 0, 0, 0, 0, 0, 0]`
- `SigningCommitment(empty)` → `[2, 0, 0, 0, 0, 0, 0, 0]`
- `AwaitingTxSignatures(empty)` → `[3, 0, 0, 0, 0, 0, 0, 0]`
- `AwaitingChannelReady(empty)` → `[4, 0, 0, 0, 0, 0, 0, 0]`
- `ChannelReady` → `[5, 0, 0, 0]`
- `ShuttingDown(empty)` → `[6, 0, 0, 0, 0, 0, 0, 0]`
- `Closed(empty)` → `[7, 0, 0, 0, 0, 0, 0, 0]`
- `AwaitingExternalFunding` → `[8, 0, 0, 0]`

---

### 2.2 外部资金通道开通 - 正常流程

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-02 | `test_open_channel_with_external_funding` | 验证外部资金通道开通的基本流程 | node_b 启用 `auto_accept_channel` | 1. 返回有效的 `channel_id`（非默认值）<br>2. `unsigned_funding_tx` 有至少一个 output<br>3. 通道状态在提交签名前**不应被持久化**到 store 中 |
| T-03 | `test_submit_signed_funding_tx` | 验证提交签名后的 funding 交易的正常流程 | 先通过 `open_external_funding_channel` 创建通道 | 1. 提交成功<br>2. 返回的 `tx_hash` 与签名交易的 hash 一致<br>3. 通道状态已从 `AwaitingExternalFunding` 转换到后续阶段（`CollaboratingFundingTx` / `SigningCommitment` / `AwaitingTxSignatures` / `AwaitingChannelReady`） |
| T-04 | `test_submit_signed_funding_tx_unblocks_acceptor_commitment_handshake` | 验证提交签名交易后，发起方和接受方都能顺利完成 commitment 握手 | 先通过 `open_external_funding_channel` 创建通道 | 1. 提交成功<br>2. 发起方（node_a）进入 `AwaitingChannelReady` 或 `ChannelReady`<br>3. 接受方（node_b）也进入 `AwaitingChannelReady` 或 `ChannelReady`<br>4. 使用轮询（最多 20 次 × 100ms 和 40 次 × 100ms）等待状态转换 |

---

### 2.3 超时处理

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-05 | `test_external_funding_timeout_abort` | 验证外部资金等待超时后通道被正确中止 | node_a 的 `external_funding_timeout_seconds` 设为 1 秒，node_b 启用 auto_accept | 1. 开通外部资金通道后不提交签名交易<br>2. 等待 `ChannelFundingAborted` 事件触发<br>3. 确认通道被中止 |
| T-06 | `test_external_funding_signed_submission_not_aborted_by_stale_timeout` | 验证已经成功提交签名交易后，过期的超时事件不会中止通道 | node_a 设 `external_funding_timeout_seconds=1` 和 `funding_timeout_seconds=10` | 1. 开通外部资金通道<br>2. 立即提交签名交易（在 1 秒超时前）<br>3. 等待 2 秒后<br>4. 通道状态仍然存在于 store 中，没有被过期的 external funding timeout 中止 |

---

### 2.4 错误处理 - 状态检查

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-07 | `test_submit_signed_funding_tx_wrong_state` | 验证对非外部资金通道提交签名交易会被拒绝 | 使用普通 `OpenChannel` 开通通道 | 1. 提交失败<br>2. 错误信息包含 "AwaitingExternalFunding" 或 "InvalidState" |
| T-08 | `test_submit_signed_funding_tx_duplicate` | 验证重复提交签名交易会被拒绝 | 先成功提交一次签名交易 | 1. 第一次提交成功<br>2. 第二次提交失败<br>3. 错误信息包含 "already been submitted"、"InvalidState" 或 "AwaitingExternalFunding" |

---

### 2.5 错误处理 - 交易验证

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-09 | `test_submit_signed_funding_tx_output_mismatch` | 验证提交的签名交易 output 与原始 unsigned tx 不一致时被拒绝 | 开通外部资金通道 | 1. 构造一个 output 不同的交易<br>2. 提交失败<br>3. 错误信息包含 "mismatch" 或 "InvalidParameter" |
| T-10 | `test_submit_signed_funding_tx_input_count_mismatch` | 验证提交的签名交易 input 数量与原始 unsigned tx 不一致时被拒绝 | 开通外部资金通道 | 1. 在 unsigned tx 基础上额外添加一个 input<br>2. 提交失败<br>3. 错误信息包含 "Input count mismatch" 或 "mismatch" |

---

### 2.6 参数验证

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-11 | `test_external_funding_invalid_tlc_expiry_delta` | 验证使用过小的 TLC expiry delta 开通外部资金通道会被拒绝 | 两个互连节点 | 1. 使用 `tlc_expiry_delta=1`（远小于 MIN_TLC_EXPIRY_DELTA）<br>2. 开通失败<br>3. 错误信息包含 "TLC expiry delta" |
| T-12 | `test_external_funding_invalid_commitment_delay` | 验证使用过小的 commitment delay epoch 开通外部资金通道会被拒绝 | 两个互连节点 | 1. 使用 `commitment_delay_epoch = EpochNumberWithFraction::new(0, 0, 1)`<br>2. 开通失败<br>3. 错误信息包含 "commitment delay" 或 "Commitment delay" |

---

### 2.7 通道生命周期 - 异常中止

| 编号 | 测试用例 | 目的 | 前提条件 | 验证点 |
|------|----------|------|----------|--------|
| T-13 | `test_external_funding_pending_reply_returns_error_when_channel_stops` | 验证当通道在等待外部资金时被中止（abandon），pending 的 RPC 调用能正确返回错误 | 两个互连节点（不启用 auto_accept） | 1. 在单独的异步任务中发起 `OpenChannelWithExternalFunding`（因为没有 auto_accept，会阻塞等待）<br>2. 等待 `ChannelCreated` 事件获取临时 channel_id<br>3. 调用 `AbandonChannel` 中止通道<br>4. 验证 `OpenChannelWithExternalFunding` 的 pending reply 返回错误<br>5. 错误信息包含 "stopped before unsigned external funding tx was returned" |

---

## 三、PR 中已包含的测试与建议补充

### 3.1 PR 已包含的测试

PR 在 `crates/fiber-lib/src/fiber/tests/channel.rs` 中新增了以下 12 个测试：

| 类别 | 测试函数 | 对应编号 |
|------|----------|----------|
| 序列化 | `test_channel_state_bincode_compatibility` | T-01 |
| 正常流程 | `test_open_channel_with_external_funding` | T-02 |
| 正常流程 | `test_submit_signed_funding_tx` | T-03 |
| 正常流程 | `test_submit_signed_funding_tx_unblocks_acceptor_commitment_handshake` | T-04 |
| 超时 | `test_external_funding_timeout_abort` | T-05 |
| 超时 | `test_external_funding_signed_submission_not_aborted_by_stale_timeout` | T-06 |
| 错误-状态 | `test_submit_signed_funding_tx_wrong_state` | T-07 |
| 错误-状态 | `test_submit_signed_funding_tx_duplicate` | T-08 |
| 错误-交易 | `test_submit_signed_funding_tx_output_mismatch` | T-09 |
| 错误-交易 | `test_submit_signed_funding_tx_input_count_mismatch` | T-10 |
| 参数验证 | `test_external_funding_invalid_tlc_expiry_delta` | T-11 |
| 参数验证 | `test_external_funding_invalid_commitment_delay` | T-12 |
| 生命周期 | `test_external_funding_pending_reply_returns_error_when_channel_stops` | T-13 |

此外还包含：
- `crates/fiber-wasm/tests/optional_params.rs`（WASM 可选参数测试）
- `tests/bruno/e2e/external-funding-open/`（端到端测试，需手动执行）

### 3.2 建议补充的测试用例

以下是 PR 当前未覆盖但值得补充的测试场景：

| 编号 | 建议测试用例 | 类别 | 说明 |
|------|------------|------|------|
| T-14 | `test_submit_signed_funding_tx_cell_dep_mismatch` | 交易验证 | 验证 `cell_deps` 被篡改时是否被拒绝（当前 `validate_external_funding_signed_tx` 未显式校验 cell_deps） |
| T-15 | `test_submit_signed_funding_tx_output_data_mismatch` | 交易验证 | 验证 `outputs_data` 被篡改时是否被正确拒绝（代码中有此验证，但无对应测试） |
| T-16 | `test_submit_signed_funding_tx_input_previous_output_mismatch` | 交易验证 | 验证某个 input 的 `previous_output` 被篡改时是否被拒绝（代码中有逐个 input 的 previous_output 校验） |
| T-17 | `test_external_funding_with_udt` | 功能覆盖 | 验证使用 UDT（自定义 token）类型的外部资金通道开通和签名提交流程 |
| T-18 | `test_external_funding_with_custom_cell_deps` | 功能覆盖 | 验证 `funding_lock_script_cell_deps` 参数是否被正确传递和使用 |
| T-19 | `test_external_funding_not_final_tx` | 交易验证 | 验证签名交易未达到 "final" 状态时被拒绝（代码中有 `is_tx_final` 校验） |
| T-20 | `test_external_funding_channel_not_found_on_submit` | 错误处理 | 验证 `submit_signed_funding_tx` 使用不存在的 `channel_id` 时的错误处理 |
| T-21 | `test_external_funding_channel_restart_recovery` | 持久化 | 验证在 `AwaitingExternalFunding` 状态下节点重启后的恢复行为（`should_persist_channel_state` 在 `enabled && !signed_submitted` 时返回 false，所以重启应丢失此通道） |
| T-22 | `test_external_funding_acceptor_side_state` | 对端行为 | 验证接受方（non-initiator）在对端使用外部资金时的状态转换和行为是否正确 |
| T-23 | `test_external_funding_concurrent_operations` | 并发 | 验证在 `AwaitingExternalFunding` 状态下同时发送其他通道命令（如 shutdown）的行为 |
| T-24 | `test_external_funding_abandon_channel` | 生命周期 | 验证在 `AwaitingExternalFunding` 状态下调用 `AbandonChannel` 能正确中止通道（`can_abort_funding` 已包含此状态） |
| T-25 | `test_external_funding_peer_commitment_signed_race` | 竞态条件 | 验证对端在 `AwaitingExternalFunding` 时提前发送 `CommitmentSigned`，然后用户提交签名交易的完整流程（代码中有处理此场景的逻辑） |
| T-26 | `test_open_channel_with_external_funding_rpc_serialization` | RPC | 验证 `OpenChannelWithExternalFundingParams` 和 `SubmitSignedFundingTxParams` 的 JSON 序列化/反序列化，确保 `unsigned_funding_tx` 和 `signed_funding_tx` 作为 JSON object 正确处理 |
| T-27 | `test_external_funding_public_channel` | 功能覆盖 | 验证外部资金通道设为 public（`public=true`）时的行为，确保 `EXTERNAL_FUNDING` flag 与 `PUBLIC` flag 可以共存 |
| T-28 | `test_external_funding_shutdown_script_required` | 参数验证 | 验证 `shutdown_script` 为空/无效时的处理（外部资金通道要求提供 shutdown_script） |

---

## 四、验证逻辑详细分析

### 4.1 `validate_external_funding_signed_tx` 验证清单

此函数是签名交易验证的核心，按以下顺序检查：

| 步骤 | 检查项 | 错误类型 | 测试覆盖 |
|------|--------|----------|----------|
| 1 | `external_funding.enabled == true` | `InvalidState` | T-07 |
| 2 | `unsigned_funding_tx` 存在 | `InvalidState` | 隐式覆盖 |
| 3 | input 数量一致 | `InvalidParameter` | T-10 |
| 4 | 每个 input 的 `previous_output` 一致 | `InvalidParameter` | 建议 T-16 |
| 5 | output 数量一致 | `InvalidParameter` | T-09 |
| 6 | 每个 output 的内容一致 | `InvalidParameter` | T-09 |
| 7 | output data 数量一致 | `InvalidParameter` | 建议 T-15 |
| 8 | 每个 output data 的内容一致 | `InvalidParameter` | 建议 T-15 |
| 9 | `is_tx_final` 为 true | `InvalidParameter` | 建议 T-19 |

### 4.2 状态持久化策略

```rust
fn should_persist_channel_state(&self, state: &ChannelActorState) -> bool {
    let external_funding = &state.ephemeral_config.external_funding;
    !external_funding.enabled || external_funding.signed_submitted
}
```

- 普通通道（`enabled=false`）：始终持久化 → `true`
- 外部资金通道 `AwaitingExternalFunding`（`enabled=true, signed_submitted=false`）：**不持久化** → `false`
- 外部资金通道签名提交后（`enabled=true, signed_submitted=true`）：持久化 → `true`

### 4.3 超时处理逻辑

```rust
fn has_funding_timeout_elapsed(&self) -> bool
```

超时判定规则：
- **外部资金超时**：`current_time - started_at > external_funding_timeout_seconds`
  - 其中 `started_at` 是存储 unsigned funding tx 时记录的时间戳
  - 默认超时时间为 300 秒（5 分钟）
- **常规 funding 超时**：`current_time - created_at > funding_timeout_seconds`
  - 其中 `created_at` 是通道创建时间
  - 默认超时时间为 86400 秒（1 天）
- 超时事件到达时，只有当前适用的超时确实已过期才中止通道，避免过期（stale）的超时事件影响已进入下一阶段的通道
- 当 `signed_submitted=true` 后，`started_at` 被清空，此后不再触发外部资金超时，而是回退到常规 funding 超时逻辑

---

## 五、e2e 测试

PR 包含手动 e2e 测试脚本，位于 `tests/bruno/e2e/external-funding-open/`：

```bash
# 启动节点
REMOVE_OLD_STATE=y ./tests/nodes/start.sh e2e/external-funding-open

# 等待节点初始化
./tests/nodes/wait.sh

# 执行成功流程测试
./tests/bruno/e2e/external-funding-open/run-success-flow.sh
```

e2e 测试步骤：
1. 连接两个节点
2. 获取 node1 和 node3 的 funding script
3. 获取 node1 开通前的余额
4. 调用 `open_channel_with_external_funding` 并获取 unsigned tx
5. 使用 ckb-cli 签名交易
6. 调用 `submit_signed_funding_tx` 提交签名交易
7. 等待通道就绪
8. 验证通道列表
9. 验证余额变化

---

## 六、测试辅助设施

PR 新增了以下测试辅助函数：

| 函数 | 文件 | 说明 |
|------|------|------|
| `new_2_nodes_with_auto_accept()` | `tests/channel.rs` | 创建两个互连节点，node_b 启用 auto_accept |
| `open_external_funding_channel()` | `tests/channel.rs` | 调用 `OpenChannelWithExternalFunding` 并返回 `(channel_id, unsigned_tx)` |
| `NetworkNodeConfigBuilder` | `test_utils.rs` | 新增配置构建器，支持自定义 fiber config |

`NetworkNodeConfigBuilder` 提供了灵活的节点配置能力：
- `.node_name()` - 设置节点名称
- `.base_dir_prefix()` - 设置临时目录前缀
- `.fiber_config_updater()` - 通过闭包自定义 FiberConfig
