---
layout: post
title: "LangChain4J代码执行工具的设计与实现 - 自主编程Agent实践"
categories: [Java, AI, LangChain4J, 大语言模型, 工具开发]
description: "深入解析基于LangChain4J的代码执行工具设计，展示如何构建具备自主编程能力的AI Agent"
keywords: [LangChain4J,代码执行,AI Agent,自主编程,Groovy,大语言模型]
---

# LangChain4J代码执行工具设计与实现 - 自主编程Agent实践

> 在AI应用开发中，如何让大语言模型具备执行代码的能力是一个重要课题。本文将详细介绍LangChain4J框架的典型用法，展示如何构建具备自主编程能力的AI Agent。

## 简介

随着大语言模型技术的发展，越来越多的应用场景需要AI不仅能生成文本，还要能够执行代码、进行数学计算和逻辑推理。本文将展示如何利用LangChain4J框架构建具备代码执行能力的AI工具。

## 核心亮点解析

### 1. 自主编程能力的设计

实现AI"自主编程"能力的核心在于以下设计：

```java
@SystemMessage({
    "你是一个具备'自我编程'能力的自主 Agent。",
    "核心指令：",
    "1. 对于任何数学计算、逻辑推理或数据处理，你必须编写 Groovy 代码并调用 executeCode 执行。",
    "2. 你可以在代码中使用 println 来打印中间过程或调试信息，这些信息会被工具捕获并返回给你。",
    "3. 如果代码运行报错，请根据错误信息进行修正，直到获得正确结果。",
    "4. 最终回答时，请告知用户你执行的代码内容。"
})
String chat(String userMessage);
```

这种设计让AI Agent具备了以下能力：
- **自我纠错**：能够根据执行结果自动修正代码错误
- **渐进式求解**：通过多次迭代逐步完善解决方案
- **透明化过程**：向用户展示完整的思考和执行过程

### 2. 控制台输出捕获机制

传统方式中，AI只能看到代码执行的最终返回值，无法获取中间过程信息。通过巧妙的设计可以解决这个问题：

```java
@Tool("执行 Groovy 代码。支持 println 打印语句。返回内容包含控制台输出和脚本返回值。")
public String executeCode(String code) {
    // 创建一个 StringWriter 来捕获脚本中的 println 输出
    StringWriter sw = new StringWriter();
    ScriptContext context = new SimpleScriptContext();
    context.setWriter(sw); // 将脚本的标准输出重定向到我们的 sw

    try {
        System.out.println("\n--- [工具触发] 正在执行 ---");
        System.out.println(code);

        // 执行代码，传入我们自定义的 context
        Object result = engine.eval(code, context);

        String capturedOutput = sw.toString();

        // 组装返回给 AI 的信息
        StringBuilder response = new StringBuilder();
        if (!capturedOutput.isEmpty()) {
            response.append("--- 控制台输出 ---\n").append(capturedOutput).append("\n");
        }
        response.append("--- 脚本返回值 ---\n").append(result != null ? result.toString() : "null");

        System.out.println(response);
        return response.toString();
    } catch (Exception e) {
        return "Execution Error: " + e.getMessage();
    }
}
```

**关键技术点：**
1. 使用`StringWriter`捕获脚本的控制台输出
2. 通过`ScriptContext`重定向标准输出流
3. 分别返回控制台输出和脚本返回值，提供完整的信息反馈

### 3. 错误处理与自我修复

完善的错误处理机制让AI能够根据错误信息进行自我修正：

```java
catch (Exception e) {
    return "Execution Error: " + e.getMessage();
}
```

这种设计使得AI Agent能够在遇到错误时：
- 识别具体的错误类型和位置
- 分析错误原因
- 自动生成修正后的代码
- 重新执行验证

## LangChain4J典型用法详解

### 1. 基础架构搭建

LangChain4J的核心架构围绕以下几个组件构建：

```java
// 1. 定义AI服务接口
public interface SelfProgrammingAgent {
    @SystemMessage({
        // 系统提示词定义AI的行为准则
    })
    String chat(String userMessage);
}

// 2. 构建AI服务实例
SelfProgrammingAgent agent = AiServices.builder(SelfProgrammingAgent.class)
    .chatModel(AiFactory.model)                    // 指定大语言模型
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))  // 配置对话记忆
    .tools(new CodeExecutionTool())               // 注册工具
    .build();
```

### 2. 工具注册与使用

LangChain4J通过`@Tool`注解简化了工具的注册和使用：

```java
@Tool("执行 Groovy 代码。支持 println 打印语句。返回内容包含控制台输出和脚本返回值。")
public String executeCode(String code) {
    // 工具实现逻辑
}
```

框架会自动：
- 解析方法签名和注解信息
- 生成工具描述供AI理解
- 处理参数传递和结果返回
- 管理工具的生命周期

### 3. 对话记忆管理

通过`MessageWindowChatMemory`实现对话上下文管理：

```java
.chatMemory(MessageWindowChatMemory.withMaxMessages(20))
```

这确保了：
- 保持对话的连贯性
- 控制内存使用量
- 支持长时间的交互对话

### 4. 系统消息配置

通过`@SystemMessage`注解定义AI的行为规范：

```java
@SystemMessage({
    "你是一个具备'自我编程'能力的自主 Agent。",
    "核心指令：",
    "1. 对于任何数学计算、逻辑推理或数据处理，你必须编写 Groovy 代码并调用 executeCode 执行。",
    "2. 你可以在代码中使用 println 来打印中间过程或调试信息...",
})
```

## 实际应用场景演示

让我们通过一个具体的例子来看AI Agent的实际表现：

```java
public static void main(String[] args) {
    SelfProgrammingAgent agent = AiServices.builder(SelfProgrammingAgent.class)
            .chatModel(AiFactory.model)
            .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
            .tools(new CodeExecutionTool())
            .build();

    // 让AI计算阶乘并展示其自我编程过程
    String response = agent.chat("请计算 10 到 20 的阶乘，每一步的结果都打印出来，最后给出总和。");
    System.out.println("\nAI Final Response: \n" + response);
}
```

### AI的思考过程分析

通过观察执行日志，可以看到AI经历了以下关键阶段：

#### 阶段1：发现"整数溢出"问题
```
现象：Factorial of 17 is -288522240
AI反应：意识到阶乘不可能是负数
自纠：主动引入BigInteger解决问题
```

#### 阶段2：应对"返回值null"困惑
```
现象：--- [执行结果] --- null
原因：脚本以println结尾，eval()返回null
进化：改用StringBuilder并显式返回结果
```

#### 阶段3：最终完美解决方案
```
成果：正确计算出阶乘和 2561327494111411200
特点：结合BigInteger确保精度，使用StringBuilder保证结果可见性
```

## 技术实现深度剖析

### 1. Groovy脚本引擎集成

选择Groovy作为脚本引擎的原因：
- 语法简洁，接近自然语言
- 与Java无缝集成
- 支持动态类型和闭包
- 执行性能良好

### 2. 反射调用机制

LangChain4J通过反射机制动态调用工具方法：

```java
Object result = method.invoke(instance, parameters);
```

这种设计的优势：
- 运行时动态绑定
- 支持多种参数类型
- 易于扩展新工具

### 3. 上下文隔离

每个工具调用都在独立的上下文中执行：

```java
ScriptContext context = new SimpleScriptContext();
context.setWriter(sw);
```

确保了：
- 变量作用域隔离
- 执行环境安全
- 结果可预测性

## 最佳实践建议

### 1. 工具设计原则

```java
// ✅ 推荐：功能单一，职责明确
@Tool("执行数据库查询")
public String executeQuery(String sql) { ... }

// ❌ 不推荐：功能混杂，难以维护
@Tool("执行各种操作")
public String doEverything(String operation, String param) { ... }
```

### 2. 错误处理策略

```java
// ✅ 推荐：详细的错误信息
catch (Exception e) {
    return "Execution Error: " + e.getClass().getSimpleName() + ": " + e.getMessage();
}

// ❌ 不推荐：过于简化的错误信息
catch (Exception e) {
    return "Error occurred";
}
```

### 3. 性能优化考虑

```java
// 缓存常用脚本引擎实例
private static final ScriptEngine engine = new ScriptEngineManager().getEngineByName("groovy");

// 限制执行时间和资源使用
// 设置合理的超时机制
```

## 扩展应用场景

### 1. 数据分析工具

```java
@Tool("执行数据分析代码")
public String analyzeData(String dataScript) {
    // 执行统计分析、数据清洗等操作
}
```

### 2. 文件处理工具

```java
@Tool("处理文件内容")
public String processFile(String filePath, String operation) {
    // 文件读写、格式转换等操作
}
```

### 3. 系统管理工具

```java
@Tool("执行系统命令")
public String executeCommand(String command) {
    // 安全的系统命令执行
}
```

## 总结

通过巧妙的设计，我们可以实现：

1. **增强的观察能力**：AI可以捕获并分析代码执行的中间过程
2. **自我纠错机制**：基于执行结果自动修正错误
3. **透明化决策**：完整展示AI的思考和执行过程
4. **灵活的扩展性**：易于添加新的工具和功能

这种设计模式为构建更智能、更自主的AI应用提供了很好的参考模板。开发者可以根据具体需求，在此基础上扩展更多功能，创造出更加强大的AI助手。

在实际应用中，建议重点关注安全性、性能优化和用户体验，确保AI工具既强大又可靠。