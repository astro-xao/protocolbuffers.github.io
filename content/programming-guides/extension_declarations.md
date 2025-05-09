+++
title = "扩展声明"
weight = 83
description = "详细描述扩展声明是什么、为什么需要它们以及如何使用它们。"
type = "docs"
+++

<!--*
# 文档新鲜度：更多信息请参见 go/fresh-source。
freshness: { owner: 'shaod' reviewed: '2024-09-16' }
*-->

## 简介 {#intro}

本页详细介绍了扩展声明是什么、为什么需要它们以及如何使用它们。

{{% alert title="注意" color="note" %}}
Proto3 不支持扩展（除了[声明自定义选项](./programming-guides/proto3/#customoptions)）。
扩展在 proto2 和 editions 中完全支持。{{% /alert %}}

如果你需要扩展的入门介绍，请阅读此[扩展指南](https://protobuf.dev/programming-guides/proto2/#extensions)

## 动机 {#motivation}

扩展声明旨在在常规字段和扩展之间取得平衡。像扩展一样，它们避免了对字段消息类型的依赖，从而在难以或无法剥离未使用消息的环境中实现更精简的构建图和更小的二进制文件。像常规字段一样，字段名称/编号会出现在封闭消息中，这使得避免冲突和方便查看已声明字段列表变得更容易。

通过扩展声明列出已占用的扩展编号，使用户更容易选择可用的扩展编号并避免冲突。

## 用法 {#usage}

扩展声明是扩展范围的一种选项。类似于 C++ 的前向声明，你可以声明扩展字段的类型、字段名和基数（单个或重复），而无需导入包含完整扩展定义的 `.proto` 文件：

```proto
edition = "2023";

message Foo {
  extensions 4 to 1000 [
    declaration = {
      number: 4,
      full_name: ".my.package.event_annotations",
      type: ".logs.proto.ValidationAnnotations",
      repeated: true },
    declaration = {
      number: 999,
      full_name: ".foo.package.bar",
      type: "int32"}];
}
```

该语法具有以下语义：

*   如果范围足够大，可以在单个扩展范围内定义多个具有不同扩展编号的 `declaration`。
*   如果扩展范围有任何声明，则该范围的*所有*扩展也必须声明。这可以防止添加未声明的扩展，并强制任何新扩展都使用声明。
*   给定的消息类型（如 `.logs.proto.ValidationAnnotations`）不需要事先定义或导入。只需检查它是否是一个有效的名称，可能在其他 `.proto` 文件中定义。
*   当此或其他 `.proto` 文件为该消息（`Foo`）定义扩展且名称或编号匹配时，将强制扩展的编号、类型和全名与此处的前向声明一致。

{{% alert title="警告" color="warning" %}}
避免对扩展范围组（如 `extensions 4, 999`）使用声明。目前尚不清楚声明适用于哪个扩展范围，并且目前不支持。{{% /alert %}}

扩展声明期望有两个不同包的扩展字段：

```proto
package my.package;
extend Foo {
  repeated logs.proto.ValidationAnnotations event_annotations = 4;
}
```

```proto
package foo.package;
extend Foo {
  optional int32 bar = 999;
}
```

### 保留声明 {#reserved}

扩展声明可以标记为 `reserved: true`，表示该扩展已不再使用且扩展定义已被删除。**不要删除扩展声明或编辑其 `type` 或 `full_name` 值**。

此 `reserved` 标签与常规字段的 reserved 关键字不同，**不需要拆分扩展范围**。

```proto {highlight="context:reserved"}
edition = "2023";

message Foo {
  extensions 4 to 1000 [
    declaration = {
      number: 500,
      full_name: ".my.package.event_annotations",
      type: ".logs.proto.ValidationAnnotations",
      reserved: true }];
}
```

如果扩展字段定义使用了声明中 `reserved` 的编号，则编译会失败。

## 在 descriptor.proto 中的表示 {#representation}

扩展声明在 descriptor.proto 中表示为 `proto2.ExtensionRangeOptions` 的字段：

```proto
message ExtensionRangeOptions {
  message Declaration {
    optional int32 number = 1;
    optional string full_name = 2;
    optional string type = 3;
    optional bool reserved = 5;
    optional bool repeated = 6;
  }
  repeated Declaration declaration = 2;
}
```

## 反射字段查找 {#reflection}

扩展声明*不会*通过常规字段查找函数（如 `Descriptor::FindFieldByName()` 或 `Descriptor::FindFieldByNumber()`）返回。像扩展一样，它们可通过扩展查找例程（如 `DescriptorPool::FindExtensionByName()`）发现。这是一个明确的设计选择，反映了声明不是定义，且没有足够信息返回完整的 `FieldDescriptor`。

从 TextFormat 和 JSON 的角度来看，声明的扩展仍然像常规扩展一样工作。这也意味着将现有字段迁移为声明的扩展时，需要首先迁移对该字段的反射性使用。

## 使用扩展声明分配编号 {#recommendation}

扩展像普通字段一样使用字段编号，因此每个扩展都必须分配一个在父消息中唯一的编号。我们建议使用扩展声明在父消息中声明每个扩展的字段编号和类型。扩展声明作为父消息所有扩展的注册表，protoc 会强制没有字段编号冲突。添加新扩展时，通常只需将上一个扩展编号加一即可。

**提示：** 关于 `MessageSet` 有[特殊指导](#message-set)，提供了脚本帮助选择下一个可用编号。

每当你删除扩展时，请确保将字段编号标记为 `reserved`，以消除意外重复使用的风险。

此约定仅为建议——protobuf 团队无意强制所有可扩展消息都遵循它。如果你作为可扩展 proto 的所有者不想通过扩展声明协调扩展编号，可以选择通过其他方式协调。但请务必小心，因为扩展编号的意外重复使用可能导致严重问题。

一种规避该问题的方法是完全避免使用扩展，改用 [`google.protobuf.Any`](./programming-guides/proto3/#any)。对于前端存储的 API 或客户端关心 proto 内容但接收系统不关心的透传系统，这可能是一个不错的选择。

### 重复使用扩展编号的后果 {#reusing}

扩展是在容器消息之外定义的字段，通常在单独的 .proto 文件中。这种定义的分布使得两个开发者很容易意外为同一个扩展字段编号创建不同的定义。

更改扩展定义的后果与标准字段相同。重复使用字段编号会导致如何从 wire 格式解码 proto 时产生歧义。protobuf 的 wire 格式非常精简，无法很好地检测使用一种定义编码、用另一种定义解码的字段。

这种歧义可能很快就会显现，比如客户端和服务器分别使用不同扩展定义进行通信。

也可能在较长时间后显现，比如用一种扩展定义编码并存储数据，之后用第二种扩展定义检索和解码。如果第一个扩展定义在数据编码和存储后被删除，这种长期情况可能难以诊断。

其后果可能包括：

1.  解析错误（最好的情况）。
2.  泄露 PII / SPII —— 如果用一种扩展定义写入 PII 或 SPII，再用另一种扩展定义读取。
3.  数据损坏 —— 如果用“错误”的定义读取数据、修改并重写。

数据定义歧义几乎肯定会导致调试时间成本，甚至可能导致需要数月才能清理的数据泄漏或损坏。

## 使用建议

### 永远不要删除扩展声明 {#never-delete}

删除扩展声明会导致将来意外重复使用的风险。如果扩展不再处理且定义已删除，可以[标记为 reserved](#reserved)。

### 永远不要为新扩展声明使用 `reserved` 列表中的字段名或编号 {#never-reuse-reserved}

保留编号可能曾用于字段或其他扩展。

不建议使用保留字段的 `full_name`，因为在使用 textproto 时可能产生歧义。

### 永远不要更改现有扩展声明的类型 {#never-change-type}

更改扩展字段类型可能导致数据损坏。

如果扩展字段是枚举或消息类型，且该类型被重命名，则更新声明名称是安全且必要的。为避免破坏，类型、扩展字段定义和扩展声明的更新应在同一次提交中完成。

### 重命名扩展字段时要小心 {#caution-renaming}

虽然重命名扩展字段对 wire 格式没有影响，但可能会破坏 JSON 和 TextFormat 解析。

