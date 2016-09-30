
- inject css/js to index.html
利用webpack，可以像引入js一样引入css，比如 `require('normalize.css/normalize.css')`将引入 `node_modules/normalize.css/normalize.css`.

实际上，用`require('xxx')`时，webpack区分3种不同方式的引入：
1. 绝对路径，如 `require('/home/me/file'), require('C:\Home\me\file')`
2. 相对路径，如 `require('../src/file'), require('./file')`
3. 模块路径(module path)，如`require('module'), require('module/lib/file')`

### 绝对路径
webpack会根据绝对路径引入相关文件。如果require的是路径，那么会查看路径下的package.json中的main字段，找到的话则根据配置`resolve.extensions`把所有符合条件的文件都引入；如果没有package.json或是没有main字段，则引入index文件；如果没有index，或index文件后缀与配置不符，则什么也不引入。

### 相对路径
相对路径是针对包含`require('xxx')`的文件的路径，如果这个文件不存在则用配置文件中的`context`值（什么情况下这个文件会不存在呢？举个例子：loader-generated files）

### 模块路径
模块路径是根据`resolve.modulesDirectories`配置的，默认包含`['web_modules',
'node_modules']`。模块的搜索过程类似于nodejs搜索模块的过程：即先查看当前路径，再查看父目录，一直向上，直到根目录。

- build for different env: dev, stage, prod
- config for different env

# 参考资料
- [https://npmcompare.com/compare/browserify,grunt,gulp,webpack] (https://npmcompare.com/compare/browserify,grunt,gulp,webpack)
