+++
title = "Go Size 语义"
weight = 630
linkTitle = "Size 语义"
description = "解释如何（不）使用 proto.Size"
type = "docs"
+++

[`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size)
函数通过遍历 proto.Message 的所有字段（包括子消息），返回其线格式编码的字节大小。

特别地，它返回的是**Go Protobuf 如何编码该消息时的大小**。

## 典型用法

### 判断消息是否为空

检查
[`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size) 是否返回 0，
是识别空消息的简单方法：

```go
if proto.Size(m) == 0 {
    // 没有设置字段（或在 proto3 中，所有字段都为默认值）；
    // 跳过处理该消息，或返回错误等。
}
```

### 限制程序输出的大小

假设你正在编写一个批处理管道，用于生成工作任务，交给下游系统处理。
下游系统适合处理小到中等规模的任务，但负载测试显示，当任务超过 500 MB 时，
系统会出现级联故障。

最佳做法是为下游系统增加保护（参见
https://cloud.google.com/blog/products/gcp/using-load-shedding-to-survive-a-success-disaster-cre-life-lessons），
但如果无法实现负载卸载，可以在管道中添加一个快速修复：

```go {highlight="context:1,proto.Size,1"}
func (*beamFn) ProcessElement(key string, value []byte, emit func(proto.Message)) {
  task := produceWorkTask(value)
  if proto.Size(task) > 500 * 1024 * 1024 {
    // 跳过所有超过 500 MB 的任务，避免压垮脆弱的下游系统。
    return
  }
  emit(task)
}
```

## 错误用法：与 Unmarshal 无关

由于 [`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size)
返回的是 Go Protobuf 编码消息时的字节数，因此在反序列化（解码）Protobuf 消息流时，
使用 `proto.Size` 是不安全的：

```go {highlight="context:1,proto.Size,1"}
func bytesToSubscriptionList(data []byte) ([]*vpb.EventSubscription, error) {
    subList := []*vpb.EventSubscription{}
    for len(data) > 0 {
        subscription := &vpb.EventSubscription{}
        if err := proto.Unmarshal(data, subscription); err != nil {
            return nil, err
        }
        subList = append(subList, subscription)
        data = data[:len(data)-proto.Size(subscription)]
    }
    return subList, nil
}
```

当 `data` 包含[非最小线格式](#non-minimal)的消息时，
`proto.Size` 可能返回与实际解码长度不同的大小，导致解析错误（最好的情况），
或最坏情况下解析出错的数据。

因此，该示例仅在所有输入消息都由（同一版本的）Go Protobuf 生成时才可靠。
这通常令人意外，也不是预期的行为。

**提示：**建议使用
[`protodelim` 包](https://pkg.go.dev/google.golang.org/protobuf/encoding/protodelim)
来读写 Protobuf 消息的定长流。

## 高级用法：预分配缓冲区

[`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size) 的一个高级用法是，
在序列化前确定所需缓冲区大小：

```go
opts := proto.MarshalOptions{
    // 可能避免 Marshal 内部额外调用 proto.Size（见文档）：
    UseCachedSize: true,
}
// 切勿提交未实现此优化的代码：
// 不要直接分配，而是从池中获取足够大的缓冲区。
// 知道缓冲区大小后，可以丢弃池中的异常值，防止长时间运行的 RPC 服务内存无限增长。
buf := make([]byte, 0, opts.Size(m))
var err error
buf, err = opts.MarshalAppend(buf, m) // 不会分配新内存
// 注意 len(buf) 可能小于 cap(buf)！详见下文：
```

注意，当启用延迟解码时，`proto.Size` 可能返回的字节数比 `proto.Marshal`
（及其变体如 `proto.MarshalAppend`）实际写入的要多！
因此，在将编码字节写入网络或磁盘时，应使用 `len(buf)`，并丢弃之前的 `proto.Size` 结果。

具体来说，当出现以下情况时，(子)消息在 `proto.Size` 和 `proto.Marshal` 之间可能会“变小”：

1.  启用了延迟解码
2.  消息以[非最小线格式](#non-minimal)到达
3.  在调用 `proto.Size` 前未访问消息，即尚未解码
4.  在 `proto.Size` 后（但在 `proto.Marshal` 前）访问了消息，导致其被延迟解码

解码后，随后的 `proto.Marshal` 会对消息重新编码（而不是仅复制其线格式），
从而隐式归一化为 Go 的编码方式，目前是最小线格式（但不要依赖这一点！）。

如你所见，这种情况较为特殊，但**最佳实践是将 `proto.Size` 结果视为上限**，
切勿假定其与实际编码后的消息大小一致。

## 背景：非最小线格式 {#non-minimal}

在编码 Protobuf 消息时，存在一种*最小线格式大小*，以及若干更大的*非最小线格式*，
它们解码后得到相同的消息。

非最小线格式（有时也称为“非规范化线格式”）指以下场景：
非重复字段多次出现、变长整型编码不最优、打包的重复字段在线上非打包等。

我们可能在以下场景遇到非最小线格式：

*   **有意为之。** Protobuf 支持通过拼接线格式来拼接消息。
*   **无意为之。** 某些（可能是第三方的）Protobuf 编码器编码时未最优化（如变长整型编码占用空间过大）。
*   **恶意攻击。** 攻击者可能特意构造 Protobuf 消息，试图通过网络触发崩溃。
