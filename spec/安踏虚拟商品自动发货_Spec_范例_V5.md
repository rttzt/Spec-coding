# [SPEC-ANTA-001] 虚拟商品自动拆单与无物流发货

---

## 📋 3.0 元数据 (Metadata)

| 字段 | 值 |
|:---|:---|
| **Spec ID** | SPEC-ANTA-001 |
| **版本** | 1.0.0 |
| **状态** | InReview → Gate 1 待审 |
| **作者** | 卫常 |
| **关联需求** | TAPD-12345678 |
| **关联系统** | `oms-order-service`, `platform-integration`, `order-event-consumer` |

---

## 📖 3.1 背景与目标 (Context & Goals)

> 👤 **填写人**: 产品经理 (PM) — 卫常
> 🔍 **审核人**: TL — 张三（确认技术可行性）

### 业务背景

目前天猫/京东店铺的虚拟商品（邮差补款、赠品、运费补差）需要人工识别、手动拆单、手动操作无物流发货。2025 年 12 月仅童装旗舰店就有 4.8 万单虚拟商品需要人工处理，每单耗时约 2 分钟，全年累计约 1600 小时人工投入。大促期间积压严重，影响发货时效考核。

### 核心价值

实现虚拟商品的自动化识别、自动拆单及自动无物流发货，消除人工干预，提升发货时效。

### 量化指标

- 虚拟订单自动处理率 > 95%
- 拆单 + 发货全流程耗时 < 5 秒
- 预计全年节省约 60 人天

```
🔍 审核项：
☐ 业务场景是否真实存在？（PM 自审）                     ✅ 已确认
☐ 量化指标是否可衡量？（TL 确认）                        ⬜ 待 TL 确认
☐ 技术可行性是否已验证？（TL 确认）                      ⬜ 待 TL 确认
```

---

## 🚧 3.2 存量事实基准 (Fact Base)

> 👤 **填写人**: 技术负责人 (TL) — 张三
> 🔍 **审核人**: TL（自审）+ 相关模块 Owner

### 禁止变更清单（红线区）

| 编号 | 模块/文件/表 | 禁止原因 | 影响范围 |
|:---|:---|:---|:---|
| RED-01 | `com.anta.finance.FinanceCalculator` | 财务结算逻辑，涉及对账和审计 | 一旦改动影响所有店铺的月度结算 |
| RED-02 | `com.anta.finance.OrderAmountService` | 订单金额计算，涉及优惠分摊 | 改动会导致金额不一致 |
| RED-03 | `oms_order` 表 | 核心订单主表，历史数据不可回滚 | 改动影响所有订单查询和统计 |
| RED-04 | `financial_reconciliation` 表 | 财务对账表 | 改动导致对账数据不一致 |
| RED-05 | `com.anta.oms.dao` 包下所有 Mapper | 禁止直接操作数据库 | 所有数据操作必须通过 Service 层 |

### 存量接口清单

| 编号 | 接口名称（逻辑） | 负责模块 | 备注 |
|:---|:---|:---|:---|
| EX-01 | 订单查询 | `oms-order-service` | `OrderQueryService.queryById(String)` — 只读，不可改 |
| EX-02 | 订单状态变更 | `oms-order-service` | `OrderStatusService.updateStatus(String, StatusEnum)` — 只读其契约 |

### 消息队列依赖

| 编号 | Topic/Event | 负责模块 | 说明 |
|:---|:---|:---|:---|
| MQ-01 | `order.outbound.completed` | RocketMQ | 订单出库完成事件，是自动发货的触发源 |

```
🔍 审核项：
☐ 红线是否完整？是否遗漏了未文档化的历史逻辑？          ⬜ 待 TL 最终确认
☐ 是否有间接影响（如改动 OrderStatusService 影响财务模块）？ ⬜ 待 TL 最终确认
☐ 标注了"不可触碰"的类/方法，AI 是否真的不会改动？       ⬜
```

---

## 🔀 3.3 增量变更描述 (Delta Changes)

> 👤 **填写人**: PM（卫常）+ TL（张三）
> 🔍 **审核人**: PM（确认业务边界）+ TL（确认技术边界）

### 新增功能

| 编号 | 功能模块 | 功能描述 | 触发条件 |
|:---|:---|:---|:---|
| NEW-01 | 虚拟商品识别引擎 (`VirtualItemIdentifier`) | 根据商品名称关键字或商品 ID 白名单识别虚拟商品 | 订单同步到中台时 |
| NEW-02 | 自动拆单服务 (`AutoSplitService`) | 将混合订单拆分为实物子订单和虚拟子订单 | 当订单同时包含实物和虚拟商品时 |
| NEW-03 | 自动发货监听器 (`AutoShipListener`) | 监听出库完成事件，对虚拟子订单执行无物流发货 | 收到 `order.outbound.completed` 事件且订单含虚拟商品时 |

### 修改逻辑

| 编号 | 模块/文件 | 修改内容 | 修改原因 |
|:---|:---|:---|:---|
| MOD-01 | 订单同步流程 | 增加"是否含虚拟商品"标记位 | 下游需要区分混合订单 |
| MOD-02 | 订单出库回调 | 增加虚拟商品过滤逻辑，避免对虚拟商品重复操作 | 虚拟商品无需实际出库 |

### 极端场景处理

| 场景 | 处理策略 |
|:---|:---|
| 拆单接口调用失败 | 重试 3 次，间隔 1s/3s/5s；3 次均失败则记录错误日志，转人工处理，原订单状态不变 |
| 电商平台发货接口超时 | 重试 3 次，间隔 2s/5s/10s；仍失败则记录 FAIL 状态并触发告警 |
| 消息重复消费 | 通过订单号 + 操作类型做幂等，已处理过的直接跳过 |
| 拆出的虚拟子订单金额异常 | 暂停自动发货，记录异常日志，转人工审核 |

```
🔍 审核项：
☐ 极端场景下的处理逻辑是否已明确？                      ✅ 已填写
☐ 回滚机制是否已定义？（拆单失败后原订单状态不变）        ✅ 已填写
☐ 是否与 Fact Base 中的红线有冲突？                     ⬜ 待 TL 确认
```

---

## 🧭 3.4 隐形边界声明 (Invisible Boundary Declaration)

> 👤 **填写人**: 技术负责人 (TL) — 张三
> 🔍 **审核人**: PM（确认业务容错策略）+ 后端开发 — 李四（确认技术实现）

**本节声明所有"字面正确但隐含语义可能有误"的维度，防止 AI 生成的代码在编译/单测通过后，上生产才暴露问题。**

### 3.4.1 幂等性 (Idempotency)

| 操作 | 幂等键 | 重复调用行为 | 说明 |
|:---|:---|:---|:---|
| 自动发货 | `order_id + platform` | 跳过已处理 | 通过 `auto_ship_record` 表唯一键 `uk_order_platform` 防重 |
| 消费出库事件 | `order_id + 事件类型` | 幂等跳过 | MQ 可能重复投递，已处理的订单直接 ACK |
| 创建发货流水 | `order_id + platform` | 返回已有记录 | INSERT 冲突时返回已有记录的 ID |

### 3.4.2 并发安全 (Concurrency Safety)

| 共享资源 | 并发风险 | 锁策略 | 锁粒度 |
|:---|:---|:---|:---|
| `auto_ship_record` 表写入 | 同一订单重复插入发货记录 | 唯一索引 `uk_order_platform` | 单条记录 |
| 订单状态更新 (`oms_order`) | 多个发货流程同时更新同一订单状态 | 乐观锁（`WHERE status = #{expectedStatus}`） | 单条记录 |
| `virtual_item_rule` 规则读取 | 无（只读操作） | 无锁，使用缓存 | — |

### 3.4.3 分布式事务 (Distributed Transaction)

| 跨系统操作 | 事务边界 | 最终一致性方案 | 补偿机制 |
|:---|:---|:---|:---|
| 天猫发货 + `auto_ship_record` 状态更新 | 本地事务 + 外部 API | 先写流水（PENDING）→ 调天猫 API → 成功则更新为 SUCCESS，失败则更新为 FAIL | 定时任务查询天猫发货状态，同步更新本地记录 |
| 京东发货 + `auto_ship_record` 状态更新 | 本地事务 + 外部 API | 同上（京东 API 调用方式不同，但方案一致） | 同上 |
| 发货成功 + 钉钉告警通知 | 本地事务提交后异步发送 | 发货成功（本地事务已提交）后发送钉钉消息，允许通知丢失 | 监控 `FAIL` 状态记录的钉钉告警不受此限制（由定时任务触发） |

### 3.4.4 错误分类 (Error Classification)

| 错误类型 | 示例错误码 | 处理策略 | 最大重试次数 | 重试间隔 |
|:---|:---|:---|:---|:---|
| 可重试错误 | `NETWORK_TIMEOUT`, `API_TIMEOUT`, `SYSTEM_BUSY` | 指数退避重试 | 3 | 2s / 5s / 10s |
| 参数错误 | `INVALID_PARAM`, `PRODUCT_NOT_FOUND` | 直接失败，钉钉告警通知运营 | 0 | — |
| 幂等成功 | `DUPLICATE_REQUEST` | 当作成功处理，记录日志 | 0 | — |
| 限流错误 | `RATE_LIMITED` | 等待后重试 | 3 | 30s / 60s / 120s |

### 3.4.5 时间语义 (Time Semantics)

| 时间字段 | 存储类型 | 时区策略 | 比较方式 |
|:---|:---|:---|:---|
| 发货时间 (`auto_ship_record.created_at`) | `DATETIME`（由数据库 `CURRENT_TIMESTAMP` 填充） | 数据库统一存储 Asia/Shanghai 时间，展示层直接使用 | 与数据库时间比较，不做转换 |
| 超时判断（发货后 24h 未回调确认） | 相对时间 | 使用 `Instant.now()` 与 `created_at` 转 UTC 时间戳比较 | 差值比较（> 24h），不依赖时区 |
| 重试间隔 | 相对时间 | `Thread.sleep()` 或定时任务调度 | 不影响数据存储 |

### 3.4.6 数据安全 (Data Security)

| 敏感字段 | 脱敏策略 | 日志输出规则 | 存储加密 |
|:---|:---|:---|:---|
| 虚拟卡券密码 | 完全不可输出 | 🚫 严禁出现在日志中（包括 DEBUG 级别） | ✅ AES-256 加密存储 |
| 用户手机号 | 保留前 3 位 + 后 4 位，中间 `****` 替换 | ✅ 脱敏后可输出（如 `138****5678`） | — |
| 订单金额 | 保留完整值（含 2 位小数） | ✅ 可输出（非敏感信息） | — |
| 天猫 API Secret | 完全不可输出 | 🚫 严禁出现在日志中 | ✅ 通过 VaultAgent 运行时获取 |

```
🔍 审核项：
☐ 发货操作的幂等性是否已通过 uk_order_platform 保证？     ⬜ 待 TL 确认
☐ 订单状态更新的乐观锁是否与现有 OrderStatusService 行为一致？ ⬜ 待后端确认
☐ 分布式事务方案中的定时补偿任务是否已存在？需新建？       ⬜ 待后端确认
☐ 可重试错误的间隔（2s/5s/10s）是否与平台限流策略匹配？    ⬜ 待后端确认
☐ 时区策略是否与现有系统约定一致？（数据库统一使用 Asia/Shanghai） ⬜ 待 DBA 确认
☐ 日志脱敏是否已在 Logback 框架层配置？（不能仅靠开发者自觉） ⬜ 待 DevOps 确认
```

---

## 🔗 3.5 技术契约 — 代码锚定 (CodeAnchor)

> 👤 **填写人**: 后端开发 — 李四
> 🔍 **审核人**: TL — 张三（确认锚定准确性）

| 编号 | 逻辑名称 | 真实类全路径 | 真实方法签名 | 返回值类型 | 所在模块 | 确认状态 | 确认人 |
|:---|:---|:---|:---|:---|:---|:---|:---|
| ANC-01 | 订单查询 | `com.anta.oms.service.OrderQueryService` | `queryById(String orderId)` | `OrderVO` | `oms-order-service` | ⬜ 待确认 | |
| ANC-02 | 订单状态更新 | `com.anta.oms.service.OrderStatusService` | `updateStatus(String orderId, StatusEnum status)` | `void` | `oms-order-service` | ⬜ 待确认 | |
| ANC-03 | 订单拆分 | `com.anta.oms.service.OrderSplitExecutor` | `executeSplit(String orderId, List<Long> skuIds)` | `SplitResultVO` | `oms-order-service` | ⬜ 待确认 | |
| ANC-04 | 天猫无物流发货 | `com.anta.platform.adapter.TmallAdapter` | `virtualShip(VirtualShipRequest req)` | `VirtualShipResponse` | `platform-integration` | ⬜ 待确认 | |
| ANC-05 | 京东无物流发货 | `com.anta.platform.adapter.JdAdapter` | `virtualShip(JdVirtualShipReq req)` | `JdVirtualShipResp` | `platform-integration` | ⬜ 待确认 | |
| ANC-06 | 出库事件消费 | `com.anta.consumer.OutboundEventListener` | `onMessage(MessageExt msg)` | `void` | `order-event-consumer` | ⬜ 待确认 | |

### ANC-03 详情（示例：订单拆分接口）

```java
// 文件路径: oms-order-service/src/main/java/com/anta/oms/service/OrderSplitExecutor.java
// 类定义:
@Service
public class OrderSplitExecutor {
    
    /**
     * 执行订单拆分
     * @param orderId 原订单号
     * @param skuIds  需要拆出到子订单的 SKU ID 列表
     * @return 拆分结果，包含原订单号和新子订单号
     * @throws OrderSplitException 当订单状态不允许拆分或拆单接口异常时
     */
    public SplitResultVO executeSplit(String orderId, List<Long> skuIds) 
            throws OrderSplitException {
        // 已有实现...
    }
}

// 返回值类型: SplitResultVO
// 字段: success(boolean), realOrderId(String), virtualOrderId(String)
```

```
🔍 审核项：
☐ 该接口是否在代码库中真实存在？（搜索验证）            ⬜ 待 TL 确认
☐ 方法签名是否与代码完全一致？（不可有任何偏差）         ⬜ 待 TL 确认
☐ 参数类型和返回值类型是否已确认？                      ⬜ 待 TL 确认
☐ 该接口的调用是否需要特殊的前置条件？（事务、鉴权）     ⬜ 待 TL 确认

⚠️ 以上所有 CodeAnchor 必须在 Gate 1 审核前全部确认。
   未确认的 CodeAnchor 意味着 AI 将使用臆造的接口名，生成的代码必定编译失败。
```

### AI 实现时必须注入的上下文（每个 CodeAnchor）

> ⚠️ 以下信息必须在 AI 实现阶段逐 Step 注入上下文窗口，缺少任何一项都会导致代码映射幻觉。

| 编号 | 注入项 | 内容 | 防幻觉作用 |
|:---|:---|:---|:---|
| ANC-01 | 方法签名 + throws | `queryById(String orderId)` — 无 throws | 防止参数类型偏差 |
| ANC-01 | 返回值类型 | `OrderVO` (字段: orderId, status, skuList, totalAmount) | 防止返回值包装层误判 |
| ANC-01 | DI 注解 | `@Service` | 防止用 `@Component` 注入失败 |
| ANC-02 | 方法签名 + throws | `updateStatus(String orderId, StatusEnum status)` — throws `OrderStatusException` | 防止遗漏 checked exception |
| ANC-02 | 枚举定义 | `StatusEnum: CREATED, PAID, SHIPPED, CANCELLED...` | 防止枚举值臆造 |
| ANC-03 | 方法签名 + throws | `executeSplit(String, List<Long>)` — throws `OrderSplitException` | 防止类型偏差 (Integer vs Long) |
| ANC-03 | 返回值类型 | `SplitResultVO` (success, realOrderId, virtualOrderId) | 防止字段名臆造 |
| ANC-04 | 方法签名 + throws | `virtualShip(VirtualShipRequest req)` — throws `PlatformException` | 防止请求体结构错误 |
| ANC-04 | 请求体结构 | `VirtualShipRequest { tid, outSid, logisticsNo, subTid }` | 防止字段名/类型臆造 |
| ANC-05 | 方法签名 + throws | `virtualShip(JdVirtualShipReq req)` — throws `PlatformException` | 与 ANC-04 结构不同，需单独注入 |
| ANC-06 | 方法签名 | `onMessage(MessageExt msg)` — 无返回值 | 防止误判返回类型 |

> 📦 **注入方式**：在 AI 实现的每个 Step 开始前，将该 Step 涉及的 CodeAnchor 对应的源文件片段复制粘贴到 AI 对话窗口中。

---

## 📡 3.6 技术契约 — 第三方 API

> 👤 **填写人**: 后端开发 — 李四
> 🔍 **审核人**: TL — 张三（确认与官方文档一致）

| 编号 | 平台 | API 名称 | 官方文档链接 | 关键字段 | 认证方式 | 确认人 |
|:---|:---|:---|:---|:---|:---|:---|
| API-01 | 天猫 | 无物流发货 | [内部 Wiki 链接] | `tid`, `out_sid`, `sub_tid` | OAuth 2.0 (店铺授权) | ⬜ |
| API-02 | 京东 | 无物流发货 | [内部 Wiki 链接] | `orderId`, `noLogisticsType` | Access Token | ⬜ |

### API-01 详情（天猫无物流发货）

```json
// Request（必须与天猫官方文档一致）:
{
  "tid": "1923456789123456",        // 订单号（天猫格式）
  "out_sid": "VIRTUAL_SHIP_001",    // 外部发货流水号
  "logistics_no": "VIRTUAL",        // 物流单号（虚拟发货固定值）
  "sub_tid": "1923456789123456_1"   // 子订单号（如果有拆单）
}

// Response:
{
  "trade_sync_ship_success": true,  // 发货是否成功
  "error_code": null,               // 错误码（成功时为 null）
  "error_msg": null                 // 错误信息
}
```

```
🔍 审核项：
☐ 字段名是否与官方最新 API 文档完全一致？               ⬜ 待后端确认
☐ 枚举值是否已逐项核对？（如 noLogisticsType 的可选值）  ⬜ 待后端确认
☐ 认证方式是否正确？OAuth scope 是否足够？              ⬜ 待后端确认
☐ 异常码含义是否已确认？                                ⬜ 待后端确认
```

---

## 🗄️ 3.7 技术契约 — 数据模型

> 👤 **填写人**: 后端开发 — 李四 + DBA
> 🔍 **审核人**: DBA（确认与数据库对齐）

### 新增表 1：虚拟商品识别规则配置

```sql
CREATE TABLE virtual_item_rule (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  match_type VARCHAR(16) NOT NULL COMMENT '匹配类型: KEYWORD / ITEM_ID',
  match_value VARCHAR(256) NOT NULL COMMENT '匹配值: 关键字或商品ID',
  platform VARCHAR(8) NOT NULL COMMENT '平台: TMALL / JD / ALL',
  is_enabled TINYINT DEFAULT 1 COMMENT '是否启用: 1-是, 0-否',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_platform_enabled (platform, is_enabled)
) COMMENT '虚拟商品识别规则配置表';
```

### 新增表 2：自动发货流水记录

```sql
CREATE TABLE auto_ship_record (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id VARCHAR(32) NOT NULL COMMENT '订单号',
  sub_order_id VARCHAR(32) COMMENT '子订单号（拆单后）',
  platform VARCHAR(8) NOT NULL COMMENT '平台: TMALL / JD',
  ship_status VARCHAR(16) NOT NULL COMMENT '发货状态: PENDING / SUCCESS / FAIL / RETRYING',
  retry_count INT DEFAULT 0 COMMENT '已重试次数',
  error_msg TEXT COMMENT '失败原因',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_order_platform (order_id, platform)
) COMMENT '自动发货流水记录表';
```

### 修改表：订单主表增加虚拟标记

| 表名 | 变更类型 | 字段名 | 字段类型 | 说明 |
|:---|:---|:---|:---|:---|
| `oms_order` | 新增字段 | `has_virtual_item` | `TINYINT DEFAULT 0` | 是否包含虚拟商品 |
| `oms_order` | 新增字段 | `virtual_status` | `VARCHAR(16)` | 虚拟商品状态: NONE/PENDING/SHIPPED/FAILED |

```
🔍 审核项：
☐ 字段类型和长度是否与现有表风格一致？                  ⬜ 待 DBA 确认
☐ 索引设计是否合理？是否影响插入性能？                   ⬜ 待 DBA 确认
☐ 修改 oms_order 表是否与 RED-03 冲突？                  ⬜ 待 TL 确认
   ⚠️ RED-03 声明 oms_order 不可修改，此处增加字段是增量修改而非破坏性变更，
   需 TL 确认是否属于允许范围。
```

---

## 🧪 3.8 验收标准与 TDD 规划

> 👤 **填写人**: 测试工程师 (QA) — 王五 + 后端开发 — 李四
> 🔍 **审核人**: QA — 王五（确认覆盖完整性）

### 场景 1：0 元未转入中台的虚拟商品自动发货

| 步骤 | 内容 |
|:---|:---|
| Given | 订单 O001 包含 1 个实物商品（SKU-001）和 1 个 0 元虚拟商品（SKU-V01，匹配关键字"邮差"），订单已同步到中台，实物商品已出库 |
| When | `order.outbound.completed` 事件触发 |
| Then | 系统识别出虚拟商品 SKU-V01，调用天猫无物流发货 API，`auto_ship_record` 表记录状态为 `SUCCESS`，实物订单不受影响 |

### 场景 2：混合订单自动拆单 + 发货

| 步骤 | 内容 |
|:---|:---|
| Given | 订单 O002 包含 1 个实物商品（SKU-002，¥299）和 1 个付费虚拟商品（SKU-V02，¥15），订单已同步到中台 |
| When | 订单进入待发货状态，系统检测到混合订单 |
| Then | 1. 系统调用 `OrderSplitExecutor.executeSplit()` 拆分出虚拟子订单 V-O002；2. 实物主订单 O002 正常进入发货流程；3. 虚拟子订单 V-O002 自动调用无物流发货 API；4. `auto_ship_record` 记录两笔流水 |

### 场景 3：拆单失败 — 转人工处理

| 步骤 | 内容 |
|:---|:---|
| Given | 订单 O003 包含混合商品，但拆单接口 `OrderSplitExecutor.executeSplit()` 返回异常 |
| When | 系统重试 3 次后仍然失败 |
| Then | 1. 系统记录 `auto_ship_record` 状态为 `FAIL`，error_msg 记录失败原因；2. 触发钉钉告警通知运营人员；3. 原订单 O003 状态保持不变，等待人工介入 |

### 场景 4：发货 API 超时 — 重试成功

| 步骤 | 内容 |
|:---|:---|
| Given | 虚拟子订单 V-O004 需要发货，天猫 API 首次调用超时 |
| When | 系统自动重试 |
| Then | 第 2 次重试成功，`auto_ship_record` 状态更新为 `SUCCESS`，retry_count 为 1 |

### 场景 5：幂等性验证

| 步骤 | 内容 |
|:---|:---|
| Given | 虚拟子订单 V-O005 已经自动发货成功 |
| When | 相同事件被重复消费（消息重复投递） |
| Then | 系统通过 `order_id + platform` 唯一键检查，发现已处理，直接跳过，不重复发货 |

---

### TDD 执行计划

| 步骤 | 测试类 | 测试方法 | 验证内容 | Mock 依赖（基于 CodeAnchor） | 状态 |
|:---|:---|:---|:---|:---|:---|
| RED-1 | `VirtualItemIdentifierTest` | `testKeywordMatch` | 关键字"邮差"能正确匹配虚拟商品 | Mock `ANC-01` (OrderQueryService) | ⬜ |
| RED-2 | `VirtualItemIdentifierTest` | `testItemIdMatch` | 商品 ID 白名单能正确匹配 | Mock `ANC-01` | ⬜ |
| RED-3 | `VirtualItemIdentifierTest` | `testPhysicalItemNotIdentified` | 实物商品不会被误识别 | Mock `ANC-01` | ⬜ |
| GREEN-1~3 | — | — | 实现 `VirtualItemIdentifier` 使测试全部通过 | — | ⬜ |
| RED-4 | `AutoSplitServiceTest` | `testMixedOrderSplit` | 混合订单拆分为实物+虚拟子订单 | Mock `ANC-03` (OrderSplitExecutor) | ⬜ |
| RED-5 | `AutoSplitServiceTest` | `testPureVirtualOrderNoSplit` | 纯虚拟订单不拆单 | — | ⬜ |
| RED-6 | `AutoSplitServiceTest` | `testSplitFailedRetry` | 拆单失败后重试 3 次转人工 | Mock `ANC-03` (模拟异常) | ⬜ |
| GREEN-4~6 | — | — | 实现 `AutoSplitService` | — | ⬜ |
| RED-7 | `AutoShipListenerTest` | `testVirtualShipOnOutbound` | 出库事件触发虚拟发货 | Mock `ANC-04` (TmallAdapter), Mock `ANC-06` (EventListener) | ⬜ |
| RED-8 | `AutoShipListenerTest` | `testShipApiTimeoutRetry` | 发货 API 超时重试 | Mock `ANC-04` (模拟超时) | ⬜ |
| RED-9 | `AutoShipListenerTest` | `testIdempotentShip` | 重复消息不重复发货 | Mock `ANC-04` | ⬜ |
| GREEN-7~9 | — | — | 实现 `AutoShipListener` | — | ⬜ |
| REFACTOR | 全部测试类 | — | 提取常量、优化逻辑、消除重复 | — | ⬜ |

> ⚠️ TDD 硬性规则：
> 1. Mock 对象必须来自 CodeAnchor 中确认过的真实接口
> 2. 测试必须先于实现代码生成（每个 RED 步骤为一个独立回合）
> 3. 第一次运行测试必须 RED（失败原因必须是"方法未实现"）

```
🔍 审核项：
☐ Given-When-Then 是否覆盖了所有正常分支？（5 个场景）   ⬜ 待 QA 确认
☐ 是否覆盖了所有异常分支？（超时、失败、重复）            ⬜ 待 QA 确认
☐ Mock 对象是否全部来自 CodeAnchor？                     ⬜ 待 QA 确认
☐ TDD 执行步骤是否可独立编译和运行？                      ⬜ 待开发确认
```

---

## ⚙️ 3.9 实施计划 — 增量验证

> 👤 **填写人**: TL — 张三
> 🔍 **审核人**: 后端开发 — 李四（确认步骤可执行）

| Step | 任务描述 | 产出文件 | 本 Step 必须注入的上下文 | 验证方式 | 状态 |
|:---|:---|:---|:---|:---|:---|
| 1 | [环境准备] 创建表 | DDL 脚本 | Runtime Profile (TiDB 6.5, MyBatis 2.3.1) | `mvn flyway:migrate` 通过 | ⬜ |
| 2 | [配置] Apollo 配置项 | 配置文件 | Runtime Profile (Apollo 配置中心引用方式) | 配置中心解析通过 | ⬜ |
| 3 | [TDD RED] VirtualItemIdentifierTest | `VirtualItemIdentifierTest.java` | BaseServiceTest.java, Mockito 4.11 用法, ANC-01 源文件片段 | `mvn test -Dtest=VirtualItemIdentifierTest` RED | ⬜ |
| 4 | [TDD GREEN] VirtualItemIdentifier | `VirtualItemIdentifier.java` | @Service 规范, @Slf4j 用法, ANC-01 源文件片段 | `mvn test -Dtest=VirtualItemIdentifierTest` GREEN | ⬜ |
| 5 | [TDD RED] AutoSplitServiceTest | `AutoSplitServiceTest.java` | BaseServiceTest.java, ANC-03 源文件片段 (含 throws), SplitResultVO | `mvn test -Dtest=AutoSplitServiceTest` RED | ⬜ |
| 6 | [TDD GREEN] AutoSplitService | `AutoSplitService.java` | ANC-03 源文件 + 调用示例, @Resource 注入规范 | `mvn test -Dtest=AutoSplitServiceTest` GREEN | ⬜ |
| 7 | [TDD RED] AutoShipListenerTest | `AutoShipListenerTest.java` | ANC-04, ANC-05, ANC-06 源文件片段, RocketMQ 原生客户端用法 | `mvn test -Dtest=AutoShipListenerTest` RED | ⬜ |
| 8 | [TDD GREEN] AutoShipListener | `AutoShipListener.java` | ANC-04/05/06 源文件 + 调用示例, BizException 定义, 脱敏规则 | `mvn test -Dtest=AutoShipListenerTest` GREEN | ⬜ |
| 9 | [重构] 提取常量、优化 | 全部源文件 | Checkstyle 配置, 代码规范文档 | `mvn test` 全部 GREEN | ⬜ |
| 10 | [集成] 端到端验证 | — | Apollo 配置值, VaultAgent 获取凭证方式 | 灰度环境验证通过 | ⬜ |

> ⚠️ AI 每完成一个 Step，必须输出编译/测试结果，人确认后才能进入下一个 Step。
> 
> ⚠️ **上下文注入规则**：每个 Step 开始前，人必须检查"本 Step 必须注入的上下文"列，将该列中的源文件片段重新注入 AI 对话窗口。不允许因"前面注入过"而跳过——长对话中早期信息可能已被截断。

```
🔍 审核项：
☐ 每个 Step 是否可独立验证？                             ⬜
☐ Step 3-8 严格遵循 RED → GREEN 回合分离原则？           ✅
☐ 验证命令是否在当前项目中可执行？                        ⬜ 待开发确认
```

---

## 🏗️ 3.10 环境与配置

> 👤 **填写人**: DevOps — 赵六
> 🔍 **审核人**: TL — 张三（确认安全合规）

| 配置项 | 真实值/引用 | 说明 |
|:---|:---|:---|
| 数据库连接 | `${DB_OMS_URL}` | Apollo 配置中心引用 |
| 天猫 API Key | `${TMALL_API_KEY}` | 密钥管理服务引用，通过 VaultAgent 获取 |
| 天猫 API Secret | `${TMALL_API_SECRET}` | 密钥管理服务引用，通过 VaultAgent 获取 |
| 京东 Access Token | `${JD_ACCESS_TOKEN}` | 密钥管理服务引用 |
| RocketMQ NameServer | `${ROCKETMQ_NAMESRV}` | Apollo 配置中心引用 |
| RocketMQ Consumer Group | `GID_ANTA_VIRTUAL_SHIP` | Consumer Group 名称 |
| 钉钉告警 Webhook | `${DINGTALK_ALERT_URL}` | Apollo 配置中心引用 |
| 虚拟商品关键字配置 | `邮差\|补款\|运费补差\|赠品` | Apollo 配置项 `virtual.item.keywords` |
| 重试次数 | `3` | Apollo 配置项 `virtual.ship.retry.max` |
| 重试间隔 | `1s, 3s, 5s` | Apollo 配置项 `virtual.ship.retry.intervals` |

```
🔍 审核项：
☐ 所有配置值是否来自真实环境？                           ⬜ 待 DevOps 确认
☐ 密钥类配置是否通过密钥管理服务引用？（严禁明文）         ⬜ 待安全审计
☐ RocketMQ Topic / Consumer Group 是否已在控制台创建？    ⬜ 待 DevOps 确认
☐ Apollo 配置项是否已创建？                              ⬜ 待 DevOps 确认
```

### 3.10.1 运行时画像 (Runtime Profile)

> 👤 **填写人**: TL — 张三 + DevOps — 赵六
> 🔍 **审核人**: 后端开发 — 李四（确认版本与项目一致）

| 类别 | 产品/框架 | 版本 | 关键依赖 |
|:---|:---|:---|:---|
| 数据库 | TiDB (MySQL 兼容) | 6.5 | `mysql-connector-java:8.0.33` |
| 消息队列 | RocketMQ | 4.9.4 | `rocketmq-client:4.9.4` (原生客户端，非 Starter) |
| 缓存 | Caffeine | 2.9.3 | `caffeine:2.9.3` (本地缓存，非 Redis) |
| 应用框架 | Spring Boot | 2.7.18 | `spring-boot-starter:2.7.18` |
| ORM | MyBatis | 2.3.1 | `mybatis-spring-boot-starter:2.3.1` (XML Mapper，非 MyBatis-Plus) |
| 测试框架 | JUnit 5 | 5.10.0 | `junit-jupiter:5.10.0` |
| Mock 框架 | Mockito | 4.11.0 | `mockito-core:4.11.0`, `mockito-inline:4.11.0` |
| 日志框架 | SLF4J + Logback | 1.2.12 | `logback-classic:1.2.12`, 使用 `@Slf4j` 注解 |

```
🔍 审核项：
☐ TiDB 6.5 的 DDL 兼容性是否已确认？（部分 MySQL 语法不兼容） ⬜ 待 DBA 确认
☐ RocketMQ 原生客户端调用方式是否与 Spec 中的 MQ-01 一致？  ⬜ 待后端确认
☐ MyBatis XML Mapper 模式是否确认？（AI 可能默认用 MyBatis-Plus） ⬜ 待后端确认
☐ Spring Boot 2.7.x 是否确认？（AI 可能使用 3.x 自动配置写法） ⬜ 待后端确认
```

---

## 🛡️ 3.11 质量门禁 (Quality Gates)

```
╔══════════════════════════════════════════════════════════════╗
║                    GATE 1: 事实基准审核                       ║
╠══════════════════════════════════════════════════════════════╣
║ 审核人: 👤 TL — 张三                                          ║
║                                                              ║
║ ☐ 禁止变更清单（RED-01 ~ RED-05）是否完整？                   ║
║ ☐ 存量接口清单（EX-01 ~ EX-02）是否准确？                     ║
║ ☐ 消息队列依赖（MQ-01）是否与真实配置一致？                    ║
║ ☐ 隐形边界声明（3.4）：幂等/并发/事务/错误/时间/安全是否完整？   ║
║ ☐ CodeAnchor（ANC-01 ~ ANC-06）是否全部锚定到真实代码？        ║
║ ☐ Context Injection Checklist 是否已准备完整？               ║
║                                                              ║
║ 签字: ___________          日期: ___________                 ║
╚══════════════════════════════════════════════════════════════╝
                              ↓
╔══════════════════════════════════════════════════════════════╗
║                    GATE 2: 技术契约审核                       ║
╠══════════════════════════════════════════════════════════════╣
║ 审核人: 👤 后端开发 — 李四                                    ║
║                                                              ║
║ ☐ 第三方 API（天猫/京东）字段是否与官方文档一致？               ║
║ ☐ 数据模型（新增表/修改表）是否与数据库对齐？                   ║
║ ☐ Runtime Profile 版本是否与 pom.xml / 实际环境一致？           ║
║ ☐ 异常码是否在系统中已定义？                                   ║
║ ☐ 接口认证方式是否已确认？（OAuth 2.0 / Access Token）          ║
║ ☐ oms_order 表新增字段是否与 RED-03 协商一致？                  ║
║                                                              ║
║ 签字: ___________          日期: ___________                 ║
╚══════════════════════════════════════════════════════════════╝
                              ↓
╔══════════════════════════════════════════════════════════════╗
║                    GATE 3: 测试规划审核                       ║
╠══════════════════════════════════════════════════════════════╣
║ 审核人: 👤 测试工程师 (QA) — 王五                             ║
║                                                              ║
║ ☐ 5 个 Given-When-Then 场景是否覆盖正常+异常分支？              ║
║ ☐ TDD 的 9 个 RED 步骤是否可独立运行？                         ║
║ ☐ 所有 Mock 对象是否来自 CodeAnchor（ANC-01 ~ ANC-06）？       ║
║ ☐ 幂等性、重试、超时等边缘场景是否覆盖？                        ║
║ ☐ 是否有遗漏的业务分支？（如：虚拟商品无库存的情况）             ║
║                                                              ║
║ 签字: ___________          日期: ___________                 ║
╚══════════════════════════════════════════════════════════════╝
                              ↓
╔══════════════════════════════════════════════════════════════╗
║                  GATE 4: 测试质量门禁                          ║
╠══════════════════════════════════════════════════════════════╣
║ 审核人: 👤 测试工程师 (QA) — 王五                             ║
║                                                              ║
║ （AI 生成代码 + 测试代码后，使用 spec-tester 审计测试质量）      ║
║                                                              ║
║ ☐ 5 个 Given-When-Then 场景是否全部有对应测试？                 ║
║ ☐ 每个测试方法是否有 ≥3 个 assert（非 assertNotNull）？         ║
║ ☐ Spec 3.4 中 4 种错误类型是否全部有异常测试？                  ║
║ ☐ 所有 Mock 对象是否在 ANC-01 ~ ANC-06 中有对应？              ║
║ ☐ 过度 Mock 率 < 50%？                                       ║
║ ☐ 测试数据格式是否与 3.7 数据模型一致？                         ║
║ ☐ H2 vs TiDB 环境差异是否已通过集成测试覆盖？                   ║
║ ☐ QA 已亲自运行 mvn test 并确认全部 GREEN？                    ║
║                                                              ║
║ 签字: ___________          日期: ___________                 ║
╚══════════════════════════════════════════════════════════════╝
```

**状态跟踪**：

```
当前状态: Draft → Gate 1 待审
目标: Gate 1 通过 → Gate 2 通过 → Gate 3 通过 → Approved → AI 实现代码+测试
                                                    ↓
                                                Gate 4 通过（测试质量确认）
                                                    ↓
                                                ReadyForReview → 上线
```

---

## 🔍 3.12 AI 代码审查清单 (Code Review Checklist)

> 👤 **填写人**: TL — 张三
> 🔍 **审核人**: TL — 张三（逐项确认后合并代码）

### AI 可自动检查

```
☐ Checkstyle 规则通过（项目配置: anta-checkstyle.xml）
☐ 未使用的 import 已清除
☐ 方法圈复杂度 ≤ 10
☐ 无空 catch 块
☐ 无 System.out.println 残留
```

### 人必须手动检查

```
业务正确性
☐ VirtualItemIdentifier 的识别逻辑是否与 Spec 3.3 NEW-01 一致？
☐ AutoSplitService 的拆单逻辑是否与 Spec 3.3 NEW-02 一致？
☐ AutoShipListener 的发货逻辑是否与 Spec 3.3 NEW-03 一致？
☐ 极端场景（拆单失败、发货超时、消息重复）处理是否与 3.3 极端场景表一致？

技术安全性
☐ AutoShipListener.onMessage() 中是否有正确的幂等逻辑？（Spec 3.4.1）
☐ auto_ship_record 表写入是否使用了唯一索引防重？（Spec 3.4.2）
☐ 天猫发货 + 本地记录更新的事务顺序是否正确？（Spec 3.4.3）
☐ RED-01 ~ RED-05 红线区的文件是否被误改？
☐ 天猫 API Secret / 京东 Access Token 是否未出现在日志中？（Spec 3.4.6）

代码一致性
☐ 是否统一使用 @Resource 注入？
☐ 异常处理是否使用 BizException（业务异常）和 PlatformException（平台异常）？
☐ 日志是否使用 @Slf4j？卡券密码是否完全不出现在日志中？
☐ 是否有重复造轮子？（如自写 DateUtil 而项目已有 com.anta.common.util.DateUtil）
☐ 包结构是否与项目一致？（com.anta.oms.{service, listener, config}）
```

```
🔍 审核项：
☐ TL 是否已逐项检查以上清单？
☐ 是否有任何检查项需要退回 AI 修改？
```

---

## 🚀 3.13 部署契约 (Deployment Contract)

> 👤 **填写人**: DevOps — 赵六
> 🔍 **审核人**: DevOps（自审）+ TL — 张三（确认安全合规）

### 容器化配置

| 配置项 | 真实值 | AI 禁止修改 |
|:---|:---|:---:|
| 基础镜像 | `anolisos:8.8-jre`（龙蜥操作系统，安全审批） | ✅ |
| JVM 参数 | `-Xms1g -Xmx2g -XX:+UseG1GC` | ✅ |
| 端口 | `8080` | — |

### 资源限制（基于压测：200 QPS 虚拟发货场景）

| 配置项 | 真实值 |
|:---|:---|
| CPU request | `500m` |
| CPU limit | `2000m` |
| Memory request | `1Gi` |
| Memory limit | `2Gi` |

### 健康检查

| 配置项 | 真实值 |
|:---|:---|
| Liveness Probe | `/actuator/health` |
| Readiness Probe | `/actuator/health/readiness` |
| 启动等待 | `30s` |

### K8s 安全策略

| 配置项 | 真实值 |
|:---|:---|
| runAsNonRoot | `true` |
| readOnlyRootFilesystem | `true` |
| allowPrivilegeEscalation | `false` |

### CI/CD Pipeline

| 步骤 | 命令 | 说明 |
|:---|:---|:---|
| 编译 | `mvn clean compile` | |
| 测试 | `mvn test` | **不可加 -DskipTests** |
| 代码检查 | `mvn checkstyle:check` | |
| 构建镜像 | `docker build -t harbor.anta.com/oms/order-service:${TAG} .` | |
| 推送镜像 | `docker push harbor.anta.com/oms/order-service:${TAG}` | |

```
🔍 审核项：
☐ 基础镜像 anolisos:8.8-jre 是否来自安全部门审批列表？       ⬜ 待 DevOps 确认
☐ 资源限制是否基于实际压测数据？                            ⬜ 待 DevOps 确认
☐ /actuator/health 端点是否在应用中存在？                   ⬜ 待后端确认
☐ mvn test 是否在 Pipeline 中且未被跳过？                   ⬜ 待 DevOps 确认
☐ 天猫 API Key/Secret 是否通过 VaultAgent 获取？（非明文）    ⬜ 待安全审计
```

---

> 👤 **准备人**: TL — 张三 + 后端开发 — 李四
> 🔍 **审核人**: TL — 张三

```
[运行时画像 — 防基础设施幻觉]
☐ TiDB 6.5 + mysql-connector-java 8.0.33
☐ RocketMQ 4.9.4 原生客户端（非 Spring Boot Starter）
☐ MyBatis 2.3.1 XML Mapper 模式（非 MyBatis-Plus）
☐ Spring Boot 2.7.18（非 3.x）
☐ Caffeine 2.9.3 本地缓存（非 Redis）
☐ JUnit 5.10 + Mockito 4.11

[数据库 Schema]
☐ oms_order 表的 DDL（含新增字段 has_virtual_item, virtual_status）
☐ 新增表 virtual_item_rule 和 auto_ship_record 的 DDL
☐ OrderStatusEnum 枚举类定义

[存量接口 — CodeAnchor V2 注入包]
☐ ANC-01 OrderQueryService.queryById() 完整源文件片段 + 调用示例
☐ ANC-02 OrderStatusService.updateStatus() 完整源文件片段 + StatusEnum
☐ ANC-03 OrderSplitExecutor.executeSplit() 完整源文件片段 + throws + SplitResultVO + 调用示例
☐ ANC-04 TmallAdapter.virtualShip() 完整源文件片段 + VirtualShipRequest 结构
☐ ANC-05 JdAdapter.virtualShip() 完整源文件片段 + JdVirtualShipReq 结构
☐ ANC-06 OutboundEventListener.onMessage() 完整源文件片段

[第三方 API]
☐ 天猫 virtualShip API Request/Response JSON Schema
☐ 京东 noLogisticsShip API Request/Response JSON Schema
☐ OAuth 2.0 Token 获取方式示例

[测试框架]
☐ pom.xml 测试依赖片段（JUnit 5, Mockito 4.x, SpringBootTest）
☐ 已有测试基类 BaseServiceTest
☐ 已有测试目录结构: src/test/java/com/anta/

[项目规范]
☐ 包结构: com.anta.oms.{service, listener, config}
☐ 日志框架: SLF4J + Logback，使用 @Slf4j
☐ 异常处理: 业务异常继承 BizException，平台异常继承 PlatformException
☐ DI 规范: 统一使用 @Resource 注入
☐ 日志脱敏: 手机号/卡券密码的 Logback 脱敏规则
```

---

*Spec 状态: InReview | 下一步: Gate 1 审核*
*核心理念: Spec 是 AI Coding 的第一生命线，每一个 🔍 标记的审核项都必须在 Gate 环节由对应角色签字确认。*
