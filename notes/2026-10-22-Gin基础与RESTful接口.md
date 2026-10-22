---

# Gin 框架基础与 RESTful 接口

## 一、Gin 核心概念与启动
**核心目标**：启动一个 Web 服务器，监听端口并接收请求。

**关键知识点**：
1. `r := gin.Default()`：创建一个 Gin 路由引擎（相当于大堂经理），包含了日志和异常恢复中间件。
2. `r.Run(":8080")`：启动 HTTP 服务并监听 8080 端口。这是一个**阻塞操作**，程序会一直挂在这里等待请求。
3. **404 陷阱**：如果没有在代码里写具体的路由规则，访问对应路径会直接返回 `404 page not found`。

## 二、GET 请求与参数接收
**核心目标**：让接口能根据前端传来的不同参数，返回不同的数据。

**1. 返回 JSON 数据**
```go
c.JSON(200, gin.H{
    "message": "这是我的第一个接口",
})
```

* `200` 是 HTTP 状态码，表示成功。
* `gin.H` 本质上是 `map[string]interface{}` 的别名（万能字典），可以塞入字符串、数字、布尔值、数组等任意类型，Gin 会自动将其转为 JSON 格式返回。

**2. 接收 Query 参数（URL 问号后面的参数）**
* 请求格式：`/hello?name=JyiDuo`
* 获取方式：`name := c.Query("name")` （拿到的永远是字符串）
* 兜底逻辑：如果前端没传，会拿到空字符串 `""`，需要写 `if name == ""` 来赋默认值。

**3. 接收 Path 参数（路径中的动态参数）**
* 请求格式：`/user/JyiDuo/88`
* 路由定义：`r.GET("/user/:name/:id", ...)` （注意必须有冒号 `:`）
* 获取方式：`name := c.Param("name")`

## 三、POST 请求与 JSON 数据绑定（核心重点）
**核心目标**：接收前端通过 Body 发来的 JSON 数据，并进行安全解析。

**1. 定义结构体（模具）**
```go
type RegisterReq struct {
    Username string `json:"username"`
    Password string `json:"password"`
}
```
* 使用 Tag（反引号 `` `json:"xxx" ` ``）告诉 Gin，JSON 里的小写键名对应 Go 里的大写变量。
* **必须使用指针 `&req` 传递**给绑定函数，否则数据无法保存到原变量中。

**2. 核心绑定代码**
```go
var req RegisterReq
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(400, gin.H{"error": "数据格式不对"})
    return // 必须 return，防止代码继续往下走
}
```
* `c.ShouldBindJSON` 负责把网络传过来的纯文本解析并塞进结构体。
* 如果前端发的是残缺的 JSON，`err != nil` 会拦截并返回 `400 Bad Request`。这是一种**防御性编程**，永远不要相信前端传来的数据。

**3. 使用数据**
通过 `req.Username`、`req.Password` 可以直接取出绑定好的值。

## 四、接口测试工具：Apifox
* 浏览器地址栏只能发 GET 请求，**发 POST 请求必须用工具（如 Apifox / Postman）**。
* 操作流程：选择 POST 方法 → 填入 URL → 在 Body 选项卡选择 JSON 格式 → 填入 JSON 数据 → 点击发送。
* 成功标志：响应区返回状态码 `200`，且能看到后端拼装的 JSON 结果。

---
