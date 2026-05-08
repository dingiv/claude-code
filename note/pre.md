# Claude Code


## 四层架构
+ UI
+ agentic 循环
+ model request / tool calling
+ 基建


## agentic 循环

对比普通的一次性大模型请求

1. 准备提示词
   1. 系统提示词
   2. 历史上下文
   3. 用户提示词
   4. RAG 召回（可选）
2. 发请求
   + 基于 Http SSE 的 POST 请求
   + 服务端返回的是 SSE chunk，每个 chunk 是一段纯文本，以一个空行 `\n\n` 分隔
   + 
3. 反向监听，等待多个 chunk，检查 chunk 类型，根据不同的 chunk 类型来调用相应的处理函数
   
   + text
   + tool calling
   + reasoning

## model request

不同的厂商，其 API 不同

agent 循环在前后端的位置，有的在前端，有的在后端。

关键在于上下文管理。