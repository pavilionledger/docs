# CIP-5: Cowboy Timer Mechanism — 优先级 / 状态触发 / 公平调度（去除时间触发）

- **Status**: Draft
- **Type**: Core / Protocol
- **Author(s)**: <Your Name / Team>
- **Created**: 2025-11-10
- **Requires**: Cowboy Whitepaper v2025-11-06；CIP-2（Off-chain Verification）；CIP-3（Dual-Metered Fees）；CIP-4（Storage & State Persistence）
- **Replaces / Supersedes**: N/A
- **Discusses**: Timer 与 Mailbox 的协议级语义与调度；**不**定义 EOB 全流程（见 CIP-6）

---


```mermaid
flowchart LR
  %% CIP-5 Timer Mechanism — Swimlane-style via subgraphs

  subgraph L1[Execution]
    E1[① 产生状态变更（可能触发 Watch 条件）<br/>※ 不使用本地时钟；以区块高度/状态为准]
  end

  subgraph L2[Timers.Index]
    I1[② IndexHeight：登记/维护 due_height（周期性仅登记下一次）]
    I2[② IndexWatch：登记满足状态条件的 Watch（match_set）]
  end

  subgraph L3[Timers.Deliver]
    D1[③ 构造候选集：<br/>· IndexHeight 中 due_height ≤ h<br/>· IndexWatch 中在父状态满足条件]
    D2[④ 计算候选优先级（priority）：<br/>priority_bucket、bid_snapshot、gas_limit_snapshot、trigger_ctx …<br/>※ 禁止使用 float/本地时间]
    D3[⑤ 约束与公平性剪枝：<br/>· MAX_FIRES_PER_BLOCK / PER_TARGET<br/>· per-target RR（轮转）<br/>· MIN_GAS_LIMIT_PER_TIMER 等]
    D4[⑥ 稳定排序（stable order）：<br/>priority_bucket → RR → FIFO（同桶）]
  end

  subgraph L4[Scheduler]
    S1[⑦ 生成 deliver_id = H(h, parent_state_root, timer_id, fire_seq)]
    S2[⑧ Materialize：产出 System message（source=Timer）]
  end

  subgraph L5[Messaging]
    M1[⑨ Enqueue：将 System message 写入目标 Actor 的 MAILBOX]
    M2[（未入选候选保留在索引侧，等待后续区块）]
  end

  subgraph L6[Mailbox]
    B1[⑩ MailboxSegment 可见（尚未执行，仅完成确定性注入）]
  end

  subgraph L7["handoff to CIP-6 (EOB)"]
    C1[⑪ EOB-3 ExecuteMailbox（按 CIP-6 比较器执行）]
    C2[EOB-5 System hooks：批量写回周期 Timer 的下一次 due_height]
    C3[EOB-6 Commit：原子提交，进入下一区块]
  end

  %% Edges (data/control flow)
  E1 --> I1
  E1 --> I2
  I1 --> D1
  I2 --> D1
  D1 --> D2 --> D3 --> D4 --> S1 --> S2 --> M1 --> B1 --> C1 --> C2 --> C3
  D4 -. 未入选 .-> M2

  %% Optional: constraints summary (read-only info)
  subgraph L8[约束（摘要）]
    K1[确定性：禁止本地时钟/随机源；同一只读快照；稳定比较键]
    K2[公平性：per-target RR + 全局/每目标配额，避免饥饿]
    K3[幂等：deliver_id 保证 Materialize 不重复；reorg 可重演]
    K4[范围：CIP-5 到“入箱”为止；执行/计费/提交见 CIP-6]
  end

```

## 1. 摘要（Abstract）

本提案定义 Cowboy 协议中的 **原生 Timer 机制**：在不依赖外部 keeper 网络的前提下，为合约/Actor 提供**按区块高度**与**按状态变更**两类触发能力，并在**区块末（EOB）**以确定、可重演的方式筛选、物化并投递到各 Actor 的 **Mailbox**。

为强化确定性与实现简洁性，本提案**移除“时间触发（timestamp-based）”**，仅保留：

- **高度触发（height-triggered）**：在高度 `h` 达到或超过 `due_height` 时触发；
- **状态触发（state-triggered）**：当被监听的键在本块被修改时触发。

拥塞时的公平性通过**老化加权（age）+ 价格信号（可选出价）+ 配额限流**实现；入箱操作具有幂等性。计费遵循 **CIP-3** 的双计量市场（Cycles/Cells）。

---

## 2. 动机（Motivation）

链上系统广泛需要“准时/准条件”的回调：周期任务、到期清算、观察某状态变更后触发的逻辑等。传统做法依赖外部 keeper、cron 或预言机，存在不确定、可审计性差的问题。原生 Timer 机制应满足：

- **强确定性**：所有节点在同一高度输出相同的入箱结果；
- **公平与弹性**：拥塞时按“老化 + 价格信号 + 配额”调度，避免饿死；
- **抗 DoS**：未入选的触发不占用执行队列，缓压在索引层；
- **可扩展**：面对海量 Actor 仍可在 EOB 毫秒级完成甄选与入箱；
- **计费可审计**：设置/取消/执行的资源占用清晰计量与结算。

---

## 3. 术语（Definitions）

- **Timer**：对某个 `target`（Actor）登记的触发器。包含 `timer_id`、`due_height` 或 `watch_keys`、`payload`、`gas_limit`、`bid`（可选）、`owner` 等元数据。
- **Fire**：某个 Timer 在高度 `h` 被判断为“应投递”的一次触发实例（具备 `fire_seq`）。
- **Mailbox**：每个 Actor 的系统消息队列。只有**被选中的 fire**才会被**物化**为 Mailbox 条目。
- **Deliver**：在 EOB 阶段，从索引中筛选候选 Timer、排序、配额剪枝，并把入选项物化为 Mailbox 条目的过程。
- **Deterministic Ordering（全序）**：一个无随机、无本地时钟、可重演的排序比较器。
- **Dual-Meter**：Cycles（执行期资源）与 Cells（状态字节）独立计量与动态基费（见 CIP-3）。

---

## 4. 适用范围（In Scope）与非目标（Out of Scope）

**In Scope**
1. Timer 的链上状态模型与索引结构；
2. 触发判定（高度 / 状态变更）、候选集构造；
3. 排序与公平调度（老化、出价、FIFO、配额）；
4. 物化入箱与幂等；
5. 计费锚点（Cycles/Cells）与安全约束；
6. 治理参数与实现参考（性能、分片）。

**Out of Scope**
- EOB 的全流程顺序与费用调整细节（见 **CIP-6: EOB Lifecycle**）。
- Off-chain Runner 的证明与争议流程（见 **CIP-2**）。
- 存储引擎的物理布局与提交流程（见 **CIP-4**）。

---

## 5. 设计概览（Design Overview）

- **交易阶段（Tx）**：只进行 `set_timeout(due_height)`、`set_interval(every_blocks)`、`set_state_watch(keys)`、`cancel_timer(id)` 等**登记/更新/取消**；不执行回调。
- **EOB 阶段（Deliver → Materialize → Execute）**：
  1) **Deliver**：根据 `due_height ≤ h` 与 `watch_keys ∩ ModifiedKeySet(h) ≠ ∅` 生成候选；按全序排序与配额剪枝；
  2) **Materialize**：把入选项物化为 Mailbox 条目，生成 `deliver_id`，快照 payload/gas/bid/trigger_ctx；
  3) **Execute（见 CIP-6）**：在本块内按固定顺序执行 Mailbox 条目，产生回执与写集；周期计时器写回下一次 `due_height`。

---

## 6. 状态模型与索引（State & Indices）

### 6.1 Timer 元数据（逻辑结构）
```
TimerMeta {
  owner: Address,
  target: ActorID,
  timer_id: u256,
  kind: ENUM { HEIGHT, STATE_WATCH, INTERVAL }
  due_height: u64?          // HEIGHT / INTERVAL 派生字段
  every_blocks: u32?        // INTERVAL 专用
  watch_keys: [KeyPrefix]?  // STATE_WATCH 专用
  payload: bytes,
  gas_limit: u64,
  bid_fp: u128 (Q32.96),    // 可选出价，定点数
  fifo_rank: u64,           // 创建序/刷新序，用于稳定排序
  fire_seq: u32,            // 本 timer 已触发次数计数
  version: u8
}
```

### 6.2 全局索引
- **IndexHeight**：`BTree<(due_height, target, timer_id)> → TimerMetaRef`
- **IndexWatch**：`Map<KeyPrefix, Set<(target, timer_id)>>`

> 注：本提案移除时间触发，因此不再定义 `IndexTime`。

---

## 7. API 规范（外部接口）

### 7.1 设置与取消
- `set_timeout(target, due_height, payload, gas_limit, bid_fp?) -> timer_id`
- `set_interval(target, every_blocks, payload, gas_limit, bid_fp?) -> timer_id`
- `set_state_watch(target, watch_keys[], payload, gas_limit, bid_fp?) -> timer_id`
- `cancel_timer(target, timer_id) -> bool`

**错误码（示例）**
- `ERR_INVALID_PARAM`、`ERR_TARGET_NOT_FOUND`、`ERR_TIMER_NOT_FOUND`、`ERR_QUOTA_EXCEEDED`、`ERR_UNSUPPORTED_TIMER_TYPE`（如传入时间参数）。

### 7.2 行为约束
- **幂等设置**：同 `(owner, target, client_nonce)` 可选映射到稳定 `timer_id`；
- **更新语义**：更新 `payload/gas_limit/bid` 会提升 `fifo_rank`（用于公平 FIFO）；
- **取消语义**：在 EOB 入箱前取消，则当块不再入选；已物化入箱的条目不受影响（见执行阶段）。

---

## 8. Deliver：触发判定与候选集构造

### 8.1 判定规则
- **高度触发**：`due_height ≤ h`。
- **状态触发**：`watch_keys ∩ ModifiedKeySet(h) ≠ ∅`，其中 `ModifiedKeySet(h)` 为本块交易执行日志中的修改键集合；必须来源于链上确定可见的执行记录。

### 8.2 候选集合并与去重
- 从 `IndexHeight.le(h)` 与 `IndexWatch[ModifiedKeySet(h)]` 取回集合，按 `(target,timer_id)` 去重；
- 对于 `INTERVAL` 类型，`due_height` 为按 `every_blocks` 推导出的下一次高度；执行成功或按规则后滚动递增。

---

## 9. 排序、公平与限流（Scheduling & Fairness）

### 9.1 评分函数（定点整数）
```
age_blocks = max(0, h - due_height)
score = W_age * age_blocks + W_bid * bid_fp + W_fifo * fifo_rank
```
- `W_*` 为治理参数；
- 禁止使用浮点数与本地时间源；
- 所有参与字段来自同一读快照（EOB 开始时）。

### 9.2 全序比较器（Total Order）
从高到低：
1. `score`（大在前）；
2. `priority_bucket`（可选分层，派生自 bid 或管理员策略）；
3. `per_target_round_robin_cursor`（目标间公平轮转，纯函数或链上状态推导）；
4. `due_height`（更早者优先）；
5. `target_id`（字典序）；
6. `timer_id`（字典序/创建序）。

### 9.3 配额限流（Quota）
- 全局：`MAX_FIRES_PER_BLOCK`；
- 每目标：`MAX_FIRES_PER_TARGET`；
- 算法：先全序排序，再以轮转方式按配额剪枝；未入选者保留在索引中，`age_blocks` 下块自然增大。

---

## 10. 物化入箱（Materialize to Mailbox）

### 10.1 Mailbox 条目结构
```
MailboxEntry {
  deliver_id = H(h, parent_state_root, timer_id, fire_seq),
  target: ActorID,
  payload_snapshot: bytes,
  gas_limit_snapshot: u64,
  bid_snapshot: u128,
  trigger_ctx: {
    kind: HEIGHT | STATE_WATCH | INTERVAL,
    due_height: u64?,
    matched_keys: [Key]?
  }
}
```

### 10.2 幂等与去重
- `deliver_id` 唯一；重复物化会被识别并丢弃；
- `fire_seq` 每次物化后 +1；
- 物化操作只读原有元数据，禁止在此阶段修改合约状态。

---

## 11. 执行语义（Execution Semantics, 摘要）

> 完整执行流水线见 **CIP-6**。本节仅给与 Timer 直接相关的约束。

- **同块执行**：被物化的 Mailbox 条目在同一 EOB 内按固定顺序执行；
- **失败学**：gas 不足/执行错误会生成回执，不影响幂等性；
- **周期计时器**：执行后写回下一次 `due_height = h + every_blocks`（或在失败场景按策略决定是否滚动）；
- **当前块创建的 Timer**：不得在**同块**被触发执行（避免“前后文不确定”）。

---

## 12. 计费与度量（Metering & Fees）

- **设置/取消**：收取 **Cycles**（执行期 CPU/IO）与必要的 **Cells**（索引元数据）；
- **物化/执行**：物化为只读；执行计入 Cycles（代码运行）与 Cells（新增状态）；
- **结算**：在 EOB 聚合本块的双表用量；按 **CIP-3** 动态调整 basefee，燃烧基费，小费给出块者。

---

## 13. 治理参数（Governance-Tunable）

- `MAX_FIRES_PER_BLOCK`（全局投递上限）；
- `MAX_FIRES_PER_TARGET`（每 Actor 投递上限）；
- `W_age, W_bid, W_fifo`（评分权重）；
- `MIN_GAS_LIMIT_PER_TIMER`、`MAX_PAYLOAD_SIZE`（健壮性）；
- `MAX_TIMERS_PER_OWNER / PER_TARGET`（滥用防护）。

> 建议这些参数在治理层可动态更新，更新在下一个治理 epoch 生效。

---

## 14. 确定性与重演性（Determinism）

- `DeliverTimers(h)` 仅依赖 `{ h, parent_state_root, IndexHeight, IndexWatch, ModifiedKeySet(h) }`；
- 禁止使用本地随机、VRF、墙钟时间；
- 全序比较器与配额剪枝算法不含实现相关分支；
- `deliver_id = H(h, parent_state_root, timer_id, fire_seq)` 确保幂等；
- reorg 时可基于新父状态**完全重演** Deliver 结果，旧入箱结果自然作废。

---

## 15. 安全与 DoS 抵抗（Security Considerations）

- **配额与背压**：未入选的 fire 不入 Mailbox，缓压在索引层，避免执行队列膨胀；
- **上限与收费**：设置/取消计费 + “每 Owner/Target 上限”抑制定时风暴；
- **出价投机**：`bid_fp` 仅影响排序，不保证当块执行；可设置最小/最大 bid 约束；
- **状态触发滥用**：对 `watch_keys` 数量与匹配规则设上限；对单块 `ModifiedKeySet` 的 fan-out 施行裁剪；
- **重入与幂等**：回调侧建议以 `(deliver_id)` 做幂等保护；
- **Gas 自杀**：为回调设置 `MIN_GAS_LIMIT_PER_TIMER`，避免大量“必失败”的垃圾触发。

---

## 16. 性能与实现建议（Implementation Notes）

- **索引结构**：
  - `IndexHeight`: B-Tree/SkipList；
  - `IndexWatch`: 哈希 → RoaringBitmap/小集合；
- **EOB 候选规模**：复杂度主要与候选数 `M` 成正比，而非 Actor 总数 `N`；
- **排序优化**：对 `score`/`age` 分档桶排可将 `O(M log M)` 近似到 `O(M)`；
- **分片并行**：按 `hash(target_id) mod S` 切片并行 Deliver，各片 K 选后做 `K-way merge`；
- **批量写回**：周期型下一次 `due_height` 在 EOB 末批量更新索引，提升写入吞吐；
- **快照只读**：EOB 开始建立只读视图，避免并发写冲突与读歧义。

---

## 17. 兼容性与迁移（Removal of Time Trigger）

- **弃用策略**：完全移除 `due_time`/`every_secs` 参数；
- **客户端替代**：SDK 提供 `after_minutes(m)`/`every_minutes(k)`，以目标出块时间 `τ` 换算为 `due_height`/`every_blocks`；
- **链下替代**：严格墙钟需求交由 **Runner** 以 `not_before_height/not_after_height` 窗口回链；
- **存量迁移**：网络升级时将现存时间型条目按 `τ` 一次性换算到 `due_height`，并标记 `version += 1`。

---

## 18. 参考伪代码（Reference Pseudocode）

```python
def collect_candidates(h, modified_keys):
    C1 = IndexHeight.range_le(h)               # O(log T + M1)
    C2 = union(IndexWatch[k] for k in modified_keys)  # bitmap union
    return dedup(C1 ∪ C2)                      # by (target, timer_id)

W_age, W_bid, W_fifo = params()

def score_of(t, h):
    age_blocks = max(0, h - t.due_height)
    return W_age*age_blocks + W_bid*t.bid_fp + W_fifo*t.fifo_rank

def deliver_timers(h, parent_state_root, modified_keys):
    C = collect_candidates(h, modified_keys)
    C_sorted = sorted(C, key=lambda t: (
        -score_of(t, h),
        priority_bucket(t),
        rr_cursor(t.target, h),
        t.due_height,
        t.target, t.timer_id
    ))

    fires, per_target = [], defaultdict(int)
    for t in C_sorted:
        if len(fires) >= MAX_FIRES_PER_BLOCK: break
        if per_target[t.target] >= MAX_FIRES_PER_TARGET: continue
        fires.append(t); per_target[t.target] += 1

    mailbox_entries = []
    for t in fires:
        deliver_id = H(h, parent_state_root, t.timer_id, t.fire_seq)
        mailbox_entries.append({
            'deliver_id': deliver_id,
            'target': t.target,
            'payload_snapshot': t.payload,
            'gas_limit_snapshot': t.gas_limit,
            'bid_snapshot': t.bid_fp,
            'trigger_ctx': {'kind': t.kind, 'due_height': t.due_height}
        })
        t.fire_seq += 1
    return mailbox_entries
```

---

## 19. 测试向量（Test Vectors, 摘要）

1. **基本高度触发**：`due_height = h`；在 `h` 入箱并执行；`h-1` 不执行。
2. **状态触发**：`watch_keys=[k1]`；当 `k1` 在 `h` 被修改则入箱；未修改则不入箱。
3. **配额剪枝**：`MAX_FIRES_PER_BLOCK = 2`；三条候选按评分排第 3 条不入箱，下一块 `age` 增大后应优先入箱。
4. **每目标限流**：`MAX_FIRES_PER_TARGET=1`；同一 `target` 三条候选，仅一条入箱；
5. **幂等入箱**：重演 `deliver_timers(h)` 结果不重复生成条目（`deliver_id` 不变）。
6. **周期滚动**：`every_blocks=5`；在 `h` 执行后，应将 `due_height := h+5` 写回索引。

---

## 20. 生态影响与部署

- **实现者**：需提供上述索引、Deliver 算法与幂等入箱；
- **合约/应用**：以高度/状态触发重写“时间触发”场景；严格墙钟由 Runner 提供；
- **SDK**：添加“分钟到块”的换算 API 与 CRON 到 Runner 的适配层；
- **治理**：上线前配置合理的配额与权重；监控 BLOCKMETA/RECEIPTS 度量以校准参数。

---

## 21. 参考与关联文档（References）

- Cowboy Whitepaper (v2025-11-06)
- **CIP-2**: Verifiable Asynchronous Off-chain Computation Framework
- **CIP-3**: Dual-Metered Gas Calculation and Dynamic Fee Market
- **CIP-4**: Storage and State Persistence
- **CIP-6 (拟)**: End-of-Block Lifecycle & Post-Execution Hooks

---

> 本文档旨在成为**独立完整**的 Timer 规范：合约如何设置/取消，协议如何在 EOB 判定/排序/入箱，如何计费与限流，以及如何保证强确定性与可扩展性。EOB 的完整顺序与费用结算细节请参见配套的 **CIP-6**。

