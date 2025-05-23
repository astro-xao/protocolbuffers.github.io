+++
title = "其他语言"
weight = 840
description = "protoc（Protocol Buffers 编译器）可以通过插件扩展以支持新语言。"
type = "docs"
+++

当前版本已包含对 C++、Java、Go、Ruby、C\# 和 Python 的编译器和 API，但编译器代码的设计使得添加对其他语言的支持变得容易。社区中有多个项目正在为 Protocol Buffers 添加新的语言实现，包括 C、Haskell、Perl、Rust 等。

我们已知的相关项目列表请参见
[第三方插件 wiki 页面](https://github.com/protocolbuffers/protobuf/blob/main/docs/third_party.md)。

## 编译器插件 {#plugins}

`protoc`（Protocol Buffers 编译器）可以通过插件扩展以支持新语言。插件其实就是一个程序，它从标准输入读取 `CodeGeneratorRequest` 协议缓冲区，然后将 `CodeGeneratorResponse` 协议缓冲区写入标准输出。这些消息类型定义在
[`plugin.proto`](./reference/cpp/api-docs/google.protobuf.compiler.plugin.pb) 中。我们建议所有第三方代码生成器都作为插件编写，这样所有生成器都能提供一致的接口并共享同一个解析器实现。

插件可以用任何编程语言编写，但 Google 官方插件是用 C++ 实现的。如果你要编写自己的插件，使用 C++ 可以方便地参考这些示例并复用相关工具。

此外，插件还可以向其他代码生成器生成的文件中插入代码。有关“插入点”的更多信息，请参见 `plugin.proto` 中的注释。例如，你可以编写一个插件，为特定的 RPC 系统生成定制的 RPC 服务代码。每种语言生成的代码所支持的插入点，请参见相应的文档。
