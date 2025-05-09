+++
title = "不支持可空 Setter/Getters"
weight = 89
description = "说明为什么 Protobuf 不支持可空的 setter 和 getter"
type = "docs"
aliases = "/programming-guides/nullable-getters-setters/"
+++

我们收到了一些反馈，有开发者希望 protobuf 能在他们所用的支持 null 的语言（尤其是 Kotlin、C# 和 Rust）中支持可空的 getter/setter。虽然这对这些语言的用户来说似乎是一个有用的特性，但这种设计选择存在权衡，因此 Protobuf 团队选择不予实现。

不支持可空字段的最大原因在于 `.proto` 文件中指定的默认值的预期行为。按照设计，调用未设置字段的 getter 时会返回该字段的默认值。

**注意：** C# 确实将 *message* 字段视为可空。这与其他语言的不一致，源于缺乏不可变消息，这使得无法创建可共享的不可变默认实例。由于 message 字段不能有默认值，因此这不会带来实际问题。

举个例子，假设有如下 `.proto` 文件：

```proto
message Msg { optional Child child = 1; }
message Child { optional Grandchild grandchild = 1; }
message Grandchild { optional int32 foo = 1 [default = 72]; }
```

以及对应的 Kotlin getter：

```kotlin
// 在我们的 API 中，getter 总是非空的：
msg.child.grandchild.foo == 72

// 如果子消息是可空的，?. 操作符无法获取默认值：
msg?.child?.grandchild?.foo == null

// 或者在使用处冗余地写出默认值：
(msg?.child?.grandchild?.foo ?: 72)
```

以及对应的 Rust getter：

```rust
// 在我们的 API 中：
msg.child().grandchild().foo()   // == 72

// 如果每个 getter 都是 Option<T>，则既冗长又无法获得默认值
msg.child().map(|c| c.grandchild()).map(|gc| gc.foo()) // == Option::None

// 在极少数需要同时获取字段存在性和数值的场景下，可以使用返回自定义 Optional 类型的 _opt() 访问器（Optional 类型类似于 Option，但也能感知默认值）：
msg.child().grandchild().foo_opt() // Optional::Unset(72)
```

如果存在可空 getter，它必然会忽略用户指定的默认值（返回 null），这会导致令人惊讶且不一致的行为。如果可空 getter 的用户想访问字段的默认值，就必须自己写代码处理 null 的情况，这反而失去了可空 getter 带来的简洁/易用的好处。

同样，我们也不提供可空 setter，因为其行为会令人费解。执行 set 后再 get 并不总能得到相同的值，调用 set 也只有在某些情况下才会影响字段的 has 位。

需要注意的是，message 类型的字段总是显式存在性字段（带有 has 方法）。Proto3 默认标量字段为隐式存在性（没有 has 方法），除非显式标记为 `optional`，而 Proto2 不支持隐式存在性。随着 [Editions](./editions/features#field_presence) 的引入，显式存在性成为默认行为，除非使用了隐式存在性特性。预计未来几乎所有字段都会有显式存在性，因此可空 getter 带来的易用性问题会比 Proto3 用户遇到的更突出。

基于上述原因，可空 setter/getter 会彻底改变默认值的使用方式。虽然我们理解其潜在用途，但我们认为其带来的不一致性和复杂性不值得实现。
