---
title: Scoop -window下包管理器
w: 下包管理器
i: true
'n': true
d: true
o: true
date: 2025-04-25 19:00:49
tags:
-  widnows
categories:
- windows
---


## 什么是Scoop?

Scoop 是 Windows 的命令行安装程序，是一个强大的包管理工具。可以在 github 上找到其项目的相关信息，[项目地址](https://www.mobaijun.com/go.html?u=aHR0cHM6Ly9naXRodWIuY29tL1Njb29wSW5zdGFsbGVyL1Njb29w), Scoop 等一系列包管理器的诞生，第一大便利就是省去了上述繁琐的「搜索 - 下载 - 安装」的步骤，让我们能够通过「一行代码」急速安装。

同时，用 Scoop 来安装和管理我们的软件：

- 集搜索、下载、安装、更新软件于一体：极大的降低了安装维护一个软件的成本，我们甚至不必在软件本身的复杂菜单中寻找那个更新按钮来更新软件自己
- 将软件干干净净的安装到电脑的「用户文件夹」下：这样既不会污染路径也不会请求不必要的权限（UAC）
- 在卸载软件的时候，能够尽量清空软件在电脑上存储的任何数据和痕迹

Scoop 最适合安装那种干净、小巧、开源的软件。并且，Scoop 也极度适合为开发者配置开发环境，不过这些很多都涉及到进阶使用技巧。下面先从基础的安装方法开始介绍。

## 安装

**要求：**

-  PowerShell >= 5.0 (如果是 Window10 则默认满足此条件)
-  请确保已允许PowerShell执行本地脚本，可以使用下面的命令开启：
```
set-executionpolicy remotesigned -scope currentuser

```


从 Power Shell  终端运行以下命令来安装 Scoop：
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```
它会将 Scoop 安装到其默认位置：`C:\Users\<YOUR USERNAME>\scoop` 。 全局安装的程序（所有用户可用，使用`--global`或 `-g` 选项）位 于`C\ProgramData\scoop`路径中。

如果想自定义安装位置, 可以在==安装前==设置 ：
用户安装目录：

```powershell
$env:SCOOP='D:\Applications\Scoop'
[Environment]::SetEnvironmentVariable('SCOOP', $env:SCOOP, 'User')
```

全局安装目录：
```powershell
$env:SCOOP_GLOBAL='F:\GlobalScoopApps'
[Environment]::SetEnvironmentVariable('SCOOP_GLOBAL', $env:SCOOP_GLOBAL, 'Machine')
```

## 常用命令 

```
scoop install  git
scoop install sudo # 申请管理员权限，和linux下sudo命令相似，全局安装必备
sudo scoop install -g python # 其实大部分我都推荐全局安装

scoop update latex # 更新
scoop update -g hugo # 更新全局安装的软件
scoop update * # 更新所有
scoop uninstall curl # 卸载
scoop uninstall -g gcc # 卸载全局安装的软件


scoop search Source-Han-Mono # 搜索
scoop info miniconda3 # 软件信息
scoop home pwsh # 打开软件主页

```



```
scoop help
Usage: scoop <command> [<args>]

Available commands are listed below.

Type 'scoop help <command>' to get more help for a specific command.

Command    Summary
-------    -------
alias      Manage scoop aliases
bucket     Manage Scoop buckets
cache      Show or clear the download cache
cat        Show content of specified manifest.
checkup    Check for potential problems
cleanup    Cleanup apps by removing old versions
config     Get or set configuration values
create     Create a custom app manifest
depends    List dependencies for an app, in the order they'll be installed
download   Download apps in the cache folder and verify hashes
export     Exports installed apps, buckets (and optionally configs) in JSON format
help       Show help for a command
hold       Hold an app to disable updates
home       Opens the app homepage
import     Imports apps, buckets and configs from a Scoopfile in JSON format
info       Display information about an app
install    Install apps
list       List installed apps
prefix     Returns the path to the specified app
reset      Reset an app to resolve conflicts
search     Search available apps
shim       Manipulate Scoop shims
status     Show status and check for new app versions
unhold     Unhold an app to enable updates
uninstall  Uninstall an app
update     Update apps, or Scoop itself
virustotal Look for app's hash or url on virustotal.com
which      Locate a shim/executable (similar to 'which' on Linux)
```
