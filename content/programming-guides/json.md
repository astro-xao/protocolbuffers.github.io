+++
title = "ProtoJSON 格式"
weight = 62
description = "介绍如何使用 Protobuf 到 JSON 的转换工具。"
type = "docs"
+++

Protobuf 支持一种规范的 JSON 编码格式，使得与不支持标准 protobuf 二进制格式的系统之间的数据共享更加容易。

ProtoJSON 格式的效率不如 protobuf 二进制格式。转换器在编码和解码消息时会消耗更多的 CPU，并且（除极少数情况外）编码后的消息占用的空间也更大。此外，ProtoJSON 格式会将你的字段名和枚举值名写入编码消息，这会导致后续更改这些名称变得更加困难。移除字段会造成兼容性破坏，并触发解析错误。简而言之，Google 更倾向于在几乎所有场景下使用标准二进制格式而不是 ProtoJSON 格式，是有充分理由的。

各类型的编码方式将在后文的表格中详细说明。

当将 JSON 编码的数据解析为 protocol buffer 时，如果某个值缺失或其值为 `null`，则会被解释为对应的[默认值](./programming-guides/editions#default)。对于单个字段的多次赋值（使用重复或等价的 JSON 键），解析时会保留最后一个值，这与二进制格式解析一致。注意，并非所有 protobuf JSON 解析器实现都符合规范，一些不符合规范的实现可能会拒绝重复键。

当从 protocol buffer 生成 JSON 编码输出时，如果 protobuf 字段为默认值且不支持字段存在性（presence），则默认情况下不会在输出中包含该字段。实现可以提供选项，将默认值字段包含在输出中。

已设置值且支持字段存在性的字段，在 JSON 编码输出中总会包含该字段，即使其为默认值。例如，proto3 中用 `optional` 关键字定义的字段支持存在性，如果被设置，则总会出现在 JSON 输出中。任何版本的 protobuf 中的消息类型字段都支持存在性，如果被设置也会出现在输出中。proto3 中隐式存在性的标量字段，只有在其值不是该类型的默认值时才会出现在 JSON 输出中。

在 JSON 文件中表示数值数据时，如果从二进制格式解析出的数字不适合对应的类型，将会像在 C++ 中强制类型转换一样（例如，将 64 位数字读取为 int32 时会被截断为 32 位）。

下表展示了各类数据在 JSON 文件中的表示方式。

<table>
  <tbody>
    <tr>
      <th>Protobuf</th>
      <th>JSON</th>
      <th>JSON 示例</th>
      <th>说明</th>
    </tr>
    <tr>
      <td>message</td>
      <td>object</td>
      <td><code>{"fooBar": v, "g": null, ...}</code></td>
      <td>生成 JSON 对象。消息字段名会转换为 lowerCamelCase，作为 JSON 对象的键。如果指定了 <code>json_name</code> 字段选项，则使用指定值作为键。解析器同时接受 lowerCamelCase 名（或 <code>json_name</code> 指定的名称）和原始 proto 字段名。<code>null</code> 是所有字段类型的可接受值，并被视为对应字段类型的默认值。但 <code>null</code> 不能用于 <code>json_name</code>。详情见<a href="/news/2023-04-28#json-name">json_name 更严格校验</a>。
      </td>
    </tr>
    <tr>
      <td>enum</td>
      <td>string</td>
      <td><code>"FOO_BAR"</code></td>
      <td>使用 proto 中指定的枚举值名称。解析器同时接受枚举名称和整数值。</td>
    </tr>
    <tr>
      <td>map&lt;K,V&gt;</td>
      <td>object</td>
      <td><code>{"k": v, ...}</code></td>
      <td>所有键都转换为字符串。</td>
    </tr>
    <tr>
      <td>repeated V</td>
      <td>array</td>
      <td><code>[v, ...]</code></td>
      <td><code>null</code> 被视为空列表 <code>[]</code>。</td>
    </tr>
    <tr>
      <td>bool</td>
      <td>true, false</td>
      <td><code>true, false</code></td>
      <td></td>
    </tr>
    <tr>
      <td>string</td>
      <td>string</td>
      <td><code>"Hello World!"</code></td>
      <td></td>
    </tr>
    <tr>
      <td>bytes</td>
      <td>base64 string</td>
      <td><code>"YWJjMTIzIT8kKiYoKSctPUB+"</code></td>
      <td>JSON 值为使用标准 base64 编码（带填充）的字符串。解析时可接受标准或 URL 安全的 base64 编码（带或不带填充）。</td>
    </tr>
    <tr>
      <td>int32, fixed32, uint32</td>
      <td>number</td>
      <td><code>1, -10, 0</code></td>
      <td>JSON 值为十进制数字。解析时可接受数字或字符串。空字符串无效。</td>
    </tr>
    <tr>
      <td>int64, fixed64, uint64</td>
      <td>string</td>
      <td><code>"1", "-10"</code></td>
      <td>JSON 值为十进制字符串。解析时可接受数字或字符串。空字符串无效。</td>
    </tr>
    <tr>
      <td>float, double</td>
      <td>number</td>
      <td><code>1.1, -10.0, 0, "NaN", "Infinity"</code></td>
      <td>JSON 值为数字或特殊字符串 "NaN"、"Infinity"、"-Infinity"。解析时可接受数字或字符串。空字符串无效。也支持指数表示法。</td>
    </tr>
    <tr>
      <td>Any</td>
      <td><code>object</code></td>
      <td><code>{"@type": "url", "f": v, ... }</code></td>
      <td>如果 <code>Any</code> 包含有特殊 JSON 映射的值，则转换为 <code>{"@type": xxx, "value": yyy}</code>。否则，值会被转换为 JSON 对象，并插入 <code>"@type"</code> 字段以指示实际数据类型。</td>
    </tr>
    <tr>
        <td>Timestamp</td>
        <td>string</td>
        <td><code>"1972-01-01T10:00:20.021Z"</code></td>
        <td>使用 RFC 3339 格式，输出总是 Z 标准化，并使用 0、3、6 或 9 位小数。也接受非 "Z" 的时区偏移。</td>
    </tr>
    <tr>
      <td>Duration</td>
      <td>string</td>
      <td><code>"1.000340012s", "1s"</code></td>
      <td>输出总是包含 0、3、6 或 9 位小数，具体取决于所需精度，后缀为 "s"。解析时可接受任意小数位（包括无小数），只要精度不超过纳秒，且必须有 "s" 后缀。</td>
    </tr>
    <tr>
      <td>Struct</td>
      <td><code>object</code></td>
      <td><code>{ ... }</code></td>
      <td>任意 JSON 对象。参见 <code>struct.proto</code>。</td>
    </tr>
    <tr>
      <td>Wrapper types</td>
      <td>多种类型</td>
      <td><code>2, "2", "foo", true, "true", null, 0, ...</code></td>
      <td>包装类型在 JSON 中的表示与被包装的原始类型相同，区别在于 <code>null</code> 允许且在数据转换和传输过程中会被保留。</td>
    </tr>
    <tr>
      <td>FieldMask</td>
      <td>string</td>
      <td><code>"f.fooBar,h"</code></td>
      <td>参见 <code>field_mask.proto</code>。</td>
    </tr>
    <tr>
      <td>ListValue</td>
      <td>array</td>
      <td><code>[foo, bar, ...]</code></td>
      <td></td>
    </tr>
    <tr>
      <td>Value</td>
      <td>value</td>
      <td></td>
      <td>任意 JSON 值。详见
        <a href="/reference/protobuf/google.protobuf#value">google.protobuf.Value</a>。
      </td>
    </tr>
    <tr>
      <td>NullValue</td>
      <td>null</td>
      <td></td>
      <td>JSON null</td>
    </tr>
    <tr>
      <td>Empty</td>
      <td>object</td>
      <td><code>{}</code></td>
      <td>空的 JSON 对象</td>
    </tr>
  </tbody>
</table>

### JSON 选项 {#json-options}

符合规范的 protobuf JSON 实现可以提供以下选项：

*   **总是输出无存在性的字段**：默认情况下，不支持存在性的字段且为默认值时不会出现在 JSON 输出中（如隐式存在性的整数为 0，字符串为空，repeated 和 map 字段为空）。实现可以提供选项，强制输出这些默认值字段。

    截至 v25.x，C++、Java 和 Python 实现尚不完全符合规范，因为该选项会影响 proto2 的 `optional` 字段，但不会影响 proto3 的 `optional` 字段。未来版本将修复此问题。

*   **忽略未知字段**：protobuf JSON 解析器默认应拒绝未知字段，但可以提供选项，在解析时忽略未知字段。

*   **使用 proto 字段名而非 lowerCamelCase 名称**：默认情况下，protobuf JSON 打印器会将字段名转换为 lowerCamelCase 并作为 JSON 名称。实现可以提供选项，使用 proto 字段名作为 JSON 名称。protobuf JSON 解析器必须同时接受 lowerCamelCase 名称和 proto 字段名。

*   **将枚举值以整数而非字符串输出**：默认情况下，JSON 输出使用枚举值的名称。实现可以提供选项，使用枚举值的数字值输出。
