+++
title = "Go 生成代码指南（Open）"
weight = 610
linkTitle = "生成代码指南（Open）"
description = "详细描述 protocol buffer 编译器针对任意 proto 定义生成的 Go 代码。"
type = "docs"
+++

proto2 和 proto3 生成代码的差异已高亮标注——注意，这些差异仅体现在本文档描述的生成代码中，基础 API 在两个版本中是相同的。建议在阅读本文档前，先阅读
[proto2 语言指南](./programming-guides/proto2)
和/或
[proto3 语言指南](./programming-guides/proto3)。

{{% alert title="注意" color="warning" %}}您正在查看旧版生成代码 API（Open Struct API）的文档。
参见
[Go 生成代码（Opaque）](./reference/go/go-generated-opaque)
获取新版 Opaque API 的相关文档。Opaque API 的介绍请见
[Go Protobuf: The new Opaque API](https://go.dev/blog/protobuf-opaque)。
{{% /alert %}}

## 编译器调用 {#invocation}

Protocol Buffer 编译器需要插件来生成 Go 代码。使用 Go 1.16 或更高版本，运行以下命令安装：

```shell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

这将在 `$GOBIN` 目录下安装 `protoc-gen-go` 可执行文件。可通过设置 `$GOBIN` 环境变量更改安装位置。该目录需加入您的 `$PATH`，以便编译器找到插件。

编译器通过 `go_out` 标志生成 Go 输出。该标志的参数为输出目录。每个输入的 `.proto` 文件会生成一个源文件，输出文件名将 `.proto` 后缀替换为 `.pb.go`。

生成的 `.pb.go` 文件在输出目录中的位置取决于编译器参数，有以下几种模式：

- 指定 `paths=import` 时，输出文件放在以 Go 包导入路径命名的目录下（如 `.proto` 文件中的 `go_package` 选项）。例如，输入文件 `protos/buzz.proto`，Go 导入路径为 `example.com/project/protos/fizz`，则输出为 `example.com/project/protos/fizz/buzz.pb.go`。如果未指定 `paths`，此为默认模式。
- 指定 `module=$PREFIX` 时，输出文件放在以 Go 包导入路径命名的目录下，但会移除指定前缀。例如，输入文件 `protos/buzz.proto`，Go 导入路径为 `example.com/project/protos/fizz`，指定 `module` 前缀为 `example.com/project`，则输出为 `protos/fizz/buzz.pb.go`。生成的 Go 包若超出模块路径会报错。此模式适合直接输出到 Go module。
- 指定 `paths=source_relative` 时，输出文件与输入文件保持相对目录。例如，输入文件 `protos/buzz.proto`，输出为 `protos/buzz.pb.go`。

`protoc-gen-go` 的专用参数通过 `go_opt` 标志传递，可多次指定。例如：

```shell
protoc --proto_path=src --go_out=out --go_opt=paths=source_relative foo.proto bar/baz.proto
```

编译器会从 `src` 目录读取 `foo.proto` 和 `bar/baz.proto`，并将 `foo.pb.go` 和 `bar/baz.pb.go` 写入 `out` 目录。编译器会自动创建嵌套子目录，但不会自动创建输出目录本身。

## 包 {#package}

每个 `.proto` 文件（包括所有依赖文件）都必须指定 Go 包导入路径。可通过以下两种方式指定：

- 在 `.proto` 文件中声明；
- 在调用 `protoc` 时通过命令行声明。

推荐在 `.proto` 文件中声明，以便集中管理并简化编译参数。如果两处都指定，则命令行参数优先。

在 `.proto` 文件中通过 `go_package` 选项指定导入路径，例如：

```proto
option go_package = "example.com/project/protos/fizz";
```

也可在命令行通过 `M${PROTO_FILE}=${GO_IMPORT_PATH}` 方式指定：

```shell
protoc --proto_path=src \
  --go_opt=Mprotos/buzz.proto=example.com/project/protos/fizz \
  --go_opt=Mprotos/bar.proto=example.com/project/protos/foo \
  protos/buzz.proto protos/bar.proto
```

由于所有 `.proto` 文件的映射可能很大，通常由构建工具（如 [Bazel](https://bazel.build/)）统一管理。如果有重复映射，以最后一个为准。

`go_package` 选项和 `M` 标志的值可包含分号分隔的包名，如 `"example.com/protos/foo;package_name"`。但不推荐这样做，默认会根据导入路径合理推导包名。

导入路径用于生成 `.proto` 文件间的 import 语句。例如，`a.proto` import 了 `b.proto`，则生成的 `a.pb.go` 需 import 包含 `b.pb.go` 的 Go 包（除非两者在同一包）。导入路径也用于构建输出文件名，详见“编译器调用”部分。

Go 导入路径与 `.proto` 文件中的
[`package` 关键字](./programming-guides/proto3#packages)
无关，前者仅用于 Go 命名空间，后者仅用于 protobuf 命名空间。Go 导入路径与 `.proto` 的 import 路径也无关。

## API 级别 {#apilevel}

生成代码会使用 Open Struct API 或 Opaque API。相关介绍见
[Go Protobuf: The new Opaque API](https://go.dev/blog/protobuf-opaque)。

根据 `.proto` 文件的语法，API 级别如下：

`.proto` 语法 | API 级别
------------- | ----------
proto2        | Open Struct API
proto3        | Open Struct API
edition 2023  | Open Struct API
edition 2024+ | Opaque API

可通过在 `.proto` 文件中设置 `api_level` editions 特性选择 API，支持文件级或消息级设置：

```proto
edition = "2023";

package log;

import "google/protobuf/go_features.proto";
option features.(pb.go).api_level = API_OPAQUE;

message LogEntry { … }
```

也可通过 `protoc` 命令行参数全局覆盖默认 API 级别：

```
protoc […] --go_opt=default_api_level=API_HYBRID
```

如需为特定文件覆盖默认 API 级别，使用 `apilevelM` 映射参数（类似于 [导入路径的 `M` 参数](./reference/go/go-generated/#package)）：

```
protoc […] --go_opt=apilevelMhello.proto=API_HYBRID
```

命令行参数同样适用于 proto2/proto3 语法的 `.proto` 文件。但如需在文件内选择 API 级别，需先迁移到 editions。

## 消息 {#message}

给定如下简单消息声明：

```proto
message Artist {}
```

编译器会生成名为 `Artist` 的结构体。`*Artist` 实现了
[`proto.Message`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Message)
接口。

[`proto` 包](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc)
提供了操作消息的函数，包括二进制格式的转换。

`proto.Message` 接口定义了 `ProtoReflect` 方法，返回
[`protoreflect.Message`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Message)
，提供基于反射的消息视图。

`optimize_for` 选项不会影响 Go 代码生成器的输出。

多 goroutine 并发访问同一消息时，需遵循以下规则：

* 并发读取字段是安全的，唯一例外是：
    * 首次访问 [lazy 字段](https://github.com/protocolbuffers/protobuf/blob/cacb096002994000f8ccc6d9b8e1b5b0783ee561/src/google/protobuf/descriptor.proto#L609) 属于修改操作。
* 并发修改不同字段是安全的。
* 并发修改同一字段不安全。
* 并发修改消息与调用 [`proto` 包](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc) 的函数（如
    [`proto.Marshal`](https://pkg.go.dev/google.golang.org/protobuf/proto#Marshal)
    或 [`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size)
    ）不安全。

### 嵌套类型

消息可嵌套声明。例如：

```proto
message Artist {
  message Name {
  }
}
```

此时编译器会生成两个结构体：`Artist` 和 `Artist_Name`。

## 字段

编译器会为消息中定义的每个字段生成结构体字段。字段类型及其为单个、repeated、map 或 oneof 字段决定了具体生成方式。

注意，生成的 Go 字段名总是采用驼峰命名，即使 `.proto` 文件中使用下划线（[推荐如此](./programming-guides/style#message-field-names)）。
大小写转换规则如下：

1. 首字母大写以导出。如果首字符为下划线，则移除并加前缀 X。
2. 内部下划线后跟小写字母时，移除下划线并将后字母大写。

因此，proto 字段 `birth_year` 变为 Go 的 `BirthYear`，`_birth_year_2` 变为 `XBirthYear_2`。

### 单一标量字段（proto2） {#singular-scalar-proto2}

如下字段定义：

```proto
optional int32 birth_year = 1;
required int32 birth_year = 1;
```

编译器会生成带有 `*int32` 字段 `BirthYear` 的结构体，并生成 `GetBirthYear()` 方法，返回 `Artist` 中的 `int32` 值或默认值。若未显式设置默认值，则使用该类型的[零值](https://golang.org/ref/spec#The_zero_value)（数字为 0，字符串为空）。

其他标量类型（如 `bool`、`bytes`、`string`）会用对应的 Go 类型，详见
[标量值类型表](./programming-guides/proto2#scalar)。

### 单一标量字段（proto3） {#singular-scalar-proto3}

如下字段定义：

```proto
int32 birth_year = 1;
optional int32 first_active_year = 2;
```

编译器会生成带有 `int32` 字段 `BirthYear` 的结构体，并生成 `GetBirthYear()` 方法，返回 `birth_year` 的 `int32` 值或该类型的[零值](https://golang.org/ref/spec#The_zero_value)（数字为 0，字符串为空）。

`FirstActiveYear` 字段为 `*int32` 类型，因为它被标记为 `optional`。

其他标量类型（如 `bool`、`bytes`、`string`）会用对应的 Go 类型，详见
[标量值类型表](./programming-guides/proto3#scalar)。
proto 中未设置的值会以该类型的[零值](https://golang.org/ref/spec#The_zero_value)表示。

### 单一消息字段 {#singular-message}

给定如下消息类型：

```proto
message Band {}
```

消息中有 `Band` 字段：

```proto
// proto2
message Concert {
  optional Band headliner = 1;
  // required 效果相同
}

// proto3
message Concert {
  Band headliner = 1;
}
```

编译器会生成如下 Go 结构体：

```go
type Concert struct {
    Headliner *Band
}
```

消息字段可设为 `nil`，表示未设置（即清除字段），这与赋值为空结构体不同。

编译器还会生成 `func (m *Concert) GetHeadliner() *Band` 辅助函数。若 `m` 为 nil 或 `headliner` 未设置，则返回 nil，可链式调用避免中间 nil 检查：

```go
var m *Concert // 默认为 nil
log.Infof("GetFoundingYear() = %d (no panic!)", m.GetHeadliner().GetFoundingYear())
```

### repeated 字段 {#repeated}

每个 repeated 字段在 Go 结构体中生成 `T` 类型的切片字段，`T` 为元素类型。例如：

```proto
message Concert {
  // 最佳实践：repeated 字段用复数名
  repeated Band support_acts = 1;
}
```

编译器生成：

```go
type Concert struct {
    SupportActs []*Band
}
```

如 `repeated bytes band_promo_images = 1;`，则生成 `[][]byte` 字段 `BandPromoImage`。如 repeated 枚举 `repeated MusicGenre genres = 2;`，则生成 `[]MusicGenre` 字段 `Genre`。

设置字段示例：

```go
concert := &Concert{
  SupportActs: []*Band{
    {}, // 第一个元素
    {}, // 第二个元素
  },
}
```

访问字段示例：

```go
support := concert.GetSupportActs() // 类型为 []*Band
b1 := support[0] // 类型为 *Band，为 support_acts 的第一个元素
```

### map 字段 {#map}

每个 map 字段在结构体中生成 `map[TKey]TValue` 类型字段，`TKey` 为键类型，`TValue` 为值类型。例如：

```proto
message MerchItem {}

message MerchBooth {
  // items 映射商品名到 MerchItem 消息
  map<string, MerchItem> items = 1;
}
```

编译器生成：

```go
type MerchBooth struct {
    Items map[string]*MerchItem
}
```

### oneof 字段 {#oneof}

对于 oneof 字段，编译器生成一个接口类型字段 `isMessageName_MyField`，并为 oneof 内的每个[单一字段](#singular-scalar-proto2)生成结构体，这些结构体都实现该接口。

例如：

```proto
package account;
message Profile {
  oneof avatar {
    string image_url = 1;
    bytes image_data = 2;
  }
}
```

编译器生成：

```go
type Profile struct {
    // 可赋值类型：
    //  *Profile_ImageUrl
    //  *Profile_ImageData
    Avatar isProfile_Avatar `protobuf_oneof:"avatar"`
}

type Profile_ImageUrl struct {
        ImageUrl string
}
type Profile_ImageData struct {
        ImageData []byte
}
```

`*Profile_ImageUrl` 和 `*Profile_ImageData` 都通过空方法实现 `isProfile_Avatar`。

设置字段示例：

```go
p1 := &account.Profile{
  Avatar: &account.Profile_ImageUrl{ImageUrl: "http://example.com/image.png"},
}

// imageData 为 []byte
imageData := getImageData()
p2 := &account.Profile{
  Avatar: &account.Profile_ImageData{ImageData: imageData},
}
```

访问字段时可用类型 switch：

```go
switch x := m.Avatar.(type) {
case *account.Profile_ImageUrl:
    // 通过 x.ImageUrl 加载图片
case *account.Profile_ImageData:
    // 通过 x.ImageData 加载图片
case nil:
    // 字段未设置
default:
    return fmt.Errorf("Profile.Avatar 类型异常 %T", x)
}
```

编译器还会生成 `func (m *Profile) GetImageUrl() string` 和 `func (m *Profile) GetImageData() []byte` 方法。每个 get 方法返回字段值或未设置时的零值。

## 枚举 {#enum}

给定如下枚举：

```proto
message Venue {
  enum Kind {
    KIND_UNSPECIFIED = 0;
    KIND_CONCERT_HALL = 1;
    KIND_STADIUM = 2;
    KIND_BAR = 3;
    KIND_OPEN_AIR_FESTIVAL = 4;
  }
  Kind kind = 1;
  // ...
}
```

编译器生成类型及常量：

```go
type Venue_Kind int32

const (
    Venue_KIND_UNSPECIFIED       Venue_Kind = 0
    Venue_KIND_CONCERT_HALL      Venue_Kind = 1
    Venue_KIND_STADIUM           Venue_Kind = 2
    Venue_KIND_BAR               Venue_Kind = 3
    Venue_KIND_OPEN_AIR_FESTIVAL Venue_Kind = 4
)
```

消息内的枚举类型名以消息名为前缀：

```go
type Venue_Kind int32
```

包级枚举：

```proto
enum Genre {
  GENRE_UNSPECIFIED = 0;
  GENRE_ROCK = 1;
  GENRE_INDIE = 2;
  GENRE_DRUM_AND_BASS = 3;
  // ...
}
```

Go 类型名与 proto 枚举名一致：

```go
type Genre int32
```

该类型有 `String()` 方法返回值名。

`Enum()` 方法初始化新分配的内存并返回指针：

```go
func (Genre) Enum() *Genre
```

编译器为每个枚举值生成常量。消息内枚举常量以消息名为前缀：

```go
const (
    Venue_KIND_UNSPECIFIED       Venue_Kind = 0
    Venue_KIND_CONCERT_HALL      Venue_Kind = 1
    Venue_KIND_STADIUM           Venue_Kind = 2
    Venue_KIND_BAR               Venue_Kind = 3
    Venue_KIND_OPEN_AIR_FESTIVAL Venue_Kind = 4
)
```

包级枚举常量以枚举名为前缀：

```go
const (
    Genre_GENRE_UNSPECIFIED   Genre = 0
    Genre_GENRE_ROCK          Genre = 1
    Genre_GENRE_INDIE         Genre = 2
    Genre_GENRE_DRUM_AND_BASS Genre = 3
)
```

编译器还会生成从整数到字符串名、从字符串名到整数的映射：

```go
var Genre_name = map[int32]string{
    0: "GENRE_UNSPECIFIED",
    1: "GENRE_ROCK",
    2: "GENRE_INDIE",
    3: "GENRE_DRUM_AND_BASS",
}
var Genre_value = map[string]int32{
    "GENRE_UNSPECIFIED":   0,
    "GENRE_ROCK":          1,
    "GENRE_INDIE":         2,
    "GENRE_DRUM_AND_BASS": 3,
}
```

注意，`.proto` 语言允许多个枚举符号有相同数值，这些为同义词。在 Go 中表现为多个名称对应同一数值。反向映射只包含 `.proto` 文件中首个出现的名称。

## 扩展（proto2） {#extensions}

给定扩展定义：

```proto
extend Concert {
  optional int32 promo_id = 123;
}
```

编译器会生成名为 `E_Promo_id` 的
[`protoreflect.ExtensionType`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#ExtensionType)
值。可用
[`proto.GetExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#GetExtension)、
[`proto.SetExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#SetExtension)、
[`proto.HasExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#HasExtension)、
[`proto.ClearExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#ClearExtension)
等函数访问扩展。`GetExtension` 和 `SetExtension` 分别返回和接受包含扩展值类型的 `interface{}`。

单一标量扩展字段的类型为
[标量值类型表](./programming-guides/proto3#scalar)
中的 Go 类型。

单一嵌入消息扩展字段类型为 `*M`，`M` 为消息类型。

repeated 扩展字段类型为单一类型的切片。

例如：

```proto
extend Concert {
  optional int32 singular_int32 = 1;
  repeated bytes repeated_strings = 2;
  optional Band singular_message = 3;
}
```

扩展值访问示例：

```go
m := &somepb.Concert{}
proto.SetExtension(m, extpb.E_SingularInt32, int32(1))
proto.SetExtension(m, extpb.E_RepeatedString, []string{"a", "b", "c"})
proto.SetExtension(m, extpb.E_SingularMessage, &extpb.Band{})

v1 := proto.GetExtension(m, extpb.E_SingularInt32).(int32)
v2 := proto.GetExtension(m, extpb.E_RepeatedString).([][]byte)
v3 := proto.GetExtension(m, extpb.E_SingularMessage).(*extpb.Band)
```

扩展可嵌套声明。例如：

```proto
message Promo {
  extend Concert {
    optional int32 promo_id = 124;
  }
}
```

此时 `ExtensionType` 名为 `E_Promo_Concert`。

## 服务 {#service}

Go 代码生成器默认不为服务生成代码。如需支持 [gRPC](https://www.grpc.io/)，请启用 gRPC 插件（参见
[gRPC Go 快速入门](https://github.com/grpc/grpc-go/tree/master/examples)）。
