+++
title = "协议缓冲区"
weight = 5
toc_hide = "true"
description = "协议缓冲区是一种与语言和平台无关的可扩展机制，用于序列化结构化数据。"
type = "docs"
no_list = "true"
+++

## 什么是协议缓冲区？

协议缓冲区（Protocol Buffers）是 Google 提供的一种与语言和平台无关、可扩展的结构化数据序列化机制——可以把它想象成更小、更快、更简单的 XML。你只需定义一次数据结构，然后就可以使用自动生成的源代码，轻松地在多种数据流和多种语言之间读写你的结构化数据。

## 选择你喜欢的语言

协议缓冲区支持在 C++、C#、Dart、Go、Java、Kotlin、Objective-C、Python 和 Ruby 中生成代码。使用 proto3，还可以支持 PHP。

## 示例实现

```proto
edition = "2023";

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;
}
```

**图 1.** 一个 proto 定义。

```java
// Java 代码
Person john = Person.newBuilder()
    .setId(1234)
    .setName("John Doe")
    .setEmail("jdoe@example.com")
    .build();
output = new FileOutputStream(args[0]);
john.writeTo(output);
```

**图 2.** 使用生成的类持久化数据。

```cpp
// C++ 代码
Person john;
fstream input(argv[1],
    ios::in | ios::binary);
john.ParseFromIstream(&input);
id = john.id();
name = john.name();
email = john.email();
```

**图 3.** 使用生成的类解析持久化数据。

## 如何开始？

<ol>

  <li>
    <a href="https://github.com/protocolbuffers/protobuf#protobuf-compiler-installation">下载并安装</a> 协议缓冲区编译器。
  </li>

  <li>
    阅读 <a href="/overview">概述</a>。
  </li>
  <li>
    按照你选择的语言，尝试 <a href="/getting-started">入门教程</a>。
  </li>
</ol>
