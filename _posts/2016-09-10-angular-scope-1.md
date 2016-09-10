---
layout: post
title: angular scope 简介 - 1
author: Leo
---

在使用angularjs过程中最经常打交道的一个概念恐怕就是scope了，不深入理解scope，在开发过程中就容易遇到一些莫名其妙的问题，比如在scope更改了一个变量，却没有在view上实时更新，这篇文章作为自己对scope理解的一个梳理。

# 什么是scope?
> Scope is an object that refers to the application model. It is an execution context for expressions. Scopes are arranged in hierarchical structure which mimic the DOM structure of the application. Scopes can watch expressions and propagate events.

说白了，scope就是一个js对象，作为view和controller之间数据交换的桥梁；view要计算某个表达式时，形如{{name}}，就会从scope里找名为name的属性（scope.name)；当scope.name发生变化时，变化会自动反映到view中，反之亦然。这就是著名的two-way data binding：我们不再需要手工写onChange去更新js中的数据，也不需要用innerHTML或是.value去更新html；这些angular自动帮我们做了。那么这是怎么做到的呢？

这篇文章作为系列1，先总结一下scope的基本用法。

# scope创建
在应用程序启动时，angularjs会创建一个初始的scope：$rootScope；随着“编译”阶段——遍历DOM树，查找所有的directive——的进行，新的scope被不断地创建。编译结束后，angular会创建一个scope树，有点类似于DOM树，子scope会继承父scope的属性；每个scope会被绑定到相应的DOM结点上：$rootScope被绑定到含有ng-app的DOM结点，以此类推。可以用Chrome查看某个结点对应的scope:

1. 在元素上右击，选择“检查”；
2. 在开发者工具中，选择"Console"，输入`$scope`，即可打印出选中结点对应的scope；
3. 可以看到，scope中包含一些$开头的属性，这是angular的约定，表示是由angular自己添加使用的，我们自己的属性不应该以$开头；所有$开头的属性中，有一个$parent，用来指向父scope；所以如果想要引用父scope，只需要类似于`scope.$parent`即可。

# scope继承
前面说到每个scope中都有一个属性$parent指向父scope，这说明scope是有继承关系的。$rootScope作为所有scope的父亲，其它scope都会继承它的属性，所以如果想在不同的controller之间共享数据，把数据放在$rootScope中是方法之一。

angular使用的是js的原型链实现的继承，即创建scope时，把父scope赋值给子scope的__proto__属性，这样就“相当于”子scope拥有了父scope的所有属性。说是“相当于”是因为，这跟真正拥有还不一样，它的实现是：当访问子scope的某个属性时，先查找该子scope的所有属性，注意，这并不包含从父scope继承过来的；如果没有找到，再接着找父scope的属性，直到找到，或父scope为null为止；而当给子scope属性赋值时，如`scope.message="child"`，实际上是创建了子scope的message属性（如果之前没有），而父scope的message属性是不变的。如果不注意这点差别，就容易出现莫名其妙的问题，像下面的代码：

<iframe width="100%" height="220" src="//jsfiddle.net/leozcx/b77xjo99/embedded/html,result,js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

乍一看，当你更新input的值时，上面的值也应该跟着变，因为子scope继承了父scope的message属性，但实际上并非如此，大家可以试试看。

这是因为，当angular编译时，它会在ng-if的scope中查找message属性，没有，继续向上查找父scope，找到，把它的值显示在view上，所以开始的时候input是有值的；但更新的时候，更新的是ng-if的scope，相当于执行了

```
$scope.message='xxx';
```
这实际上是子scope上创建了一个新的属性message，而不会影响父scope，所以上边的值不会跟着变。

这里顺便说一下解决方法，有几种：

1. 不用`ng-if`，改为`ng-show`. 这是因为，`ng-show`并没有创建新的scope，更新input时更新的就是同一个scope的message。
2. 不要使用主类型string，而用object，如下所示。这是因为，子scope和父scope引用的同一个object，在哪更改都会更新唯一的一个object。
<iframe width="100%" height="220" src="//jsfiddle.net/leozcx/s01rw2q6/1/embedded/html,result,js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

# isolating scopes
scope继承可以让我们不必重复定义一些属性，这正是大多时候我们所需要的。但有时候这样反而会造成麻烦，特别是在可复用的组件(directive)这个场景中，如果组件的scope是继承自父scope，那么就要求父scope必须定义某些属性，这对父scope来说有可能是不必要的；同时，如果同一个页面有多个相同组件，相互之间也会受到干扰，像下面这种情况：
<iframe width="100%" height="300" src="//jsfiddle.net/leozcx/e70c649r/1/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
这种情况下就需要isolating scope，即不会从父scope继承任何属性:
<iframe width="100%" height="300" src="//jsfiddle.net/leozcx/983yL0xt/1/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
通过指定`scope`属性，新的directive将会创建一个新的isolating scope，不会从父scope继承任何属性。
但我们还可能有需要，从父scope传入某些参数，我们可以通过在scope属性中指定新的属性，像下面这样：
<iframe width="100%" height="300" src="//jsfiddle.net/leozcx/voa1mhtv/1/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

# 总结
这种文章简单总结了angular中scope的相关概念及常规用法，接下来将重点看一下scope是如何实现two-way data binding的。

# 参考文献
- [AngularJS Scopes: An Introduction](http://blog.carbonfive.com/2014/02/11/angularjs-scopes-an-introduction/)
- [https://docs.angularjs.org/guide/scope](https://docs.angularjs.org/guide/scope)
