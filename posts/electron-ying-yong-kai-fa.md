---
title: 'Electron 应用开发'
date: 2024-12-08 09:00:31
tags: [electron,note,app,fily]
published: true
hideInList: false
feature: /post-images/electron-ying-yong-kai-fa.png
isTop: false
---
## 一、了解Electron

Electron 是由 Github 开发，用 HTML，CSS 和 JavaScript 来构建跨平台桌面应用程序的一个开源库。最初的时候是属于 Github 开源代码编辑器 Atom 的一部分，在 2014 春季这两个项目相继开源。

<!-- more -->

**里程碑事件**

| 时间         | 里程碑                                         |
| ------------ | ---------------------------------------------- |
| 2013 年 4 月 | Atom Shell 项目启动 。                         |
| 2014 年 5 月 | Atom Shell 被开源 。                           |
| 2015 年 4 月 | Atom Shell 被重命名为 Electron 。              |
| 2016 年 5 月 | Electron 发布了 v1.0.0 版本 。                 |
| 2016 年 5 月 | Electron 构建的应用程序可上架 Mac App Store 。 |
| 2016 年 8 月 | Windows Store 支持 Electron 构建的应用程序 。  |

**Electron优点**：

+ 跨平台，降低开发成本。
+ 强大的 npm 生态，提升开发效率
+ 简单易学的 JavaScript 语言，基于 Web 的桌面开发，不用再去学习 C# 或者 Swift，降低学习成本。
+ 苹果商店和微软商店都支持提交 Electron 应用程序。
+ 开发速度快，适合需要段时间内看到效果的应用开发

**缺点**：

+ 打包体积大
+ Chromium比较吃内存
+ 跨平台并不代表一次编写，处处运行，需要在各个系统上测试、打包
+ 版本更新频繁，多个版本间有兼容性的问题

**跨平台开发框架比较**
| 框架                  | 支持语言             | 优缺点                                                       |
| --------------------- | -------------------- | ------------------------------------------------------------ |
| Qt                    | C++/Python           | 优点：流行、稳定，内置自绘引擎，保证不同系统上界面一致，PyQt可以使用Python开发<br/>缺点：高分屏缩放显示问题，商业需授权，免费版本版权问题 |
| GTK                   | C/C++/JS/Rust等      | 优点：自绘引擎，提供大量系统相关API，商业授权友好<br/>缺点：只在Linux环境下流行，其他环境下开发困难 |
| wxWidgets             | C++                  | 优点：稳定、成熟<br/>缺点：没有自绘引擎，界面风格不统一      |
| FLTK                  | C++                  | 优点：轻量，OpenGL引擎，有Rust绑定fltk-rs<br/>缺点：绘图API少 |
| Duilib                | C++                  | 优点：开源、有不少大厂使用<br/>缺点：只支持Windows、不支持高分屏、几乎没有系统级API，更新不及时，开发环境搭建麻烦 |
| Sciter                | C++/Rust/Go/Python等 | 优点：精简，个人使用免费，内部有一个浏览器核心，自研脚本类似JS<br/>缺点：闭源，商业收费，不支持Flex布局，非官方绑定质量差，社区不成熟 |
| CEF                   | C/C++/Go/Python/Java | 优点：基于Chromium，装机量大，支持HTML/CSS/JS特性<br/>缺点：体积大、几乎不提供系统及API，版本发布频繁 |
| MAUI                  | C#                   | 优点：微软开发，有自绘引擎，支持桌面端和移动端<br/>缺点：暂无稳定版，界面有XAML描述 |
| Compose Multiplatform | Kotlin               | 优点：新，自绘引擎Skia（Goole）<br/>缺点：稳定欠缺，打包需要封装JRE |
| flutter-desktop       | Dart                 | 优点：新，自绘引擎Skia，支持FFI<br/>缺点：不稳定，Dart大括号嵌套不友好，打包体积大 |
| webview2              | C#/C++               | 优点：可以复用系统当中已存在的webview2二进制资源，打包体积小<br/>缺点：只支持Windows，闭源 |
| TAURI                 | Rust                 | 优点：安装包小，新，开源，最近比较火<br/>缺点：新，不稳定，webview有的问题TARI都有 |
| NW.js（node-webkit）  | JS                   | 优点：开源，开发方便<br/>缺点：封装的系统级API不如Electron，生态不完善，API不稳定 |
| Electron              | JS                   | 优点：作者是NW.js原成员，Github，微软等大厂在用（github客户端，VSCode），生态完善，系统级API多<br/>缺点：打包体积大 |
| ImGui                 | C++                  | 优点：对游戏开发者友好，支持多种引擎比如OpenGL，Directx，Vulkan等，打包体积小<br/>缺点：循环绘制整个界面，消耗CPU，有一些小问题 |

如果是会C/C++，可选的框架就很多，可以根据需求选择最合适的框架，但是如果会一点JS，这里的Electron可能是比较合适的。首先是开源，开发语言JS相对大众，遇到问题也比较容易找到解决方案；其次是Electron的生态完善，有比较多的大厂在使用，比如微软的VS Code，Github官方的客户端GUI，Slack，Postman等，国内也有很多使用案例，比如迅雷X，utools等，更多的Electron 应用可以在 [🔗这里](https://www.electronjs.org/apps) 发现。

![Electron - www.electronjs.org](https://xuhang.github.io/post-images/1733619671523.png)

## 二、使用Electron开发应用
### 问题
在项目开发中，大部分时间是使用QQ作为沟通工具，那项目相关的需求文档、接口文档也基本上是通过QQ分享，时间久了在QQ的本地目录里就会有很多不同项目的相关文档，要找到就会很困难，而且很多文档名仅差几个字，很难区分每个文档属于哪个项目下的。

### 方案1
每次涉及新的需求，在用户目录下新建一个该项目的文件夹，每次有收到项目相关的文件就将文档存放到该路径下，再次使用时就在该目录下查找。

![](https://xuhang.github.io/post-images/1733619772394.png)

### 方案2
方案1虽然能将文件分项目存储，但是每个文件的内容很难通过文件获取；每次进入项目目录需要打开资源管理器，进入用户目录，选择相关项目，步骤繁琐；……这里我们开发一个专门解决这个痛点的软件，用来管理项目相关的文件，该软件有以下功能：

+ [x] 分项目管理所属文件
+ [x] 以易读的方式显示文件的加入时间
+ [x] 支持直接从聊天软件对话框中拖拽文件到项目中
+ [ ] 如果其他人需要，可以将项目所有文件打包分享
+ [x] 可以对每个文件注释，标注该文件的作用、内容等
+ [x] 快速定位到文件和打开文件
+ [ ] 针对每个项目内嵌一个Kanban来记录具体任务的完成情况
+ [ ]  根据不同的命名规则自动关联到文件的最新版本
+ [x] 加入沟通功能，可以快速与小组成员分享文件


| Windows UI                                                   | MacOS UI                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![](https://xuhang.github.io/post-images/1733619845296.png)  |

### 开发

由于Electron版本更新较快，接口和功能可能会有较大的变动，建议参考官方文档[🔗官方文档](https://www.electronjs.org/zh/docs/latest/)（有中文）。

#### **0. Electron中常用的模块**

可以从命名规则中得知，如果引入的是类，用户是可以创建多个实例对象的，如果引入的是对象，则是Electron已经创建好的，且一个应用里只有一个实例。

```javascript
const { 
    app, // 对该应用程序的引用，一个应用可以包含0-多个窗口
    BrowserWindow, // 窗口类
    Menu, // 菜单类
    Tray, // 任务栏图标
    Notification, // 通知
    shell, // Shell对象，执行系统命令
    ipcMain, // IpcMain对象，用来触发和监听消息事件
    globalShortcut, // GlobalShortcut对象，用来处理全局快捷键的绑定和解绑
    screen, // Screen对象，用来获取显示器相关数据
    nativeImage // NativeImage对象，用来生成和获取图片
} = require('electron');
```

#### **1. 单实例**

单实例是为了保证只有一个在工作，防止误操作多次双击后启动多个实例。这里多个实例没有意义，所以限制多实例的生成，如果重复创建app对象则直接退出。

```javascript
const gotTheLock = app.requestSingleInstanceLock()
if (!gotTheLock) {
    app.quit()
} else {
    app.on('second-instance', (event, commandLine, workingDirectory) => {
            // 当运行第二个实例时,将会聚焦到myWindow这个窗口
            if (win) {
                if (win.isMinimized()) win.restore()
                win.focus()
            }
        })
        // app 启动创建窗口
    app.on('ready', createWindow);
}
```
#### **2. 自定义协议**

在浏览器中常用的协议有`http`、`https`、`file`等，自定义协议可以让我们通过在浏览器访问自定义schema打开我们的应用，并执行指定操作。这里我们以软件名称`fily`作为协议名称，后面的uri路径为要访问的项目名称(目前支持英文字母和数字的名称，中文需要额外的编解码)。

```http
fily://project-name
```
实现方式MacOS和Windows有区别，MacOS通过`app.setAsDefaultProtocolClient`设置默认协议，Windows通过第二实例打开。
```javascript
// MacOS
if (process.defaultApp) {
    if (process.argv.length >= 2) {
        app.setAsDefaultProtocolClient('fily', process.execPath, [path.resolve(process.argv[1])])
    }
} else {
    app.setAsDefaultProtocolClient('fily')
}

app.on('open-url', (event, url) => {
    openUrl(url);
})

//Windows
const gotTheLock = app.requestSingleInstanceLock()
if (!gotTheLock) {
    app.quit()
} else {
    app.on('second-instance', (event, commandLine, workingDirectory) => {
            // 当运行第二个实例时,将会聚焦到myWindow这个窗口
            if (win) {
                if (win.isMinimized()) win.restore()
                win.focus()
                let lastCommand = commandLine[commandLine.length - 1];  // ①
                if (lastCommand.indexOf('fily://') === 0) {             // ②
                    openUrl(lastCommand)                                // ③
                }
            }
        })
        // app 启动创建窗口
    app.on('ready', createWindow);
}
```

#### **3. 进程间通信**

Electron分主进程和渲染进程，主程序 (main process)，它运行在 Node.js环境里，负责控制应用的生命周期、显示原生界面、执行特殊操作并管理渲染器进程 (renderer processes)。应用中的每个页面都在一个单独的进程中运行，我们称这些进程为 渲染器(renderer) 。 渲染器也能访问前端开发常会用到的 API和工具，例如用于打包并压缩代码的webpack，还有用于构建用户界面的 React 。

| 通信方向                 | 代码                                              |
| ------------------------ | ------------------------------------------------- |
| 主进程监听事件           | `ipcMain.on(EVENT-NAME, (event, data) => {})`     |
| 主进程向渲染进程发送通知 | `win.webContents.send(EVENT-NAME, {})`            |
| 渲染进程监听事件         | `ipcRenderer.on(EVENT-NAME, (event, data) => {})` |
| 渲染进程向主进程发送通知 | `ipcRenderer.send(EVENT-NAME, {})`                |

主进程和渲染进程主要负责的功能范围不同，进程间的通信通过事件监听和消息通知来交互完成。

> Electron 的主进程是一个拥有着完全操作系统访问权限的 Node.js 环境。 除了 Electron 模组之外，你也可以使用 Node.js 内置模块 和所有通过 npm 安装的软件包。另一方面，出于安全原因，渲染进程默认跑在网页页面上，而并非 Node.js里。为了将 Electron 的不同类型的进程桥接在一起，我们需要使用被称为 预加载 的特殊脚本。

#### **4. 创建窗口**

|                                                              |              |
| ------------------------------------------------------------ | ------------ |
|![](https://xuhang.github.io/post-images/1733619943936.png)  | Windows   UI |
| ![](https://xuhang.github.io/post-images/1733619968189.png) | MacOS UI     |

软件打开之后需要展示给用户使用的界面，ELectron封装了不同操作系统的UI界面，所以在不同系统上使用会有不同的风格。为了统一风格，我们去掉窗口的边框，使用HTML+CSS绘制一套统一的界面。可以从上面的对比图看出，Windows和MacOS上的顶部titleBar的样式是有区别的，Windows的titleBar会显示菜单，MacOS是空白的，因为MacOS的菜单是当程序窗口是活动窗口时，在显示器左上角会显示当前程序的菜单，所以两个平台的菜单绑定是有区别的。

![](https://xuhang.github.io/post-images/1733620004039.png)

```javascript
// 创建窗口要在app启动完成之后开始
app.on('ready', createWindow);

function createWindow() {
    // 读取package.json文件中的属性
    var package = require('./package.json')
    console.log('starting ' + package.name + ' ' + package.version)

    // 隐藏菜单 Menu.setApplicationMenu(null)
    // 这里菜单仅对MacOS有效，因为Windows隐藏边框之后没有菜单栏
    const appMenu = Menu.buildFromTemplate(appMenuTemplate);
    Menu.setApplicationMenu(appMenu)

    // 创建浏览器窗口并设置宽高
    win = new BrowserWindow({
        width: 1200,
        height: 800,
        titleBarStyle: 'customButtonsOnHover',
        icon: __dirname + './img/fily.png',
        maximizable: false,
        minimizable: false,
        fullscreenable: false,
        frame: false, // 是否有边框
        // resizable: true,        // 是否可调整大小
        transparent: true, //是否透明
        webPreferences: { // 新版Electron添加了一些安全限制，这里需要设置
            nodeIntegration: true,
            enableRemoteModule: true,
            contextIsolation: false
        }
    })
    remoteMain.initialize()
    remoteMain.enable(win.webContents);

    // MacOS底部的Dock栏图标
    if (systemType === "Darwin") {
        app.dock.setIcon(path.join(__dirname, './img/icon_512x512.png'));
        // 设置Dock菜单
        const dockMenu = Menu.buildFromTemplate(dockMenuTempalte);
        app.dock.setMenu(dockMenu);
    }

    // 加载页面 -- 向用户展示的界面
    win.loadFile('./page/projectRelatedFiles.html')

    // Windows 任务栏图标菜单（右下角） / MacOS任务栏图片（右上角）
    // tray = new Tray(path.join(__dirname, "./img/fily.png"));
    const trayImage = nativeImage.createFromPath(path.join(__dirname, "./img/icons.iconset/file20Template.png"));
    trayImage.resize({ width: 16, height: 16 })
    trayImage.setTemplateImage(true);
    tray = new Tray(trayImage);
    const contextMenu = Menu.buildFromTemplate([
        { label: '退出', click: function() { app.quit() } }
    ])
    tray.setToolTip('Fily - File manager for you!')
    tray.setContextMenu(contextMenu)

    // 添加window关闭触发事件
    win.on('close', () => {
        win = null;
        app.quit();
    });

    win.setProgressBar(0.7);
}

```
#### **5. 添加快捷键注册**

可以通过注册全局快捷键快速执行某个操作或者在菜单项里对指定的菜单绑定快捷键。

```javascript
// 注册全局快捷键-退出应用
globalShortcut.register('Ctrl+W', () => {
    win = null;
    app.quit();
});
// 全局快捷键-打开开发中工具
globalShortcut.register('Ctrl+Shift+I', () => {
    win.webContents.toggleDevTools()
})

// 通过构建菜单绑定快捷键执行菜单操作
const appMenuTemplate = [{
    label: 'Projects',
    submenu: [{
        label: 'New Project',
        accelerator: 'Cmd+N',
        click: () => {
            win.webContents.send('new-project-dialog', {})
        }
    }, {
        label: 'Rename Project'
    }, {
        label: 'Delete Project'
    }, {
        type: 'separator'
    }, {
        label: 'Archive'
    }]
}, {
    label: 'Files',
    submenu: [{
        label: 'Import File'
    }, {
        label: 'Import Folder'
    }]
}]
```
#### **6. 添加进度条**

Windows和MacOS都可以在任务（Dock）栏的图标那里显示一个进度条，通常可以用来显示下载进度或者任务完成进度。

| Windows UI                                                   | MacOS UI                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|![](https://xuhang.github.io/post-images/1733620043980.png)  |

```javascript
// 这里的进度条在70%的进度
win.setProgressBar(0.7);
```

#### **7. 任务栏预览快捷操作（仅Windows）**

在任务栏的预览视图那里可以设置快捷操作键，这里我们通过添加两个按钮用于访问“前一个”和“后一个”项目。

![](https://xuhang.github.io/post-images/1733620060775.png)

```javascript
// 主线程
win.setThumbarButtons([{
    tooltip: '上一张',
    icon: path.join(__dirname, './resource/icos/pre.ico'),
    click() { switchThumbImage(-1) }
}, {
    tooltip: '下一张',
    icon: path.join(__dirname, './resource/icos/next.ico'),
    click() { switchThumbImage(1) }
}])

function switchThumbImage(delta) {
    if (delta > 0) {
        win.webContents.send('show-next-project', {})
    } else {
        win.webContents.send('show-prev-project', {})
    }
}

// 渲染进程
ipcRenderer.on('show-next-project', (event, message) => {
    // document.getElementById('gallery_image').setAttribute('src', message['src']);
    let curProj = $('.projlist ul li.active')
    let nextProj;
    if (!!curProj[0]) {
        nextProj = curProj.next();
        if (!nextProj[0]) {
            nextProj = $('.projlist ul li').first()
        }
    } else {
        nextProj = $('.projlist ul li').first();
    }
    nextProj.click();
})
ipcRenderer.on('show-prev-project', (event, message) => {
    let curProj = $('.projlist ul li.active')
    let prevProj;
    if (!!curProj[0]) {
        prevProj = curProj.prev();
        if (!prevProj[0]) {
            prevProj = $('.projlist ul li').last()
        }
    } else {
        prevProj = $('.projlist ul li').last();
    }
    prevProj.click();
})
```

#### **8. 通知**

这里监听右上角的关闭按钮，点击关闭之后会弹出通知，并延迟1s后退出应用。

| Windows UI                                                   | MacOS UI                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|![](https://xuhang.github.io/post-images/1733620099033.png) |

```javascript
ipcMain.on('close-window', (event, data) => {
    if (!Notification.isSupported()) {
        console.log('当前系统不支持通知');
    }
    app.setAppUserModelId("我的Electorn示例")
    const notification = {
        appId: "com.xuhang.electron.demo",
        icon: path.join(__dirname, './img/icon_512x512.png'),
        title: '正在关闭……',
        body: 'Bye ヾ(•ω•`)o'
    }
    new Notification(notification).show()

    setTimeout(() => {
        console.log('fily is exiting ...')
        app.quit();
    }, 1000);
});
```

#### **9. 数据存储**

项目列表和项目相关的文件关联关系需要持久化才能保证数据不丢失，通常可以选择将数据保存到远程数据库，但是这样必须要有一个实时在线的服务。也可以选择数据保存在本地，这样既能保证数据实时可用，也能保证数据的安全性，一来也是因为数据没有在线访问的必要。

离线存储有几种可选方案：

1. 利用浏览器的localStorage存储，优点是浏览器原生支持，Electron本身就包含了Chromium，缺点是如果清空浏览器缓存数据也丢失了。
2. 利用Electron的文件读写API，自定义存储格式和存储路径，将数据保存到本地文件，优点是没有第三方依赖，缺点是需要自己实现数据的序列化与反序列化。
3. 利用第三方库，如lowdb将数据以JSON格式存储在本地文件，优点是提供了增删改查的API，缺点是有学习成本。
4. 集成SQLite，优点是数据有各种工具可以读取，也可以通过网盘将数据进行同步，SQLite也比较稳定成熟，缺点是Electron对SQLite的集成比较复杂，打包后体积也会变大。

综合各种方案的优劣，我们前期需要快速出一个可用的版本，所以选择了lowdb。

#### **10. 引入jQuery**

前端开发的发展速度很快，可能两三年技术路线就会更新迭代，jQuery虽然过时了，但是对于非前端开发来说还是相对熟悉的框架，所以为了尽可能简化开发这里还是使用jQuery。

Electron默认启用了Node.js的require模块，而jQuery等新版本框架为了支持commondJS标准，当Window中存在require时，会启用模块引入的方式。这里主要针对jQuery1.11.1及以后的版本。

```javascript
<script src="../script/jquery.min.js"></script>
<script>
    if (typeof module === 'object') {
        window.jQuery = window.$ = module.exports;
    }
</script>
```

由于Electron和ES6对于模块的导入导出采用了不同的方式，所以在引用JS文件时很容出现报错。关于各种Javascript运行时环境下的模块导入导出可以参见 [🔗这篇文章](https://mp.weixin.qq.com/s?__biz=MzIzMDE5NTQyMQ==&mid=2247484053&idx=1&sn=dec6d77da1de255d1b9a92289c3a3df3)

#### **11. 开发者工具**

为了方便对页面进行调试，需要为每个窗口添加打开/关闭开发者工具的快捷键监听事件。在页面的js文件中为window对象添加事件监听器并在按键为`Ctrl+Shift+I`是打开/关闭开发者工具，针对当前活动窗口的快捷键才会生效 。

```javascript
window.addEventListener('keydown', (event) => {
    if (event.ctrlKey && event.shiftKey && event.key === 'I') {
        remote.getCurrentWindow().toggleDevTools();
    }
})
```

#### **12. 聊天功能**

在界面左上角重载按钮的旁边有个聊天图标，点击会在屏幕右侧出现一个透明窗口，该窗口实现了鼠标点击穿透，但同时也保持窗口置顶，最下方的文字输入区域设置透明度0.6，但是为了能输入文字，该区域为点击不穿透。该聊天功能使用腾讯的IM云服务，每个打开该聊天界面的终端会自动申请加入一个开放群，目前仅实现了文字聊天功能。

| Windows UI                                                   | MacOS UI                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![](https://xuhang.github.io/post-images/1733620151597.png) |





## 三、打包发布

开发完成后将代码打包并生成系统安装包。之前主要使用`electron-package`和`electron-builder`来打包，目前Electron官方推荐使用`electron-forge`，后者针对发布和更新会更方便一点。

electron-builder在package.json中添加如下参数：

```json
"build": {
    "productName": "fily",
    "appId": "com.xuhang.electron.demo",
    "copyright": "Copyright © 2021-2022 xu",
    "directories": {
        "output": "build"
    },
    "nsis": {
        "oneClick": false,
        "allowElevation": true,
        "allowToChangeInstallationDirectory": true,
        "installerIcon": "./img/icon.ico",
        "uninstallerIcon": "./img/icon.ico",
        "installerHeaderIcon": "./img/icon.ico",
        "createDesktopShortcut": true,
        "createStartMenuShortcut": true,
        "shortcutName": "Fily"
    },
    "win": {
        "icon": "./img/icon.ico",
        "target": [
            {
                "target": "nsis",
                "arch": [
                    "ia32"
                ]
            }
        ]
    },
    "dmg": {
        "background": "media/images/dmg-bg.png",
        "icon": "media/images/icon.icns",
        "iconSize": 100,
        "sign": false,
        "contents": [
            {
                "x": 112,
                "y": 165
            },
            {
                "type": "link",
                "path": "/Applications",
                "x": 396,
                "y": 165
            }
        ]
    },
    "mac": {
        "target": [
            "dmg"
        ],
        "icon": "./img/Icon.icns",
        "hardenedRuntime": true,
        "identity": null,
        "entitlements": "electron-builder/entitlements.plist",
        "entitlementsInherit": "electron-builder/entitlements.plist",
        "provisioningProfile": "electron-builder/comalibabaslobs.provisionprofile"
    },
    "pkg": {
        "isRelocatable": false,
        "overwriteAction": "upgrade"
    },
    "mas": {
        "icon": "./img/Icon.icns",
        "hardenedRuntime": true,
        "entitlements": "electron-builder/entitlements.mas.plist",
        "entitlementsInherit": "electron-builder/entitlements.mas.plist"
    }
}
```

electron-forge添加如下参数：

```json
"config": {
    "forge": {
        "packagerConfig": {},
        "makers": [
            {
                "name": "@electron-forge/maker-squirrel",
                "config": {
                    "name": "fily"
                }
            },
            {
                "name": "@electron-forge/maker-zip",
                "platforms": [
                    "darwin"
                ]
            },
            {
                "name": "@electron-forge/maker-deb",
                "config": {}
            },
            {
                "name": "@electron-forge/maker-rpm",
                "config": {}
            }
        ]
    }
}
```

然后配置script脚步指定指定的任务：

```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "electron-forge start",
    "package": "electron-forge package",
    "dist": "npm run package && electron-builder build",
    "make": "electron-forge make"
}
```

## 四、源码

该实验项目开源，代码存放在 [🔗Github](https://github.com/xuhang/fily.git)


[1]: https://www.electronjs.org/apps
[2]: https://github.com/xuhang/fily.git