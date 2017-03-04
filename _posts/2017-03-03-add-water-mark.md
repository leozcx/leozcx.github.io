---
layout: post
title: 给网页加水印
author: Leo
---

为防止信息泄露，给网页加水印是一种常见的方法；本篇文章介绍一种添加水印的方法，经测试效果不错；特点：

- 不影响现有代码

- 可以任意给网页的不同部分添加水印

- 纯前端js实现

- 可简单防止用户通过浏览器开发者工具隐藏水印

## 实现

代码地址：https://github.com/leozcx/watermark.git

最终效果如下：

<iframe width="100%" height="300" src="//jsfiddle.net/leozcx/aoessb24/1/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

### 生成水印

#### 1. 生成图片
水印的特点是，包含一段标识信息，同时需要覆盖足够的区域，这很自然想到用background，指定image，并让它在x,y 2个方向上重复展示；那么问题是：如何生成image呢？

简单调研，前端生成image主要用到canvas；canvas标签是HTML5新增的元素，主要作用是支持用js画图；canvas是个很强大的标签，有关canvas的详细介绍可以移步[这里] (https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API). 用canvas把信息画成图之后，调用`toDataURL()`方法就可以得到一个url，该url实际包含了Base64过的图像信息，可以直接用在background上。

代码比较简单，演示如下：

```javascript
function createImageUrl(options) {
  var canvas = document.createElement('canvas');
  var text = options.text;
  canvas.width = options.width
  canvas.height = options.height
  var ctx = canvas.getContext('2d');
  ctx.shadowOffsetX = 2;     //X轴阴影距离，负值表示往上，正值表示往下
  ctx.shadowOffsetY = 2;     //Y轴阴影距离，负值表示往左，正值表示往右
  ctx.shadowBlur = 2;     //阴影的模糊程度
  // ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';    //阴影颜色
  ctx.font = options.font
  ctx.fillStyle = "rgba(204,204,204,0.45)"
  ctx.rotate(options.rotateDegree);
  ctx.translate(options.trax, options.tray);
  ctx.textAlign = 'left';
  ctx.fillText(text, 35, 32);    //实体文字
  return canvas.toDataURL('image/png')
}
```

#### 2. 设置背景

有了图片之后，只要把图片放在某个div的background上就行了；这里只需要注意几个要点：

- position: fixed； 这样可以保证不管内容如何滚动，水印都能显示；
- pointer-events: none; 阻止水印影响页面内容
- 动态计算出水印的位置

代码示例：

```javascript
function createContainer(options, forceCreate) {
  let oldDiv = document.getElementById(options.id)
  if(!forceCreate && oldDiv)
    return container
  let url = createImageUrl(options)
  var div = oldDiv ? oldDiv : document.createElement('div');
  div.id = options.id
  let parentEl = options.preventTamper ? document.body : (options.parentEl || document.body)
  if(typeof parentEl === 'string') {
    if(parentEl.startsWith('#'))
      parentEl = parentEl.substring(1)
    parentEl = document.getElementById(parentEl)
  }
  let rect = parentEl.getBoundingClientRect()
  options.style.left = (options.left || rect.left) + 'px'
  options.style.top = (options.top ||rect.top) + 'px'
  div.style.cssText = getStyleText(options)
  div.setAttribute('class', '')
  backgroundUrl =  'url(' + url + ') repeat top left';
  div.style.background = backgroundUrl
  !oldDiv && parentEl.appendChild(div)
  return div
}
```

### 防止客户端篡改

好了，现在水印可以显示了；但这样只能防止小白用户，稍微有点技术的用户就知道，可以用浏览器的开发者工具来动态更改dom，比如display: none；就可以隐藏水印；所以还需要加一点机制防止用户进行篡改；当然，从本质上来说是没有绝对的办法在客户端去防用户的，所以这里的trick只是增加了用户篡改的难度。

一种思路是监测水印div的变化，一旦发生变化，则重新生成水印；沿着这个思路往下走，如何监测水印div变化呢？

#### 1. 不间断比较div的值

开始采用的是这种方法，记录刚生成的div的innerHTML，每隔几秒就取一次新的值，通过比较2者的md5，如果发生变化则重新生成。但这个方法有几个缺点：

- 滞后性，修改不能马上被监测后；而如果间隔时间过短，则可能影响性能；
- 生成md5也有不小的开销，特别是打开多个页面的时候；

所以这种方法不可行。

#### 2. MutationObserver

通过查询文档，发现浏览器提供了一种监测元素变化的API: (MutationObserver) [https://developer.mozilla.org/en/docs/Web/API/MutationObserver], 并且主流浏览器都支持，所以改用这种方式监测；代码示例：

```javascript
function observe(options, observeBody) {
  let target = container
  observer = new MutationObserver(function(mutations) {
    observer.disconnect()
    container = createContainer(options, true)
    var config = { attributes: true, childList: true, characterData: true, subtree:true };
    observer.observe(target, config);
  });
  var config = { attributes: true, childList: true, characterData: true, subtree:true };
  observer.observe(target, config);

  //observe body element, recreate if the element is deleted
  var pObserver = new MutationObserver(function(mutations) {
    mutations.forEach(function(m) {
      if(m.type === 'childList' && m.removedNodes.length > 0) {
        let watermarkNodeRemoved = false
        for(let n of m.removedNodes) {
          if(n.id === options.id) {
            watermarkNodeRemoved = true
          }
        }
        container = createContainer(options)
        observe(options, false)
      }
    })
  }); 
  pObserver.observe(document.body, {childList: true,subtree:true});
}
```

这里有一点需要注意，MutationObserver只能监测到诸如属性改变、增删子结点等，对于自己本身被删除，是没有办法的；这个可以通过一个小trick来解决：同时监测父结点，看div是否被删除。
