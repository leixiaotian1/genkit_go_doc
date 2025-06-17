### 使用 Go 上手 Genkit

本指南将向您展示如何在 Go 应用中开始使用 Genkit。

如果您发现库或本文档存在问题，请在我们的 [GitHub 仓库](https://github.com/firebase/genkit)中报告。

### 发出您的第一个请求

安装 Go 1.24 或更高版本。请参阅 Go 官方文档中的“[下载和安装](https://go.dev/doc/install)”部分。

使用 Genkit 包初始化一个新的 Go 项目目录：

**终端窗口**
```bash
mkdir genkit-intro && cd genkit-intro

go mod init example/genkit-intro

go get github.com/firebase/genkit/go
```

创建一个 `main.go` 文件，并填入以下示例代码：

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

    // 使用 Google AI 插件和 Gemini 2.0 Flash 初始化 Genkit。
    g, err := genkit.Init(ctx,
        genkit.WithPlugins(&googlegenai.GoogleAI{}),
        genkit.WithDefaultModel("googleai/gemini-2.0-flash"),
    )
    if err != nil {
        log.Fatalf("无法初始化 Genkit: %v", err)
    }

    resp, err := genkit.Generate(ctx, g, ai.WithPrompt("人生的意义是什么？"))
    if err != nil {
        log.Fatalf("无法生成模型响应: %v", err)
    }

    log.Println(resp.Text())
}
```

通过设置 `GEMINI_API_KEY` 环境变量来配置您的 Gemini API 密钥：

**终端窗口**
```bash
export GEMINI_API_KEY=<你的 API 密钥>
```

如果您还没有 API 密钥，请在 [Google AI Studio](https://aistudio.google.com/app/apikey) 中创建一个。Google AI 提供了慷慨的免费套餐，无需信用卡即可开始使用。

运行应用以查看模型的响应：

**终端窗口**
```bash
go run .
# 示例输出（可能有所不同）：
# 人生没有一个普遍公认的意义；这是一个非常个人化的问题。
# 许多人通过人际关系、个人成长、做出贡献、追求幸福或发现自己的目标来找到意义。
```

### 后续步骤

既然您已经配置好使用 Genkit 发出模型请求，接下来可以学习如何利用 Genkit 的更多功能来构建您的 AI 驱动的应用和工作流。要开始了解 Genkit 的其他功能，请参阅以下指南：

*   **开发者工具**：了解如何设置和使用 Genkit 的命令行工具（CLI）和开发者界面，以帮助您在本地测试和调试应用。
*   **生成内容**：了解如何使用 Genkit 的统一生成 API，通过任何支持的模型生成文本和结构化数据。
*   **创建 Flow**：了解如何使用称为 Flow 的特殊 Genkit 函数，这些函数为工作流提供端到端的可见性，并能通过 Genkit 工具进行丰富的调试。
*   **管理提示**：了解 Genkit 如何帮助您将提示（Prompt）和配置作为代码一起进行管理。