---
layout: post
title: ES6 箭头函数
author: Leo
---

  箭头函数(Arrow Functions)是ES6引入的一个新的功能，它有2个作用：

1. 简化代码
2. 区别于传统函数的this

  先看下面的例子：

```javascript
//传统方式定义函数
var addition = function(a, b) {
    return a + b;
}
//箭头函数
let addition = (a, b) => a + b;
```
  可以看到，原来需要3行定义一个函数，用箭头函数的话只需要1行，极大减少了代码量。

# 语法

## 基本语法

```javascript
  (param1, param2, ..., paramN) => { statements }
  (param1, param2, ..., paramN) => expression //等同于 -> { return expression } 
  //如果只有一个参数，括号可以省略
  singleParma => { statements }
  //如果没有参数，需要一个空括号
  () => { statements }
```

## 高级语法

```javascript
  //如果返回值是一个object literal，需要用括号括起来
  params => ({foo: bar})
  //参数支持rest参数及默认参数
  (param1, param2, ...rest) => { statements }
  (param1='value1', param2, ..., paramN='valueN') => { statements }
  //参数支持解构赋值(destructuring assignment)
  var f = ([a, b] = [1, 2], {x:c} = {x: a + b}) => a + b + c;
  f(); //6: a = 1, b = 2, c = 3
```

  javascript的箭头函数用的是“胖箭头”(=>), 区别于java8 Lambada表达式采用的“瘦箭头”(->)。

# this

  先看一个例子：
  
```javascript
function bar() {
  setTimeout(function() {
    console.log(this.a);
  }, 100)
}
var obj = {a: 2};
bar.call(obj);
```
大家可以先思考一下，上面代码执行的结果是什么呢？

undefined.

并没有输出“2”，而是"undefined"。这是因为，在普通函数中，this指向的是调用函数的对象，这里输出语句的函数是被window调用的，而window对象中并没有a这个属性，所以会输出"undefined"。

再看一下如果换成箭头函数会是什么结果：

```javascript
function foo() {
  setTimeout(() => {
    console.log(this.a);
  }, 100);
}
var obj = {a: 2};
foo.call(obj);
```
这回输出的结果是"2"。

箭头函数则会捕获其所在上下文的  this 值，作为自己的 this 值，因此上面的代码中this指向的是foo.

在箭头函数出现之前我们通常用以下方法解决this的问题：

```javascript
function bar() {
  var self = this;
  setTimeout(function() {
    console.log(self.a);
  }, 100)
}
var obj = {a: 2};
bar.call(obj);
```
即把this赋值给额外的一个变量self，并将变量放在闭包中；而用箭头函数则可以更轻松解决这个问题。

# 小结

  代码简洁是程序员追求的终极目标，箭头函数确实会缩减代码量；同时又提供了额外的this功能；所以推荐大家使用。

# 参考文献
- [https://github.com/metagrover/ES6-for-humans#2-arrow-functions](https://github.com/metagrover/ES6-for-humans#2-arrow-functions)
- [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) 
- [https://seanlin0800.gitbooks.io/types-grammar/content/source/ch2/lexical_this.html](https://seanlin0800.gitbooks.io/types-grammar/content/source/ch2/lexical_this.html)
