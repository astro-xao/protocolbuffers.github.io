+++
title = "技术技巧"
weight = 70
description = "介绍处理 Protocol Buffers 时常用的一些设计模式。"
type = "docs"
+++

你也可以将设计和使用相关的问题发送到
[Protocol Buffers 讨论组](http://groups.google.com/group/protobuf)。

## 常见文件名后缀 {#suffixes}

通常会以多种不同格式将消息写入文件。我们建议为这些文件使用以下扩展名。

内容                                                                   | 扩展名
------------------------------------------------------------------------- | ---------
[文本格式](./reference/protobuf/textformat-spec) | `.txtpb`
[二进制格式](./programming-guides/encoding)        | `.binpb`
[JSON 格式](./programming-guides/proto3#json)     | `.json`

对于文本格式，`.textproto` 也很常见，但我们推荐使用 `.txtpb`，因为它更简洁。

## 多消息流式处理 {#streaming}

如果你想将多个消息写入同一个文件或流，需要自己跟踪每个消息的起止位置。Protocol Buffer 的二进制格式不是自描述的，因此解析器无法自行判断消息的结束位置。最简单的解决方法是，在写入每个消息前先写入其大小。读取时，先读取大小，再将对应字节读入缓冲区，然后解析该缓冲区。（如果想避免复制字节到单独缓冲区，可以查看 `CodedInputStream` 类（C++ 和 Java 均有），它可以限制读取的字节数。）

## 大数据集 {#large-data}

Protocol Buffers 并不适合处理大型消息。一般来说，如果每条消息超过 1MB，建议考虑其他方案。

不过，Protocol Buffers 非常适合处理大型数据集中的单条消息。通常，大型数据集由许多结构化的小数据组成。虽然 Protocol Buffers 无法一次处理整个数据集，但用它编码每个小数据块可以大大简化问题：你只需处理一组字节串，而不是一组结构体。

Protocol Buffers 没有内置对大数据集的支持，因为不同场景需要不同的解决方案。有时只需简单的记录列表，有时则需要类似数据库的结构。每种方案都应作为独立库开发，只有需要的人才需承担相应的成本。

## 自描述消息 {#self-description}

Protocol Buffers 本身不包含类型描述。因此，仅凭原始消息而没有对应的 `.proto` 文件，很难提取有用数据。

不过，.proto 文件的内容本身可以用 Protocol Buffers 表示。源码包中的 `src/google/protobuf/descriptor.proto` 定义了相关消息类型。`protoc` 可以通过 `--descriptor_set_out` 选项输出 `FileDescriptorSet`，它表示一组 .proto 文件。你可以这样定义自描述协议消息：

```proto
syntax = "proto3";

import "google/protobuf/any.proto";
import "google/protobuf/descriptor.proto";

message SelfDescribingMessage {
  // 描述类型及其依赖的 FileDescriptorProto 集合。
  google.protobuf.FileDescriptorSet descriptor_set = 1;

  // 以 Any 消息编码的消息及其类型。
  google.protobuf.Any message = 2;
}
```

通过使用如 `DynamicMessage`（C++ 和 Java 均有）等类，你可以编写工具来操作 `SelfDescribingMessage`。

需要注意的是，这一功能没有包含在 Protocol Buffer 库中，因为 Google 内部并未遇到相关需求。

该技术需要平台支持基于描述符的动态消息。在使用自描述消息前，请确认你的平台支持此功能。
