# 核心实战：基于 Eino 实现 Function Calling（工具调用）

## 一、什么是 Function Calling（工具调用）？
之前的 AI 只能“聊天”，遇到需要操作数据库（比如新建备忘录）的需求时，它只能回答“抱歉，我做不到”。
**Function Calling** 是让大模型拥有“手脚”的核心技术。我们把本地函数的说明书发给大模型，大模型根据用户的意图，决定是否调用该函数，并自动提取出需要的参数。最后，我们将函数的执行结果返回给大模型，让它生成最终的回答。

## 二、实现流程（核心四步）

### 1. 编写工具（给 AI 看的说明书）
在 `tools/memo_tool.go` 中，我们定义了 `CreateMemoParams` 结构体，并使用 Eino 的 `utils.InferTool` 自动生成工具描述：
```go
memoTool, err := utils.InferTool("create_memo", "当用户想要新建一条备忘录或记事时使用此工具", CreateMemoFunc)
原理：InferTool 底层利用反射（Reflect）扫描了 CreateMemoFunc 的签名和结构体里的 jsonschema 标签，自动拼装成符合大模型标准的 JSON Schema 格式。

2. Handler 层绑定工具
在 handler/chat.go 中，我们把工具注入到 ChatModel 中：

go
toolsList := tools.GetMemoTools()
ChatModelWithTools, err := utils.ChatModel.WithTools(toolsList)
3. 拦截并解析工具调用（最核心的一步）
第一次调用大模型后，检查它是否决定使用工具：

go
if len(resp.ToolCalls) > 0 {
    toolCall := resp.ToolCalls[0]
    if toolCall.Function.Name == "create_memo" {
        // 关键点：AI 传回来的 Arguments 是 JSON 字符串（如 {"content": "xxx"}），必须用 json.Unmarshal 解析
        var params tools.CreateMemoParams
        json.Unmarshal([]byte(toolCall.Function.Arguments), &params)
        
        // 手动执行本地业务逻辑（存数据库）
        memo := model.CreateMemo(params.Content)
        // ...
    }
}
4. 结果二次回传（闭环）
执行完本地函数后，把结果作为 schema.Tool 角色的消息，追加到上下文，进行第二次请求，让大模型做最终总结：

go
messages = append(messages, resp) // 追加 AI 决定调用工具的消息
messages = append(messages, &schema.Message{
    Role:       schema.Tool,
    Content:    fmt.Sprintf("操作成功，备忘录ID为 %d", memo.ID),
    ToolCallID: toolCall.ID,
})

finalResp, _ := ChatModelWithTools.Generate(ctx, messages)
// 最终前端拿到的就是 finalResp.Content
三、踩坑与版本避雷记录
WithTools 参数类型不匹配：

报错：WithTools 期待的是 []*schema.ToolInfo，但传入了 []tool.BaseTool。

解决：修改 GetMemoTools() 的返回值为 []*schema.ToolInfo，在函数内部使用 memoTool.Info(context.Background()) 转换。

导入了未使用的包：

修改返回值后，github.com/cloudwego/eino/components/tool 包失去了用途，编辑器报 imported and not used。直接删掉即可。

Go 切片判断优化（S1009）：

编辑器提示 should omit nil check。直接写 if len(resp.ToolCalls) > 0，不要写 if resp.ToolCalls != nil && len(...) > 0，因为切片的 len 对 nil 是安全的（返回 0）。

数据结构强转：

绝对不要尝试把 ID 直接转成字符串返回，Go 里会变成乱码。必须用 fmt.Sprintf("%d", id)。