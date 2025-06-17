### 工具调用

工具调用（Tool calling），也称为函数调用（function calling），是一种结构化的方式，它赋予 LLM（大型语言模型）能力，可以向调用它的应用程序发起请求。您定义好希望提供给模型的工具，模型则会在需要时向您的应用发起工具调用请求，以完成您给出的提示。

工具调用的用例通常可以归为以下几类：

*   **让 LLM 访问它未经训练的信息**
    *   频繁变化的信息，例如股票价格或当前天气。
    *   特定于您应用领域的信息，例如产品信息或用户个人资料。
    *   请注意这与检索增强生成（RAG）的重叠之处，RAG 也是一种让 LLM 将事实信息整合到其生成内容中的方法。当您有大量信息，或者与提示最相关的信息不明确时，RAG 是一个更重量级的解决方案。另一方面，如果仅需一个函数调用或数据库查询就能检索到 LLM 需要的信息，那么工具调用则更为合适。

*   **在 LLM 工作流中引入一定程度的确定性**
    *   执行 LLM 自身无法可靠完成的计算。
    *   在特定情况下强制 LLM 生成一字不差的文本，例如在回答关于应用服务条款的问题时。

*   **由 LLM 发起并执行一个动作**
    *   在由 LLM 驱动的家庭助手中开关灯。
    *   在由 LLM 驱动的餐厅代理中预订餐桌。

#### 开始之前

如果您想运行本页的代码示例，请先完成[入门指南](https://cloud.google.com/docs/genkit/get-started-go)中的步骤。所有示例都假定您已经设置了一个安装了 Genkit 依赖项的项目。

本页讨论的是 Genkit 模型抽象的一项高级功能，因此在深入了解之前，您应该熟悉[使用 AI 模型生成内容](https://cloud.google.com/docs/genkit/generate)页面上的内容。您还应该熟悉 Genkit 用于定义输入和输出 Schema 的系统，这在 [Flows](https://cloud.google.com/docs/genkit/flows) 页面上有讨论。

### 工具调用概述

从高层次来看，一个典型的与 LLM 进行工具调用的交互过程如下：

1.  调用方应用向 LLM 发出一个带有请求的提示，并在提示中包含一个 LLM 可用于生成响应的工具列表。
2.  LLM 要么生成一个完整的响应，要么以特定格式生成一个工具调用请求。
3.  如果调用方收到一个完整的响应，则请求完成，交互结束；但如果调用方收到一个工具调用请求，它会执行任何适当的逻辑，并向 LLM 发送一个新请求，其中包含原始提示（或其变体）以及工具调用的结果。
4.  LLM 像步骤 2 一样处理这个新提示。

为了让这个过程能正常工作，必须满足几个要求：

*   **模型必须经过训练**，以便在需要时发起工具请求来完成提示。大多数通过 Web API 提供的大型模型（如 Gemini）都可以做到这一点，但更小和更专业的模型通常不能。如果您尝试向不支持工具调用的模型提供工具，Genkit 将会抛出错误。
*   **调用方应用必须以模型期望的格式**向模型提供工具定义。
*   **调用方应用必须提示模型**以应用期望的格式生成工具调用请求。

### 使用 Genkit 进行工具调用

Genkit 为支持工具调用的模型提供了一个统一的接口。每个模型插件都确保满足上一节中提到的后两个标准，并且 `genkit.Generate()` 函数会自动执行前面描述的工具调用循环。

#### 模型支持

工具调用支持取决于模型、模型 API 和 Genkit 插件。请查阅相关文档以确定是否可能支持工具调用。此外：

*   如果您尝试向不支持它的模型提供工具，Genkit 将会抛出错误。
*   如果插件导出了模型引用，`ModelInfo.Supports.Tools` 属性将指示它是否支持工具调用。

#### 定义工具

使用 `genkit.DefineTool()` 函数来编写工具定义：

```go
package main

import (
  "context"
  "fmt"
  "log"

  "github.com/firebase/genkit/go/ai"
  "github.com/firebase/genkit/go/genkit"
  "github.com/firebase/genkit/go/plugins/googlegenai"
)

// 定义工具的输入结构体
type WeatherInput struct {
  // jsonschema_description 用于告诉模型这个字段的用途
  Location string `json:"location" jsonschema_description:"要查询天气的位置"`
}

func main() {
  ctx := context.Background()

  g, err := genkit.Init(ctx,
    genkit.WithPlugins(&googlegenai.GoogleAI{}),
    genkit.WithDefaultModel("googleai/gemini-1.5-flash"), // 更新了模型名称
  )
  if err != nil {
    log.Fatalf("Genkit 初始化失败: %v", err)
  }

  genkit.DefineTool(
    g, "getWeather", "获取给定位置的当前天气",
    func(ctx context.Context, input WeatherInput) (string, error) {
      // 在这里，我们通常会进行 API 调用或数据库查询。
      // 在这个例子中，我们只返回一个固定的值。
      log.Printf("工具 'getWeather' 被调用，位置: %s", input.Location)
      return fmt.Sprintf("%s 当前的天气是 17°C，晴。", input.Location), nil
    })
}
```

这里的语法看起来就像 `genkit.DefineFlow()` 的语法；但是，您**必须**编写一个描述。请特别注意描述的措辞和详细程度，因为它对于 LLM 能否决定并恰当使用该工具至关重要。

#### 使用工具

在您的提示中包含已定义的工具以生成内容。

**使用 `genkit.Generate()`:**

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithPrompt("旧金山的天气怎么样？"),
  ai.WithTools(getWeatherTool),
)
```

**使用 `genkit.DefinePrompt()`:**

```go
weatherPrompt, err := genkit.DefinePrompt(g, "weatherPrompt",
  ai.WithPrompt("{% verbatim %}{{location}}{% endverbatim %} 的天气怎么样？"),
  ai.WithTools(getWeatherTool),
)
if err != nil {
  log.Fatal(err)
}

resp, err := weatherPrompt.Execute(ctx,
  ai.WithInput(map[string]any{"location": "旧金山"}),
)
```

**使用 `.prompt` 文件:**

创建一个名为 `prompts/weatherPrompt.prompt` 的文件（假设使用默认的提示目录）：

```yaml
---
system: "使用你拥有的工具来回答问题。"
tools: [getWeather]
input:
  schema:
    location: string
---

{{location}} 的天气怎么样？
```

然后在您的 Go 代码中执行它：

```go
// 假设 weatherPrompt.prompt 文件存在于 ./prompts 目录中。
weatherPrompt := genkit.LookupPrompt(g, "weatherPrompt")
if weatherPrompt == nil {
  log.Fatal("未找到名为 'weatherPrompt' 的提示")
}

resp, err := weatherPrompt.Execute(ctx,
  ai.WithInput(map[string]any{"location": "旧金山"}),
)
```

如果 LLM 需要使用 `getWeather` 工具来回答提示，Genkit 将自动处理这次工具调用。

### 显式处理工具调用

如果您想完全控制这个工具调用循环，例如为了应用更复杂的逻辑，可以将 `WithReturnToolRequests()` 选项设置为 `true`。这样一来，确保所有工具请求都得到满足就成了您的责任：

```go
getWeatherTool := genkit.DefineTool(
    g, "getWeather", "获取给定位置的当前天气",
    func(ctx context.Context, input WeatherInput) (string, error) {
        // 工具实现...
        return "晴天", nil
    },
)

// 第一步：生成请求，但要求返回工具请求而不是自动执行
resp, err := genkit.Generate(ctx, g,
    ai.WithPrompt("旧金山的天气怎么样？"),
    ai.WithTools(getWeatherTool),
    ai.WithReturnToolRequests(true), // 设置为 true
)
if err != nil {
    log.Fatal(err)
}

// 第二步：手动处理所有工具请求
parts := []*ai.Part{}
for _, req := range resp.ToolRequests() {
    tool := genkit.LookupTool(g, req.Name)
    if tool == nil {
        log.Fatalf("未找到工具 %q", req.Name)
    }

    // 执行工具
    output, err := tool.RunRaw(ctx, req.Input)
    if err != nil {
        log.Fatalf("工具 %q 执行失败: %v", tool.Name(), err)
    }

    // 将工具的输出打包成一个 ToolResponsePart
    parts = append(parts,
        ai.NewToolResponsePart(&ai.ToolResponse{
            Name:   req.Name,
            Ref:    req.Ref,
            Output: output,
        }))
}

// 第三步：将历史消息和工具的响应一起发回给模型
resp, err = genkit.Generate(ctx, g,
    ai.WithMessages(append(resp.History(), ai.NewMessage(ai.RoleTool, nil, parts...))...),
)
if err != nil {
    log.Fatal(err)
}

// 此时 resp 包含最终的文本响应
log.Println(resp.Text())
```