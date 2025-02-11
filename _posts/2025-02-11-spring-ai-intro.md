---
layout: post  
title: 快速入门 SpringAI
categories: [ spring, ai, tutorial, technology ]  
description: 介绍如何利用 SpringAI 框架快速构建一个多功能智能应用，包括聊天、流式输出、图片分析、情感分析和函数调用，实现 AI 能力在 Spring Boot 项目中的高效集成。  
keywords: SpringAI, Ollama, 聊天, 流式输出, 图片分析, 情感分析, 函数调用, AI, Spring Boot, 快速入门  
mermaid: false  
sequence: false  
flow: false  
mathjax: false  
mindmap: false  
mindmap2: false
---



# 快速入门 SpringAI：打造智能应用

## 📌 介绍

SpringAI 是 Spring 生态系统中新兴的 AI 集成框架，旨在帮助开发者轻松构建 AI 驱动的应用。本篇文章将基于 **SpringAI + Ollama**，带你从零搭建一个支持 **聊天、图片分析、情感分析、函数调用** 的智能应用。

## 🏗️ 环境准备

### 1️⃣ 依赖引入

在 `pom.xml` 中添加 SpringAI 相关依赖：

```plain
      <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
        </dependency>
```

### 2️⃣ 配置 AI 模型

在 `application.yml` 中配置 Ollama 模型：

```plain
spring:
  ai:
    ollama:
      base-url: http://localhost:11434  # Ollama 本地 API
      chat:
        options:
          model: MFDoom/deepseek-r1-tool-calling:8b  # 支持 function calling
```

## 🛠️ 构建 AI 控制器

### 1️⃣ 聊天功能

在 `ChatController.java` 中，我们定义了一个 `POST` 接口 `/api/ai/chat`，用于处理用户输入并返回 AI 生成的回复。

```plain
@PostMapping("/chat")
public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
    OllamaOptions options = OllamaOptions.builder()
            .temperature(request.temperature())
            .build();

    UserMessage userMessage = new UserMessage(request.message());
    ChatResponse response = chatModel.call(new Prompt(userMessage, options));

    return ResponseEntity.ok(response);
}
```

📌 **核心逻辑**：

+ 通过 `UserMessage` 封装用户输入
+ 传递 `temperature` 参数，控制 AI 生成文本的随机性
+ 调用 `chatModel.call()` 获取 AI 回复

### 2️⃣ 流式聊天

如果需要 **流式** 输出 AI 生成的回复，可以使用 `SseEmitter` 进行 Server-Sent Events 推送：

```plain
@GetMapping("/stream-chat")
public SseEmitter streamChat(@RequestParam String message) {
    SseEmitter emitter = new SseEmitter(60_000L);

    chatModel.stream(new Prompt(new UserMessage(message)))
            .subscribe(chunk -> {
                try {
                    emitter.send(SseEmitter.event().data(chunk.getResult().getOutput().getContent()));
                } catch (IOException e) {
                    emitter.completeWithError(e);
                }
            }, emitter::completeWithError, emitter::complete);

    return emitter;
}
```

📌 **流式响应** 适用于长文本生成，如代码补全、长篇文章写作等。

### 3️⃣ 图片分析

支持 **图片上传** 并进行 AI 解析：

```plain
@PostMapping(value = "/analyze-image", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<ChatResponse> analyzeImage(@RequestParam("file") MultipartFile file,
                                                 @RequestParam(required = false) String question) throws IOException {
    if (file.isEmpty() || !ALLOWED_IMAGE_TYPES.contains(file.getContentType())) {
        throw new IllegalArgumentException("无效的文件类型");
    }

    Media media = new Media(MimeTypeUtils.parseMimeType(file.getContentType()), new ByteArrayResource(file.getBytes()));
    String promptText = question != null ? question : "详细描述这张图片";
    UserMessage userMessage = new UserMessage(promptText, media);

    ChatResponse response = chatModel.call(new Prompt(userMessage,
            OllamaOptions.builder().model(OllamaModel.LLAVA).temperature(0.5).build()));

    return ResponseEntity.ok(response);
}
```

📌 **AI 分析图片并返回描述**，支持 **多种图片格式** (`jpeg/png/webp`)。

### 4️⃣ 情感分析

```plain
@GetMapping("/sentiment-analysis")
public ResponseEntity<ChatResponse> analyzeSentiment(String text) {
    String prompt = "分析以下文本的情感倾向（积极/中性/消极）：\n" + text;

    ChatResponse response = chatModel.call(new Prompt(prompt,
            OllamaOptions.builder().temperature(0.2).build()));

    return ResponseEntity.ok(response);
}
```

📌 **应用场景**：

+ 用户评论情感分析
+ 舆情监控

### 5️⃣ 函数调用（Function Calling）

**使用 AI 调用外部 API（如天气查询）：**

```plain
@GetMapping("/function_calling")
public ResponseEntity<String> getWeather(@RequestParam String location) {
    UserMessage userMessage = new UserMessage("查询 " + location + " 的天气");

    var promptOptions = OllamaOptions.builder()
            .functionCallbacks(List.of(FunctionCallback.builder()
                    .function("CurrentWeather", new MockWeatherService())
                    .description("获取指定位置的天气")
                    .inputType(MockWeatherService.Request.class)
                    .build()))
            .build();

    ChatResponse response = chatModel.call(new Prompt(userMessage, promptOptions));
    return ResponseEntity.ok(response.getResult().getOutput().getText());
}



public class MockWeatherService implements Function<MockWeatherService.Request, MockWeatherService.Response> {

    private static final Logger logger = Logger.getLogger(MockWeatherService.class.getName());

    @Override
    public Response apply(Request request) {

        logger.log(Level.INFO, "MockWeatherService.apply: request: {0}", request);

        double temperature = 0;
        if (request.location().contains("Paris")) {
            temperature = 15;
        } else if (request.location().contains("Tokyo")) {
            temperature = 10;
        } else if (request.location().contains("San Francisco")) {
            temperature = 30;
        }

        logger.log(Level.INFO, "MockWeatherService.apply: temperature: {0}", temperature);

        return new Response(temperature, 15, 20, 2, 53, 45, Unit.C);
    }

    /**
     * Temperature units.
     */
    public enum Unit {

        /**
         * Celsius.
         */
        C("metric"),
        /**
         * Fahrenheit.
         */
        F("imperial");

        /**
         * Human readable unit name.
         */
        public final String unitName;

        Unit(String text) {
            this.unitName = text;
        }

    }

    /**
     * Weather Function request.
     */
    @JsonInclude(Include.NON_NULL)
    @JsonClassDescription("Weather API request")
    public record Request(@JsonProperty(required = true,
            value = "location") @JsonPropertyDescription("The city and state e.g. San Francisco, CA") String location,
                          @JsonProperty(required = true, value = "unit") @JsonPropertyDescription("Temperature unit") Unit unit) {

    }

    /**
     * Weather Function response.
     */
    public record Response(double temp, double feels_like, double temp_min, double temp_max, int pressure, int humidity,
                           Unit unit) {

    }

    /**
     * Weather Function response with city.
     */
    @JsonInclude(JsonInclude.Include.NON_NULL)
    @JsonClassDescription("Weather API response with city")
    public record ResponseWithCity(
            @JsonProperty(required = true, value = "city") @JsonPropertyDescription("The city") String city,
            @JsonProperty(required = true, value = "temp") @JsonPropertyDescription("The temperature in Celsius") double temp,
            @JsonProperty(required = true, value = "feels_like") @JsonPropertyDescription("The feels like temperature in Celsius") double feels_like,
            @JsonProperty(required = true, value = "temp_min") @JsonPropertyDescription("The minimum temperature in Celsius") double temp_min,
            @JsonProperty(required = true, value = "temp_max") @JsonPropertyDescription("The maximum temperature in Celsius") double temp_max,
            @JsonProperty(required = true, value = "pressure") @JsonPropertyDescription("The atmospheric pressure") int pressure,
            @JsonProperty(required = true, value = "humidity") @JsonPropertyDescription("The humidity") int humidity,
            @JsonProperty(required = true, value = "unit") @JsonPropertyDescription("The temperature unit") Unit unit) {

    }


}
```

📌 **核心功能**：

+ **AI 解析用户输入** 并 **调用天气 API**
+ **MockWeatherService** 作为天气查询的模拟服务



## 6️⃣ 函数调用和结构化输出
```java
 /**
     * 获取天气信息，并以结构化方式输出
     * 此方法首先构建用户消息和提示选项，包括函数回调，以调用天气服务
     * 然后，根据天气服务的响应，再次调用模型以结构化方式处理和输出信息
     *
     * @param location 地理位置参数，用于获取指定地点的天气信息
     * @return 结构化后的天气信息列表，包含不同城市的天气详情
     */
    @GetMapping("/function_calling2")
    public ResponseEntity<Object> getWeather2(@RequestParam String location) {
        // 初始化用户消息，包含询问天气的意图
        UserMessage userMessage = new UserMessage("What's the weather like in San Francisco, Tokyo, and Paris?");

        // 构建提示选项，包括函数回调，用于获取指定地点的天气信息
        var promptOptions = OllamaOptions.builder()
                .functionCallbacks(List.of(FunctionCallback.builder()
                        .function("CurrentWeather", new MockWeatherService()) // (1) 指定函数名称和实例
                        .description("Get the weather in location") // (2) 函数描述，说明其用途
                        .inputType(MockWeatherService.Request.class) // (3) 函数输入类型，即签名
                        .build())) // 构建函数回调
                .build();

        // 调用聊天模型，传递用户消息和提示选项，获取天气信息的响应
        ChatResponse response = chatModel.call(new Prompt(userMessage, promptOptions));
        // 提取响应中的文本信息
        String text = response.getResult().getOutput().getText();

        // 对上一步得到的温度数据文字结果，再次进行大模型调用，进行格式化输出
        var outputConverter = new BeanOutputConverter<>(new ParameterizedTypeReference<List<MockWeatherService.ResponseWithCity>>() {
        });
        // 构建新的提示，用于格式化输出天气信息
        Prompt prompt = new Prompt(text,
                OllamaOptions.builder()
                        .format(outputConverter.getJsonSchemaMap())
                        .build());
        // 再次调用聊天模型，传递新的提示，以获取格式化后的天气信息
        ChatResponse response2 = chatModel.call(prompt);
        // 提取格式化后的天气信息文本
        String content = response2.getResult().getOutput().getText();

        // 将格式化后的天气信息文本转换为对象列表
        List<MockWeatherService.ResponseWithCity> converted = outputConverter.convert(content);
        // 返回转换后的天气信息列表作为响应
        return ResponseEntity.ok(converted);
    }
```

