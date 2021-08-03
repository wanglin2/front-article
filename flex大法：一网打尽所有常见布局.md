「本文已参与好文召集令活动，点击查看：[后端、大前端双赛道投稿，2万元奖池等你挑战！](https://juejin.cn/post/6978685539985653767)」


`flex`全称`Flexible Box`模型，顾名思义就是灵活的盒子，不过一般都叫弹性盒子，所有`PC`端及手机端现代浏览器都支持，所以不用担心它的兼容性，有了这玩意，妈妈再也不用担心我们的布局。

先简单介绍一下，要使用`flex`布局，需要先给一个容器元素设置`display:flex`让它变成`flex`容器，然后其所有的直接子元素就变成`flex`子元素了，在`flex`里存在两根轴，叫主轴和交叉轴，互相垂直，主轴默认水平，`flex`子元素默认会沿主轴排列，可以控制`flex`子元素在主轴上伸缩，主轴方向可以设置，相关的`css`属性分为两类，一类是给`flex`容器设置的，一类是给`flex`子元素设置的，本文在介绍一些典型场景实现的同时也会顺带讲解部分属性，当然更详细的内容可以阅读[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flexible_Box_Layout)上的教程。



# 单列布局

单列布局是最简单的布局了，从上到下排列，如图：

![image-20210702153217161.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd560cfc1d964d0fbd092e9772cb5726~tplv-k3u1fbpfcp-watermark.image)

可以使用三个`div`来表示头、内容和尾，然后把外层容器，即`body`设为`flex`容器，因为`flex`默认的主轴是水平的，我们需要把它设置为垂直的，然后再设置元素在交叉轴居中即可：

![image-20210705135947864.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb40f68baea44882a14d4e8557f56c30~tplv-k3u1fbpfcp-watermark.image)

当然更常见的情况是内容高度不确定，这样我们往往会希望在内容高度不满一屏时底部内容挨着底边，超过一屏时跟在最后，这首先需要容器元素有固定的高度，否则何来底边，我们可以把`html`和`body`的高度都设为`100%`，然后去掉给`content`元素设置的高度，并给它添加一个带高度的子元素：

![image-20210705140211900.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe68931b7ec5403ebc9710da889698fc~tplv-k3u1fbpfcp-watermark.image)

接下来需要使用到`flex-grow`属性，这个是`flex`子元素上的属性，用来控制容器还有空间剩余时，`flex`子元素怎么进行扩展，默认值是`0`，也就是不扩展，子元素会显示为它们默认的大小，这个所谓的默认大小分几种情况：

1.如果子元素的另一个属性`flex-basis`设置了不为`auto`的具体数值，那么无论元素有没有设置具体大小都显示为该属性定义的尺寸；

2.如果子元素的`flex-basis`的值为`auto`（默认值），那么如果元素设置了具体的大小那么显示为该设置的尺寸；

3.否则取决于元素内容的`max-content`大小；

当`flex-grow`设为一个正数时，那么各个子元素会按设置的份数来按比例分配剩余可用的空间，比如剩余空间为`90px`，三个子元素该属性值都设为`1`，那么每个元素将在原来大小的基础上加上`90/3=30px`。

根据上述原理，我们只需要给`content`元素的`flex-grow`属性设为`1`即可，其他都是`0`，所以剩余空间将全给`content`元素：

![image-20210705140847890.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0440ca859164e07bd21a45998eda23d~tplv-k3u1fbpfcp-watermark.image)

这样内容不足时底部就可以挨着底边了，但是当内容过多，超过一屏时：

![image-20210702163153012.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90fa4c85599d4e4da04d67c0b8bc691c~tplv-k3u1fbpfcp-watermark.image)

可以看到头和尾都没了，这是因为`flex-shrink`的原因，这个也是`flex`子元素上的属性，用来控制当子元素的尺寸之和已经超过容器了要怎么收缩元素，默认值为`1`，就是按比例减去要收缩的空间，理论上是这样，但实际上并没有这么简单，接下来简单测试一下：

容器元素`body`为`800px`高，上中下高度分别为`100+1000+100=1200px`，根据`1:1:1`的`flex-shrink`计算总权重为`1*100+1*1000+1*100=1200`，子元素总高度超过容器`400px`，这多出的要按的比例从各自高度中减去，具体为：
```js
(400*1*100)/1200=33.33px
(400*1*1000)/1200=333.33px
(400*1*100)/1200=33.33px
```
，也就是分别都减去上述值，减完后理论上各自的高度变成了`66.67px、666.67.67px、66.67px`，但是实际上：

![image-20210705141827907.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ccdfa4a16a24c83ab07f7e10d04bb2e~tplv-k3u1fbpfcp-watermark.image)

可以看到头和尾都变成了`0`，内容高度没有变，这是为啥呢？上面我们提到了`max-content`，同样，这里对应着`min-content`的概念，虽然正常来说应该变成我们计算出的尺寸才对，但是减少到元素内容的`min-content`后它就不会再变小了，`content`元素有个高度为`1000`的子元素，这个高度就是它的`min-content`，所以它不会缩小，它一个元素就比容器元素高了，再加上头和尾因为都没有内容，所以虽然理论上它们不是为`0`的，但是为了更好的显示效果，浏览器就直接把它们减少到`0`了，我们可以随便给头和尾加一点文字，文字的高度就会变成它们的`min-content`，它们的高度也就不会变成`0`：

![image-20210705144449255.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd593975811c4dc69db32922d5fe0903~tplv-k3u1fbpfcp-watermark.image)

所以这就意味着不要想着去精确计算，把它交给浏览器，浏览器会给你以最好的方式呈现。

那么解决头和尾不要消失的问题很简单，可以给它们也加个固定高度的子元素，但是最简单的方法是把它们的`flex-shrink`设为`0`，也就是不收缩：

![image-20210705144934806.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5769cb67c8df44a0b2b75985e64dad9e~tplv-k3u1fbpfcp-watermark.image)

这样就实现我们的需求了。



# 经典导航栏

![image-20210703141726544.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ff86ca6766e4c689128665eb964ebaf~tplv-k3u1fbpfcp-watermark.image)

如图所示是一个经典的网站导航栏的布局，`logo`和导航在左，用户信息在右，不用`flex`可能会使用浮动，上图使用浮动还好，但是如果右边是两个块，那么右边浮动的元素的显示顺序和书写顺序要不一致才行，或者再用一个元素包裹一下，使用`flex`则没有这种烦恼。

该场景可以使用一个容器来包裹左边的`logo`和导航，再设置`justify-content:space-between`来实现，但是有个小技巧可以不用这个包裹元素，就是利用`margin`的`auto`值，回忆一下我们以前水平居中都是怎么做的，是不是这样`margin:0 auto`，`margin-left`和`margin-right`的默认值是`0`，如果设置为`auto`，将会根据剩余可用空间来计算，这也是为什么能水平居中，因为左右都想尽量多，那么就只能平分了，对于本示例，我们只给用户名`flex`子元素设置`margin-left:auto`，那么剩余空间将全部给它，也就相当于把用户块挤到右边去了：

![image-20210703154629584.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8baa78480ada40f2ad8003e3e5f642b4~tplv-k3u1fbpfcp-watermark.image)


# 隔行交叉显示

![image-20210703155438667.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1723527719344a17bff5770bf4883d78~tplv-k3u1fbpfcp-watermark.image)

有时候为了不让布局太单调，即使一个列表是同类数据，展示上也会做成上述隔行交叉的样式，这个使用`flex`可以轻松的做到，给`2n`的行设置`flex-direction: row-reverse`即可让偶数行的主轴方向由默认的从左向右变成从右向左：

![image-20210705093825696.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f2631edbace48db8bcba9d9f9e606d4~tplv-k3u1fbpfcp-watermark.image)

此外也可以使用`order`属性，这个属性可以让`flex`子元素按`order`的数值大小来排序显示，我们可以默认左边的设为`2`，右边的设为`3`，然后在偶数行再给右边的设为`1`，自然就跑到前面去了：

![image-20210705094409057.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edb008e48f2a42638e96aa76e3354065~tplv-k3u1fbpfcp-watermark.image)


# 网格布局

此网格非`grid`布局，虽然网格列表用`grid`是最好的，但是本文的主角是`flex`，假设我们要实现下面这样一个列表：

![image-20210705191309843.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b04a802b2384a78a7f7fafc7bb4b8d0~tplv-k3u1fbpfcp-watermark.image)

上述列表对`flex`来说是不擅长的，因为要带间距，所以不能简单的把子元素宽度设为`25%`，否则再加上外边距，一行肯定显示不下四个，那你可能会想，那我宽度就少一点好了，比如设为`20%`，然后允许扩展，即`flex-grow:1`，那样不就可以把减去子元素宽度及外边距还剩下的空间再还给子元素了吗，试试看：

![image-20210705194536333.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2914d4c89f34aa7b2975b492f711e89~tplv-k3u1fbpfcp-watermark.image)

可以看到前面一切正常，但是最后一行因为只有一个元素，且设置了允许扩展，所以它被拉满整行了，这种效果显然不是我们要的。

其实我们可以使用内边距来做间距，设置一下子元素的`box-sizing:border-box`，让内边距包含在宽度内，这样就可以放心的把子元素的宽度设为`25%`了，当然这样的缺点是里面得再嵌套一个元素用来作为实际的内容容器。

![image-20210705193728280.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d16df8fa6a545ffa859a1a045ecfa76~tplv-k3u1fbpfcp-watermark.image)


# 圣杯布局

![image-20210705161203330.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9578d0f037b4b61820d8b5499ffd66d~tplv-k3u1fbpfcp-watermark.image)

所谓圣杯布局如上所示，头尾高度固定，宽度占满，中间的内容部分分为三列，两侧宽度固定，高度占满，中间的内容部分随着浏览器宽度变化，其实就是我们上面讲过的【单列布局】的中间部分变成三列而已，实现完全没有啥特别的，以【单列布局】为基础，给`content`添加三个子元素，两侧定宽，并且不允许收缩，中间允许扩展即可：

![image-20210705162141012.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e884c4be115d416eb9c01a110e350a17~tplv-k3u1fbpfcp-watermark.image)


# 垂直居中

不知道各位最开始用`flex`是为什么，反正笔者就是冲着垂直居中去的，用它实在是太简单了，之前还考虑是不是定高呀，用什么定位呀，用`flex`就是两步，一让父元素变成弹性盒子，二设置交叉轴的元素排布方式为居中就完事了：

![image-20210703135214049.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/559ce0edca70436b94d29c31058c7d0a~tplv-k3u1fbpfcp-watermark.image)

如果还需要水平居中的话就再给容器元素设置主轴的排列方式为`justify-content:center`，现在连让文字居中我都是用`flex`，无情的抛弃了`text-align`和`line-height`。



# 高度自动对齐

有些时候同一列的元素为了美观我们希望他们的高度是一样的，如果内容固定不变当然可以直接写死高度，但如果可变的话就不能写死了：

![image-20210703111013531.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd34ed5d75a14a518bd139a365aac6fd~tplv-k3u1fbpfcp-watermark.image)

这个场景使用`flex`完全不需要额外设置什么属性，只要给容器元素设置`display:flex`即可，因为`flex`容器有个属性`align-items`，用来设置`flex`子元素在交叉轴上如何对齐，默认值为`stretch`，即如果`flex`子元素未设置高度，那么将占满整个容器的高度，因为我们并没有给容器设置高度，所以容器的高度就是所有`flex`子元素里最大的高度。

![image-20210703135746891.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39e41727a6e84053a8c1321a4e5d9b10~tplv-k3u1fbpfcp-watermark.image)

# 小结

本文以标题党的名义总结了部分常见布局使用`flex`的实现，要灵活使用`flex`还是需要理解它的一些属性的意义，此外也需要知道`flex`的边界在哪，哪些是它不擅长的。

本文总结的难免会有不全，或者有更好的实现，欢迎讨论。
