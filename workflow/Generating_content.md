### 使用 AI 模型生成内容

生成式 AI 的核心是 AI 模型。生成式模型最突出的两个例子是大型语言模型（LLMs）和图像生成模型。这些模型接收输入（称为“提示”，通常是文本、图像或两者的结合），并据此产生输出，如文本、图像，甚至音频或视频。

这些模型的输出可以达到惊人的逼真程度：LLMs 生成的文本看起来就像是人类写的，而图像生成模型可以产生非常接近真实照片或人类创作的艺术作品的图像。

此外，事实证明 LLMs 的能力已超越了简单的文本生成：

*   编写计算机程序。
*   为完成一个更大的任务规划子任务。
*   组织非结构化数据。

*   从文本语料库中理解和提取信息数据。
*   根据活动的文本描述，遵循并执行自动化活动。

有许多来自不同提供商的模型可供您选择。每个模型都有其自身的优缺点，一个模型可能在某项任务上表现出色，但在其他任务上则表现不佳。利用生成式 AI 的应用通常可以根据手头的任务使用多个不同的模型而受益。

作为应用开发者，您通常不直接与生成式 AI 模型交互，而是通过以 Web API 形式提供的服务进行交互。尽管这些服务通常功能相似，但它们都通过不同且不兼容的 API 提供这些功能。如果您想使用多个模型服务，就必须使用它们各自专有的 SDK，这些 SDK 之间可能互不兼容。而且，如果您想从一个模型升级到最新、功能最强的模型，您可能需要重新构建整个集成。

Genkit 通过提供一个统一的接口来解决这一挑战，该接口抽象了访问任何潜在生成式 AI 模型服务的细节，并已提供多个预构建的实现。围绕 Genkit 构建您的 AI 应用，不仅简化了首次进行生成式 AI 调用的过程，也使得在出现新模型时，组合多个模型或替换一个模型变得同样简单。

#### 开始之前

如果您想运行此页面上的代码示例，请先完成[入门指南](https://cloud.google.com/docs/genkit/get-started-go)中的步骤。所有示例都假定您已在项目中将 Genkit 安装为依赖项。

### Genkit 支持的模型

Genkit 的设计足够灵活，可以潜在地使用任何生成式 AI 模型服务。其核心库定义了与模型交互的通用接口，而模型插件则定义了与特定模型及其 API 交互的实现细节。

Genkit 团队维护着用于与 Vertex AI、Google Generative AI 和 Ollama 提供的模型交互的插件：

*   **Gemini** 系列 LLMs，通过 Google GenAI 插件提供。
*   **Gemma 3、Llama 4** 以及更多开放模型，通过 Ollama 插件提供（您必须自己托管 Ollama 服务器）。

### 加载和配置模型插件

在您能使用 Genkit 开始生成内容之前，您需要加载并配置一个模型插件。如果您已阅读过入门指南，那么您已经完成了这一步。否则，请在继续之前，参阅入门指南或具体插件的文档并按照其中的步骤操作。

### `genkit.Generate()` 函数

在 Genkit 中，您与生成式 AI 模型交互的主要接口是 `genkit.Generate()` 函数。

最简单的 `genkit.Generate()` 调用指定了您想使用的模型和一个文本提示：

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

  g, err := genkit.Init(ctx,
    genkit.WithPlugins(&googlegenai.GoogleAI{}),
    genkit.WithDefaultModel("googleai/gemini-2.0-flash"),
  )
  if err != nil {
    log.Fatalf("could not initialize Genkit: %v", err)
  }

  resp, err := genkit.Generate(ctx, g,
    // 为一家海盗主题餐厅设计一道菜单菜品。
    ai.WithPrompt("Invent a menu item for a pirate themed restaurant."),
  )
  if err != nil {
    log.Fatalf("could not generate model response: %v", err)
  }

  log.Println(resp.Text())
}
```

当您运行这个简短的示例时，它会打印一些调试信息，然后是 `genkit.Generate()` 调用的输出，通常是 Markdown 文本，如下例所示：

```markdown
## 黑心宝藏 (The Blackheart's Bounty)

**一道丰盛的炖菜，由慢炖牛肉制成，用朗姆酒和糖蜜调味，盛在
一个挖空的炮弹里，配上硬皮面包和一勺浓郁的菠萝莎莎酱。**

**描述：** 这道菜是为了向海盗们在公海上享用的丰盛餐点致敬。
牛肉鲜嫩多汁，融入了朗姆酒和糖蜜的温暖香料味。菠萝莎莎酱增添了
一丝甜味和酸度，平衡了炖菜的浓郁口感。炮弹形状的餐具增添了趣味
和主题感，使这道菜成为任何海盗主题冒险的完美选择。
```

再次运行此脚本，您会得到不同的输出。

前面的代码示例将生成请求发送到了默认模型，这是您在配置 Genkit 实例时指定的。

您也可以为单个 `genkit.Generate()` 调用指定一个模型：

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithModelName("googleai/gemini-2.5-pro"),
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
)
```

模型字符串标识符的格式为 `providerid/modelid`，其中提供商 ID（在此例中为 `googleai`）标识了插件，而模型 ID 是一个插件特定的字符串，用于标识模型的特定版本。

这些示例也说明了一个重点：当您使用 `genkit.Generate()` 进行生成式 AI 模型调用时，更换您想使用的模型只需向 `model` 参数传递一个不同的值。通过使用 `genkit.Generate()` 而不是原生的模型 SDK，您可以更灵活地在应用中使用多个不同的模型，并在未来轻松更换模型。

到目前为止，您只看到了最简单的 `genkit.Generate()` 调用示例。然而，`genkit.Generate()` 也为与生成式模型进行更高级的交互提供了接口，您将在接下来的部分中看到。

### 系统提示

一些模型支持提供一个**系统提示**，它向模型发出指令，告诉模型应该如何响应用户的消息。您可以使用系统提示来指定诸如希望模型扮演的角色、其响应的语调以及响应的格式等特征。

如果您使用的模型支持系统提示，您可以通过 `ai.WithSystem()` 选项提供一个：

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithSystem("你是一位食品行业的营销顾问。"),
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
)
```

对于不支持系统提示的模型，`ai.WithSystem()` 会通过修改请求来模拟系统提示的效果。

### 模型参数

`genkit.Generate()` 函数接受一个 `ai.WithConfig()` 选项，通过它您可以指定控制模型如何生成内容的可选设置：

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithModelName("googleai/gemini-2.0-flash"),
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
  ai.WithConfig(&googlegenai.GeminiConfig{
    MaxOutputTokens: 500,
    StopSequences:   []string{"<end>", "<fin>"},
    Temperature:     0.5,
    TopP:            0.4,
    TopK:            50,
  }),
)
```

具体支持哪些参数取决于单个模型和模型 API。然而，上例中的参数几乎在所有模型中都很常见。以下是这些参数的解释：

#### 控制输出长度的参数

*   **MaxOutputTokens**

    LLMs 操作的单位称为**词元（token）**。一个词元通常（但不一定）映射到一个特定的字符序列。当您向模型传递一个提示时，它首先会把您的提示字符串分词成一个词元序列。然后，LLM 从这个分词后的输入生成一个词元序列。最后，这个词元序列被转换回文本，即您的输出。

    最大输出词元数参数为 LLM 生成的词元数量设置了一个上限。每个模型可能使用不同的分词器，但一个好的经验法则是，一个英文单词约由 2 到 4 个词元组成。

    如前所述，有些词元可能不映射到字符序列。一个例子是，通常有一个表示序列结束的词元：当 LLM 生成此词元时，它会停止生成更多内容。因此，LLM 生成的词元数少于最大值是可能且常见的，因为它生成了“停止”词元。

*   **StopSequences**

    您可以使用此参数设置当生成时表示 LLM 输出结束的词元或词元序列。此处使用的正确值通常取决于模型的训练方式，并且通常由模型插件设置。但是，如果您已提示模型生成另一个停止序列，您可以在此指定它。

    请注意，您指定的是字符序列，而不是词元本身。在大多数情况下，您将指定一个模型分词器会映射到单个词元的字符序列。

#### 控制“创造性”的参数

**temperature**、**top-p** 和 **top-k** 参数共同控制您希望模型有多“具创造性”。本节对这些参数的含义进行了非常简要的解释，但更重要的一点是：这些参数用于调整 LLM 输出的特性。它们的最佳值取决于您的目标和偏好，并且很可能只能通过实验找到。

*   **Temperature** (温度)

    LLMs 本质上是词元预测机。对于给定的词元序列（例如提示），LLM 会为其词汇表中的每个词元预测它接下来出现在序列中的可能性。温度是一个缩放因子，这些预测值在归一化为 0 到 1 之间的概率之前，会先除以这个因子。

    **较低的温度值**（介于 0.0 和 1.0 之间）会放大词元之间可能性的差异，结果是模型更不可能产生它已评估为不太可能的词元。这通常被认为输出的创造性较低。虽然 0.0 在技术上不是一个有效值，但许多模型会将其视为指示模型应表现得具有确定性，并且只考虑最可能的那一个词元。

    **较高的温度值**（大于 1.0）会压缩词元之间可能性的差异，结果是模型变得更有可能产生它之前评估为不太可能的词元。这通常被认为输出的创造性更高。一些模型 API 会强制设定一个最高温度，通常是 2.0。

*   **TopP**

    Top-p 是一个介于 0.0 和 1.0 之间的值，通过指定词元的累积概率来控制您希望模型考虑的可能词元的数量。例如，值为 1.0 意味着考虑所有可能的词元（但仍考虑每个词元的概率）。值为 0.4 意味着只考虑那些概率加起来达到 0.4 的最可能词元，并排除其余词元。

*   **TopK**

    Top-k 是一个整数值，也用于控制您希望模型考虑的可能词元的数量，但这次是通过明确指定最大词元数。指定值为 1 意味着模型应该表现得具有确定性。

#### 试验模型参数

您可以使用**开发者界面**来试验这些参数对不同模型和提示组合所生成输出的影响。使用 `genkit start` 命令启动开发者界面，它会自动加载您项目中配置的插件所定义的所有模型。您可以快速尝试不同的提示和配置值，而无需在代码中反复进行这些更改。

### 将模型与其配置配对

鉴于每个提供商甚至特定模型都可能有自己的配置模式或需要特定的设置，使用 `ai.WithModelName()` 和 `ai.WithConfig()` 分别设置选项可能会容易出错，因为后者与前者之间不是强类型绑定的。

为了将模型与其配置配对，您可以创建一个模型引用，然后将其传递到 `generate` 调用中：

```go
model := googlegenai.GoogleAIModelRef("gemini-2.0-flash", &googlegenai.GeminiConfig{
  MaxOutputTokens: 500,
  StopSequences:   []string{"<end>", "<fin>"},
  Temperature:     0.5,
  TopP:            0.4,
  TopK:            50,
})

resp, err := genkit.Generate(ctx, g,
  ai.WithModel(model),
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
)
if err != nil {
  log.Fatal(err)
}
```

模型引用的构造函数将强制要求提供正确的配置类型，这可以减少不匹配的情况。

### 结构化输出

当将生成式 AI 作为应用程序的一个组件使用时，您通常希望输出的格式不是纯文本。即使您只是生成内容以显示给用户，结构化输出也能让您更美观地呈现它。但对于更高级的生成式 AI 应用，例如以编程方式使用模型的输出，或将一个模型的输出馈送给另一个模型，结构化输出是必须的。

在 Genkit 中，您可以通过在调用 `genkit.Generate()` 时指定一个输出类型来请求模型的结构化输出：

```go
type MenuItem struct {
  Name        string   `json:"name"`
  Description string   `json:"description"`
  Calories    int      `json:"calories"`
  Allergens   []string `json:"allergens"`
}

resp, err := genkit.Generate(ctx, g,
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
  ai.WithOutputType(MenuItem{}),
)
if err != nil {
  // 一个可能的错误是响应不符合该类型。
  log.Fatal(err) 
}
```

模型输出类型是使用 `invopop/jsonschema` 包以 JSON schema 的形式指定的。这提供了运行时类型检查，弥合了静态 Go 类型和生成式 AI 模型不可预测的输出之间的鸿沟。该系统让您可以编写能够依赖于成功 `generate` 调用总是返回符合您 Go 类型的输出的代码。

当您在 `genkit.Generate()` 中指定一个输出类型时，Genkit 在幕后做了几件事：

1.  用关于所选输出格式的额外指导来增强提示。这同时也有一个副作用，即向模型明确指定了您到底想生成什么内容（例如，不仅建议一个菜单项，还要生成描述、过敏原列表等）。
2.  验证输出是否符合 schema。
3.  将模型输出封送（marshal）到一个 Go 类型中。

要从成功的 `generate` 调用中获取结构化输出，请在该类型的空值上调用模型响应的 `Output()` 方法：

```go
var item MenuItem
if err := resp.Output(&item); err != nil {
  log.Fatalf(err)
}

log.Printf("%s (%d 卡路里, %d 种过敏原): %s\n",
  item.Name, item.Calories, len(item.Allergens), item.Description)
```

或者，您可以使用 `genkit.GenerateData()` 进行更简洁的调用：

```go
item, resp, err := genkit.GenerateData[MenuItem](ctx, g,
  ai.WithPrompt("为一家海盗主题餐厅设计一道菜单菜品。"),
)
if err != nil {
  log.Fatal(err)
}

log.Printf("%s (%d 卡路里, %d 种过敏原): %s\n",
  item.Name, item.Calories, len(item.Allergens), item.Description)
```

此函数需要输出类型参数，但会自动设置 `ai.WithOutputType()` 选项并在返回该值之前调用 `ModelResponse.Output()`。

#### 处理错误

请注意，在前面的示例中，`genkit.Generate()` 调用可能会导致错误。当模型未能生成符合 schema 的输出时，就可能发生一种错误。处理此类错误的最佳策略将取决于您的具体用例，但这里有一些通用提示：

*   **尝试不同的模型**。要成功获得结构化输出，模型必须能够生成 JSON 格式的输出。像 Gemini 这样最强大的 LLMs 功能足够强大可以做到这一点；然而，较小的模型，例如您会与 Ollama 一起使用的一些本地模型，可能无法可靠地生成结构化输出，除非它们经过专门训练。
*   **简化 schema**。LLMs 可能难以生成复杂或深度嵌套的类型。如果您无法可靠地生成结构化数据，请尝试使用更清晰的名称、更少的字段或更扁平的结构。
*   **重试 `genkit.Generate()` 调用**。如果您选择的模型只是偶尔无法生成符合规范的输出，您可以像处理网络错误一样处理该错误，并使用某种增量退避策略重试请求。

### 流式传输

在生成大量文本时，您可以通过在生成时就呈现输出——即**流式传输**输出来改善用户体验。流式传输的一个常见例子可以在大多数 LLM 聊天应用中看到：用户可以在模型生成对其消息的响应时阅读它，这提高了应用程序的感知响应速度，并增强了与智能对手聊天的错觉。

在 Genkit 中，您可以使用 `ai.WithStreaming()` 选项进行流式输出：

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithPrompt("为一家海盗主题餐厅建议一份完整的菜单。"),
  ai.WithStreaming(func(ctx context.Context, chunk *ai.ModelResponseChunk) error {
    // 对数据块进行处理...
    log.Println(chunk.Text())
    return nil
  }),
)
if err != nil {
  log.Fatal(err)
}

log.Println(resp.Text())
```

### 多模态输入

到目前为止，您所见的示例都使用文本字符串作为模型提示。虽然这仍然是提示生成式 AI 模型最常见的方式，但许多模型也可以接受其他媒体作为提示。媒体提示最常与文本提示结合使用，指示模型对媒体执行某些操作，例如为图像添加标题或转录音频记录。

接受媒体输入的能力以及您可以使用的媒体类型完全取决于模型及其 API。例如，Gemini 2.0 系列模型可以接受图像、视频和音频作为提示。

要向支持媒体提示的模型提供媒体提示，您不是向 `genkit.Generate()` 传递一个简单的文本提示，而是传递一个由媒体部分和文本部分组成的数组。此示例使用可公开访问的 HTTPS URL 指定图像。

```go
resp, err := genkit.Generate(ctx, g,
  ai.WithModelName("googleai/gemini-2.0-flash"),
  ai.WithMessages(
    NewUserMessage(
      NewMediaPart("image/jpeg", "https://example.com/photo.jpg"),
      NewTextPart("为这张图片写一首诗。"),
    ),
  ),
)
```

您也可以通过将媒体数据编码为数据 URL 来直接传递它。例如：

```go
image, err := ioutil.ReadFile("photo.jpg")
if err != nil {
  log.Fatal(err)
}

resp, err := genkit.Generate(ctx, g,
  ai.WithModelName("googleai/gemini-2.0-flash"),
  ai.WithMessages(
    NewUserMessage(
      NewMediaPart("image/jpeg", "data:image/jpeg;base64," + base64.StdEncoding.EncodeToString(image)),
      NewTextPart("为这张图片写一首诗。"),
    ),
  ),
)
```

所有支持媒体输入的模型都支持数据 URL 和 HTTPS URL。一些模型插件增加了对其他媒体源的支持。例如，Vertex AI 插件还允许您使用 Cloud Storage (`gs://`) URL。

### 后续步骤

#### 了解更多关于 Genkit 的信息

*   作为应用开发者，您影响生成式 AI 模型输出的主要方式是通过提示。阅读[使用 Dotprompt 管理提示](https://cloud.google.com/docs/genkit/prompts)以了解 Genkit 如何帮助您开发有效的提示并在代码库中管理它们。
*   尽管 `genkit.Generate()` 是每个由生成式 AI 驱动的应用的核心，但真实世界的应用通常在调用生成式 AI 模型前后都需要额外的工作。为了反映这一点，Genkit 引入了 **Flows** 的概念，它们像函数一样定义，但增加了额外的功能，如可观测性和简化的部署。要了解更多信息，请参阅[定义 AI 工作流](https://cloud.google.com/docs/genkit/flows)。

#### LLM 的高级用法

您的应用可以使用一些技术来从 LLMs 中获得更多的好处。

*   增强 LLMs 能力的一种方法是向它们提示一个列表，列出它们可以向您请求更多信息或请求您执行某些操作的方式。这被称为**工具调用**或**函数调用**。经过训练以支持此功能的模型可以用特殊格式的响应来回应提示，这向调用应用程序指示它应该执行某些操作并将结果连同原始提示一起发送回 LLM。Genkit 拥有库函数，可以自动化工具调用实现中的提示生成和调用-响应循环元素。要了解更多信息，请参阅[工具调用](https://cloud.google.com/docs/genkit/tools)。
*   **检索增强生成（RAG）**是一种将领域特定信息引入模型输出的技术。这是通过在将提示传递给语言模型之前将相关信息插入提示中来完成的。一个完整的 RAG 实现需要您将多种技术结合在一起：文本嵌入生成模型、向量数据库和大型语言模型。请参阅[检索增强生成（RAG）](https://cloud.google.com/docs/genkit/rag)以了解 Genkit 如何简化协调这些不同元素的过程。