# chonkpilot-router

ChonkPilot 的 **LLM 协议路由与适配** 组件：把各家大模型的请求、响应与流式事件归一为同一套 canonical 模型。

## 定位

- **纯库**：只依赖 Go 标准库，不引入 MQ / 数据库 / 可执行文件
- **无状态**：`Router` 只持有 provider 注册表与协议适配器表；会话、轮次、计价、调度均属调用方职责
- **零外部耦合**：不依赖 ChonkPilot 主仓任何代码

## 目录

| 路径 | 说明 |
|---|---|
| `router.go` | `New(Config)` / `Register(Spec)` / `Unregister(name)` / `Call(ctx, Request)` / `Capabilities(provider)` |
| `reconcile.go` | provider 配置增量对账 `Reconcile([]Spec) (Report, error)` |
| `retry.go` | 重试策略（连接期透明重试；已产出事件后不再重试） |
| `error.go` | 错误分类（network / timeout / rate_limit / server / auth / protocol / invalid）与退避 `RetryWait` |
| `caps.go` | 能力声明 |
| `internal/canon/` | canonical 请求 / 响应 / 事件 / 错误类型 |
| `internal/adaptor/openai/` | OpenAI Chat Completions 与 Responses |
| `internal/adaptor/anthropic/` | Anthropic Messages |
| `internal/adaptor/echo/` | 回显适配器（测试用；同时是「无可用 provider」时的内置兜底） |

## 用法

```go
r := router.New(router.Config{})            // 内置 openai / anthropic / responses / echo 适配器
if err := r.Register(router.Spec{           // Spec 字段见 types.go
    Name:     "my-openai",
    Protocol: "openai",
}); err != nil {
    return err
}

ch, err := r.Call(ctx, req)                 // 返回事件通道；通道关闭 = 本次调用结束
if err != nil { return err }
for ev := range ch {
    switch ev.Type {
    case router.EvText:  // 增量文本
    case router.EvUsage: // token 用量（prompt / completion / total / reasoning / cached）
    case router.EvDone:  // 正常结束
    case router.EvError: // 失败（带 Phase / Kind）
    }
}
```

> `Call` 只负责「发出去、把事件吐回来」。**计价不在本组件**：`EvUsage` + `Request.Provider` 一并交给调用方，由调用方决定价格与账单归属。

## 测试

```bash
go test ./...
```

含 `deps_test.go`：断言本模块只依赖标准库（守住「纯库」边界）。

## 与主仓的关系

ChonkPilot 主仓通过 `go.mod` 的 `replace` 引用本模块做本地联调；发布后改为 `require` + 语义化版本。
