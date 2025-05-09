+++
title = "避免盲目跟风"
weight = 90
description = "避免在不需要的情况下使用功能。"
type = "docs"
+++

不要在 proto 文件中进行
[盲目跟风](https://en.wikipedia.org/wiki/Cargo_cult_programming)
设置。如果你是基于现有的 schema 定义创建新的 proto 文件，除非你理解某个 option 设置的必要性，否则不要随意应用。

## 针对 Editions 的最佳实践 {#editions}

除非确有必要，避免在 `.proto` 文件中应用
[editions 功能](/editions/features)。
这些功能通常表示使用了实验性的未来行为或已弃用的过去行为。最新 edition 的最佳实践始终为默认。新的 proto schema 定义内容应保持无功能设置，除非你希望提前采用某个即将推出的功能。

在不了解设置原因的情况下复制功能设置，可能会导致代码出现意外行为。
