### 检索增强生成 (RAG)

Genkit 提供了有助于您构建检索增强生成（Retrieval-Augmented Generation, RAG）工作流的抽象，以及提供了与相关工具集成的插件。

#### 什么是 RAG？

检索增强生成是一种将外部信息源整合到 LLM 响应中的技术。能够这样做非常重要，因为尽管 LLM 通常是在广泛的资料上进行训练的，但 LLM 的实际应用往往需要特定的领域知识（例如，您可能希望使用 LLM 来回答客户关于贵公司产品的问题）。

一种解决方案是使用更具体的数据对模型进行**微调（fine-tune）**。然而，无论是在计算成本方面，还是在准备足够训练数据所需的工作量方面，这种方法都可能非常昂贵。

相比之下，RAG 的工作原理是在将提示传递给模型时，将外部数据源整合到提示中。例如，您可以想象提示“Bart 和 Lisa 是什么关系？”可能会通过在前面添加一些相关信息而被扩展（“增强”），从而变成提示：“Homer 和 Marge 的孩子名叫 Bart、Lisa 和 Maggie。Bart 和 Lisa 是什么关系？”

这种方法有几个优点：

*   **更具成本效益**，因为您不必重新训练模型。
*   您可以**持续更新您的数据源**，LLM 可以立即利用更新后的信息。
*   您现在有可能在 LLM 的响应中**引用参考资料**。

另一方面，使用 RAG 自然意味着更长的提示，而一些 LLM API 服务会根据您发送的每个输入词元（token）收费。最终，您必须为您的应用评估成本的权衡。

RAG 是一个非常广泛的领域，有许多不同的技术用于实现最佳质量的 RAG。核心的 Genkit 框架提供了两个主要的抽象来帮助您实现 RAG：

*   **Indexer（索引器）**：将文档添加到“索引”中。
*   **Embedder（嵌入器）**：将文档转换为向量表示。
*   **Retriever（检索器）**：根据查询从“索引”中检索文档。

这些定义是故意写得比较宽泛的，因为 Genkit 对于“索引”是什么，或者究竟如何从中检索文档，并没有固定的看法。Genkit 只提供了一个 `Document` 格式，其他一切都由检索器或索引器的实现提供商定义。

#### Indexer（索引器）

索引负责以一种可以根据特定查询快速检索相关文档的方式来跟踪您的文档。这通常是通过**向量数据库**来实现的，它使用称为**嵌入（embeddings）**的多维向量来索引您的文档。文本嵌入（以一种不透明的方式）代表了文本段落所表达的概念；这些嵌入是使用专门的机器学习模型生成的。通过使用其嵌入来索引文本，向量数据库能够将概念上相关的文本聚集在一起，并检索与新的文本字符串（即查询）相关的文档。

在您为了生成内容而检索文档之前，您需要将它们**摄取（ingest）**到您的文档索引中。一个典型的摄取工作流会执行以下操作：

1.  将大文档分割成更小的文档块，以便只有相关的部分被用来增强您的提示——这个过程称为**“分块（chunking）**”。这是必要的，因为许多 LLM 的上下文窗口有限，将整个文档包含在提示中是不切实际的。
    *   Genkit 不提供内置的分块库；但是，有与 Genkit 兼容的开源库可用。

2.  为每个块生成嵌入。根据您使用的数据库，您可能会使用嵌入生成模型明确地执行此操作，或者您可能会使用数据库提供的嵌入生成器。

3.  将文本块及其索引添加到数据库中。

如果您处理的是稳定的数据源，您可能很少运行或只运行一次摄取工作流。另一方面，如果您处理的是经常变化的数据，您可能需要持续运行摄取工作流（例如，在一个 Cloud Firestore 触发器中，每当文档更新时就运行）。

#### Embedder（嵌入器）

嵌入器是一个函数，它接收内容（文本、图像、音频等）并创建一个编码了原始内容语义的数字向量。如上所述，嵌入器在索引过程中被利用。但是，它们也可以独立使用，以在没有索引的情况下创建嵌入。

#### Retriever（检索器）

检索器是一个封装了与任何类型文档检索相关的逻辑的概念。最流行的检索案例通常包括从向量存储中检索。然而，在 Genkit 中，检索器可以是任何返回数据的函数。

要创建一个检索器，您可以使用提供的实现之一或创建自己的实现。

### 支持的索引器、检索器和嵌入器

Genkit 通过其插件系统提供索引器和检索器支持。以下插件是官方支持的：

*   [Pinecone](https://www.pinecone.io/) 云向量数据库

此外，Genkit 通过预定义的代码模板支持以下向量存储，您可以根据您的数据库配置和 schema 对其进行自定义：

*   [PostgreSQL with pgvector](https://github.com/pgvector/pgvector)

嵌入模型支持通过以下插件提供：

| 插件 | 模型 |
| :--- | :--- |
| Google Generative AI | Text embedding |

### 定义一个 RAG 工作流

以下示例展示了您如何将一组餐厅菜单的 PDF 文档摄取到一个向量数据库中，并在一个确定哪些食品可用的工作流中检索它们。

#### 安装依赖

在这个例子中，我们将使用来自 `langchaingo` 的 `textsplitter` 库和 `ledongthuc/pdf` PDF 解析库：

**终端窗口**
```bash
go get github.com/tmc/langchaingo/textsplitter
go get github.com/ledongthuc/pdf
```

#### 定义一个索引器

以下示例展示了如何创建一个索引器来摄取一组 PDF 文档并将其存储在本地向量数据库中。

它使用了 Genkit 开箱即用的基于文件的本地向量相似性检索器，用于简单的测试和原型设计。**请勿在生产环境中使用此检索器。**

**创建索引器**
```go
// 导入 Genkit 的基于文件的向量检索器（请勿在生产中使用）
import "github.com/firebase/genkit/go/plugins/localvec"

// Vertex AI 提供了 text-embedding-004 嵌入模型
import "github.com/firebase/genkit/go/plugins/vertexai"

ctx := context.Background()

g, err := genkit.Init(ctx, genkit.WithPlugins(&vertexai.VertexAI{}))
if err != nil {
  log.Fatal(err)
}

if err = localvec.Init(); err != nil {
  log.Fatal(err)
}

menuPDFIndexer, _, err := localvec.DefineIndexerAndRetriever(g, "menuQA",
    localvec.Config{Embedder: vertexai.VertexAIEmbedder(g, "text-embedding-004")})
if err != nil {
  log.Fatal(err)
}
```

**创建分块配置**

此示例使用 `textsplitter` 库，它提供了一个简单的文本分割器，可将文档分解为可以进行向量化的段落。

以下定义配置了分块函数，使其返回 200 个字符的文档段落，并且块之间有 20 个字符的重叠。

```go
splitter := textsplitter.NewRecursiveCharacter(
    textsplitter.WithChunkSize(200),
    textsplitter.WithChunkOverlap(20),
)
```

关于此库的更多分块选项，可以在 `langchaingo` 文档中找到。

**定义您的索引器工作流**
```go
genkit.DefineFlow(
    g, "indexMenu",
    func(ctx context.Context, path string) (any, error) {
        // 从 PDF 中提取纯文本。将逻辑包装在 Run 中，
        // 以便它在您的追踪中显示为一个步骤。
        pdfText, err := genkit.Run(ctx, "extract", func() (string, error) {
            return readPDF(path)
        })
        if err != nil {
            return nil, err
        }

        // 将文本分割成块。将逻辑包装在 Run 中，
        // 以便它在您的追踪中显示为一个步骤。
        docs, err := genkit.Run(ctx, "chunk", func() ([]*ai.Document, error) {
            chunks, err := splitter.SplitText(pdfText)
            if err != nil {
                return nil, err
            }

            var docs []*ai.Document
            for _, chunk := range chunks {
                docs = append(docs, ai.DocumentFromText(chunk, nil))
            }
            return docs, nil
        })
        if err != nil {
            return nil, err
        }

        // 将块添加到索引中。
        err = ai.Index(ctx, menuPDFIndexer, ai.WithDocs(docs...))
        return nil, err
    },
)

// 从 PDF 提取纯文本的辅助函数。摘自：
// https://github.com/ledongthuc/pdf
func readPDF(path string) (string, error) {
    f, r, err := pdf.Open(path)
    if f != nil {
        defer f.Close()
    }
    if err != nil {
        return "", err
    }

    reader, err := r.GetPlainText()
    if err != nil {
        return "", err
    }

    bytes, err := io.ReadAll(reader)
    if err != nil {
        return "", err
    }

    return string(bytes), nil
}
```

**运行索引器工作流**

**终端窗口**
```bash
genkit flow:run indexMenu "'menu.pdf'"
```

运行 `indexMenu` 工作流后，向量数据库将被植入文档，并准备好在带有检索步骤的 Genkit 工作流中使用。

#### 定义一个带有检索功能的工作流

以下示例展示了您如何在 RAG 工作流中使用检索器。与索引器示例一样，此示例使用 Genkit 的基于文件的向量检索器，**您不应在生产环境中使用它。**

```go
ctx := context.Background()

g, err := genkit.Init(ctx, genkit.WithPlugins(&vertexai.VertexAI{}))
if err != nil {
    log.Fatal(err)
}

if err = localvec.Init(); err != nil {
    log.Fatal(err)
}

model := vertexai.VertexAIModel(g, "gemini-1.5-flash")

_, menuPdfRetriever, err := localvec.DefineIndexerAndRetriever(
    g, "menuQA", localvec.Config{Embedder: vertexai.VertexAIEmbedder(g, "text-embedding-004")},
)
if err != nil {
    log.Fatal(err)
}

genkit.DefineFlow(
  g, "menuQA",
  func(ctx context.Context, question string) (string, error) {
    // 检索与用户问题相关的文本。
    resp, err := ai.Retrieve(ctx, menuPdfRetriever, ai.WithTextDocs(question))
    if err != nil {
        return "", err
    }

    // 调用 Generate，在您的提示中包含菜单信息。
    return genkit.GenerateText(ctx, g,
        ai.WithModel(model),
        ai.WithDocs(resp.Documents...), // 将检索到的文档作为上下文
        ai.WithSystem(`
你正在扮演一个乐于助人的 AI 助手，可以回答关于 Genkit Grub Pub 菜单上
食物的问题。
只使用提供的上下文来回答问题。如果你不知道，不要编造答案。不要在菜单上
添加或更改项目。`),
        ai.WithPrompt(question),
  )
})
```

### 编写您自己的索引器和检索器

您也可以创建自己的检索器。如果您的文档管理在 Genkit 不支持的文档存储中（例如：MySQL、Google Drive 等），这将非常有用。Genkit SDK 提供了灵活的方法，让您可以提供用于获取文档的自定义代码。

您还可以定义在 Genkit 中现有检索器之上构建的自定义检索器，并在其上应用高级 RAG 技术（例如重新排序或提示扩展）。

例如，假设您有一个想要使用的自定义重新排序函数。以下示例定义了一个自定义检索器，它将您的函数应用于前面定义的菜单检索器：

```go
type CustomMenuRetrieverOptions struct {
    K          int // 最终返回的数量
    PreRerankK int // 重新排序前的数量
}

advancedMenuRetriever := genkit.DefineRetriever(
    g, "custom", "advancedMenuRetriever",
    func(ctx context.Context, req *ai.RetrieverRequest) (*ai.RetrieverResponse, error) {
        // 处理使用我们自定义类型传递的选项。
        opts, _ := req.Options.(CustomMenuRetrieverOptions)
        // 当字段未定义或 req.Options 不是 CustomMenuRetrieverOptions 时，
        // 将字段设置为默认值。
        if opts.K == 0 {
            opts.K = 3
        }
        if opts.PreRerankK == 0 {
            opts.PreRerankK = 10
        }

        // 像简单情况下一样调用检索器。
        resp, err := ai.Retrieve(ctx, menuPdfRetriever,
            ai.WithDocs(req.Query),
            ai.WithConfig(localvec.RetrieverOptions{K: opts.PreRerankK}),
        )
        if err != nil {
            return nil, err
        }

        // 使用您的自定义函数对返回的文档进行重新排序。
        rerankedDocs := rerank(resp.Documents)
        resp.Documents = rerankedDocs[:opts.K]

        return resp, nil
    },
)
```