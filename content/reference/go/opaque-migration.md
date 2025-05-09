+++
title = "Go Opaque API 迁移"
weight = 650
linkTitle = "Opaque API 迁移"
description = "介绍自动迁移到 Opaque API 的方法。"
type = "docs"
+++

Opaque API 是 Protocol Buffers 针对 Go 编程语言的最新实现版本。旧版本现在称为 Open Struct API。请参阅 [Go Protobuf: Releasing the Opaque API](https://go.dev/blog/protobuf-opaque) 博客文章以了解介绍。

迁移到 Opaque API 是逐步进行的，可以按每个 proto 消息或每个 `.proto` 文件进行，通过设置 Protobuf Editions 功能的 `api_level` 选项为以下值之一：

*   `API_OPEN` 选择 Open Struct API；这是 2024 年 12 月之前的唯一 API。
*   `API_HYBRID` 是 Open 和 Opaque 之间的过渡：Hybrid API 也包含访问器方法（便于你更新代码），但仍然像以前一样导出结构体字段。性能没有差异；此 API 级别仅用于迁移。
*   `API_OPAQUE` 选择 Opaque API。

目前，默认值为 `API_OPEN`，但即将发布的 [Protobuf Edition 2024](./editions/overview) 将默认值更改为 `API_OPAQUE`。

要在 Edition 2024 之前使用 Opaque API，请如下设置 `api_level`：

```proto
edition = "2023";

package log;

import "google/protobuf/go_features.proto";
option features.(pb.go).api_level = API_OPAQUE;

message LogEntry { … }
```

在你将现有文件的 `api_level` 更改为 `API_OPAQUE` 之前，所有对生成的 proto 代码的现有用法都需要更新。`open2opaque` 工具可以帮助完成此操作。

为了方便起见，你还可以通过 `protoc` 命令行参数覆盖默认 API 级别：

```
protoc […] --go_opt=default_api_level=API_OPAQUE
```

要为特定文件（而不是所有文件）覆盖默认 API 级别，请使用 `apilevelM` 映射参数（类似于 [导入路径的 `M` 参数](./reference/go/go-generated/#package)）：

```
protoc […] --go_opt=apilevelMhello.proto=API_OPAQUE
```

这些命令行参数同样适用于仍然使用 proto2 或 proto3 语法的 `.proto` 文件，但如果你想在 `.proto` 文件内部选择 API 级别，则需要先将该文件迁移到 editions。

## 自动化迁移 {#automated}

我们尽量让你将现有项目迁移到 Opaque API 变得尽可能简单：我们的 open2opaque 工具可以完成大部分工作！

安装迁移工具的方法如下：

```
go install google.golang.org/open2opaque@latest
```

{{% alert title="注意" color="info" %}}如果你在自动迁移过程中遇到任何问题，请参阅 [Opaque API: 手动迁移](./reference/go/opaque-migration-manual) 指南。{{% /alert %}}

### 项目准备 {#projectprep}

确保你的构建环境和项目使用的是足够新的 Protocol Buffers 和 Go Protobuf 版本：

1.  从 [protobuf 发布页面](https://github.com/protocolbuffers/protobuf/releases/latest) 更新 protobuf 编译器（protoc）到 29.0 或更高版本。

1.  更新 protobuf 编译器 Go 插件（protoc-gen-go）到 1.36.0 或更高版本：

    ```
    go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
    ```

1.  在每个项目中，将 `go.mod` 文件中的 protobuf 模块更新到 1.36.0 或更高版本：

    ```
    go get google.golang.org/protobuf@latest
    ```

    {{% alert title="注意" color="note" %}}如果你还没有导入 `google.golang.org/protobuf`，你可能还在使用旧模块。请参阅 [google.golang.org/protobuf 公告（2020）](https://go.dev/blog/protobuf-apiv2)，并在返回本页面前完成代码迁移。{{% /alert %}}

### 步骤 1. 切换到 Hybrid API {#setup}

使用 `open2opaque` 工具将你的 `.proto` 文件切换到 Hybrid API：

```
open2opaque setapi -api HYBRID $(find . -name "*.proto")
```

你的现有代码将继续构建。Hybrid API 是 Open 和 Opaque API 之间的过渡，添加了新的访问器方法，但保持结构体字段可见。

### 步骤 2. `open2opaque rewrite` {#rewrite}

要将你的 Go 代码重写为使用 Opaque API，请运行 `open2opaque rewrite` 命令：

```
open2opaque rewrite -levels=red github.com/robustirc/robustirc/...
```

你可以指定一个或多个 [包或模式](https://pkg.go.dev/cmd/go#hdr-Package_lists_and_patterns)。

例如，如果你有如下代码：

```go
logEntry := &logpb.LogEntry{}
if req.IPAddress != nil {
    logEntry.IPAddress = redactIP(req.IPAddress)
}
logEntry.BackendServer = proto.String(host)
```

工具会将其重写为使用访问器：

```go
logEntry := &logpb.LogEntry{}
if req.HasIPAddress() {
    logEntry.SetIPAddress(redactIP(req.GetIPAddress()))
}
logEntry.SetBackendServer(host)
```

另一个常见示例是用结构体字面量初始化 protobuf 消息：

```go
return &logpb.LogEntry{
    BackendServer: proto.String(host),
}
```

在 Opaque API 中，等价写法是使用 Builder：

```go
return logpb.LogEntry_builder{
    BackendServer: proto.String(host),
}.Build()
```

该工具将可用的重写分为不同级别。`-levels=red` 参数启用所有重写，包括需要人工审核的更改。可用的级别如下：

*   <span style="background-color: lightgreen">green：</span> 安全重写（高置信度）。包括工具所做的大多数更改。这些更改无需仔细检查，甚至可以自动提交，无需人工监督。
*   <span style="background-color: yellow">yellow：</span>（合理置信度）这些重写需要人工审核。它们应该是正确的，但请务必检查。
*   <span style="background-color: salmon">red：</span> 潜在危险的重写，涉及罕见和复杂的模式。这些需要仔细人工审核。例如，当现有函数接受 `*string` 参数时，典型的修复方式 `proto.String(msg.GetFoo())` 并不适用，如果该函数意图通过写指针（`*foo = "value"`）来更改字段值。

许多程序只需绿色更改即可完全迁移。在你将 proto 消息或文件迁移到 Opaque API 之前，必须完成所有级别的重写，此时你的代码中不再有直接结构体访问。

### 步骤 3. 迁移与验证 {#migrate-and-verify}

要完成迁移，使用 `open2opaque` 工具将你的 `.proto` 文件切换到 Opaque API：

```
open2opaque setapi -api OPAQUE $(find . -name "*.proto")
```

现在，任何尚未重写为 Opaque API 的代码都将无法编译。

运行你的单元测试、集成测试和其他验证步骤（如有）。

## 有疑问？遇到问题？

首先，请查看 [Opaque API 常见问题](./reference/go/opaque-faq)。如果仍未解决你的问题，请参阅 [在哪里可以提问或报告问题？](./reference/go/opaque-faq#questions)
