---
title: "论linux如何设置动态壁纸"
image:
    path: /assets/images/live-wallpapers.gif
    thumbnail: /assets/images/live-wallpapers-thumbnail.gif
    caption: "动态壁纸，骚不骚"
excerpt: "这篇文章将会教大家如何在linux设置动态壁纸"
tags:
    - Linux
    - 壁纸
    - 教程
last_modified_at: 2018-07-30T20:50:51+08:00
---

## 前言
最终实现的效果就和上面那张图一样, 原理其实就是把视频格式的文件用播放器渲染到桌面, 说白了就相当于[wallpaper engine](https://store.steampowered.com/app/431960/Wallpaper_Engine/), 这个wallpaper engine是windows下的一个动态壁纸软件(~~但是收费~~), 今天我们要实现的就是类似它这个功能, 只不过平台是linux(~~按理来说Mac应该也可以~~), 而且我们自己DIY的还有个好处就是完全免费(~~如果你用windows的话还是乖乖掏钱吧~~)

## 播放器选择
那么首先需要选择一个播放器, 这里推荐mplayer(~~Linux神器~~)

* 如果你是Ubuntu那么可以: `sudo apt-get install mplayer`
* 如果你是Archlinux那么可以: `sudo pacman -S mplayer`
* 其它同理, 一般包管理器里面都会包含mplayer

## 窗口定位器(~~自创叫法~~)
### xwinwrap
* xwinwrap介绍

    官方介绍:
>XWinWrap is a small utility written a loooong time ago that allowed you to stick most of the apps to your desktop background. What this meant was you could use an animated screensaver (like glmatrix, electric sheep, etc) or even a movie, and use it as your wallpaper.

    简单来说这是一个古老的工具, 用来让动图和视频等当你的桌面(~~我理解这个过程是把视频显示重定位到桌面~~)

* xwinwrap安装
    * Archlinux: `yaourt -S shantz-xwinwrap-bzr`
    * 其它Linux发行版: https://github.com/ujjwal96/xwinwrap


* xwinwrap使用

    直接上命令: ``mkfifo /tmp/wallpaper; xwinwrap -fs -nf -ov -- mplayer -shuffle -slave -input file=/tmp/wallpaper -loop 0 -wid WID -nolirc `find ~/wallpapers -type f` &``

    命令解释:
    * `mkfifo /tmp/wallpaper`
        这条命令的意思是创建一个路径为`/tmp/wallpaper`的命名管道, 后续可以用来控制mplayer的行为, 例如暂停, 下一个等
    * ``xwinwrap -fs -nf -ov -- mplayer -shuffle -slave -input file=/tmp/wallpaper -loop 0 -wid WID -nolirc `find ~/wallpapers -type f` &``
        * `-fs`: 全屏
        * `-nf`: 禁止聚焦, 壁纸当然不需要聚焦
        * `-ov`: 设置允许被覆盖, 如果没设置这个会导致你设置壁纸后整个屏幕只有壁纸
        * `--`: 分隔符, 表示后面的都是输入视频的命令
        * `-shuffle`: 随机切换后面指定的视频
        * `-slave`: 切换到被动式, 可以被控制
        * `-input`: 这里我使用了命名管道的形式来控制, 路径就是之前创建的`/tmp/wallpaper`
        * `-loop`: 循环次数, 0表示永久循环
        * `-wid`: 指定播放渲染的窗口ID, 前面的xwinwrap会创建window ID为`WID`的窗口, 所以这里指定为WID
        * `-nolirc`: 关闭红外线遥控
        * `` `find ~/wallpapers -type f` &``: 遍历家目录下的wallpapers目录以及子目录的文件, 设置所有找到的视频文件作为壁纸, 并且切换到后台运行

    <br />最终效果就是搜索家目录下的`wallpapers`目录下的所有视频文件随机设置壁纸, 并且也是有声音的！







