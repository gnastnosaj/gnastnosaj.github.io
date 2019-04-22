---
title: Electron学习笔记
date: 2019-04-19 13:20:13
tags:
---

# 关于

按照官方的说法，Electron是由Github开发，通过HTML、CSS和JavaScript来构建跨平台桌面应用程序的一个开源库。因此可以基于Electron来创建原生的桌面应用，它可以具备操作窗口、菜单、通知、文件等原生能力。这也是本次学习使用Electron的主要目的。

# 快速入门

从Github上拉取Electron API 演示应用程序(https://github.com/electron/electron-api-demos) ，由于演示程序也是基于Electron构建，所以可以通过分析其源码来熟悉一个常规的Electron程序的整体结构以及掌握包括编译、打包、运行、生成安装包等构建过程。其次，演示程序提供了调用API的示例代码，通过分析示例代码、查看实时运行效果可以快速掌握API的使用。
<img src="/images/electron-api-demos.png" style="width: 50%; height: 50%; margin-left: 0px;">

# 项目构建

使用官方推荐的electron forge来构建项目
```
$ npm install -g electron-forge
$ electron-forge init electron-demo
$ electron-forge start
```
<img src="/images/electron-forge-start.png" style="width: 50%; height: 50%; margin-left: 0px;">
打包并生成安装程序
```
$ electron-forge make
√ Checking your system
√ Resolving Forge Config
We need to package your application before we can make it
√ Preparing to Package Application for arch: x64
√ Compiling Application
√ Preparing native dependencies
√ Packaging Application
Making for the following targets:
√ Making for target: squirrel - On platform: win32 - For arch: x64
```
![](/images/electron-forge-make-1.png)![](/images/electron-forge-make-2.png)

# Electron API

常用的Electron API在Electron API 演示应用程序中均有示例代码，且可以实时查看运行效果，这里就不再赘述。以下主要记录一些API使用过程中遇到的问题以及注意事项。

- 通知API不生效
在Github上的issue列表发现已有人提过此问题![](/images/electron-notification-issue.png)大致意思是AppUserModelId在不主动设置的情况下会被设置为某个默认值，导致通知的AppUserModelId和进程的AppUserModelId不匹配。解决方案如下：
```
//1.将运行时环境添加到开始菜单（windows10要求，否则无法接收到通知。）
//node_modules\electron\dist\electron.exe
//2.主动设置AppUserModelId
app.setAppUserModelId(process.execPath)
```
- 进程间通讯API
```
//主进程向渲染进程发送消息
mainWindow.webContents.send('<event>', <request>);

//渲染进程向主进程发送消息
ipcRenderer.send('<event>', <request>);

//主进程接收消息并返回
ipcMain.on('<event>', (event, <request>) => {
    event.sender.send('<event>', <response>);
});

//渲染进程接收消息并返回
ipcRenderer.on('<event>', (event, <request>) => {
    event.sender.send('<event>', <response>);
});

//渲染进程间通讯可以通过WindowID来指定发送和接收方
const firstWindowID = BrowserWindow.getFocusedWindow().id;
secondWindow.webContents.send('<event>', <request>, firstWindowID);

ipcRenderer.on('<event>', (event, <request>, firstWindowID) => {
    const firstWindow = BrowserWindow.fromId(firstWindowID);
    firstWindow.webContents.send('<event>', <response>);
});
```
