# 如何实现 ChatGPT 一样的流式输出

---

这是 `AI 网关专题` 的第 2 篇。

上一篇我们先把一件更底层的事讲清楚了：

> **Go 后端里，为什么要先把大模型调用收口成统一调用层。**

但调用层抽出来以后，真实项目里很快就会遇到下一步：

> **怎么把模型生成过程一段一段地返回给前端。**

很多团队第一次做 AI 聊天时，后端代码大概是这样的：

```go
resp, err := gateway.Chat(ctx, req)
if err != nil {
	return err
}

return c.JSON(http.StatusOK, resp)
```

这种方式当然能跑。

但用户体验通常会很一般。

因为用户会感觉自己不是在“对话”，而是在“提交一个表单，然后等系统算完再返回一个大结果”。

而我们平时在用 ChatGPT、Claude 这一类产品时，已经很习惯另一种体验了：

- 输入问题后，几乎立刻开始返回内容
- 内容是一段一段往外冒
- 用户能感受到系统“正在思考”
- 如果内容不对，可以早点打断

这就是流式输出真正的价值。

它不只是“界面更炫”，而是会直接影响三个东西：

- 首字返回时间
- 用户体感等待时间
- 整个 AI 对话链路的交互方式

这篇文章，我们就把这件事拆开讲透：

> **Go 后端里，如何把大模型的流式结果稳定地返回给前端。**

这篇你会拿到 3 个结论：

- 第一版为什么更推荐用 SSE，而不是直接上 WebSocket
- 流式输出这条链路到底应该怎么拆
- 流式协议、done 信号和 usage 收口为什么必须提前定好

---

## 01 什么叫“像 ChatGPT 一样的流式输出”

很多人一提到流式输出，第一反应是：

> 不就是“边生成边返回”吗？

方向没错，但真要在业务系统里落地，不能只停在这个层面。

一个更贴近工程实现的定义应该是：

> **后端在模型持续生成内容的过程中，把增量结果按统一协议实时推送给前端，并能正确处理结束、报错、中断和审计。**

这里至少有四个关键词。

### 第一，增量返回

不是等整段答案生成完再一次性返回，而是：

- 先返回第一段
- 再返回下一段
- 最后返回完成信号

### 第二，统一协议

底层模型厂商的流式协议往往并不一样。

有的按 chunk 返回。
有的按 delta 返回。
有的字段叫 `content`。
有的字段叫 `reasoning_content`。
有的最后一包里才带 usage。

如果后端不先把协议统一，前端最后一定会越接越乱。

### 第三，生命周期完整

真正可用的流式输出，不只是“有字回来”。

还要能区分：

- 正在输出
- 正常结束
- 用户中断
- 上游报错
- 网关超时

### 第四，可治理

一旦进入真实业务系统，就不能只关心显示效果。

还要考虑：

- 这次流式请求是谁发起的
- 走了哪个模型
- 返回了多长时间
- 中途是不是断了
- 最后用了多少 Token

所以流式输出，从来不是一个单纯的前端效果问题。

它本质上是：

> **模型流、网关协议、HTTP 传输和前端消费共同组成的一条链路。**

---

## 02 为什么很多团队第一版流式输出都做得不太稳

很多第一版实现，通常是这样长出来的：

```text
前端发起请求
后端直接调用模型 SDK 的 stream 接口
拿到一段内容就 write 一次
最后 close 掉连接
```

Demo 阶段这没什么问题。

但只要准备往线上走，问题很快就会出现。

### 问题一：前端和模型协议绑死了

如果后端只是把厂商原始 chunk 原样透传给前端，那么前端就会直接依赖：

- 厂商字段名
- 厂商结束标记
- 厂商错误结构
- 厂商流式粒度

后面你一换模型提供方，前端也得跟着改。

### 问题二：结束信号不清晰

很多实现里，前端只能靠“连接断开”来猜这次输出是不是结束了。

这会带来两个问题：

- 正常结束和异常断开很难区分
- 前端没法做更稳的收尾处理

### 问题三：中断处理不完整

用户关闭页面、点击停止生成、网络中断，这些都属于非常常见的场景。

如果后端没有明确感知 `context.Context` 的取消信号，就可能出现：

- 前端断了，模型还在生成
- 请求已经没意义了，Token 还在继续烧
- 日志里看起来像“成功结束”，但其实用户早走了

### 问题四：流式日志和审计断档

同步接口比较容易记录完整响应。

但流式接口如果没提前设计，往往会出现两种极端：

- 要么完全不记
- 要么每个 chunk 都打一条日志，最后日志爆炸

这两种都不适合线上系统。

### 问题五：代理层把流式“吞掉”了

你本地调试明明是一段一段返回。

一上 Nginx、网关、CDN 或某些云环境，前端却突然变成“最后一口气全出来”。

这通常不是模型的问题，而是：

- 响应被缓冲了
- 压缩策略不合适
- 反向代理没有关闭 buffering

所以流式输出这件事，真正要做稳，不能只盯着模型 SDK。

---

## 03 在 Go 后端里，第一版最推荐的方案是什么

如果你是做 Web 后端、管理后台、企业知识库、AI 助手这类系统，我会更建议第一版优先走：

> **SSE（Server-Sent Events）**

而不是一上来就用 WebSocket。

原因很简单。

### SSE 更适合“服务端持续推送文本”

AI 聊天的主链路，大多数时候是：

> 前端发一个请求，后端持续返回一段段文本。

这本来就是 SSE 最擅长的场景。

### SSE 基于普通 HTTP

这意味着：

- 更容易接入现有 Go Web 服务
- 更容易复用鉴权、中间件、日志
- 更容易通过浏览器直接消费
- 更适合和现有 REST / RPC 接口并存

### SSE 的心智负担更低

WebSocket 更强，但也更重。

你要处理：

- 连接状态
- 双向消息协议
- 心跳
- 重连
- 多类消息复用

如果当前只是“把模型生成过程稳定返回给前端”，SSE 往往更划算。

所以第一版可以先定一个很清楚的策略：

> **模型层可以是流，网关对前端暴露 SSE。**

这通常是最稳的落地方式。

---

## 04 一条完整的流式链路，应该怎么拆

把它拆开看，会更清楚：

```text
浏览器
  -> HTTP Handler
  -> App Service / Biz
  -> AI Gateway
  -> Provider Adapter
  -> Model Vendor Stream
```

每一层都只做自己那一层的事。

### HTTP Handler

负责：

- 建立 SSE 响应头
- 检查连接是否支持 flush
- 把统一 chunk 写回浏览器

### Biz / App Service

负责：

- 组织场景上下文
- 拼接消息
- 决定这次走哪个流式能力

### AI Gateway

负责：

- 统一流式返回结构
- 路由到不同模型提供方
- 记录调用开始、结束、错误、耗时

### Provider Adapter

负责：

- 把厂商流式协议转换成统一 chunk
- 屏蔽字段差异
- 识别厂商结束信号

### Frontend

负责：

- 订阅 SSE
- 按 chunk 累积内容
- 区分 done、error、close

你会发现，这个拆法和上一篇讲的“调用层收口”是完全一致的。

流式输出不是另外起一套系统。

它只是：

> **在统一调用层之上，再补一条统一的流式链路。**

---

## 05 先定协议，比先写代码更重要

很多团队做流式输出时，最大的坑就是：

> 一上来先写 `for chunk := range stream`。

代码很快能写出来。

但协议一旦没想清楚，后面前后端就会持续返工。

第一版我更建议先把统一 chunk 结构定下来。

例如：

```go
type StreamEvent string

const (
	StreamEventDelta StreamEvent = "delta"
	StreamEventDone  StreamEvent = "done"
	StreamEventError StreamEvent = "error"
)

// StreamChunk 表示网关对外统一的流式返回结构。
type StreamChunk struct {
	Event   StreamEvent `json:"event"`
	Content string      `json:"content,omitempty"`
	TraceID string      `json:"traceID,omitempty"`
	Model   string      `json:"model,omitempty"`
	Done    bool        `json:"done,omitempty"`
	Error   *ErrorInfo  `json:"error,omitempty"`
	Usage   *Usage      `json:"usage,omitempty"`
}

type ErrorInfo struct {
	Code    string `json:"code"`
	Message string `json:"message"`
}

type Usage struct {
	PromptTokens     int64 `json:"promptTokens"`
	CompletionTokens int64 `json:"completionTokens"`
	TotalTokens      int64 `json:"totalTokens"`
}
```

这个结构里，第一版最值得保留的字段有四类：

### 1. 事件类型

前端要知道这是一段普通增量，还是结束信号，还是错误信号。

### 2. 文本内容

增量文本单独放在 `content` 里，保持语义简单。

### 3. 元信息

例如 `traceID`、`model`，方便日志排查和问题回放。

### 4. usage

很多模型厂商只会在最后一包给 usage。

但你可以在统一协议层保留这个位置，让前端和日志系统都能稳定接住。

先把这个协议定稳，后面不管你接 OpenAI、Qwen、DeepSeek 还是 Ollama，前端都不需要反复重写。

---

## 06 Go 里最小可用的流式接口该怎么写

我们先看一版很实用的接口抽象。

```go
// StreamWriter 负责把统一 chunk 发给上层调用方。
type StreamWriter interface {
	WriteChunk(ctx context.Context, chunk *StreamChunk) error
}

// Gateway 定义对外暴露的流式调用能力。
type Gateway interface {
	StreamChat(ctx context.Context, req *ChatRequest, writer StreamWriter) error
}
```

这个抽象有两个好处。

### 第一，不把协议绑死在 HTTP 层

一旦你把 `http.ResponseWriter` 直接传进网关层，边界很快就会乱掉。

网关层最应该关心的，不是 HTTP 细节，而是：

> 我产生了一个统一 chunk，交给上层发出去。

### 第二，更方便复用

今天你用的是 SSE。

后面如果你想接：

- WebSocket
- gRPC stream
- 内部消息总线

那么只要换一个 `StreamWriter` 的实现即可。

这就是为什么流式输出最好不要直接写成“模型 SDK + HTTP Handler 黏在一起”。

---

## 07 SSE Handler 第一版可以怎么实现

这一层反而不需要搞太复杂。

一版够用、而且边界比较清晰的写法可以是这样：

```go
type SSEWriter struct {
	writer  http.ResponseWriter
	flusher http.Flusher
}

func NewSSEWriter(w http.ResponseWriter) (*SSEWriter, error) {
	flusher, ok := w.(http.Flusher)
	if !ok {
		return nil, errors.New("response writer does not support flush")
	}

	return &SSEWriter{
		writer:  w,
		flusher: flusher,
	}, nil
}

func (w *SSEWriter) WriteChunk(ctx context.Context, chunk *StreamChunk) error {
	select {
	case <-ctx.Done():
		return ctx.Err()
	default:
	}

	payload, err := json.Marshal(chunk)
	if err != nil {
		return fmt.Errorf("marshal stream chunk: %w", err)
	}

	_, err = fmt.Fprintf(w.writer, "event: %s\n", chunk.Event)
	if err != nil {
		return fmt.Errorf("write sse event: %w", err)
	}

	_, err = fmt.Fprintf(w.writer, "data: %s\n\n", payload)
	if err != nil {
		return fmt.Errorf("write sse data: %w", err)
	}

	w.flusher.Flush()
	return nil
}
```

HTTP 入口层则可以这样收口：

```go
func (h *Handler) StreamChat(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")
	w.Header().Set("X-Accel-Buffering", "no")

	writer, err := NewSSEWriter(w)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	req, err := h.buildChatRequest(r)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	err = h.gateway.StreamChat(ctx, req, writer)
	if err != nil {
		streamErr := writer.WriteChunk(ctx, &StreamChunk{
			Event: StreamEventError,
			Error: &ErrorInfo{
				Code:    "stream_failed",
				Message: err.Error(),
			},
		})
		if streamErr != nil {
			return
		}
	}
}
```

这个实现里，有几个点很关键。

### `Content-Type` 必须是 `text/event-stream`

这决定了浏览器按 SSE 去消费，而不是普通文本响应。

### `X-Accel-Buffering: no`

如果你的流式链路里会经过 Nginx，这个头非常重要。

它能显式告诉代理层：

> 不要缓冲，尽快往下游刷。

### 每次写 chunk 后都要 `Flush()`

如果只写不 flush，很多时候前端仍然会等到缓冲区累积后才看到结果。

---

## 08 Provider 适配层，真正应该做什么

这一层是整个流式输出里最容易被低估的部分。

因为很多人会觉得：

> 模型 SDK 已经帮我 stream 了，我只要转发就行。

但真正上线时，Provider 适配层至少要做四件事。

### 1. 把厂商 chunk 转成统一 chunk

例如厂商返回：

```json
{
  "choices": [
    {
      "delta": {
        "content": "你好"
      }
    }
  ]
}
```

适配层应该把它转成：

```json
{
  "event": "delta",
  "content": "你好"
}
```

这样前端永远只看统一协议。

### 2. 识别结束信号

有些厂商通过 `finish_reason` 标识完成。
有些通过特殊 chunk。
有些最后一包还会夹带 usage。

这些差异都应该收在 Provider 内部。

### 3. 保留 usage 和 trace 信息

流式模式下，usage 很可能不是边输出边有，而是结束时才补齐。

所以适配层需要把最后的 usage 补进 done chunk 里。

### 4. 正确处理中断

如果上层 `ctx.Done()` 已经触发，Provider 就应该尽快停掉底层流，避免无意义继续消费。

这一点尤其关键。

因为模型流通常是计费中的外部资源。

---

## 09 前端为什么不要只靠“连接断开”判断结束

这个点很容易被忽略。

如果前端只靠浏览器连接结束来判断一次流式输出完成，会有几个问题：

- 网络异常和正常结束不容易区分
- 没法优雅展示“回答完成”
- usage、trace、model 信息不好在最后统一收口

所以更稳的做法是：

> **正常结束时，后端主动发一个 `done` 事件。**

例如：

```json
{
  "event": "done",
  "done": true,
  "usage": {
    "promptTokens": 120,
    "completionTokens": 260,
    "totalTokens": 380
  }
}
```

这样前端逻辑会清楚很多。

- `delta`：持续拼接文本
- `error`：展示错误
- `done`：结束 loading，记录 usage，完成一次回答

这比“等连接自己断开看看像不像结束”稳得多。

---

## 10 流式输出上线前，最容易漏掉的 6 个点

如果你准备把第一版推到测试环境，我会建议至少把下面这 6 件事补齐。

### 1. 超时控制

不要让一次流式请求无限挂着。

至少要区分：

- 上游模型超时
- 网关整体超时
- 用户主动取消

### 2. 代理层缓冲检查

本地能流，不代表线上能流。

一定要验证：

- Nginx buffering
- 网关层压缩策略
- 云平台响应缓冲

### 3. 中断日志

要能区分：

- 用户主动取消
- 浏览器断连
- 上游厂商报错
- 网关自己超时

否则后面排查会非常痛苦。

### 4. 最终 usage 收口

流式链路不能只把字吐给前端就结束。

还要记得把最终 usage 记进日志、成本统计和审计链路。

### 5. 部分输出内容的审计策略

流式模式下，内容不是一次性生成。

你要提前想好：

- 是否记录完整最终回答
- 是否记录中途增量
- 是否需要脱敏

### 6. 前端停止生成能力

只做“能输出”，不做“能停止”，体验会差很多。

尤其是长回答和错误回答场景。

所以前后端最好尽快补齐用户主动取消这条链路。

---

## 11 第一版最推荐的落地顺序

如果你现在就要在 Go 项目里把流式输出做起来，我会建议按这个顺序推进：

### 第一步：先统一 chunk 协议

先不要急着纠结 SSE 细节。

先把网关对外的 `StreamChunk` 定稳。

### 第二步：接一个模型提供方跑通

例如先接 OpenAI 或 Qwen 的流式接口。

先跑通：

- 增量输出
- done 收尾
- error 返回

### 第三步：补 SSE Handler

把统一 chunk 按 SSE 协议推给前端。

### 第四步：补日志、usage 和 traceID

让流式请求不只是“能用”，而是“可追踪”。

### 第五步：再考虑多 Provider 和模型路由

这样节奏会稳很多。

如果一上来就想把 WebSocket、多模型、多路由、审计、重试、取消、回放一次全做完，第一版通常会很重。

流式输出真正值得优先做的，不是面面俱到，而是：

> **先把这条链路清晰、稳定地跑通。**

---

## 12 最后

很多人第一次做 AI 聊天，会把注意力全放在模型效果上。

但只要一接到真实系统里，就会很快发现：

> **体验的上限，不只取决于模型，也取决于后端链路怎么设计。**

流式输出就是其中一个非常典型的例子。

它看起来像一个前端交互细节。

但真正落地时，背后会牵动：

- 模型流协议
- 网关抽象
- HTTP 传输
- 中断控制
- usage 统计
- 日志和审计

所以这件事最稳的做法，从来不是“把模型 SDK 直接往前端透”。

而是：

> **在 Go 后端里，把流式输出也收口成统一能力。**

这样你后面再往上补：

- 多模型切换
- 成本统计
- 敏感内容治理
- 工具调用
- RAG 引用来源

整个系统才不会越长越乱。

下一篇，我们可以继续往前走一步：

> **AI 网关里，Token 成本统计到底应该放在哪一层。**

如果你想按专题顺着往下走，更推荐先接这篇：

- [Provider 抽象该怎么设计](/Users/liujun/workspace/shop/weixin-mp/Go实战/网关/Provider%20抽象该怎么设计/README.md)
