### 定义 AI 工作流

您的应用 AI 功能的核心是生成式模型请求，但很少能简单地将用户输入传递给模型，然后将模型输出直接显示给用户。通常，模型调用前后都需要伴随着一些前处理和后处理步骤。例如：

*   检索上下文信息，以便与模型调用一起发送。
*   在聊天应用中，检索用户当前会话的历史记录。
*   使用一个模型将用户输入重新格式化，以使其适合传递给另一个模型。
*   在向用户展示模型输出之前，评估其“安全性”。
*   组合多个模型的输出。

为了让任何与 AI 相关的任务成功，这个工作流中的每一步都必须协同工作。

在 Genkit 中，您使用一种称为 **Flow** 的结构来表示这种紧密耦合的逻辑。Flow 的编写方式与普通函数一样，使用标准的 Go 代码，但它们增加了一些旨在简化 AI 功能开发的额外能力：

*   **类型安全**：输入和输出的 schema，提供静态和运行时的类型检查。
*   **与开发者界面集成**：使用开发者界面独立于您的应用程序代码来调试 Flow。在开发者界面中，您可以运行 Flow 并查看其每一步的追踪信息。
*   **简化的部署**：将 Flow 直接部署为 Web API 端点，可使用任何能够托管 Web 应用的平台。

Genkit 的 Flow 是轻量且无侵入性的，不会强迫您的应用遵循任何特定的抽象。Flow 的所有逻辑都用标准 Go 编写，Flow 内部的代码也无需感知 Flow 的存在。

### 定义和调用 Flow

最简单的形式下，一个 Flow 只是包装了一个函数。以下示例包装了一个调用 `genkit.Generate()` 的函数：

```go
menuSuggestionFlow := genkit.DefineFlow(g, "menuSuggestionFlow",
    func(ctx context.Context, theme string) (string, error) {
        resp, err := genkit.Generate(ctx, g,
            // 为一个 %s 主题的餐厅设计一道菜单菜品。
            ai.WithPrompt("Invent a menu item for a %s themed restaurant.", theme),
        )
        if err != nil {
            return "", err
        }

        return resp.Text(), nil
    })
```

仅仅通过这样包装您的 `genkit.Generate()` 调用，您就增加了一些功能：这样做可以让您从 Genkit 命令行工具（CLI）和开发者界面中运行该 Flow，并且这是 Genkit 多个功能（包括部署和可观测性，后续章节将讨论）的必要条件。

#### 输入和输出 Schema

Genkit Flow 相较于直接调用模型 API 的最重要优势之一是输入和输出的类型安全。在定义 Flow 时，您可以定义 schema，其方式与您为 `genkit.Generate()` 调用定义输出 schema 非常相似；但与 `genkit.Generate()` 不同的是，您还可以指定输入 schema。

这是上一个例子的改进版，它定义了一个接收字符串输入并输出一个对象的 Flow：

```go
type MenuItem struct {
    Name        string `json:"name"`
    Description string `json:"description"`
}

menuSuggestionFlow := genkit.DefineFlow(g, "menuSuggestionFlow",
    func(ctx context.Context, theme string) (MenuItem, error) {
        // GenerateData 会自动请求符合 MenuItem 类型的结构化输出
        return genkit.GenerateData[MenuItem](ctx, g,
            ai.WithPrompt("Invent a menu item for a %s themed restaurant.", theme),
        )
    })
```

请注意，Flow 的 schema 不必与 Flow 内部 `genkit.Generate()` 调用的 schema 完全一致（实际上，一个 Flow 甚至可能不包含 `genkit.Generate()` 调用）。下面是该示例的一个变体，它调用了 `genkit.GenerateData()`，但使用结构化输出来格式化一个简单的字符串，并由 Flow 返回。请注意我们如何将 `MenuItem` 作为类型参数传递；这相当于传递 `WithOutputType()` 选项并在响应中获取该类型的值。

```go
type MenuItem struct {
    Name        string `json:"name"`
    Description string `json:"description"`
}

menuSuggestionMarkdownFlow := genkit.DefineFlow(g, "menuSuggestionMarkdownFlow",
    func(ctx context.Context, theme string) (string, error) {
        item, _, err := genkit.GenerateData[MenuItem](ctx, g,
            ai.WithPrompt("Invent a menu item for a %s themed restaurant.", theme),
        )
        if err != nil {
            return "", err
        }

        return fmt.Sprintf("**%s**: %s", item.Name, item.Description), nil
    })
```

#### 调用 Flow

一旦定义了 Flow，您就可以在您的 Go 代码中调用它：

```go
item, err := menuSuggestionFlow.Run(context.Background(), "bistro")
```

传递给 Flow 的参数必须符合其输入 schema。

如果您定义了输出 schema，Flow 的响应将会符合该 schema。例如，如果您将输出 schema 设置为 `MenuItem`，则 Flow 的输出将包含其属性：

```go
item, err := menuSuggestionFlow.Run(context.Background(), "bistro")
if err != nil {
    log.Fatal(err)
}

log.Println(item.Name)
log.Println(item.Description)
```

### 流式 Flow

Flow 支持使用与 `genkit.Generate()` 的流式接口类似的接口进行流式传输。当您的 Flow 生成大量输出时，流式传输非常有用，因为您可以在生成输出时就将其呈现给用户，从而提高应用的感知响应速度。一个熟悉的例子是，基于聊天的 LLM 界面通常会在生成响应时将其流式传输给用户。

下面是一个支持流式传输的 Flow 示例：

```go
type Menu struct {
    Theme  string     `json:"theme"`
    Items  []MenuItem `json:"items"`
}

type MenuItem struct {
    Name        string `json:"name"`
    Description string `json:"description"`
}

menuSuggestionFlow := genkit.DefineStreamingFlow(g, "menuSuggestionFlow",
    func(ctx context.Context, theme string, callback core.StreamCallback[string]) (Menu, error) {
        item, _, err := genkit.GenerateData[MenuItem](ctx, g,
            ai.WithPrompt("Invent a menu item for a %s themed restaurant.", theme),
            ai.WithStreaming(func(ctx context.Context, chunk *ai.ModelResponseChunk) error {
                // 在这里，您可以在使用 StreamCallback 将数据块发送到
                // 输出流之前对其进行某种处理。在本例中，我们直接输出
                // 数据块的文本，未作修改。
                return callback(ctx, chunk.Text())
            }),
        )
        if err != nil {
            return Menu{}, err // 返回空结构体
        }

        return Menu{
            Theme: theme,
            Items: []MenuItem{item},
        }, nil
    })
```

`StreamCallback[string]` 中的 `string` 类型指定了您的 Flow 流式传输的值的类型。这不一定需要与返回类型相同，返回类型是 Flow 完整输出的类型（在本例中为 `Menu`）。

在这个例子中，Flow 流式传输的值与 Flow 内部 `genkit.Generate()` 调用流式传输的值直接耦合。尽管情况通常如此，但这并非必须：您可以根据 Flow 的需要，随时使用回调函数向流中输出值。

#### 调用流式 Flow

流式 Flow 可以像非流式 Flow 一样使用 `menuSuggestionFlow.Run(ctx, "bistro")` 来运行，也可以进行流式调用：

```go
streamCh, err := menuSuggestionFlow.Stream(context.Background(), "bistro")
if err != nil {
    log.Fatal(err)
}

for result := range streamCh {
    if result.Err != nil {
        log.Fatalf("流错误: %v", result.Err)
    }
    if result.Done {
        // 流已结束，result.Output 包含最终的完整结果
        log.Printf("主题为 %s 的菜单:\n", result.Output.Theme)
        for _, item := range result.Output.Items {
            log.Printf(" - %s: %s", item.Name, item.Description)
        }
    } else {
        // result.Stream 包含流式传输的数据块
        log.Println("流数据块:", result.Stream)
    }
}
```

### 从命令行运行 Flow

您可以使用 Genkit 命令行工具（CLI）从命令行运行 Flow：

**终端窗口**
```bash
genkit flow:run menuSuggestionFlow '"French"'
```

对于流式 Flow，您可以通过添加 `-s` 标志将流式输出打印到控制台：

**终端窗口**
```bash
genkit flow:run menuSuggestionFlow '"French"' -s
```

从命令行运行 Flow 对于测试 Flow，或运行那些按需执行的任务（例如，运行一个将文档摄取到向量数据库中的 Flow）非常有用。

### 调试 Flow

将 AI 逻辑封装在 Flow 中的一个优点是，您可以使用 Genkit 开发者界面独立于您的应用来测试和调试 Flow。

开发者界面依赖于 Go 应用的持续运行，即使逻辑已经执行完毕。如果您刚开始使用，并且 Genkit 还不是一个更广泛应用的一部分，请在 `main()` 函数的最后一行添加 `select {}`，以防止应用关闭，这样您就可以在界面中检查它。

要启动开发者界面，请从您的项目目录运行以下命令：

**终端窗口**
```bash
genkit start -- go run .
```

在开发者界面的 **Run** 标签页中，您可以运行项目中定义的任何 Flow：

![Flow 运行器截图](https://cloud.google.com/docs/genkit/images/flow-runner.png)

运行 Flow 后，您可以通过点击 **View trace** (查看追踪) 或查看 **Inspect** (检查) 标签页来检查 Flow 调用的追踪信息。

### 部署 Flow

您可以将您的 Flow 直接部署为 Web API 端点，供您的应用客户端调用。部署在其他几个页面中有详细讨论，但本节简要概述了您的部署选项。

#### net/http 服务器

要使用任何 Go 托管平台（如 Cloud Run）部署 Flow，请使用 `genkit.DefineFlow()` 定义您的 Flow，并使用 `genkit.Handler()` 提供的 Flow 处理器启动一个 `net/http` 服务器：

```go
package main

import (
  "context"
  "log"
  "net/http"

  "github.com/firebase/genkit/go/ai"
  "github.com/firebase/genkit/go/genkit"
  "github.com/firebase/genkit/go/plugins/googlegenai"
  "github.com/firebase/genkit/go/plugins/server"
)

type MenuItem struct {
  Name        string `json:"name"`
  Description string `json:"description"`
}

func main() {
  ctx := context.Background()

  g, err := genkit.Init(ctx, genkit.WithPlugins(&googlegenai.GoogleAI{}))
  if err != nil {
    log.Fatal(err)
  }

  menuSuggestionFlow := genkit.DefineFlow(g, "menuSuggestionFlow",
    func(ctx context.Context, theme string) (MenuItem, error) {
      item, _, err := genkit.GenerateData[MenuItem](ctx, g,
        ai.WithPrompt("Invent a menu item for a %s themed restaurant.", theme),
      )
      return item, err
    })

  mux := http.NewServeMux()
  mux.HandleFunc("POST /menuSuggestionFlow", genkit.Handler(menuSuggestionFlow))
  log.Fatal(server.Start(ctx, "127.0.0.1:3400", mux))
}
```

`server.Start()` 是一个可选的辅助函数，用于启动服务器并管理其生命周期，包括捕获中断信号以方便本地开发，但您也可以使用自己的方法。

要提供代码库中定义的所有 Flow，您可以使用 `genkit.ListFlows()`：

```go
mux := http.NewServeMux()
for _, flow := range genkit.ListFlows(g) {
    mux.HandleFunc("POST /"+flow.Name(), genkit.Handler(flow))
}
log.Fatal(server.Start(ctx, "127.0.0.1:3400", mux))
```

您可以通过如下的 POST 请求来调用一个 Flow 端点：

**终端窗口**
```bash
curl -X POST "http://localhost:3400/menuSuggestionFlow" \
    -H "Content-Type: application/json" -d '{"data": "banana"}'
```

#### 其他服务器框架

您也可以使用其他服务器框架来部署您的 Flow。例如，您只需几行代码就可以使用 Gin：

```go
router := gin.Default()
for _, flow := range genkit.ListFlows(g) {
    router.POST("/"+flow.Name(), func(c *gin.Context) {
        genkit.Handler(flow)(c.Writer, c.Request)
    })
}
log.Fatal(router.Run(":3400"))
```

有关部署到特定平台的信息，请参阅 [Genkit 与 Cloud Run](https://cloud.google.com/docs/genkit/deploy-go)。