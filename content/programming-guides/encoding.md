+++
title = "编码"
weight = 60
description = "解释 Protocol Buffers 如何将数据编码到文件或传输到网络。"
type = "docs"
+++

本文档介绍了 protocol buffer 的*线格式*（wire format），它定义了消息在网络上传输和在磁盘上占用空间的详细方式。你在应用中使用 protocol buffer 时通常不需要了解这些细节，但如果你需要做优化，这些信息会很有用。

如果你已经了解相关概念但需要参考资料，可以直接跳到[速查表](#cheat-sheet)部分。

[Protoscope](https://github.com/protocolbuffers/protoscope) 是一种非常简单的语言，用于描述底层线格式的片段，我们将用它来为各种消息的编码提供可视化参考。Protoscope 的语法由一系列*标记*（token）组成，每个标记都精确编码为特定的字节序列。

例如，反引号用于表示原始十六进制字面量，如 `` `70726f746f6275660a` ``。它会被编码为字面量中表示的确切字节。引号用于表示 UTF-8 字符串，如 `"Hello, Protobuf!"`。这个字面量等同于 `` `48656c6c6f2c2050726f746f62756621` ``（如果你仔细观察，会发现它由 ASCII 字节组成）。我们会在讨论线格式的过程中介绍更多 Protoscope 语言的内容。

Protoscope 工具还可以将已编码的 protocol buffer 以文本形式导出。参见 https://github.com/protocolbuffers/protoscope/tree/main/testdata 获取示例。

本文所有示例均假设你使用的是 2023 版或更高版本。

## 一个简单的消息 {#simple}

假设你有如下非常简单的消息定义：

```proto
message Test1 {
  int32 a = 1;
}
```

在应用中，你创建了一个 `Test1` 消息并将 `a` 设为 150。然后你将消息序列化到输出流。如果你能检查编码后的消息，你会看到三个字节：

```proto
08 96 01
```

目前为止，看起来很小且全是数字——但这代表什么？如果你用 Protoscope 工具导出这些字节，你会看到类似 `1: 150` 的内容。它是如何知道消息内容的？

## 基于 128 的变长整数（Varint）{#varints}

变长整数（*varint*）是线格式的核心。它允许用 1 到 10 个字节编码无符号 64 位整数，较小的值使用更少的字节。

varint 的每个字节都有一个*续位*，用于指示后面的字节是否属于同一个 varint。这个续位是字节的*最高有效位*（MSB，有时也叫*符号位*）。低 7 位是有效载荷；最终的整数通过将各字节的 7 位有效载荷拼接起来得到。

例如，数字 1 编码为 `` `01` ``——它只有一个字节，所以 MSB 没有被设置：

```proto
0000 0001
^ msb
```

而 150 编码为 `` `9601` ``——稍微复杂一些：

```proto
10010110 00000001
^ msb    ^ msb
```

如何确定这是 150？首先去掉每个字节的 MSB，因为它只是用来告诉我们数字是否结束（如你所见，第一个字节的 MSB 被设置，表示 varint 不止一个字节）。这些 7 位有效载荷是小端序。转换为大端序，拼接后按无符号 64 位整数解释：

```proto
10010110 00000001        // 原始输入
 0010110  0000001        // 去掉续位
 0000001  0010110        // 转为大端序
   00000010010110        // 拼接
 128 + 16 + 4 + 2 = 150  // 解释为无符号 64 位整数
```

由于 varint 对 protocol buffer 至关重要，在 protoscope 语法中，我们直接用整数表示 varint。`150` 就等同于 `` `9601` ``。

## 消息结构 {#structure}

protocol buffer 消息是一系列键值对。消息的二进制版本只用字段号作为键——字段的名称和声明类型只能在解码端通过消息类型定义（即 `.proto` 文件）确定。Protoscope 无法获取这些信息，因此只能提供字段号。

消息编码时，每个键值对会被转换为一个*记录*，包含字段号、线类型和有效载荷。线类型告诉解析器后面的有效载荷有多大。这允许旧的解析器跳过它们不理解的新字段。这种方案有时被称为[标签-长度-值](https://zh.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value)（TLV）。

线类型有六种：`VARINT`、`I64`、`LEN`、`SGROUP`、`EGROUP` 和 `I32`

ID  | 名称    | 用于
--- | ------ | --------------------------------------------------------
0   | VARINT | int32, int64, uint32, uint64, sint32, sint64, bool, enum
1   | I64    | fixed64, sfixed64, double
2   | LEN    | string, bytes, embedded messages, packed repeated fields
3   | SGROUP | group start（已弃用）
4   | EGROUP | group end（已弃用）
5   | I32    | fixed32, sfixed32, float

记录的“标签”通过字段号和线类型组合成一个 varint，公式为 `(field_number << 3) | wire_type`。换句话说，解码表示字段的 varint 后，低 3 位是线类型，其余位是字段号。

让我们再看一下前面的简单例子。你现在知道流中的第一个数字总是 varint 键，这里是 `` `08` ``，去掉 MSB 后：

```proto
000 1000
```

取最后三位得到线类型（0），右移三位得到字段号（1）。Protoscope 用整数加冒号和线类型表示标签，所以这些字节可以写作 `1:VARINT`。

因为线类型是 0，即 `VARINT`，我们知道需要解码一个 varint 作为有效载荷。如上所述，字节 `` `9601` `` 解码为 150，得到我们的记录。用 Protoscope 表示为 `1:VARINT 150`。

如果标签后有空格，Protoscope 可以推断类型。它会查看下一个标记并猜测你的意图（详细规则见 [Protoscope's language.txt](https://github.com/protocolbuffers/protoscope/blob/main/language.txt)）。例如，`1: 150` 后面紧跟 varint，Protoscope 推断类型为 `VARINT`。如果写 `2: {}`，它看到 `{` 会猜测为 `LEN`；写 `3: 5i32` 会猜测为 `I32`，等等。

## 更多整数类型 {#int-types}

### 布尔和枚举 {#bools-and-enums}

布尔和枚举都按 `int32` 编码。布尔值总是编码为 `` `00` `` 或 `` `01` ``。在 Protoscope 中，`false` 和 `true` 是这两个字节串的别名。

### 有符号整数 {#signed-ints}

如前所述，所有与线类型 0 相关的 protocol buffer 类型都按 varint 编码。但 varint 是无符号的，所以不同的有符号类型（`sint32`、`sint64` 与 `int32`、`int64`）对负数的编码方式不同。

`intN` 类型将负数按二进制补码编码，这意味着作为无符号 64 位整数时，其最高位被设置。因此，*必须使用全部十个字节*。例如，`-2` 被 protoscope 转换为

```proto
11111110 11111111 11111111 11111111 11111111
11111111 11111111 11111111 11111111 00000001
```

这是 2 的*二进制补码*，在无符号运算中定义为 `~0 - 2 + 1`，其中 `~0` 是全 1 的 64 位整数。理解为什么会产生这么多 1 是一个有趣的练习。

<!-- mdformat off(the asterisks cause bullets) -->
`sintN` 使用“ZigZag”编码而不是二进制补码来编码负数。正整数 `p` 编码为 `2 * p`（偶数），负整数 `n` 编码为 `2 * |n| - 1`（奇数）。编码结果在正负数之间“之”字形切换。例如：
<!-- mdformat on -->

原始有符号值 | 编码后
----------- | -------
0           | 0
-1          | 1
1           | 2
-2          | 3
...         | ...
0x7fffffff  | 0xfffffffe
-0x80000000 | 0xffffffff

换句话说，每个值 `n` 的编码方式为

```
(n << 1) ^ (n >> 31)
```

对于 `sint32`，或

```
(n << 1) ^ (n >> 63)
```

用于 64 位版本。

解析 `sint32` 或 `sint64` 时，会将其值解码回原始有符号值。

在 protoscope 中，整数后缀 `z` 表示用 ZigZag 编码。例如，`-500z` 等同于 varint `999`。

### 非 varint 数字类型 {#non-varints}

非 varint 数值类型很简单——`double` 和 `fixed64` 使用线类型 `I64`，告诉解析器后面是固定的 8 字节数据。我们可以用 `5: 25.4` 指定一个 `double` 记录，或用 `6: 200i64` 指定一个 `fixed64` 记录。两种情况下，省略显式线类型会默认推断为 `I64`。

同理，`float` 和 `fixed32` 使用线类型 `I32`，表示后面是 4 字节。语法是在数字后加 `i32` 后缀。`25.4i32` 和 `200i32` 都会输出 4 字节。标签类型会被推断为 `I32`。

## 长度前缀记录 {#length-types}

*长度前缀*是线格式中的另一个重要概念。`LEN` 线类型的长度是动态的，由标签后紧跟的 varint 指定，然后是有效载荷。

考虑如下消息结构：

```proto
message Test2 {
  string b = 2;
}
```

字段 `b` 是字符串，字符串用 `LEN` 编码。如果我们将 `b` 设为 `"testing"`，它会被编码为字段号 2 的 `LEN` 记录，内容为 ASCII 字符串 `"testing"`。结果是 `` `120774657374696e67` ``。拆分字节如下：

```proto
12 07 [74 65 73 74 69 6e 67]
```

标签 `` `12` `` 是 `00010 010`，即 `2:LEN`。后面的字节是 int32 varint `7`，再后面七个字节是 `"testing"` 的 UTF-8 编码。int32 varint 意味着字符串最大长度为 2GB。

在 Protoscope 中，这写作 `2:LEN 7 "testing"`。不过，重复写字符串长度可能不方便（在 Protoscope 文本中，字符串已经用引号包裹）。用大括号包裹 Protoscope 内容会自动生成长度前缀：`{"testing"}` 等价于 `7 "testing"`。`{}` 总是被字段推断为 `LEN` 记录，所以可以简写为 `2: {"testing"}`。

`bytes` 字段编码方式相同。

### 子消息 {#embedded}

子消息字段同样使用 `LEN` 线类型。下面是一个嵌套了我们最初例子消息 `Test1` 的消息定义：

```proto
message Test3 {
  Test1 c = 3;
}
```

如果 `Test1` 的 `a` 字段（即 `Test3` 的 `c.a` 字段）设为 150，编码结果为 ``1a03089601``。拆分如下：

```proto
 1a 03 [08 96 01]
```

最后三个字节（`[]` 内）正好是我们[第一个例子](#simple)中的字节。这些字节前面是一个 `LEN` 类型标签和长度 3，与字符串编码方式完全相同。

在 Protoscope 中，子消息写法非常简洁。``1a03089601`` 可写作 `3: {1: 150}`。

## 缺失元素 {#optional}

缺失字段的编码很简单：如果字段不存在，就不写入记录。这意味着“庞大”的 proto 只要设置了少量字段，编码结果会非常稀疏。

<span id="packed"></span>

## 重复元素 {#repeated}

从 2023 版开始，原始类型的 `repeated` 字段（任何[标量类型](./programming-guides/proto2#scalar)，不包括 `string` 或 `bytes`）默认采用[打包](./editions/features#repeated_field_encoding)编码。

打包的 `repeated` 字段不会为每个元素单独编码记录，而是编码为一个包含所有元素的 `LEN` 记录。解码时，从 `LEN` 记录中依次解出每个元素，直到有效载荷耗尽。下一个元素的起始位置由前一个元素的长度决定，而长度又取决于字段类型。例如：

```proto
message Test4 {
  string d = 4;
  repeated int32 e = 6;
}
```

我们构造一个 `Test4` 消息，`d` 设为 `"hello"`，`e` 设为 `1`、`2`、`3`，编码结果*可能*为 `` `3206038e029ea705` ``，Protoscope 写法为：

```proto
4: {"hello"}
6: {3 270 86942}
```

但如果 repeated 字段被设置为展开（覆盖默认打包状态）或不可打包（如字符串和消息），则每个值单独编码记录。并且，`e` 的记录不必连续出现，可以与其他字段交错；只有同一字段的记录顺序被保留。因此，也可以这样：

```proto
5: 1
5: 2
4: {"hello"}
5: 3
```

只有原始数值类型的 repeated 字段可以声明为“打包”。这些类型通常使用 `VARINT`、`I32` 或 `I64` 线类型。

注意，虽然通常没有必要为打包 repeated 字段编码多个键值对，但解析器必须能接受多个键值对。在这种情况下，有效载荷应当拼接。每对必须包含完整数量的元素。如下编码也是有效的：

```proto
6: {3 270}
6: {86942}
```

protocol buffer 解析器必须能将打包和非打包 repeated 字段互相兼容解析。这允许你在向现有字段添加 `[packed=true]` 时保持前后兼容。

### Oneof {#oneofs}

[`Oneof` 字段](./programming-guides/proto2#oneof)的编码方式与不在 oneof 中时相同。oneof 的规则与其在线格式上的表示无关。

### 最后一个获胜 {#last-one-wins}

通常，编码消息不会有多个非 repeated 字段实例。但解析器应能处理这种情况。对于数值类型和字符串，如果同一字段出现多次，解析器接受*最后*出现的值。对于嵌套消息字段，解析器会合并同一字段的多个实例，类似于 `Message::MergeFrom` 方法——即后一个实例的所有单一标量字段替换前一个，单一嵌套消息合并，repeated 字段拼接。这样，解析两个编码消息的拼接结果与分别解析后合并对象的结果完全一致。即：

```cpp
MyMessage message;
message.ParseFromString(str1 + str2);
```

等价于：

```cpp
MyMessage message, message2;
message.ParseFromString(str1);
message2.ParseFromString(str2);
message.MergeFrom(message2);
```

这个特性有时很有用，因为即使你不知道消息类型，也可以通过拼接合并两个消息。

### Map {#maps}

Map 字段只是特殊 repeated 字段的简写。如果有

```proto
message Test6 {
  map<string, int32> g = 7;
}
```

实际上等同于

```proto
message Test6 {
  message g_Entry {
    string key = 1;
    int32 value = 2;
  }
  repeated g_Entry g = 7;
}
```

因此，map 的编码方式与 repeated 消息字段完全相同：作为一系列 `LEN` 类型记录，每条记录有两个字段。

## Groups {#groups}

Groups 是已弃用的特性，不应再使用，但它们仍然存在于线格式中，这里简单介绍一下。

Group 有点像子消息，但用特殊标签而不是 `LEN` 前缀分隔。每个 group 在消息中有一个字段号，用于这些特殊标签。

字段号为 `8` 的 group 以 `8:SGROUP` 标签开始。`SGROUP` 记录没有有效载荷，仅表示 group 开始。列出 group 内所有字段后，用对应的 `8:EGROUP` 标签结束。`EGROUP` 记录也没有有效载荷，所以 `8:EGROUP` 就是整个记录。group 字段号必须匹配。如果遇到 `7:EGROUP` 而期望 `8:EGROUP`，消息格式错误。

Protoscope 提供了便捷的 group 写法。你可以不用写

```proto
8:SGROUP
  1: 2
  3: {"foo"}
8:EGROUP
```

而直接写

```proto
8: !{
  1: 2
  3: {"foo"}
}
```

这会自动生成合适的 group 起止标记。`!{}` 语法只能紧跟未指定类型的标签表达式，如 `8:`。

## 字段顺序 {#order}

字段号在 `.proto` 文件中可以任意顺序声明。顺序不会影响消息的序列化方式。

消息序列化时，已知或[未知字段](./programming-guides/proto2#updating)的顺序没有保证。序列化顺序是实现细节，具体实现可能随时变化。因此，protocol buffer 解析器必须能解析任意顺序的字段。

### 含义 {#implications}

*   不要假设序列化消息的字节输出是稳定的。对于包含其他 protocol buffer 消息的字节字段的消息尤其如此。
*   默认情况下，对同一 protocol buffer 消息实例重复调用序列化方法，可能不会产生相同的字节输出。即，默认序列化不是确定性的。
    *   确定性序列化只保证同一二进制文件的输出一致。不同版本的二进制文件输出可能不同。
*   以下检查对于 protocol buffer 消息实例 `foo` 可能失败：
    *   `foo.SerializeAsString() == foo.SerializeAsString()`
    *   `Hash(foo.SerializeAsString()) == Hash(foo.SerializeAsString())`
    *   `CRC(foo.SerializeAsString()) == CRC(foo.SerializeAsString())`
    *   `FingerPrint(foo.SerializeAsString()) == FingerPrint(foo.SerializeAsString())`
*   以下场景中，逻辑等价的 protocol buffer 消息 `foo` 和 `bar` 可能序列化为不同的字节输出：
    *   `bar` 由旧服务器序列化，部分字段被视为未知。
    *   `bar` 由不同编程语言实现的服务器序列化，字段顺序不同。
    *   `bar` 有字段以非确定性方式序列化。
    *   `bar` 有字段存储了 protocol buffer 消息的序列化字节输出，而该消息序列化方式不同。
    *   `bar` 由新服务器序列化，因实现变更字段顺序不同。
    *   `foo` 和 `bar` 是同一组消息以不同顺序拼接的结果。

## 编码 proto 的大小限制 {#size-limit}

序列化后的 proto 必须小于 2 GiB。许多 proto 实现会拒绝序列化或解析超过此限制的消息。

## 速查表 {#cheat-sheet}

以下是线格式最重要部分的便捷参考。

```none
message    := (tag value)*

tag        := (field << 3) bit-or wire_type;
                编码为 uint32 varint
value      := varint      当 wire_type == VARINT,
              i32         当 wire_type == I32,
              i64         当 wire_type == I64,
              len-prefix  当 wire_type == LEN,
              <empty>     当 wire_type == SGROUP 或 EGROUP

varint     := int32 | int64 | uint32 | uint64 | bool | enum | sint32 | sint64;
                编码为 varint（sintN 先用 ZigZag 编码）
i32        := sfixed32 | fixed32 | float;
                编码为 4 字节小端序；
                等价 C 类型（u?int32_t, float）的 memcpy
i64        := sfixed64 | fixed64 | double;
                编码为 8 字节小端序；
                等价 C 类型（u?int64_t, double）的 memcpy

len-prefix := size (message | string | bytes | packed);
                size 编码为 int32 varint
string     := 有效 UTF-8 字符串（如 ASCII）；
                最多 2GB 字节
bytes      := 任意 8 位字节序列；
                最多 2GB 字节
packed     := varint* | i32* | i64*,
                `.proto` 中指定类型的连续值
```

另见
[Protoscope 语言参考](https://github.com/protocolbuffers/protoscope/blob/main/language.txt)。

### 关键说明 {#cheat-sheet-key}

`message   := (tag value)*`
:   一条消息编码为零个或多个标签和值对。

`tag        := (field << 3) bit-or wire_type`
:   标签由 `wire_type`（最低三位）和 `.proto` 文件中定义的字段号组合而成。

`value      := varint   当 wire_type == VARINT, ...`
:   值的存储方式取决于标签中指定的 `wire_type`。

`varint     := int32 | int64 | uint32 | uint64 | bool | enum | sint32 | sint64`
:   varint 可用于存储上述任意类型。

`i32        := sfixed32 | fixed32 | float`
:   fixed32 可用于存储上述任意类型。

`i64        := sfixed64 | fixed64 | double`
:   fixed64 可用于存储上述任意类型。

`len-prefix := size (message | string | bytes | packed)`
:   长度前缀值存储为长度（编码为 varint），后跟上述任意类型。

`string     := 有效 UTF-8 字符串（如 ASCII）`
:   字符串必须使用 UTF-8 编码，最大不能超过 2GB。

`bytes      := 任意 8 位字节序列`
:   bytes 可存储自定义数据类型，最大 2GB。

`packed     := varint* | i32* | i64*`
:   当你需要存储 `.proto` 定义类型的连续值时使用 packed。标签只在第一个值前出现一次，后续值不再重复标签，从而摊薄标签开销。
