## 前言
在前端开发的过程中，经常会看到浏览器在做一个`Recalculate Style`的任务。https://codepen.io/TGuoW/pen/GRNzeZv 在这个页面内点击按钮，录制profile，便会发现，仅仅给main元素增加了一个完全不影响样式的class，竟然会影响到1001个节点。而在我们的预期中，增加这个class，只会影响main元素自身，而不会影响到它的子节点。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/878d3755e3184c32bcc3e928cf8bd45d~tplv-k3u1fbpfcp-watermark.image)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab0238f629354ebe9c18c7169dcbd2a3~tplv-k3u1fbpfcp-watermark.image)

在实例代码html层级结构如此简单的场景下，增加一个无关class，耗时约2ms。在业务开发时，会遇到非常多的更加复杂的场景，耗时可能会突破毫秒级到达秒级。因此，了解chrome为什么会重计算这么多节点，有助于我们写出高性能的css。

**css优化方式网络上已经很多文章了，本文的重点是从原理上去解释为什么这些css写法会导致性能降低**

文章基于chromium的官方文档，约3500字。笔者难免有理解错误或者笔误的地方，欢迎指出。

## 先说结论
1. 尽量避免一个复合选择器中的高优先级选择器匹配到页面中较多的元素
2. 避免使用属性选择器和标签选择器
3. 避免使用否定选择器
4. 避免使用兄弟元素选择器
5. css module或者bem可以 **一定程度** 上避免`Recalculate Style`的性能问题。这些情况下，一般我们的css选择器不会太深，一个选择器也很少会匹配到页面上比较多的不应该匹配到的元素（当然，如果你硬是匹配到了我也没办法）

## 样式失效标记
首先，我们要理解为什么需要`Recalculate Style`，重计算样式？很简单，因为dom操作过后，某些元素样式发生了改变。

浏览器要做的，就是在元素样式发生改变后，能够正确的重新计算这些元素的样式，这样才能使页面表现正常。

假想一下，如果开发者操作一个dom之后，文档中所有的dom都重计算，那么就可以保证每个dom的样式信息都是正确的。

但基于性能考虑，不可能这么干。

因此，在样式重新计算的过程中，chrome做了一个操作，先把所有的需要重计算的元素给标记出来。

## 样式重计算整体过程

1. 开发者对dom进行了某些操作，比如修改dom，增删dom等
2. chrome标记样式可能失效的元素
3. 对所有被标记的元素重计算

基于chrome官方的说法，第二步和第三步的耗时基本相等。那么我们可以得到一个结论，减少被标记的失效元素，就能减少`Recalculate Style`的耗时。


## 树结构的更改导致样式失效

比如插入了一个节点，chrome的处理比较简单粗暴，会重新计算它的所有子节点的样式，并且由于兄弟选择器等的存在，可能会重新计算它的兄弟元素的样式。

举个例子，存在以下css
```
.a、.a * 、.a〜* 、. a〜* * {}
````

在元素上将class属性设置为“ a”时，第一个选择器将选择元素本身，第二个选择器将选择元素的所有后代，第三个选择器将其所有兄弟姐妹，第四个选择器将其所有兄弟姐妹的子孙。 因此，必须重新计算所有这些后代和兄弟姐妹的样式。 

以上是dom结构导致的style 重计算，不是本文的重点。本文重点在于研究dom 属性的改变导致 后代元素和兄弟元素 样式重计算的原理。核心概念有两个： **特征** 和 **失效集**， 失效集又分为**后代失效集**和**兄弟元素失效集**。

## 特征（feature）
我们所写的每一条css，都是有特征的。特征从css规则的最右选择器中提取出来。

举个简单例子
```
// 这条css规则的特征是.b
.a .b {}
```
当例子复杂一点，最右选择器是一个复合选择器
```
// 这条css规则的特征是#b
.a #b.b {}
```
那么问题就来了，为什么特征是`#b`而不是`.b`呢？顾名思义，特征（feature）即是一个选择器中最明显的部分，是有优先级的，在一个复合选择器中，id选择器比class选择器特征更明显。

选择器的优先级顺序如下
1. id选择器
2. class选择器
3. 属性选择器
4. 元素选择器

我们来两个demo验证一下这个优先级。

demo1中（链接：https://codepen.io/TGuoW/pen/OJbGPxd ），页面在一秒后给根div加上了main这个class，而css规则中非常明确的只会匹配第二个div，但是最终浏览器的`Recalculate Style`却重计算了两个元素。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02a6735326654429aa38f23c8e79f182~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ee546246ea349ea91b4ae582bc2d08d~tplv-k3u1fbpfcp-watermark.image)

demo2，我们修改一下demo，让页面中存在两个 class 为 b 的 div（链接：https://codepen.io/TGuoW/pen/vYyMErz ）。神奇的事情发生了，这一次chrome只重计算了一个元素。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a53ec64be2434327abe8b364d2ff9c51~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b77190c9eef645668be528fccf2dbb24~tplv-k3u1fbpfcp-watermark.image)

综上，说明了在id选择器和class选择器同时存在的复合选择器中，浏览器以id选择器为准，即id选择器的优先级比class选择器更高（其它优先级可通过类似demo自行验证）。因此，我们可以得到一个优化的tip：**尽量避免一个复合选择器中的高优先级选择器匹配到页面中较多的元素**，比如这种选择器写法`#a > span{}`，会导致匹配到`#a`下面所有的`span`，而不是仅仅是儿子`span`。

## 失效集
先说一下为什么会存在失效集这种东西。因为上面我们提到，样式重计算过程中需要标记元素，那么怎么找到哪些元素需要标记？总不能每次都重新遍历一次css规则吧。因此就需要一个失效集，指明这个元素的某个属性更改了之后会影响到哪些元素。

在chrome的实现中，一开始，会去遍历所有的css样式来计算它的失效集。

下面我们一一介绍 后代元素失效集 和 同级元素失效集。为什么没有 祖先元素失效集 呢？因为css没有祖先选择器这种东西。
### 后代元素失效集（Descendant Invalidation Sets）

后代元素失效集的定义是

1. 给定元素E，并且在E上修改了属性P。当E的后代F中的一个属性是在P的后代失效集中，那么F需要重新计算。这里的P指元素id、元素class和其它attribute。
2. 当P的后代失效集为空，那么在元素E上修改属性P，只会重新计算元素E的样式。
3. 在后代失效集上存在一个flag，`wholeSubtreeInvalid`，顾名思义，这个flag为true时，E的所有后代都会重新计算样式。
4. 在后代失效集上存在一个flag，`treeBoundaryCrossing`，如果P的修改会导致shadow dom中的元素样式失效，这个flag设置为true。
5. 在后代失效集上存在一个flag，`insertPointCrossing`，如果P的简单选择器右边有:: content伪元素，或者P的简单选择器位于：host或：host-context伪类中，则在P的集合上设置insertPointCrossing标志。

后代元素失效集类似于一个`key-value`的结构，key值是一条css规则中的非最右选择器，value就是我们上面提到的特征（feature）的集合。

那么举一些例子，让我们容易去理解后代失效集的生成规则：

#### 例子一
这个很简单，选择器只选中了类名中含有a的元素，那么它的后代失效集为空。为空的意思就是，开发者在一个dom上新增或者删除class a，不会影响到这个dom的后代。
```
// css
.a { }
// Invalidation Sets
.a { }
```

#### 例子二
这里同样很简单，css中选择了`.a`中的后代`.b`，当class a发生变化时，比如`addClass('a')`或者`removeClass('a')`都会导致类`a`的后代元素中带有类`b`的样式重新计算。因此`.a`的后代无效集中有`.b`，并且class b发生变化不会导致任何后代发生变化，所以`.b`的后代属性集为空
```
// css
.a .b { }

// Invalidation sets
.a { .b }
.b { }
```

#### 例子三
这种情况下，我们会发现，`.a`和`.b`的失效集都是`{ .c }`。为什么`.a`的失效集是`{ .c }`，跟`.b`完全没有关联？**个人猜测**，一方面如果要跟`.b`关联起来，需要做更多的工作，性能可能还不如把`.a`下面所有的`.b` 都计算一遍；另一方面，这种情况下，宁可杀错也不放过，重计算`.a`下面所有的`.b`，可以保证`.b`的样式绝对是正确的。
```
// css
.a .b .c { }

// Invalidation sets
.a { .c }
.b { .c }
.c { }
```

#### 例子四

```
// css
#x * { }
#x .a { }
.a :not(.b) { }

Invalidation sets:

.a { * }
#x { *, .a }
```
经过上面的例子一和二，我们可以理解为什么#x的后代无效是`{ *, .a }`，但是为什么`.a`的后代无效集是`{*}`？有兴趣的朋友可以在https://codepen.io/TGuoW/pen/QWGYXpJ 这里点击按钮，css规则如下，点击按钮前后，没有任何一个元素匹配了这个规则。但是会发现有1002个元素重计算了。


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c27e0c35bfb4279a7e4ed34e6d7e92b~tplv-k3u1fbpfcp-watermark.image)

> There are some simple selectors which are currently skipped and not added to the invalidation sets. One example is negated selectors. It is not impossible to implement, but we currently do not support negated members of invalidation sets.

chrome官方的解释是目前不支持否定选择器设置后代失效集，虽然可以实现，但我们就是不支持。。。

因此，**尽量避免`:not`选择器**。


### 同级元素失效集（Sibling Invalidation Set）
上面介绍的都属于亲子关系的元素失效集。那么当使用同级元素选择器时，比如`+`，`~`时会发生什么。当使用同级元素选择器时，同样不可能所有同级元素都重计算，因此需要一种规则去规定此时应该重计算哪些元素。

先举个例子，存在以下的css和html结构，不论移除了哪一个class，都会导致span的样式失效，需要重计算。但是在此过程中，不论是我们前端开发者还是浏览器开发者，都不希望`fieldSet`的样式被重计算，它的样式跟下面css规则没有关系，这时候需要一种类似于后代失效集的规则来描述哪些节点的样式会失效。
```
// css
.a + .b ~ .c { ... }

// html
<div class="a"></div>
<div class="b"></div>
<fieldset></fieldset>
<span class="c"></span>
```

#### 定义

一个同级元素失效集包含4个信息
1. 最大同级距离
2. 匹配的特征
3. flag。标志同级元素本身是否失效
4. 兄弟姐妹的后代失效集

我们通过一个例子来理解一下上面的定义
```
.q + .r { ... }
.q + .s .t { ... }
.q + * + .u > .v { ... }
```
在上面的css规则中，`.q`的兄弟元素失效集如下
1. 最大同级距离为2。这个我们可以从第三条规则中得到，即`.q`距离它影响到的最远的兄弟`.u`距离为2
2. 匹配的特征为`.r`、`.s`、`.u`。这里也很明显，第一条规则的兄弟特征为`.r`，第二条为`.s`，第三条为`.u`。
3. flag为true，由于第一条规则的影响，`.r`是失效的，`.r`为`.q`的同级。
4. 后代失效集为`{ .t, .v }`，基于第二和第三条规则得到的后代失效集。

基于这个失效集定义，它会使更多的元素样式失效，如官方文档中所说，这种方式很保守，但足够正确。那么我们通过一个demo来说明一下，哪些情况下，会导致元素失效的数量增多。

还是上面的css规则，假设我们的dom结构如下。
```

<div class="q">

</div>
<div class="r">
    <div class="t"></div>
    <div class="v"></div>
</div>
<div class="u">
    <div class="t"></div>
</div>
```
将第一个div的class移除，我们会发现有5个元素被重计算了。这五个元素即第一个div下面的五个div。为什么重计算了五个。
* 首先，同级元素失效集中的flag为true，因此导致含有特征的同级元素失效了，`.r`和`.u`均失效（**两个**），`.u`的失效并不在我们的预期内。
* 其次，后代失效集中含有`.t`和`.v`，因此，`.r`的后代`.t`和`.v`失效，又**两个**。但这两个的失效也不在我们的预期内，预期是`.r`失效就可以了。
* 最后，`.u`的后代`.t`失效，**一个**。这个依然不在我们的预期内，我们预期是`.u`的后代`.v`失效。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ec0fcf5a21e43d08e4eb0af145248b8~tplv-k3u1fbpfcp-watermark.image)

综上，重计算了5个元素，结果有4个是无意义的计算。当dom结构，css规则复杂的情况下，这个数字可能会被放大很多倍。但是，目前的web应用越来越复杂，要求写css的开发者完全理解这一套失效集的规则，并应用到日常开发中，心智负担非常大。

笔者个人的建议是**避免使用兄弟元素选择器**。
## Reference

* https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/core/css/style-invalidation.md
