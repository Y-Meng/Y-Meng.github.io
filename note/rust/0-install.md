* [教程](https://rustwiki.org/zh-CN/book/ch01-01-installation.html)

# linux || macos
```shell
# 打开终端输入如下命令
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```
这个命令将下载一个脚本并开始安装 rustup 工具，此工具将安装 Rust 的最新稳定版本。可能会提示你输入密码。如果安装成功，将出现下面这行：

> Rust is installed now. Great!

重启终端，从新加载环境变量验证是否安装成功。
```shell
rustup -V # rust版本管理工具
rustc --version # 编译器
cargo --version # 包管理器
```

# 更新和卸载
```shell
rustup update # 更新
rustup self uninstall # 卸载
```


# cargo常用命令
```shell
cargo new xxx # 创建新项目
cargo build #构建项目。
cargo run #一步构建并运行项目。
cargo check #构建项目而无需生成二进制文件来检查错误。
```

# vscode 开发环境
安装插件（从上至下重要性递减）：
* rust-analyzer：它会实时编译和分析你的 Rust 代码，提示代码中的错误，并对类型进行标注。你也可以使用官方的 rust 插件取代。
* rust syntax：为代码提供语法高亮。
* better toml：Rust 使用 toml 做项目的配置管理。
* crates 帮助你分析当前项目的依赖是否是最新的版本。
* rust test lens：可以帮你快速运行某个 Rust 测试。