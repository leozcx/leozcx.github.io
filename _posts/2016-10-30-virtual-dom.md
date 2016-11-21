---
layout: post
title: Virtual DOM 浅析 
author: Leo
---

不知道是不是React原创，但是Virtual DOM的确是React一大卖点：用Virtual DOM，效率就是高。vue 2.0以前，并没有用Virtual DOM，但一直说自己的渲染方式跟Virutal
DOM相比并不落下风，甚至在某些情况下要优于React的Virtual DOM; 但vue 2.0却完全重写，改为基于Virtual DOM；Virtual DOM的优势不言自明。Virtual DOM似乎是未来的方向，但什么是Virtual
DOM，它是怎么实现的，我却从没好好想想，现在就尝试学习一下Virtual DOM。

# Real DOM
W3C对于DOM的定义是：

> 文档对象模型是平台无关，语言无关的编程接口，编程语言可以利用它来访问、更新文档。

所以，DOM实际上是与编程语言无关的，虽然我们大多数时候都是用javascript来操作DOM，但我们应该知道，完全可以用其它语言来做同样的事，比如说Python；但绝大多数情况下我们是在浏览器环境下操作DOM，所有浏览器都内置javascript支持，所以当我们说javascript的时候，通常会默认包含DOM；但我们应该知道：javascript和DOM是2件事，2种不同的事物。

DOM提供了一系列的对象以及方法，用javascript可以很方便地调用这些方法，比如说我们经常使用的`getElementById()`，就是由对象`document`提供的，而`document`是实现了`Document`接口的对象。比较常见接口包括：

```
EventTarget <- Node <- Element <- HTMLElement <- HTMLDivElement
EventTarget <- Window
EventTarget <- Node <- Document
```

当我们用`document.createElement('div')`来创建一个div元素时，实际上创建了一个`HTMLDivElement`对象：该对象继承一它所有父接口的属性以及方法。

# Real DOM 的问题

## 创建慢

W3C标准规定了每个接口必须包含哪些属性，必须实现哪些方法，拿一个最常见的div元素来说，它包含了几十个属性，所以创建DOM包含了额外的开销：我们并不需要的很多属性也会被创建；下面的例子比较创建DOM和创建普通的js对象，可以看到创建js对象的效率要明显优于创建DOM:

<iframe width="100%" height="300" src="//jsfiddle.net/leozcx/n2gz27v9/embedded/js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

## 更新慢

上面的例子仅仅演示了创建DOM，并没有插入到文档中，所以创建慢表现的并不明显；在实际应用中，创建了DOM通常要插入到文档结构中，从而会影响到界面的更新：更新慢才是影响DOM操作的最大杀手。我们先看一下浏览器的工作流程.

简单来说，浏览器从收到server响应，到把页面展现在用户面前，大致需要经过以下几个步骤：

```
解析HTML生成DOM tree -> 生成Render tree -> Layout -> Paint
```

其中，DOM tree是整个HTMl文档的树状表示，文档中的每个元素都在树中有一个位置；DOM tree，再加上css, style，生成Render tree，相比DOM tree，Render
tree是用来指示浏览器做显示的，所以只包括可显示的元素，不可显示元素，比如<head>, `display:none` 的元素是不以Render tree中的；接下来是根据Render
tree计算每个元素的大小，以及应该出现的位置，称为“Layout”；最后加上颜色，设置字体等，即"Paint"。

当我们操作DOM时，有可能会触发上面中的若干步，比如：

- 生成一个新的元素并添加到文档中，需要更新DOM tree，从而触发后面的所有步骤；
- 更新某个元素的样式，会更新Render tree；如果该元素的大小、位置发生变化，则还要触发Layout, Paint；这个过程可能不光影响到该元素本身，还有可能影响到它的子结点，兄弟结点等；

因为整个过程很耗时，现代浏览器都会足够智能地进行优化：尽量减少影响范围，即只重新计算需要更新的部分；比如，一个input的值发生变化，重新计算的只是这个input本身；一个元素的background-color发生变化，只需要触发Paint，同时只Paint该元素本身；更改全局字体，则需要重新Layout,
Paint整个Render tree.

尽管浏览器很智能，开发者却不是那么智能；开发者经常会无意中触发没有必要的Layout, Paint, 比如

```javascript
for (var i = 0; i < 100; i++) {
  var tmp = document.createElement("div");
  var txt = document.createTextNode("t" + i);
  tmp.appendChild(txt);
  dom.appendChild(tmp);
}
```

上面的代码会触发100次layout, paint，很显然我们并不需要那么多。

Virtual DOM就是来帮我们解决这个问题的：Virtual Dom足够聪明，它知道界面上哪些需要更新，哪些不需要。

# Virtual DOM

Virtual DOM的基本思想是，尽可能少的DOM刷新。不用Virtual DOM的情况下，当我们的数据发生改变时，需要直接操作DOM去展现数据的变化；Virtual
DOM相当于在数据与DOM之间又增加了一层，数据的变化直接反应在Virtual DOM上，再由Virtual DOM来决定怎么去更新DOM。所以，本质上Virtual
DOM并没有加快DOM操作，该Layout还是要Layout，该Paint还是要Paint，但是它可以最大限度的减少不必要的刷新。那么怎么实现呢？

## 不过是object

前面提到过，DOM是实现了一系列接口的对象；Virtual DOM与此类似，不过是普通的javascript对象，只不过与DOM对象相比，它不需要包含那么多的属性。与DOM tree一样，Virtual
DOM也是以tree状结构存在，可以认为是DOM tree的镜像。纯javascript对象操作要比DOM对象操作快的多，所以虽然Virtual DOM属于新增加的一层，效率上并没有太大影响。

比如，我们可以定义这样一个Virtual DOM对象，与DOM中的Element对应：

```javascript
function VirtualNode(tagName, properties, children) {
  this.tagName = tagName; //tag name, e.g. 'div'
  this.properties = properties; //properties of the tage, e.g. style, class
  this.children = children; // children of the node
}
```

我们的对象很简陋，只包含最少的必须属性；这样的对象操作起很快。

## 从object到Real DOM

可以很方便地从上面的VirtualNode对象创建DOM对象，像下面这样：

```javascript
function createElement(vnode) {
  var ele = document.createElement(vnode.tageName);
  if(ele.properties) {
      for(var key in ele.properties) {
          ele.setAttribute(key, ele.properties[key]);
      }
  }
  if(vnode.children) {
      vnode.children.forEach((c) => {
        var cEle = createElement(c);
        ele.appendChild(cEle);
      });
  }
  return ele;
}
```

## 部分刷新

到现在为止，Virtual DOM并没有显示出它的威力，甚至效率感觉会更差：创建Virtual
Node的开销；但现代javascript引擎都非常高效，创建对象这类基本操作并不会有太大开销；同时带来了新的可能性：控制何时、以及如何创建DOM。设想一下，当数据改变时，首先更新Virtual
DOM，我们完全不用担心这一步的开销，纯对象的操作非常高效；同时我们就有了新、旧2个Virtual DOM tree；通过比较这2棵树的不同，有选择性地更新DOM，从而达到尽量少地操作DOM的效果。

## diff

比较任意2棵树的不同的时间复杂度是O(n^3)，效率很低；但对于DOM tree这个特定的场景，可以通过优化达到O(n)的时间复杂度，因为只需要按层比较:

![DOM tree diff](https://cl.ly/1T0A141P1E08/tree-diff.png)

通过遍历Virtual DOM tree可以得到所有发生变化的结点，这些变化是可以枚举的，无非是：

- 增加/减少结点
- 结点text发生变化
- 结点属性发生变化
- 结点顺序发生变化

用深度优先算法遍历原始树，同时与新树对应结点作比较，即可定义出发生的变化，是增加结点，还是结点的属性改变；针对不同的变化，我们只需要记录执行这个变化所需要的所有信息，就可以将变化apply到真正的DOM上。比如，针对减少结点，我们需要将结点从DOM tree中删除，只需要记录：

- 类型：结点删除
- 结点：结点对象

即可以做对应的操作；

针对属性变化，常见的class发生变化，我们需要将新的class应用到对应的结点上，只需要记录：

- 类型：属性变化
- 结点：结点对象
- 其它：属性名，值

通过执行diff过程，我们可以得到一个列表，该列表中包括所有发生的变化，以及应用这些变化所需要的信息，即可通过patch操作反应到真实DOM tree上。

## patch

patch是Virtual
DOM整个流程中真正操作DOM的部分，针对不同的变化类型，进行不同的patch操作。比如，如果是增加结点，只需要调用`node.appendChild()`；删除结点则需要调用`node.removeChild()`；对于属性更新，可以删除不存在的属性，增加新增的属性；我们在diff阶段已经记录了所有需要的信息，所以可以很方便的生成新的DOM tree。

# 小结

这些仅是Virtual DOM的基本思想，实际应用需要考虑更多细节的因素，比如更高效的diff算法，如何判断结点顺序发生变化，如何patch等等；接下来有时间分析一下Matt-Esch/virtual-dom的实现。

# 参考资料
1. [https://github.com/Matt-Esch/virtual-dom](https://github.com/Matt-Esch/virtual-dom)
2. [http://calendar.perfplanet.com/2013/diff/](http://calendar.perfplanet.com/2013/diff/)
