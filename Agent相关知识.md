# RAG检索增强生成

## 介绍

RAG = Retrieval-Augmented Generation，中文叫“检索增强生成”。

其实就是：将“**用户问题**”和“**与这个问题相关的文档切片**”结合成“**增强提示词**”丢给大模型。

RAG 的思路就是把“回答”拆成两段：

1. 先**检索**：从知识库里找和用户问题最相关的文本片段。
2. 再**生成**：把“用户问题”+ “检索出来的相关文本片段”一起发给大模型，让它基于这些资料作答。

它不是“让模型更聪明”，而是“让模型回答时**更有依据**”。

- RAG 查的是你自己的、本地沉淀好的知识
- MCP 搜索查的是外部实时信息

**RAG 和记忆、联网搜索区别？**

**RAG 不是聊天记忆**

你的项目同时有 Redis 聊天记忆和 RAG：

- 聊天记忆：存最近 20 条会话消息，用于“记住刚刚聊了什么”
- RAG：存长期知识文档，用于“回答时查资料”
  - 
- `Memory` 解决上下文连续性
- `RAG` 解决知识依据

**RAG 也不是联网搜索**

所以你现在其实有三套“信息来源”：

- 对话上下文
- 本地知识库 RAG
- 外部搜索工具
  - 

区别是：

如果以后用户问“2026 年某地分数线、最新政策、最新就业数据”，那更应该走搜索工具；如果问“你的项目设定、固定咨询原则、已沉淀 FAQ”，那更适合走 RAG。

**4. 你这个项目为什么很适合 RAG**

因为你的业务不是“陪聊”，而是“给现实决策建议”。

比如高考志愿、考研、专业选择、就业建议，最怕两件事：

- 说得很自信，但没依据
- 用旧知识回答新问题
  - 

- “风格”靠 system prompt
- “事实依据”靠 RAG
- “实时性”靠 MCP 搜索
- “连续对话”靠 Redis memory
  - 

1. 你项目当前最值得做的 3 个 RAG 升级点，例如 rerank、metadata 过滤、避免重复灌库。

## 具体流程

1. 首先把自己的文档**切割成切片**

> **文档为啥要切分？**
>
> 切分不是为了存储方便，而是为了检索精度。
>
> 因为整篇文档太长，用户问一个很小的问题时，直接拿整篇去匹配，相关性会很差。切成段后，向量检索才能找到真正相关的那几段。

1. 对每个切片进行**向量转换**（通过Embedding模型）

> **为啥要把切片进行向量转换？**
>
> 机器擅长用向量比较相似度。

1. 将得到的每个切片的向量**储存到向量数据库**里

![img](https://hcndqd5h8ebu.feishu.cn/space/api/box/stream/download/asynccode/?code=ODAwNGQ2YTQ2YzFiMDVjYzdhOGIzMzMyM2M3YjdjZjVfeDg2eUpiTFNEWUlVNUVxUkRLN1RHdG1PZGFtMEpHbmFfVG9rZW46VjBGemJsd05Lb09lV3h4SERpQWNqaDBObjFlXzE3NzcxMzQwMTI6MTc3NzEzNzYxMl9WNA)

1. 用户提问，先把这个**问题变成一个向量**表示（通过Embedding模型），叫它“提问向量”。
2. 将提问向量与向量数据库里的向量进行一个余弦相似度计算，**得到相似度较高的文档切片。**
3. 相似度较高的文档切片用“Rank模型”**进行排名**
4. 把按相似度排好序的文档切片和用户问题加在一起**得到增强提示词**（增强提示词 = 按相似度排好序的文档切片+用户问题）
5. 把增强提示词给大模型
6. 得到最终答案

![img](https://hcndqd5h8ebu.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGVkMmE5NmY0MzFjNzE4MWU2MGJhZTdhNTY4YTExYWZfUnJ1eXU3RXlCWFhKNG5UT1NMbTlYMHJqMm5DT1EyVzRfVG9rZW46THVGQWJ5QTZRb2wwQUh4aVpQRmNodHRBbjJnXzE3NzcxMzQwMTI6MTc3NzEzNzYxMl9WNA)

## 解决的问题

它本质上是在补大模型的两个短板：

- 补“记忆边界”：大模型可能知道很多互联网上的常识，但是不知道私有文档，也不知道实时数据
  - `知识过时`：模型参数里的知识不是**实时更新**的。比如你问它高考分数线，他可能回答你去年的。
  - `缺少私有知识`：比如公司内部文档 ，学校发的文件，模型训练时根本没见过，所以回答不上来。
- 补“事实依据”：大模型有时候会瞎说，瞎编
  - `幻觉`：明明不知道，也能**编得**像真的。比如你问它相关论文，它可能随便编一个论文名字。
  - `事实不可追溯`：回答出来了，但你不知道它依据哪段资料。

## **与项目相关**

**我项目里的 RAG 记成一句话**

我的项目不是让大模型直接回答升学和职业问题，而是先去你维护的知识文档里找依据，再让 Qwen 用“张雪峰风格”把依据组织成回答。

### 文本切割策略

- 每段最多 300 字符
- 段间重叠 100 字符

```Java
@Bean
public EmbeddingStoreIngestor embeddingStoreIngestor() {
    // 按段落切分文档：每段最多 300 字符，段落间允许 100 字符重叠。
    DocumentByParagraphSplitter paragraphSplitter = new DocumentByParagraphSplitter(300, 100);

    return EmbeddingStoreIngestor.builder()
            .documentSplitter(paragraphSplitter)
            // 在每段文本前加上文件名，提升检索结果的可解释性。
            .textSegmentTransformer(textSegment -> TextSegment.from(
                    textSegment.metadata().getString("file_name") + "\n" + textSegment.text(),
                    textSegment.metadata()
            ))
            .embeddingModel(embeddingModel)
            .embeddingStore(embeddingStore)
            .build();
}
```

### 设置overlap的意义

！！！！重叠的意义是避免一句话刚好被切断，导致上下文丢失。

例如：

 原文：除非那个211学校所在的城市特别好，否则能报985就报985.

  切割文本1: 除非那个211学校所在的城市特别好，否则

  切割文本2: 能报985就报985

 如果用户提问：分数同时够985和211的话，是一定要去985吗？

  进行相似度检测的时候，可能命中的就是“切割文本2”：“能报985就报985”

  那么将文本2和问题丢给大模型之后得到的答案可能就是：“一定要去985”。丢失了上下文中的限制条件。

### **向量转换和存储**

> 原始文本是“人能看懂”的语义
>
> embedding 是把文本压成“机器擅长比较相似度”的向量
>
> pgvector 负责存这些向量，并支持相似度检索

#### 向量转换模型

embedding 模型来自 DashScope 的 `text-embedding-v4`

```YAML
# LangChain4j 模型配置。
langchain4j:
  community:
    dashscope:
      chat-model:
        # 普通同步聊天所用模型。
        model-name: qwen-plus
        api-key: ${DASHSCOPE_API_KEY:replace-with-your-dashscope-api-key}
      streaming-chat-model:
        # 流式聊天所用模型。
        model-name: qwen-plus
        api-key: ${DASHSCOPE_API_KEY:replace-with-your-dashscope-api-key}
      embedding-model:
        # 文本向量化模型，供 RAG 建库与检索时使用。
        model-name: text-embedding-v4
        api-key: ${DASHSCOPE_API_KEY:replace-with-your-dashscope-api-key}
```

#### 向量库存储

用的是 `pgvector`,维度是 `1024`

```Java
@Bean
public EmbeddingStore<TextSegment> initEmbeddingStore() {
    // 创建 pgvector 存储对象。
    // 这里设置了 createTable(true)，首次启动时会自动建表。
    // dropTableFirst(true) 表示每次启动会先删旧表再重建，适合演示，不适合正式长期积累数据。
    return PgVectorEmbeddingStore.builder()
            .table(table)
            .dropTableFirst(true)
            .createTable(true)
            .host(host)
            .port(port)
            .user(user)
            .password(password)
            .dimension(1024)
            .database(database)
            .build();
}
```

在入库前还做了一个挺实用的小增强：把 `file_name` 拼到 segment 文本前面

```Java
@Bean
public EmbeddingStoreIngestor embeddingStoreIngestor() {
    // 按段落切分文档：每段最多 300 字符，段落间允许 100 字符重叠。
    DocumentByParagraphSplitter paragraphSplitter = new DocumentByParagraphSplitter(300, 100);

    return EmbeddingStoreIngestor.builder()
            .documentSplitter(paragraphSplitter)
            // 在每段文本前加上文件名，提升检索结果的可解释性。
            .textSegmentTransformer(textSegment -> TextSegment.from(
                    textSegment.metadata().getString("file_name") + "\n" + textSegment.text(),
                    textSegment.metadata()
            ))
            .embeddingModel(embeddingModel)
            .embeddingStore(embeddingStore)
            .build();
}
```

好处：检索命中后更容易知道内容来自哪个文件

### **用户提问后的检索**

当前策略是：

- 把用户问题也转成向量
- 去 pgvector 里找最相近的片段
- 最多取 `5` 条
- 只保留分数 `>= 0.75` 的结果

这就是最基础的向量召回。

```Java
@Bean
public ContentRetriever contentRetriever() {
    // 这里定义问答时的向量检索策略：
    // 1. 先把用户问题向量化
    // 2. 在向量库里找最相近的片段
    // 3. 只保留分数足够高的结果，避免把噪声传给模型
    ContentRetriever contentRetriever = EmbeddingStoreContentRetriever.builder()
            .embeddingStore(embeddingStore)
            .embeddingModel(embeddingModel)
            .maxResults(5)
            .minScore(0.75)
            .build();

    // 这里还没有接入 rerank 或更复杂的召回策略，先返回基础检索器。
    return contentRetriever;
}
```

我现在是“基础版 RAG”，不是“高级检索链路”。

### **生成增强提示词，交给**大模型

- `.contentRetriever(contentRetriever)` 把检索器接进了对话链路
- `.chatModel(chatModel)` 和 `.streamingChatModel(streamingChatModel)` 负责真正生成答案

```TypeScript
@Bean
public AiChat aiChat() {
    // AiServices.builder 会根据 AiChat 接口定义，动态生成实现类。
    return AiServices.builder(AiChat.class)
            // 普通问答走这个模型。
            .chatModel(chatModel)
            // 流式问答走这个模型。
            .streamingChatModel(streamingChatModel)
            // 对话前先进行知识检索，把命中的片段一起喂给模型。
            .contentRetriever(contentRetriever)
            // 用 sessionId 作为 memoryId，把最近 20 条消息持久化在 Redis 中。
            .chatMemoryProvider(memoryId -> MessageWindowChatMemory
                    .builder()
                    .id(memoryId)
                    .chatMemoryStore(redisChatMemoryStore)
                    .maxMessages(20)
                    .build())
            // 注册本地 Java 工具，模型在需要时可以自动调用。
            .tools(new TimeTool(), ragTool, emailTool)
            // 注册外部 MCP 工具，例如联网搜索。
            .toolProvider(mcpToolProvider)
            .build();
}
```

系统提示词进一步限制回答边界

```TypeScript
@InputGuardrails({SafeInputGuardrail.class})
public interface AiChat {
    /**
     * 普通同步对话接口。
     *
     * @param sessionId 会话 id，同时也会作为记忆的 key
     * @param prompt    用户输入
     * @return 模型返回的完整文本
     */
    @SystemMessage(fromResource = "system-prompt/chat-bot.txt")
    String chat(@MemoryId Long sessionId, @UserMessage String prompt);


    /**
     * 流式对话接口。
     *
     * @param sessionId 会话 id，同时也会作为记忆的 key
     * @param prompt    用户输入
     * @return 按片段持续输出的文本流
     */
    @SystemMessage(fromResource = "system-prompt/chat-bot.txt")
    Flux<String> streamChat(@MemoryId Long sessionId, @UserMessage String prompt);
}
```

所以你项目里的完整执行路径，可以理解成：

```
前端请求 /chat -> aiChat.chat(...) -> LangChain4j 先做检索 -> 把检索结果和用户问题一起给 Qwen -> 生成答案
@PostMapping("/chat")
public String chat(@RequestBody ChatRequest chatRequest) {
    // 在模型调用前，把用户和会话信息放进线程上下文，便于监控模块打点。
    MonitorContextHolder.setContext(MonitorContext.builder()
            .userId(chatRequest.getUserId())
            .sessionId(chatRequest.getSessionId())
            .build());

    // 交给 LangChain4j 代理执行一次普通对话。
    String chat = aiChat.chat(chatRequest.getSessionId(), chatRequest.getPrompt());

    // 请求结束后手动清理上下文，避免线程复用时串数据。
    MonitorContextHolder.clearContext();
    return chat;
}
```

# 对话记忆

## 介绍

记忆不是“模型真的把你永久记住了”。

更准确地说，记忆是：**系统把过去的信息存下来，并在下一轮推理时重新取出来，再喂给****大模型****。**

所以记忆的本质只有两步：

1. **存（用Redis）**
2. **取出来再利用**

一句话：

> 模型负责生成，应用系统负责帮模型“记住历史”。  

## 记忆分层

### 1. 瞬时记忆

- 只存在于**当前这一轮调用**
- 本质就是这次 prompt 里带给模型看的内容
- 包括：
  - 系统提示词
  - 当前用户问题
  - 最近几轮历史消息
  - RAG 检索结果
  - 工具返回结果

一句话：

> 这一轮 prompt 里带了什么，模型这次就“记得”什么。

### 2. 短期记忆

- 作用：维护**当前会话**的上下文
- 常见实现：`Session + Redis + 滑动窗口`
- 典型用途：
  - 记住用户前面刚说过的话
  - 让多轮对话连起来

例如：

- 第一轮：我是河南考生，560 分，理科
- 第二轮：我适合计算机还是自动化？
  - 

如果没有短期记忆，第二轮模型只看到“计算机还是自动化”，上下文是不够的。

### 3. 长期记忆

- 作用：跨很多轮对话，记住一些**可语义召回**的内容
- 常见实现：`Vector DB`
- 适合存：
  - 用户偏好
  - 历史重要结论
  - 专业知识片段
  - 经验型问答
    - 

一句话：

> 长期记忆更像“语义资料库”，不是按主键精确查，而是按语义召回。

### 4. 跨会话持久化

- 作用：把关键事实**结构化长期保存**
- 常见实现：`MySQL / PostgreSQL / NoSQL`
- 适合存：
  - 用户画像
  - 任务进度
  - 目标院校
  - 分数、省份、选科
  - 明确偏好和限制条件
    - 

一句话：

> 这层强调的是“准”和“可更新”，不是“语义模糊匹配”。

## 它们的区别

- **瞬时记忆**：只活在本轮 prompt 里
- **短期记忆**：服务当前会话
- **长期记忆**：面向语义召回
- **跨会话持久化**：面向结构化事实保存
  - 

## 与项目相关

### 项目里现在已经做好的记忆

我项目里现在真正落地的，主要是：**短期记忆**

实现方式：

- 前端传 `sessionId`
- LangChain4j 用 `@MemoryId` 把它当作记忆 key
- 用 `MessageWindowChatMemory` 做滑动窗口
- 用 `RedisChatMemoryStore` 把历史消息存到 Redis
- 每次新对话时，再把最近 20 条消息带回当前 prompt
  - 

一句话：

> 我项目里的“记忆”，本质上不是模型自己记住了用户，而是系统把会话历史重新喂给模型。

### 记忆是怎么接进对话链路的

```Java
@Bean
public AiChat aiChat() {
    // AiServices.builder 会根据 AiChat 接口定义，动态生成实现类。
    return AiServices.builder(AiChat.class)
            // 普通问答走这个模型。
            .chatModel(chatModel)
            // 流式问答走这个模型。
            .streamingChatModel(streamingChatModel)
            // 对话前先进行知识检索，把命中的片段一起喂给模型。
            .contentRetriever(contentRetriever)
            // 用 sessionId 作为 memoryId，把最近 20 条消息持久化在 Redis 中。
            .chatMemoryProvider(memoryId -> MessageWindowChatMemory
                    .builder()
                    .id(memoryId)
                    .chatMemoryStore(redisChatMemoryStore)
                    .maxMessages(20)
                    .build())
            // 注册本地 Java 工具，模型在需要时可以自动调用。
            .tools(new TimeTool(), ragTool, emailTool)
            // 注册外部 MCP 工具，例如联网搜索。
            .toolProvider(mcpToolProvider)
            .build();
}
```

解释：

- `.chatMemoryProvider(...)`：把记忆机制接入对话链路
- `MessageWindowChatMemory`：说明现在是窗口式短期记忆
- `.id(memoryId)`：同一个会话 id 对应同一段历史
- `.chatMemoryStore(redisChatMemoryStore)`：历史消息放在 Redis
- `.maxMessages(20)`：只取最近 20 条，不是无限长

### memoryId 是什么

```Java
@InputGuardrails({SafeInputGuardrail.class})
public interface AiChat {
    /**
     * 普通同步对话接口。
     *
     * @param sessionId 会话 id，同时也会作为记忆的 key
     * @param prompt    用户输入
     * @return 模型返回的完整文本
     */
    @SystemMessage(fromResource = "system-prompt/chat-bot.txt")
    String chat(@MemoryId Long sessionId, @UserMessage String prompt);
    /**
     * 流式对话接口。
     *
     * @param sessionId 会话 id，同时也会作为记忆的 key
     * @param prompt    用户输入
     * @return 按片段持续输出的文本流
     */
    @SystemMessage(fromResource = "system-prompt/chat-bot.txt")
    Flux<String> streamChat(@MemoryId Long sessionId, @UserMessage String prompt);
}
```

这里的 `@MemoryId` 表示：

> `sessionId` 不只是普通参数，它还是“去哪里找历史消息”的 key。

### sessionId 哪来

```Java
@PostMapping("/chat")
public String chat(@RequestBody ChatRequest chatRequest) {
    // 在模型调用前，把用户和会话信息放进线程上下文，便于监控模块打点。
    MonitorContextHolder.setContext(MonitorContext.builder().userId(chatRequest.getUserId()).sessionId(chatRequest.getSessionId()).build());

    // 交给 LangChain4j 代理执行一次普通对话。
    String chat = aiChat.chat(chatRequest.getSessionId(), chatRequest.getPrompt());

    // 请求结束后手动清理上下文，避免线程复用时串数据。
    MonitorContextHolder.clearContext();
    return chat;
}
```

所以链路是：

1. 前端传 `sessionId`
2. Controller 调 `aiChat.chat(sessionId, prompt)`
3. LangChain4j 看到 `@MemoryId`
4. 再去 Redis 取这段会话的历史记录
5. 把历史记录一起放进当前轮上下文

### 历史消息存在哪里

```Java
@Bean
public RedisChatMemoryStore redisChatMemoryStore() {
    // 创建 Redis 版聊天记忆存储器，后续 MessageWindowChatMemory 会用它读写历史消息。
    return RedisChatMemoryStore.builder()
            .host(host)
            .port(port)
            .password(password)
            .ttl(ttl)
            .user("default")
            .build();
}
```

说明：

- 历史消息存在 `Redis`
- 不是只存在 JVM 内存里
- 所以应用短时间重启后，记忆还在

Redis 过期时间配置

```YAML
spring:
  data:
    redis:
      # 本地 Redis 连接参数。
      # 默认值适合演示，真实密码建议放到 application-local.yml 或环境变量里。
      host: ${DEV_REDIS_HOST:localhost}
      password: ${DEV_REDIS_PASSWORD:replace-with-your-dev-redis-password}
      port: ${DEV_REDIS_PORT:6380}
      ttl: 3600
      timeout: 30000
```

- 当前会话记忆默认保留 3600 秒
- 这是短期会话记忆，不是永久记忆

## 项目里“记忆”在运行时发生了什么

一次请求进入后，大致是这样：

1. 用户发来 `prompt`
2. 系统拿到 `sessionId`
3. 去 Redis 找到这个 `sessionId` 对应的最近历史消息
4. 再把这些历史消息和系统提示词、当前问题、RAG 检索结果一起组成当前轮上下文
5. 最后交给大模型生成答案
   1. 

一句话：

> 你看到的“模型记得前面聊过什么”，其实是因为系统把历史消息重新塞回去了。

## 我项目里长期记忆做到哪了

我项目现在有 **向量库能力**，但主要用于 **知识库 RAG**，还不是完整的“用户长期记忆”。

例如现在已经有：

- 文档切片
- embedding
- pgvector
- 语义检索
  - 

但存进去的主要是：

- 项目知识文档
- 问答知识片段
  - 

还没有系统化做成：

- 用户长期偏好记忆
- 跨天的用户画像记忆
- 历史决策结论记忆
  - 

所以更准确地说：

> 我项目现在已经实现了短期记忆；长期记忆的技术底座有了，但主要在做 RAG，不是完整的用户长期记忆系统。

## 最后总结

记忆本质上不是模型自己永久记住了什么，而是系统把过去的信息分层保存，并在下一轮推理时重新取出来。

在我的项目里：

- **瞬时记忆**：本轮 prompt 里的内容
- **短期记忆**：`sessionId + Redis + MessageWindowChatMemory`
- **长期记忆**：目前更多体现在知识库 RAG
- **跨会话持久化**：还没有完整展开做
  - 

一句最适合背的总结：

> 我项目里的记忆，本质上是 LangChain4j 通过 `@MemoryId` 把 `sessionId` 当成会话 key，再从 Redis 中取出最近 20 条历史消息，连同系统提示词、当前问题和 RAG 检索结果一起交给大模型，所以模型看起来像“记得前面聊过什么”。

# 方法调用

## 介绍

Function Calling，本质上就是：

让大语言模型在需要的时候，不是直接靠嘴回答，而是**先去调用工具，再根据工具结果生成最终答案。**

比如：

- 查当前时间
- 发邮件
- 写入知识库
- 联网搜索
  - 

它解决的核心问题是：

- 大语言模型不知道**实时信息**
- 大语言模型不擅长**精确计算**
- 大语言模型不能直接操作外部系统
- 大语言模型只靠内部知识，容易瞎编
  - 

一句话：

> Function Calling 不是让大语言模型更会说，而是让大语言模型“会做事”。  

## 工作流程

整个流程可以拆成 5 步：

1. 先把可用工具**注册给大语言模型**
2. 把工具的**名称、描述、参数要求**，一起告诉大语言模型
3. 用户提问后，大语言模型判断这次是否需要调用工具
4. 如果需要，就输出一个结构化的“调用哪个工具、传什么参数”
5. 我的框架执行工具，把返回结果再交回给大模型，生成最终答案
   1. 

你可以把它理解成：

> 先让大语言模型做“决策”，再让工具做“执行”，最后再让大语言模型做“表达”。  

## 具体例子

![img](https://hcndqd5h8ebu.feishu.cn/space/api/box/stream/download/asynccode/?code=MzMwMGE5OTI2ZWMwMGJlNmQ1NjBmYjkwODVkZTBjYjdfTGJ4YTFtNTNkWGZxV01kQkRWWWpZdmxpREE1b3BYUjlfVG9rZW46UGY4NGJIZFM5bzhkTXB4R28yNGN1V285blZkXzE3NzcxMzQwMTI6MTc3NzEzNzYxMl9WNA)

比如用户问：

```Plain
你知道现在是北京时间几点？
```

这时候大语言模型如果只靠自己猜，就不靠谱。

正确做法是：

1. 系统先把“当前有一个查时间的工具”告诉大语言模型
2. 大语言模型判断：这个问题不能瞎答，应该调工具
3. 大语言模型输出结构化调用指令，比如：

```JSON
{
  "tool_calls": [
    {
      "name": "getCurrentTime",
      "arguments": {}
    }
  ]
}
```

1. 框架执行这个工具
2. 工具返回：

```Plain
2025-11-18 15:30:45 星期二 (中国标准时间)
```

1. 再把这个结果交给大语言模型
2. 大语言模型最终回答：

```Plain
现在是北京时间 2025 年 11 月 18 日 15 点 30 分，星期二。
```

所以你那张时序图，本质上讲的就是一句话：

> 大语言模型先决定要不要调用工具，工具执行完后，大语言模型再根据工具结果组织自然语言答案。  

## 关键技术点

### 1. 工具注册

工具必须先注册给大语言模型，大语言模型才知道“自己能调用什么”。

在 LangChain4j 里，最常见的方式就是用 `@Tool` 注解声明方法。

例如你项目里的时间工具：

代码来源：`src/main/java/com/miles/zhangxuefengagent/tool/TimeTool.java`

```Java
public class TimeTool {

    @Tool("getCurrentTime")
    public String getCurrentTimeInShanghai() {
        // 固定按中国标准时间输出，保证回复里的时间来源一致。
        ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
        return now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss EEEE (中国标准时间)"));
    }
}
```

这里的 `@Tool("getCurrentTime")` 就是在告诉框架：

- 这是一个可被大语言模型调用的方法
- 工具名叫 `getCurrentTime`
  - 

### 2. Prompt 注入

工具注册完之后，框架不会真的把 Java 方法直接丢给大语言模型。

它会把工具信息转成一份模型能理解的“工具说明”，本质上就是：

- 工具名
- 工具描述
- 参数名
- 参数类型
- 参数约束

这些内容会随着 prompt 一起发给大语言模型。

所以大语言模型并不是“突然会调 Java 方法了”，而是：

> 框架先把工具规则翻译成大语言模型能看懂的 schema，再让大语言模型决定是否调用。  

### 3. 结构化输出

大语言模型不能只说一句“我要调用时间工具”。

它必须按框架要求，输出结构化调用结果。

例如：

```JSON
{
  "tool_calls": [
    {
      "name": "getCurrentTime",
      "arguments": {}
    }
  ]
}
```

框架解析这段结构化内容后，才知道：

- 该调哪个工具
- 参数是什么
  - 

### 4. 工具执行

结构化结果出来之后，就进入真正执行阶段。

例如时间工具会真正去取系统时间，而不是模型自己脑补时间。

项目里的时间工具代码是：

代码来源：`src/main/java/com/miles/zhangxuefengagent/tool/TimeTool.java`

```Java
@Tool("getCurrentTime")
public String getCurrentTimeInShanghai() {
    ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
    return now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss EEEE (中国标准时间)"));
}
```

这说明：

- 大语言模型只负责决定要不要调
- 真正执行获取时间的是 Java 代码
  - 

### 5. 结果融合

工具执行完，不是直接把原始结果返回给用户就结束了。

通常还要把工具结果重新拼进上下文，再让大语言模型生成最终回答。

所以最后给用户看到的不是冷冰冰的工具返回值，而是：

- 更自然
- 更符合角色设定
- 更符合上下文语气
  - 

一句话：

> 工具负责拿结果，大语言模型负责把结果说成人话。  

## 与项目相关

### 项目里的 Function Calling 是怎么接进去的

项目里，工具是在 `AiChatService` 里统一注册进 Agent 的。

代码来源：`src/main/java/com/miles/zhangxuefengagent/ai/AiChatService.java`

```Java
@Bean
public AiChat aiChat() {
    // AiServices.builder 会根据 AiChat 接口定义，动态生成实现类。
    return AiServices.builder(AiChat.class)
            // 普通问答走这个模型。
            .chatModel(chatModel)
            // 流式问答走这个模型。
            .streamingChatModel(streamingChatModel)
            // 对话前先进行知识检索，把命中的片段一起喂给模型。
            .contentRetriever(contentRetriever)
            // 用 sessionId 作为 memoryId，把最近 20 条消息持久化在 Redis 中。
            .chatMemoryProvider(memoryId -> MessageWindowChatMemory
                    .builder()
                    .id(memoryId)
                    .chatMemoryStore(redisChatMemoryStore)
                    .maxMessages(20)
                    .build())
            // 注册本地 Java 工具，模型在需要时可以自动调用。
            .tools(new TimeTool(), ragTool, emailTool)
            // 注册外部 MCP 工具，例如联网搜索。
            .toolProvider(mcpToolProvider)
            .build();
}
```

解释：

- `.tools(...)`：注册本地 Java 工具
- `.toolProvider(...)`：注册外部 MCP 工具
  - 

也就是说，你项目里的 Function Calling 不只有一种来源。

### 项目里的本地工具

#### 1. 时间工具

代码来源：`src/main/java/com/miles/zhangxuefengagent/tool/TimeTool.java`

```Java
public class TimeTool {

    @Tool("getCurrentTime")
    public String getCurrentTimeInShanghai() {
        ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
        return now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss EEEE (中国标准时间)"));
    }
}
```

作用：

- 当用户问当前时间时，模型可以调用它
- 避免模型自己瞎猜时间
  - 

#### 2. 知识写入工具

代码来源：`src/main/java/com/miles/zhangxuefengagent/tool/RagTool.java`

```Java
@Tool("当用户想要保存问答对、知识点或者向知识库添加新信息时调用此工具。将问题、答案和目标文件名作为参数。")
public String addKnowledgeToRag(String question, String answer, String fileName) {
    log.info("Tool 调用: 正在保存知识 - Q: {}, file: {}", question, fileName);

    String formattedContent = String.format("### Q：%s\n\nA：%s", question, answer);

    if (fileName == null || fileName.isBlank()) {
        fileName = "zhangxuefengagent.md";
    }
    if (!fileName.endsWith(".md")) {
        fileName = fileName + ".md";
    }

    boolean writeSuccess = appendToFile(formattedContent, fileName);
    if (!writeSuccess) {
        return "保存失败：无法写入本地文件系统，请检查日志。";
    }

    try {
        Metadata metadata = Metadata.from("file_name", fileName);
        Document document = Document.from(formattedContent, metadata);
        embeddingStoreIngestor.ingest(document);

        log.info("Tool 执行成功: 知识已同步至 RAG");
        return "成功！已将该知识点保存到文档 [" + fileName + "] 并同步至向量数据库。";
    } catch (Exception e) {
        log.error("RAG - 向量化失败", e);
        return "文件写入成功，但向量数据库更新失败：" + e.getMessage();
    }
}
```

作用：

- 用户想保存知识时，模型可以自动调这个工具
- 先写文件
- 再写入向量库
  - 

这就是一个很典型的 Function Calling 场景：

> 模型不直接“假装保存成功”，而是真的调用工具去落盘和入库。  

#### 3. 邮件工具

代码来源：`src/main/java/com/miles/zhangxuefengagent/tool/EmailTool.java`

```Java
@Tool("向特定用户发送电子邮件。")
public String sendEmail(String targetEmail, String subject, String content) {
    try {
        log.info("Tool 调用: 正在发送邮件 -> To: {}, Subject: {}", targetEmail, subject);
        
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(fromEmail);
        message.setTo(targetEmail);
        message.setSubject(subject);
        message.setText(content);

        mailSender.send(message);
        
        log.info("邮件发送成功");
        return "邮件已成功发送给 " + targetEmail;
    } catch (Exception e) {
        log.error("邮件发送失败", e);
        return "邮件发送失败: " + e.getMessage();
    }
}
```

作用：

- 用户明确要求发邮件时，模型可以自动调用它
- 真正去发邮件，而不是只生成一段“已发送”的文字
  - 

### 项目里的外部工具

你项目除了本地 Java 工具，还接了 MCP 外部工具。

代码来源：`src/main/java/com/miles/zhangxuefengagent/config/McpToolConfig.java`

```Java
@Bean
public McpToolProvider mcpToolProvider() {
    // 创建一个基于 HTTP SSE 的 MCP 传输层，当前接的是 web_search 工具。
    McpTransport searchTransport = new HttpMcpTransport.Builder()
            .sseUrl("https://open.bigmodel.cn/api/mcp/web_search/sse?Authorization=" + apiKey.trim())
            .build();

    // 把传输层包装成 MCP Client，LangChain4j 会通过它发现并调用远程工具。
    McpClient searchClient = new DefaultMcpClient.Builder()
            .key("BigModelSearchMcpClient")
            .transport(searchTransport)
            .build();

    // 当前只启用了搜索工具客户端。
    return McpToolProvider.builder()
            .mcpClients(searchClient)
            .build();
}
```

这说明你的项目里，模型不仅能调本地工具，还能调远程搜索工具。

一句话：

> 本地工具解决“执行本地能力”，MCP 工具解决“接外部服务能力”。  

## 兜底方案

Function Calling 最大的问题，不是“调不调工具”，而是：

> 工具一旦失败，系统怎么优雅收场。  

### 1. 参数无效

比如：

- 邮箱格式不对
- 文件名不合法
- 日期参数缺失
  - 

兜底思路：

- 在工具内部先做参数校验
- 参数不合法时，不要让底层异常直接暴露给用户
- 返回可理解的错误提示，或者让模型引导用户重新输入
  - 

例如你项目里的 `RagTool`，其实已经有一点这种兜底思路：

```Java
if (fileName == null || fileName.isBlank()) {
    fileName = "zhangxuefengagent.md";
}
if (!fileName.endsWith(".md")) {
    fileName = fileName + ".md";
}
```

这就是：

- 参数不完整时，给默认值
- 参数格式不标准时，自动修正
  - 

### 2. 业务数据不存在

比如：

- 要查的记录不存在

- 搜索结果为空

- 指定知识文件不存在

  

兜底思路：

- 不要直接报空指针、数据库异常

- 返回语义化结果给模型

- 再由模型生成自然语言反馈

  

例如可以返回：

```Plain
未找到对应记录，请确认条件是否正确。
```

而不是直接把内部错误栈甩给用户。

### 3. 服务异常或超时

比如：

- 邮件服务挂了

- 外部搜索超时

- 写入向量库失败

  

你项目里已经有一部分这种处理方式。

例如邮件工具：

```Java
catch (Exception e) {
    log.error("邮件发送失败", e);
    return "邮件发送失败: " + e.getMessage();
}
```

再比如知识写入工具：

```Java
catch (Exception e) {
    log.error("RAG - 向量化失败", e);
    return "文件写入成功，但向量数据库更新失败：" + e.getMessage();
}
```

这种思路的好处是：

- 工具失败不会把整个 Agent 调用直接打崩
- 能把失败转换成可理解的话术
  - 

不过如果要更稳，正式项目里最好不要直接把 `e.getMessage()` 暴露给用户。

更稳妥的写法是：

- 日志里记详细异常
- 返回给用户统一、温和、可操作的话术

例如：

```Plain
当前服务繁忙，请稍后再试。
```

### 4. 权限不足

如果某些工具涉及用户隐私、敏感数据、管理员能力，就不能只要模型想调就调。

兜底思路：

- 工具层做权限判断
- 权限不足时直接拒绝
- 不暴露内部权限规则和系统细节
  - 

例如统一返回：

```Plain
您无权执行该操作。
```

### 5. 工具不可用

如果某个外部服务连续失败，比如：

- 搜索接口挂了
- 邮件服务不可用
- 第三方 API 大面积超时
  - 

更稳的做法是：

- 接熔断器

- 连续失败后临时禁用该工具

- 走降级逻辑

- 同时告警

  

这属于更完整的工程化兜底。

## 最后总结

Function Calling 的本质是：

> 模型负责判断要不要调工具，工具负责真正执行，模型再根据工具结果生成最终答案。  

在我项目里，它已经落地成了两类能力：

- 本地 Java 工具：时间、知识写入、发邮件
- 外部 MCP 工具：联网搜索



它带来的最大价值是：

- 提高回答准确性
- 扩展模型能力边界
- 让模型从“只会说”变成“能执行”
  - 

一句最适合背的总结：

> 我项目里的 Function Calling，本质上是 LangChain4j 先把 `@Tool` 标注的方法和 MCP 工具注册给模型，框架再把工具 schema 注入 prompt。用户提问后，模型如果判断需要工具，就会输出结构化调用信息，框架执行工具并拿到结果，再把结果拼回上下文，最后由模型生成自然语言答案。如果工具失败，则通过参数校验、语义化返回、异常捕获、权限控制和熔断降级来做兜底。  
