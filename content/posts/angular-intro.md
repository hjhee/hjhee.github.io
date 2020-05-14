+++
title = "Angular开发环境配置指南"
date = 2018-03-02T10:35:38+01:00
tags = ["angular"]
draft = false
+++

本文介绍了如何从Github上下载代码、在本机搭建Angular开发环境以及浏览本机由该Angular编写的网站。

<!--more-->

# Angular介绍

Angular是一个前端框架，由Node.js驱动，用于网页开发人员编写网站界面。利用Angular编写的网页可以将数据和界面分离，由此解耦了前端开发和后端开发。Angular提供了数据绑定功能，给HTML网页编写动态页面提供了更多便利。

关于Angular可参考如下文章：

- [Angular开发者指南(1) -- Angular介绍](https://yalishizhude.github.io/2015/09/10/angular-introduction/)
- [前端开发框架简介：angular 和 react](https://cloud.tencent.com/developer/article/1004715)

# 开发环境搭建

下面介绍如何获取最新的项目代码并安装必要的开发工具以生成网页。项目前端代码托管于[gas_fEnd](https://github.com/hjhee/gas_fEnd)，项目后端代码托管于[gas_bEnd](https://github.com/hjhee/gas_bEnd)。

## 本地网站构建流程

1. 安装开发工具
2. 获取代码
3. 生成网页
4. 更新项目代码

## 安装开发工具

Angular项目代码通过Git管理，为了方便跟进项目，有必要安装Git客户端获取并更新项目代码。为了部署方便，本文利用[chocolatey](https://chocolatey.org/)管理所有与之相关的软件。

安装chocolatey：利用管理员权限打开cmd窗口，并在cmd下执行下面的代码。

```batch
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```

安装git：在上一步打开的cmd窗口内执行下面的代码。
```batch
choco install -y git.install
```

安装Node.js：Angular项目由Node.js管理，需要安装npm以获取需要的依赖包。
```batch
choco install -y nodejs-lts
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

安装Angular CLI：通过npm安装Angular CLI以执行Angular代码
```batch
cnpm install -g @angular/cli
```

关于chocolatey可参考如下文章：
- [Windows下的包管理器 Chocolatey 的使用](https://www.jianshu.com/p/abaa0e8c261f)
- [Windows下的包管理器Chocolatey](https://www.jianshu.com/p/831aa4a280e7)

关于git可参考如下文章：

- [Git简介](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001373962845513aefd77a99f4145f0a2c7a7ca057e7570000)
- [Git详解之一 Git介绍](http://blog.csdn.net/tronteng/article/details/7202577)

关于Node.js可参考如下文章：

- [Node.js 究竟是什么？](https://www.ibm.com/developerworks/cn/opensource/os-nodejs/)
- [深入浅出Node.js（一）：什么是Node.js](http://www.infoq.com/cn/articles/what-is-nodejs)

## 获取代码

在第一次获取代码的时候，需要通过git客户端获取，以`D:\dev\web\gas`为例说明步骤：

确保目录`D:\dev\web`存在，若不存在应首先建立好相关目录。git将Angular代码保存到`D:\dev\web\gas`目录，该目录不需要事先创建，若已存在须该目录内不包含任何文件（隐藏文件以及非隐藏文件）。

打开一个cmd窗口，并执行如下代码，注意从这里开始都不需要管理员权限。
```batch
cd /d D:\dev\web
git clone https://github.com/hjhee/gas_fEnd.git gas
cd /d D:\dev\web\gas
cnpm install
```

# 生成网页

Angular项目通过Typescript描述网页，需要运行本地服务器渲染HTML模板，以渲染本地/网络数据至模板并显示。打开cmd窗口并运行如下代码，注意在访问本地网页的期间不能关闭该cmd窗口。

```batch
cd /d D:\dev\web\gas
ng serve
```

现在可以通过浏览器访问[http://localhost:4200/](http://localhost:4200/)以查看Angular生成的网页。

关于Typescript可参考如下文章：

- [一篇缺失的 TypeScript 介绍](https://juejin.im/entry/599154035188257da64bef9d)

# 维护项目

在项目开发过程中，代码交由GitHub.com托管。利用以下命令从[gas_fEnd](https://github.com/hjhee/gas_fEnd)获取最新的代码至本地：

```batch
cd /d D:\dev\web\gas
git fetch origin master
git reset --hard FETCH_HEAD
git clean -df
```

关于GitHub.com可参考如下文章：

- [初识 GitHub · 简介篇](http://blog.csdn.net/qq_35246620/article/details/66980283)