# AssistantAgent 技术报告

> 版本: 0.2.6 | 生成日期: 2026-05-13

---

## 1. 项目概述

AssistantAgent 是一个基于 **Spring Boot 3.4 + Spring AI 1.1.0** 的多模块 Java 17 项目，实现了 **CodeAct（Code as Action）** 模式的智能体框架。核心理念是让 LLM 生成 Python 代码，然后在 GraalVM 沙箱中安全执行，实现"代码即行动"的 Agent 范式。

项目采用 **Spring AI Alibaba** 作为 LLM 后端（默认模型 `qwen-max`），集成了评估引擎、经验管理、搜索、回复、学习、触发器、动态工具等多个扩展子系统。

### 1.1 技术栈

| 技术 | 版本 | 用途 |
|---|---|---|
| Java | 17 | 编程语言 |
| Spring Boot | 3.4.8 | 应用框架 |
| Spring AI | 1.1.0 | AI 抽象层 |
| Spring AI Alibaba | 1.1.2.2 | DashScope LLM 后端 |
| GraalVM Polyglot | 24.2.1 | Python 沙箱执行 |
| OpenTelemetry | 1.35.0 | 可观测性 |
| Spring AI MCP Client | - | MCP 协议工具集成 |
| TinyPinyin | - | 中文转拼音命名规范化 |

### 1.2 项目版本

- **GroupId:** `com.alibaba.agent.assistant`
- **ArtifactId:** `assistant-agent`
- **Version:** `0.2.6`
- **License:** Apache 2.0

---

## 2. 架构设计

### 2.1 模块依赖关系

项目包含 8 个 Maven 模块，依赖链如下：

```
assistant-agent-common  (共享枚举、常量、接口、工具类)
        ↓
assistant-agent-core  (GraalVM 沙箱执行器、工具注册表、CodeAct 引擎)
        ↓
assistant-agent-evaluation  (评估图引擎)
assistant-agent-prompt-builder  (动态 Prompt 组装)
        ↓
assistant-agent-extensions  (扩展模块集合：experience, learning, search, reply, trigger, dynamic)
        ↓
assistant-agent-management  (经验 CRUD API + SKILL 导入导出)
assistant-agent-autoconfigure  (Spring Boot 自动配置)
        ↓
assistant-agent-start  (可运行 Spring Boot 应用 + 示例配置)
```

### 2.2 模块概览

| 模块 | 职责 | 核心类 |
|---|---|---|
| `common` | 共享类型系统：枚举、常量、CodeactTool 接口、参数树、返回值 Schema | `CodeactTool`, `ParameterTree`, `ShapeNode` |
| `core` | GraalVM 沙箱执行器、工具注册表、代码上下文、可观测性 | `GraalCodeExecutor`, `DefaultCodeactToolRegistry` |
| `evaluation` | 基于 DAG 的多维度意图识别评估引擎 | `EvaluationSuite`, `GraphBasedEvaluationExecutor` |
| `prompt-builder` | 动态 Prompt 组装框架（Strategy + Chain of Responsibility） | `PromptContributor`, `PromptContributorManager` |
| `extensions` | 7 个扩展子系统（搜索、回复、经验、学习、触发器、动态工具、评估集成） | 各子模块 SPI 接口 |
| `management` | REST API 控制器 + SKILL 导入导出 + Web 管理控制台 | `ExperienceManagementController` |
| `autoconfigure` | Spring Boot 自动配置，组装完整 Agent | `CodeactAgent`, `CodeactAgentBuilder` |
| `start` | 可运行应用入口 + Demo 数据 + 配置文件 | `AssistantAgentApplication`, `CodeactAgentConfig` |

### 2.3 核心架构模式

项目采用 **CodeAct（Code as Action）** 模式，区别于传统的 Function Calling / Tool Use 模式：

```
传统模式:  User → LLM → Tool Call → Result → LLM → Response
CodeAct:   User → LLM → write_code → GraalVM 沙箱执行 → execute_code → Result → Response
```

在 CodeAct 模式下，LLM 生成完整的 Python 函数代码，而非单次工具调用参数。代码可以组合多个工具、包含复杂逻辑、处理异常情况，具有更强的表达能力和灵活性。

### 2.4 运行时数据流

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────┐
│  BEFORE_AGENT Hooks (按优先级排序执行)            │
│  5: CodeactToolsStateInitHook (注入工具元数据)    │
│  8: CodeactToolSignatureAgentHook (注入Python签名)│
│  20: FastIntentReactHook (快速意图匹配)           │
│  50: FastIntentHook (快速意图检测)                │
│  100: BeforeAgentEvaluationHook (评估套件执行)    │
│  200: PromptContributorModelHook (Prompt注入)     │
└─────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────┐
│  ReactAgent 循环 (LLM 推理)                      │
│  LLM → write_code (注册Python函数)               │
│  LLM → execute_code (GraalVM沙箱执行)            │
│  LLM → send_message (回复用户)                   │
│  LLM → search/unified_search (搜索信息)          │
│  LLM → subscribe_trigger (创建定时任务)           │
└─────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────┐
│  AFTER_AGENT Hooks                               │
│  AfterAgentLearningHook (在线学习)                │
└─────────────────────────────────────────────────┘
```

---

## 3. 模块详细分析

### 3.1 assistant-agent-common（共享基础模块）

#### 3.1.1 模块职责

作为整个项目的根基模块，提供了所有其他模块共享的类型系统、常量定义和工具接口。不包含业务逻辑，仅定义契约。

#### 3.1.2 核心类结构

**枚举类型：**

| 枚举 | 值 | 用途 |
|---|---|---|
| `Language` | `PYTHON`, `JAVASCRIPT`, `JAVA` | 支持的编程语言 |
| `ParameterType` | `STRING`, `INTEGER`, `NUMBER`, `BOOLEAN`, `OBJECT`, `ARRAY`, `NULL`, `UNKNOWN` | JSON Schema 类型映射 |
| `PrimitiveType` | `STRING`, `INTEGER`, `NUMBER`, `BOOLEAN`, `NULL` | 返回值原始类型 |
| `SchemaSource` | `DECLARED`, `OBSERVED` | Schema 来源（声明 vs 运行时观察） |

**常量类：**

- `CodeactStateKeys` — 定义了 `OverAllState` 中所有状态键（如 `GENERATED_CODES`, `EXECUTION_HISTORY`, `EXPERIENCE_PREFETCH_QUERY` 等），是整个 Agent 管道的共享"状态字典"。
- `HookPriorityConstants` — 定义 Hook 执行优先级常量，数值越小优先级越高（5-500）。

**核心接口 `CodeactTool`：**

```java
public interface CodeactTool extends ToolCallback {
    CodeactToolDefinition getCodeactDefinition();
    CodeactToolMetadata getCodeactMetadata();
    default String getName();
    default String getDescription();
    default ParameterTree getParameterTree();
    default ReturnSchema getDeclaredReturnSchema();
}
```

**领域子接口（SPI 标记接口）：**

| 接口 | 扩展 | 领域 | 内部枚举 |
|---|---|---|---|
| `ExperienceCodeactTool` | `CodeactTool` | 经验管理 | `ExperienceOperationType {SAVE,QUERY,UPDATE,DELETE}` |
| `SearchCodeactTool` | `CodeactTool` | 搜索 | `SearchScope {PROJECT,KNOWLEDGE,WEB,UNIFIED}` |
| `ReplyCodeactTool` | `CodeactTool` | 回复 | `ReplyChannelType {PRIMARY,SECONDARY}` |
| `LearningCodeactTool` | `CodeactTool` | 学习 | `LearningOperationType {TRIGGER,QUERY,STOP}` |
| `TriggerCodeactTool` | `CodeactTool` | 触发器 | `TriggerSourceType {TIME,EVENT,CONDITION,MANUAL}` |

#### 3.1.3 参数类型系统

模块实现了一套完整的 **JSON Schema 到结构化类型系统** 的映射：

```
CodeactToolDefinition
  ├── ParameterTree (从 JSON Schema inputSchema 解析)
  │     ├── List<ParameterNode> (参数节点)
  │     │     ├── name, type (ParameterType), description
  │     │     ├── required, defaultValue, format, enumValues
  │     │     ├── items (ARRAY), properties (OBJECT)
  │     │     └── unionVariants (oneOf/anyOf)
  │     ├── Set<String> requiredNames
  │     └── toPythonSignature() → 生成 Python 函数签名
  │
  └── ReturnSchema (返回值结构描述)
        ├── successShape (ShapeNode)
        ├── errorShape (ShapeNode)
        └── typeHint, sources, sampleCount
```

**ShapeNode 层次结构（返回值类型建模）：**

```
ShapeNode (抽象基类)
  ├── PrimitiveShapeNode  (原始类型: str/int/float/bool/None)
  ├── ObjectShapeNode     (固定字段对象: Dict[str, Any])
  ├── ArrayShapeNode      (数组: List[T])
  ├── MapShapeNode        (动态键字典: Dict[str, T])
  ├── UnionShapeNode      (联合类型: Union[T1, T2, ...])
  └── UnknownShapeNode    (未知类型: Any)
```

每个 ShapeNode 都能通过 `getPythonTypeHint()` 生成对应的 Python 类型提示，用于 Prompt 生成。

---

### 3.2 assistant-agent-core（核心执行引擎）

#### 3.2.1 模块职责

提供 CodeAct 模式的核心执行能力：GraalVM 沙箱执行器、工具注册表、代码上下文管理、返回值 Schema 观察和 OpenTelemetry 可观测性。

#### 3.2.2 GraalVM 沙箱执行器（`GraalCodeExecutor`）

**核心执行流程：**

```
execute(functionName, args, toolContext)
  │
  ├── 1. 代码解析: SessionCodeManager.getFunction() 获取 GeneratedCode
  │     (会话级代码覆盖全局代码)
  │
  ├── 2. 函数名提取: environmentManager.extractFunctionName()
  │
  ├── 3. 代码组装:
  │     ├── generateImports() (Python 导入)
  │     ├── injectCustomVariables() (自定义变量注入)
  │     ├── getMergedFunctions() (会话+全局函数合并)
  │     └── buildFunctionInvocationCode() (安全的参数绑定)
  │         └── 使用 inspect.signature 过滤参数
  │
  ├── 4. GraalVM 执行:
  │     ├── Context.newBuilder("python")
  │     │     .allowHostAccess(HostAccess.ALL)
  │     │     .allowIO(false)  // 安全限制
  │     │     .allowNativeAccess(false)
  │     ├── 注入桥接对象:
  │     │     ├── agent_tools (AgentToolBridge → 调用 Spring AI ToolCallback)
  │     │     ├── agent_state (StateBridge → 读写 OverAllState)
  │     │     ├── logger (LoggerBridge → SLF4J 日志)
  │     │     └── CodeactTool 动态生成的 Python 代理类/函数
  │     ├── eval(code) 执行
  │     └── 结果转换 (Value → Java 对象)
  │         ├── null, Number, Boolean, String
  │         ├── Array → List
  │         ├── Dict → Map (递归转换)
  │         └── 循环检测 (identity hash set)
  │
  └── 5. 返回 ExecutionRecord (成功/失败, 结果, 耗时, 调用追踪)
```

**安全设计：**
- 默认禁用 IO 和 Native Access（`allowIO=false`, `allowNativeAccess=false`）
- 执行超时控制（默认 30 秒）
- 每次执行创建新的 GraalVM Context，避免状态泄漏
- 结果在 Context 关闭前完成转换，防止延迟求值问题

**CodeactTool 注入机制：**

`GraalCodeExecutor` 会将注册的 `CodeactTool` 动态生成为 Python 可调用的代码：

```python
# 类级工具 → 生成 Python 类
class SearchTools:
    @staticmethod
    def search_project(query: str, top_k: int = 10) -> Dict[str, Any]:
        """..."""
        return json.loads(__tool_registry__.callTool("search_project", json.dumps({"query": query, "top_k": top_k})))

# 全局工具 → 生成独立函数
def send_message(text: str) -> Dict[str, Any]:
    return json.loads(__tool_registry__.callTool("send_message", json.dumps({"text": text})))
```

Python 代码通过 `__tool_registry__`（`ToolRegistryBridge`）回调 Java 端的工具实现。

#### 3.2.3 工具注册表（`DefaultCodeactToolRegistry`）

三层架构：

```
Layer 1: CodeactToolRegistry (接口)
  ├── register(CodeactTool)
  ├── getTool(name) / getToolByAlias(alias)
  ├── getToolsForLanguage(Language)
  ├── getReturnSchema(toolName)
  └── generateStructuredToolPrompt(Language)

Layer 2: DefaultCodeactToolRegistry (实现)
  ├── ConcurrentHashMap<String, CodeactTool> tools
  ├── ConcurrentHashMap<String, String> aliasToName
  └── ConcurrentHashMap<String, CodeactToolDefinition> toolDefinitions
  注册时: 解析 inputSchema → ParameterTree, 注册 ReturnSchema

Layer 3: ToolRegistryBridge (Python 回调代理)
  ├── callTool(toolName, argsJson) → 查找工具 → 调用 → 记录调用追踪
  ├── 观察返回值 Schema → ShapeExtractor → ReturnSchemaMerger
  └── 追踪 callTrace 和 replyToUserTrace
```

**返回值 Schema 观察管道：**

```
Python 调用工具 → ToolRegistryBridge.callTool()
  → ReturnSchemaRegistry.observe(toolName, jsonResult)
    → ShapeExtractor.extract(json) → ShapeNode 树
      ├── 区分 ObjectShapeNode (固定字段) 和 MapShapeNode (动态键)
      └── 通过标识符分析和值同质性判断
    → ReturnSchemaMerger.mergeShapes() → 增量合并
      ├── 对象: 缺失字段标记为 optional
      ├── 数组: 合并 item 类型
      └── 联合: 添加新变体
```

#### 3.2.4 代码上下文管理

```
CodeContext (内存级函数注册表, 按 Language 分组)
  └── ConcurrentHashMap<String, GeneratedCode>

SessionCodeManager (会话级代码管理, 持久化到 OverAllState)
  ├── getFunction() → 会话代码优先于全局代码
  ├── getMergedFunctions() → 合并会话+全局函数 (会话覆盖全局)
  └── 支持序列化/反序列化 GeneratedCode
```

#### 3.2.5 可观测性（OpenTelemetry）

六种 Span 类型：

| Span | 用途 | 上下文类 |
|---|---|---|
| `hook` | Hook 执行观测 | `HookObservationContext` |
| `interceptor` | 拦截器执行观测 | `InterceptorObservationContext` |
| `react` | React 阶段观测 | `ReactPhaseObservationContext` |
| `execution` | 代码执行观测 | `CodeactExecutionObservationContext` |
| `codegen` | 代码生成观测 | `CodeGenerationObservationContext` |
| `tool.call` | 工具调用观测 | `CodeactToolCallObservationContext` |

辅助类：
- `BaseAgentObservationLifecycleListener` — 实现 `GraphLifecycleListener`，管理会话级 Span 生命周期
- `OpenTelemetryObservationHelper` — 独立的 Span 管理工具
- `HookObservationHelper` / `InterceptorObservationHelper` — 静态观测工具

---

### 3.3 assistant-agent-evaluation（评估图引擎）

#### 3.3.1 模块职责

实现基于 **有向无环图（DAG）** 的多维度意图识别评估引擎。将评估准则编译为 DAG，支持依赖感知排序、并行执行、批处理、条件执行、超时处理和多模态评估。

#### 3.3.2 图构建与执行

**图构建流程（`EvaluationSuiteBuilder`）：**

```
EvaluationCriteria (评估准则列表)
  │
  ├── 每个准则 → 图节点 (包装 CriterionEvaluationAction)
  │
  ├── buildExecutionLevels() (拓扑排序)
  │     ├── Level 0: 无依赖的准则
  │     ├── Level N: max(依赖准则的 level) + 1
  │     └── 同一层级的准则可并行执行
  │
  └── 连接边: START → level0_nodes → join_0 → level1_nodes → join_1 → ... → END
      (fan-out/fan-in 模式实现并行)
```

**单准则执行流程（`CriterionEvaluationAction`）：**

```
apply(OverAllState)
  │
  ├── 1. 提取依赖结果 (<depName>_result 状态键)
  │
  ├── 2. 条件执行检查 (ConditionalExecutionConfig)
  │     └── 不满足 → 返回 SKIPPED 结果
  │
  ├── 3. 超时控制 (CompletableFuture.get(timeoutMs))
  │
  ├── 4a. 批处理模式 (CriterionBatchingConfig):
  │     ├── 解析源集合 (SourcePathResolver)
  │     ├── 分批处理 (Semaphore 并发控制)
  │     └── 聚合结果 (BatchAggregationStrategy)
  │
  └── 4b. 普通模式:
        ├── ExecutionContextFactory 创建上下文
        ├── EvaluatorRegistry 查找评估器
        └── evaluator.evaluate() 执行评估
```

#### 3.3.3 评估器类型

| 类型 | 实现类 | 用途 |
|---|---|---|
| `LLM_BASED` | `LLMBasedEvaluator` | 使用 ChatModel 评估，支持结构化响应解析（RESULT:/REASONING:） |
| `RULE_BASED` | `RuleBasedEvaluator` | Java 代码逻辑评估（阈值、格式校验等） |
| `MULTIMODAL_LLM_BASED` | `MultimodalLLMBasedEvaluator` | 扩展 LLM 评估器，支持图像/媒体处理 |

#### 3.3.4 结果类型

| 类型 | 说明 |
|---|---|
| `BOOLEAN` | 布尔判断 |
| `ENUM` | 枚举选择（带选项列表） |
| `SCORE` | 数值评分 |
| `TEXT` | 文本输出 |
| `JSON` | JSON 结构 |
| `LIST` | 列表结果 |

#### 3.3.5 批处理聚合策略

| 策略 | 逻辑 |
|---|---|
| `ANY_TRUE` | 任一批次为 true 则返回 true（OR 门） |
| `ALL_TRUE` | 所有批次为 true 才返回 true（AND 门） |
| `MERGE_LISTS` | 合并所有批次的列表结果（去重保序） |

#### 3.3.6 核心数据模型

```
EvaluationSuite (评估套件)
  ├── name, CompiledGraph, defaultCriterionTimeoutMs (10s)
  └── List<EvaluationCriterion> (评估准则)

EvaluationCriterion (评估准则)
  ├── name, description, resultType, dependsOn
  ├── evaluatorRef, config, workingMechanism
  ├── fewShots, reasoningPolicy, customPrompt
  ├── contextBindings, batchingConfig
  ├── conditionalExecution, multimodalConfig
  └── timeoutMs, defaultValue

EvaluationResult (评估结果)
  ├── Map<String, CriterionResult> (准则名 → 结果)
  ├── timing
  └── EvaluationStatistics (total, success, failed, skipped, timeout, error)
```

---

### 3.4 assistant-agent-prompt-builder（动态 Prompt 组装）

#### 3.4.1 模块职责

提供动态 Prompt 组装框架，支持多个 Prompt 贡献者按优先级顺序注入内容到 LLM 上下文。

#### 3.4.2 核心接口

```java
// Prompt 贡献者 SPI
public interface PromptContributor {
    String getName();
    boolean shouldContribute(PromptContributorContext context);
    PromptContribution contribute(PromptContributorContext context);
    default int getPriority() { return 100; }
}

// 贡献结果（不可变值对象）
public class PromptContribution {
    String systemTextToPrepend();     // 追加到系统提示前面
    String systemTextToAppend();      // 追加到系统提示后面
    List<Message> messagesToPrepend(); // 追加到对话前面
    List<Message> messagesToAppend();  // 追加到对话后面
}
```

#### 3.4.3 组装流程

```
PromptContributorManager.assemble(context)
  │
  ├── 按 getPriority() 升序排序贡献者
  │
  └── 遍历每个贡献者:
      ├── shouldContribute() → false 则跳过
      ├── contribute() → 获取 PromptContribution
      ├── 合并 systemTextToPrepend ("\n\n" 连接)
      ├── 合并 systemTextToAppend ("\n\n" 连接)
      ├── 收集 messagesToPrepend (过滤 SystemMessage)
      ├── 收集 messagesToAppend (过滤 SystemMessage)
      └── 单个贡献者失败不影响其他（try-catch 隔离）
```

#### 3.4.4 注入机制（`PromptContributorModelHook`）

通过 `BEFORE_MODEL` Hook 将贡献内容注入 LLM 上下文：

1. 调用 `PromptContributorManager.assemble()` 获取所有贡献
2. 通过 MD5 哈希去重（避免重复注入相同内容）
3. 包装为 XML 格式的合成工具调用消息：
   ```xml
   <additional_system_guidance>
     <guidance id="ContributorName_abc12345">
       ... 内容 ...
     </guidance>
   </additional_system_guidance>
   ```
4. 以 `AssistantMessage`（包含 `__get_system_guidance__` 工具调用）+ `ToolResponseMessage` 对的形式注入

---

### 3.5 assistant-agent-extensions（扩展模块集合）

这是一个单 Maven 模块，包含 7 个逻辑子包，每个子包通过 SPI 模式提供可插拔扩展。

#### 3.5.1 搜索扩展（Search）

**功能：** 提供统一的多源搜索框架，支持项目代码、知识库、Web 搜索。

**SPI 接口：**

| 接口 | 用途 |
|---|---|
| `SearchProvider` | 单个搜索数据源插件 |
| `SearchFacade` | 统一搜索入口（协调多个 Provider） |

**搜索源类型：** `PROJECT`, `KNOWLEDGE`, `WEB`, `EXPERIENCE`, `CUSTOM`

**工具生成：**
- `SearchCodeactToolFactory` — 为每个 `SearchProvider` 创建一个 `SearchCodeactTool`
- `UnifiedSearchCodeactTool` — 聚合所有 Provider 的统一搜索工具

**配置前缀：** `spring.ai.alibaba.codeact.extension.search`

#### 3.5.2 回复扩展（Reply）

**功能：** 提供可插拔的回复通道框架（IDE 文本、IM 通知、Webhook 等）。

**SPI 接口：**

| 接口 | 用途 |
|---|---|
| `ReplyChannelDefinition` | 回复通道模板定义 |
| `ReplyToolConfigProvider` | 动态回复工具配置提供者 |

**默认配置：** 创建 `send_message` 工具，目标通道 `IDE_TEXT`

**工具生成：** `ReplyCodeactToolFactory` 根据配置 + 通道定义创建 `ReplyCodeactTool`

**配置前缀：** `spring.ai.alibaba.codeact.extension.reply`

#### 3.5.3 经验扩展（Experience）

**功能：** 管理可复用的知识产物（经验），实现 SKILLS 标准的渐进式披露机制。

**经验类型：**

| 类型 | 说明 |
|---|---|
| `REACT` | 工作流策略（LLM 行为指导） |
| `TOOL` | 工具使用指南（含工具绑定配置） |
| `COMMON` | 通用知识（身份信息、平台介绍等） |

**渐进式披露（三级）：**

```
L1: 候选卡展示 → 有限信息（名称、描述、标签）
L2: read_exp → 完整内容（代码、工作流、工具使用说明）
L3: read_exp_doc → 参考文档（详细参考资料）
```

**披露策略：**

| 策略 | 行为 |
|---|---|
| `PROGRESSIVE` | 需要显式调用 read_exp 才能获取完整内容 |
| `DIRECT` | 高置信度/短内容时自动注入到 Prompt |

**快速意图引擎（FastIntent）：**

支持 AND/OR/NOT 组合条件匹配：

| 匹配器 | 条件类型 |
|---|---|
| `message_prefix` | 消息前缀匹配 |
| `message_regex` | 正则表达式匹配 |
| `metadata_exists` | 元数据存在检查 |
| `metadata_equals` | 元数据相等检查 |
| `metadata_in` | 元数据包含检查 |
| `state_equals` | 状态值相等检查 |
| `tool_arg_equals` | 工具参数匹配 |

匹配成功后可直接执行预定义的工具调用计划（`ExperienceArtifact.ReactArtifact`）。

**多租户支持：** 通过 `ExperienceMetadata.tenantIdList` 实现，支持全局经验（`isGlobal()`）。

**配置前缀：** `spring.ai.alibaba.codeact.extension.experience`

#### 3.5.4 学习扩展（Learning）

**功能：** 提供在线/离线学习框架，从 Agent 执行中提取可复用经验。

**SPI 接口：**

| 接口 | 用途 |
|---|---|
| `LearningExtractor<T>` | 通用记录提取器（泛型） |
| `LearningRepository<T>` | 通用记录存储库（泛型） |
| `LearningStrategy` | 学习决策策略 |
| `LearningExecutor` | 学习任务执行器 |

**在线学习触发点：**

| 触发源 | Hook | 说明 |
|---|---|---|
| `AFTER_AGENT` | `AfterAgentLearningHook` | Agent 执行完成后触发 |
| `AFTER_MODEL` | `AfterModelLearningHook` | 模型调用后触发 |
| `TOOL_CALL` | `LearningToolInterceptor` | 工具调用拦截 |

**离线学习：**
- `ExperienceLearningGraph` — 基于图引擎的离线学习
- `LearningScheduledTask` — 定时任务执行器
- `LearningScheduleConfig` — 调度配置（CRON/FIXED_DELAY/FIXED_RATE）

**LLM 驱动的经验提取（`ExperienceLearningExtractor`）：**
1. 判断执行是否值得学习
2. 提取结构化经验（类型、描述、内容、标签）
3. 保存到 `ExperienceRepository`

**配置前缀：** `spring.ai.alibaba.codeact.extension.learning`

#### 3.5.5 触发器扩展（Trigger）

**功能：** 提供 LLM 可创建的定时/事件驱动任务系统，使用 GraalVM 沙箱执行条件/动作/放弃函数。

**核心组件：**

| 组件 | 用途 |
|---|---|
| `TriggerManager` | CRUD 管理器（订阅/取消/暂停/恢复/列表） |
| `TriggerExecutor` | 核心执行器（恢复会话快照 → GraalVM 执行） |
| `ExecutionBackend` | 调度后端 SPI（默认 Spring TaskScheduler） |
| `SpringSchedulerExecutionBackend` | 支持 CRON/FIXED_DELAY/FIXED_RATE/ONE_TIME |

**4 个 CodeactTool：**

| 工具 | 用途 |
|---|---|
| `SubscribeTrigger` | 创建订阅（含调度配置 + Python 函数代码） |
| `UnsubscribeTrigger` | 取消订阅 |
| `ListTriggers` | 列出订阅 |
| `GetTriggerDetail` | 获取订阅详情 |

**执行流程：**

```
触发器触发
  → TriggerExecutor.execute(TriggerDefinition)
    → 恢复 SessionSnapshot (会话状态)
    → 构建 CodeContext + CodeactToolRegistry
    → 创建 GraalCodeExecutor
    → 执行 conditionFunction (Python) → 判断条件
    → 执行 executeFunction (Python) → 执行动作
    → 执行 abandonFunction (Python) → 判断是否放弃
```

**配置前缀：** `spring.ai.alibaba.codeact.extension.trigger`

#### 3.5.6 动态工具扩展（Dynamic）

**功能：** 运行时从外部源（MCP 服务器、OpenAPI/HTTP 端点）动态注册工具。

**SPI 接口：** `DynamicCodeactToolFactory` — 外部工具工厂

**已实现的工厂：**

| 工厂 | ID | 用途 |
|---|---|---|
| `McpDynamicToolFactory` | `"mcp"` | 将 Spring AI MCP `ToolCallbackProvider` 转换为 `CodeactTool` |
| `HttpDynamicToolFactory` | `"http"` | 解析 OpenAPI 3.1 文档，生成 HTTP 工具 |

**MCP 工具适配：**
- `McpToolCallbackAdapter` — 将 MCP `ToolCallback` 适配为 `AbstractDynamicCodeactTool`
- `McpServerAwareToolCallback` — 提供服务器元数据（名称、描述）
- 支持按服务器名称分组工具

**HTTP 工具：**
- 解析 OpenAPI 3.1 文档
- 构建嵌套 inputSchema（path/query/headers/body 参数）
- `HttpDynamicCodeactTool` 执行 HTTP 请求
- 中文工具名自动转拼音（`NameNormalizer`）

#### 3.5.7 评估集成扩展（Evaluation Integration）

**功能：** 将评估框架集成到 Agent 生命周期。

**Hook 类：**

| Hook | 位置 | 用途 |
|---|---|---|
| `BeforeAgentEvaluationHook` | `BEFORE_AGENT` | Agent 执行前运行评估套件 |
| `BeforeModelEvaluationHook` | `BEFORE_MODEL` | 模型调用前评估 |
| `ReactBeforeModelEvaluationHook` | `BEFORE_MODEL` | React 阶段专用评估 |

**配置前缀：** `spring.ai.alibaba.codeact.extension.evaluation`

#### 3.5.8 Prompt 集成扩展（Prompt Integration）

**功能：** 将 prompt-builder 框架集成到 Agent 生命周期。

**核心类：**

| 类 | 用途 |
|---|---|
| `PromptContributorModelHook` | 抽象 `BEFORE_MODEL` Hook，收集并注入 Prompt 贡献 |
| `ReactPromptContributorModelHook` | React 管道具体实现 |
| `OverAllStatePromptContributorContext` | 基于 `OverAllState` 的上下文实现 |
| `EvaluationBasedPromptContributor` | 基于评估结果的抽象 Prompt 贡献者 |

**配置前缀：** `spring.ai.alibaba.codeact.extension.prompt`

---

### 3.6 assistant-agent-management（管理模块）

#### 3.6.1 模块职责

提供 REST API 控制器、SKILL 导入导出和 Web 管理控制台。

#### 3.6.2 REST API 端点

**经验管理 API (`/exp-console/api/experiences`)**

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 分页列表（类型/关键词/租户筛选） |
| GET | `/search` | 关键词搜索 |
| GET | `/stats` | 按类型统计 |
| GET | `/{id}` | 获取详情 |
| GET | `/{id}/assets/**` | 加载资产内容 |
| POST | `/` | 创建经验 |
| POST | `/{id}/resummarize` | 重新生成引用描述 |
| PUT | `/{id}` | 更新经验 |
| DELETE | `/{id}` | 删除经验 |

**SKILL 导入导出 API (`/exp-console/api/skills`)**

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/import` | 从 SKILL.md 导入 |
| POST | `/preview` | 预览导入 |
| POST | `/import-package` | 从 ZIP/tgz 导入 |
| POST | `/preview-package` | 预览包导入 |
| GET | `/export/{id}` | 导出为 SKILL.md |
| GET | `/export-package/{id}` | 导出为 ZIP 下载 |
| GET | `/export?type=X` | 批量导出 |

**工具源 API (`/exp-console/api/tool-sources`)**

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 列出所有工具源 |
| GET | `/{sourceId}/tools` | 列出源中的工具 |
| POST | `/{sourceId}/import` | 导入工具 |
| GET | `/{sourceId}/imported` | 列出已导入工具 |
| POST | `/{sourceId}/sync` | 同步工具 |

**租户 API (`/exp-console/api/tenants`)**

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 列出可用租户 |

#### 3.6.3 SKILL 导入导出

**SKILL 包格式（ZIP/tgz）：**

```
skill-package/
  ├── SKILL.md           (必需，含 YAML frontmatter + Markdown 正文)
  ├── package.json       (可选，含 name/version/description/cli 配置)
  ├── references/        (参考文档，.md/.yaml/.yml)
  ├── scripts/           (脚本，标记为 sandbox-only asset)
  ├── assets/            (其他资产)
  └── evals/             (评估配置)
```

**导入流程：**

```
SkillPackage (ZIP/tgz)
  ├── SkillPackageParser.parseAuto() (魔术字节检测格式)
  │     ├── tgz: 手动 tar 解析（无外部依赖）
  │     └── zip: ZipInputStream 标准解析
  │
  ├── SkillContentClassifier (文件分类: REFERENCE vs ASSET)
  │
  ├── DescriptionResolver (描述解析优先链)
  │     ├── H1 标题提取
  │     ├── YAML frontmatter description
  │     ├── ReferenceSummarizer (LLM 生成)
  │     └── 回退: "(no description) <path>"
  │
  ├── 创建 REACT 经验 (SKILL.md → Experience)
  │
  ├── 提取 CLI 绑定 (package.json → CliBinding)
  │
  └── 创建 TOOL 经验 (如有 CLI 绑定)
      └── 链接 REACT ↔ TOOL (relatedExperiences)
```

#### 3.6.4 SPI 接口

| 接口 | 用途 | 默认实现 |
|---|---|---|
| `ExperienceManagementService` | 经验 CRUD 服务 | `RepositoryBackedExperienceManagementService` |
| `SkillExchangeService` | SKILL 导入导出 | `InMemorySkillExchangeService` |
| `ToolSourceBrowser` | 外部工具源浏览 | `InMemoryToolSourceBrowser` (空实现) |
| `TenantListProvider` | 租户列表提供 | Lambda（空列表） |
| `ReferenceSummarizer` | 引用描述生成 | `LlmReferenceSummarizer`（有 ChatModel）或 `NoopReferenceSummarizer` |

---

### 3.7 assistant-agent-autoconfigure（自动配置模块）

#### 3.7.1 模块职责

Spring Boot 自动配置层，将所有模块（common、core、extensions、evaluation）和外部库（Spring AI Alibaba、GraalVM、MCP Client、Studio）组装为完整的 CodeAct Agent 系统。

#### 3.7.2 核心 Agent 组装（`CodeactAgent` + `CodeactAgentBuilder`）

`CodeactAgent` 继承自 `ReactAgent`，是整个系统的核心入口。通过 `CodeactAgentBuilder` 流式构建：

```
CodeactAgent.builder()
  .name("CodeactAgent")
  .systemPrompt(SYSTEM_PROMPT)
  .model(chatModel)                          // DashScope qwen-max
  .language(Language.PYTHON)
  .enableInitialCodeGen(true)
  .allowIO(false)                            // 安全限制
  .allowNativeAccess(false)
  .executionTimeout(30000)                   // 30s 超时
  .tools(reactTools)                         // React 阶段工具
  .codeactTools(allCodeactTools)             // CodeAct 阶段工具
  .hooks(allHooks)                           // 所有 Hook
  .experienceProvider(...)
  .fastIntentService(...)
  .saver(new MemorySaver())                  // 多轮对话上下文
  .build()
```

**`build()` 内部流程：**

1. 创建 `DefaultCodeactToolRegistry`（含可选 `ReturnSchemaRegistry`）
2. 注册所有 `CodeactTool` 实例
3. 自动注册 `CodeactToolsStateInitHook`（优先级 5，注入工具元数据到状态）
4. 自动注册 `CodeactToolSignatureAgentHook`（优先级 8，注入 Python 工具签名到 Prompt）
5. 创建 `CodeContext`、`PythonEnvironmentManager`、`GraalCodeExecutor`
6. 创建三个核心工具回调：`write_code`、`write_condition_code`、`execute_code`
7. 构建 `AgentLlmNode` 和 `AgentToolNode`（含拦截器和工具）
8. 构造并返回 `CodeactAgent`

#### 3.7.3 核心工具

| 工具 | 类 | 用途 |
|---|---|---|
| `write_code` | `WriteCodeTool` | 注册 LLM 生成的 Python 函数到 CodeContext |
| `write_condition_code` | `WriteConditionCodeTool` | 注册布尔条件函数（用于触发器） |
| `execute_code` | `ExecuteCodeTool` | 在 GraalVM 沙箱中执行已注册的函数 |

#### 3.7.4 评估自动配置（`DefaultEvaluationSuiteConfig`）

```
DefaultEvaluationSuiteConfig
  ├── 条件: spring.ai.alibaba.codeact.extension.evaluation.enabled=true
  ├── 注入: DefaultEvaluationProperties, ChatModel
  ├── SPI: EvaluationCriterionProvider[] (消费模块实现)
  │
  ├── 创建 LLMBasedEvaluator ("llm-based")
  ├── 创建 EvaluationSuite ("default-suite") ← 来自 Provider 的准则
  ├── 创建 EvaluationService (DefaultEvaluationService)
  ├── 创建 CodeactEvaluationContextFactory
  └── 创建 Hook 链:
      ├── ReactBeforeModelEvaluationHook
      └── ReactPromptContributorModelHook
```

#### 3.7.5 SPI 接口

| 接口 | 用途 |
|---|---|
| `EvaluationCriterionProvider` | 消费模块实现，提供自定义评估准则 |

---

### 3.8 assistant-agent-start（启动模块）

#### 3.8.1 模块职责

可运行的 Spring Boot 应用入口，包含 Demo 数据、自定义经验、评估套件和完整配置。

#### 3.8.2 应用启动流程

```
1. main() → SpringApplication.run()
   │
2. 组件扫描 (@ComponentScan):
   │   ├── com.alibaba.assistant.agent.start (Demo 组件)
   │   ├── com.alibaba.assistant.agent.autoconfigure (自动配置)
   │   └── com.alibaba.assistant.agent.extension (扩展实现)
   │
3. Bean 创建 (依赖顺序):
   │   ├── DashScopeChatModel
   │   ├── ExperienceRepository / ExperienceProvider
   │   ├── FastIntentService
   │   ├── MockIdeTextChannelDefinition / Mock*SearchProvider
   │   ├── ExperienceEvaluationCriterionProvider (enhanced_user_input, is_fuzzy)
   │   ├── DemoRuleBasedEvaluationProvider (input_signal_summary)
   │   ├── ReactPhaseGuidancePromptContributor
   │   ├── 所有 Hook beans
   │   └── CodeactAgent (核心 Agent Bean)
   │
4. CommandLineRunner:
   │   ├── Order 100: DemoExperienceConfig (种子 4 条 Demo 经验)
   │   └── Order 1000: ExperienceTestValidator (验证经验查询)
   │
5. ApplicationReadyEvent:
       打印启动横幅 (Chat UI: http://localhost:8080/chatui/index.html)
                    (Console: http://localhost:8080/exp-console/)
```

#### 3.8.3 Demo 经验数据

| 类型 | 名称 | 用途 |
|---|---|---|
| COMMON | 魔力红身份介绍 | 定义 Agent 身份为"魔力红"AI 研发助手 |
| COMMON | 魔礼海平台介绍 | 定义"魔礼海"智能研发平台 |
| REACT | 身份询问响应策略 | 用户问"你是谁"时直接回答，不调用工具 |
| REACT | 小明系数计算 | 复杂多步计算示例，含 FastIntentConfig 和预定义工具计划 |

#### 3.8.4 评估套件

**默认准则（来自 `ExperienceEvaluationCriterionProvider`）：**

| 准则名 | 类型 | 结果类型 | 说明 |
|---|---|---|---|
| `enhanced_user_input` | LLM_BASED | TEXT | 净化用户输入，去除填充词 |
| `is_fuzzy` | LLM_BASED | ENUM(模糊/一般/清晰) | 意图清晰度三级分类 |

**Demo 准则（来自 `DemoRuleBasedEvaluationProvider`）：**

| 准则名 | 类型 | 结果类型 | 说明 |
|---|---|---|---|
| `input_signal_summary` | RULE_BASED | TEXT | 检测输入信号（计算/URL/代码/问题/数字/纯文本） |

#### 3.8.5 配置文件

**`application.yml`（活跃配置）：**

```yaml
server:
  port: 8080

spring.ai:
  dashscope:
    api-key: ${DASHSCOPE_API_KEY}
    chat:
      options:
        model: qwen-max

# 扩展配置（均默认启用）
spring.ai.alibaba.codeact.extension:
  experience.enabled: true
  experience.console.enabled: true
  learning.enabled: true
  reply.enabled: true
  evaluation.enabled: true
  prompt.enabled: true
```

**`application-reference.yml`（完整配置参考）：** 包含所有可用配置选项和详细注释。

---

## 4. 设计模式总结

### 4.1 架构模式

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **CodeAct (Code as Action)** | 整体架构 | LLM 生成代码并在沙箱中执行，而非单次工具调用 |
| **SPI (Service Provider Interface)** | 所有扩展子模块 | 通过 `@Component` 自动发现 + `@ConditionalOnMissingBean` 允许覆盖 |
| **Hook/Interceptor** | Agent 生命周期 | `BEFORE_AGENT`, `BEFORE_MODEL`, `AFTER_AGENT` 等阶段注入行为 |
| **DAG (有向无环图)** | 评估引擎 | 准则依赖关系构建 DAG，支持并行执行 |

### 4.2 设计模式

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **Builder** | `CodeactAgentBuilder`, `PromptContribution.Builder` 等 | 流式构建复杂对象 |
| **Strategy** | `LearningStrategy`, `BatchAggregationStrategy`, `Evaluator` | 运行时选择算法 |
| **Factory** | `SearchCodeactToolFactory`, `ReplyCodeactToolFactory` 等 | 创建工具实例 |
| **Registry** | `CodeactToolRegistry`, `EvaluatorRegistry`, `ReturnSchemaRegistry` | 集中管理可查找对象 |
| **Facade** | `DefaultSearchFacade` | 统一多源搜索入口 |
| **Template Method** | `EvaluationBasedPromptContributor`, `AbstractDynamicCodeactTool` | 定义算法骨架 |
| **Adapter** | `BaseSearchCodeactTool`, `McpToolCallbackAdapter` | 适配接口 |
| **Bridge/Proxy** | `AgentToolBridge`, `StateBridge`, `LoggerBridge` | Java-Python 互操作桥 |
| **Composite** | `ShapeNode` 层次结构 | 递归类型组合 |
| **Observer** | OpenTelemetry 可观测性, `GraphLifecycleListener` | 事件通知 |
| **Chain of Responsibility** | `PromptContributorManager` | 按优先级顺序处理 |

### 4.3 Spring Boot 集成模式

| 模式 | 说明 |
|---|---|
| `@AutoConfiguration` | 各扩展模块独立自动配置 |
| `@ConditionalOnProperty` | 通过配置开关启用/禁用扩展 |
| `@ConditionalOnMissingBean` | 默认实现可被覆盖 |
| `@ConfigurationProperties` | 类型安全的配置绑定 |
| `@Component` 自动发现 | SPI 接口实现自动注册 |

---

## 5. 关键流程分析

### 5.1 工具调用流程

```
LLM 生成 Python 代码
  │  write_code(functionName, description, parameters, code)
  ▼
WriteCodeTool.apply()
  │  验证代码格式 → 去除 Markdown 围栏 → 验证函数名 → 注册到 CodeContext
  ▼
LLM 决定执行
  │  execute_code(functionName, args)
  ▼
ExecuteCodeTool.apply()
  │  获取 OverAllState → 注入自定义变量 → GraalCodeExecutor.execute()
  ▼
GraalCodeExecutor.execute()
  │  解析函数 → 组装代码 → 创建 GraalVM Context → 注入桥接对象
  │  → eval(python_code) → 结果转换 → 返回 ExecutionRecord
  ▼
Python 代码执行中调用工具
  │  search_tools.search_project(query="...")
  │    → __tool_registry__.callTool("search_project", json.dumps({...}))
  │      → ToolRegistryBridge.callTool() → 查找 CodeactTool → tool.call()
  │        → 记录调用追踪 → 观察返回值 Schema → 返回结果
  ▼
执行完成
  │  ExecutionRecord 存入 OverAllState
  ▼
LLM 根据结果决定下一步（回复用户/继续执行）
```

### 5.2 经验渐进式披露流程

```
用户输入 → ExperiencePrefetchHook (BEFORE_AGENT)
  │  查询候选经验 → 存入 OverAllState (EXPERIENCE_PREFETCHED_CANDIDATES)
  ▼
FastIntentReactHook (BEFORE_AGENT, 优先级 20)
  │  匹配 FastIntentConfig → 命中 → 直接执行工具计划
  ▼
未命中 → ExperienceDisclosurePromptContributor (Prompt 注入)
  │  生成 <experience_disclosure> XML → 注入候选卡
  ▼
LLM 决定读取经验 → 调用 read_exp(id)
  │  ExperienceDisclosureService.read() → 返回完整内容
  ▼
LLM 决定读取参考文档 → 调用 read_exp_doc(id, paths)
  │  ExperienceDisclosureService.readDocs() → 返回参考文档
  ▼
LLM 基于经验内容生成代码/执行操作
```

### 5.3 评估流程

```
用户输入 → BeforeAgentEvaluationHook (BEFORE_AGENT)
  │  构建 EvaluationContext (input, executionResult, environment)
  ▼
EvaluationService.evaluate(suite, context)
  │
  ▼
GraphBasedEvaluationExecutor.execute()
  │  注入初始状态 → 识别并行节点 → 绑定线程池
  │  → CompiledGraph.invokeAndGetOutput()
  ▼
图执行 (DAG)
  │  Level 0: enhanced_user_input (LLM 净化输入) ← 并行
  │           input_signal_summary (规则检测信号) ← 并行
  │  Level 1: is_fuzzy (依赖 enhanced_user_input, LLM 判断清晰度)
  │
  ▼
结果存储 → OverAllStateEvaluationResultStore
  │
  ▼
ReactPhaseGuidancePromptContributor
  │  读取 is_fuzzy → 生成执行指导 Prompt
  │  模糊: 不要执行代码，向用户确认
  │  一般: 继续分析，仅确认关键缺失参数
  │  清晰: 直接执行，给出明确结果
```

---

## 6. 配置参考

### 6.1 核心配置

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `spring.ai.dashscope.api-key` | (必填) | DashScope API Key |
| `spring.ai.dashscope.chat.options.model` | `qwen-max` | LLM 模型 |
| `server.port` | `8080` | HTTP 端口 |

### 6.2 扩展配置

| 前缀 | 开关键 | 默认值 | 说明 |
|---|---|---|---|
| `spring.ai.alibaba.codeact.extension.experience` | `enabled` | `true` | 经验管理 |
| `spring.ai.alibaba.codeact.extension.search` | `enabled` | `true` | 搜索 |
| `spring.ai.alibaba.codeact.extension.reply` | `enabled` | `true` | 回复通道 |
| `spring.ai.alibaba.codeact.extension.learning` | `enabled` | `true` | 学习 |
| `spring.ai.alibaba.codeact.extension.trigger` | `enabled` | `true` | 触发器 |
| `spring.ai.alibaba.codeact.extension.evaluation` | `enabled` | `true` | 评估 |
| `spring.ai.alibaba.codeact.extension.prompt` | `enabled` | `true` | Prompt 贡献 |
| `experience.console` | `enabled` | `false` | 管理控制台 |

### 6.3 MCP 配置

```yaml
spring.ai.mcp.client:
  enabled: false          # 默认禁用
  type: SYNC
  request-timeout: 30s
```

---

## 7. 测试策略

### 7.1 测试框架

- JUnit 5 (`junit-jupiter`)
- Spring Boot Test (`spring-boot-starter-test`)
- 测试不需要 `DASHSCOPE_API_KEY`（纯单元测试）

### 7.2 现有测试

| 模块 | 测试类 | 说明 |
|---|---|---|
| core | `GraalCodeExecutorStringLiteralTest` | 测试 Python 字符串字面量转义 |
| core | `ShapeExtractorTest` | 测试返回值 Schema 提取和合并 |
| autoconfigure | `WriteCodeToolRequestDeserializationTest` | 测试参数反序列化容错 |
| management | `InMemorySkillExchangeServiceTest` | 测试 SKILL 包导入 |
| management | `SkillPackageParserTest` | 测试包格式解析 |

### 7.3 测试覆盖目标

- 新代码覆盖率 > 60%
- 遵循 `shouldReturnXWhenY` 命名模式

---

## 8. CI/CD

### 8.1 构建命令

```bash
mvn clean install -DskipTests    # 完整构建（跳过测试）
mvn test                          # 运行所有测试
mvn test -Dtest=ClassName         # 运行单个测试类
```

### 8.2 启动应用

```bash
export DASHSCOPE_API_KEY=your_key
cd assistant-agent-start && mvn spring-boot:run
```

### 8.3 GitHub Actions

工作流文件：`.github/workflows/build.yml`

```bash
mvn clean install -B -V    # 构建
mvn test -B                 # 测试
```

运行环境：JDK 17 (Temurin)

---

## 9. 代码规范

| 规范 | 说明 |
|---|---|
| 代码风格 | Google Java Style (4 空格缩进, 120 字符行宽) |
| 注解处理 | Lombok（`common` 和 `core` 模块需启用） |
| 许可证头 | 所有源文件需要 Apache 2.0 许可证头 |
| Javadoc | 所有公共 API 需要 Javadoc |
| 日志格式 | `ClassName#methodName - description: key={value}` |
| 测试命名 | `shouldReturnXWhenY` 模式 |
| 提交信息 | Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`) |

---

## 10. 附录

### 10.1 SPI 接口总览

| SPI 接口 | 模块 | 发现机制 | 配置前缀 |
|---|---|---|---|
| `SearchProvider` | extensions/search | `@Component` | `extension.search` |
| `SearchFacade` | extensions/search | `@ConditionalOnMissingBean` | `extension.search` |
| `ReplyChannelDefinition` | extensions/reply | `@Component` | `extension.reply` |
| `ReplyToolConfigProvider` | extensions/reply | `@Component` | `extension.reply` |
| `ExperienceProvider` | extensions/experience | `@ConditionalOnMissingBean` | `extension.experience` |
| `ExperienceRepository` | extensions/experience | `@ConditionalOnMissingBean` | `extension.experience` |
| `FastIntentConditionMatcher` | extensions/experience | `@Component` | `extension.experience` |
| `LearningExtractor<T>` | extensions/learning | `@Component` | `extension.learning` |
| `LearningRepository<T>` | extensions/learning | `@Component` | `extension.learning` |
| `LearningStrategy` | extensions/learning | `@ConditionalOnMissingBean` | `extension.learning` |
| `LearningExecutor` | extensions/learning | `@ConditionalOnMissingBean` | `extension.learning` |
| `ExecutionBackend` | extensions/trigger | `@ConditionalOnMissingBean` | `extension.trigger` |
| `DynamicCodeactToolFactory` | extensions/dynamic | 编程式注册 | N/A |
| `PromptContributor` | extensions/prompt | `@Component` | `extension.prompt` |
| `Evaluator` | evaluation | `@Component` | `extension.evaluation` |
| `EvaluationCriterionProvider` | autoconfigure | `@Component` | `starter.evaluation` |
| `ExperienceManagementService` | management | `@ConditionalOnMissingBean` | `experience.console` |
| `SkillExchangeService` | management | `@ConditionalOnMissingBean` | `experience.console` |
| `ToolSourceBrowser` | management | `@ConditionalOnMissingBean` | `experience.console` |
| `TenantListProvider` | management | `@ConditionalOnMissingBean` | `experience.console` |
| `ReferenceSummarizer` | management | `@ConditionalOnBean(ChatModel)` | `experience.console` |
| `CodeactVariableProvider` | core | `@ConditionalOnMissingBean` | N/A |

### 10.2 Hook 优先级表

| 优先级 | Hook | 阶段 | 说明 |
|---|---|---|---|
| 5 | `CodeactToolsStateInitHook` | BEFORE_AGENT | 注入工具元数据到状态 |
| 8 | `CodeactToolSignatureAgentHook` | BEFORE_AGENT | 注入 Python 工具签名 |
| 20 | `FastIntentReactHook` | BEFORE_AGENT | 快速意图匹配 |
| 50 | Fast Intent Hook | BEFORE_AGENT | 快速意图检测 |
| 100 | `BeforeAgentEvaluationHook` | BEFORE_AGENT | 评估套件执行 |
| 100 | `BeforeModelEvaluationHook` | BEFORE_MODEL | 模型阶段评估 |
| 200 | `PromptContributorModelHook` | BEFORE_MODEL | Prompt 贡献注入 |
| 500 | 默认优先级 | — | 无特殊排序 |

### 10.3 状态键（CodeactStateKeys）

| 键 | 类型 | 说明 |
|---|---|---|
| `GENERATED_CODES` | `List<GeneratedCode>` | 当前会话生成的代码 |
| `EXECUTION_HISTORY` | `List<ExecutionRecord>` | 执行历史 |
| `CURRENT_EXECUTION` | `ExecutionRecord` | 当前执行结果 |
| `USER_ID` | `String` | 用户 ID |
| `AVAILABLE_TOOL_NAMES` | `List<String>` | 可用工具白名单 |
| `CODEACT_TOOL_NAMES` | `List<String>` | 所有注入工具名 |
| `CODEACT_TOOL_METADATA_LIST` | `List<Map<String, Object>>` | 工具元数据 |
| `LANGUAGE` | `String` | 编程语言 |
| `EXPERIENCE_PREFETCH_QUERY` | `String` | 首轮预取查询 |
| `EXPERIENCE_PREFETCH_STATUS` | `String` | 预取状态 (NOT_RUN/SKIPPED/COMPLETED) |
| `EXPERIENCE_PREFETCHED_CANDIDATES` | `GroupedExperienceCandidates` | 预取的经验卡 |
| `EXPERIENCE_DETAIL_CACHE` | `Map<String, ReadExpResponse>` | 经验详情缓存 |
| `EXPERIENCE_DIRECT_GROUNDINGS` | `List<DirectExperienceGrounding>` | 直接注入的经验内容 |
| `EXPERIENCE_ALLOWED_REACT_TOOL_NAMES` | `List<String>` | React 直接调用允许的工具 |
| `EXPERIENCE_READ_EXP_IDS` | `List<String>` | 已读取的经验 ID |
| `SESSION_GENERATED_CODES` | `Map<String, GeneratedCode>` | 会话级代码映射 |
