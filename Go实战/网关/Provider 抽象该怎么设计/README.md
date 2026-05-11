# Provider 抽象该怎么设计

---

这是 `AI 网关专题` 的第 3 篇。

前面两篇，我们已经把 AI 网关里最先要立住的两件事讲清楚了：

- 为什么要先把模型调用收口成统一入口
- 为什么流式输出也要做成统一能力

接下来就会自然走到一个更核心的问题：

> **不同模型厂商，到底该怎么接进同一套网关。**

很多团队刚开始会直接写成：

```go
if provider == "openai" {
	return openaiClient.Chat(ctx, req)
}

if provider == "qwen" {
	return qwenClient.Chat(ctx, req)
}

if provider == "deepseek" {
	return deepseekClient.Chat(ctx, req)
}
```

短期当然能跑。

但只要模型厂商一多，问题马上就会开始堆：

- 同步接口一套判断
- 流式接口再来一套判断
- embedding 又来一套判断
- 各家字段名、错误结构、usage 格式都不一样

最后你会发现，所谓“AI 网关”，很可能只是：

> **把厂商差异从业务层挪到了另一个更集中的 if-else 文件里。**

这篇文章，我们就专门讲这一层：

> **Provider 抽象，到底应该怎么设计，才能既不虚，也不乱。**

这篇你会拿到 3 个结论：

- Provider 抽象真正要隔离的变化到底是什么
- 第一版接口为什么只建议先覆盖 chat、stream、embedding
- Provider、Router、Gateway 三层边界应该怎么分

---

## 01 Provider 抽象的目标，不是“优雅”，而是“隔离变化”

先把目标讲清楚。

Provider 抽象最重要的价值，不是让代码看起来更像设计模式。

真正的价值是：

> **把模型厂商变化，隔离在一层可控边界里。**

这层变化通常来自五个地方：

### 1. 请求结构不同

同样是聊天接口，不同厂商在这些字段上往往都不一样：

- messages 格式
- temperature 名字或范围
- max tokens 参数名
- system prompt 位置
- 工具调用字段

### 2. 响应结构不同

有的直接返回完整文本。
有的返回 choices。
有的 usage 单独挂一层。
有的会把 reasoning 内容单独拆出来。

### 3. 流式协议不同

这往往是最麻烦的部分。

- chunk 粒度不同
- 结束信号不同
- usage 返回时机不同
- 错误 chunk 结构不同

### 4. 错误语义不同

同样是限流：

- 有的返回 HTTP 429
- 有的返回业务 code
- 有的 message 里才带可识别信息

### 5. 能力边界不同

不是每家 provider 都同时支持：

- chat
- stream chat
- embedding
- function call
- structured output

所以 Provider 抽象的真正目的，就是把这些变化收进去。

---

## 02 很多团队一上来就会踩的两个坑

### 错误一：抽象得太早、太大

有些人一开始就想设计一整套“万能 Provider 框架”：

- 二十多个接口
- 十几层 option
- 一堆 hook
- 还没接一家模型，就先抽象半年

这通常会导致两件事：

- 抽象脱离真实差异
- 接第一家 provider 就发现接口不贴地

### 错误二：完全不抽象，只写分支判断

另一种极端则是：

> 能跑就行，先堆 if-else。

短期快，但后面会遇到：

- 同步、流式、embedding 重复适配
- 错误码映射分散
- usage 提取逻辑重复
- 路由层直接依赖厂商细节

所以更合适的做法是：

> **先围绕真实场景，抽出一层刚好够用的 Provider 接口。**

---

## 03 第一版先把哪三类能力接住，就够了

如果你现在是在做第一版 AI 网关，我会建议 Provider 接口先只围绕三类能力展开：

1. `Chat`
2. `StreamChat`
3. `Embedding`

为什么是这三类？

因为它们最常见，也最容易形成稳定边界。

反过来，一些能力不建议一上来就硬塞进去：

- agent orchestration
- business prompt assembly
- route policy
- audit policy

这些都不属于 Provider 本身的职责。

Provider 只做一件事：

> **把统一请求转换成厂商请求，再把厂商响应转换回统一响应。**

---

## 04 一版够用的 Provider 接口，长什么样就够了

先看一版比较稳的接口形状：

```go
type Provider interface {
	Name() string
	Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
	StreamChat(ctx context.Context, req *ChatRequest, writer StreamWriter) error
	Embedding(ctx context.Context, req *EmbeddingRequest) (*EmbeddingResponse, error)
}
```

这版接口看起来很朴素，但它的边界非常清楚。

### `Name()`

返回 provider 名称，方便日志、路由和审计使用。

### `Chat()`

负责同步问答。

### `StreamChat()`

负责流式问答，但输出的仍然是网关统一的 `StreamChunk`，不是厂商原始 chunk。

### `Embedding()`

负责向量化能力，为后面知识库、检索链路打基础。

这一层不要直接返回厂商 SDK 类型。

因为一旦返回了，抽象边界就破了。

---

## 05 Provider 进到内部，最好怎么拆

Provider 接口定完以后，内部最好不要继续写成一个超大的文件。

更实用的拆法通常是：

```text
provider_openai/
  client.go
  chat.go
  stream.go
  embedding.go
  mapper.go
  errors.go
```

这样拆的好处是，职责会很清楚。

### `client.go`

负责初始化 SDK 客户端、鉴权配置、base URL 等底层连接信息。

### `chat.go`

负责同步 chat 请求和响应适配。

### `stream.go`

负责流式协议转换。

### `embedding.go`

负责 embedding 请求与响应映射。

### `mapper.go`

负责统一结构和厂商结构之间的转换。

### `errors.go`

负责厂商错误映射为网关统一错误。

这种拆法的重点不是“文件好看”。

而是：

> **让变化集中，让重复减少。**

---

## 06 真正值得单独拎出来的，其实是 Mapper

很多实现里，最容易被忽略的一层就是 mapper。

但实际上，它是最值得单独放出来的。

因为 Provider 的本质，就是两次转换：

- 统一请求 -> 厂商请求
- 厂商响应 -> 统一响应

如果这两层转换逻辑散在 `Chat()`、`StreamChat()`、`Embedding()` 里，后面很快就会乱。

例如：

```go
func buildOpenAIChatRequest(req *ChatRequest) *openai.ChatRequest
func parseOpenAIChatResponse(resp *openai.ChatResponse) *ChatResponse
func parseOpenAIStreamChunk(chunk *openai.StreamChunk) *StreamChunk
```

这样做有两个直接好处。

### 第一，差异更容易看清

你在做 Qwen、DeepSeek、Ollama 适配时，更容易直接对比：

- 哪些字段是一致的
- 哪些字段是厂商特有的
- 哪些需要网关层补默认值

### 第二，更方便测试

mapper 比直接测 SDK 调用更容易做单元测试。

尤其是：

- 请求字段映射
- usage 转换
- 错误码识别
- 流式 chunk 转换

---

## 07 一进流式场景，Provider 最容易乱在哪里

同步接口好抽象。

真正难的是流式接口。

因为流式接口的差异点更多，而且链路更长。

最常见的坑有四个。

### 1. 直接把厂商 stream chunk 往上抛

这样做会让前端、日志层甚至业务层都开始感知厂商结构。

后面一换 provider，所有上层都得跟着改。

### 2. 没有明确 done chunk

很多厂商是“流关了就算结束”。

但网关层最好自己补一个统一的 `done` 事件。

### 3. usage 丢失

很多 provider 的 usage 在最后一包或者流结束时才拿到。

如果不在 Provider 内部统一补齐，上层就很容易统计不到。

### 4. ctx 取消没有及时传到底层

如果用户已经停掉请求，但 Provider 还继续消费模型流，就会白白浪费资源和费用。

所以流式 Provider 这层最重要的不是“能读流”。

而是：

> **要把流安全、完整、统一地收回来。**

---

## 08 Provider 和模型路由，边界到底怎么配

这两个层次很容易被写混。

一个简单判断方式是：

- Provider 决定“怎么调这一家”
- Router 决定“这次调哪一家”

也就是说：

Provider 不应该自己决定场景路由。

例如这种代码就不太合适：

```go
if req.Scene == "faq" {
	useCheapModel()
}
```

这类逻辑更应该放在路由层。

Provider 只负责：

- 接收已经选定的 provider / model
- 正确完成调用
- 返回统一结果

边界一旦清楚，后面多 provider 才不会越接越乱。

---

## 09 第一版到底要不要补“能力探测”

有些团队会问：

> 不同 provider 支持能力不一样，要不要专门加一层 capability 定义？

答案是：

> **如果你已经开始接第二家、第三家 provider，就值得做。**

例如：

```go
type Capability struct {
	Chat       bool
	StreamChat bool
	Embedding  bool
}
```

然后在 Provider 上补一个：

```go
Capabilities() Capability
```

这样做的价值是：

- 路由层更容易判断可用能力
- 配置校验更容易做
- 前端或管理后台更容易展示模型支持情况

但如果你现在只接一两家、能力也非常单一，就没必要为了“完整”硬加很多复杂字段。

还是那句话：

> **围绕真实差异做抽象，不要围绕想象做抽象。**

---

## 10 如果现在就开做，最稳的顺序是什么

如果你现在就准备开做，我会建议按这个顺序来。

### 第一步：先定统一请求 / 响应协议

先把 `ChatRequest`、`ChatResponse`、`StreamChunk`、`EmbeddingRequest` 定下来。

### 第二步：先接一家 Provider

例如先接 OpenAI。

把：

- 同步 chat
- stream chat
- usage 提取
- 错误映射

先完整跑通。

### 第三步：抽出 Mapper

让请求、响应、错误转换开始稳定下来。

### 第四步：再接第二家 Provider

例如接 Qwen 或 DeepSeek。

到这一步，你才真正能验证：

> 你的 Provider 抽象是不是贴地。

### 第五步：再补 capability 和配置校验

这样抽象会更稳，不会一开始就设计过度。

---

## 11 最后

Provider 抽象这件事，最容易走向两个极端：

- 要么完全不抽，最后厂商差异一路往上漏
- 要么抽得太大，结果第一家接完就发现接口不贴地

更合适的做法其实很朴素：

> **先围绕 chat、stream、embedding 三类真实能力，做一层统一适配。**

只要这层边界清楚，后面你再往上补：

- 模型路由
- 失败重试
- Token 成本统计
- 多模型治理

整个网关才会有真正的可维护性。

下一篇，我们就继续往前走：

> **模型路由，到底应该放在哪一层。**

顺着这一组继续读，下一篇就在这里：

- [模型路由应该放在哪一层](/Users/liujun/workspace/shop/weixin-mp/Go实战/网关/模型路由应该放在哪一层/README.md)
