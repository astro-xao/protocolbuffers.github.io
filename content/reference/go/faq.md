+++
title = "Go 常见问题"
weight = 620
linkTitle = "常见问题"
description = "关于在 Go 中实现 Protocol Buffers 的常见问题及解答。"
type = "docs"
+++

## 版本

### `github.com/golang/protobuf` 和 `google.golang.org/protobuf` 有什么区别？ {#modules}

[`github.com/golang/protobuf`](https://pkg.go.dev/github.com/golang/protobuf?tab=overview) 模块是最初的 Go Protocol Buffers API。

[`google.golang.org/protobuf`](https://pkg.go.dev/google.golang.org/protobuf?tab=overview) 模块是该 API 的更新版本，设计更简洁、易用且安全。新版 API 的主要特性包括反射支持，以及将用户接口与底层实现分离。

我们建议新代码使用 `google.golang.org/protobuf`。

`github.com/golang/protobuf` 的 v1.4.0 及更高版本会封装新实现，允许程序逐步采用新 API。例如，`github.com/golang/protobuf/ptypes` 中定义的知名类型只是新模块中类型的别名。因此，[`google.golang.org/protobuf/types/known/emptypb`](https://pkg.go.dev/google.golang.org/protobuf/types/known/emptypb) 和 [`github.com/golang/protobuf/ptypes/empty`](https://pkg.go.dev/github.com/golang/protobuf/ptypes/empty) 可以互换使用。

### 什么是 `proto1`、`proto2` 和 `proto3`？ {#proto-versions}

这些是 Protocol Buffers *语言* 的不同版本，与 Go 的 *实现* 无关。

*   `proto3` 是当前的语言版本，也是最常用的版本。我们建议新代码使用 proto3。
*   `proto2` 是较早的版本，虽然已被 proto3 取代，但仍然完全支持。
*   `proto1` 是已废弃的版本，从未开源。

### 有多个 `Message` 类型，我该用哪个？ {#message-types}

*   [`"google.golang.org/protobuf/proto".Message`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Message) 是当前 Protocol Buffers 编译器生成的所有消息实现的接口类型。用于操作任意消息的函数（如 [`proto.Marshal`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Marshal) 或 [`proto.Clone`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Clone)）接受或返回此类型。

*   [`"google.golang.org/protobuf/reflect/protoreflect".Message`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Message) 是描述消息反射视图的接口类型。

    可通过 `proto.Message` 的 `ProtoReflect` 方法获得 `protoreflect.Message`。

*   [`"google.golang.org/protobuf/reflect/protoreflect".ProtoMessage`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#ProtoMessage) 是 `"google.golang.org/protobuf/proto".Message` 的别名，两者可互换。

*   [`"github.com/golang/protobuf/proto".Message`](https://pkg.go.dev/github.com/golang/protobuf/proto?tab=doc#Message) 是旧版 Go Protocol Buffers API 定义的接口类型。所有生成的消息类型都实现了该接口，但该接口未描述这些消息应有的行为。新代码应避免使用此类型。

## 常见问题

### "`go install`": `working directory is not part of a module` {#working-directory}

在 Go 1.15 及以下版本中，如果你设置了环境变量 `GO111MODULE=on` 并在非模块目录下运行 `go install`，请将 `GO111MODULE` 设置为 `auto` 或取消该环境变量。

在 Go 1.16 及以上版本中，可以通过指定明确的版本在模块外运行 `go install`：`go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`

### `constant -1 overflows protoimpl.EnforceVersion` {#enforce-version}

你正在使用需要更新版本 `"google.golang.org/protobuf"` 模块的生成 `.pb.go` 文件。

请使用以下命令更新：

```shell
go get -u google.golang.org/protobuf/proto
```

### `undefined: "github.com/golang/protobuf/proto".ProtoPackageIsVersion4` {#enforce-version-apiv1}

你正在使用需要更新版本 `"github.com/golang/protobuf"` 模块的生成 `.pb.go` 文件。

请使用以下命令更新：

```shell
go get -u github.com/golang/protobuf/proto
```

### 什么是 Protocol Buffers 命名空间冲突？ {#namespace-conflict}

所有链接到 Go 二进制文件的 Protocol Buffers 声明都会被插入到全局注册表中。

每个 protobuf 声明（如枚举、枚举值或消息）都有一个绝对名称，即 [包名](./programming-guides/proto2#packages) 与 `.proto` 源文件中声明的相对名称拼接而成（如 `my.proto.package.MyMessage.NestedMessage`）。protobuf 语言假定所有声明都是全局唯一的。

如果两个 protobuf 声明在 Go 二进制文件中具有相同名称，就会导致命名空间冲突，注册表将无法正确解析该声明。根据使用的 Go protobuf 版本，这可能会在初始化时 panic，或静默丢弃冲突，导致运行时潜在 bug。

### 如何解决 Protocol Buffers 命名空间冲突？ {#fix-namespace-conflict}

解决命名空间冲突的方法取决于冲突的原因。

常见冲突原因：

*   **被 vendored 的 .proto 文件。** 当同一个 `.proto` 文件被生成到两个或多个 Go 包并链接到同一个 Go 二进制文件时，会导致所有声明冲突。通常发生在 `.proto` 文件被 vendored 并生成 Go 包，或生成的 Go 包本身被 vendored。建议避免 vendor，改为依赖该 `.proto` 文件的集中式 Go 包。

    *   如果 `.proto` 文件由外部方维护且缺少 `go_package` 选项，应与该文件所有者协作，指定一个集中式 Go 包，供多个用户依赖。

*   **缺失或过于通用的 proto 包名。** 如果 `.proto` 文件未指定包名或包名过于通用（如 "my_service"），则很可能与其他声明冲突。建议每个 `.proto` 文件都使用全局唯一的包名（如加公司前缀）。

{{% alert title="警告" color="warning" %}}
对 `.proto` 文件的包名进行追溯性更改会破坏向后兼容性，尤其是用于扩展字段、存储在 `google.protobuf.Any` 或 gRPC 服务定义的类型。{{% /alert %}}

从 `google.golang.org/protobuf` v1.26.0 开始，Go 程序启动时如果存在多个冲突的 protobuf 名称会报硬错误。虽然最好修复冲突源，但可以通过以下两种方式临时规避致命错误：

1.  **编译时。** 可通过链接器初始化变量指定冲突处理行为：`go build -ldflags "-X google.golang.org/protobuf/reflect/protoregistry.conflictPolicy=warn"`
2.  **程序运行时。** 可通过环境变量设置冲突处理行为：`GOLANG_PROTOBUF_REGISTRATION_CONFLICT=warn ./main`

### 为什么 `reflect.DeepEqual` 在比较 protobuf 消息时表现异常？ {#deepequal}

生成的 Protocol Buffers 消息类型包含内部状态，即使消息等价也可能不同。

此外，`reflect.DeepEqual` 并不了解 protobuf 消息的语义，可能会报告本质上相等的消息为不等。例如，map 字段中一个为 `nil`，另一个为零长度非 `nil`，语义上等价，但 `reflect.DeepEqual` 会认为不等。

请使用 [`proto.Equal`](https://pkg.go.dev/google.golang.org/protobuf/proto#Equal) 比较消息值。

在测试中，也可以使用 [`"github.com/google/go-cmp/cmp"`](https://pkg.go.dev/github.com/google/go-cmp/cmp?tab=doc) 包配合 [`protocmp.Transform()`](https://pkg.go.dev/google.golang.org/protobuf/testing/protocmp#Transform) 选项。`cmp` 可比较任意数据结构，[`cmp.Diff`](https://pkg.go.dev/github.com/google/go-cmp/cmp#Diff) 可生成可读性强的差异报告。

```go
if diff := cmp.Diff(a, b, protocmp.Transform()); diff != "" {
  t.Errorf("unexpected difference:\n%v", diff)
}
```

## Hyrum 定律

### 什么是 Hyrum 定律，为什么要在此说明？ {#hyrums-law}

[Hyrum 定律](https://www.hyrumslaw.com/) 指：

> 当 API 用户足够多时，无论你在契约中承诺了什么：你系统的所有可观察行为都会被某些人依赖。

Go Protocol Buffers API 最新版的设计目标之一，是尽量避免提供无法承诺长期稳定的可观察行为。我们认为，在未承诺的领域保持不稳定，比给出虚假的稳定性承诺更好，否则项目长期依赖后再变更会带来更大影响。

### 为什么错误文本经常变化？ {#unstable-errors}

依赖错误文本的测试很脆弱，文本变化时容易出错。为防止测试依赖错误文本，本模块产生的错误文本是故意不稳定的。

如果需要判断错误是否由 [`protobuf`](https://pkg.go.dev/mod/google.golang.org/protobuf) 模块产生，我们保证所有错误都能通过 [`proto.Error`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Error) 和 [`errors.Is`](https://pkg.go.dev/errors?tab=doc#Is) 匹配。

### 为什么 [`protojson`](https://pkg.go.dev/google.golang.org/protobuf/encoding/protojson) 的输出经常变化？ {#unstable-json}

我们不承诺 Go 实现的 Protocol Buffers [JSON 格式](./programming-guides/proto3#json) 长期稳定。规范只规定了合法 JSON，但未规定 *规范化* 格式。为避免输出看似稳定，我们故意引入细微差异，使字节级比较容易失败。

如需一定的输出稳定性，建议用 JSON 格式化工具处理输出。

### 为什么 [`prototext`](https://pkg.go.dev/google.golang.org/protobuf/encoding/prototext) 的输出经常变化？ {#unstable-text}

我们不承诺 Go 实现的文本格式长期稳定。protobuf 文本格式没有规范化标准，我们希望未来能改进 `prototext` 输出。为防止用户依赖其稳定性，我们故意引入不稳定。

如需一定的稳定性，建议用 [`txtpbfmt`](https://github.com/protocolbuffers/txtpbfmt) 格式化输出。可在 Go 中直接调用 [`parser.Format`](https://pkg.go.dev/github.com/protocolbuffers/txtpbfmt/parser?tab=doc#Format)。

## 其他

### 如何将 Protocol Buffers 消息用作哈希键？ {#hash}

你需要规范化序列化，即消息序列化输出在时间上保证稳定。遗憾的是，目前没有规范化序列化的标准。你需要自行实现，或避免这种需求。

### 我可以为 Go Protocol Buffers 实现添加新特性吗？ {#new-feature}

也许可以。我们欢迎建议，但对新增内容非常谨慎。

Go 实现力求与其他语言实现一致，因此我们不倾向于添加仅适用于 Go 的特性。Go 特有的特性会影响 Protocol Buffers 作为语言无关数据交换格式的目标。

除非你的想法仅针对 Go 实现，否则建议加入 [protobuf 讨论组](http://groups.google.com/group/protobuf) 提出建议。

如有 Go 实现相关建议，请在我们的 issue 跟踪器提交：[https://github.com/golang/protobuf/issues](https://github.com/golang/protobuf/issues)

### 我可以为 `Marshal` 或 `Unmarshal` 添加自定义选项吗？ {#new-marshal-option}

只有当其他实现（如 C++、Java）也有该选项时才可以。Protocol Buffers 的编码（包括二进制、JSON 和文本）必须跨实现一致，这样不同语言间才能互通。

我们不会为 Go 实现添加影响 `Marshal` 或 `Unmarshal` 输出数据的选项，除非至少有一个其他受支持实现也有等价选项。

### 我可以自定义 `protoc-gen-go` 生成的代码吗？ {#custom-code}

通常不可以。Protocol Buffers 旨在成为语言无关的数据交换格式，特定实现的自定义化与此目标相悖。
