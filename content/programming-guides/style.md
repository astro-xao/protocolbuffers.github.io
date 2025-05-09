+++
title = "风格指南"
weight = 50
description = "为如何最佳组织您的 proto 定义提供指导。"
type = "docs"
+++

本文档为 `.proto` 文件提供了风格指南。遵循这些约定，可以让您的协议缓冲区消息定义及其对应的类保持一致且易于阅读。

## 标准文件格式 {#standard-file-formatting}

*   保持每行长度不超过 80 个字符。
*   使用 2 个空格缩进。
*   字符串建议使用双引号。

## 文件结构 {#file-structure}

文件应命名为 `lower_snake_case.proto`。

所有文件应按以下顺序排列：

1.  许可证头（如适用）
1.  文件概述
1.  语法声明
1.  包声明
1.  导入（排序）
1.  文件选项
1.  其他内容

## 标识符命名风格 {#identifier}

Protobuf 标识符使用以下命名风格之一：

1.  TitleCase
  *   包含大写字母、小写字母和数字
  *   首字母为大写字母
  *   每个单词的首字母大写
1.  lower_snake_case
  *   包含小写字母、下划线和数字
  *   单词之间用单个下划线分隔
1.  UPPER_SNAKE_CASE
  *   包含大写字母、下划线和数字
  *   单词之间用单个下划线分隔
1.  camelCase
  *   包含大写字母、小写字母和数字
  *   首字母为小写字母
  *   每个后续单词的首字母大写
  *   **注意：** 下述风格指南不建议在 .proto 文件中使用 camelCase，仅为说明某些语言生成的代码可能会将标识符转换为此风格。

所有情况下，缩写应视为单个单词：使用 `GetDnsRequest` 而不是 `GetDNSRequest`，使用 `dns_request` 而不是 `d_n_s_request`。

#### 标识符中的下划线 {#underscores}

不要在名称的开头或结尾使用下划线。任何下划线后面都应跟字母（而不是数字或第二个下划线）。

此规则的动机在于，每种 protobuf 语言实现可能会将标识符转换为本地语言风格：.proto 文件中的 `song_id` 字段在不同语言中可能分别变为 `SongId`、`songId` 或 `song_id`。

仅在字母前使用下划线，可以避免名称在一种风格下不同，但转换为另一种风格后发生冲突的情况。

例如，`DNS2` 和 `DNS_2` 转换为 TitleCase 都会变为 `Dns2`。允许这两种命名会导致在某些语言中生成的代码保持原有 UPPER_SNAKE_CASE 风格并广泛使用，后来在另一种语言中转换为 TitleCase 时发生冲突。

因此，应使用 `XYZ2` 或 `XYZ_V2`，而不是 `XYZ_2` 或 `XYZ_2V`。

## 包 {#packages}

包名应使用点分隔的 lower_snake_case。

多单词包名可以使用 lower_snake_case 或点分隔（大多数语言中点分隔包名会作为嵌套包/命名空间）。

包名应尽量简短且唯一，基于项目名称。包名不应为 Java 包（如 `com.x.y`）；应使用 `x.y` 作为包名，并根据需要使用 `java_package` 选项。

## 消息名 {#message-names}

消息名应使用 TitleCase。

```proto
message SongRequest {
}
```

## 字段名 {#field-names}

字段名（包括扩展）应使用 snake_case。

重复字段应使用复数命名。

```proto
string song_name = 1;
repeated Song songs = 2;
```

## oneof 名称 {#oneof-names}

oneof 名称应使用 lower_snake_case。

```proto
oneof song_id {
  string song_human_readable_id = 1;
  int64 song_machine_id = 2;
}
```

## 枚举 {#enums}

枚举类型名应使用 TitleCase。

枚举值名应使用 UPPER_SNAKE_CASE。

```proto
enum FooBar {
  FOO_BAR_UNSPECIFIED = 0;
  FOO_BAR_FIRST_VALUE = 1;
  FOO_BAR_SECOND_VALUE = 2;
}
```

第一个枚举值应为零值，并以 `_UNSPECIFIED` 或 `_UNKNOWN` 结尾。该值可用作未知/默认值，并应与任何语义值区分。更多关于未指定枚举值的信息，请参见[Proto 最佳实践页面](./best-practices/dos-donts#unspecified-enum)。

#### 枚举值前缀 {#enum-value-prefixing}

枚举值在语义上不受其包含的枚举名作用域限制，因此两个同级枚举中不能有相同的值名。例如，以下内容会被 protoc 拒绝，因为两个枚举中的 `SET` 值被认为处于同一作用域：

```proto
enum CollectionType {
  COLLECTION_TYPE_UNSPECIFIED = 0;
  SET = 1;
  MAP = 2;
  ARRAY = 3;
}

enum TennisVictoryType {
  TENNIS_VICTORY_TYPE_UNSPECIFIED = 0;
  GAME = 1;
  SET = 2;
  MATCH = 3;
}
```

当枚举定义在文件顶层（未嵌套在消息定义中）时，名称冲突风险较高；此时同级包括其他文件中设置了相同包的枚举，protoc 可能无法在代码生成时检测到冲突。

为避免这些风险，强烈建议：

*   为每个值添加枚举名前缀（转换为 UPPER_SNAKE_CASE）
*   将枚举嵌套在包含消息中

任一方式都可避免冲突，但建议优先使用带前缀的顶层枚举，而不是仅为规避冲突而创建消息。由于某些语言不支持在“结构体”类型中定义枚举，优先使用前缀可确保各语言绑定方式一致。

## 服务 {#services}

服务名和方法名应使用 TitleCase。

```proto
service FooService {
  rpc GetSomething(GetSomethingRequest) returns (GetSomethingResponse);
  rpc ListSomething(ListSomethingRequest) returns (ListSomethingResponse);
}
```

更多服务相关建议，请参见
[为每个方法创建唯一的 Proto](./best-practices/api#unique-protos)
和
[不要在顶级请求或响应 Proto 中包含原始类型](./programming-guides/api#dont-include-primitive-types)
以及 Proto 最佳实践中的
[在单独文件中定义消息类型](./best-practices/dos-donts#separate-files)。

## 应避免事项 {#avoid}

### Required 字段 {#required}

Required 字段用于强制在解析字节流时必须设置某字段，否则拒绝解析消息。该约束通常不会在内存中构建消息时强制执行。Required 字段在 proto3 中已被移除。

虽然在模式层面强制 Required 字段直观上很有吸引力，但 protobuf 的主要设计目标之一是支持长期的模式演进。无论某字段现在看起来多么“必须”，未来都有可能不再需要（例如 `int64 user_id` 未来可能迁移为 `UserId user_id`）。

尤其是在中间件服务器可能转发其无需处理的消息时，`required` 的语义对长期演进目标造成了很大阻碍，因此现在强烈不建议使用。

参见
[Required 已强烈弃用](./programming-guides/proto2#required-deprecated)。

### Groups {#groups}

Groups 是嵌套消息的另一种语法和线格式。Groups 在 proto2 中已弃用，在 proto3 中被移除。应使用嵌套消息定义及其字段代替 group 语法。

参见 [groups](./programming-guides/proto2#groups)。
