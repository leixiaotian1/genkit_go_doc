### 评估 (Evaluation)

评估是一种测试形式，可帮助您验证 LLM 的响应并确保它们满足您的质量标准。

Genkit 通过插件支持第三方评估工具，并结合了强大的可观测性功能，可深入了解您由 LLM 驱动的应用程序的运行时状态。Genkit 工具可帮助您自动提取数据，包括输入、输出和中间步骤的信息，以评估 LLM 响应的端到端质量，并了解系统构建块的性能。

#### 评估类型

Genkit 支持两种类型的评估：

1.  **基于推理的评估（Inference-based evaluation）**：此类型的评估针对一组预先确定的输入运行，评估相应输出的质量。
    *   这是最常见的评估类型，适用于大多数用例。此方法测试系统在每次评估运行中的实际输出。
    *   您可以通过目视检查结果来手动进行质量评估。或者，您可以使用**评估指标（evaluation metric）**来自动化评估过程。

2.  **原始评估（Raw evaluation）**：此类型的评估直接评估输入的质量，不进行任何推理。此方法通常与使用指标的自动化评估一起使用。评估所需的所有字段（例如，输入、上下文、输出和参考）必须存在于输入数据集中。当您的数据来自外部源（例如，从您的生产追踪中收集）并且您希望对收集的数据质量进行客观测量时，这非常有用。
    *   有关更多信息，请参阅本页的[高级用法](#高级用法)部分。

本节说明了如何使用 Genkit 执行**基于推理的评估**。

### 快速入门

执行以下步骤以快速开始使用 Genkit。

#### 设置

使用现有的 Genkit 应用或按照我们的[入门指南](https://cloud.google.com/docs/genkit/get-started-go)创建一个新应用。

添加以下代码来定义一个简单的 RAG 应用程序以进行评估。在本指南中，我们使用一个总是返回相同文档的虚拟检索器。

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

func main() {
    ctx := context.Background()

    // 初始化 Genkit
    g, err := genkit.Init(ctx,
        genkit.WithPlugins(&googlegenai.GoogleAI{}),
        genkit.WithDefaultModel("googleai/gemini-2.0-flash"),
    )
    if err != nil {
        log.Fatalf("Genkit 初始化错误: %v", err)
    }

    // 总是返回相同事实的虚拟检索器
    dummyRetrieverFunc := func(ctx context.Context, req *ai.RetrieverRequest) (*ai.RetrieverResponse, error) {
        facts := []string{
            "狗是人类最好的朋友",
            "狗是从狼进化和驯化而来的",
        }
        // 仅将事实作为文档返回。
        var docs []*ai.Document
        for _, fact := range facts {
            docs = append(docs, ai.DocumentFromText(fact, nil))
        }
        return &ai.RetrieverResponse{Documents: docs}, nil
    }
    factsRetriever := genkit.DefineRetriever(g, "local", "dogFacts", dummyRetrieverFunc)

    m := googlegenai.GoogleAIModel(g, "gemini-2.0-flash")
    if m == nil {
        log.Fatal("未能找到模型")
    }

    // 一个简单的问答 flow
    genkit.DefineFlow(g, "qaFlow", func(ctx context.Context, query string) (string, error) {
        factDocs, err := ai.Retrieve(ctx, factsRetriever, ai.WithTextDocs(query))
        if err != nil {
            return "", fmt.Errorf("检索失败: %w", err)
        }
        llmResponse, err := genkit.Generate(ctx, g,
            ai.WithModelName("googleai/gemini-2.0-flash"),
            ai.WithPrompt("用给定的上下文回答这个问题: %s", query),
            ai.WithDocs(factDocs.Documents...)
        )
        if err != nil {
            return "", fmt.Errorf("生成失败: %w", err)
        }
        return llmResponse.Text(), nil
    })
}
```

您可以选择性地向您的应用程序添加评估指标，以便在评估时使用。本指南使用来自 `evaluators` 包的 `EvaluatorRegex` 指标。

```go
import (
    "github.com/firebase/genkit/go/plugins/evaluators"
)

func main() {
    // ...

    metrics := []evaluators.MetricConfig{
        {
            MetricType: evaluators.EvaluatorRegex,
        },
    }

    // 初始化 Genkit
    g, err := genkit.Init(ctx,
        genkit.WithPlugins(
            &googlegenai.GoogleAI{},
            &evaluators.GenkitEval{Metrics: metrics}, // 添加此插件
        ),
        genkit.WithDefaultModel("googleai/gemini-2.0-flash"),
    )
    // ...
}
```

> **注意**：确保 `evaluators` 包已安装在您的 go 项目中：
> **终端窗口**
> ```bash
> go get github.com/firebase/genkit/go/plugins/evaluators
> ```

启动您的 Genkit 应用程序。

**终端窗口**
```bash
genkit start -- go run main.go
```

#### 创建一个数据集

创建一个数据集来定义我们想要用于评估我们 flow 的示例。

1.  转到开发者界面的 `http://localhost:4000`，然后单击 **Datasets** 按钮打开数据集页面。

2.  单击 **Create Dataset** 按钮打开创建数据集对话框。
    a. 为您的新数据集提供一个 `datasetId`。本指南使用 `myFactsQaDataset`。
    b. 选择 **Flow dataset** 类型。
    c. 将验证目标字段留空，然后单击 **Save**。

3.  您的新数据集页面出现，显示一个空的数据集。按照以下步骤向其添加示例：
    a. 单击 **Add example** 按钮打开示例编辑器面板。
    b. 只有 **Input** 字段是必需的。在 **Input** 字段中输入 "Who is man's best friend?"，然后单击 **Save** 将示例添加到您的数据集中。
    > 如果您配置了 `EvaluatorRegex` 指标并想尝试一下，您需要指定一个 **Reference** 字符串，其中包含要与输出匹配的模式。对于前面的输入，将 **Reference** 输出文本设置为 `(?i)dog`，这是一个不区分大小写的正则表达式模式，用于匹配 flow 输出中的单词“dog”。
    c. 重复步骤 (a) 和 (b) 几次以添加更多示例。本指南将以下示例输入添加到数据集中：
        *   "Can I give milk to my cats?"
        *   "From which animals did dogs evolve?"
    > 如果您正在使用正则表达式评估器，请使用相应的参考字符串：
    > *   `(?i)don't know`
    > *   `(?i)wolf|wolves`
    >
    > 请注意，这是一个精心设计的示例，正则表达式评估器可能不是评估 `qaFlow` 响应的正确选择。但是，本指南可以应用于您选择的任何 Genkit Go 评估器。

到此步骤结束时，您的数据集应该有 3 个示例，其值如上所述。

#### 运行评估并查看结果

1.  要开始评估 flow，请在您的数据集页面上单击 **Run new evaluation** 按钮。您也可以从 **Evaluations** 选项卡开始新的评估。

2.  选择 **Flow** 单选按钮以评估一个 flow。

3.  选择 `qaFlow` 作为要评估的目标 flow。

4.  选择 `myFactsQaDataset` 作为用于评估的目标数据集。

5.  如果您使用 Genkit 插件安装了评估器指标，您可以在此页面上看到这些指标。选择您想在此评估运行中使用的指标。这完全是可选的：省略此步骤仍将在评估运行中返回结果，但没有任何关联的指标。
    > 如果您没有提供任何参考值并且正在使用 `EvaluatorRegex` 指标，您的评估将会失败，因为此指标需要设置参考。

6.  单击 **Run evaluation** 开始评估。根据您正在测试的 flow，这可能需要一些时间。评估完成后，会出现一条成功消息，其中包含一个查看结果的链接。单击该链接转到**评估详情**页面。

您可以在此页面上看到评估的详细信息，包括原始输入、提取的上下文和指标（如果有）。

### 核心概念

#### 术语

了解以下术语可以帮助您正确理解本页提供的信息：

*   **评估 (Evaluation)**：评估是评估系统性能的过程。在 Genkit 中，这样的系统通常是一个 Genkit 原语，例如一个 flow 或一个模型。评估可以是自动的或手动的（人工评估）。
*   **批量推理 (Bulk inference)**：推理是在一个 flow 或模型上运行一个输入以获得相应输出的行为。批量推理涉及同时对多个输入执行推理。
*   **指标 (Metric)**：评估指标是对推理进行评分的标准。示例包括准确性、忠实度、恶意性、输出是否为英语等。
*   **数据集 (Dataset)**：数据集是用于基于推理的评估的示例集合。数据集通常由 **Input** 和可选的 **Reference** 字段组成。**Reference** 字段不影响评估的推理步骤，但它会逐字传递给任何评估指标。在 Genkit 中，您可以通过开发者界面创建数据集。Genkit 中有两种类型的数据集：**Flow 数据集**和 **Model 数据集**。

#### 支持的评估器

Genkit 支持多种评估器，一些是内置的，另一些是外部提供的。

**Genkit 评估器**

Genkit 包含少量内置评估器，从 JS 评估器插件移植而来，以帮助您入门：

*   `EvaluatorDeepEqual` — 检查生成的输出是否与提供的参考输出深度相等。
*   `EvaluatorRegex` — 检查生成的输出是否与参考字段中提供的正则表达式匹配。
*   `EvaluatorJsonata` — 检查生成的输出是否与参考字段中提供的 [JSONATA](https://jsonata.org/) 表达式匹配。

### 高级用法

除了其基本功能外，Genkit 还为某些评估用例提供了高级支持。

#### 使用 CLI 进行评估

Genkit CLI 提供了丰富的 API 用于执行评估。这在开发者界面不可用的环境中（例如在 CI/CD 工作流中）特别有用。

Genkit CLI 提供 3 个主要的评估命令：`eval:flow`、`eval:extractData` 和 `eval:run`。

**`eval:flow` 命令**

`eval:flow` 命令在输入数据集上运行基于推理的评估。此数据集可以作为 JSON 文件提供，也可以通过引用您 Genkit 运行时中的现有数据集来提供。

**终端窗口**
```bash
# 引用一个现有的数据集
genkit eval:flow qaFlow --input myFactsQaDataset

# 或者，使用一个文件中的数据集
genkit eval:flow qaFlow --input testInputs.json
```

> **注意**：在运行这些 CLI 命令之前，请确保您已启动 genkit 应用。
> **终端窗口**
> ```bash
> genkit start -- go run main.go
> ```

在这里，`testInputs.json` 应该是一个对象数组，其中包含一个 `input` 字段和一个可选的 `reference` 字段，如下所示：

```json
[
  {
    "input": "法语中奶酪这个词是什么？"
  },
  {
    "input": "什么绿色蔬菜看起来像花椰菜？",
    "reference": "Broccoli"
  }
]
```

如果您的 flow 需要身份验证，您可以使用 `--context` 参数指定它：

**终端窗口**
```bash
genkit eval:flow qaFlow --input testInputs.json --context '{"auth": {"email_verified": true}}'
```

默认情况下，`eval:flow` 和 `eval:run` 命令使用所有可用的指标进行评估。要在配置的评估器的子集上运行，请使用 `--evaluators` 标志并提供一个以逗号分隔的评估器名称列表：

**终端窗口**
```bash
genkit eval:flow qaFlow --input testInputs.json --evaluators=genkitEval/regex,genkitEval/jsonata
```

您可以在开发者界面的 `localhost:4000/evaluate` 查看您的评估运行结果。

**`eval:extractData` 和 `eval:run` 命令**

为了支持原始评估，Genkit 提供了从追踪中提取数据并在提取的数据上运行评估指标的工具。例如，当您使用不同的框架进行评估或从不同环境收集推理以在本地测试输出质量时，这非常有用。

您可以批量运行您的 Genkit flow 并从结果追踪中提取一个评估数据集。原始评估数据集是评估指标的输入集合，不运行任何先前的推理。

1.  在您的测试输入上运行您的 flow：
    **终端窗口**
    ```bash
    genkit flow:batchRun qaFlow testInputs.json
    ```

2.  提取评估数据：
    **终端窗口**
    ```bash
    genkit eval:extractData qaFlow --maxRows 2 --output factsEvalDataset.json
    ```

导出的数据格式与前面介绍的数据集格式不同。这是因为此数据旨在直接与评估指标一起使用，没有任何推理步骤。以下是提取数据的语法。

```typescript
Array<{
  "testCaseId": string,
  "input": any,
  "output": any,
  "context": any[],
  "traceIds": string[],
}>;
```

数据提取器会自动定位检索器并将生成的文档添加到 `context` 数组中。您可以使用 `eval:run` 命令在此提取的数据集上运行评估指标。

**终端窗口**
```bash
genkit eval:run factsEvalDataset.json
```

默认情况下，`eval:run` 会针对所有配置的评估器运行，与 `eval:flow` 一样，`eval:run` 的结果会显示在开发者界面的评估页面上，位于 `localhost:4000/evaluate`。