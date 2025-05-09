+++
title = "Go Opaque API：手动迁移"
weight = 660
linkTitle = "Opaque API：手动迁移"
description = "介绍如何手动迁移到 Opaque API。"
type = "docs"
+++

Opaque API 是 Protocol Buffers 在 Go 语言中的最新实现。旧版本现在称为 Open Struct API。请参阅 [Go Protobuf: Releasing the Opaque API](https://go.dev/blog/protobuf-opaque) 博客文章以了解简介。

本文档为将 Go Protobuf 用法从旧的 Open Struct API 迁移到新的 Opaque API 的用户指南。

{{% alert title="警告" color="warning" %}} 您正在查看手动迁移指南。通常建议使用 `open2opaque` 工具自动迁移。请参阅 [Opaque API 迁移](./reference/go/opaque-migration) 获取更多信息。{{% /alert %}}

[生成代码指南](./reference/go/go-generated-opaque) 提供了更多细节。本文将新旧 API 进行对比。

### 消息构造

假设有如下 protobuf 消息定义：

```proto
message Foo {
  uint32 uint32 = 1;
  bytes bytes = 2;
  oneof union {
    string    string = 4;
    MyMessage message = 5;
  }
  enum Kind { … };
  Kind kind = 9;
}
```

以下是如何从字面值构造该消息的示例：

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
m := &pb.Foo{
  Uint32: proto.Uint32(5),
  Bytes:  []byte("hello"),
}
```

</td>
<td>

```go
m := pb.Foo_builder{
  Uint32: proto.Uint32(5),
  Bytes:  []byte("hello"),
}.Build()
```

</td>
</tr>
</table>

可以看到，builder 结构体允许 Open Struct API（旧）与 Opaque API（新）之间几乎 1:1 的转换。

通常建议使用 builder 以提高可读性。仅在极少数情况下（如在高频循环中创建 Protobuf 消息）才建议使用 setter。详情请参阅 [Opaque API 常见问题：应使用 builder 还是 setter？](./reference/go/opaque-faq#builders-vs-setters)。

对于 [oneof](#oneofs) 字段有例外：Open Struct API（旧）为每个 oneof case 使用包装结构体类型，而 Opaque API（新）将 oneof 字段视为普通消息字段：

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
m := &pb.Foo{
  Uint32: myScalar,  // 可能为 nil
  Union:  &pb.Foo_String{myString},
  Kind:   pb.Foo_SPECIAL_KIND.Enum(),
}
```

</td>
<td>

```go
m := pb.Foo_builder{
  Uint32: myScalar,
  String: myString,
  Kind:   pb.Foo_SPECIAL_KIND.Enum(),
}.Build()
```

</td>
</tr>
</table>

对于 oneof union 相关的 Go 结构体字段，只能有一个字段被赋值。如果多个 oneof case 字段被赋值，则以 .proto 文件中字段声明顺序的最后一个为准。

### 标量字段

假设有如下带标量字段的消息定义：

```proto
message Artist {
  int32 birth_year = 1;
}
```

对于 Go 使用标量类型（bool、int32、int64、uint32、uint64、float32、float64、string、[]byte 和 enum）的 Protobuf 字段，会生成 `Get` 和 `Set` 访问器方法。具有 [显式存在性](./programming-guides/field_presence/) 的字段还会有 `Has` 和 `Clear` 方法。

对于名为 `birth_year` 的 int32 字段，将生成如下访问器方法：

```go
func (m *Artist) GetBirthYear() int32
func (m *Artist) SetBirthYear(v int32)
func (m *Artist) HasBirthYear() bool
func (m *Artist) ClearBirthYear()
```

`Get` 返回字段的值。如果字段未设置或消息接收者为 nil，则返回默认值。默认值为 [零值](https://go.dev/ref/spec#The_zero_value)，除非通过 default 选项显式设置。

`Set` 将提供的值存储到字段中。若在 nil 消息接收者上调用会 panic。

对于 bytes 字段，使用 nil []byte 调用 `Set` 也会被视为已设置。例如，随后调用 `Has` 会返回 true。随后调用 `Get` 会返回零长度切片（可能为 nil 或空切片）。用户应使用 `Has` 判断存在性，而不是依赖 `Get` 是否返回 nil。

`Has` 报告字段是否已赋值。在 nil 消息接收者上调用返回 false。

`Clear` 清除字段。在 nil 消息接收者上调用会 panic。

**字符串字段示例代码：**

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
// 获取字段值
s := m.GetBirthYear()

// 设置字段
m.BirthYear = proto.Int32(1989)

// 检查是否存在
if s.BirthYear != nil { … }

// 清除字段
m.BirthYear = nil
```

</td>
<td>

```go
// 获取字段值
s := m.GetBirthYear()

// 设置字段
m.SetBirthYear(1989)

// 检查是否存在
if m.HasBirthYear() { … }

// 清除字段
m.ClearBirthYear()
```

</td>
</tr>
</table>

### 消息字段

假设有如下带消息类型字段的消息定义：

```proto
message Band {}

message Concert {
  Band headliner = 1;
}
```

消息类型字段会生成 `Get`、`Set`、`Has` 和 `Clear` 方法。

对于名为 `headliner` 的消息类型字段，将生成如下访问器方法：

```go
func (m *Concert) GetHeadliner() *Band
func (m *Concert) SetHeadliner(*Band)
func (m *Concert) HasHeadliner() bool
func (m *Concert) ClearHeadliner()
```

`Get` 返回字段的值。如果未设置或在 nil 消息接收者上调用则返回 nil。判断 `Get` 是否为 nil 等价于判断 `Has` 是否为 false。

`Set` 将提供的值存储到字段中。在 nil 消息接收者上调用会 panic。用 nil 指针调用 `Set` 等价于调用 `Clear`。

`Has` 报告字段是否已赋值。在 nil 消息接收者上调用返回 false。

`Clear` 清除字段。在 nil 消息接收者上调用会 panic。

**示例代码：**

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque（新）</td>
        </tr>
<tr>
<td>

```go
// 获取字段值
b := m.GetHeadliner()

// 设置字段
m.Headliner = &pb.Band{}

// 检查是否存在
if s.Headliner != nil { … }

// 清除字段
m.Headliner = nil
```

</td>
<td>

```go
// 获取字段值
s := m.GetHeadliner()

// 设置字段
m.SetHeadliner(&pb.Band{})

// 检查是否存在
if m.HasHeadliner() { … }

// 清除字段
m.ClearHeadliner()
```

</td>
</tr>
</table>

### 重复字段

假设有如下带重复消息类型字段的消息定义：

```proto
message Concert {
  repeated Band support_acts = 2;
}
```

重复字段会生成 `Get` 和 `Set` 方法。

`Get` 返回字段的值。如果字段未设置或消息接收者为 nil，则返回 nil。

`Set` 将提供的值存储到字段中。在 nil 消息接收者上调用会 panic。`Set` 会存储传入切片头的副本。对切片内容的更改会反映到重复字段中。因此，如果用空切片调用 `Set`，随后调用 `Get` 会返回同一个切片。对于 wire 或文本序列化输出，传入的 nil 切片与空切片无区别。

对于消息 `Concert` 上名为 `support_acts` 的重复消息类型字段，将生成如下访问器方法：

```go
func (m *Concert) GetSupportActs() []*Band
func (m *Concert) SetSupportActs([]*Band)
```

**示例代码：**

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
// 获取整个重复字段
v := m.GetSupportActs()

// 设置字段
m.SupportActs = v

// 获取某个元素
e := m.SupportActs[i]

// 设置某个元素
m.SupportActs[i] = e

// 获取长度
n := len(m.GetSupportActs())

// 截断
m.SupportActs = m.SupportActs[:i]

// 追加
m.SupportActs = append(m.GetSupportActs(), e)
m.SupportActs = append(m.GetSupportActs(), v...)

// 清除字段
m.SupportActs = nil
```

</td>
<td>

```go
// 获取整个重复字段
v := m.GetSupportActs()

// 设置字段
m.SetSupportActs(v)

// 获取某个元素
e := m.GetSupportActs()[i]

// 设置某个元素
m.GetSupportActs()[i] = e

// 获取长度
n := len(m.GetSupportActs())

// 截断
m.SetSupportActs(m.GetSupportActs()[:i])

// 追加
m.SetSupportActs(append(m.GetSupportActs(), e))
m.SetSupportActs(append(m.GetSupportActs(), v...))

// 清除字段
m.SetSupportActs(nil)
```

</td>
</tr>
</table>

### Map 字段

假设有如下带 map 类型字段的消息定义：

```proto
message MerchBooth {
  map<string, MerchItems> items = 1;
}
```

Map 字段会生成 `Get` 和 `Set` 方法。

`Get` 返回字段的值。如果字段未设置或消息接收者为 nil，则返回 nil。

`Set` 将提供的值存储到字段中。在 nil 消息接收者上调用会 panic。`Set` 会存储传入 map 引用的副本。对传入 map 的更改会反映到字段中。

对于消息 `MerchBooth` 上名为 `items` 的 map 字段，将生成如下访问器方法：

```go
func (m *MerchBooth) GetItems() map[string]*MerchItem
func (m *MerchBooth) SetItems(map[string]*MerchItem)
```

**示例代码：**

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
// 获取整个 map
v := m.GetItems()

// 设置字段
m.Items = v

// 获取某个元素
v := m.Items[k]

// 设置某个元素
// 若 m.Items 为 nil 会 panic
// 应先检查 m.Items 是否为 nil
m.Items[k] = v

// 删除元素
delete(m.Items, k)

// 获取 map 长度
n := len(m.GetItems())

// 清除字段
m.Items = nil
```

</td>
<td>

```go
// 获取整个 map
v := m.GetItems()

// 设置字段
m.SetItems(v)

// 获取某个元素
v := m.GetItems()[k]

// 设置某个元素
// 若 m.GetItems() 为 nil 会 panic
// 应先检查 m.GetItems() 是否为 nil
m.GetItems()[k] = v

// 删除元素
delete(m.GetItems(), k)

// 获取 map 长度
n := len(m.GetItems())

// 清除字段
m.SetItems(nil)
```

</td>
</tr>
</table>

### Oneof 字段

对于每个 oneof union 分组，消息上会有 `Which`、`Has` 和 `Clear` 方法。每个 oneof case 字段也会有 `Get`、`Set`、`Has` 和 `Clear` 方法。

假设有如下 oneof 字段 `image_url` 和 `image_data`，位于 oneof `avatar` 中：

```proto
message Profile {
  oneof avatar {
    string image_url = 1;
    bytes image_data = 2;
  }
}
```

该 oneof 的 Opaque API 生成如下：

```go
func (m *Profile) WhichAvatar() case_Profile_Avatar { … }
func (m *Profile) HasAvatar() bool { … }
func (m *Profile) ClearAvatar() { … }

type case_Profile_Avatar protoreflect.FieldNumber

const (
  Profile_Avatar_not_set_case case_Profile_Avatar = 0
  Profile_ImageUrl_case case_Profile_Avatar = 1
  Profile_ImageData_case case_Profile_Avatar = 2
)
```

`Which` 返回已设置的 case 字段编号。若未设置或在 nil 消息接收者上调用则返回 0。

`Has` 报告 oneof 内是否有字段被设置。在 nil 消息接收者上调用返回 false。

`Clear` 清除当前已设置的 oneof 字段。在 nil 消息接收者上调用会 panic。

每个 oneof case 字段的 Opaque API 生成如下：

```go
func (m *Profile) GetImageUrl() string { … }
func (m *Profile) GetImageData() []byte { … }

func (m *Profile) SetImageUrl(v string) { … }
func (m *Profile) SetImageData(v []byte) { … }

func (m *Profile) HasImageUrl() bool { … }
func (m *Profile) HasImageData() bool { … }

func (m *Profile) ClearImageUrl() { … }
func (m *Profile) ClearImageData() { … }
```

`Get` 返回 case 字段的值。若 case 字段未设置或在 nil 消息接收者上调用则返回零值。

`Set` 将提供的值存储到 case 字段，并隐式清除 oneof union 中之前已赋值的 case 字段。对 oneof 消息类型 case 字段用 nil 值调用 `Set` 会设置为空消息。在 nil 消息接收者上调用会 panic。

`Has` 报告 case 字段是否被设置。在 nil 消息接收者上调用返回 false。

`Clear` 清除 case 字段。如果之前已设置，则 oneof union 也会被清除。如果 oneof union 已设置为其他字段，则不会清除。在 nil 消息接收者上调用会 panic。

**示例代码：**

<table width="100%">
        <tr>
                <td width="50%">Open Struct API（旧）</td>
                <td width="50%">Opaque API（新）</td>
        </tr>
<tr>
<td>

```go
// 获取已设置的 oneof 字段
switch m.GetAvatar().(type) {
case *pb.Profile_ImageUrl:
  … = m.GetImageUrl()
case *pb.Profile_ImageData:
  … = m.GetImageData()
}

// 设置字段
m.Avatar = &pb.Profile_ImageUrl{"http://"}
m.Avatar = &pb.Profile_ImageData{img}

// 检查是否有 oneof 字段被设置
if m.Avatar != nil { … }

// 清除字段
m.Avatar = nil

// 检查特定字段是否被设置
_, ok := m.GetAvatar().(*pb.Profile_ImageUrl)
if ok { … }

// 清除特定字段
_, ok := m.GetAvatar().(*pb.Profile_ImageUrl)
if ok {
  m.Avatar = nil
}

// 拷贝 oneof 字段
m.Avatar = src.Avatar
```

</td>
<td>

```go
// 获取已设置的 oneof 字段
switch m.WhichAvatar() {
case pb.Profile_ImageUrl_case:
  … = m.GetImageUrl()
case pb.Profile_ImageData_case:
  … = m.GetImageData()
}

// 设置字段
m.SetImageUrl("http://")
m.SetImageData([]byte("…"))

// 检查是否有 oneof 字段被设置
if m.HasAvatar() { … }

// 清除字段
m.ClearAvatar()

// 检查特定字段是否被设置
if m.HasImageUrl() { … }

// 清除特定字段
m.ClearImageUrl()

// 拷贝 oneof 字段
switch src.WhichAvatar() {
case pb.Profile_ImageUrl_case:
  m.SetImageUrl(src.GetImageUrl())
case pb.Profile_ImageData_case:
  m.SetImageData(src.GetImageData())
}
```

</td>
</tr>

</table>

### 反射

在 proto 消息类型上使用 Go 的 `reflect` 包访问结构体字段和标签的代码，在迁移离 Open Struct API 后将不再可用。代码需要迁移到 [protoreflect](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect)。

一些常见库在底层使用 Go `reflect`，例如：

*   [encoding/json](https://pkg.go.dev/encoding/json/)
    *   请使用 [protobuf/encoding/protojson](https://pkg.go.dev/google.golang.org/protobuf/encoding/protojson)。
*   [pretty](https://pkg.go.dev/github.com/kr/pretty)
*   [cmp](https://pkg.go.dev/github.com/google/go-cmp/cmp)
    *   若要正确使用 `cmp.Equal` 比较 protobuf 消息，请使用 [protocmp.Transform](https://pkg.go.dev/google.golang.org/protobuf/testing/protocmp#Transform)
