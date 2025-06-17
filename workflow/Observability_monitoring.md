### 监控

Genkit 提供了两种互补的监控功能：**OpenTelemetry 导出**和使用**开发者界面进行的追踪检查**。

#### OpenTelemetry 导出

Genkit 完全集成了 OpenTelemetry，并提供了用于导出遥测数据的钩子。

Google Cloud 插件可将遥测数据导出到 Cloud 运维套件（Cloud's operations suite）。

#### 追踪存储

追踪存储功能是对遥测检测功能的补充。它让您可以在 Genkit 开发者界面中检查您 flow 运行的追踪信息。

当您在开发环境中运行 Genkit flow 时（例如，在使用 `genkit start` 或 `genkit flow:run` 时），此功能就会被启用。