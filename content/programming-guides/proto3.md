+++
title = "语言指南 (proto 3)"
weight = 40
description = "介绍如何在您的项目中使用 Protocol Buffers 语言的 proto3 版本。"
type = "docs"
+++

本指南介绍如何使用 protocol buffer 语言来构建你的 protocol buffer 数据，包括 `.proto` 文件的语法以及如何根据 `.proto` 文件生成数据访问类。内容涵盖了 protocol buffer 语言的 **proto3** 版本。

有关**editions**语法的信息，请参见 [Protobuf Editions Language Guide](./programming-guides/editions)。

有关**proto2**语法的信息，请参见 [Proto2 Language Guide](./programming-guides/proto2)。

这是一个参考指南——如果你需要一个包含本文件中许多特性的分步示例，请参见你所选语言的 [教程](./getting-started)。

## 定义消息类型 {#simple}

首先让我们来看一个非常简单的例子。假设你想定义一个搜索请求消息格式，每个搜索请求包含一个查询字符串、你感兴趣的结果页码，以及每页的结果数。你可以使用如下的 `.proto` 文件来定义该消息类型。

```proto
syntax = "proto3";

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 results_per_page = 3;
}
```

*   文件的第一行指定你正在使用 protocol buffer 语言规范的 proto3 版本。

        *   `edition`（或 proto2/proto3 的 `syntax`）必须是文件的第一行非空、非注释内容。
        *   如果没有指定 `edition` 或 `syntax`，protocol buffer 编译器会假定你正在使用 [proto2](./programming-guides/proto2)。

*   `SearchRequest` 消息定义了三个字段（名称/值对），每个字段对应你想在该类型消息中包含的一项数据。每个字段都有名称和类型。

### 指定字段类型 {#specifying-types}

在前面的例子中，所有字段都是[标量类型](#scalar)：两个整数（`page_number` 和 `results_per_page`）和一个字符串（`query`）。你也可以为字段指定[枚举类型](#enum)和其他消息类型等复合类型。

### 分配字段编号 {#assigning}

你必须为消息定义中的每个字段分配一个介于 `1` 和 `536,870,911` 之间的编号，具体限制如下：

-   给定的编号在该消息的所有字段中**必须唯一**。
-   字段编号 `19,000` 到 `19,999` 保留给 Protocol Buffers 实现。如果你在消息中使用了这些保留编号，protocol buffer 编译器会报错。
-   你不能使用任何之前[保留](#fieldreserved)的字段编号，也不能使用已分配给[扩展](./programming-guides/proto2#extensions)的字段编号。

该编号**一旦消息类型投入使用后不可更改**，因为它用于标识 [消息的 wire 格式](./programming-guides/encoding)中的字段。
“更改”字段编号等同于删除该字段并用新编号创建一个同类型的新字段。如何正确操作，请参见[删除字段](#deleting)。

字段编号**绝不应被重复使用**。不要将字段编号从[保留列表](#fieldreserved)中移除后用于新的字段定义。详见[重复使用字段编号的后果](#consequences)。

你应将 1 到 15 号字段编号用于最常用的字段。较小的字段编号在 wire 格式中占用更少空间。例如，1 到 15 号字段编号编码时只需一个字节，16 到 2047 号字段编号编码时需两个字节。详情请参见 [Protocol Buffer 编码](./programming-guides/encoding#structure)。

#### 重复使用字段编号的后果 {#consequences}

重复使用字段编号会导致 wire 格式消息解码时产生歧义。

protocol buffer 的 wire 格式非常精简，无法检测用一种定义编码、用另一种定义解码的字段。

用一种定义编码字段，再用不同定义解码该字段，可能导致：

-   开发者花费大量时间调试
-   解析/合并错误（最好的情况）
-   泄露 PII/SPII
-   数据损坏

字段编号重复使用的常见原因：

-   字段重新编号（有时为了让字段编号更美观）。重新编号实际上等同于删除并重新添加所有涉及的字段，导致 wire 格式不兼容。
-   删除字段后未[保留](#fieldreserved)该编号，导致后续被复用。

字段编号限制为 29 位而不是 32 位，是因为有三位用于指定字段的 wire 格式。详情请参见 [编码主题](./programming-guides/encoding#structure)。

<a id="specifying-field-rules"></a>

### 指定字段基数 {#field-labels}

消息字段可以是以下几种之一：

*   *单个*：

    在 proto3 中，单个字段有两种类型：

    *   `optional`：（推荐）一个 `optional` 字段有两种可能的状态：

        *   字段已设置，包含被显式设置或从 wire 解析的值。它会被序列化到 wire。
        *   字段未设置，将返回默认值。它不会被序列化到 wire。

        你可以检查该值是否被显式设置。

        为了最大兼容性，推荐使用 `optional` 而不是 *隐式* 字段，适用于 protocol buffer 的不同版本和 proto2。

    *   *隐式*：（不推荐）隐式字段没有显式的基数字段标签，行为如下：

        *   如果字段是消息类型，则行为与 `optional` 字段相同。
        *   如果字段不是消息类型，有两种状态：

            *   字段被设置为非默认（非零）值，该值被显式设置或从 wire 解析。它会被序列化到 wire。
            *   字段被设置为默认（零）值。它不会被序列化到 wire。实际上，你无法判断该默认（零）值是被设置/解析得到还是根本未提供。更多内容见 [字段存在性](./programming-guides/field_presence)。

*   `repeated`：该字段类型在格式良好的消息中可以重复出现零次或多次。重复值的顺序会被保留。

*   `map`：这是一个键/值对字段类型。详见 [Maps](./programming-guides/encoding#maps)。

#### Repeated 字段默认使用 Packed 编码 {#use-packed}

在 proto3 中，标量数值类型的 `repeated` 字段默认使用 `packed` 编码。

你可以在 [Protocol Buffer 编码](./programming-guides/encoding#packed) 了解更多关于 `packed` 编码的信息。

#### 消息类型字段始终具有字段存在性 {#field-presence}

在 proto3 中，消息类型字段已经具有字段存在性。因此，添加 `optional` 修饰符不会改变该字段的存在性。

下面代码示例中的 `Message2` 和 `Message3` 的定义，在所有语言中生成的代码相同，在二进制、JSON 和 TextFormat 表示上没有区别：

```proto
syntax="proto3";

package foo.bar;

message Message1 {}

message Message2 {
  Message1 foo = 1;
}

message Message3 {
  optional Message1 bar = 1;
}
```

#### 格式良好的消息 {#well-formed}

“格式良好”用于描述 protocol buffer 消息时，指的是被序列化/反序列化的字节。protoc 解析器会验证 proto 定义文件是否可解析。

单个字段在 wire 格式字节中可以出现多次。解析器会接受该输入，但只有最后一次出现的字段会通过生成的绑定访问。详见 [最后一个获胜](./programming-guides/encoding#last-one-wins)。

### 添加更多消息类型 {#adding-types}

可以在一个 `.proto` 文件中定义多个消息类型。如果你要定义多个相关的消息，这很有用。例如，如果你想定义与 `SearchResponse` 消息类型对应的回复消息格式，可以将其添加到同一个 `.proto` 文件中：

```proto
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 results_per_page = 3;
}

message SearchResponse {
 ...
}
```

**合并消息会导致膨胀** 虽然可以在一个 `.proto` 文件中定义多个消息类型（如 message、enum 和 service），但如果在一个文件中定义了大量具有不同依赖的消息，会导致依赖膨胀。建议每个 `.proto` 文件包含尽可能少的消息类型。

### 添加注释 {#adding-comments}

要为 `.proto` 文件添加注释：

*   推荐在 .proto 代码元素前一行使用 C/C++/Java 风格的行尾注释 `//`
*   也接受 C 风格的内联/多行注释 `/* ... */`

    *   使用多行注释时，推荐每行前加 `*`。

```proto
/**
 * SearchRequest 表示一个搜索查询，带有分页选项，用于指示响应中包含哪些结果。
 */
message SearchRequest {
  string query = 1;

  // 请求第几页？
  int32 page_number = 2;

  // 每页返回多少结果。
  int32 results_per_page = 3;
}
```

### 删除字段 {#deleting}

如果不正确地删除字段，可能会导致严重问题。

当你不再需要某个字段，并且客户端代码中的所有引用都已删除时，可以从消息中删除该字段定义。但你**必须**[保留被删除的字段编号](#fieldreserved)。如果不保留该字段编号，未来开发者可能会复用该编号。

你还应该保留字段名，以便你的消息的 JSON 和 TextFormat 编码仍然可以解析。

<a id="fieldreserved"></a>

#### 保留字段编号 {#reserved-field-numbers}

如果你通过完全删除字段或注释掉字段来[更新](#updating)消息类型，未来开发者可能会在更新类型时复用该字段编号。这会导致严重问题，详见 [复用字段编号的后果](#consequences)。为避免这种情况，请将已删除的字段编号添加到 `reserved` 列表。

如果未来开发者尝试使用这些保留的字段编号，protoc 编译器会生成错误信息。

```proto
message Foo {
  reserved 2, 15, 9 to 11;
}
```

保留字段编号范围是包含的（`9 to 11` 等同于 `9, 10, 11`）。

#### 保留字段名 {#reserved-field-names}

以后复用旧字段名通常是安全的，除非使用 TextProto 或 JSON 编码时字段名被序列化。为避免此风险，可以将已删除的字段名添加到 `reserved` 列表。

保留字段名只影响 protoc 编译器行为，不影响运行时行为，只有一个例外：TextProto 实现可能会在解析时丢弃具有保留名的未知字段（不像其他未知字段那样报错，目前只有 C++ 和 Go 实现如此）。运行时 JSON 解析不受保留名影响。

```proto
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

注意，不能在同一个 `reserved` 语句中混合字段名和字段编号。

### `.proto` 文件会生成什么？ {#generated}

当你对 `.proto` 文件运行 [protocol buffer 编译器](#generating) 时，编译器会生成你所选语言的代码，用于处理你在文件中描述的消息类型，包括获取和设置字段值、将消息序列化到输出流、从输入流解析消息等。

*   **C++**：编译器为每个 `.proto` 文件生成 `.h` 和 `.cc` 文件，每个消息类型对应一个类。
*   **Java**：编译器生成 `.java` 文件，每个消息类型对应一个类，并有一个专门的 `Builder` 类用于创建消息实例。
*   **Kotlin**：除了生成 Java 代码外，编译器还为每个消息类型生成 `.kt` 文件，提供改进的 Kotlin API，包括简化消息实例创建的 DSL、可空字段访问器和拷贝函数。
*   **Python**：略有不同，Python 编译器为每个 `.proto` 文件生成一个包含每个消息类型静态描述符的模块，运行时通过 *metaclass* 创建所需的数据访问类。
*   **Go**：编译器为每个 `.proto` 文件生成 `.pb.go` 文件，每个消息类型对应一个类型。
*   **Ruby**：编译器生成 `.rb` 文件，包含你的消息类型的 Ruby 模块。
*   **Objective-C**：编译器为每个 `.proto` 文件生成 `pbobjc.h` 和 `pbobjc.m` 文件，每个消息类型对应一个类。
*   **C#**：编译器为每个 `.proto` 文件生成 `.cs` 文件，每个消息类型对应一个类。
*   **PHP**：编译器为每个消息类型生成 `.php` 消息文件，为每个 `.proto` 文件生成 `.php` 元数据文件。元数据文件用于将有效消息类型加载到描述符池。
*   **Dart**：编译器为每个消息类型生成 `.pb.dart` 文件，每个消息类型对应一个类。

你可以通过选择的语言教程了解如何使用各自的 API。更多 API 细节见相关 [API 参考](./reference/)。

## 标量值类型 {#scalar}

标量消息字段可以有以下类型——下表显示了 `.proto` 文件中指定的类型，以及自动生成类中的对应类型：

<div>
  <table>
    <tbody>
      <tr>
        <th>Proto Type</th>
        <th>说明</th>
      </tr>
      <tr>
        <td>double</td>
        <td></td>
      </tr>
      <tr>
        <td>float</td>
        <td></td>
      </tr>
      <tr>
        <td>int32</td>
        <td>使用变长编码。对负数编码效率低——如果字段可能有负值，建议用 sint32。</td>
      </tr>
      <tr>
        <td>int64</td>
        <td>使用变长编码。对负数编码效率低——如果字段可能有负值，建议用 sint64。</td>
      </tr>
      <tr>
        <td>uint32</td>
        <td>使用变长编码。</td>
      </tr>
      <tr>
        <td>uint64</td>
        <td>使用变长编码。</td>
      </tr>
      <tr>
        <td>sint32</td>
        <td>使用变长编码。有符号 int 值。比普通 int32 更高效地编码负数。</td>
      </tr>
      <tr>
        <td>sint64</td>
        <td>使用变长编码。有符号 int 值。比普通 int64 更高效地编码负数。</td>
      </tr>
      <tr>
        <td>fixed32</td>
        <td>始终为四字节。如果值经常大于 2<sup>28</sup>，比 uint32 更高效。</td>
      </tr>
      <tr>
        <td>fixed64</td>
        <td>始终为八字节。如果值经常大于 2<sup>56</sup>，比 uint64 更高效。</td>
      </tr>
      <tr>
        <td>sfixed32</td>
        <td>始终为四字节。</td>
      </tr>
      <tr>
        <td>sfixed64</td>
        <td>始终为八字节。</td>
      </tr>
      <tr>
        <td>bool</td>
        <td></td>
      </tr>
      <tr>
        <td>string</td>
        <td>字符串必须始终包含 UTF-8 编码或 7 位 ASCII 文本，且不能超过 2<sup>32</sup>。</td>
      </tr>
      <tr>
        <td>bytes</td>
        <td>可以包含任意字节序列，长度不超过 2<sup>32</sup>。</td>
      </tr>
    </tbody>
  </table>
</div>

<div>
  <table style="width: 100%;overflow-x: scroll;">
    <tbody>
      <tr>
        <th>Proto Type</th>
        <th>C++ 类型</th>
        <th>Java/Kotlin 类型<sup>[1]</sup></th>
        <th>Python 类型<sup>[3]</sup></th>
        <th>Go 类型</th>
        <th>Ruby 类型</th>
        <th>C# 类型</th>
        <th>PHP 类型</th>
        <th>Dart 类型</th>
        <th>Rust 类型</th>
      </tr>
      <tr>
        <td>double</td>
        <td>double</td>
        <td>double</td>
        <td>float</td>
        <td>float64</td>
        <td>Float</td>
        <td>double</td>
        <td>float</td>
        <td>double</td>
        <td>f64</td>
      </tr>
      <tr>
        <td>float</td>
        <td>float</td>
        <td>float</td>
        <td>float</td>
        <td>float32</td>
        <td>Float</td>
        <td>float</td>
        <td>float</td>
        <td>double</td>
        <td>f32</td>
      </tr>
      <tr>
        <td>int32</td>
        <td>int32_t</td>
        <td>int</td>
        <td>int</td>
        <td>int32</td>
        <td>Fixnum 或 Bignum（按需）</td>
        <td>int</td>
        <td>integer</td>
        <td>int</td>
        <td>i32</td>
      </tr>
      <tr>
        <td>int64</td>
        <td>int64_t</td>
        <td>long</td>
        <td>int/long<sup>[4]</sup></td>
        <td>int64</td>
        <td>Bignum</td>
        <td>long</td>
        <td>integer/string<sup>[6]</sup></td>
        <td>Int64</td>
        <td>i64</td>
      </tr>
      <tr>
        <td>uint32</td>
        <td>uint32_t</td>
        <td>int<sup>[2]</sup></td>
        <td>int/long<sup>[4]</sup></td>
        <td>uint32</td>
        <td>Fixnum 或 Bignum（按需）</td>
        <td>uint</td>
        <td>integer</td>
        <td>int</td>
        <td>u32</td>
      </tr>
      <tr>
        <td>uint64</td>
        <td>uint64_t</td>
        <td>long<sup>[2]</sup></td>
        <td>int/long<sup>[4]</sup></td>
        <td>uint64</td>
        <td>Bignum</td>
        <td>ulong</td>
        <td>integer/string<sup>[6]</sup></td>
        <td>Int64</td>
        <td>u64</td>
      </tr>
      <tr>
        <td>sint32</td>
        <td>int32_t</td>
        <td>int</td>
        <td>int</td>
        <td>int32</td>
        <td>Fixnum 或 Bignum（按需）</td>
        <td>int</td>
        <td>integer</td>
        <td>int</td>
        <td>i32</td>
      </tr>
      <tr>
        <td>sint64</td>
        <td>int64_t</td>
        <td>long</td>
        <td>int/long<sup>[4]</sup></td>
        <td>int64</td>
        <td>Bignum</td>
        <td>long</td>
        <td>integer/string<sup>[6]</sup></td>
        <td>Int64</td>
        <td>i64</td>
      </tr>
      <tr>
        <td>fixed32</td>
        <td>uint32_t</td>
        <td>int<sup>[2]</sup></td>
        <td>int/long<sup>[4]</sup></td>
        <td>uint32</td>
        <td>Fixnum 或 Bignum（按需）</td>
        <td>uint</td>
        <td>integer</td>
        <td>int</td>
        <td>u32</td>
      </tr>
      <tr>
        <td>fixed64</td>
        <td>uint64_t</td>
        <td>long<sup>[2]</sup></td>
        <td>int/long<sup>[4]</sup></td>
        <td>uint64</td>
        <td>Bignum</td>
        <td>ulong</td>
        <td>integer/string<sup>[6]</sup></td>
        <td>Int64</td>
        <td>u64</td>
      </tr>
      <tr>
        <td>sfixed32</td>
        <td>int32_t</td>
        <td>int</td>
        <td>int</td>
        <td>int32</td>
        <td>Fixnum 或 Bignum（按需）</td>
        <td>int</td>
        <td>integer</td>
        <td>int</td>
        <td>i32</td>
      </tr>
      <tr>
        <td>sfixed64</td>
        <td>int64_t</td>
        <td>long</td>
        <td>int/long<sup>[4]</sup></td>
        <td>int64</td>
        <td>Bignum</td>
        <td>long</td>
        <td>integer/string<sup>[6]</sup></td>
        <td>Int64</td>
        <td>i64</td>
      </tr>
      <tr>
        <td>bool</td>
        <td>bool</td>
        <td>boolean</td>
        <td>bool</td>
        <td>bool</td>
        <td>TrueClass/FalseClass</td>
        <td>bool</td>
        <td>boolean</td>
        <td>bool</td>
        <td>bool</td>
      </tr>
      <tr>
        <td>string</td>
        <td>std::string</td>
        <td>String</td>
        <td>str/unicode<sup>[5]</sup></td>
        <td>string</td>
        <td>String (UTF-8)</td>
        <td>string</td>
        <td>string</td>
        <td>String</td>
        <td>ProtoString</td>
      </tr>
      <tr>
        <td>bytes</td>
        <td>std::string</td>
        <td>ByteString</td>
        <td>str (Python 2), bytes (Python 3)</td>
        <td>[]byte</td>
        <td>String (ASCII-8BIT)</td>
        <td>ByteString</td>
        <td>string</td>
        <td>List<int></td>
        <td>ProtoBytes</td>
      </tr>
    </tbody>
  </table>
</div>

<sup>[1]</sup> Kotlin 使用 Java 中对应的类型，即使对于无符号类型也是如此，以确保在 Java/Kotlin 混合代码库中的兼容性。

<sup>[2]</sup> 在 Java 中，无符号 32 位和 64 位整数使用其有符号对应类型表示，最高位直接存储在符号位中。

<sup>[3]</sup> 在所有情况下，给字段赋值时都会进行类型检查以确保其有效性。

<sup>[4]</sup> 64 位或无符号 32 位整数在解码时始终表示为 long，但如果设置字段时给定的是 int，则也可以是 int。无论哪种情况，设置时的值必须适合所表示的类型。参见 [2]。

<sup>[5]</sup> Python 字符串在解码时表示为 unicode，但如果给定 ASCII 字符串则可以是 str（此行为可能会更改）。

<sup>[6]</sup> 在 64 位机器上使用整数，在 32 位机器上使用字符串。

你可以在 [Protocol Buffer Encoding](./programming-guides/encoding) 中了解更多关于这些类型在序列化消息时的编码方式。

## 默认字段值 {#default}

当解析消息时，如果编码的消息字节中不包含某个字段，则在解析后的对象中访问该字段会返回该字段的默认值。默认值是类型相关的：

*   字符串的默认值为空字符串。
*   字节的默认值为空字节。
*   布尔值的默认值为 false。
*   数值类型的默认值为零。
*   消息字段未设置。其具体值依赖于语言。详情请参见 [生成代码指南](./reference/)。
*   枚举的默认值为**第一个定义的枚举值**，该值必须为 0。参见 [枚举默认值](#enum-default)。

重复字段的默认值为空（通常在相应语言中为空列表）。

map 字段的默认值为空（通常在相应语言中为空 map）。

注意，对于隐式存在的标量字段，一旦消息被解析，就无法判断该字段是被显式设置为默认值（例如布尔值被设置为 `false`），还是根本未设置：在定义消息类型时应考虑这一点。例如，如果你不希望某个布尔值被设置为 `false` 时触发某些行为，也不要让该行为在默认情况下发生。还要注意，如果标量消息字段**被**设置为其默认值，则该值不会被序列化到线上。如果 float 或 double 值被设置为 +0，则不会被序列化，但 -0 被视为不同的值，会被序列化。

有关默认值在生成代码中的具体表现，请参见你所选语言的 [生成代码指南](./reference/)。

## 枚举类型 {#enum}

在定义消息类型时，你可能希望某个字段只能取预定义值列表中的一个。例如，你想为每个 `SearchRequest` 添加一个 `corpus` 字段，该字段可以是 `UNIVERSAL`、`WEB`、`IMAGES`、`LOCAL`、`NEWS`、`PRODUCTS` 或 `VIDEO`。你只需在消息定义中添加一个 `enum`，为每个可能的值定义一个常量即可。

在下面的示例中，我们添加了一个名为 `Corpus` 的枚举及其所有可能的值，并定义了一个类型为 `Corpus` 的字段：

```proto
enum Corpus {
    CORPUS_UNSPECIFIED = 0;
    CORPUS_UNIVERSAL = 1;
    CORPUS_WEB = 2;
    CORPUS_IMAGES = 3;
    CORPUS_LOCAL = 4;
    CORPUS_NEWS = 5;
    CORPUS_PRODUCTS = 6;
    CORPUS_VIDEO = 7;
}

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 results_per_page = 3;
    Corpus corpus = 4;
}
```

### 枚举默认值 {#enum-default}

`SearchRequest.corpus` 字段的默认值为 `CORPUS_UNSPECIFIED`，因为这是枚举中定义的第一个值。

在 proto3 中，枚举定义的第一个值**必须**为零，并且建议命名为 `ENUM_TYPE_NAME_UNSPECIFIED` 或 `ENUM_TYPE_NAME_UNKNOWN`。原因如下：

*   必须有一个零值，以便我们可以使用 0 作为数值[默认值](#default)。
*   零值需要作为第一个元素，以兼容 [proto2](./programming-guides/proto2) 语义，其中第一个枚举值为默认值，除非显式指定了其他值。

还建议该第一个默认值仅表示“未指定”，不应有其他语义含义。

### 枚举值别名 {#enum-aliases}

你可以通过为不同的枚举常量分配相同的值来定义别名。为此，你需要将 `allow_alias` 选项设置为 `true`。否则，protocol buffer 编译器在发现别名时会生成警告信息。虽然所有别名值在序列化时都是有效的，但反序列化时只会使用第一个值。

```proto
enum EnumAllowingAlias {
    option allow_alias = true;
    EAA_UNSPECIFIED = 0;
    EAA_STARTED = 1;
    EAA_RUNNING = 1;
    EAA_FINISHED = 2;
}

enum EnumNotAllowingAlias {
    ENAA_UNSPECIFIED = 0;
    ENAA_STARTED = 1;
    // ENAA_RUNNING = 1;  // 取消注释此行会导致警告信息。
    ENAA_FINISHED = 2;
}
```

枚举常量必须在 32 位整数范围内。由于 `enum` 值在线上使用 [varint 编码](./programming-guides/encoding)，负值效率较低，因此不推荐使用。你可以在消息定义中定义 `enum`，如前例所示，也可以在外部定义——这些 `enum` 可以在同一个 `.proto` 文件的任意消息定义中复用。你还可以在一个消息中使用另一个消息声明的 `enum` 类型，语法为 `_MessageType_._EnumType_`。

当你在 `.proto` 文件中使用 `enum` 并运行 protocol buffer 编译器时，生成的代码会为 Java、Kotlin 或 C++ 生成相应的 `enum`，或为 Python 生成特殊的 `EnumDescriptor` 类，用于在运行时生成带有整数值的符号常量集。

{{% alert title="重要" color="warning" %}} 生成的代码可能会受到语言对枚举成员数量的限制（某些语言为几千个）。请查阅你计划使用的语言的相关限制。
{{% /alert %}}

反序列化时，未识别的枚举值会保存在消息中，但在消息反序列化后如何表示取决于语言。在支持开放枚举类型（允许超出指定符号范围的值）的语言（如 C++ 和 Go）中，未知枚举值会以其底层整数表示存储。在 Java 等封闭枚举类型的语言中，会用枚举中的一个 case 表示未识别的值，并可通过特殊访问器获取底层整数。无论哪种情况，如果消息被序列化，未识别的值仍会随消息一起序列化。

{{% alert title="重要" color="warning" %}} 关于枚举的预期行为与实际在不同语言中的表现差异，参见 [Enum Behavior](./programming-guides/enum)。
{{% /alert %}}

有关如何在应用程序中使用消息枚举的更多信息，请参见你所选语言的 [生成代码指南](./reference/)。

### 保留值 {#reserved}

如果你通过完全移除或注释掉枚举项来[更新](#updating)枚举类型，未来的用户在更新类型时可能会复用这些数值。这可能导致严重问题，比如加载旧 `.proto` 实例时出现数据损坏、隐私漏洞等。为避免这种情况，可以将已删除条目的数值（和/或名称，名称在 JSON 序列化时也可能引发问题）声明为 `reserved`。如果未来用户尝试使用这些标识符，protocol buffer 编译器会报错。你可以使用 `max` 关键字指定保留数值范围的最大值。

```proto
enum Foo {
    reserved 2, 15, 9 to 11, 40 to max;
    reserved "FOO", "BAR";
}
```

注意，不能在同一个 `reserved` 语句中混合字段名和数值。

## 使用其他消息类型 {#other}

你可以将其他消息类型作为字段类型。例如，如果你想在每个 `SearchResponse` 消息中包含 `Result` 消息，可以在同一个 `.proto` 文件中定义 `Result` 消息类型，然后在 `SearchResponse` 中指定类型为 `Result` 的字段：

```proto
message SearchResponse {
    repeated Result results = 1;
}

message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
}
```

### 导入定义 {#importing}

在前面的例子中，`Result` 消息类型与 `SearchResponse` 定义在同一个文件中——如果你要用作字段类型的消息类型已经在另一个 `.proto` 文件中定义了怎么办？

你可以通过 *import* 其他 `.proto` 文件来使用其中的定义。只需在文件顶部添加 import 语句：

```proto
import "myproject/other_protos.proto";
```

默认情况下，你只能使用直接导入的 `.proto` 文件中的定义。但有时你可能需要将 `.proto` 文件移动到新位置。与其直接移动文件并一次性更新所有调用点，不如在旧位置放一个占位 `.proto` 文件，通过 `import public` 转发所有导入到新位置。

**注意：** Java 中的 public import 功能在移动整个 .proto 文件或使用 `java_multiple_files = true` 时最有效。在这些情况下，生成的名称保持稳定，无需更新代码中的引用。如果在未启用 `java_multiple_files = true` 的情况下仅移动部分 .proto 文件，虽然技术上可行，但需要同时更新许多引用，迁移难度并未明显降低。Kotlin、TypeScript、JavaScript、GCL 以及使用 protobuf 静态反射的 C++ 目标不支持此功能。

`import public` 依赖项可以被任何导入包含 `import public` 语句的 proto 的代码传递性地依赖。例如：

```proto
// new.proto
// 所有定义都移到这里
```

```proto
// old.proto
// 所有客户端都导入此 proto。
import public "new.proto";
import "other.proto";
```

```proto
// client.proto
import "old.proto";
// 你可以使用 old.proto 和 new.proto 中的定义，但不能使用 other.proto
```

protocol buffer 编译器会在通过 `-I`/`--proto_path` 标志指定的一组目录中查找被导入的文件。如果未指定该标志，则在编译器运行目录下查找。一般建议将 `--proto_path` 设置为项目根目录，并为所有导入使用全限定名。

### 使用 proto2 消息类型 {#proto2}

你可以导入 [proto2](./programming-guides/proto2) 消息类型并在 proto3 消息中使用，反之亦然。但 proto2 枚举不能直接在 proto3 语法中使用（如果导入的 proto2 消息使用了它们则没问题）。

## 嵌套类型 {#nested}

你可以在一个消息类型内部定义和使用其他消息类型，如下例所示——这里的 `Result` 消息被定义在 `SearchResponse` 消息内部：

```proto
message SearchResponse {
    message Result {
        string url = 1;
        string title = 2;
        repeated string snippets = 3;
    }
    repeated Result results = 1;
}
```

如果你想在父消息类型之外复用这个消息类型，可以通过 `_Parent_._Type_` 的方式引用它：

```proto
message SomeOtherMessage {
    SearchResponse.Result result = 1;
}
```

你可以任意深度地嵌套消息。在下面的例子中，注意两个名为 `Inner` 的嵌套类型是完全独立的，因为它们定义在不同的消息中：

```proto
message Outer {       // Level 0
    message MiddleAA {  // Level 1
        message Inner {   // Level 2
            int64 ival = 1;
            bool  booly = 2;
        }
    }
    message MiddleBB {  // Level 1
        message Inner {   // Level 2
            int32 ival = 1;
            bool  booly = 2;
        }
    }
}
```

## 更新消息类型 {#updating}

如果现有的消息类型已经不能满足你的需求——比如你想为消息格式增加一个字段——但又希望继续使用旧格式生成的代码，不用担心！当你使用二进制 wire 格式时，更新消息类型不会破坏任何现有代码，非常简单。

{{% alert title="注意" color="note" %}} 如果你使用 JSON 或
[proto text format](./reference/protobuf/textformat-spec)
来存储 protocol buffer 消息，你能在 proto 定义中做的更改会有所不同。{{% /alert %}}

请查看
[Proto 最佳实践](./best-practices/dos-donts) 以及以下规则：

*   不要更改任何已有字段的字段编号。更改字段编号等同于删除该字段并添加一个新字段（类型相同但编号不同）。如果你想重新编号字段，请参见
        [删除字段](#deleting) 的说明。
*   如果你添加新字段，使用“旧”消息格式序列化的消息仍然可以被新生成的代码解析。你需要注意这些元素的[默认值](#default)，以便新代码能正确与旧代码生成的消息交互。同样，由新代码创建的消息也可以被旧代码解析：旧二进制只会在解析时忽略新字段。详情见
        [未知字段](#unknowns) 部分。
*   字段可以被移除，只要该字段编号在更新后的消息类型中不再被使用。你可能想重命名该字段，比如加上前缀“OBSOLETE_”，或者将该字段编号
        [保留](#fieldreserved)，以防将来用户在 `.proto` 文件中意外复用该编号。
*   `int32`、`uint32`、`int64`、`uint64` 和 `bool` 彼此兼容——这意味着你可以在这些类型之间更改字段类型，而不会破坏前向或后向兼容性。如果从 wire 解析出一个不适合对应类型的数字，会得到与在 C++ 中强制类型转换相同的效果（例如，如果将 64 位数字作为 int32 读取，会被截断为 32 位）。
*   `sint32` 和 `sint64` 彼此兼容，但与其他整数类型*不兼容*。如果写入的值在 INT_MIN 到 INT_MAX 之间，用任一类型解析都能得到相同的值。如果写入的 sint64 超出该范围并作为 sint32 解析，varint 会被截断为 32 位，然后进行 zigzag 解码（这会导致观察到不同的值）。
*   只要字节是有效的 UTF-8，`string` 和 `bytes` 兼容。
*   如果字节包含编码后的消息实例，嵌入式消息与 `bytes` 兼容。
*   `fixed32` 与 `sfixed32` 兼容，`fixed64` 与 `sfixed64` 兼容。
*   对于 `string`、`bytes` 和消息字段，单个（singular）与 `repeated` 兼容。对于 repeated 字段的序列化数据，期望该字段为单个的客户端会取最后一个输入值（如果是基本类型字段），或合并所有输入元素（如果是消息类型字段）。注意，这对数值类型（包括 bool 和 enum）通常**不安全**。数值类型的 repeated 字段默认以
        [packed](./programming-guides/encoding#packed) 格式序列化，
        这在期望单个字段时无法正确解析。
*   就 wire 格式而言，`enum` 与 `int32`、`uint32`、`int64` 和 `uint64` 兼容（注意如果值不适合会被截断）。但要注意，客户端代码在反序列化消息时可能会有不同处理方式：例如，未识别的 proto3 `enum` 值会保留在消息中，但具体如何表示取决于语言。int 字段总是保留其值。
*   将单个 `optional` 字段或扩展变为**新**的 `oneof` 成员是二进制兼容的，但对于某些语言（如 Go），生成代码的 API 会发生不兼容的变化。因此，Google 在其公共 API 中不会做此类更改，详见
        [AIP-180](https://google.aip.dev/180#moving-into-oneofs)。同样地，如果你能确保没有代码同时设置多个字段，将多个字段移入新的 `oneof` 可能是安全的。将字段移入已有的 `oneof` 则不安全。同理，将单字段 `oneof` 改为 `optional` 字段或扩展是安全的。
*   在 `map<K, V>` 与对应的 `repeated` 消息字段之间切换是二进制兼容的（见下文 [Maps](#maps) 了解消息布局和其他限制）。但更改的安全性取决于应用：在反序列化和重新序列化消息时，使用 `repeated` 字段定义的客户端会生成语义等价的结果；但使用 `map` 字段定义的客户端可能会重新排序条目并丢弃重复键的条目。

## 未知字段 {#unknowns}

未知字段是指 protocol buffer 序列化数据中，解析器无法识别的字段。例如，当旧的二进制解析由新二进制发送的数据（包含新字段）时，这些新字段在旧二进制中就成为未知字段。

Proto3 消息会保留未知字段，并在解析和序列化输出时包含它们，这与 proto2 的行为一致。

### 保留未知字段 {#retaining}

某些操作可能导致未知字段丢失。例如，执行以下操作时，未知字段会丢失：

*   将 proto 序列化为 JSON。
*   遍历消息中的所有字段以填充新消息。

为避免丢失未知字段，请遵循以下建议：

*   使用二进制格式，避免在数据交换中使用文本格式。
*   使用面向消息的 API，如 `CopyFrom()` 和 `MergeFrom()`，进行数据复制，而不是逐字段复制。

TextFormat 是一个特殊情况。序列化为 TextFormat 时，会用字段编号打印未知字段。但如果 TextFormat 数据中有使用字段编号的条目，解析回二进制 proto 会失败。

## Any {#any}

`Any` 消息类型允许你将消息作为嵌入类型使用，而无需其 .proto 定义。`Any` 包含一个任意序列化消息的 `bytes`，以及一个 URL 作为该消息类型的全局唯一标识符，并可解析到该类型。要使用 `Any` 类型，你需要
[import](#other) `google/protobuf/any.proto`。

```proto
import "google/protobuf/any.proto";

message ErrorStatus {
    string message = 1;
    repeated google.protobuf.Any details = 2;
}
```

给定消息类型的默认 type URL 为
`type.googleapis.com/_packagename_._messagename_`。

不同语言实现会提供运行时库辅助方法，以类型安全的方式打包和解包 `Any` 值——例如，在 Java 中，`Any` 类型有专门的 `pack()` 和 `unpack()` 方法，在 C++ 中有
`PackFrom()` 和 `UnpackTo()` 方法：

```cpp
// 将任意消息类型存储到 Any 中。
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// 从 Any 读取任意消息。
ErrorStatus status = ...;
for (const google::protobuf::Any& detail : status.details()) {
    if (detail.Is<NetworkErrorDetails>()) {
        NetworkErrorDetails network_error;
        detail.UnpackTo(&network_error);
        ... 处理 network_error ...
    }
}
```

## Oneof {#oneof}

如果你的消息有许多单个字段，并且同一时间最多只会设置一个字段，可以使用 oneof 特性来强制这种行为并节省内存。

Oneof 字段类似于可选字段，但同一个 oneof 中的所有字段共享内存，并且同一时间最多只能设置一个字段。设置 oneof 的任意成员会自动清除其他成员。你可以使用特殊的 `case()` 或 `WhichOneof()` 方法（取决于所用语言）检查 oneof 中设置的是哪个值（如果有的话）。

注意，如果*设置了多个值，则以 proto 中顺序最后设置的值覆盖之前所有值*。

oneof 字段的字段编号在所属消息内必须唯一。

### 使用 Oneof {#using-oneof}

在 `.proto` 文件中定义 oneof 时，使用 `oneof` 关键字和 oneof 名称，如 `test_oneof`：

```proto
message SampleMessage {
    oneof test_oneof {
        string name = 4;
        SubMessage sub_message = 9;
    }
}
```

然后将 oneof 字段添加到 oneof 定义中。你可以添加任意类型的字段，除了 `map` 字段和 `repeated` 字段。如果需要将 repeated 字段添加到 oneof，可以使用包含 repeated 字段的消息类型。

在生成的代码中，oneof 字段有与普通字段相同的 getter 和 setter。你还会获得一个特殊方法，用于检查 oneof 中设置的是哪个值（如果有的话）。你可以在相关的 [API 参考](./reference/) 中了解更多 oneof API 的信息。

### Oneof 特性 {#oneof-features}

*   设置 oneof 字段会自动清除该 oneof 的其他成员。因此，如果你设置了多个 oneof 字段，只有*最后*设置的字段会有值。

        ```cpp
        SampleMessage message;
        message.set_name("name");
        CHECK_EQ(message.name(), "name");
        // 调用 mutable_sub_message() 会清除 name 字段，并将 sub_message 设置为新的 SubMessage 实例，其字段均未设置。
        message.mutable_sub_message();
        CHECK(message.name().empty());
        ```

*   如果解析器在 wire 上遇到同一 oneof 的多个成员，只有最后遇到的成员会在解析后的消息中被使用。解析 wire 数据时，从字节开头开始，依次读取下一个值，并应用以下解析规则：

        *   首先，检查同一 oneof 中是否已设置*不同*字段，如果是，则清除它。
        *   然后像字段不在 oneof 中一样应用内容：
                *   基本类型会覆盖已设置的值
                *   消息类型会合并到已设置的值

*   oneof 不能为 `repeated`。

*   反射 API 支持 oneof 字段。

*   如果你将 oneof 字段设置为默认值（如将 int32 oneof 字段设为 0），该 oneof 字段的“case”会被设置，并且该值会被序列化到 wire 上。

*   如果你使用 C++，请确保代码不会导致内存崩溃。如下示例代码会崩溃，因为调用 `set_name()` 方法后，`sub_message` 已被删除。

        ```cpp
        SampleMessage message;
        SubMessage* sub_message = message.mutable_sub_message();
        message.set_name("name");      // 会删除 sub_message
        sub_message->set_...            // 此处崩溃
        ```

*   同样在 C++ 中，如果你对带有 oneof 的两个消息调用 `Swap()`，每个消息会交换 oneof 的 case：如下例中，`msg1` 会有 `sub_message`，`msg2` 会有 `name`。

        ```cpp
        SampleMessage msg1;
        msg1.set_name("name");
        SampleMessage msg2;
        msg2.mutable_sub_message();
        msg1.swap(&msg2);
        CHECK(msg1.has_sub_message());
        CHECK_EQ(msg2.name(), "name");
        ```

### 向后兼容性问题 {#backward}

在添加或移除 oneof 字段时要小心。如果检查 oneof 的值返回 `None`/`NOT_SET`，这可能意味着 oneof 尚未被设置，或者它已被设置为 oneof 的不同版本中的字段。无法区分这两种情况，因为无法知道线上未知字段是否属于该 oneof。

#### 标签重用问题 {#reuse}

*   **将单个字段移入或移出 oneof**：在消息序列化和解析后，可能会丢失部分信息（某些字段会被清除）。不过，你可以安全地将单个字段移入一个**新的** oneof，如果已知只会设置其中一个字段，也可以移动多个字段。详情见[更新消息类型](#updating)。
*   **删除 oneof 字段后再添加回来**：这可能会在消息序列化和解析后清除当前已设置的 oneof 字段。
*   **拆分或合并 oneof**：这与移动单个字段有类似的问题。

## Maps {#maps}

如果你想在数据定义中创建关联映射，protocol buffer 提供了便捷的语法：

```proto
map<key_type, value_type> map_field = N;
```

其中，`key_type` 可以是任意整数类型或字符串类型（即除浮点类型和 `bytes` 外的任意[标量](#scalar)类型）。注意，枚举和 proto 消息都不能作为 `key_type`。`value_type` 可以是除 map 以外的任意类型。

例如，如果你想创建一个项目映射，每个字符串 key 对应一个 `Project` 消息，可以这样定义：

```proto
map<string, Project> projects = 3;
```

### Maps 特性 {#maps-features}

*   Map 字段不能为 `repeated`。
*   Map 值的线格式顺序和遍历顺序未定义，因此不能依赖 map 项的顺序。
*   生成 `.proto` 的文本格式时，map 会按 key 排序。数值 key 按数值排序。
*   从 wire 解析或合并时，如果有重复的 map key，则使用最后出现的 key。文本格式解析 map 时，如果有重复 key，解析可能失败。
*   如果为 map 字段提供了 key 但没有 value，序列化时的行为依赖于语言。在 C++、Java、Kotlin 和 Python 中，会序列化类型的默认值；在其他语言中则不会序列化任何内容。
*   在 map `foo` 的同一作用域下不能有名为 `FooEntry` 的符号，因为 `FooEntry` 已被 map 的实现占用。

所有支持的语言都已提供生成的 map API。你可以在相关[API 参考](./reference/)中了解更多信息。

### 向后兼容性 {#backwards}

map 语法在 wire 上等价于如下定义，因此即使 protocol buffer 实现不支持 map，也能处理你的数据：

```proto
message MapFieldEntry {
    key_type key = 1;
    value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

任何支持 map 的 protocol buffer 实现都必须能生成并接受上述早期定义的数据。

## Packages {#packages}

你可以在 `.proto` 文件中添加可选的 `package` 说明符，以防止 protocol buffer 消息类型之间的命名冲突。

```proto
package foo.bar;
message Open { ... }
```

然后可以在消息类型字段定义时使用 package 说明符：

```proto
message Foo {
    ...
    foo.bar.Open open = 1;
    ...
}
```

package 说明符对生成代码的影响取决于所选语言：

*   **C++**：生成的类会被包裹在 C++ 命名空间中。例如，`Open` 会在 `foo::bar` 命名空间下。
*   **Java** 和 **Kotlin**：package 会作为 Java 包使用，除非在 `.proto` 文件中显式提供了 `option java_package`。
*   **Python**：`package` 指令被忽略，因为 Python 模块根据文件系统位置组织。
*   **Go**：`package` 指令被忽略，生成的 `.pb.go` 文件的包名由对应的 `go_proto_library` Bazel 规则决定。开源项目**必须**提供 `go_package` 选项或设置 Bazel 的 `-M` 标志。
*   **Ruby**：生成的类会被包裹在嵌套的 Ruby 命名空间中，并转换为 Ruby 的首字母大写风格（如果首字符不是字母，则加前缀 `PB_`）。例如，`Open` 会在 `Foo::Bar` 命名空间下。
*   **PHP**：package 会在转换为 PascalCase 后作为命名空间，除非在 `.proto` 文件中显式提供了 `option php_namespace`。例如，`Open` 会在 `Foo\Bar` 命名空间下。
*   **C#**：package 会在转换为 PascalCase 后作为命名空间，除非在 `.proto` 文件中显式提供了 `option csharp_namespace`。例如，`Open` 会在 `Foo.Bar` 命名空间下。

注意，即使 `package` 指令不会直接影响生成代码（如 Python），仍强烈建议为 `.proto` 文件指定 package，否则可能导致描述符命名冲突，并影响 proto 在其他语言中的可移植性。

### Packages 与名称解析 {#name-resolution}

protocol buffer 语言中的类型名称解析方式类似 C++：首先搜索最内层作用域，然后依次向外层搜索，每个 package 都被视为其父 package 的“内部”。以 `.` 开头（如 `.foo.bar.Baz`）表示从最外层作用域开始。

protocol buffer 编译器通过解析导入的 `.proto` 文件来解析所有类型名称。每种语言的代码生成器都知道如何在该语言中引用每个类型，即使作用域规则不同。

## 定义服务 {#services}

如果你希望将消息类型用于 RPC（远程过程调用）系统，可以在 `.proto` 文件中定义 RPC 服务接口，protocol buffer 编译器会为你选择的语言生成服务接口代码和存根。例如，若要定义一个 RPC 服务，包含一个以 `SearchRequest` 作为输入、返回 `SearchResponse` 的方法，可以这样定义：

```proto
service SearchService {
    rpc Search(SearchRequest) returns (SearchResponse);
}
```

最直接的 protocol buffer RPC 系统是 [gRPC](https://grpc.io)：由 Google 开发的跨语言、跨平台开源 RPC 系统。gRPC 与 protocol buffer 配合良好，可以直接通过 protocol buffer 编译器插件从 `.proto` 文件生成相关 RPC 代码。

如果不想用 gRPC，也可以将 protocol buffer 与自定义 RPC 实现结合使用。详情见[Proto2 语言指南](./programming-guides/proto2#services)。

还有许多第三方项目正在为 protocol buffer 开发 RPC 实现。已知项目列表见[第三方插件 wiki 页面](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)。

## JSON 映射 {#json}

标准的 protocol buffer 二进制 wire 格式是 protobuf 系统间通信的首选序列化格式。若需与使用 JSON 的系统通信，protocol buffer 支持[JSON](./programming-guides/json) 的规范编码。

## Options {#options}

`.proto` 文件中的各个声明可以用多种*选项*进行注解。选项不会改变声明的整体含义，但可能影响其在特定上下文中的处理方式。完整选项列表见 [`/google/protobuf/descriptor.proto`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/descriptor.proto)。

部分选项为文件级选项，应写在顶层作用域，不能写在 message、enum 或 service 定义内部。部分选项为消息级选项，应写在消息定义内部。部分选项为字段级选项，应写在字段定义内部。选项还可用于枚举类型、枚举值、oneof 字段、服务类型和服务方法，但目前这些没有有用的选项。

以下是常用选项：

*   `java_package`（文件选项）：用于生成 Java/Kotlin 类的包名。如果 `.proto` 文件未显式指定 `java_package`，则默认使用 proto package（即 `.proto` 文件中的 "package" 关键字指定的包）。但 proto package 通常不适合作为 Java 包，因为它们不一定以反向域名开头。如果不生成 Java 或 Kotlin 代码，此选项无效。

        ```proto
        option java_package = "com.example.foo";
        ```

*   `java_outer_classname`（文件选项）：用于生成的 Java 外部包装类的类名（即文件名）。如果 `.proto` 文件未显式指定 `java_outer_classname`，则类名会将 `.proto` 文件名转换为驼峰命名（如 `foo_bar.proto` 变为 `FooBar.java`）。如果禁用 `java_multiple_files` 选项，则为该 `.proto` 文件生成的所有类/枚举等都会作为嵌套类/枚举等包含在该外部包装类中。如果不生成 Java 代码，此选项无效。

        ```proto
        option java_outer_classname = "Ponycopter";
        ```

*   `java_multiple_files`（文件选项）：若为 false，则该 `.proto` 文件只生成一个 `.java` 文件，所有为顶层消息、服务和枚举生成的 Java 类/枚举等都会嵌套在外部类（见 `java_outer_classname`）中。若为 true，则为每个顶层消息、服务和枚举分别生成 `.java` 文件，且该 `.proto` 文件生成的外部包装类不会包含任何嵌套类/枚举等。该布尔选项默认为 `false`。如果不生成 Java 代码，此选项无效。

        ```proto
        option java_multiple_files = true;
        ```

*   `optimize_for`（文件选项）：可设置为 `SPEED`、`CODE_SIZE` 或 `LITE_RUNTIME`。这会影响 C++ 和 Java 代码生成器（以及可能的第三方生成器）：

        *   `SPEED`（默认）：protocol buffer 编译器会为消息类型生成高度优化的序列化、解析及其他常用操作代码。
        *   `CODE_SIZE`：编译器会生成最小化的类，并依赖共享的反射代码实现序列化、解析等操作。生成代码比 `SPEED` 模式小，但操作速度较慢。类的公共 API 与 `SPEED` 模式完全一致。适用于包含大量 `.proto` 文件且对速度要求不高的应用。
        *   `LITE_RUNTIME`：编译器会生成仅依赖 "lite" 运行库（`libprotobuf-lite` 而非 `libprotobuf`）的类。lite 运行库比完整库小一个数量级，但省略了描述符和反射等功能。适用于如手机等受限平台。编译器仍会生成高效实现，类只实现 `MessageLite` 接口（而非完整的 `Message` 接口）。

        ```proto
        option optimize_for = CODE_SIZE;
        ```

*   `cc_generic_services`、`java_generic_services`、`py_generic_services`（文件选项）：**通用服务已弃用。** 控制 protocol buffer 编译器是否为 C++、Java 和 Python 生成基于[服务定义](#services)的抽象服务代码。出于兼容性考虑，默认值为 `true`。但自 2.3.0 版（2010 年 1 月）起，建议 RPC 实现通过[代码生成插件](./reference/cpp/api-docs/google.protobuf.compiler.plugin.pb)生成更具体的代码，而不是依赖“抽象”服务。

        ```proto
        // 本文件依赖插件生成服务代码。
        option cc_generic_services = false;
        option java_generic_services = false;
        option py_generic_services = false;
        ```

*   `cc_enable_arenas`（文件选项）：为 C++ 生成代码启用[arena 分配](./reference/cpp/arenas)。

*   `objc_class_prefix`（文件选项）：设置 Objective-C 生成类和枚举的前缀。无默认值。建议使用 3-5 个大写字母的前缀，[Apple 推荐](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html#//apple_ref/doc/uid/TP40011210-CH10-SW4)。所有两字母前缀已被 Apple 保留。

*   `packed`（字段选项）：在基本数值类型的 repeated 字段上默认为 `true`，使用更紧凑的[编码](./programming-guides/encoding#packed)。如需使用非打包 wire 格式，可设为 `false`。这为 2.3.0 之前的解析器提供兼容性（很少需要），例如：

        ```proto
        repeated int32 samples = 4 [packed = false];
        ```

*   `deprecated`（字段选项）：若设为 `true`，表示该字段已弃用，不应在新代码中使用。在大多数语言中无实际影响。在 Java 中会生成 `@Deprecated` 注解。C++ 中，clang-tidy 在使用弃用字段时会发出警告。未来，其他语言的代码生成器也可能为字段访问器生成弃用注解，从而在编译使用该字段的代码时发出警告。如果该字段无人使用且希望阻止新用户使用，可用 [reserved](#fieldreserved) 语句替换字段声明。

        ```proto
        int32 old_field = 6 [deprecated = true];
        ```

### 枚举值选项 {#enum-value-options}

支持枚举值选项。你可以使用 `deprecated` 选项来表示某个值不应再被使用。你也可以通过扩展来创建自定义选项。

下面的示例展示了如何添加这些选项的语法：

```proto
import "google/protobuf/descriptor.proto";

extend google.protobuf.EnumValueOptions {
    optional string string_name = 123456789;
}

enum Data {
    DATA_UNSPECIFIED = 0;
    DATA_SEARCH = 1 [deprecated = true];
    DATA_DISPLAY = 2 [
        (string_name) = "display_value"
    ];
}
```

读取 `string_name` 选项的 C++ 代码可能如下所示：

```cpp
const absl::string_view foo = proto2::GetEnumDescriptor<Data>()
        ->FindValueByName("DATA_DISPLAY")->options().GetExtension(string_name);
```

参见 [自定义选项](#customoptions) 了解如何将自定义选项应用于枚举值和字段。

### 自定义选项 {#customoptions}

Protocol Buffers 还允许你定义和使用自己的选项。注意，这属于**高级特性**，大多数用户并不需要。如果你确实需要创建自定义选项，请参阅
[Proto2 语言指南](./programming-guides/proto2#customoptions)
获取详细信息。注意，创建自定义选项需要用到
[扩展](./programming-guides/proto2#extensions)，
而在 proto3 中，扩展仅允许用于自定义选项。

### 选项保留 {#option-retention}

选项有一个 *保留*（retention）概念，用于控制选项是否会保留在生成的代码中。选项默认具有 *运行时保留*，意味着它们会保留在生成的代码中，因此在运行时的描述符池中可见。不过，你可以设置 `retention = RETENTION_SOURCE`，指定某个选项（或选项中的字段）不应在运行时保留。这称为 *源保留*。

选项保留是一个高级特性，大多数用户无需关注，但如果你希望使用某些选项而不希望它们增加二进制文件的代码体积时，这会很有用。具有源保留的选项在 `protoc` 和 `protoc` 插件中依然可见，因此代码生成器可以利用它们自定义行为。

可以直接在选项上设置保留方式，如下所示：

```proto
extend google.protobuf.FileOptions {
    optional int32 source_retention_option = 1234
            [retention = RETENTION_SOURCE];
}
```

也可以在普通字段上设置保留方式，此时仅当该字段出现在选项中时才生效：

```proto
message OptionsMessage {
    int32 source_retention_field = 1 [retention = RETENTION_SOURCE];
}
```

你也可以设置 `retention = RETENTION_RUNTIME`，但这没有实际效果，因为这是默认行为。当消息字段被标记为 `RETENTION_SOURCE` 时，其全部内容都会被丢弃；其中的字段无法通过设置 `RETENTION_RUNTIME` 来覆盖。

{{% alert title="注意" color="note" %}} 从 Protocol Buffers 22.0 起，选项保留的支持仍在完善中，目前仅支持 C++ 和 Java。Go 从 1.29.0 开始支持。Python 支持已完成，但尚未发布新版本。
{{% /alert %}}

### 选项目标 {#option-targets}

字段有一个 `targets` 选项，用于控制该字段作为选项时可以应用于哪些实体类型。例如，如果某字段设置了 `targets = TARGET_TYPE_MESSAGE`，则该字段不能作为自定义选项应用于枚举（或其他非消息实体）。protoc 会强制执行此约束，如果违反目标限制会报错。

乍看之下，这个特性似乎没必要，因为每个自定义选项都是为特定实体的 options 消息扩展的，已经限定了选项的实体类型。但当你有一个共享的 options 消息应用于多种实体类型，并希望控制该消息中各字段的使用时，选项目标就很有用。例如：

```proto
message MyOptions {
    string file_only_option = 1 [targets = TARGET_TYPE_FILE];
    int32 message_and_enum_option = 2 [targets = TARGET_TYPE_MESSAGE,
                                                                         targets = TARGET_TYPE_ENUM];
}

extend google.protobuf.FileOptions {
    optional MyOptions file_options = 50000;
}

extend google.protobuf.MessageOptions {
    optional MyOptions message_options = 50000;
}

extend google.protobuf.EnumOptions {
    optional MyOptions enum_options = 50000;
}

// 正确：该字段允许用于文件选项
option (file_options).file_only_option = "abc";

message MyMessage {
    // 正确：该字段允许用于消息和枚举选项
    option (message_options).message_and_enum_option = 42;
}

enum MyEnum {
    MY_ENUM_UNSPECIFIED = 0;
    // 错误：file_only_option 不能用于枚举选项
    option (enum_options).file_only_option = "xyz";
}
```

## 生成你的类 {#generating}

要为 `.proto` 文件中定义的消息类型生成 Java、Kotlin、Python、C++、Go、Ruby、Objective-C 或 C# 代码，需要在 `.proto` 文件上运行 protocol buffer 编译器 `protoc`。如果你还没有安装编译器，
[下载软件包](./downloads) 并按照 README 中的说明操作。对于 Go，还需要为编译器安装专用的代码生成插件；你可以在 GitHub 的 [golang/protobuf](https://github.com/golang/protobuf/) 仓库找到插件和安装说明。

Protocol Compiler 的调用方式如下：

```sh
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```

*   `IMPORT_PATH` 指定查找 `.proto` 文件时用于解析 `import` 指令的目录。如果省略，则使用当前目录。可以多次传递 `--proto_path` 选项以指定多个导入目录。`-I=_IMPORT_PATH_` 是 `--proto_path` 的简写。

**注意：** 相对于 `proto_path` 的文件路径在同一个二进制文件中必须全局唯一。例如，如果有 `proto/lib1/data.proto` 和 `proto/lib2/data.proto`，则不能同时使用 `-I=proto/lib1 -I=proto/lib2`，因为 `import "data.proto"` 会产生歧义。应使用 `-Iproto/`，这样全局名称分别为 `lib1/data.proto` 和 `lib2/data.proto`。

如果你要发布一个库，且其他用户可能直接使用你的消息，建议在路径中包含唯一的库名，以避免文件名冲突。如果一个项目有多个目录，最佳实践是将 `-I` 设置为项目的顶级目录。

*   你可以提供一个或多个*输出指令*：

        *   `--cpp_out` 在 `DST_DIR` 生成 C++ 代码。详见
                [C++ 生成代码参考](./reference/cpp/cpp-generated)。
        *   `--java_out` 在 `DST_DIR` 生成 Java 代码。详见
                [Java 生成代码参考](./reference/java/java-generated)。
        *   `--kotlin_out` 在 `DST_DIR` 生成额外的 Kotlin 代码。详见
                [Kotlin 生成代码参考](./reference/kotlin/kotlin-generated)。
        *   `--python_out` 在 `DST_DIR` 生成 Python 代码。详见
                [Python 生成代码参考](./reference/python/python-generated)。
        *   `--go_out` 在 `DST_DIR` 生成 Go 代码。详见
                [Go 生成代码参考](./reference/go/go-generated-opaque)。
        *   `--ruby_out` 在 `DST_DIR` 生成 Ruby 代码。详见
                [Ruby 生成代码参考](./reference/ruby/ruby-generated)。
        *   `--objc_out` 在 `DST_DIR` 生成 Objective-C 代码。详见
                [Objective-C 生成代码参考](./reference/objective-c/objective-c-generated)。
        *   `--csharp_out` 在 `DST_DIR` 生成 C# 代码。详见
                [C# 生成代码参考](./reference/csharp/csharp-generated)。
        *   `--php_out` 在 `DST_DIR` 生成 PHP 代码。详见
                [PHP 生成代码参考](./reference/php/php-generated)。

        作为额外的便利，如果 `DST_DIR` 以 `.zip` 或 `.jar` 结尾，编译器会将输出写入指定名称的单个 ZIP 格式归档文件。`.jar` 输出还会包含 Java JAR 规范要求的清单文件。注意，如果输出归档已存在，将会被覆盖。

*   必须提供一个或多个 `.proto` 文件作为输入。可以一次指定多个 `.proto` 文件。虽然文件名是相对于当前目录的，但每个文件必须位于某个 `IMPORT_PATH` 下，以便编译器确定其规范名称。

## 文件位置 {#location}

建议不要将 `.proto` 文件与其他语言源文件放在同一目录下。可以在项目根包下创建一个 `proto` 子包用于存放 `.proto` 文件。

### 位置应与语言无关 {#location-language-agnostic}

在使用 Java 代码时，将相关的 `.proto` 文件与 Java 源码放在同一目录下很方便。但如果有非 Java 代码也要使用这些 proto，路径前缀就不再合适。因此，通常应将 proto 文件放在与语言无关的相关目录下，如 `//myteam/mypackage`。

唯一的例外是明确只会在 Java 场景下使用 proto，比如用于测试时。

## 支持的平台 {#platforms}

关于以下内容的信息：

*   有关支持的操作系统、编译器、构建系统和 C++ 版本，请参阅
    [Foundational C++ Support Policy](https://opensource.google/documentation/policies/cplusplus-support)。
*   有关支持的 PHP 版本，请参阅
    [Supported PHP versions](https://cloud.google.com/php/getting-started/supported-php-versions)。
