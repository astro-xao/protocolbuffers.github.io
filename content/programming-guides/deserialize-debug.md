+++
title = "反序列化 Debug Proto 表示"
weight = 89
description = "如何在 Protocol Buffers 中记录调试信息。"
type = "docs"
+++

从 30.x 版本开始，Protobuf 的 `DebugString` API（如 `Message::DebugString`、`Message::ShortDebugString`、`Message::Utf8DebugString`）、其他 Protobuf API（如 `proto2::ShortFormat`、`proto2::Utf8Format`）、Abseil 字符串函数（如 `absl::StrCat`、`absl::StrFormat`、`absl::StrAppend` 和 `absl::Substitute`），以及 Abseil 日志 API，将会自动把 proto 参数转换为新的调试格式。相关公告请参见[这里](./news/2024-12-04/)。

与 Protobuf DebugString 输出格式不同，新的调试格式会自动将敏感字段的值替换为字符串 "[REDACTED]"（不带引号）。此外，为确保这种新的输出格式无法被 Protobuf TextFormat 解析器反序列化（无论底层 proto 是否包含 SPII 字段），我们会添加一组随机链接指向本文，并插入随机长度的空白序列。新的调试格式如下所示：

```none
goo.gle/debugstr
spii_field: [REDACTED]
normal_field: "value"
```

请注意，新的调试格式与 DebugString 格式的输出仅有以下两点不同：

*   URL 前缀
*   SPII 字段的值被替换为 "[REDACTED]"（不带引号）

新的调试格式绝不会移除任何字段名；它只会在字段被认为是敏感时将其值替换为 "[REDACTED]"。
**如果你在输出中没有看到某些字段，那是因为这些字段在 proto 中未被设置。**

**提示：** 如果你只看到 URL 而没有其他内容，说明你的 proto 是空的！

## 为什么会有这个 URL？

我们希望确保没有人将面向人类的 protobuf 消息可读表示反序列化。历史上，`.DebugString()` 和 `TextFormat` 是可以互换的，现有系统也常用 DebugString 来传输和存储数据。

我们希望敏感数据不会意外出现在日志中。因此，在将 protobuf 消息转换为字符串（"[REDACTED]"）之前，我们会自动对某些字段值进行脱敏。这降低了意外日志记录带来的安全和隐私风险，但如果其他系统反序列化你的消息，则可能导致数据丢失。为了解决这个风险，我们有意将机器可读的 TextFormat 与用于日志消息的人类可读调试格式分开。

### 为什么我的网页里会有链接？为什么我的代码会产生这种新的“调试表示”？

这是有意为之，为了让你的 proto 的“调试表示”（例如通过日志输出）与 TextFormat 不兼容。我们希望阻止任何人依赖调试机制在程序间传输数据。历史上，调试格式（由 DebugString API 生成）和 TextFormat 被错误地混用。我们希望通过这种有意的努力，防止这种情况继续发生。

我们特意选择了链接而不是不易察觉的格式变化，是为了有机会提供更多上下文。这在 UI 中可能会很显眼，比如在网页表格中显示状态信息。你可以改用 `TextFormat::PrintToString`，它不会脱敏任何信息且保留格式。但请谨慎使用该 API —— 它没有任何内置保护。一般来说，如果你要写入调试日志或生成状态消息，应该继续使用带链接的调试格式。即使你当前没有处理敏感数据，也要记住系统会变化，代码会被复用。

### 我尝试将该消息转换为 TextFormat，但发现每次进程重启格式都会变化。

这是有意为之。不要尝试解析这种调试格式的输出。我们保留随时更改语法的权利。调试格式的语法会在每个进程中随机变化，以防止无意中产生依赖。如果调试格式的语法变化会破坏你的系统，说明你应该使用 TextFormat API，而不是 proto 的调试表示。

## 常见问题

### 我可以到处都用 TextFormat 吗？

不要用 TextFormat 生成日志消息。这会绕过所有内置保护，你有可能意外记录敏感信息。即使你的系统当前没有处理任何敏感数据，未来也可能发生变化。

请区分日志和需要被其他系统进一步处理的信息，分别使用调试表示或 TextFormat。

### 我想写既可读又可被机器读取的配置文件

这种情况下，你可以显式使用 TextFormat。你需要自行确保配置文件不包含任何 PII。

### 我在写单元测试，想在断言中比较 DebugString

如果你想比较 protobuf 值，可以像下面这样使用 `MessageDifferencer`：

```cpp
using google::protobuf::util::MessageDifferencer;
...
MessageDifferencer diff;
...
diff.Compare(foo, bar);
```

除了忽略格式和字段顺序的差异外，你还会获得更好的错误信息。

