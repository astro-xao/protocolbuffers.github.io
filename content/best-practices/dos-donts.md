+++
title = "Proto 最佳实践"
weight = 90
description = "分享编写 Protocol Buffers 的最佳实践。"
type = "docs"
aliases = "/programming-guides/dos-donts"
+++

客户端和服务端永远不会在完全相同的时间更新——即使你尝试同时更新它们。某一方可能会被回滚。不要假设你可以做破坏性更改，并且客户端和服务端会保持同步。

<a id="dont-re-use-a-tag-number"></a>

## **不要**重复使用标签号 {#reuse-number}

绝不要重复使用标签号。这会导致反序列化出错。即使你认为没有人在用该字段，也不要重复使用标签号。如果更改曾经上线过，可能在某些日志中还存在序列化的 proto 版本。或者在其他服务器上的旧代码会因此崩溃。

<a id="do-reserve-tag-numbers-for-deleted-fields"></a>

## **要**为已删除字段保留标签号 {#reserve-tag-numbers}

当你删除一个不再使用的字段时，应该保留其标签号，防止将来被误用。只需 `reserved 2, 3;` 即可，无需指定类型（这样还能减少依赖！）。你也可以保留名称，避免回收已删除的字段名：`reserved "foo", "bar";`。

<a id="do-reserve-numbers-for-deleted-enum-values"></a>

## **要**为已删除枚举值保留编号 {#reserve-deleted-numbers}

当你删除一个不再使用的枚举值时，应该保留其编号，防止将来被误用。只需 `reserved 2, 3;` 即可。你也可以保留名称，避免回收已删除的值名：`reserved "FOO", "BAR";`。

<a id="do-put-new-enum-aliases-last"></a>

## **要**将新的枚举别名放在最后 {#enum-aliases-last}

添加新的枚举别名时，应将新名称放在最后，以便服务有时间更新。

如果要安全地移除原始名称（如果它被用于数据交换，实际上[不推荐](#text-format-interchange)），你必须执行以下步骤：

*   在旧名称下方添加新名称，并弃用旧名称（序列化器仍会使用旧名称）
*   所有解析器都升级后，交换两个名称的顺序（序列化器开始使用新名称，解析器接受两者）
*   所有序列化器都升级后，可以删除已弃用的名称。

> **注意：** 理论上客户端不应使用旧名称进行数据交换，但对于广泛使用的枚举名称，仍建议遵循上述步骤。

<a id="dont-change-the-type-of-a-field"></a>

## **不要**更改字段类型 {#change-type}

几乎不要更改字段类型；这会导致反序列化出错，与重复使用标签号类似。
[protobuf 文档](/programming-guides/proto2#updating)
列举了极少数可以更改的情况（如 `int32`、`uint32`、`int64` 和 `bool` 之间）。但更改字段的消息类型**会导致破坏**，除非新消息是旧消息的超集。

<a id="dont-add-a-required-field"></a>

## **不要**添加必填字段 {#add-required}

绝不要添加必填字段，建议用 `// required` 注释来说明 API 合约。必填字段被认为有害，已在 proto3 中完全移除。所有字段应为 optional 或 repeated。你无法预知消息类型会存在多久，也无法预知四年后某人是否被迫用空字符串或零来填充你定义的必填字段。

对于 proto3，没有 `required` 字段，因此此建议不适用。

<a id="dont-make-a-message-with-lots-of-fields"></a>

## **不要**创建包含大量字段的消息 {#lots-of-fields}

不要创建包含“很多”（比如上百个）字段的消息。在 C++ 中，每个字段无论是否被赋值，都会增加大约 65 位的内存占用（8 字节指针，如果声明为 optional，还会有一个位用于标记是否被赋值）。当 proto 过大时，生成的代码甚至可能无法编译（例如 Java 对方法大小有限制）。

<a id="do-include-an-unspecified-value-in-an-enum"></a>

## **要**在枚举中包含未指定值 {#unspecified-enum}

枚举应在声明的第一个值中包含默认的 `FOO_UNSPECIFIED`。当 proto2 枚举添加新值时，旧客户端会将该字段视为未设置，getter 会返回默认值或第一个声明的值（如果没有默认值）。为保证与 [proto 枚举][proto-enums] 的一致性，第一个声明的枚举值应为默认的 `FOO_UNSPECIFIED`，且编号为 0。不要将默认值声明为有实际意义的值，这有助于协议随时间演进。所有声明在同一消息下的枚举值在 C++ 命名空间中相同，因此应使用枚举名作为前缀，避免编译错误。如果不需要跨语言常量，`int32` 能保留未知值且生成的代码更少。注意 [proto 枚举][proto-enums] 要求第一个值为 0，并且可以对未知枚举值进行序列化和反序列化。

[example-unspecified]: http://cs/#search/&q=file:proto%20%22_UNSPECIFIED%20=%200%22&type=cs
[proto-enums]: /programming-guides/proto2#enum

<a id="dont-use-cc-macro-constants-for-enum-values"></a>

## **不要**将 C/C++ 宏常量用作枚举值 {#macro-constants}

使用 C++ 语言已定义的单词（如 `math.h` 头文件中的内容）可能导致编译错误，尤其当 `#include` 某些头文件在 `.proto.h` 之前。避免使用如 "`NULL`"、"`NAN`"、"`DOMAIN`" 等宏常量作为枚举值。

<a id="do-use-well-known-types-and-common-types"></a>

## **要**使用知名类型和通用类型 {#well-known-common}

强烈建议使用以下常见、共享类型。例如，不要在代码中使用 `int32 timestamp_seconds_since_epoch` 或 `int64 timeout_millis`，当已有合适的通用类型时！

<a id="well-known-types"></a><a id="common-types"></a>

*   [`duration`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/duration.proto)
    表示有符号、固定长度的时间段（如 42s）。
*   [`timestamp`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/timestamp.proto)
    表示独立于时区和日历的时间点（如 2017-01-15T01:30:15.01Z）。
*   [`interval`](https://github.com/googleapis/googleapis/blob/master/google/type/interval.proto)
    表示独立于时区和日历的时间区间（如 2017-01-15T01:30:15.01Z - 2017-01-16T02:30:15.01Z）。
*   [`date`](https://github.com/googleapis/googleapis/blob/master/google/type/date.proto)
    表示完整的日历日期（如 2005-09-19）。
*   [`month`](https://github.com/googleapis/googleapis/blob/master/google/type/month.proto)
    表示一年中的月份（如四月）。
*   [`dayofweek`](https://github.com/googleapis/googleapis/blob/master/google/type/dayofweek.proto)
    表示星期几（如星期一）。
*   [`timeofday`](https://github.com/googleapis/googleapis/blob/master/google/type/timeofday.proto)
    表示一天中的时间（如 10:42:23）。
*   [`field_mask`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/field_mask.proto)
    表示一组符号字段路径（如 f.b.d）。
*   [`postal_address`](https://github.com/googleapis/googleapis/blob/master/google/type/postal_address.proto)
    表示邮政地址（如 1600 Amphitheatre Parkway Mountain View, CA 94043 USA）。
*   [`money`](https://github.com/googleapis/googleapis/blob/master/google/type/money.proto)
    表示带货币类型的金额（如 42 USD）。
*   [`latlng`](https://github.com/googleapis/googleapis/blob/master/google/type/latlng.proto)
    表示经纬度对（如 37.386051 纬度和 -122.083855 经度）。
*   [`color`](https://github.com/googleapis/googleapis/blob/master/google/type/color.proto)
    表示 RGBA 颜色空间中的颜色。

<a id="do-define-widely-used-message-types-in-separate-files"></a>

## **要**将常用消息类型定义在单独文件中 {#separate-files}

定义 proto schema 时，每个文件应只包含一个消息、枚举、扩展、服务或循环依赖组。这样便于重构。分文件管理比从一个大文件中提取消息更容易。遵循此做法还能保持 proto 文件较小，提升可维护性。

如果这些类型会被项目外广泛使用，建议将其放在无依赖的独立文件中。这样其他人可以轻松使用这些类型，而不会引入你其他 proto 文件的传递依赖。

更多内容见
[1-1-1 规则](/best-practices/1-1-1)。

<a id="dont-change-the-default-value-of-a-field"></a>

## **不要**更改字段的默认值 {#change-default-value}

几乎不要更改 proto 字段的默认值。这会导致客户端和服务端版本不一致。当客户端读取未设置的值时，结果可能与服务端读取同一未设置值时不同，尤其当它们的构建版本跨越 proto 更改时。Proto3 已移除设置默认值的能力。

<a id="dont-go-from-repeated-to-scalar"></a>

## **不要**从 repeated 改为标量类型 {#repeated-to-scalar}

虽然不会导致崩溃，但会丢失数据。对于 JSON，repeated 与标量不一致会导致整个*消息*丢失。对于 proto3 的数值字段和 proto2 的 `packed` 字段，从 repeated 改为标量会丢失该*字段*的所有数据。对于 proto3 的非数值字段和未注解的 proto2 字段，从 repeated 改为标量会导致最后一个反序列化的值“获胜”。

从标量改为 repeated 在 proto2 和 proto3（`[packed=false]`）中是可以的，因为二进制序列化时，标量值会变成单元素列表。

<a id="do-follow-the-style-guide-for-generated-code"></a>

## **要**遵循生成代码的风格指南 {#follow-style-guide}

proto 生成的代码会在普通代码中被引用。确保 `.proto` 文件中的选项不会导致生成的代码违反风格指南。例如：

*   `java_outer_classname` 应遵循
    https://google.github.io/styleguide/javaguide.html#s5.2.2-class-names

*   `java_package` 和 `java_alt_package` 应遵循
    https://google.github.io/styleguide/javaguide.html#s5.2.1-package-names

*   `package` 虽然在没有 `java_package` 时用于 Java，但始终直接对应 C++ 命名空间，因此应遵循
    https://google.github.io/styleguide/cppguide.html#Namespace_Names。
    如有冲突，Java 使用 `java_package`。

*   `ruby_package` 应为 `Foo::Bar::Baz`，而不是 `Foo.Bar.Baz`。

<a id="never-use-text-format-messages-for-interchange"></a>

## **不要**使用文本格式消息进行数据交换 {#text-format-interchange}

文本序列化格式如 text format 和 JSON 会将字段和枚举值表示为字符串。因此，使用旧代码反序列化这些格式的 proto 时，如果字段或枚举值被重命名，或新增字段、枚举值、扩展，都会失败。数据交换时应尽量使用二进制序列化，文本格式仅用于人工编辑和调试。

如果你在 API 或数据存储中使用 proto 转换为 JSON，可能无法安全地重命名字段或枚举。

<a id="never-rely-on-serialization-stability-across-builds"></a>

## **绝不要**依赖不同构建间的序列化稳定性 {#serialization-stability}

proto 序列化的稳定性在不同二进制文件或同一二进制的不同构建间无法保证。不要依赖它，例如用于构建缓存键。

<a id="dont-generate-java-protos-in-the-same-java-package-as-other-code"></a>

## **不要**将 Java proto 生成在与其他代码相同的包中 {#generate-java-protos}

应将 Java proto 源码生成到与手写 Java 源码不同的包中。`package`、`java_package` 和 `java_alt_api_package` 选项控制
[生成的 Java 源码的输出位置](/reference/java/java-generated#package)。
确保手写 Java 代码不在同一包下。常见做法是将 proto 生成到项目的 `proto` 子包中，该包**只**包含 proto（即没有手写源码）。

## 避免将语言关键字用作字段名 {#avoid-keywords}

如果消息、字段、枚举或枚举值的名称是目标语言的关键字，protobuf 可能会更改字段名，并且访问方式可能与普通字段不同。例如，参见
[关于 Python 的警告](/reference/python/python-generated#keyword-conflicts)。

还应避免在文件路径中使用关键字，这也可能导致问题。

## **要**使用 java_outer_classname {#java-outer-classname}

每个 proto schema 定义文件都应设置 `java_outer_classname` 选项，其值为 `.proto` 文件名去掉点号后转为 TitleCase。例如，文件 `student_record_request.proto` 应设置：

```java
option java_outer_classname = "StudentRecordRequestProto";
```

## 附录 {#appendix}

### API 最佳实践 {#api-best-practices}

本文仅列出极易导致破坏的更改。关于如何优雅地扩展 proto API 的更高层次建议，请参见
[API 最佳实践](/best-practices/api)。

