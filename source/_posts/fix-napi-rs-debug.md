---
title: napi-rs 应用的调试问题解决
date: 2023-07-28 15:34:30
tags:
---

这几天想通过 rspack 项目来学习一下 rust 应用的开发。没想到在起步阶段就踩了个大坑。

开发环境

1. 系统：osx m1
2. 编辑器：vscode（插件：rust-analyzer、codeLLDB）
3. 预置环境

   1. node 和 npm 和 napi
   2. rust 和 cargo 和 rust-analyzer
   3. lldb

## 一、rspack 项目的 debug 失败

根据 rspack 项目提供的 contribute.md 中的流程，完成环境配置和项目的启动后，

在`crates/node_binding/src/lib.rs`​ 的 Rspack 中的 new 方法打上 vscode 断点，

再打开 vscode 的 debug 栏，执行 debug-rspack-2 选项。

结果 `${workspaceFolder}/examples/angular`​ 目录成功完成了编译产出了 dist 目录，却没有进入我们的断点。

很明显，该表现可以被描述为：程序能正常运行，但断点无法正常执行

## 二、建立最小可复现项目

rspack 项目依赖太多，内容太庞杂，可能产生影响的噪音元素也随之更多。已知 rspack 是通过 @napi/rs 编译为 .node 文件而被 js 方调用，所以我们首先要验证一个最简单的基于 @napi/rs 的项目能否正常 debug。

首先使用 `napi`​ 命令初始化一个项目 `napi new`​

> 什么？你还没有 napi 命令？先使用 `npm i @napi-rs/cli -g ​`​ 装一个吧，[官方文档](https://napi.rs/docs/introduction/getting-started)

在项目内根目录新建文件 `test.js`​，内容主要就是调一下`src/lib.rs`​ 里的方法即可。

例如 `lib.rs`​ 内容是

```rust
#![deny(clippy::all)]

use napi_derive::napi;

#[napi]
pub fn sum(a: i32, b: i32) -> i32 {
  a + b
}
```

那么`test.js`​ 就是

```js
import { sum } from "./index.js";

console.log(sum(1, 2));
```

然后编写 debug 配置。

新建文件 `.vscode/tasks.json`​ 内容如下：

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "script": "build:debug",
      "group": "build",
      "problemMatcher": [],
      "label": "npm build:debug"
    }
  ]
}
```

新建文件 `.vscode/launch.json`​ 内容如下：

```json
{
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "sourceLanguages": ["rust"],
      "name": "debug rust",
      "program": "node",
      "preLaunchTask": "npm build:debug",
      "args": ["--inspect", "${file}"]
    }
  ]
}
```

之后在 `src/lib.rs`​ 的 sum 方法中打上断点。然后将当前窗口打开到`test.js`​ 后执行 debug 的 `debug rust`​ 选项。

没一会，控制台里输出了我们期待的结果 `3`​，却没有顺利进入断点。

## 三、尝试用 lldb 直接调试

为了验证是工具的问题，还是 vscode 的问题。我尝试跳过 vscode 使用命令行来进行 debug。已知 vscode 是使用 lldb 进行 debug 的，所以我在命令行里使用 lldb 进行 debug 应该原理是一样的。

lldb 启动 debug 的方式：`lldb -- node test.mjs`​

可惜查了很多资料，也没弄明白在 lldb 里怎么为 rs 文件打上断点。但起码 lldb 能正常运行，所以在 lldb 里出问题的可能性就很小了。

## 四、检查 rust-analyzer

回想 vscode 执行 debug 的工具链：

vscode -> rust-analyzer -> lldb

既然 lldb 是好的，那下一步就应该验证一下 rust-analyzer 是否出了问题。验证方法也比较简单，我是对 lib.rs 执行 vscode 的命令: `rust-analyzer: Expand macro recursively`​ 发现返回内容是 `not available`​ 发现 rust-analyzer 可能不对劲。

你也可以直接在命令行里执行 `rust-analyzer --help`​ 结果我这里直接报错说 rust-analyzer 版本和我的系统不匹配（我是 m1）。

于是我通过官网重新安装了 rust-analyzer。（上一个有问题的版本的 rust-analyzer 好像是通过 vscode 的插件安装进来的）

重启了 vscode 后再次验证 debug，结果依然失败。

## 五、山重水复疑无路

之后这个问题继续困扰了我一下午。而在今天忙完手上的工作后，再试一下发现依然不行时，我的脑洞突然开了一下，想，重启一下试试，万一有用呢？

嘿，没想到，结果重启了一下还真就行了。

之前我一直对“重启试试”是不抱有好感的，结果今天才明白，老一辈人诚不欺我啊。

## 六、总结

所以针对 napi/rs 在 vscode 中 debug 的问题，如果你按照教程安装了依赖、插件，声明了 debug 配置，启动时仍然遇到无法 debug 的问题，建议检查一下 lldb 与 rust-analyzer 的版本是否匹配你的系统，如果不匹配，记得重新安装后重启下电脑喔！
