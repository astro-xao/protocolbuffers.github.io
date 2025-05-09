+++
title = "Protocol Buffer 编译器安装"
weight = 15
description = "如何安装协议缓冲编译器。"
type = "docs"
no_list = "true"
linkTitle = "Protoc 安装"
+++

协议缓冲编译器 `protoc` 用于编译包含服务和消息定义的 `.proto` 文件。请选择以下方法之一安装 `protoc`。

### 安装预编译二进制文件（任意操作系统） {#binary-install}

要从预编译的二进制文件安装协议编译器的最新版本，请按照以下步骤操作：

1.  访问 https://github.com/google/protobuf/releases，手动下载与你的操作系统和计算机架构对应的 zip 文件（`protoc-<version>-<os>-<arch>.zip`），或使用如下命令获取文件：

    ```sh
    PB_REL="https://github.com/protocolbuffers/protobuf/releases"
    curl -LO $PB_REL/download/v30.2/protoc-30.2-linux-x86_64.zip

    ```

2.  将文件解压到 `$HOME/.local` 或你选择的目录。例如：

    ```sh
    unzip protoc-30.2-linux-x86_64.zip -d $HOME/.local
    ```

3.  更新你的环境变量 PATH，将 `protoc` 可执行文件所在路径加入。例如：

    ```sh
    export PATH="$PATH:$HOME/.local/bin"
    ```

### 使用包管理器安装 {#package-manager}

{{% alert title="警告" color="warning" %}} 使用包管理器安装后，请运行
`protoc --version` 检查 `protoc` 的版本，确保其足够新。某些包管理器安装的 `protoc` 版本可能较旧。请参阅
[版本支持页面](https://protobuf.dev/support/version-support) 对比你的版本号与所用语言支持的次版本号。{{% /alert %}}

你可以在 Linux、macOS 或 Windows 下使用包管理器安装协议编译器 `protoc`，命令如下。

-   Linux，使用 `apt` 或 `apt-get`，例如：

    ```sh
    apt install -y protobuf-compiler
    protoc --version  # 确保编译器版本为 3 及以上
    ```

-   MacOS，使用 [Homebrew](https://brew.sh)：

    ```sh
    brew install protobuf
    protoc --version  # 确保编译器版本为 3 及以上
    ```

-   Windows，使用
    [Winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/)

    ```sh
    > winget install protobuf
    > protoc --version # 确保编译器版本为 3 及以上
    ```

### 其他安装选项 {#other}

如果你希望从源码构建协议编译器，或访问旧版本的预编译二进制文件，请参阅
[下载 Protocol Buffers](https://protobuf.dev/downloads)。

