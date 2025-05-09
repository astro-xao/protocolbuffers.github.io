+++
title = "概述"
weight = 10
description = "Protocol Buffers 是一种与语言和平台无关的可扩展机制，用于序列化结构化数据。"
type = "docs"
+++

它类似于 JSON，但更小、更快，并且可以生成原生语言绑定。你只需定义一次数据结构，然后就可以使用专门生成的源代码，轻松地将结构化数据读写到各种数据流中，并支持多种编程语言。

Protocol Buffers 包含定义语言（在 `.proto` 文件中创建）、proto 编译器生成的用于与数据交互的代码、语言特定的运行时库、用于写入文件（或通过网络连接发送）的序列化格式，以及序列化后的数据。

## Protocol Buffers 解决了哪些问题？ {#solve}

Protocol Buffers 提供了一种用于类型化、结构化数据包的序列化格式，适用于几兆字节以内的数据。该格式既适用于临时网络通信，也适用于长期数据存储。Protocol Buffers 可以通过添加新信息进行扩展，而不会使现有数据失效或需要更新代码。

Protocol Buffers 是 Google 内部最常用的数据格式，广泛用于服务器间通信以及磁盘上的数据归档存储。Protocol Buffer 的 *消息* 和 *服务* 由工程师编写的 `.proto` 文件描述。以下是一个 `message` 示例：

```proto
edition = "2023";

message Person {
    string name = 1;
    int32 id = 2;
    string email = 3;
}
```

在构建时，proto 编译器会对 `.proto` 文件进行处理，生成用于操作对应 Protocol Buffer 的多种编程语言代码（详见后文 [跨语言兼容性](#cross-lang)）。每个生成的类都包含对每个字段的简单访问器，以及用于将整个结构序列化和解析为原始字节的方法。以下是使用这些生成方法的示例：

```java
Person john = Person.newBuilder()
        .setId(1234)
        .setName("John Doe")
        .setEmail("jdoe@example.com")
        .build();
output = new FileOutputStream(args[0]);
john.writeTo(output);
```

由于 Protocol Buffers 被广泛用于 Google 的各种服务，并且其中的数据可能会长期存在，因此保持向后兼容性至关重要。Protocol Buffers 支持无缝更改，包括添加新字段和删除现有字段，而不会破坏现有服务。更多内容请参见后文 [无需更新代码即可更新 Proto 定义](#updating-defs)。

## 使用 Protocol Buffers 有哪些好处？ {#benefits}

Protocol Buffers 非常适合需要以与语言和平台无关、可扩展的方式序列化结构化、记录型、类型化数据的场景。它们最常用于定义通信协议（通常与 gRPC 一起使用）和数据存储。

使用 Protocol Buffers 的优势包括：

*   数据存储紧凑
*   解析速度快
*   支持多种编程语言
*   通过自动生成的类实现优化功能

### 跨语言兼容性 {#cross-lang}

同一消息可以被任何受支持编程语言编写的代码读取。你可以在一个平台上用 Java 程序采集数据，根据 `.proto` 定义进行序列化，然后在另一个平台上用 Python 应用程序提取序列化数据中的特定值。

Protocol Buffers 编译器 protoc 直接支持以下语言：

*   [C++](./reference/cpp/cpp-generated#invocation)
*   [C#](./reference/csharp/csharp-generated#invocation)
*   [Java](./reference/java/java-generated#invocation)
*   [Kotlin](./reference/kotlin/kotlin-generated#invocation)
*   [Objective-C](./reference/objective-c/objective-c-generated#invocation)
*   [PHP](./reference/php/php-generated#invocation)
*   [Python](./reference/python/python-generated#invocation)
*   [Ruby](./reference/ruby/ruby-generated#invocation)

以下语言由 Google 支持，但项目源码托管在 GitHub 上，protoc 编译器通过插件支持：

*   [Dart](https://github.com/google/protobuf.dart)
*   [Go](https://github.com/protocolbuffers/protobuf-go)

还有其他语言由第三方 GitHub 项目支持，详见
[Protocol Buffers 第三方插件](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)。

### 跨项目支持 {#cross-proj}

你可以通过将 `message` 类型定义在项目代码库之外的 `.proto` 文件中，实现跨项目使用 Protocol Buffers。如果你预计某些 `message` 类型或枚举会被广泛使用，可以将它们单独放在没有依赖的文件中。

Google 内部广泛使用的 proto 定义示例有
[`timestamp.proto`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/timestamp.proto)
和
[`status.proto`](https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto)。

### 无需更新代码即可更新 Proto 定义 {#updating-defs}

软件产品通常要求向后兼容，但很少要求向前兼容。只要在更新 `.proto` 定义时遵循一些
[简单实践](./programming-guides/proto3#updating)，旧代码就能读取新消息，只是会忽略新增字段。对于旧代码，已删除字段会有默认值，已删除的 repeated 字段会为空。有关 “repeated” 字段的说明，请参见后文 [Protocol Buffers 定义语法](#syntax)。

新代码也能透明地读取旧消息。新字段在旧消息中不会出现，此时 Protocol Buffers 会提供合理的默认值。

### Protocol Buffers 不适用的场景 {#not-good-fit}

Protocol Buffers 并不适用于所有数据场景，尤其是：

*   Protocol Buffers 假设整个消息可以一次性加载到内存中，且不大于对象图。对于超过几兆字节的数据，建议使用其他方案；处理大数据时，可能会因多次序列化导致内存使用激增。
*   Protocol Buffers 序列化后，同一数据可能有多种二进制表示。无法在不完全解析的情况下比较两个消息是否相等。
*   消息本身不压缩。虽然可以像其他文件一样用 zip 或 gzip 压缩，但针对特定类型数据的专用压缩算法（如 JPEG、PNG）会更高效。
*   对于涉及大量多维浮点数组的科学和工程应用，Protocol Buffers 在大小和速度上都不是最优选择。此类应用建议使用 [FITS](https://en.wikipedia.org/wiki/FITS) 等格式。
*   Protocol Buffers 在 Fortran、IDL 等科学计算常用的非面向对象语言中支持较差。
*   Protocol Buffers 消息本身不自描述，但有完整的反射式 schema 可用于实现自描述。即，必须有对应的 `.proto` 文件才能完全解析。
*   Protocol Buffers 不是任何组织的正式标准，因此不适用于有法律或标准化要求的场景。

## 谁在使用 Protocol Buffers？ {#who-uses}

许多项目都在使用 Protocol Buffers，包括：

+   [gRPC](https://grpc.io)
+   [Google Cloud](https://cloud.google.com)
+   [Envoy Proxy](https://www.envoyproxy.io)

## Protocol Buffers 如何工作？ {#work}

下图展示了使用 Protocol Buffers 处理数据的流程。

![编译流程，展示 proto 文件、生成代码和编译类的创建](../images/protocol-buffers-concepts.png) \
**图 1. Protocol Buffers 工作流程**

Protocol Buffers 生成的代码提供了从文件和流中检索数据、提取单个值、检查数据是否存在、将数据序列化回文件或流等实用方法。

以下代码示例展示了 Java 中的这一流程。如前所示，这是一个 `.proto` 定义：

```proto
message Person {
    string name = 1;
    int32 id = 2;
    string email = 3;
}
```

编译该 `.proto` 文件会生成一个 `Builder` 类，可用于创建新实例，如下所示：

```java
Person john = Person.newBuilder()
        .setId(1234)
        .setName("John Doe")
        .setEmail("jdoe@example.com")
        .build();
output = new FileOutputStream(args[0]);
john.writeTo(output);
```

你还可以用 Protocol Buffers 在其他语言（如 C++）中反序列化数据：

```cpp
Person john;
fstream input(argv[1], ios::in | ios::binary);
john.ParseFromIstream(&input);
int id = john.id();
std::string name = john.name();
std::string email = john.email();
```

## Protocol Buffers 定义语法 {#syntax}

定义 `.proto` 文件时，可以指定字段的基数（单个或 repeated）。在 proto2 和 proto3 中，还可以指定字段是否为 optional。在 proto3 中，将字段设为 optional
[会将其从隐式 presence 改为显式 presence](./programming-guides/field_presence)。

设置字段基数后，指定数据类型。Protocol Buffers 支持常见的原始数据类型，如整数、布尔值和浮点数。完整列表见
[标量值类型](./programming-guides/proto3#scalar)。

字段还可以是：

*   `message` 类型，可用于嵌套定义（如重复数据集）。
*   `enum` 类型，可指定一组选项。
*   `oneof` 类型，适用于有多个可选字段且同一时间最多只设置一个字段的消息。
*   `map` 类型，可为定义添加键值对。

消息可以通过 **扩展** 机制定义消息外的字段。例如，protobuf 库的内部消息 schema 允许为自定义选项扩展。

更多选项请参见
[proto2](./programming-guides/proto2)、
[proto3](./programming-guides/proto3) 或
[2023 版](./programming-guides/editions) 的语言指南。

设置基数和数据类型后，需要为字段命名。命名时需注意：

*   字段名一旦投入生产后更改可能很困难甚至不可能。
*   字段名不能包含短横线。更多语法见
        [消息和字段命名](./programming-guides/style#message-field-names)。
*   对于 repeated 字段，使用复数命名。

命名后，需要分配字段编号。字段编号不能被重复使用。如果删除字段，应保留其编号，防止被误用。

## 其他数据类型支持 {#data-types}

Protocol Buffers 支持多种标量值类型，包括变长编码和定长整数。你还可以通过定义消息类型创建自定义复合数据类型。除了简单和复合类型外，还发布了若干[常用类型](./best-practices/dos-donts#common)。

## 历史 {#history}

关于 Protocol Buffers 项目的历史，请参见
[Protocol Buffers 的历史](./history)。

## Protocol Buffers 开源理念 {#philosophy}

Protocol Buffers 于 2008 年开源，旨在让 Google 以外的开发者也能享受其内部带来的好处。我们通过定期更新语言来支持开源社区，同时满足内部需求。虽然我们会接受部分外部开发者的 pull request，但无法始终优先处理不符合 Google 需求的功能请求和 bug 修复。

## 开发者社区 {#community}

想要获取 Protocol Buffers 的最新动态，或与 protobuf 开发者和用户交流，
[加入 Google Group](https://groups.google.com/g/protobuf)。

## 其他资源 {#additional-resources}

*   [Protocol Buffers GitHub](https://github.com/protocolbuffers/protobuf/)
        * [教程](https://protobuf.dev/getting-started/)
