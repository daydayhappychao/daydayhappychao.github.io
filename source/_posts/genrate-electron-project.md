---
title: 从零构建一个 electron 项目
date: 2023-07-03 09:46:12
tags: electron
---

本文将记录我是如何从零开始完成一个基于 electron 的桌面应用的初始化

# 一、初始化 npm 项目

electron 项目基于 node，所以在新建一个空目录后需要使用

```bash
npm init
```

将其初始化为一个 npm / node 项目进行管理。

# 二、实现 MVP

本章内容为让项目能以最小的依赖启动

参考文档内容: https://www.electronjs.org/zh/docs/latest/tutorial/tutorial-first-app

## 2.1 安装依赖

```
npm install electron
```

最小依赖只需要 electron 就足够启动一个桌面应用。

## 2.2 编写最基础的代码

### 2.2.1 渲染代码编写

渲染的内容基本等同于 web 那一套（可以简单理解为在桌面应用里内嵌了一个浏览器），所以可以把开发 web 的经验完全平移过来使用。

例如在根目录新建一个 index.html，内容如下:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta
      http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self'"
    />
    <meta
      http-equiv="X-Content-Security-Policy"
      content="default-src 'self'; script-src 'self'"
    />
    <title>Hello from Electron renderer!</title>
  </head>
  <body>
    <h1>Hello from Electron renderer!</h1>
    <p>👋</p>
  </body>
</html>
```

### 2.2.2 桌面应用逻辑代码编写

编写项目 package.json 中的 main 文件

> 例如 package.json 中的 main 字段为 index.js，则对应项目目录下的 index.js 文件

代码内容如下:

```javascript
const { app, BrowserWindow } = require("electron");

const createWindow = () => {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
  });

  win.loadFile("index.html");
};

app.whenReady().then(() => {
  createWindow();
});
```

内容为通过`app.whenReady`方法完成 electron 对桌面应用的初始化启动，之后通过`BrowserWindow`对象将刚刚编写的渲染代码连接到桌面应用中。

## 2.3 启动应用

在 package.json 中加入脚本

```json
"script": {
    "dev": "electron ."
}
```

执行 `npm run dev` 启动后即可看到 html 中的内容输出到了一个新启动的桌面应用上。

## 2.4 代码理解

上面编写的代码让我想到了用于测试和爬虫的无头浏览器的执行链路:

```
编写脚本 -> 使用该脚本唤起一个浏览器 -> 使用该脚本控制浏览器的行为
```

都是通过脚本(`main.js`)启动一个守护进程来控制浏览器的内容和行为的操作。

# 三、版本打包、发布与更新

桌面应用与 web 应用一个很大的区别就是它的版本更新不能依赖于 http 因而做不到完全自动化。根据之前对安卓、ios 应用的了解，桌面应用应该会有一个资源分发平台（自建或第三方），应用会在启动时请求这个资源分发平台以校验版本，再根据分发平台的能力对“版本更新”这个动作进行全量或增量的资源下载与替换。

## 3.1 工具的选择

工欲善其事必先利其器，在一个称不上十分熟悉的领域，直接使用现存的三方工具远比自己造轮子来的有性价比。  
electron 的构建发布工具目前主要有:

1. [electron-forge](https://www.electronforge.io/)
2. [electron-builder](https://www.electron.build/)

在 2017 年时，两者还有很大的功能上的分歧，具`electron-builder`开发者自己[描述](https://github.com/electron-userland/electron-builder/issues/1193), `electron-builder`是一个更专注于 electron 项目构建的脚本工具，而`electron-forge`更像是一个功能齐全的 ide。

但时过境迁，沧海桑田，就像`rollup`与`webpack`的殊途同归一样，在经历了这 6 年的风风雨雨后，两者在功能上已经很难再找出大的区别，基本你支持的我也都支持，你能做到我也能做到，无非就是谁用着顺手的事了。

所以就如 `redux`/`mobx`/`recoil`作为 react 的状态管理工具除了使用的方式有所区别，能实现的功能都是完全一致的。`electron-forge` 与 `electron-builder` 也是能完全平替的。

我先尝试了 `electron-builder`，但他因为内部依赖版本不对齐的版本导致 `electron-builder start` 无法运行，很难让人认为他是一个稳定的系统，所以我决定使用`electron-forge`进行开发。

## 3.2 为项目接入 electron-forge

## 3.2.1 使用 cli 接入

```bash
npm install --save-dev @electron-forge/cli
npx electron-forge import
```

## 3.2.2 使用新的命令启动项目

执行完上述命令后可以发现 package.json 中的 script 字段内容被替换成了 3 个基于 electron-forge 脚本的命令。
在命令行中使用

```bash
npm start
```

启动后得到和接入 electron-forget 一样的输出。

## 3.3 使用 electron-forge 进行打包

执行命令

```bash
npm run make
```

得到产物文件夹`out`，其中`out/make/xxx/xxx/xxx.zip`即可执行文件所在的路径。

## 3.4 使用 electron-forge 进行发布与推送

### 3.4.1 使用 electron-release-server 搭建更新服务器

在 [electron 文档](https://www.electronjs.org/zh/docs/latest/tutorial/updates#%E7%AC%AC%E4%B8%80%E6%AD%A5%E9%83%A8%E7%BD%B2%E6%9B%B4%E6%96%B0%E6%9C%8D%E5%8A%A1%E5%99%A8)中能看到官方推荐了几款可用作更新服务器的开源软件。综合观察后选择 `electron-release-server`。

**1、首先我们为 electron-release-server 项目构建镜像**

[electron-release-server 项目](https://github.com/ArekSredzki/electron-release-server/tree/master)中自带了 Dockerfile，稍加修改后（主要是针对网络伟大墙）即可构建出我们需要的镜像。

> 注意: 该容器的启动依赖 PostgreSQL，配置内容看 `config/docker.js`

**2、之后我们通过 electron-release-server 提供的站点进行操作**

访问启动端口对应的 web 服务即可访问 `electron-release-server`。

我们修改`forge.config.js`中的 makers 字段新增内容:

```javascript
{
    name: "@electron-forge/maker-dmg",
    config: {
    background: "./assets/dmg-background.png",
    format: "ULFO",
    },
}
```

再执行

```bash
npm run make
```

可以看到在 `out/make` 目录下生成了 `.dmg` 的资源。将该文件上传到 `electron-release-server` 的管理后台并指定操作系统为 OSX 后即可从`electron-release-server` 主页下载系统对应版本的应用。

> 我通过下载安装的应用打开时会报错: "应用已损坏 xxxx" 的报错，解决方法是: https://zhuanlan.zhihu.com/p/99489849

**3、最后我们需要接入更新能力**

首先从`electron-release-server`文档可以看到更新的 url 是:

1. /update/:platform/:version[/:channel]
2. /update/flavor/:flavor/:platform/:version[/:channel]

根据 [electron 的文档](https://www.electronjs.org/zh/docs/latest/tutorial/updates#%E4%BD%BF%E7%94%A8%E5%85%B6%E4%BB%96%E6%9B%B4%E6%96%B0%E6%9C%8D%E5%8A%A1)提供的自动更新的方案，使用 electron 的 autoUpdater api 进行检查、下载、重启并更新。

在客户端代码中新建一个`update.ts`内容如下(此时我已经把项目从 js 迁移到了 ts，这块比较简单且与 electron 无关不做展开):

```typescript
import {
  app,
  BrowserWindow,
  ipcMain,
  dialog,
  autoUpdater,
  MessageBoxOptions,
} from "electron";

export class UpdateManager {
  private static instance: UpdateManager = new UpdateManager();

  public static getInstance() {
    return this.instance;
  }

  baseUrl = "http://localhost:8080";
  flavorName = "wangchao-test";
  mainWindow: BrowserWindow | undefined;

  sendUpdateMessage(text: any) {
    this.mainWindow?.webContents.send("update-message", text);
  }

  init(window: BrowserWindow) {
    console.log("ipcMain-update,init");
    this.mainWindow = window;
    let platform = "";
    let arch = "";

    if (process.platform === "darwin") {
      platform = "osx";
      if (process.arch === "arm64") {
        arch = "arm64";
      } else {
        arch = "64";
      }
    } else if (process.platform === "win32") {
      platform = "windows";
      if (process.arch === "x64") {
        arch = "64";
      } else {
        arch = "32";
      }
    } else if (process.platform === "linux") {
      platform = "linux";
      if (process.arch === "x64" || process.arch === "ppc64") {
        arch = "64";
      } else {
        arch = "32";
      }
    }

    const feedUrl =
      this.baseUrl +
      "/update/flavor/" +
      this.flavorName +
      "/" +
      platform +
      "_" +
      arch +
      "/" +
      app.getVersion() +
      "/stable";
    autoUpdater.setFeedURL({ url: feedUrl });

    autoUpdater.on("error", (error) => {
      console.log("ipcMain-update,error:" + error);
      this.sendUpdateMessage({ cmd: "error", message: error });
    });

    autoUpdater.on("update-not-available", () => {
      console.log("ipcMain-update,update-not-available");
      this.sendUpdateMessage({
        cmd: "update-not-available",
        message: "has not found any updates",
      });
    });

    autoUpdater.on("update-available", () => {
      console.log("ipcMain-update,update-available");
      this.sendUpdateMessage({
        cmd: "update-available",
        message: "find new version",
      });
    });

    autoUpdater.on("update-downloaded", (event, releaseNotes, releaseName) => {
      console.log("ipcMain-update,update-downloaded:" + event);
      this.sendUpdateMessage({
        cmd: "update-downloaded",
        message: {
          releaseNotes: releaseNotes,
          releaseName: releaseName,
        },
      });

      const dialogOpts: MessageBoxOptions = {
        type: "info",
        buttons: ["确定", "稍后"],
        title: "版本更新",
        message: "发现新版本:" + releaseName,
        detail: "新版本已经下载完成，是否现在重启?",
      };
      dialog.showMessageBox(dialogOpts).then((returnValue) => {
        if (returnValue.response === 0) {
          autoUpdater.quitAndInstall();
        }
      });
    });

    ipcMain.on("checkForUpdate", (event, args) => {
      console.log("ipcMain-update,checkForUpdate");
      autoUpdater.checkForUpdates();
    });
  }
}
```

上述代码的核心是计算`feedUrl`部分，其他都与 electron 文档的内容一致。

并在入口文件（2.2.2 中编写的 js）接入`update.ts`的逻辑:

```typescript
import { app, BrowserWindow } from "electron";
import { UpdateManager } from "./update";

const createWindow = () => {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
  });

  win.loadFile("index.html");

  // 这里开始是新增的
  if (app.isPackaged) {
    UpdateManager.getInstance().init(win);
  }
  // 新增结束
};

app.whenReady().then(() => {
  createWindow();
});
```

再通过 `preload.ts` 接通 electron api 与前端

```typescript
import { contextBridge, ipcRenderer } from "electron";

contextBridge.exposeInMainWorld("electronAPI", {
  logUpdateMsg: () => {
    ipcRenderer.on("update-message", function (event, args) {
      console.log(
        "ipcRenderer,update-message,message:" + JSON.stringify(args.message)
      );
      if ("update-not-available" === args.cmd) {
        console.log("ipcRenderer,update-not-available");
      } else if ("update-available" === args.cmd) {
        console.log("ipcRenderer,update-available");
      } else if ("update-downloaded" === args.cmd) {
        console.log("ipcRenderer,update-downloaded");
      } else if ("error" === args.cmd) {
        console.log("ipcRenderer,error");
      }
    });
  },
  checkUpdate: () => {
    ipcRenderer.send("checkForUpdate");
  },
});

```

之后在前端页面合适的地方调用`window.electronAPI.checkUpdate()`即可完成更新。

> 注意，自动更新能力在 mac 上是需要付费注册成开发者获得证书后才能正常使用，所以目前无法在 mac 上正常使用。

此时我们的代码层面已经支持了配合`electron-release-server`的更新服务，接下来我们需要把包含更新能力的代码同步到`electron-release-server`上并下载，再进行新版本的上传即可完成整套更新的流程。

> electron-forge 提供了配套 electron-release-server 的发布插件 [@electron-forge/publisher-electron-release-server](https://www.electronforge.io/config/publishers/electron-release-server)可以通过 `electron-forge publish` 命令即可完成发布。


# 四、业务代码构建

至此我们已经完成了最基础的设施的建设，一个 electron 项目已经能顺利进行开发、发布、下载、更新。下一步就可以进入到业务的开发了。业务本身的复杂度导致前面开发时使用的这种一个文件的方式需要被拓展成多文件、多包引入的开发形式，所以为项目配置打包方式也将成为这个项目基础设施的一部分。  

先简单分析一下项目的情景，目前项目中的代码根据运行环境可以分成两部分。  

第一部分是在 node runtime 里运行的以 `index.ts` 为入口的文件，包括 `index.ts`、`preload.ts`、`update.ts`。

第二部分实在 chromium runtime 里运行的以 `index.html` 为入口的文件，包括 `index.html` 和它引用的资源文件（js、css）。

这两者的打包应该是分开的，前者更像是我们平时写 node 应用时的打包方案，后者更像是我们写前端应用的打包方案。

所以我们将两份代码存储到两个不同的文件夹以进行区分，在项目根目录建立文件夹 `src/client`与 `src/server`，其中 `src/client` 用于存放 chromium runtime 代码，`src/server` 用于存放 node runtime 代码。

## 4.1 Node Runtime 代码构建

Node Runtime 代码的构建更适合 bundless 的模式，所以我们使用 def 进行构建。

1. 在 package.json 中声明 `rainaConfig.type` 为 `npm`（因为目前 type 字段只支持 `page` 与 `npm`）
2. 新建 rainaConfig.js, 写入内容:
```javascript
module.exports = {
  build: {
    bundleless: {
      enable: true,
      sourceDir: "src/server",
      outputDir: "dist/server",
    },
  },
};

```
3. 执行命令 `raina dev`，即可看到在 `dist/server` 目录下生成了对应的编译后的文件


