+++
title = "枚举行为"
weight = 55
description = "解释 Protocol Buffers 中枚举的当前工作方式及其理想工作方式。"
type = "docs"
+++

不同语言库中枚举的行为各不相同。本主题介绍了这些不同的行为，以及将 protobufs 迁移到所有语言一致状态的计划。如果你想了解如何使用枚举，请参阅
[proto2](./programming-guides/proto2#enum) 和
[proto3](./programming-guides/proto3#enum) 语言指南中的相关章节。

## 定义 {#definitions}

枚举有两种不同的类型（*开放* 和 *封闭*）。它们的行为完全相同，除了对未知值的处理方式不同。实际上，这意味着简单场景下行为一致，但某些边界情况会有有趣的影响。

为便于说明，假设我们有如下 `.proto` 文件（此处有意未指定 `syntax = "proto2"` 还是 `syntax = "proto3"`）：

```
enum Enum {
    A = 0;
    B = 1;
}

message Msg {
    optional Enum enum = 1;
}
```

*开放* 和 *封闭* 的区别可以用一个问题来概括：

> 当程序解析包含字段 1 且值为 `2` 的二进制数据时会发生什么？

*   **开放**枚举会解析值 `2` 并直接存储在字段中。访问器会报告该字段已*设置*，并返回代表 `2` 的内容。
*   **封闭**枚举会解析值 `2` 并将其存储在消息的未知字段集中。访问器会报告该字段为*未设置*，并返回枚举的默认值。

## *封闭* 枚举的影响

*封闭* 枚举在解析重复字段时会带来意想不到的后果。当解析 `repeated Enum` 字段时，所有未知值都会被放入
[未知字段](./programming-guides/proto3/#unknowns)
集中。序列化时，这些未知值会被再次写出，但*不会在原列表中的原始位置*。例如，给定如下 `.proto` 文件：

```
enum Enum {
    A = 0;
    B = 1;
}

message Msg {
    repeated Enum r = 1;
}
```

如果线格式包含字段 1 的值为 `[0, 2, 1, 2]`，则重复字段解析后为 `[0, 1]`，而 `[2, 2]` 会被存储为未知字段。重新序列化消息后，线格式将变为 `[0, 1, 2, 2]`。

类似地，值为*封闭*枚举的 map，在值未知时会将整个条目（键和值）放入未知字段。

## 历史 {#history}

在引入 `syntax = "proto3"` 之前，所有枚举都是*封闭*的。Proto3 专门引入了*开放*枚举，以解决*封闭*枚举带来的意外行为。

## 规范 {#spec}

以下是 protobuf 合规实现的行为规范。由于细节复杂，许多实现并不完全符合规范。不同实现的行为详见[已知问题](#known-issues)。

*   `proto2` 文件导入 `proto2` 文件中定义的枚举时，该枚举应视为**封闭**。
*   `proto3` 文件导入 `proto3` 文件中定义的枚举时，该枚举应视为**开放**。
*   `proto3` 文件导入 `proto2` 文件中定义的枚举时，`protoc` 编译器会报错。
*   `proto2` 文件导入 `proto3` 文件中定义的枚举时，该枚举应视为**开放**。

## 已知问题 {#known-issues}

### C++ {#cpp}

所有已知 C++ 版本都不符合规范。当 `proto2` 文件导入 `proto3` 文件中定义的枚举时，C++ 会将该字段视为**封闭**枚举。
在 editions 中，该行为由已弃用的字段特性
[`features.(pb.cpp).legacy_closed_enum`](./editions/features#legacy_closed_enum)
表示。迁移到合规行为有两种方式：

*   移除该字段特性。推荐此方式，但可能导致运行时行为变化。移除后，未识别的整数会被强制转换为枚举类型存储在字段中，而不是放入未知字段集。
*   将枚举改为封闭。不推荐此方式，且如果*其他人*也在使用该枚举，可能导致运行时行为变化。未识别的整数会进入未知字段集。

### C&#35; {#csharp}

所有已知 C# 版本都不符合规范。C# 将所有枚举视为**开放**。

### Java {#java}

所有已知 Java 版本都不符合规范。当 `proto2` 文件导入 `proto3` 文件中定义的枚举时，Java 会将该字段视为**封闭**枚举。

在 editions 中，该行为由已弃用的字段特性
[`features.(pb.java).legacy_closed_enum`](./editions/features#legacy_closed_enum)
表示。迁移到合规行为有两种方式：

*   移除该字段特性。可能导致运行时行为变化。移除后，未识别的整数会被存储在字段中，枚举 getter 会返回 `UNRECOGNIZED`。之前这些值会被放入未知字段集。
*   将枚举改为封闭。如果*其他人*也在使用，可能导致运行时行为变化。未识别的整数会进入未知字段集。

> **注意：** Java 对**开放**枚举的处理有一些意外情况。如下定义：
>
> ```
> syntax = "proto3";
>
> enum Enum {
>   A = 0;
>   B = 1;
> }
>
> message Msg {
>   repeated Enum name = 1;
> }
> ```
>
> Java 会生成 `Enum getName()` 和 `int getNameValue()` 方法。`getName` 方法对于超出已知集合的值（如 `2`）会返回 `Enum.UNRECOGNIZED`，而 `getNameValue` 会返回 `2`。
>
> 同样，Java 会生成 `Builder setName(Enum value)` 和 `Builder setNameValue(int value)` 方法。`setName` 方法传入 `Enum.UNRECOGNIZED` 时会抛出异常，而 `setNameValue` 会接受 `2`。

### Kotlin {#kotlin}

所有已知 Kotlin 版本都不符合规范。当 `proto2` 文件导入 `proto3` 文件中定义的枚举时，Kotlin 会将该字段视为**封闭**枚举。

Kotlin 基于 Java，具有相同的特殊情况。

### Go {#go}

所有已知 Go 版本都不符合规范。Go 将所有枚举视为**开放**。

### JSPB {#jspb}

所有已知 JSPB 版本都不符合规范。JSPB 将所有枚举视为**开放**。

### PHP {#php}

PHP 是合规的。

### Python {#python}

Python 在 4.22.0 及以上版本（2023 年第一季度发布）是合规的。

不再受支持的旧版本不合规。当 `proto2` 文件导入 `proto3` 文件中定义的枚举时，非合规 Python 版本会将该字段视为**封闭**枚举。

### Ruby {#ruby}

所有已知 Ruby 版本都不符合规范。Ruby 将所有枚举视为**开放**。

### Objective-C {#obj-c}

Objective-C 在 3.22.0 及以上版本（2023 年第一季度发布）是合规的。

不再受支持的旧版本不合规。当 `proto2` 文件导入 `proto3` 文件中定义的枚举时，非合规 ObjC 版本会将该字段视为**封闭**枚举。

### Swift {#swift}

Swift 是合规的。

### Dart {#dart}

Dart 将所有枚举视为**封闭**。
