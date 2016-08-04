---
layout: post
title: 用nconf作为node应用的配置管理工具
author: Leo
---

把配置信息写在单独的配置文件，而不是hardcode在代码里绝对是一个好习惯，node应用可以用nconf作为配置管理工具。

### 安装 
```shell
npm install nconf -S
```

### 使用
nconf使用json作为配置文件的格式，所以第一步先要创建一个json文件，比如：

```
{
  "timeout": 10,
  "gitHub": {
    "version": "3.0.0",
    "host": "api.github.com"
  }
}
```
这里每个key的引号不能省略。保存文件到"config/app.json"。  
在代码中，首先载入配置文件:  

```shell
nconf = require('nconf');
nconf.argv().env();
if (process.env.DEVELOPMENT)
	nconf.file('dev', "./config/app-development.json");
nconf.file('./config/app.json');
```
越先载入的优先级越高，比如上面配置项的优先级为：argv > env > file。
<<<<<<< HEAD

=======
>>>>>>> 63583d139c1d40f74c850d580e21cafdb4a00a5b
### 不同的配置提供方式
配置可以不同的方式提供，上面的例子就涉及到：参数，环境变量，文件。nconf支持可扩展的配置来源，比如可以将配置保存在redis数据库中，用"nconf-redis"读取。
