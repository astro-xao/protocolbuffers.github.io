+++
title = "Go 生成代码指南（Opaque）"
weight = 615
linkTitle = "生成代码指南（Opaque）"
description = "详细描述 protocol buffer 编译器针对任意协议定义生成的 Go 代码。"
type = "docs"
+++

proto2 和 proto3 生成代码的任何差异都会被高亮标注——请注意，这些差异仅体现在本文档描述的生成代码中，基础 API 在两个版本中是相同的。建议在阅读本文档前，先阅读
[proto2 语言指南](./programming-guides/proto2)
和/或
[proto3 语言指南](./programming-guides/proto3)。

{{% alert title="注意" color="warning" %}}您正在查看 Opaque API 的文档，这是当前版本。如果您正在处理使用旧版 Open Struct API 的 .proto 文件（可通过 .proto 文件中的 API level 设置判断），请参阅
[Go 生成代码（Open）](./reference/go/go-generated)
获取相应文档。Opaque API 的介绍见
[Go Protobuf: The new Opaque API](https://go.dev/blog/protobuf-opaque)。{{% /alert %}}

## 编译器调用 {#invocation}

Protocol Buffer 编译器需要插件来生成 Go 代码。使用 Go 1.16 或更高版本，运行以下命令安装：

```shell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

这将在 `$GOBIN` 目录下安装 `protoc-gen-go` 二进制文件。通过设置 `$GOBIN` 环境变量可更改安装位置。该目录必须在您的 `$PATH` 中，以便编译器找到它。

编译器通过 `go_out` 标志生成 Go 输出。该标志的参数为您希望编译器写入 Go 输出的目录。每个输入的 `.proto` 文件会生成一个源文件，输出文件名为将 `.proto` 扩展名替换为 `.pb.go`。

输出目录下 `.pb.go` 文件的具体位置取决于编译器标志，有以下几种模式：

- 指定 `paths=import` 标志时，输出文件放在以 Go 包导入路径命名的目录下（如 `.proto` 文件中的 `go_package` 选项）。例如，输入文件 `protos/buzz.proto`，Go 导入路径为 `example.com/project/protos/fizz`，输出文件为 `example.com/project/protos/fizz/buzz.pb.go`。如果未指定 `paths` 标志，这是默认模式。
- 指定 `module=$PREFIX` 标志时，输出文件放在以 Go 包导入路径命名的目录下，但会移除指定前缀。例如，输入文件 `protos/buzz.proto`，Go 导入路径为 `example.com/project/protos/fizz`，指定 `example.com/project` 作为 `module` 前缀，输出文件为 `protos/fizz/buzz.pb.go`。生成的 Go 包超出模块路径会报错。该模式适合直接输出到 Go module。
- 指定 `paths=source_relative` 标志时，输出文件与输入文件在相对目录下。例如，输入文件 `protos/buzz.proto`，输出文件为 `protos/buzz.pb.go`。

`protoc-gen-go` 的特定标志通过 `go_opt` 传递。可以传递多个 `go_opt`。例如：

```shell
protoc --proto_path=src --go_out=out --go_opt=paths=source_relative foo.proto bar/baz.proto
```

编译器会从 `src` 目录读取 `foo.proto` 和 `bar/baz.proto`，并将 `foo.pb.go` 和 `bar/baz.pb.go` 写入 `out` 目录。编译器会自动创建必要的子目录，但不会自动创建输出目录本身。

## 包 {#package}

生成 Go 代码时，必须为每个 `.proto` 文件（包括所有依赖的 `.proto` 文件）提供 Go 包导入路径。有两种方式指定：

- 在 `.proto` 文件中声明
- 在调用 `protoc` 时通过命令行声明

推荐在 `.proto` 文件中声明，这样可以集中管理，并简化编译命令。如果同时在文件和命令行指定，以命令行为准。

在 `.proto` 文件中通过 `go_package` 选项指定导入路径。例如：

```proto
option go_package = "example.com/project/protos/fizz";
```

也可以在命令行通过 `M${PROTO_FILE}=${GO_IMPORT_PATH}` 方式指定。例如：

```shell
protoc --proto_path=src \
  --go_opt=Mprotos/buzz.proto=example.com/project/protos/fizz \
  --go_opt=Mprotos/bar.proto=example.com/project/protos/foo \
  protos/buzz.proto protos/bar.proto
```

由于所有 `.proto` 文件的映射可能很大，通常由构建工具（如 [Bazel](https://bazel.build/)）自动处理。如果有重复映射，以最后一个为准。

`go_package` 选项和 `M` 标志的值可以包含用分号分隔的包名，如 `"example.com/protos/foo;package_name"`。不推荐这样用，默认会根据导入路径合理推导包名。

导入路径用于生成 import 语句。例如，`a.proto` 导入 `b.proto`，则 `a.pb.go` 需导入包含 `b.pb.go` 的 Go 包（除非同包）。导入路径也用于构建输出文件名，详见“编译器调用”部分。

Go 导入路径与 `.proto` 文件中的
[`package` 说明符](./programming-guides/proto3#packages)
无关，后者仅用于 protobuf 命名空间，前者仅用于 Go 命名空间。Go 导入路径与 `.proto` 导入路径也无关。

## API 级别 {#apilevel}

生成的代码会使用 Open Struct API 或 Opaque API。介绍见
[Go Protobuf: The new Opaque API](https://go.dev/blog/protobuf-opaque)。

根据 `.proto` 文件的语法，API 级别如下：

`.proto` 语法 | API 级别
------------- | ----------
proto2        | Open Struct API
proto3        | Open Struct API
edition 2023  | Open Struct API
edition 2024+ | Opaque API

可通过在 `.proto` 文件设置 `api_level` editions 特性选择 API，支持按文件或按消息设置：

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

如需仅为特定文件覆盖，使用 `apilevelM` 映射标志（类似于 [导入路径的 `M` 标志](#package)）：

```
protoc […] --go_opt=apilevelMhello.proto=API_HYBRID
```

命令行标志同样适用于 proto2 或 proto3 语法的 `.proto` 文件，但如需在文件内选择 API 级别，需先迁移到 editions。

## 消息 {#message}

给定如下简单消息声明：

```proto
message Artist {}
```

编译器会生成名为 `Artist` 的结构体。`*Artist` 实现
[`proto.Message`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Message)
接口。

[`proto` 包](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc)
提供了操作消息的函数，包括二进制格式的转换。

`proto.Message` 接口定义了 `ProtoReflect` 方法，返回
[`protoreflect.Message`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Message)
，提供基于反射的消息视图。

`optimize_for` 选项不会影响 Go 代码生成器的输出。

多 goroutine 并发访问同一消息时，规则如下：

* 并发读取字段是安全的，唯一例外是：
    * 首次访问 [lazy field](https://github.com/protocolbuffers/protobuf/blob/cacb096002994000f8ccc6d9b8e1b5b0783ee561/src/google/protobuf/descriptor.proto#L609) 属于修改操作。
* 并发修改不同字段是安全的。
* 并发修改同一字段不安全。
* 与 [`proto` 包](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc) 的函数（如 [`proto.Marshal`](https://pkg.go.dev/google.golang.org/protobuf/proto#Marshal) 或 [`proto.Size`](https://pkg.go.dev/google.golang.org/protobuf/proto#Size)）并发修改消息不安全。

### 嵌套类型

消息可以嵌套声明。例如：

```proto
message Artist {
  message Name {
  }
}
```

此时，编译器会生成两个结构体：`Artist` 和 `Artist_Name`。

## 字段

编译器会为每个消息字段生成访问器方法（setter 和 getter）。

注意，生成的 Go 访问器方法总是使用驼峰命名法，即使 `.proto` 文件中字段名为下划线风格（[推荐用法](./programming-guides/style)）。大小写转换规则如下：

1. 首字母大写以导出。如果首字符为下划线，则移除并加前缀 X。
2. 内部下划线后跟小写字母时，移除下划线并将后字母大写。

因此，proto 字段 `birth_year` 可通过 `GetBirthYear()` 访问，`_birth_year_2` 可通过 `GetXBirthYear_2()` 访问。

### 单一标量字段（proto2） {#singular-scalar-proto2}

如下字段定义：

```proto
optional int32 birth_year = 1;
required int32 birth_year = 1;
```

编译器生成如下访问器方法：

```go
func (m *Artist) GetBirthYear() int32 { ... }
func (m *Artist) SetBirthYear(v int32) { ... }
func (m *Artist) HasBirthYear() bool { ... }
func (m *Artist) ClearBirthYear() { ... }
```

`GetBirthYear()` 返回 `birth_year` 的 `int32` 值，若未设置则返回默认值。未显式设置默认值时，使用该类型的[零值](https://golang.org/ref/spec#The_zero_value)（数字为 0，字符串为空）。

其他标量类型（如 `bool`、`bytes`、`string`），`int32` 替换为对应 Go 类型，详见
[标量值类型表](./programming-guides/proto2#scalar)。

### 单一标量字段（proto3） {#singular-scalar-proto3}

如下字段定义：

```proto
int32 birth_year = 1;
optional int32 first_active_year = 2;
```

编译器生成如下访问器方法：

```go
func (m *Artist) GetBirthYear() int32 { ... }
func (m *Artist) SetBirthYear(v int32) { ... }
// 注意：没有 HasBirthYear() 或 ClearBirthYear() 方法；
// proto3 字段仅在声明为 optional 时才有 presence：
// /programming-guides/field_presence.md

func (m *Artist) GetFirstActiveYear() int32 { ... }
func (m *Artist) SetFirstActiveYear(v int32) { ... }
func (m *Artist) HasFirstActiveYear() bool { ... }
func (m *Artist) ClearFirstActiveYear() { ... }
```

`GetBirthYear()` 返回 `birth_year` 的 `int32` 值，若未设置则返回该类型的[零值](https://golang.org/ref/spec#The_zero_value)（数字为 0，字符串为空）。

其他标量类型同理，详见
[标量值类型表](./programming-guides/proto3#scalar)。
proto 未设置的值会以该类型的[零值](https://golang.org/ref/spec#The_zero_value)表示。

### 单一消息字段 {#singular-message}

给定如下消息类型：

```proto
message Band {}
```

消息中含有 `Band` 字段：

```proto
// proto2
message Concert {
  optional Band headliner = 1;
  // required 时生成代码相同。
}

// proto3
message Concert {
  Band headliner = 1;
}
```

编译器会生成如下访问器方法：

```go
type Concert struct { ... }

func (m *Concert) GetHeadliner() *Band { ... }
func (m *Concert) SetHeadliner(v *Band) { ... }
func (m *Concert) HasHeadliner() bool { ... }
func (m *Concert) ClearHeadliner() { ... }
```

`GetHeadliner()` 即使 `m` 为 nil 也可安全调用，可链式调用无需中间 nil 检查：

```go
var m *Concert // 默认为 nil
log.Infof("GetFoundingYear() = %d (不会 panic!)", m.GetHeadliner().GetFoundingYear())
```

字段未设置时，getter 返回默认值。消息类型默认值为 nil 指针。

setter 不会自动做 nil 检查，不能对可能为 nil 的消息安全调用 setter。

### 重复字段 {#repeated}

重复字段的访问器方法使用切片类型。例如：

```proto
message Concert {
  // 最佳实践：重复字段用复数名：
  // /programming-guides/style#repeated-fields
  repeated Band support_acts = 1;
}
```

编译器生成如下访问器方法：

```go
type Concert struct { ... }

func (m *Concert) GetSupportActs() []*Band { ... }
func (m *Concert) SetSupportActs(v []*Band) { ... }
```

如 `repeated bytes band_promo_images = 1;`，则生成 `[][]byte` 类型的访问器。重复 [枚举](#enum) 字段 `repeated MusicGenre genres = 2;`，生成 `[]MusicGenre` 类型的访问器。

构建 `Concert` 消息示例（使用 [builder](#builders)）：

```go
concert := Concert_builder{
  SupportActs: []*Band{
    {}, // 第一个元素
    {}, // 第二个元素
  },
}.Build()
```

也可用 setter：

```go
concert := &Concert{}
concert.SetSupportActs([]*Band{
    {}, // 第一个元素
    {}, // 第二个元素
})
```

访问字段：

```go
support := concert.GetSupportActs() // 类型为 []*Band
b1 := support[0] // 类型为 *Band，即 support_acts 的第一个元素
```

### Map 字段 {#map}

每个 map 字段生成类型为 `map[TKey]TValue` 的访问器，`TKey` 为键类型，`TValue` 为值类型。例如：

```proto
message MerchItem {}

message MerchBooth {
  // items 映射商品名到 MerchItem 消息
  map<string, MerchItem> items = 1;
}
```

编译器生成如下访问器方法：

```go
type MerchBooth struct { ... }

func (m *MerchBooth) GetItems() map[string]*MerchItem { ... }
func (m *MerchBooth) SetItems(v map[string]*MerchItem) { ... }
```

### Oneof 字段 {#oneof}

oneof 字段会为其中每个[单一字段](#singular-scalar-proto2)生成访问器。

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

编译器生成如下访问器方法：

```go
type Profile struct { ... }

func (m *Profile) WhichAvatar() case_Profile_Avatar { ... }
func (m *Profile) GetImageUrl() string { ... }
func (m *Profile) GetImageData() []byte { ... }

func (m *Profile) SetImageUrl(v string) { ... }
func (m *Profile) SetImageData(v []byte) { ... }

func (m *Profile) HasAvatar() bool { ... }
func (m *Profile) HasImageUrl() bool { ... }
func (m *Profile) HasImageData() bool { ... }

func (m *Profile) ClearAvatar() { ... }
func (m *Profile) ClearImageUrl() { ... }
func (m *Profile) ClearImageData() { ... }
```

使用 [builder](#builders) 设置字段示例：

```go
p1 := accountpb.Profile_builder{
  ImageUrl: proto.String("https://example.com/image.png"),
}.Build()
```

或用 setter：

```go
// imageData 为 []byte
imageData := getImageData()
p2 := &accountpb.Profile{}
p2.SetImageData(imageData)
```

访问字段时，可用 switch 语句判断 `WhichAvatar()` 结果：

```go
switch m.WhichAvatar() {
case accountpb.Profile_ImageUrl_case:
    // 用 m.GetImageUrl() 加载图片

case accountpb.Profile_ImageData_case:
    // 用 m.GetImageData() 加载图片

case accountpb.Profile_Avatar_not_set_case:
    // 字段未设置

default:
    return fmt.Errorf("Profile.Avatar 出现未知 oneof 字段 %v", x)
}
```

### Builder {#builders}

Builder 是在单个表达式中构建和初始化消息的便捷方式，尤其适用于嵌套消息和单元测试。

与其他语言（如 Java）不同，Go protobuf builder 不建议在函数间传递。应立即调用 `Build()` 并传递生成的 proto 消息，后续用 setter 修改字段。

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

编译器会生成类型和一组常量：

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

消息内的枚举类型名以消息名开头：

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

`Enum()` 方法分配新内存并返回指针：

```go
func (Genre) Enum() *Genre
```

编译器为每个枚举值生成常量。消息内枚举常量以消息名开头：

```go
const (
    Venue_KIND_UNSPECIFIED       Venue_Kind = 0
    Venue_KIND_CONCERT_HALL      Venue_Kind = 1
    Venue_KIND_STADIUM           Venue_Kind = 2
    Venue_KIND_BAR               Venue_Kind = 3
    Venue_KIND_OPEN_AIR_FESTIVAL Venue_Kind = 4
)
```

包级枚举常量以枚举名开头：

```go
const (
    Genre_GENRE_UNSPECIFIED   Genre = 0
    Genre_GENRE_ROCK          Genre = 1
    Genre_GENRE_INDIE         Genre = 2
    Genre_GENRE_DRUM_AND_BASS Genre = 3
)
```

编译器还会生成从整数到字符串名、从名到值的映射：

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

注意，`.proto` 语言允许多个枚举符号有相同数值，称为同义词。Go 中表现为多个名字对应同一数值。反向映射只包含第一个出现的名字。

## 扩展（proto2） {#extensions}

给定扩展定义：

```proto
extend Concert {
  optional int32 promo_id = 123;
}
```

编译器会生成
[`protoreflect.ExtensionType`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#ExtensionType)
值 `E_Promo_id`。可用
[`proto.GetExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#GetExtension)、
[`proto.SetExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#SetExtension)、
[`proto.HasExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#HasExtension)、
[`proto.ClearExtension`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#ClearExtension)
访问消息扩展。`GetExtension` 和 `SetExtension` 分别返回和接受包含扩展值类型的 `interface{}`。

单一标量扩展字段，扩展值类型为
[标量值类型表](./programming-guides/proto3#scalar)
中的 Go 类型。

单一嵌入消息扩展字段，扩展值类型为 `*M`，M 为消息类型。

重复扩展字段，扩展值类型为单一类型的切片。

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

此时，`ExtensionType` 值名为 `E_Promo_Concert`。

## 服务 {#service}

Go 代码生成器默认不为服务生成代码。如启用 [gRPC](https://www.grpc.io/) 插件（见
[gRPC Go 快速入门](https://github.com/grpc/grpc-go/tree/master/examples)），则会生成 gRPC 支持代码。

