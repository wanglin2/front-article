# 相识

`console`一定是各位前端er最熟悉的小伙伴了，无论是`console`控制台，还是`console`对象，做前端做久了，打开一个网页总是莫名自然的顺手打开控制台，有些调皮的网站还会故意在控制台输出一些有意思的东西，比如招聘信息，像百度的：

![image-20210602194947190.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/200659bac3d845dcaf5d3c060646b7e8~tplv-k3u1fbpfcp-watermark.image)

其他的不说，真的每年都更新，看着还挺让人热血沸腾。

另外输出一些花里胡哨的字符图形也是很常见的，比如天猫的：

![image-20210602195543909.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ca0beed5d0745f99f2d992e81175eab~tplv-k3u1fbpfcp-watermark.image)

也有一些网站可能不喜欢被人调试，只要打开控制台就自动进入调试模式，还是无限`debugger`的那种，最简单的实现方式如下：

```js
setInterval(() =>{
    debugger
}, 1000)
```

破解也不难，有兴趣的可以百度一下。

不知道是否有人像我一样，做前端很多年，就靠一个`console.log`走天下，仿佛`console`就一个`log`方法，如果是，那么本文就来一起进一步认识一下`console`对象吧。



# 相知

`console`对象是由宿主环境提供的，如浏览器和`nodejs`，作为全局对象的一个属性，不需要通过构造函数创建，直接使用即可，`console`对象的`__proto__`指向的是一个空对象，所以 `console`对象的方法都挂在对象自身，在 `chrome`控制台打印`console`可以看有如下方法或属性：

![image-20210603104148209.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/840f631cae944707a60e4221dbc7b516~tplv-k3u1fbpfcp-watermark.image)

`console`输出信息的方法都可以接收多个以逗号分隔的参数，打印的时候会在同一行进行显示，不会换行，想要换行的话请使用`console`方法打印多次。

另外在不同的浏览器上同一个方法可能会有差异，鉴于大家基本都是使用`chrome`，所以以下内容大部分都是在`chrome`下的效果。

直接罗列`api`说实话挺无聊的，所以我们按场景来看。



## 场景1：输出普通的调试信息，如数字、字符串、对象、数组、函数等

可以使用`console.log`或`console.info`，这两个方法基本是一样的：

![image-20210603140257900.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3a91022bc7a4049b58c3a70f79e3e2b~tplv-k3u1fbpfcp-watermark.image)


## 场景2：想输出不同等级的调试信息，如警告信息或报错信息

调试级别的信息可以使用`console.debug`方法，控制台默认是不显示的，想要看到的话需要勾上控制台对应的选项：

![image-20210603142111283.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25b71137f1604ab4a4a12bd182db1d29~tplv-k3u1fbpfcp-watermark.image)

警告信息可以使用`console.warn`方法，会将这行信息添加黄色的背景以及一个感叹号图标，同时会显示堆栈信息：

![image-20210603142358541.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d59766984a54c0984dc4a28d4f6e9d2~tplv-k3u1fbpfcp-watermark.image)

错误信息可以使用`console.error`方法，会将这行信息添加红色的背景以及一个叉号图标，同时会显示堆栈信息：

![image-20210603142503302.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53b7ac3ca10a4f0e9c90cd6a673e489c~tplv-k3u1fbpfcp-watermark.image)


## 场景3：想查看某个`DOM`元素的所有属性

比如说我想看`body`元素的所有属性要怎么看呢：

```js
console.log(document.body)
```

这样在控制台打印出的是`dom`结构，看不到具体是属性：

![image-20210603200507246.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6e08cfc91954ea8bf45cc8a516f263e~tplv-k3u1fbpfcp-watermark.image)

那怎么办呢，可以使用`for in`来遍历：

```js
for(let p in document.body) {
    console.log(p, document.body[p])
}
```

还有一个简单的方法是把它作为数组的一项或者是对象的一个属性值：

```js
console.log([document.body], {body:document.body})
```

![image-20210604101218408.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56c64bf4183c4b24965480de9e124433~tplv-k3u1fbpfcp-watermark.image)

当然，以上都不是最简单的，最简单的是直接使用`console.dir`方法：

![image-20210604101459074.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63ee643d4fbd4193a69fdf2400fb2add~tplv-k3u1fbpfcp-watermark.image)


## 场景4：想查看具体的调用位置、调用堆栈等信息

只需要找到调用位置的话，`log`、`info`、`error`等方法都可以，如果还想查看调用堆栈信息的话可以使用`console.assert`、`console.error`、`console.warn`以及专门的方法`console.trace`，`trace`方法可以不带参数：

![image-20210604110405501.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2aedd983bdf043b5b7db3c2b914998ed~tplv-k3u1fbpfcp-watermark.image)


## 场景5：有时候`console`写多了，打印出太多信息，无法一眼看出都是哪里的，也不容易分清楚哪些是相关联的

这个可以手动把其他的都给注释掉，只留你本次需要的（这要你说？），当然如果你愿意多敲几行代码的话，也可以使用`console.group`方法来进行分组显示，使用`console.groupEnd`方法结束分组，可以多级嵌套：

```js
console.group(xxx)
xxx
console.groupEnd()
```

![image-20210604135050309.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0806b9ff193c4e45bde83c92c19d1b4d~tplv-k3u1fbpfcp-watermark.image)


# 相恋

## 场景1：实现一下上述百度的效果

`console`要输出换行的字符需要使用`\n`：

```js
console.log('每一个星球都有一个驱动核心，\n每一种思想都有影响力的种子。\n感受世界的温度，\n年轻的你也能成为改变世界的动力，\n百度珍惜你所有的潜力。\n你的潜力，是改变世界的动力！')
```

输出红色的字可以使用占位符，占位符格式为：`console.log('%x其他字符', 'xxx', [xxx, xxx...])`

设置样式使用`%c`占位符，可以使用多个，为占位符后面的字符应用样式，替换完占位符还剩下的参数也会正常打印出来：

```js
console.log('%c百度2021校园招聘简历投递：', 'color:red', 'https://talent.baidu.com/external/baidu/campus.html')
```

支持常用的样式属性：

```js
console.log(
    '%c街%c角%c小%c林', 
    'font-size: 20px;margin-right: 5px', 
    'color: #58A7F2', 
    'font-size: 24px;background: #F4605F;color: #fff;padding: 5px', 
    'border: 1px solid #8F4CFF;padding: 10px;border-radius: 50%'
)
```

![image-20210604143402908.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5a07fdbd8f74a38893ababd9451ae46~tplv-k3u1fbpfcp-watermark.image)

除了`%c`，还有其他几个占位符：`%i`、`%f`、`%s`等，因为不太常用，所以就不具体介绍了。



## 场景2：在控制台画条龙吧

看来最近很流行画龙啊，行，满足你：

```js
console.log('%c', 'background-image: url(/龙.jpg); background-size: 100%; padding:267px 300px;')
```

![image-20210604153417001.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f450071d90684d37a8f6db7898c25833~tplv-k3u1fbpfcp-watermark.image)

ps.在`chrome`不知道为啥没有效果，以上是在`edge`浏览器上的效果。

（用图片就属于耍赖皮了啊，而且图片的支持性很差，估计很多浏览器都显示不出来，能不能换种方式？）

要求还挺多，不能用图片，那就和上述天猫的那只猫一样给你用字符画吧，不过这样我们需要先把图片转成字符，原理和大帅的那篇文章一样，只不过是把`div`换成字符。

使用`canvas`获取到图片的像素数据后，使用两层循环嵌套，外层遍历高，内层遍历宽，迭代高的时候添加一个换行符`\n`，迭代宽的时候，根据当前像素点的`r`、`g`、`b`信息判断是添加空字符还是非空字符，最后拼接完成的字符就是我们要打印的字符，不过需要注意的是因为我们是一个像素点对应一个字符，但是字符的实际大小肯定是比一个像素大的，比如一个`16px`的文字，那么最终我们得到的字符图形将是原图片的16倍，这显然太大了，控制台显示不下，所以需要缩小，怎么缩小呢，有两个方法，一个是缩小图片，图片小了，像素点自然就少了，二是减少取样点，比如每隔`10px`我们取一个点，这样的问题是最终图形可能会和原图片有点偏差。

```js
// 加载龙的图片
let img = new Image()
img.src = './龙.jpg'
img.onload = () => {
    draw()
}
// 把图片绘制到canvas里
const draw = () => {
    const canvas = document.getElementById('canvas')
    canvas.width = img.width
    canvas.height = img.height
    const ctx = canvas.getContext('2d')
    ctx.drawImage(img, 0, 0, img.width, img.height)
    // 获取像素数据
    const imgData = ctx.getImageData(0, 0, img.width, img.height).data
    // 拼接字符
    join(imgData)
}
// 把像素数据拼接成字符
const join = (data) => {
    let gap = 10
    let str = ''
    for (let h = 0; h < img.height; h += gap) {
        str += '\n'
        for (let w = 0; w < img.width; w += gap) {
            str += ' '// 因为字符的高度普遍都比其宽度大，所以额外添加一个空字符平衡一下，否则最终的图形会感觉被拉高了
            let pos = (h * img.width + w) * 4
            let r = data[pos]
            let g = data[pos + 1]
            let b = data[pos + 2]
            // rgb转换成yuv格式，根据y（亮度）来判断显示什么字符
            let y = r * 0.299 + g * 0.578 + b * 0.114
            if (y >= 190) {
                // 浅色
                str += ' '
            } else {
                // 深色
                str += '#'
            }
        }
    }
    console.log(str)
}
```

效果如下：

![image-20210605110505602.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/443b4bd6d1034988ba7e8a812eb5350e~tplv-k3u1fbpfcp-watermark.image)

可以看到虽然大致形状出来了，但是细节少了很多，另外一种缩小图片的方式有兴趣可以自行尝试，效果可能会比这种好一点。不过也不用这么麻烦，有很多网站就可以直接帮你转，比如：[https://www.degraeve.com/img2txt.php](https://www.degraeve.com/img2txt.php)。



# 相爱

## 场景1：怎么更方便的打印对象

对象，我们都知道它是引用类型，平时开发中，我们经常会打印某个对象或数组，如果没有修改它的话当然没有什么问题，但是如果中途对它有多次修改，又想看每次修改后的这一时刻的数据，很遗憾，直接使用`console.log`或`dir`之类的方法最终显示的都是该对象最后时刻的数据：

```js
let obj = {a: 1, b: [1, 2, 3]}
console.log(obj)
obj.a = 2
console.error(obj)
obj.b.push(4)
console.dir(obj)
```

![image-20210605112948478.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d39d2bd0859d46e9bcdf75ae577f4c31~tplv-k3u1fbpfcp-watermark.image)

可以看到旁边都有个叹号，移上去会显示一行提示：`This value was evaluated upon first expanding,It may have changed since then.`，意思就是这个值计算了一次，但是后面可能是会变化的，所以我们往往会使用：`console.log(JSON.stringify(obj))`或者深拷贝一下再打印，有没有更简单的方法呢？我们可以给`console`加两个方法，一个叫`console.obj`，先深拷贝一下再打印，另一个叫`console.str`，把对象序列化后再打印：

```js
console.obj = function (...args) {
    let newArgs = args.map((item) => {
        if (Object.prototype.toString.call(item) === '[object Object]' || Array.isArray(item)) {
            return deepClone(item)
        } else {
            return item
        }
    })
    console.log(...newArgs)
}

console.str = function (...args) {
    let newArgs = args.map((item) => {
        try {
            let obj = JSON.stringify(item)
            return obj
        } catch(e) {
            return item
        }
    })
    console.log(...newArgs)
}
```



## 场景2：怎么在生产环境去掉console

想去掉生产环境的`console`可以通过`webpack`的插件来做，也可以拦截一下`console`对象的方法，判断是否是生产环境，是的话就不打印日志了，让我们来重写一下`console`对象：

```js
let oldConsole = window.console
let newConsole = Object.create(null)
// 其他方法这里暂时省略了
;['log'].forEach((method) => {
    newConsole[method] = function (...args) {
        // 非开发环境直接返回
        if (process.env.NODE_ENV !== 'development') {
            return
        }
        oldConsole[method](...args)
    }
})
window.console = newConsole
```

重写`console`可以用在任何需要知道`console`调用的场景下，比如前端监控日志上报。



# 相守

`nodejs`中的`console`和浏览器的是有点差异的，这个显而易见，毕竟命令行肯定没有浏览器这么强大：

![image-20210607185347466.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e17f9a2b986e4c059928c536e0404c51~tplv-k3u1fbpfcp-watermark.image)

如图所示，`log`、`info`、`error`、`warn`、`debug`这几个方法表面上看起来没有什么区别，`error`和`warn`不像在浏览器上一样有堆栈信息，`trace`还是保持着一致，对于对象的打印也是直接展开的，所以想要格式化的显示需要自行对要打印的对象进行处理，比如对于纯对象：

```js
console.log(JSON.stringify({a: 1, b: [1, 2, 3]}, null, 4))
```

![image-20210607190436253.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f35625dbc1c84a3ea751bdd9f4e345bd~tplv-k3u1fbpfcp-watermark.image)

另外`%c`的占位符显然也是没有效果的，如果想要打印出带颜色的可以使用[chalk](https://github.com/chalk/chalk)之类的工具库，其他一些方法的输出效果如果命令行不支持的话最终都会直接调用`console.log`来处理。

浏览器环境里没有`Console`类，但是`nodejs`里是存在的，有两种方式获取到：

```js
const { Console } = require('console')
const { Console } = console
```

通过`Console`类可以根据你的需求传入参数来实例化一个新的`console`实例：

```js
/*
stdout：可写流，用来输出信息
stderr：可选的可写流，用来输出错误信息，不传则使用stdout
ignoreErrors：在写入底层流时忽略错误
*/
new Console(stdout[, stderr][, ignoreErrors])
```

默认的全局`console`是输出到标准输出流和标准错误流，相当于：

```js
new Console(process.stdout, process.stderr)
```

那么你完全可以选择把日志输出到指定的文件里：

```js
const output = fs.createWriteStream('./stdout.log')
const errorOutput = fs.createWriteStream('./stderr.log')
new Console(output, errorOutput)
```



# 再见

纵是不舍，终有离别，各位看到这里的有缘人我们下次再见吧~
