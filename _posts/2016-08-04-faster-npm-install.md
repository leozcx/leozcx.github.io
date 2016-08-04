---
layout: post
title: 设置国内npm仓库加快下载速度
author: Leo
---

直接使用npm安装包的时候动不动会因为网络问题导致安装失败，淘宝提供了一个npm的国内镜像，从淘宝镜像下载速度明显快了很多。

## 升级node及npm 
如果node比较老，需要先升级，否则可能导致从国内镜像下载失败。由于我的操作系统是windows，我没有找到像Linux那样简单的升级node的方式，虽然也有 `nvmw` 可以管理node版本，但我一直没有试成功，总是有这样那样的错误，所以只好老老实实地下载安装包安装。

## 安装npmrc
----------
需要将`npm` 的registry设置为淘宝的registry https://registry.npm.taobao.org. 这里我使用的是npmrc，它有2个好处：

1. 不需要每次执行设置registry的命令
2. 可以管理多个npm配置，随时在不同的配置间切换

安装npmrc:

```
npm install -g npmrc
```
常用命令：

```shell
#列出当前有几个配置
npmrc
#创建新的配置
npmrc -c taobao
#在不同的配置间切换
npmrc “配置名”
```

## 配置registry

### 首先创建新的配置

  `npmrc -c taobao`
  
### 设置registry

  `npm config set registry https://registry.npm.taobao.org`

## 另一种选择
在使用`npmrc` 的时候发现有这样一个选项： `npmrc -r <registry>`, 可以设置当前使用哪个镜像，这其中就包括中国的镜像：

```
npmrc -r cn
```
执行 `npm config get registry` 发现设置的是 http://r.cnpmjs.org/ ，google了一下，貌似这是国人发起的私有npm项目，而它的国内镜像指向的就是淘宝。 
这种设置更简单。

## 问题
更换registry后发现还有一些，比如使用`yo` 的时候安装generator总是失败，可能是这些镜像不支持官方npm的某些api.
