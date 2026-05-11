# 统一错误处理应该怎么做

---

AI 网关做到这一步，通常已经开始有这些能力了：

- 统一请求和响应结构
- Provider 抽象
- 模型路由
- 流式输出
- 超时、重试、限流

但只要一接进真实业务系统，很快又会遇到另一个很麻烦的问题：

> **错误越来越多，但谁也说不清这些错误到底该怎么处理。**

最典型的混乱通常是这样的：

- 厂商 SDK 抛一套错误
- Provider 再包一层错误
- 网关再改一层 message
- Handler 最后直接把 `err.Error()` 返回给前端

结果就是：

- 前端不知道哪些错误可重试
- 业务层不知道哪些错误该降级
- 日志里全是字符串，不可统计
- 厂商一换，错误语义又跟着漂

所以这篇文章，我们专门来讲：

> **AI 网关里的统一错误处理，到底该怎么做，才能既稳定，又可治理。**

这篇你会拿到 3 个结论：

- 统一错误处理真正要统一的不是文案，而是语义
- 第一版最值得稳定下来的错误分类有哪些
- 同步和流式错误为什么应该围绕同一套 code 来处理

---

## 01 统一错误处理的目标，不是统一文案，而是统一语义

很多人一说统一错误处理，第一反应是：

> 把错误文案都改成同一个格式。

这当然是表面上的统一。

但 AI 网关更重要的，其实是语义统一。

也就是：

> **不管底层是哪一家 provider，网关对上层都应该提供稳定、可判断的错误分类。**

因为业务层真正关心的不是：

- 这句英文报错长什么样

而是：

- 能不能重试
- 能不能降级
- 要不要提示用户稍后再试
- 是不是参数写错了
- 是不是模型根本不可用

这才是错误统一最重要的价值。

---

## 02 第一版最容易犯的两个错

### 错误一：直接往上抛原始错误

例如：

代码示例：

    return nil, err

短期最省事。

但一旦接多家 provider，就会出现：

- 不同厂商 message 风格完全不同
- 同类错误没有统一识别方式
- 前端和业务层只能做字符串判断

### 错误二：层层包 message，最后看不出本质

例如：

代码示例：

    return fmt.Errorf("call qwen failed: %w", err)

再上一层：

代码示例：

    return fmt.Errorf("gateway chat failed: %w", err)

最后你拿到一长串上下文，但仍然回答不了：

> 这到底是超时、限流、参数错误，还是 provider 故障？

所以统一错误处理真正要解决的，不是“多包几层”。

而是：

> **先把错误归类，再决定上下文怎么补。**

---

## 03 第一版先把哪些错误分类立住，就够了

如果你是在做 AI 网关起步阶段，我会建议先把错误分类控制在一个很小但够用的集合里。

例如：

- `ErrInvalidRequest`
- `ErrUnauthorized`
- `ErrRateLimited`
- `ErrTimeout`
- `ErrProviderUnavailable`
- `ErrModelNotFound`
- `ErrInternal`

这个集合不需要一开始就铺得很大。

重点是：

- 分类稳定
- 业务能判断
- 日志能统计

### `ErrInvalidRequest`

表示请求本身有问题。

例如：

- 参数非法
- messages 为空
- 模型名格式错误

### `ErrUnauthorized`

表示认证或权限不通过。

例如：

- API Key 无效
- 当前租户无权限调用该模型

### `ErrRateLimited`

表示命中限流，无论是网关自己的限流还是上游 provider 限流，都可以先收敛到这一类。

### `ErrTimeout`

表示调用超时。

不管是首字超时、整体超时还是上游超时，起步阶段都可以先归到同一大类，再通过 meta 细分。

### `ErrProviderUnavailable`

表示 provider 当前不可用。

例如：

- 上游 5xx
- 连接失败
- 服务异常

### `ErrModelNotFound`

表示模型不存在、已下线或当前 provider 不支持该模型。

### `ErrInternal`

表示网关内部未分类异常，作为最后兜底。

---

## 04 统一错误结构，最适合长成什么样

很多团队只定义 error type，不定义结构化返回。

但在 AI 网关里，最好统一一个结构。

例如：

代码示例：

    type ErrorCode string
    
    const (
    	ErrorCodeInvalidRequest      ErrorCode = "invalid_request"
    	ErrorCodeUnauthorized        ErrorCode = "unauthorized"
    	ErrorCodeRateLimited         ErrorCode = "rate_limited"
    	ErrorCodeTimeout             ErrorCode = "timeout"
    	ErrorCodeProviderUnavailable ErrorCode = "provider_unavailable"
    	ErrorCodeModelNotFound       ErrorCode = "model_not_found"
    	ErrorCodeInternal            ErrorCode = "internal"
    )
    
    type GatewayError struct {
    	Code      ErrorCode
    	Message   string
    	Provider  string
    	Model     string
    	Retryable bool
    	Cause     error
    }

这一版里，最有价值的字段通常是：

### `Code`

这是最稳定的分类字段，前端、业务层、日志系统都可以围绕它判断。

### `Message`

给用户或调用方看的可读信息。

### `Provider` / `Model`

方便排查问题落在哪一层。

### `Retryable`

这是很实用的一个字段。

因为很多错误的核心判断就是：

> 有没有必要重试。

### `Cause`

保留底层错误，方便日志里追根溯源。

---

## 05 Provider 在错误统一里，最该负责哪一段

这个边界非常关键。

Provider 层最应该做的，不是发明新错误体系。

而是：

> **把厂商错误映射成网关统一错误。**

例如：

- 某厂商 HTTP 429 -> `ErrRateLimited`
- 某厂商 timeout -> `ErrTimeout`
- 某厂商 model not exist -> `ErrModelNotFound`

这层一旦做好，后面的 Router、Governance、Handler、Frontend 都可以稳定围绕统一错误工作。

换句话说：

- Provider 负责翻译
- Gateway 负责使用

这才是比较清楚的职责边界。

---

## 06 一进流式场景，为什么错误处理更不能含糊

同步请求出错比较简单。

要么返回结果，要么返回错误。

但流式请求不一样。

因为错误可能发生在三个阶段：

### 第一，流开始前就失败

这时还可以像普通错误一样直接返回。

### 第二，流已经开始输出，过程中失败

这时不能再用普通 HTTP 错误语义。

更合适的做法通常是：

> **在 SSE / stream 协议里主动发一个统一的 `error` 事件。**

### 第三，流因为用户取消而结束

这通常不应该被当成普通系统错误。

更适合单独标成：

- user cancelled
- client disconnected

也就是说，流式错误处理必须区分：

- 失败
- 中断
- 正常结束

不能全都靠“连接断了”去猜。

---

## 07 错误语义一旦统一，上层到底能拿到什么

这件事的价值，最终还是要回到上层能不能用。

如果错误分类做稳了，前端和业务层通常就可以做这些清晰判断：

### 前端可以做

- `rate_limited`：提示稍后再试
- `timeout`：提示网络或服务较慢
- `unauthorized`：提示无权限
- `model_not_found`：提示当前模型不可用

### 业务层可以做

- `provider_unavailable`：走降级策略
- `rate_limited`：切备用模型
- `timeout`：缩小请求范围或改走同步兜底

### 日志和监控可以做

- 统计超时率
- 统计限流命中率
- 统计不同 provider 的失败类型分布

这才是统一错误处理真正的收益。

---

## 08 第一版最容易漏掉的 4 个细节

### 1. 只统一 code，不保留 cause

这样虽然表面统一了，但后面排查时会丢掉很多底层信息。

### 2. 只保留原始 message，不给稳定 code

这样前端和业务层最后还是只能做字符串判断。

### 3. 所有错误都标成 retryable

这会导致重试逻辑泛滥，甚至让无意义请求一遍遍打上游。

### 4. 流式错误没有统一协议

如果同步和流式用两套完全不同的错误语义，前后端后面会越接越乱。

---

## 09 真准备落地时，最稳的顺序是什么

如果你现在准备开始做，我会建议这样推进。

### 第一步：先定义统一错误 code

先控制在一个小而稳定的集合里。

### 第二步：在 Provider 层补错误映射

先接一家 provider，再接第二家，看分类是否仍然成立。

### 第三步：在 Gateway 层统一包结构

让同步和流式都围绕同一套错误语义返回。

### 第四步：把错误 code 纳入日志和监控

没有统计，统一错误的价值会打折很多。

### 第五步：最后再补更细的子分类

例如：

- 首字超时
- 总超时
- provider 限流
- gateway 限流

这些可以在大类稳定后再逐步细化。

---

## 10 最后

统一错误处理这件事，看起来像一个很基础的话题。

但在 AI 网关里，它实际上决定了三件事：

- 上层能不能稳定分支判断
- 前端能不能做清晰提示
- 日志和监控能不能形成有效统计

如果这层没处理好，后面你即使把路由、Provider、流式输出都做出来了，系统仍然会很难治理。

所以更合适的做法是：

> **先把错误统一成稳定语义，再去补上下文、提示文案和降级策略。**

这样整个网关的行为才会越来越清楚，而不是越来越像一团字符串。
