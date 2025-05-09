+++
title = "Go Opaque API 常见问题解答"
weight = 670
linkTitle = "Opaque API 常见问题解答"
description = "关于 Opaque API 的常见问题列表。"
type = "docs"
+++

<style>
.good pre {
    border-left: 2px solid;
    border-left-color: #0b8043;
    background-color: #e2f3eb !important;
}
.bad pre {
    border-left: 2px solid;
    border-left-color: #c53929;
    background-color: #fbe9e7 !important;
}
</style>

Opaque API 是 Protocol Buffers 针对 Go 语言实现的最新版本。旧版本现在称为 Open Struct API。请参阅 [Go Protobuf: The new Opaque API](https://go.dev/blog/protobuf-opaque) 博客文章以了解介绍。

本常见问题解答回答了关于新 API 及迁移过程中的常见问题。

## 创建新的 .proto 文件时应使用哪个 API？ {#which}

我们建议新开发选择 Opaque API。Protobuf Edition 2024（参见 [Protobuf Editions 概览](./editions/overview/)）将使 Opaque API 成为默认选项。

## 如何为我的消息启用新的 Opaque API？ {#enable}

在 Protobuf Edition 2023（撰写时为当前版本）中，可以通过在 `.proto` 文件中将 `api_level` editions 特性设置为 `API_OPAQUE` 来选择 Opaque API。可以按文件或按消息设置：

```proto
edition = "2023";

package log;

import "google/protobuf/go_features.proto";
option features.(pb.go).api_level = API_OPAQUE;

message LogEntry { … }
```

Protobuf Edition 2024 将默认使用 Opaque API，届时无需额外导入或选项：

```proto
edition = "2024";

package log;

message LogEntry { … }
```

Protobuf Edition 2024 预计将在 2025 年初发布。

你也可以通过 `protoc` 命令行参数覆盖默认 API 级别：

```
protoc […] --go_opt=default_api_level=API_HYBRID
```

如需仅为特定文件覆盖默认 API 级别，可使用 `apilevelM` 映射参数（类似于 [导入路径的 `M` 参数](./reference/go/go-generated/#package)）：

```
protoc […] --go_opt=apilevelMhello.proto=API_HYBRID
```

命令行参数同样适用于仍使用 proto2 或 proto3 语法的 `.proto` 文件，但如果你希望在 `.proto` 文件中选择 API 级别，则需先将该文件迁移到 editions。

## 如何启用延迟解码（Lazy Decoding）？ {#lazydecoding}

1.  将代码迁移到 opaque 实现。
2.  在需要延迟解码的 proto 子消息字段上设置 `[lazy = true]` 选项。
3.  运行单元和集成测试，然后部署到预发布环境。

## 启用延迟解码后错误会被忽略吗？ {#lazydecodingerrors}

不会。
[`proto.Marshal`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Marshal) 即使在解码被延迟到首次访问时，也始终会验证 wire 格式数据。

## 我可以在哪里提问或报告问题？ {#questions}

如果你发现 `open2opaque` 迁移工具有问题（如代码重写不正确），请在 [open2opaque 问题追踪器](https://github.com/golang/open2opaque/issues) 报告。

如果你发现 Go Protobuf 有问题，请在 [Go Protobuf 问题追踪器](https://github.com/golang/protobuf/issues/) 报告。

## Opaque API 有哪些优势？ {#benefits}

Opaque API 带来了诸多优势：

*   使用更高效的内存表示，降低内存和垃圾回收成本。
*   支持延迟解码，可显著提升性能。
*   修复了许多棘手问题。使用 Opaque API 时，指针地址比较、意外共享或 Go 反射的非预期使用等 bug 都能避免。
*   通过支持基于性能分析的优化，实现理想的内存布局。

详见 [Go Protobuf: The new Opaque API 博客文章](https://go.dev/blog/protobuf-opaque)。

## Builder 和 Setter 哪个更快？ {#builders-faster}

通常，使用 builder 的代码：

```go
_ = pb.M_builder{
    F: &val,
}.Build()
```

比下面这种等价写法要慢：

```go
m := &pb.M{}
m.SetF(val)
```

原因如下：

1.  `Build()` 会遍历消息中的所有字段（即使未显式设置），并将其值（如有）复制到最终消息。对于字段较多的消息，这种线性性能会有影响。
2.  可能会有额外的堆分配（如 `&val`）。
3.  如果存在 oneof 字段，builder 可能会显著变大并占用更多内存。builder 为每个 oneof 联合成员分配一个字段，而消息本身只需一个字段。

除了运行时性能，如果你关心二进制体积，避免使用 builder 会生成更少的代码。

## 如何使用 Builder？ {#builders-how}

Builder 设计为 *值类型* 并应立即调用 `Build()`。避免使用 builder 的指针或将 builder 存储在变量中。

```go {.good}
m := pb.M_builder{
        // ...
}.Build()
```

```go {.bad}
// 错误：避免使用指针
m := (&pb.M_builder{
        // ...
}).Build()
```

```go {.bad highlight="content:b.:= "}
// 错误：避免存储到变量
b := pb.M_builder{
        // ...
}
m := b.Build()
```

在其他语言中，proto 消息是不可变的，因此用户习惯于在构造 proto 消息时传递 builder 类型。但 Go proto 消息是可变的，无需传递 builder，只需传递 proto 消息即可。

```go {.bad}
// 错误：避免传递 builder
func populate(mb *pb.M_builder) {
    mb.Field1 = proto.Int32(4711)
    //...
}
// ...
mb := pb.M_builder{}
populate(&mb)
m := mb.Build()
```

```go {.good}
func populate(mb *pb.M) {
    mb.SetField1(4711)
    //...
}
// ...
m := &pb.M{}
populate(m)
```

Builder 旨在模仿 Open Struct API 的复合字面量构造方式，而不是 proto 消息的替代表示。

推荐模式也更高效。直接在 builder 结构体字面量上调用 `Build()` 可获得良好优化。单独调用 `Build()` 则难以优化，因为编译器难以识别哪些字段已被填充。如果 builder 生命周期较长，小对象如标量可能需要堆分配，后续还需垃圾回收。

## 应该用 Builder 还是 Setter？ {#builders-vs-setters}

构造空的 protocol buffer 时，建议使用 `new` 或空复合字面量。两者都是 Go 中构造零值的惯用方式，比空 builder 更高效。

```go {.good}
m1 := new(pb.M)
m2 := &pb.M{}
```

```go {.bad}
// 错误：不必要的复杂
m1 := pb.M_builder{}.Build()
```

如需构造非空 protocol buffer，可选择使用 setter 或 builder。两者都可以，但大多数人会觉得 builder 更易读。如果你关注性能，[setter 通常比 builder 略快](#builders-faster)。

```go {.good}
// 推荐：使用 builder
m1 := pb.M1_builder{
        Submessage: pb.M2_builder{
                Submessage: pb.M3_builder{
                        String: proto.String("hello world"),
                        Int:    proto.Int32(42),
                }.Build(),
                Bytes: []byte("hello"),
        }.Build(),
}.Build()
```

```go
// 也可以：使用 setter
m3 := &pb.M3{}
m3.SetString("hello world")
m3.SetInt(42)
m2 := &pb.M2{}
m2.SetSubmessage(m3)
m2.SetBytes([]byte("hello"))
m1 := &pb.M1{}
m1.SetSubmessage(m2)
```

如某些字段需在设置前进行条件判断，可结合使用 builder 和 setter。

```go {.good}
m1 := pb.M1_builder{
        Field1: value1,
}.Build()
if someCondition() {
        m1.SetField2(value2)
        m1.SetField3(value3)
}
```

## 如何影响 open2opaque 的 builder 行为？ {#builders-flags}

`open2opaque` 工具的 `--use_builders` 参数可取以下值：

*   `--use_builders=everywhere`：始终使用 builder，无例外。
*   `--use_builders=tests`：仅在测试中使用 builder，其他情况用 setter。
*   `--use_builders=nowhere`：从不使用 builder。

## 性能提升有多大？ {#performance-benefit}

这高度依赖于你的工作负载。以下问题可帮助你评估性能：

*   Go Protobuf 占用你多少 CPU？某些工作负载（如日志分析流水线）在 Protobuf 输入记录上计算统计信息，约 50% 的 CPU 用于 Go Protobuf。这类场景下性能提升会很明显。相反，如果程序仅有 3-5% 的 CPU 用于 Go Protobuf，则性能提升通常微不足道。
*   程序对延迟解码的适应性如何？如果大量输入消息从未被访问，延迟解码可节省大量工作。此模式常见于代理服务器（原样转发输入）或高选择性日志分析流水线（基于高层谓词丢弃许多记录）。
*   消息定义中是否有大量带显式 presence 的基础字段？Opaque API 对整数、布尔、枚举和浮点等基础字段采用更高效的内存表示，但对字符串、repeated 字段或子消息则无此优化。

## Proto2、Proto3 和 Editions 与 Opaque API 有何关系？ {#proto23editions}

proto2 和 proto3 指的是 `.proto` 文件的不同语法版本。[Protobuf Editions](./editions/overview) 是二者的继任者。

Opaque API 只影响 `.pb.go` 文件中的生成代码，不影响你在 `.proto` 文件中的写法。

Opaque API 的行为与 `.proto` 文件使用的语法或 edition 无关。但如果你希望按文件选择 Opaque API（而不是在运行 `protoc` 时用命令行参数），则必须先将文件迁移到 editions。详见 [如何为我的消息启用新的 Opaque API？](#enable)。

## 为什么只改变基础字段的内存布局？ {#memorylayout}

[公告博客的“Opaque structs use less memory”部分](https://go.dev/blog/protobuf-opaque#lessmemory) 解释道：

> 这种性能提升 [更高效地建模字段 presence] 很大程度上取决于你的 protobuf 消息结构：该变化只影响整数、布尔、枚举和浮点等基础字段，不影响字符串、repeated 字段或子消息。

自然会有人问，为什么字符串、repeated 字段和子消息在 Opaque API 中仍然用指针表示？原因有二：

### 考虑一：内存使用

将子消息表示为值而非指针会增加内存使用：每种 Protobuf 消息类型都带有内部状态，即使子消息未实际设置也会消耗内存。

对于字符串和 repeated 字段，情况更复杂。比较一下字符串值和字符串指针的内存占用：

Go 变量类型 | 是否设置 | [字]数                  | 字节数
------------ | ------- | ------------------------ | ------
`string`     | 是      | 2（data, len）           | 16
`string`     | 否      | 2（data, len）           | 16
`*string`    | 是      | 1（data）+2（data, len） | 24
`*string`    | 否      | 1（data）                | 8

[word]: https://en.wikipedia.org/wiki/Word_(computer_architecture)

（slice 也类似，但 slice 头需要 3 个字：data、len、cap。）

如果你的字符串字段大多未设置，使用指针可节省内存。当然，这会带来更多分配和指针，增加垃圾回收压力。

Opaque API 的优势在于可以无需用户代码变更就调整表示方式。当前内存布局在引入时最优，但如果今天或五年后重新评估，也许会选择不同布局。

如 [公告博客的“Making the ideal memory layout possible”部分](https://go.dev/blog/protobuf-opaque#idealmemory) 所述，未来我们希望能按工作负载做出这些优化决策。

### 考虑二：延迟解码

除了内存使用外，还有另一个限制：启用 [延迟解码](#lazydecoding) 的字段必须用指针表示。

Protobuf 消息可安全并发访问（但不可并发修改），因此如果两个 goroutine 触发延迟解码，需要某种协调。该协调通过 [`sync/atomic` 包](https://pkg.go.dev/sync/atomic) 实现，可原子更新指针，但不能原子更新 slice 头（超出一个 [字]）。

目前 `protoc` 仅允许对（非 repeated）子消息启用延迟解码，但该理由对所有字段类型都适用。
