---
layout: post
title: "构建具有长期记忆的 AI 助手 - 基于 LangChain4j 实现永续对话系统"
categories: [Java, AI, LangChain4j, Agent, 长期记忆，工具调用]
description: "深入解析如何使用 LangChain4j 框架实现具有长期记忆和联网搜索能力的 AI 助手，通过本地持久化和工具调用机制打造个性化对话系统"
keywords: [LangChain4j,AI 助手，长期记忆，工具调用，Tavily 搜索，Agent,Java]
---

# 构建具有长期记忆的 AI 助手 - 基于 LangChain4j 实现永续对话系统

> 在 AI 应用开发中，如何让助手拥有"记忆"和"实时信息获取"能力是关键挑战。本文将展示如何使用 LangChain4j 框架，通过本地文件持久化、自定义工具开发和联网搜索集成，构建一个能够记住用户偏好、主动更新记忆并实时获取信息的智能助手。

## 简介

传统的聊天机器人往往缺乏持续记忆能力，每次对话都是独立的。而真正的智能助手应该能够：
- **记住用户信息**：姓名、偏好、历史对话中的重要内容
- **主动更新记忆**：在对话过程中自动记录关键信息
- **获取实时信息**：联网搜索最新新闻、事实和数据
- **自然交互**：流畅的对话体验，不过度暴露技术细节

本文将通过一个完整的实战项目，展示如何实现这些功能。

## 核心架构设计

### 1. 整体架构

```
┌─────────────────────────────────────────────────────┐
│                  用户交互层                          │
│              (Console/REST API)                     │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│               AiServices 代理层                      │
│         (自动管理对话记忆和工具调用)                   │
└─────────────────────────────────────────────────────┘
        ↓                       ↓
┌──────────────────┐   ┌──────────────────┐
│   工具调用层      │   │   聊天模型层      │
│  - MemoryTool   │   │  - Ollama/GPT    │
│  - WebSearch    │   │  - 上下文管理     │
└──────────────────┘   └──────────────────┘
        ↓
┌──────────────────┐
│   持久化存储      │
│  (Markdown 文件)  │
└──────────────────┘
```

### 2. 关键技术组件

- **LangChain4j**：Java 版 LangChain 框架，提供 Agent 编排能力
- **MessageWindowChatMemory**：滑动窗口对话记忆
- **@Tool 注解**：声明式工具定义
- **Tavily Web Search**：联网搜索引擎
- **本地文件存储**：轻量级持久化方案

## 第一步：项目依赖配置

### Maven 依赖管理

```xml
<project>
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>dev.langchain4j</groupId>
                <artifactId>langchain4j-bom</artifactId>
                <version>1.12.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- LangChain4j 核心 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-spring-boot-starter</artifactId>
        </dependency>

        <!-- OpenAI 集成（可选） -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
        </dependency>

        <!-- Ollama 本地模型集成 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-ollama</artifactId>
        </dependency>

        <!-- Tavily 搜索引擎 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-web-search-engine-tavily</artifactId>
        </dependency>

        <!-- Spring Boot Web -->

        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
</project>
```

**关键依赖说明：**
- `langchain4j-bom`：统一版本管理，避免依赖冲突
- `langchain4j-ollama`：支持本地大模型（如 Qwen、Llama）
- `langchain4j-web-search-engine-tavily`：专业的 AI 搜索引擎

## 第二步：模型工厂设计

### 统一的模型配置

```java
@Getter
public class AiFactory {
    
    // 聊天模型配置
    public static final ChatModel model = 
        OllamaChatModel.builder()
            .baseUrl("http://localhost:11434")
            .modelName("gpt-oss:20b")  // 或 qwen3:0.6b
            .temperature(0.3)          // 降低随机性，提高准确性
            .timeout(ofSeconds(60))    // 设置超时
            .logRequests(true)         // 记录请求日志
            .logResponses(true)        // 记录响应日志
            .think(false)              // 禁用思考过程（节省 token）
            .build();

    // 流式聊天模型（用于实时输出）
    public static final OllamaStreamingChatModel streamingChatModel = 
        OllamaStreamingChatModel.builder()
            .baseUrl("http://localhost:11434")
            .modelName("qwen3:0.6b")
            .temperature(0.3)
            .timeout(ofSeconds(60))
            .logRequests(true)
            .logResponses(true)
            .build();
}
```

**配置要点：**
- **Temperature 设置**：0.3 适合事实性问答，降低幻觉
- **Timeout 设置**：60 秒防止长时间等待
- **日志记录**：调试阶段开启，生产环境关闭
- **模型选择**：本地模型用 Ollama，云端可用 OpenAI

## 第三步：长期记忆工具实现

### 记忆管理工具类

```java
public static class MemoryTool {
    private static final String FILE_NAME = "ai_perpetual_memory.md";
    private final Path path = Paths.get(FILE_NAME);

    public MemoryTool() {
        // 初始化记忆文件
        if (!Files.exists(path)) {
            try {
                Files.writeString(path, 
                    "# AI 长期记忆\n\n" +
                    "## 用户画像\n未记录\n\n" +
                    "## 核心事件摘要\n暂无");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @Tool("读取本地 Markdown 长期记忆。当你需要了解用户的历史背景、偏好或之前的对话摘要时使用。")
    public String readLongTermMemory() {
        try {
            System.out.println("[系统] 🧠 正在读取长期记忆...");
            return Files.readString(path);
        } catch (IOException e) {
            return "读取失败：" + e.getMessage();
        }
    }

    @Tool("更新本地 Markdown 长期记忆。当你获知了用户的新偏好、新身份或重要事件时，请提炼并重写完整的 Markdown 内容存入。")
    public String updateLongTermMemory(String newContent) {
        try {
            System.out.println("[系统] ✍️ 正在更新长期记忆...");
            Files.writeString(path, newContent);
            return "记忆更新成功。";
        } catch (IOException e) {
            return "更新失败：" + e.getMessage();
        }
    }
}
```

**设计亮点：**
1. **@Tool 注解**：声明式工具定义，LangChain4j 自动注册
2. **自然语言描述**：工具描述清晰，帮助 AI 理解何时调用
3. **异常处理**：友好的错误提示，避免程序崩溃
4. **初始化逻辑**：首次运行时自动创建记忆文件

### 记忆文件结构示例

```markdown
# AI 长期记忆

## 用户画像
- 姓名：张三
- 职业：软件工程师
- 技术栈：Java, Spring Boot, Kubernetes
- 兴趣：人工智能、分布式系统

## 核心事件摘要
### 2026-03-09
- 用户开始使用本 AI 助手
- 用户对 LangChain4j 框架感兴趣
- 用户计划开发智能客服系统
```

## 第四步：Agent 接口定义

### 系统提示词设计

```java
public interface Assistant {
    @SystemMessage({
        "你是一个拥有'永续记忆'和'实时搜索'能力的 AI 助手。",
        
        // 1. 记忆能力
        "1. 记忆：你通过本地 Markdown 文件存储长期信息。",
        "   - 如果用户提到以前的事情，先调用 readLongTermMemory。",
        "   - 如果不确定用户的历史信息，主动查询记忆。",
        
        // 2. 搜索能力
        "2. 搜索：如果你对当前事实不确定或需要获取实时新闻，",
        "   你必须调用 searchWeb 工具进行联网搜索。",
        "   - 不要依赖过时的训练数据。",
        "   - 涉及日期、新闻、股价等动态信息时必须搜索。",
        
        // 3. 记忆更新
        "3. 记录：每当对话中有值得永久记住的信息时，",
        "   立即调用 updateLongTermMemory 更新文件。",
        "   - 用户姓名、职业、喜好",
        "   - 重要计划和目标",
        "   - 特殊事件和约定",
        
        // 4. 回答风格
        "4. 回答：直接回答用户，语气自然。",
        "   - 只有在确实搜索了资料时才简要说明来源。",
        "   - 不要过度暴露技术细节。"
    })
    String chat(String message);
}
```

**提示词设计原则：**
- **明确职责**：清晰定义记忆和搜索的使用场景
- **触发条件**：具体说明何时调用工具
- **行为准则**：规定回答风格和边界
- **结构化**：分条列出，便于模型理解

## 第五步：构建永续对话服务

### 主程序入口

```java
public static void main(String[] args) {
    // 1. 强制指定 HTTP 客户端（解决依赖冲突）
    System.setProperty(
        "langchain4j.http.clientBuilderFactory",
        "dev.langchain4j.http.client.jdk.JdkHttpClientBuilderFactory"
    );

    // 2. 初始化 Tavily 搜索引擎
    WebSearchEngine searchEngine = TavilyWebSearchEngine.builder()
        .apiKey("tvly-dev-xxxx-xxxx") // ⚠️ 生产环境请放入环境变量
        .timeout(Duration.ofSeconds(5))
        .build();

    // 3. 构建 Agent 服务
    Assistant agent = AiServices.builder(Assistant.class)
        .chatModel(AiFactory.model)
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .tools(new MemoryTool(), WebSearchTool.from(searchEngine))
        .build();

    // 4. 控制台交互循环
    Scanner scanner = new Scanner(System.in);
    System.out.println("=== 🤖 永续助手已启动 (输入 exit 退出) ===");

    while (true) {
        System.out.print("\nUser: ");
        String userInput = scanner.nextLine();
        if ("exit".equalsIgnoreCase(userInput)) break;

        try {
            String response = agent.chat(userInput);
            System.out.println("\nAI: " + response);
        } catch (Exception e) {
            System.err.println("发生错误：" + e.getMessage());
        }
    }
    scanner.close();
}
```

**关键配置解析：**

#### 1. HTTP 客户端配置
```java
System.setProperty(
    "langchain4j.http.clientBuilderFactory",
    "dev.langchain4j.http.client.jdk.JdkHttpClientBuilderFactory"
);
```
**作用**：当 Classpath 存在多个 HTTP 客户端（如 OkHttp、Apache HttpClient）时，强制指定使用 JDK 原生客户端，避免冲突。

#### 2. 对话记忆配置
```java
.chatMemory(MessageWindowChatMemory.withMaxMessages(20))
```
**作用**：保留最近 20 条消息作为短期记忆，超出后自动移除最早的消息。

#### 3. 工具注册
```java
.tools(new MemoryTool(), WebSearchTool.from(searchEngine))
```
**作用**：注册记忆工具和搜索工具，AI 可以自主决定调用时机。

## 第六步：完整交互流程演示

### 实际运行效果

![AI 助手控制台界面](/images/posts/ai/20260309.png)

*图：AI 助手在控制台中的实际运行效果，展示了从用户提问到记忆更新的完整流程*

### 场景 1：查询实时信息

```
User: 今天几号，有什么国际新闻吗

[系统] 🔍 正在搜索网络...  (AI 自动调用 searchWeb)

AI: 今天是 **2026 年 3 月 9 日**。以下是当天的主要国际新闻：

| 主题 | 主要内容 |
|------|----------|
| **油价飙升** | 由于伊朗军事行动升级，全球原油供应受压，油价显著上涨 |
| **美国部分政府停摆** | 机场安检出现长达数小时的排队现象 |
| **国际妇女节** | 联合国及各国政府举行性别平等庆祝活动 |

如果你想了解更详细的报道，随时告诉我！
```

**流程解析：**
1. 用户提问涉及日期和新闻
2. AI 识别需要实时信息
3. 自动调用 `searchWeb` 工具
4. 整合搜索结果生成回答

### 场景 2：记忆更新与新闻摘要

```
User: 总结下今天的新闻，写入记忆里面

[系统] ✍️ 正在更新长期记忆...  (AI 自动调用 updateLongTermMemory)

AI: 已将今天（2026‑03‑09）的主要新闻摘要写入长期记忆。
    如果需要进一步讨论或获取更详细信息，随时告诉我！
```

**记忆文件更新后：**
```markdown
# AI 长期记忆

## 用户画像
- 姓名：李明
- 职业：Java 开发工程师

## 核心事件摘要
### 2026-03-09
- 用户自我介绍
- **今日新闻摘要**：
  - 伊朗军事行动导致油价飙升
  - 美国部分政府停摆影响机场安检
  - 国际妇女节全球庆祝活动
```

**关键点：**
- AI 主动调用 `updateLongTermMemory` 工具
- 自动提炼新闻要点并格式化存储
- 保持记忆文件的结构化格式

### 场景 3：记忆读取

```
User: 你还记得我是做什么的吗？

[系统] 🧠 正在读取长期记忆...  (AI 自动调用 readLongTermMemory)

AI: 当然记得！你是一名 Java 开发工程师，我们之前聊过 LangChain4j 框架和 AI 助手开发。
```

## 核心技术原理：LangChain4j ReAct 模式

### 什么是 ReAct？

**ReAct (Reasoning + Acting)** 是一种结合推理和行动的智能体架构。LangChain4j 通过 ReAct 模式，让 AI 能够：
- **推理（Reasoning）**：分析用户问题，判断是否需要外部信息
- **行动（Acting）**：调用工具获取所需数据
- **再推理（Re-reasoning）**：基于工具结果生成最终回答

### ReAct 工作流程

```plain
用户提问 → AI 分析 → 需要工具？→ 是 → 调用工具 → 获取结果 → 生成回答
                              ↓ 否
                              → 直接回答
```

### LangChain4j 的 ReAct 实现

LangChain4j 通过 `AiServices` 自动处理 ReAct 循环：

1. **第一次 LLM 调用**：分析问题，决定是否使用工具
2. **工具执行**：如果需要，调用相应的工具（如 `readLongTermMemory`、`searchWeb`）
3. **第二次 LLM 调用**：基于工具结果生成最终回答

这个过程完全自动化，开发者无需手动编写循环逻辑。

### ReAct 在本文的应用

**场景示例：用户问"你还记得我是做什么的吗？"**

```
第 1 轮 - Reasoning:
├─ 用户问题：询问历史信息
├─ AI 判断：需要查询记忆
└─ 决策：调用 readLongTermMemory 工具

第 2 轮 - Acting:
├─ 执行工具：readLongTermMemory()
├─ 返回结果："用户是 Java 开发工程师..."
└─ 将结果加入对话历史

第 3 轮 - Re-reasoning:
├─ 基于记忆结果重新分析
└─ 生成回答："当然记得！你是一名 Java 开发工程师..."
```

### ReAct 的优势

| 特性 | 传统方式 | ReAct 方式 |
|------|----------|------------|
| **决策能力** | 固定流程 | AI 自主判断是否使用工具 |
| **灵活性** | 硬编码逻辑 | 动态选择工具 |
| **准确性** | 可能答非所问 | 基于实时信息回答 |
| **可扩展性** | 需修改代码 | 添加新工具即可 |

**关键点：**
- ✅ **两次 LLM 调用**：第一次决定行动，第二次生成回答
- ✅ **工具结果反馈**：将工具执行结果加入上下文
- ✅ **自主决策**：AI 根据问题类型自主选择工具
- ✅ **透明过程**：可以在控制台看到工具调用日志

## 实践注意事项

### 1. 记忆文件格式化

**推荐结构：**
```markdown
# AI 长期记忆

## 用户画像
- 基本信息：姓名、年龄、地区
- 职业背景：职位、行业、技术栈
- 个人偏好：兴趣爱好、习惯

## 核心事件摘要
### YYYY-MM-DD
- 事件 1
- 事件 2

### YYYY-MM-DD
- ...
```

**优点：**
- 结构清晰，便于 AI 理解
- 按时间排序，易于追溯
- Markdown 格式，人类可读性强

### 2. 搜索引擎配置

```java
WebSearchEngine searchEngine = TavilyWebSearchEngine.builder()
    .apiKey(System.getenv("TAVILY_API_KEY"))  // 从环境变量读取
    .maxResults(5)                            // 限制结果数量
    .timeout(Duration.ofSeconds(5))           // 超时控制
    .build();
```

**🔑 API Key 获取步骤：**
1. 访问 [https://app.tavily.com/home](https://app.tavily.com/home) 注册账号
2. 登录后在 Dashboard 页面复制 API Key
3. 设置环境变量（推荐方式）：
   ```bash
   # Windows PowerShell
   $env:TAVILY_API_KEY="tvly-your-api-key"
   
   # Linux/Mac
   export TAVILY_API_KEY="tvly-your-api-key"
   ```
4. 或者直接在代码中使用（仅限测试）：
   ```java
   .apiKey("tvly-your-api-key")
   ```

**💰 免费额度说明：**
- **Developer 计划**：每月 1000 次搜索调用
- **无需信用卡**：注册即可使用
- **个人开发友好**：适合学习、原型开发和小规模应用
- **升级灵活**：超出后可升级到 Pro 计划（$19/月，1万次调用）

**优化建议：**
- API Key 放入环境变量，避免硬编码
- 根据需求调整 `maxResults`，平衡准确性和速度
- 设置合理的超时时间

### 3. 对话记忆大小

```java
.chatMemory(MessageWindowChatMemory.withMaxMessages(20))
```

**选择策略：**
- **10-20 条**：适合日常对话，节省 token
- **20-50 条**：适合复杂任务，保持更多上下文
- **100+ 条**：适合专业领域，需要大量背景信息

## 性能优化建议

### 1. 记忆读取缓存

```java
public class CachedMemoryTool extends MemoryTool {
    private String cachedContent;
    private long lastReadTime;

    @Override
    public String readLongTermMemory() {
        // 5 分钟内的缓存有效
        if (cachedContent != null && 
            System.currentTimeMillis() - lastReadTime < 300000) {
            return cachedContent;
        }
        
        cachedContent = super.readLongTermMemory();
        lastReadTime = System.currentTimeMillis();
        return cachedContent;
    }
}
```

### 2. 批量记忆更新

```java
@Tool("批量更新记忆条目")
public String batchUpdateMemories(List<MemoryEntry> entries) {
    StringBuilder sb = new StringBuilder();
    sb.append("# AI 长期记忆\n\n");
    
    // 合并新旧记忆
    mergeMemories(sb, entries);
    
    Files.writeString(path, sb.toString());
    return "批量更新成功，共 " + entries.size() + " 条记录";
}
```

### 3. 异步日志记录

```java
@Component
public class AsyncLogger {
    
    @Async
    public void logInteraction(String userInput, String aiResponse, long duration) {
        // 异步写入日志文件
        // 用于后续分析和优化
    }
}
```

## 扩展功能方向

### 1. 多模态记忆

```java
@Tool("保存对话截图到本地")
public String saveConversationSnapshot(String imageUrl) {
    // 下载并保存图片
    // 记录图片路径到记忆文件
}
```

### 2. 知识图谱集成

```java
@Tool("查询知识图谱中的实体关系")
public String queryKnowledgeGraph(String entity) {
    // 连接 Neo4j 或其他图数据库
    // 返回关联实体和信息
}
```

### 3. 定时任务提醒

```java
@Tool("设置定时提醒")
public String scheduleReminder(String task, LocalDateTime time) {
    // 创建定时任务
    // 到时间后主动通知用户
}
```

## 总结

| 组件 | 功能 | 实现方式 |
|------|------|----------|
| **长期记忆** | 持久化用户信息 | Markdown 文件 + @Tool |
| **短期记忆** | 保持对话上下文 | MessageWindowChatMemory |
| **联网搜索** | 获取实时信息 | Tavily Web Search |
| **工具调用** | 扩展 AI 能力 | LangChain4j @Tool |
| **模型适配** | 支持多种 LLM | Ollama/OpenAI |

**核心优势：**
1. ✅ **真正的长期记忆**：不依赖向量数据库，轻量级实现
2. ✅ **实时信息获取**：联网搜索避免知识过时
3. ✅ **灵活的扩展性**：可以轻松添加新工具
4. ✅ **本地化部署**：支持 Ollama 等本地模型，保护隐私
5. ✅ **简洁的架构**：无需复杂的中间件

**适用场景：**
- 个性化 AI 助手
- 智能客服系统
- 学习伴侣
- 项目管理助手
- 代码助手

通过本文的介绍，你已经掌握了构建具有长期记忆 AI 助手的完整技术栈。下一步可以尝试：
- 添加更多实用工具（日历、待办事项、文件管理）
- 集成向量数据库实现语义搜索
- 开发 Web 界面提供更友好的交互体验
- 接入企业系统集成业务数据

动手实践吧，打造属于你自己的专属 AI 助手！🚀
