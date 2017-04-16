---
layout: post
title: Immutable 应用及实现
author: Leo
---

# Why immutable

Immutable，不可变的：对象一旦创建，就不能再改变；有什么用呢？让开发更简单。看下面的例子：

```javascript
let params = {
  a: 1
}
...
function fn1 (payload) {
  payload.b = 2 // 修改了外面的params
  ...
}
```

逻辑需要我们在函数体内新增了属性b，同时无意中把外面的params也同时修改了；如果我们在外面的逻辑中需要依赖属性b，那我们可能很难判断b是在哪里修改的，这样引起的bug会很难修复；我们不应该在函数内修改传过来的参数，但你并没有规则真的禁止你这样做；实际上这样的代码很常见。结果就是，程序出了bug，我们可能花很长的时间寻找对象都在哪里被修改，无形中降低了开发效率。

另外一个好处是cache。现代的很多前端MVVM框架接管了将model渲染到view的工作，其中一个优化就是：如果model没有变，那么就不需要更新view。比如下面的例子：

```javascript
let profile = {
  name: 'Leo',
  friends: [
    {
      id: 103742,
      name: 'Cat'
    },
    ...
  ]
}

profile.friends.push({id: 3243, name: 'Dog'})
render(profile)
function render(profile) {
  // 这样是不行的
  if (profile === _lastProfile)
    return
  ...
}
```

我们希望当profile没有变化的时候，不重新渲染view，所以判断当前的profile与上一次的profile是否相等。我们知道这样是行不通的，因为在对象是可变的情况下，即使内容发生了变化，它们的引用还是相等，这就导致即使真的发生了变化，view也得不到更新。

好了，让我们再来看一下对象不可变的情况下，上面2个例子会是什么情况。

```javascript
let params = {
  a: 1
}
...
function fn1 (payload) {
  payload = {...payload} // 或者Object.assign({}, payload)
  payload.b = 2 // 不会修改外面的params
  ...
}
```
我们通过对象展开运算符新生成了一个对象，对新的对象进行操作，从而保证原对象没有发生变化。


```javascript
let profile = {
  name: 'Leo',
  friends: [
    {
      id: 103742,
      name: 'Cat'
    },
    ...
  ]
}

let newProfile = {...profile}
newProfile.friends.push({id: 3243, name: 'Dog'})
render(newProfile)
function render(profile) {
  // profile和_lastProfile不是同一个引用
  if (profile === _lastProfile)
    return
  ...
}
```

这回 `===` 可以工作了。

# 性能

到这里我们可能有个疑问：每更新一次对象我们就要深拷贝一次，对象很大的时候性能会不会有问题？的确，当我们的对象包含很多属性时，比如 todos 对象包含 10000 个对象时，Object.assign的性能会有明显的下降：

```javascript
let o = {}, s = {}
for (let i = 0; i < 100000; i++) {
  o[i] = i
}
for (let i = 0; i < 10000; i++) {
  o[i] = i
}
console.time('mutable 100000')
o['100001'] = 100001
console.timeEnd('mutable 100000')
console.time('Object.assign 100000')
let o2 = Object.assign({}, o)
o2['100001'] = 100001
console.timeEnd('Object.assign 100000')
console.time('mutable 10000')
s['10001'] = 10001
console.timeEnd('mutable 10000')
console.time('Object.assign 10000')
let s2 = Object.assign({}, s)
o2['10001'] = 10001
console.timeEnd('Object.assign 10000')
```

这里，我们分别对比直接修改、`Object.assign` ，对包含10000、100000属性的对象操作消耗的时间，结果如下：

![性能](https://raw.githubusercontent.com/leozcx/leozcx.github.io/master/images/immutable1.png)

可以看到，对于10000个属性的对象，`Object.assign` 性能虽有所下降，但降的并不多；当属性达到100000时，有明显的下降。

那么，怎么才能达到既要immutable，又要性能呢？

# immutable-js

immutable-js是Facebook开源的一款实现了immutable特性的库，它通过巧妙的方式，在imutable和性能之间达到一种平衡： [Persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure)

## Persistent data structure

> In computing, a persistent data structure is a data structure that always preserves the previous version of itself when it is modified. Such data structures are effectively immutable, as their operations do not (visibly) update the structure in-place, but instead always yield a new updated structure.
	
简单来说，在做拷贝时，并不需要拷贝所有的属性，只需要拷贝发生变化的那些，不变的那些还可以被重用，这就是 structure sharing。

比如，我们有这样一个对象：

```javascript
const data = {
  to: 7,
  tea: 3,
  ted: 4,
  ten: 12,
  A: 15,
  i: 11,
  in: 5,
  inn: 9
}
```
我们用 [字典树Trie] (https://en.wikipedia.org/wiki/Trie) 这种数据结构保存它：
![字典树](https://raw.githubusercontent.com/leozcx/leozcx.github.io/master/images/trie.png)

对于属性获取，比如要获取属性 `ten`, 只需要按如下顺序 `t -> e -> n` 查找，即可找到值 12.

对于属性更新呢，比如 `data.tea = 14`, 我们要新建一颗树，注意观察，我们并不需要新建所有的结点，只需要新建 root, t, e, a，即可，其余的都可以重用：

![重用旧结点](https://raw.githubusercontent.com/leozcx/leozcx.github.io/master/images/trie2.png)

注意绿色的结点，就是需要新建的结点。

旧的结点可以很快被GC回收：

![GC回收旧结点](https://raw.githubusercontent.com/leozcx/leozcx.github.io/master/images/trie3.png)

这就是Immutable.Map的基本原理：每个结点有32个子结点，每个子结点为0表示尚未设值，为1表示有值，查找时先对属性进行hash，通过选择合适的hash函数，可以保证hash值在[0, 2^32 -1]之间，通过与11111做与运算，可以快速地确定属性应该放在哪一位上；如果当前位已经有值，那么就下钻一层，同时把hash值右移5位，重复同样的操作，直到找到的那个位置还没有设值，则把值放在那个位置。

![Immutalbe.Map](https://raw.githubusercontent.com/leozcx/leozcx.github.io/master/images/immutable3.png)

实际上当属性个数小于8个时，只用一个数组来保存各个属性。


# 总结

Immutable可以在一定程度上简化我们的开发，特别是与redux结果使用效果更佳。但是不是一定要用Immutable-js呢？未必。当数据量不大时，用 `Object.assin` 可以达到同样的效果，性能也没有太大的下降。当数据量很大时，immutable-js 可以很大程度上提高性能。但同时，它也带来一些不便：只能用immutable-js提供的数据结构。

# 参考资料

- [https://medium.com/@dtinth/immutable-js-persistent-data-structures-and-structural-sharing-6d163fbd73d2](https://medium.com/@dtinth/immutable-js-persistent-data-structures-and-structural-sharing-6d163fbd73d2)