### 使用 Dotprompt 管理提示

**提示工程（Prompt engineering）**是您作为应用开发者影响生成式 AI 模型输出的主要方式。例如，在使用 LLMs 时，您可以通过精心设计提示来影响模型响应的语调、格式、长度和其他特性。

您编写这些提示的方式将取决于您所使用的模型；为一个模型编写的提示在用于另一个模型时可能表现不佳。同样，您设置的模型参数（如 temperature、top-k 等）也会因模型不同而对输出产生不同的影响。

要让这三个因素——模型、模型参数和提示——协同工作以产生您想要的输出，通常不是一个简单的过程，往往需要大量的迭代和实验。Genkit 提供了一个名为 **Dotprompt** 的库和文件格式，旨在使这一迭代过程更快、更方便。

Dotprompt 的设计围绕着**“提示即代码”**的前提。您将提示及其所针对的模型和模型参数与您的应用程序代码分开定义。然后，您（或者甚至是一个不参与编写应用程序代码的人）可以使用 Genkit 开发者界面快速迭代提示和模型参数。一旦您的提示按预期工作，您就可以将它们导入到您的应用程序中并使用 Genkit 来运行它们。

您的每个提示定义都存放在一个扩展名为 `.prompt` 的文件中。以下是这些文件外观的示例：

```yaml
---
model: googleai/gemini-1.5-flash
config:
  temperature: 0.9
input:
  schema:
    location: string
    style?: string
    name?: string
  default:
    location: a restaurant
---

你是一位世界上最热情的 AI 助手，目前正在 {{location}} 工作。

向一位客人打招呼{{#if name}}，他叫 {{name}}{{/if}}{{#if style}}，请使用 {{style}} 的风格{{/if}}。
```

在三个破折号之间的部分是 **YAML front matter**，类似于 GitHub Markdown 和 Jekyll 使用的 front matter 格式；文件的其余部分是提示内容，可以选择性地使用 [Handlebars](https://handlebarsjs.com/) 模板。接下来的部分将更详细地介绍构成 `.prompt` 文件的各个部分以及如何使用它们。

#### 开始之前

在阅读此页面之前，您应该熟悉[使用 AI 模型生成内容](https://cloud.google.com/docs/genkit/generate)页面上涵盖的内容。

如果您想运行此页面上的代码示例，请先完成[入门指南](https://cloud.google.com/docs/genkit/get-started-go)中的步骤。所有示例都假定您已在项目中将 Genkit 安装为依赖项。

### 创建提示文件

尽管 Dotprompt 提供了几种不同的创建和加载提示的方式，但它最适合于将提示组织在单个目录（或其子目录）下作为 `.prompt` 文件的项目。本节将向您展示如何使用这种推荐的设置来创建和加载提示。

#### 创建提示目录

Dotprompt 库期望在您的项目根目录下找到一个存放提示的目录，并自动加载在该目录中找到的任何提示。默认情况下，该目录名为 `prompts`。例如，使用默认目录名，您的项目结构可能如下所示：

```
your-project/
├── prompts/
│   └── hello.prompt
├── main.go
├── go.mod
└── go.sum
```

如果您想使用不同的目录，可以在配置 Genkit 时指定它：

```go
// 指定一个名为 llm_prompts 的目录
g, err := genkit.Init(ctx.Background(), ai.WithPromptDir("./llm_prompts"))
```

#### 创建提示文件

创建 `.prompt` 文件有两种方法：使用文本编辑器，或使用开发者界面。

**使用文本编辑器**

如果您想使用文本编辑器创建提示文件，请在您的 `prompts` 目录中创建一个扩展名为 `.prompt` 的文本文件：例如 `prompts/hello.prompt`。

这是一个最简单的提示文件示例：

```yaml
---
model: vertexai/gemini-1.5-flash
---
你是一位世界上最热情的 AI 助手。向用户打招呼并提供帮助。
```

破折号之间的部分是 YAML front matter；文件的其余部分是提示内容，可以选择性地使用 Handlebars 模板。Front matter 部分是可选的，但大多数提示文件至少会包含指定模型的元数据。本页的其余部分将向您展示如何在此基础上，利用 Dotprompt 的功能来增强您的提示文件。

**使用开发者界面**

您也可以使用开发者界面中的模型运行器来创建提示文件。首先，编写应用程序代码，导入 Genkit 库并将其配置为使用您感兴趣的模型插件。例如：

```go
package main

import (
  "context"
  "log"

  "github.com/firebase/genkit/go/ai"
  "github.com/firebase/genkit/go/genkit"
  "github.com/firebase/genkit/go/plugins/googlegenai"
)

func main() {
  g, err := genkit.Init(context.Background(), genkit.WithPlugins(&googlegenai.GoogleAI{}))
  if err != nil {
    log.Fatal(err)
  }

  // 阻塞程序执行结束，以便使用开发者界面。
  select {}
}
```

在同一个项目中加载开发者界面：

**终端窗口**
```bash
genkit start -- go run .
```

在 **Model** 部分，从插件提供的模型列表中选择您想使用的模型。

![Genkit 开发者界面模型运行器](https://cloud.google.com/docs/genkit/images/model-runner.png)

然后，试验提示和配置，直到得到满意的结果。准备好后，按 **Export** 按钮并将文件保存到您的 `prompts` 目录中。

### 运行提示

创建提示文件后，您可以从您的应用程序代码中运行它们，或使用 Genkit 提供的工具来运行。无论您想如何运行提示，首先都需要编写导入 Genkit 库和您感兴趣的模型插件的应用程序代码。例如：

```go
package main

import (
  "context"
  "log"

  "github.com/firebase/genkit/go/ai"
  "github.com/firebase/genkit/go/genkit"
  "github.com/firebase/genkit/go/plugins/googlegenai"
)

func main() {
  g, err := genkit.Init(context.Background(), genkit.WithPlugins(&googlegenai.GoogleAI{}))
  if err != nil {
    log.Fatal(err)
  }

  // 阻塞程序执行结束，以便使用开发者界面。
  select {}
}
```

如果您将提示存储在非默认目录中，请确保在配置 Genkit 时指定它。

#### 从代码中运行提示

要使用一个提示，首先使用 `genkit.LookupPrompt()` 函数加载它：

```go
// 加载名为 'hello' 的提示 (来自 hello.prompt 文件)
helloPrompt := genkit.LookupPrompt(g, "hello")
```

一个可执行的提示具有与 `genkit.Generate()` 类似的选项，其中许多可以在执行时被覆盖，包括输入（请参阅关于指定输入 Schema 的部分）、配置等：

```go
resp, err := helloPrompt.Execute(context.Background(),
  ai.WithModelName("googleai/gemini-2.0-flash"), // 覆盖模型
  ai.WithInput(map[string]any{"name": "John"}), // 提供输入变量
  ai.WithConfig(&googlegenai.GeminiConfig{Temperature: 0.5}), // 覆盖配置
)
```

您传递给提示调用的任何参数都将覆盖在提示文件中指定的相同参数。

有关可用选项的描述，请参阅[使用 AI 模型生成内容](https://cloud.google.com/docs/genkit/generate)。

#### 使用开发者界面

在您优化应用的提示时，可以在 Genkit 开发者界面中运行它们，以便独立于应用程序代码快速迭代提示和模型配置。

从您的项目目录加载开发者界面：

**终端窗口**
```bash
genkit start -- go run .
```

![Genkit 开发者界面提示运行器](https://cloud.google.com/docs/genkit/images/prompt-runner-v2.png)

将提示加载到开发者界面后，您可以使用不同的输入值运行它们，并试验更改提示措辞或配置参数如何影响模型输出。当您对结果满意时，可以点击 **Export prompt** 按钮将修改后的提示保存回您的项目目录。

### 模型配置

在提示文件的 front matter 块中，您可以选择性地为您的提示指定模型配置值：

```yaml
---
model: googleai/gemini-2.0-flash
config:
  temperature: 1.4
  topK: 50
  topP: 0.4
  maxOutputTokens: 400
  stopSequences:
    - '<end>'
    - '<fin>'
---
```

这些值直接映射到可执行提示接受的 `WithConfig()` 选项：

```go
resp, err := helloPrompt.Execute(context.Background(),
    ai.WithConfig(&googlegenai.GeminiConfig{
        Temperature:     1.4,
        TopK:            50,
        TopP:            0.4,
        MaxOutputTokens: 400,
        StopSequences:   []string{"<end>", "<fin>"},
    }))
```

有关可用选项的描述，请参阅[使用 AI 模型生成内容](https://cloud.google.com/docs/genkit/generate)。

### 输入和输出 Schema

您可以通过在 front matter 部分定义输入和输出 Schema 来为您的提示指定它们。这些 Schema 的使用方式与传递给 `genkit.Generate()` 请求或 Flow 定义的 Schema 非常相似：

```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    theme?: string
  default:
    theme: "pirate"
output:
  schema:
    dishname: string
    description: string
    calories: integer
    allergens(array): string
---
为一家 {{theme}} 主题的餐厅设计一道菜单菜品。
```

此代码产生以下结构化输出：

```go
package main

import (
  "context"
  "log"

  "github.com/firebase/genkit/go/ai"
  "github.com/firebase/genkit/go/genkit"
  "github.com/firebase/genkit/go/plugins/googlegenai"
)

func main() {
  ctx := context.Background()
  g, err := genkit.Init(ctx, genkit.WithPlugins(&googlegenai.GoogleAI{}))
  if err != nil {
    log.Fatal(err)
  }

  menuPrompt := genkit.LookupPrompt(g, "menu")
  if menuPrompt == nil {
    log.Fatal("未找到名为 'menu' 的提示")
  }

  resp, err := menuPrompt.Execute(ctx,
    ai.WithInput(map[string]any{"theme": "medieval"}),
  )
  if err != nil {
    log.Fatal(err)
  }

  var output map[string]any
  if err := resp.Output(&output); err != nil {
    log.Fatal(err)
  }

  log.Println(output["dishname"])
  log.Println(output["description"])

  // 阻塞程序执行结束，以便使用开发者界面。
  select {}
}
```

在 `.prompt` 文件中定义 Schema 有多种选择：Dotprompt 自己的 Schema 定义格式 **Picoschema**；标准的 **JSON Schema**；或者作为对您应用程序代码中定义的 Schema 的引用。以下部分将更详细地描述这些选项。

#### Picoschema

上例中的 Schema 是用一种名为 Picoschema 的格式定义的。Picoschema 是一种紧凑的、为 YAML 优化的 Schema 定义格式，它简化了为 LLM 使用场景定义 Schema 最重要属性的过程。这是一个更长的 Schema 示例，它指定了一个应用可能存储的关于一篇文章的信息：

```yaml
schema:
  title: string # string, number, 和 boolean 类型这样定义
  subtitle?: string # 可选字段用 `?` 标记
  draft?: boolean, true 表示草稿状态
  status?(enum, 审批状态): [PENDING, APPROVED]
  date: string, 发布日期，例如 '2024-04-09' # 描述跟在逗号后面
  tags(array, 文章的相关标签): string # 数组用括号表示
  authors(array):
    name: string
    email?: string
  metadata?(object): # 对象也用括号表示
    updatedAt?: string, 最后更新的 ISO 时间戳
    approvedBy?: integer, 审批者 ID
  extra?: any, 任意额外数据
  (*): string, 通配符字段
```

上述 Schema 等同于以下 Go 类型：

```go
type Article struct {
  Title    string   `json:"title"`
  Subtitle string   `json:"subtitle,omitempty" jsonschema:"required=false"`
  Draft    bool     `json:"draft,omitempty"`  // True 表示草稿状态
  Status   string   `json:"status,omitempty" jsonschema:"enum=PENDING,enum=APPROVED"` // 审批状态
  Date     string   `json:"date"`   // 发布日期，例如 '2025-04-07'
  Tags     []string `json:"tags"`   // 文章的相关标签
  Authors  []struct {
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
  } `json:"authors"`
  Metadata struct {
    UpdatedAt  string `json:"updatedAt,omitempty"`  // 最后更新的 ISO 时间戳
    ApprovedBy int    `json:"approvedBy,omitempty"` // 审批者 ID
  } `json:"metadata,omitempty"`
  Extra any `json:"extra"` // 任意额外数据
}
```

Picoschema 支持 `string`, `integer`, `number`, `boolean`, 和 `any` 等标量类型。对象、数组和枚举通过字段名后的括号表示。

由 Picoschema 定义的对象，除非用 `?` 标记为可选，否则所有属性都是必需的，并且不允许额外的属性。当一个属性被标记为可选时，它也被设为可空（nullable），以便为 LLMs 返回 null 而不是省略字段提供更大的灵活性。

在对象定义中，特殊键 `(*)` 可用于声明“通配符”字段定义。这将匹配任何未由显式键提供的额外属性。

#### JSON Schema

Picoschema 不支持完整 JSON Schema 的许多功能。如果您需要更健壮的 Schema，可以提供一个 JSON Schema 来代替：

```yaml
output:
  schema:
    type: object
    properties:
      field1:
        type: number
        minimum: 20
```

{# TODO: 在实现后，讨论在代码中定义 Schema 以在 .prompt 文件中引用的问题。 #}

### 提示模板

`.prompt` 文件中跟在 front matter（如果存在）之后的部分是提示本身，它将被传递给模型。虽然这个提示可以是一个简单的文本字符串，但您通常会希望将用户输入合并到提示中。为此，您可以使用 **Handlebars** 模板语言来指定您的提示。提示模板可以包含引用您的提示输入 Schema 所定义值的占位符。

您在关于输入和输出 Schema 的部分已经看到了这一点：

```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    theme?: string
  default:
    theme: "pirate"
output:
  schema:
    dishname: string
    description: string
    calories: integer
    allergens(array): string
---
为一家 {{theme}} 主题的餐厅设计一道菜单菜品。
```

在这个例子中，当您运行提示时，Handlebars 表达式 `{{theme}}` 会解析为输入中 `theme` 属性的值。要向提示传递输入，请像下面的例子一样调用提示：

```go
menuPrompt = genkit.LookupPrompt(g, "menu")

resp, err := menuPrompt.Execute(context.Background(),
    ai.WithInput(map[string]any{"theme": "medieval"}),
)
```

请注意，因为输入 Schema 将 `theme` 属性声明为可选并提供了默认值，所以您可以省略该属性，提示将使用默认值进行解析。

Handlebars 模板还支持一些有限的逻辑结构。例如，作为提供默认值的替代方案，您可以使用 Handlebars 的 `#if` 辅助函数来定义提示：

```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    theme?: string
---
为一家{{#if theme}} {{theme}} 主题的{{else}}主题{{/if}}餐厅设计一道菜单菜品。
```

在这个例子中，当 `theme` 属性未指定时，提示会渲染为“为一家主题餐厅设计一道菜单菜品。”。

有关所有内置逻辑辅助函数的信息，请参阅 [Handlebars 文档](https://handlebarsjs.com/guide/builtin-helpers.html)。

除了您的输入 Schema 定义的属性外，您的模板还可以引用由 Genkit 自动定义的值。接下来的几节将描述这些自动定义的值以及如何使用它们。

#### 多消息提示

默认情况下，Dotprompt 构建一个具有“user”角色的单条消息。然而，某些提示，例如系统提示，最好表示为多条消息的组合。

`{{role}}` 辅助函数提供了一种直接构建多消息提示的方法：

```yaml
---
model: vertexai/gemini-2.0-flash
input:
  schema:
    userQuestion: string
---
{{role "system"}}
你是一位乐于助人的 AI 助手，非常喜欢谈论食物。试着将食物融入到你所有的对话中。

{{role "user"}}
{{userQuestion}}
```

#### 多模态提示

对于支持多模态输入（例如图像和文本一起）的模型，您可以使用 `{{media}}` 辅助函数：

```yaml
---
model: vertexai/gemini-2.0-flash
input:
  schema:
    photoUrl: string
---
用一个详细的段落描述这张图片：

{{media url=photoUrl}}
```

URL 可以是 `https:` 或 base64 编码的 `data:` URI，用于“内联”图像。在代码中，这将是：

```go
multimodalPrompt = genkit.LookupPrompt(g, "multimodal")

resp, err := multimodalPrompt.Execute(context.Background(),
    ai.WithInput(map[string]any{"photoUrl": "https://example.com/photo.jpg"}),
)
```

另请参阅[使用 AI 模型生成内容](https://cloud.google.com/docs/genkit/generate#multimodal-input)页面上关于构建 `data:` URL 的示例。

#### Partials (可复用模板)

Partials 是可重用的模板，可以包含在任何提示中。对于具有共同行为的相关提示，Partials 特别有用。

加载提示目录时，任何以**下划线 (`_`)** 为前缀的文件都被视为一个 Partial。因此，一个名为 `_personality.prompt` 的文件可能包含：

```handlebars
你应该像一个{{#if style}}{{style}}{{else}}乐于助人的助手{{/if}}那样说话。
```

然后可以将其包含在其他提示中：

```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    name: string
    style?: string
---
{{ role "system" }}
{{>personality style=style}}

{{ role "user" }}
给用户一个友好的问候。

用户名: {{name}}
```

Partials 使用 `{{>PARTIAL的名称 参数...}}` 语法插入。如果没有为 Partial 提供参数，它将使用与父提示相同的上下文执行。

Partials 接受命名参数或代表上下文的单个位置参数。这对于渲染列表成员等任务很有帮助。

`_destination.prompt`
```handlebars
-   {{name}} ({{country}})
```

`chooseDestination.prompt`
```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    destinations(array):
      name: string
      country: string
---
帮助用户在这些度假目的地之间做出选择：

{{#each destinations}}
{{>destination this}}
{{/each}}
```

#### 在代码中定义 Partials

您也可以使用 `genkit.DefinePartial()` 在代码中定义 Partials：

```go
genkit.DefinePartial(g, "personality", "像一个{% verbatim %}{{#if style}}{{style}}{{else}}{% endverbatim %}乐于助人的助手{% verbatim %}{{/if}}{% endverbatim %}那样说话。")
```

在代码中定义的 Partials 在所有提示中都可用。

#### 定义自定义辅助函数

您可以定义自定义辅助函数来在提示内部处理和管理数据。辅助函数使用 `genkit.DefineHelper()` 全局注册：

```go
genkit.DefineHelper(g, "shout", func(input string) string {
    return strings.ToUpper(input)
})
```

一旦定义了辅助函数，您就可以在任何提示中使用它：

```yaml
---
model: googleai/gemini-2.0-flash
input:
  schema:
    name: string
---

你好，{{shout name}}!!!
```

### 提示变体

因为提示文件只是文本，您可以（也应该！）将它们提交到您的版本控制系统中，从而简化了随时间比较更改的过程。通常，提示的微调版本只能在生产环境中与现有版本并排测试才能得到充分验证。Dotprompt 通过其**变体（variants）**功能支持这一点。

要创建一个变体，请创建一个 `[名称].[变体].prompt` 文件。例如，如果您在提示中使用 Gemini 2.0 Flash，但想看看 Gemini 2.5 Pro 是否会表现更好，您可以创建两个文件：

*   `myPrompt.prompt`: “基准”提示
*   `myPrompt.gemini25pro.prompt`: 一个名为 `gemini25pro` 的变体

要使用提示变体，请在加载时指定 `variant` 选项：

```go
// 注意，名称中包含了变体
myPrompt := genkit.LookupPrompt(g, "myPrompt.gemini25pro")
```

变体的名称包含在生成追踪的元数据中，因此您可以在 Genkit 追踪检查器中比较和对比变体之间的实际性能。

### 在代码中定义提示

到目前为止讨论的所有示例都假定您的提示是在单个目录（或其子目录）下的独立 `.prompt` 文件中定义的，并且在运行时可供您的应用访问。Dotprompt 是围绕此设置设计的，其作者认为这是整体上最佳的开发者体验。

然而，如果您的用例不受此设置的良好支持，您也可以使用 `genkit.DefinePrompt()` 函数在代码中定义提示：

```go
type GeoQuery struct {
    CountryCount int `json:"countryCount"`
}

type CountryList struct {
    Countries []string `json:"countries"`
}

geographyPrompt, err := genkit.DefinePrompt(
    g, "GeographyPrompt",
    ai.WithSystem("你是一位地理老师。只在用户询问地理问题时回答。"),
    ai.WithPrompt("告诉我世界上人口最多的 {% verbatim %}{{countryCount}}{% endverbatim %} 个国家。"),
    ai.WithConfig(&googlegenai.GeminiConfig{Temperature: 0.5}),
    ai.WithInputType(GeoQuery{CountryCount: 10}), // 默认值为 10。
    ai.WithOutputType(CountryList{}),
)
if err != nil {
    log.Fatal(err)
}

resp, err := geographyPrompt.Execute(context.Background(), ai.WithInput(GeoQuery{CountryCount: 15}))
if err != nil {
    log.Fatal(err)
}

var list CountryList
if err := resp.Output(&list); err != nil {
    log.Fatal(err)
}

log.Printf("国家: %s", list.Countries)
```

提示也可以被渲染成一个 `GenerateActionOptions`，然后可以对其进行处理并传递给 `genkit.GenerateWithRequest()`：

```go
actionOpts, err := geographyPrompt.Render(ctx, ai.WithInput(GeoQuery{CountryCount: 15}))
if err != nil {
    log.Fatal(err)
}

// 对值进行一些处理...
actionOpts.Config = &googlegenai.GeminiConfig{Temperature: 0.8}

// 没有中间件或流式传输
resp, err := genkit.GenerateWithRequest(ctx, g, actionOpts, nil, nil)
```

请注意，除了 `WithMiddleware()` 之外，所有提示选项都会传递给 `GenerateActionOptions`。如果使用 `Prompt.Render()` 而不是 `Prompt.Execute()`，则必须单独传递 `WithMiddleware()`。