## 前言
在前端开发的过程中，经常会看到浏览器在做一个`Recalculate Style`的任务。https://codepen.io/TGuoW/pen/GRNzeZv 在这个页面内点击按钮，录制profile，便会发现，仅仅给main元素增加了一个完全不影响样式的class，竟然会影响到1001个节点。而在我们的预期中，增加这个class，只会影响main元素自身，而不会影响到它的子节点。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/878d3755e3184c32bcc3e928cf8bd45d~tplv-k3u1fbpfcp-watermark.image)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab0238f629354ebe9c18c7169dcbd2a3~tplv-k3u1fbpfcp-watermark.image)

在实例代码html层级结构如此简单的场景下，增加一个无关class，耗时约2ms。在业务开发时，会遇到非常多的更加复杂的场景，耗时可能会突破毫秒级到达秒级。因此，了解chrome为什么会重计算这么多节点，有助于我们写出高性能的css。

**css优化方式网络上已经很多文章了，本文的重点是从原理上去解释为什么这些css写法会导致性能降低**

## 先说结论
1. 

## 样式重计算整体过程

* 开发者对dom实施操作，操作通过某些方式读取dom（如`getBoundingClientRect`）
* chrome标记样式可能失效的元素
* 对所有被标记的元素重计算

那么减少被标记的失效元素，就能减少`Recalculate Style`的耗时

## 样式失效标记
首先，当一个dom的style或者class发生改变，chrome 是无法**完全**预知它会导致哪些dom的样式需要重计算，那么如何去保证重计算后页面的样式符合预期？

假想一下，如果开发者操作一个dom之后，文档中所有的dom都进行重计算，那么就可以保证每个dom的样式信息都是正确的。

但明显不可能这么干。

### 树结构的更改导致样式失效

比如插入了一个节点，chrome的处理比较简单粗暴，会重新计算它的所有子节点的样式，并且由于兄弟选择器等的存在，可能会重新计算它的兄弟元素的样式。

举个例子，存在以下css
```
.a、.a * 、.a〜* 、. a〜* * {}
````

在元素上将class属性设置为“ a”时，第一个选择器将选择元素本身，第二个选择器将选择元素的所有后代，第三个选择器将其所有兄弟姐妹，第四个选择器将其所有兄弟姐妹的子孙。 因此，必须重新计算所有这些后代和兄弟姐妹的样式。 


### 后代失效集（Descendant Invalidation Sets）
以上是dom结构导致的style 重计算，不是本文的重点。重点在于研究dom 属性的改变导致的重计算，本节内容主要关于dom属性变化，导致后代重计算的原理。核心概念有两个： **特征** 和 **失效集**。
#### 特征（feature）
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
那么问题就来了，为什么特征是`#b`而不是`.b`呢？顾名思义，特征（feature）即是一个选择器中最明显的部分，在一个html内，id选择器比class选择器特征更明显。
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

上面两个demo验证了id选择器的优先级比class选择器更高（其它优先级可通过类似demo自行验证）。因此，我们可以得到一个优化的tip：**尽量避免一个复合选择器中的高优先级选择器匹配到页面中较多的元素**，比如这种选择器写法`#a > span{}`，会导致匹配到`#a`下面所有的`span`，而不是仅仅是儿子`span`。

#### 失效集

在chrome的实现中，会去遍历所有的css样式来计算它的后代失效集。

后代失效集的定义是

1. 给定元素E，并且在E上修改了属性P。当E的后代F中的一个属性是在P的后代失效集中，那么F需要重新计算。这里的P指元素id、元素class和其它attribute。
2. 当P的后代失效集为空，那么在元素E上修改属性P，只会重新计算元素E的样式。
3. 在后代失效集上存在一个flag，`wholeSubtreeInvalid`，顾名思义，这个flag为true时，E的所有后代都会重新计算样式。
4. 在后代失效集上存在一个flag，`treeBoundaryCrossing`，如果P的修改会导致shadow dom中的元素样式失效，这个flag设置为true。
5. 在后代失效集上存在一个flag，`insertPointCrossing`，如果P的简单选择器右边有:: content伪元素，或者P的简单选择器位于：host或：host-context伪类中，则在P的集合上设置insertPointCrossing标志。

后代失效集类似于一个`key-value`的结构，key值是一条css规则中的非最右选择器，value就是我们上面提到的特征（feature）的集合。

那么举一些例子，让我们容易去理解后代失效集的生成规则：

#### 例子一
这个很简单，选择器只选中了类名中含有a的元素，那么它的后代失效集为空。
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

因此，css优化手段之三，尽量避免`:not`选择器。

#### 例子五


所以css性能优化方式之二：少写css（这种优化方式可以忽略，因为它虽然有用，但效果微乎其微）


### 同级元素失效集
上面介绍的都属于亲子关系的元素失效集。那么当使用同级元素选择器时，比如`+`，`~`时会发生什么。当使用同级元素选择器时，同样不可能所有同级元素都重计算，因此需要一种规则去规定此时应该重计算哪些元素。

## Reference

* https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/core/css/style-invalidation.md
