# 如何让AI作出杰出的软件

---

## 一、问题：为什么AI生成的代码总是"差点意思"

### 1.1 AI编程的三个阶段

```
第一阶段：辅助编程
  └─ AI是代码补全工具

第二阶段：协作编程
  └─ AI是能写代码的助手

第三阶段：自主编程 ← 我们在这里
  └─ AI是能独立完成任务的Agent
```

### 1.2 核心问题：Implementation Trap（实现陷阱）

**Martin Fowler 的洞察**：

> AI太快了。描述一个功能，几秒内就能生成几百行代码。
> 但"理解要做什么"和"讨论如何构建"是两件不同的事。
> AI把它们合并成了一件。

**AI跳过的是什么**：
- 组件边界如何划分
- 使用现有基础设施还是引入新抽象
- 接口应该是什么样
- 如何处理错误

### 1.3 代价：隐形设计债务

AI不仅跳过设计，还**倾向于添加功能**。

请求一个通知服务 → AI返回包含：
- 速率限制
- 分析钩子
- Webhook系统
- ...这些你根本没要求的东西

**每一行你没要求添加的代码**：
- 都是必须审查的技术债务
- 都是必须写的测试
- 都是必须维护的表面积

---

## 二、解决方案：Design-First Collaboration

### 2.1 白板原则

**人类工程师如何协作**：

```
配对编程时，优秀的工程师不会一开始就写代码
↓，他们会先去白板
↓
讨论组件、辩论数据流、争论边界
↓
只有对齐后，才坐下来写代码
↓
白板不是开销，是真正思考发生的地方
```

### 2.2 Design-First五层模型

**让AI先讨论设计，再写代码**：

| 层级 | 内容 | 问题 |
|------|------|------|
| **Level 1** | Capabilities（能力） | 系统需要做什么？核心需求，不含实现细节 |
| **Level 2** | Components（组件） | 有哪些构建块？服务、模块、主要抽象 |
| **Level 3** | Interactions（交互） | 组件如何通信？数据流、API调用、事件 |
| **Level 4** | Contracts（契约） | 接口是什么？函数签名、类型、模式 |
| **Level 5** | Implementation（实现） | 写代码 |

**核心规则**：代码只在Level 5出现，且必须经过上级批准。

### 2.3 实践案例

**场景**：构建一个通知服务

**Level 1 - Capabilities**：
- AI确认理解：v1版本仅邮件投递
- 需要重试逻辑
- 基本状态追踪
- 明确：SMS/Webhook不在范围内

**Level 2 - Components**：
- AI提议5个组件，包括一个`RetryQueue`抽象封装BullMQ
- 工程师质疑：这个抽象是必要的吗？
- BullMQ内置重试机制+指数退避，不需要额外抽象
- AI简化了组件结构

**Level 3 - Interactions**：
- 更简单的组件结构带来更直接的数据流
- Handler → BullMQ job → worker处理
- 重试由BullMQ原生处理

**Level 4 - Contracts**：
- 定义接口：`NotificationPayload`, `NotificationResult`, `EmailProvider`, `DeliveryTracker`
- **副作用**：可以用这些契约生成测试
- 先测试，再实现（TDD）

**Level 5 - Implementation**：
- 代码比任何单一提示词生成的都更对齐
- 范围已确认
- 架构适合现有基础设施
- 契约已定义并测试

---

## 三、Spec-Driven Development vs Vibe Engineering

### 3.1 两种范式对比

| | Spec-Driven Development | Vibe Engineering |
|---|---|---|
| **核心理念** | 用规格文档控制AI | 强化工程师专业能力 |
| **依赖** | 流程文档 | 测试、代码审查 |
| **代表人物** | GitHub、AWS | Simon Willison |
| **问题** | 可能变成Waterfall 2.0 | 需要高素质工程师 |

### 3.2 SDD的三个层次

```
Spec-First     → 写规格，然后生成代码（目前大多停留于此）
Spec-Anchored  → 规格作为持续对齐的锚点
Spec-as-Source → 规格成为源代码的正式来源
```

### 3.3 现实情况

- 大多数SDD工具停留在第一层
- 规格写完用完就丢弃
- 没有换到相应的品质保证

### 3.4 Vibe Engineering

**Simon Willison（Python Django共同发明者）提出**：

> 与其靠流程文件控制AI，不如强化既有的工程实践。
> 让AI放大开发者的专业能力。

**关键洞察**：
- SDD是以流程文档为本
- Vibe Engineering是以开发者专业为本
- 后者更实际

---

## 四、Spec-Kit vs OpenSpec

### 4.1 Spec-Kit（重规范）

**特点**：
- 严格的AI编程工作流体系
- 强约束流程
- 适合：复杂企业级项目

**批评**：
- 小项目没必要
- 大项目规格描述不清楚
- 大量文档占用上下文，反而影响生成
- 文档不及时更新会误导Agent

### 4.2 OpenSpec（轻规范）

**特点**：
- 极简务实的规范
- 只记录必要信息
- 适合：小型迭代项目

### 4.3 核心问题

**不是选哪个工具，而是**：

> 你的规范是否真正对齐了代码？
> 规范是否成为source of truth？

---

## 五、人机协作最佳实践

### 5.1 知识工程

**为什么重要**：

```
AI = 正规员工
需要给他们编写详尽的文档
```

**知识工程包括**：
- 业务逻辑文档
- 团队规范
- 领域知识
- 代码架构决策

### 5.2 渐进式设计对话

**一个具体提示词模板**：

```
I need to build [feature]. Before writing any code, walk me through the design:

- Capabilities: [your scope constraints]
- Components: [your infrastructure and architecture constraints]
- Interactions: [your integration and data flow patterns]
- Contracts: [your type and interface conventions]

Present each level separately. Wait for my approval before moving to the next.
No code until the contracts are agreed.
```

### 5.3 复杂度分级

| 任务复杂度 | 从哪层开始 | 示例 |
|-----------|-----------|------|
| 简单工具 | Level 4（契约） | 日期格式化、字符串处理 |
| 单组件 | Level 2（组件） | 验证服务、API端点 |
| 多组件功能 | Level 1（能力） | 通知系统、支付集成 |
| 新系统集成 | Level 1 + 深度Level 3 | 第三方API、事件驱动管道 |

### 5.4 Human-in-the-Loop

**核心原则**：

> AI不只是为人类工作，而是与人类一起工作。
> 没有治理和Human-in-the-Loop保护，AI不只是自动化工作——它加速错误。

---

## 六、能力要求：从使用者到设计者

### 6.1 开发者技能转型

```
AI编程时代要求的技能转变：

旧技能：
- 写代码
- 调试bug
- 优化性能

新技能：
- 定义规范
- 评估产出
- 架构思维
- Agent治理
```

### 6.2 知识体系构建

**团队核心竞争力**：

> 未来团队的核心竞争力，将取决于构建、维护和利用「AI可读写知识体系」的能力。

---

## 七、结论：如何让AI作出杰出的软件

### 7.1 核心原则

| 原则 | 说明 |
|------|------|
| **Design-First** | 先讨论设计，再写代码 |
| **Spec作为Source of Truth** | 规格要持续对齐，不能写完就丢 |
| **Human-in-the-Loop** | 人要参与关键决策点 |
| **知识工程** | 为AI提供可读写的上下文 |

### 7.2 行动清单

**个人层面**：
- [ ] 学习Design-First方法论
- [ ] 建立架构思维，不只是编码能力
- [ ] 学会定义规范和评估产出

**团队层面**：
- [ ] 建立知识工程体系
- [ ] 制定AI编程规范和流程
- [ ] 平衡效率提升与风险管控

**技术层面**：
- [ ] 选择合适的AI编程工具（Cursor/Kiro/Gemini CLI）
- [ ] 建立代码审查和质量门禁
- [ ] 完善测试体系

### 7.3 最终洞察

> AI不会改变软件架构本质上需要人类理解问题和解决方案必须适应的上下文。
> 但AI会改变软件架构的实现方式——通过更多软件围绕AI的本质上不确定性。

**最简单的规则**：

```
规则：不要代码，直到设计被认可。
一切从这里开始。
```

---

*参考资料：Martin Fowler《Design-First Collaboration》、Thoughtworks《Spec-Driven Development》、ihower《Spec-Driven Development的美好愿景与残酷现实》*
