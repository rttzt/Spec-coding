# AGENTS.md — 安踏 OMS 项目 AI 编码规范

> **这是什么？** 这份文件是写给 AI 编码助手看的"项目入职手册"。AI 每次开始对话时会自动读取它，就像你入职时看的 README。
>
> **什么该放这里？** 跨所有功能不变的永久性约束：技术栈版本、代码规范、永久红线、项目结构约定。
>
> **什么不该放这里？** 随需求变化的信息（CodeAnchor、特定 API 字段、新增表结构）——这些放在每个 Spec 文档里。

---

## 1. 项目身份 (Project Identity)

| 字段 | 值 |
|:---|:---|
| **项目名称** | 安踏 OMS 订单管理系统 |
| **组织** | 安踏集团 |
| **主要语言** | Java 8+ |
| **构建工具** | Maven |
| **代码仓库** | [内部 GitLab] |
| **配置中心** | Apollo |
| **密钥管理** | Vault / KMS |

---

## 2. 运行时画像 (Runtime Profile)

> ⚠️ **AI 的训练数据是"互联网平均"，而项目的运行时环境有特定版本。以下信息用于防止框架/中间件版本误用。**

| 类别 | 产品/框架 | 版本 | 关键依赖 |
|:---|:---|:---|:---|
| 应用框架 | Spring Boot | 2.7.18 | `spring-boot-starter:2.7.18` |
| ORM | MyBatis-Plus | 3.5.3.1 | `mybatis-plus-boot-starter:3.5.3.1` |
| 数据库 | MySQL | 8.0.33 | `mysql-connector-java:8.0.33` |
| 消息队列 | RocketMQ | 4.9.4 | `rocketmq-client:4.9.4` |
| 缓存 | Redis (Jedis) | 3.7.0 | `jedis:3.7.0` |
| 测试框架 | JUnit 5 | 5.10.0 | `junit-jupiter:5.10.0` |
| Mock 框架 | Mockito | 4.11.0 | `mockito-core:4.11.0` |
| 日志框架 | SLF4J + Logback | 1.2.12 | `logback-classic:1.2.12` |
| JSON | Jackson | 2.15.3 | `jackson-databind:2.15.3` |
| 工具库 | Lombok | 1.18.30 | `lombok:1.18.30` |
| 工具库 | Hutool | 5.8.23 | `hutool-all:5.8.23` |

---

## 3. 项目模块结构

```
anta-oms/
├── oms-common/          # 公共模块：工具类、通用异常、基础 VO
├── oms-order-service/   # 订单服务（主要修改区）
├── oms-platform-adapter/ # 平台适配层：天猫、京东等第三方发货
├── oms-finance/         # 财务模块（⚠️ 绝对禁区）
├── oms-message/         # 消息模块：MQ 消费和生产
└── oms-api/             # 对外 API 网关
```

### 包命名规范

```
com.anta.oms.{模块}.{分层}

分层：
  controller  — REST 接口层
  service     — 业务逻辑层（接口）
  service.impl — 业务逻辑层（实现）
  mapper      — MyBatis Mapper
  entity      — 数据库实体
  vo/dto      — 数据传输对象
  config      — 配置类
  enums       — 枚举类
  exception   — 异常类
```

---

## 4. 永久红线 (Permanent Fact Base)

> 🛑 **这些模块/类/表绝对不能修改！即使某个 Spec 提出了需求，也需要 TL 单独审批。**

| 编号 | 模块/文件/表 | 禁止原因 | 误改影响 |
|:---|:---|:---|:---|
| RED-00 | `com.anta.oms.finance.*` | 财务结算逻辑，涉及对账 | 财务数据错误，资损 |
| RED-01 | `com.anta.oms.order.PaymentService` | 支付流程核心逻辑 | 支付失败或重复扣款 |
| RED-02 | `oms_finance.*` 表族 | 财务数据库表 | 历史数据损坏 |
| RED-03 | `com.anta.oms.common.BaseEntity` | 所有实体的基类 | 全系统级影响 |

---

## 5. 代码规范 (Coding Conventions)

### 5.1 日志

```java
// ✅ 正确：使用 Lombok + SLF4J
@Slf4j
public class XxxService {
    public void doSomething() {
        log.info("订单处理开始, orderId={}", orderId);
        log.error("发货失败, orderId={}", orderId, e);
    }
}

// 🚫 禁止
System.out.println(...)
System.err.println(...)
e.printStackTrace()
```

### 5.2 异常处理

```java
// ✅ 正确：统一使用 BizException
throw new BizException(ErrorCode.ORDER_NOT_FOUND, "订单不存在: " + orderId);

// 🚫 禁止吞异常
try { ... } catch (Exception e) { }  // 什么都不做
try { ... } catch (Exception e) { e.printStackTrace(); }  // 只打印不处理
```

### 5.3 事务

```java
// ✅ 正确：只在写操作上加事务
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) { ... }

// 🚫 禁止：读操作加事务、事务内调用外部 API
```

### 5.4 返回值

```java
// ✅ 统一用 ResultVO<T> 包装
public ResultVO<OrderDetailVO> getOrder(Long orderId) { ... }

// ResultVO 结构：
// { "code": 200, "message": "success", "data": { ... } }
```

### 5.5 Lombok

```java
// ✅ 必须使用
@Data / @Getter / @Setter
@Slf4j
@Builder
@AllArgsConstructor / @NoArgsConstructor

// 🚫 禁止手写 getter/setter/toString
```

### 5.6 命名

| 类型 | 规范 | 示例 |
|:---|:---|:---|
| Service 接口 | `XxxService` | `OrderService` |
| Service 实现 | `XxxServiceImpl` | `OrderServiceImpl` |
| Mapper | `XxxMapper` | `OrderMapper` |
| Entity | 表名驼峰 | `Order` |
| VO | `XxxVO` | `OrderDetailVO` |
| DTO | `XxxDTO` | `CreateOrderDTO` |
| 枚举 | `XxxEnum` | `OrderStatusEnum` |

---

## 6. AI 行为约束 (AI Behavior Rules)

> 🛑 这些是 AI 编码时必须遵守的硬性规则。如果在实现过程中遇到冲突，必须停下来确认，不得自行绕过。

### 6.1 绝对禁止

| 🛑 行为 | 原因 |
|:---|:---|
| 自行填写接口类名和方法签名（臆造） | 必须通过 CodeAnchor 锚定真实代码 |
| 在同一次响应中同时生成测试和实现 | 违反 TDD 三回合原则 |
| 生成包含明文密钥的代码 | 密钥必须通过配置中心/KMS 引用 |
| 修改财务模块（`com.anta.oms.finance.*`）的任何代码 | 永久红线 |
| 声称"测试通过"而不提供 `mvn test` 的实际输出 | 防止自我验证幻觉 |
| 声称"Spec 已就绪，可以进入实现" | 签字是人的责任 |

### 6.2 必须做到

| ✅ 行为 | 说明 |
|:---|:---|
| 引用接口前先确认 CodeAnchor 是否在 Spec 中 | 不臆造接口 |
| 生成代码后输出完整的文件路径 | 便于定位和编译验证 |
| 遇到不确定的技术选型时明确提出替代方案 | 不自行决定 |
| 修改已有文件时给出变更摘要 | 说明改了什么、为什么改 |

---

## 7. 验证命令 (Verification Commands)

```bash
# 编译单个模块
mvn compile -pl oms-order-service

# 运行单个测试类
mvn test -pl oms-order-service -Dtest=OrderSplitTest

# 运行单个测试方法
mvn test -pl oms-order-service -Dtest=OrderSplitTest#testSplitSuccess

# 运行全部测试
mvn test

# 代码风格检查
mvn checkstyle:check
```

---

## 8. Spec Coding 工作流关联

> 本文件是 Spec Coding V5 框架的**永久上下文层**。每个 Spec 只需要关注本次需求的独有信息。

```
AGENTS.md（本文件）
  ↓ 每次对话自动加载（永久层）
  │  项目技术栈 · 代码规范 · 永久红线 · 验证命令
  │
  ├──→ spec-writer（写 Spec）
  │     AI 已知项目规范，只问需求相关的问题
  │
  ├──→ spec-reviewer（审 Spec）
  │     新增 Pass 0：本 Spec 是否与 AGENTS.md 冲突？
  │
  └──→ AI 增量实现（写代码）
        AI 已知版本和规范，代码风格自动对齐
```

### 与 Spec 的分工

| 信息层 | 存放位置 | 内容 |
|:---|:---|:---|
| **永久层** | `AGENTS.md` | 技术栈版本、代码规范、永久红线、项目结构 |
| **临时层** | 每个 Spec 的注入清单 | CodeAnchor 接口、专用 API 字段、新增表 DDL |

---

## 9. 维护规则

- **谁维护？** TL（技术负责人）负责维护本文件。
- **何时更新？**
  - 技术栈升级（如 Spring Boot 升级）时
  - 发现新的永久红线时
  - 代码规范变更时
  - 新增项目模块时
- **何时不改？** 每个功能需求的 Spec 信息不写入这里——放到 Spec 里。

---

*文件版本: 1.0.0 | 创建日期: 2026-04-25 | 配套框架: Spec Coding V5 人机协同版*
