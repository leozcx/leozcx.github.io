---
layout: post
title: js以及Unicode
author: Leo
---

说实话，我一直很害怕处理编码集的问题，以前写jsp，或者写Java时，动不动就会出现中文乱码；这时候的解决方案，也是随手百度一下，通过会搜出若干方案，然后一个一个地试；通常情况下总有一款能解决问题。之所以没有彻底搞懂，是因为觉得编码集很复杂，什么Unicode, utf-8, utf-16, gb2312, gbk... 今天闲来无事，决心把它彻底搞懂，结果发现，其实并没有想像地那么复杂。这篇文件作为学习笔记，以便日后过来翻看；也希望跟我有同样困惑的人看到以后，可以有原来如此的释放感。

本文先从Unicode开始，顺便介绍容易混淆的几个概念，比如utf-8, gb2312等。

## 什么是Unicode

> Unicode给每个字符提供了一个唯一的数字，
> 
> 不论是什么平台、
>
> 不论是什么程序、
>
> 不论是什么语言。

简单来说，Unicode定义了数字，以及数字与字符之间的对应关系。因为，计算机是不认识字符的，它不能理解一个字符到底是英文小写字母 'a'，还是汉字 '好'; 它认识的只有二进制数字；计算机之所以能够在屏幕上显示正确的字符，是因为有这样一种规则存在：这个数字应该显示成什么字符。

实际上在Unicode之前，已经存在上百种这样的“规则”，我们称之为字符集，不同国家，不同地区都可能有不同的字符集，这造成的后果是：同一个数字在字符集A中是这个字符，在字符集B中又是另外一个字符；或者根本就没有这个字符；这无疑给信息的传播带来不便。Unicode应运而生，旨在为全世界字符提供一个统一的规则。关于Unicode历史可参见参考资料。

先来看几个Unicode例子：

- A -> U+0041 (1000001)
- 好 -> U+597D （101100101111101）
- !  -> U+0021  (100001)

方便起见，我们用16进制表示数字。

实际上，汉字的编码范围是4E00 - 9FD5；英文字符（拉丁字符）的编码范围是 0000 - 007F；这里不得不提一下西方语系在计算机世界的天然优越性：只需要这么几个编码就可以表示所有的单词，而汉语却需要几万个。

## 基本概念

### Characters and Code Point

为了避免混淆，以下部分名词直接用英文表示。

> **Abstract Character**: A unit of information used for the organization, control, or representation of textual data.
 
Abstract Character: 用来组织、控制或表示文字的基本信息单元。比如字母'a'，标点逗号','，汉字'好'... 每个abstract character都有名字，比如 LATIN SMALL LETTER A...

> **Code point** is a number assigned to a single character.

Code point就是我们所说的数字了，通常用 `U+<hex>` 来表示：`U+`表示Unicode; `<hex>`表示16进制数字。

在上面我们看到了拉丁文及汉字的Code point范围，整个Unicode的Code point范围是：U+0000 - U+10FFFF.

Unicode又可以看成是Code point到 Character的映射。并不是所有在上面范围内的数字都有对应的Character，0000 - 10FFFF 之间共有1,114,112个数字，但目前只有128,237个有character映射。

### Plane

> **Plane** is a range of 65,536 contiguous Unicode code points from U+n0000 up to U+nFFFF, where n can take values from 0 to 16.

Plane把Code point又进一步分成17个范围：

- Plane 0: U+0000 - U+FFFF
- Plane 1: U+**1**0000 - U+**1**FFFF
- ...
- Plane 16: U+**10**0000 - U+**10**FFFF

其中Plane 0称为 *Basic Multilangual Plane(BMP)*, 剩下的16个称为 *Astral Plane*.

BMP包括大多数语言的字符，以及常用的符号，像中文，拉丁文都在这个Plane里。

Astral Plane，又被称为 Supplementary Plane，包括更多的符号，但不常用。

### Code unit

> **Code unit** is a bit sequence used to encode each character within a given encoding form.

上面提到的**abstracter character**以及**code point**都是上层的概念，要落实到计算机的存储上，还需要另外一个概念：**Code unit**.

字符集编码就是将**code point**转化成**code unit**的过程。

常见编码有utf-8, utf-16, utf-32.

多数的js引擎用的是utf-16编码。

utf-16全称是"16-bit Unicode Transformation Format"，它的规则是：
- BPM中的code points用16位存储，即1个code unit；
- Astral Place中的code points用2个code units.

举个例子，要存储 LATIN SMALL LETTER A(a)，它的code point是U+0061，在BMP范围内，所以用1个code unit即16位来存储：0x0061.

### Surrogate pairs

> **Surrogate pair** is a representation for a single abstract character that consists of a sequence of code units of two 16-bit code units, where the first value of the pair is a **high-surrogate** code unit and the second value is a **low-surrogate** code unit.

BPM范围内的code points的编码规则很简单，它们的数字是相等的；对于Astral Plane呢？用2个code unit。对。但问题出现了，计算机如何知道当前这个code unit是处于BPM范围中的code point，还是和下面一个code unit共同代表一个Astral Plane code point呢？答案就是Surrogate pairs。

utf-16并非简单的把Astral Plane code point直接转换成2个16位，而是做一次转换: utf-16规定了2个特殊范围，它们独立时不代表任何的code point，只能与后面的一个code unit共同使用；这2个特殊范围是：

- U+D800—U+DBFF (1,024 code points): high surrogates
- U+DC00—U+DFFF (1,024 code points): low surrogates

所以Astral Plane code point需要转化成一对surrogate pair: 比如U+1F600会转化成\uD83D\uDE00 

转换代码可以用如下代码表示：

```javascript
function getSurrogatePair(astralCodePoint) {  
  let highSurrogate = 
     Math.floor((astralCodePoint - 0x10000) / 0x400) + 0xD800;
  let lowSurrogate = (astralCodePoint - 0x10000) % 0x400 + 0xDC00;
  return [highSurrogate, lowSurrogate];
}
getSurrogatePair(0x1F600); // => [0xDC00, 0xDFFF]

function getAstralCodePoint(highSurrogate, lowSurrogate) {  
  return (highSurrogate - 0xD800) * 0x400 
      + lowSurrogate - 0xDC00 + 0x10000;
}
getAstralCodePoint(0xD83D, 0xDE00); // => 0x1F600 
```

### 组合符号

> A **grapheme**, or **symbol**, is a minimally distinctive unit of writing in the context of a particular writing system.

**grapheme** 是用户直观上感受到的*+*一个**符号，显示在屏幕上被称为一个**glyph**.

多数情况下，一个Unicode character代表一个glyph，比如'LATIN SMALL LETTER F'的unicode是 U+0066，显示在屏幕上是'f'：一个字符。

但还存在另外的情况，比如这个字符：&#xe5; 。 从直观的视觉上来看，这是一个字符，但实际上它是由2个unicode字符组合而成：字符a以及上面的圆圈：U+0061以及U+030A.

U+030A修改了前面的字符，称为**combining mark**

> **Combining mark** is a character that applies to the precedent base character to create a grapheme.

**Combining mark** 通常不会独立出现，它需要与前面的字符共同组成一个有意义的字符。

## JS中的Unicode

对Unicode有了一个基本认识之后，我们来看一下js中的Unicode.

[es6 规范](http://www.ecma-international.org/ecma-262/6.0/#sec-source-text) 指出：

> ECMAScript code is expressed using Unicode, version 5.1 or later. ECMAScript source text is a sequence of code points. All Unicode code point values from U+0000 to U+10FFFF, including surrogate code points, may occur in source text where permitted by the ECMAScript grammars.

也就是说，js代码是也可用任意的Unicode字符书写，只要符合语法。但通常情况下，我们要用BMP中的字符，不要搞稀奇古怪的字符。

es6对于String的[规范](http://www.ecma-international.org/ecma-262/6.0/#sec-ecmascript-language-types-string-type)指出:

> The String type is the set of all ordered sequences of zero or more 16-bit unsigned integer values (“elements”) up to a maximum length of 2^53-1 elements. The String type is generally used to represent textual data in a running ECMAScript program, in which case each element in the String is treated as a **UTF-16 code unit** value. Each element is regarded as occupying a position within the sequence. These positions are indexed with nonnegative integers. The first element (if any) is at index 0, the next element (if any) at index 1, and so on. The **length** of a String is the number of elements (i.e., 16-bit values) within it. The empty String has length zero and therefore contains no elements.

注意上面对于length的定义，每个元素是以16-bit为单位的；这对于BMP来说是符合我们的直观感受的，但对于非BMP，则未必：
<iframe width="100%" height="220" src="//jsfiddle.net/leozcx/vz7gwqmm/embedded/js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

在上面的例子中，笑脸在我们的直观感受中是一个字符，但由于它包括2个code unit，它的length是2.

所以当我们思考string时，最好把它当成是**code unit**的序列，而跟它是如何在屏幕上显示的无关。

### 字符串比较

根据上面的规范，字符串是code unit的序列，那么跟它是如何书写的无关，所以会有如下等式：
<iframe width="100%" height="130" src="//jsfiddle.net/leozcx/jb3yg0zv/1/embedded/js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

这2个字符串看起来相同，实际上也相同。但情况并不总是这样，看下面这个例子：
<iframe width="100%" height="180" src="//jsfiddle.net/leozcx/4Lgtx81o/3/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

虽然在屏幕上看起来一样，但其实它们用的不是相同的unit code:

- str1用的是U+00E7 LATIN SMALL LETTER C WITH CEDILLA
- str2用的是combining mark: U+0063 LATIN SMALL LETTER C 再加上 combining mark U+0327 COMBINING CEDILLA

如何处理这种情况呢？我们希望看起来一样的字符串，比较起来也一样，这需要用**Normalization**

### Normalization

> **Normalization** is a string conversion to a canonical representation, to ensure that canonical-equivalent (and/or compatibility-equivalent) strings have unique representations.

即，把字符串转化成统一的权威的表示。

在js中用`str.normalize()`方法可以实现，所以上面的例子normalize后有如下结果：

<iframe width="100%" height="180" src="//jsfiddle.net/leozcx/4Lgtx81o/5/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

## 概念澄清

讲到这里，相信大家对开头提到的几个概念应该有清晰的理解了：

- Unicode, GB2312, GBK, ISO 8859-1, ASCII 都是不同的字符集，它们字义了数字到字符的映射关系，可以称为**charsets**
- utf-8, utf-16 定义的是数字如何存储，它们是Unicode对于存储的实现方式，可以称为**encodings**

## 参数文件
[http://unicode.org/] (http://unicode.org/)

[https://rainsoft.io/what-every-javascript-developer-should-know-about-unicode/] (https://rainsoft.io/what-every-javascript-developer-should-know-about-unicode/)

[http://unicodebook.readthedocs.io/unicode.html](http://unicodebook.readthedocs.io/unicode.html)