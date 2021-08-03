---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight: juejin
---
## 开头

饼图，很常见的一种图表，使用任何一个图表库都能轻松的渲染出来，但是，我司的交互想法千奇百怪，布局捉摸不透，本身饼图是没啥可变的，但是配套的图例千变万化，翻遍`ECharts`配置文档都还原不出来，那么有两条路可以选，一是跟交互说实现不了，说服交互按图表库的布局来，但是一般交互可能会对你灵魂拷问，为什么别人都能做出来，你做不出来？所以我选第二种，自己做一个得了。



用`canvas`实现一个饼图很简单，所以本文在介绍使用`vue`高仿一个`ECharts`饼图的实现过程中会顺便回顾一下`canvas`的一些知识点，先来看一下本次的成果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0b29247123b44df8ca46698ee1aa1ba~tplv-k3u1fbpfcp-watermark.image)



## 布局及初始化工作

布局很简单，一个`div`容器，一个`canvas`元素即可。

```vue
<template>
  <div class="chartContainer" ref="container">
    <canvas ref="canvas"></canvas>
  </div>
</template>
```

容器的宽高写死，`canvas`的宽高需要通过本身的属性`width`和`height`来设置，最好不要使用`css`来设置，因为`canvas`画布默认的宽高是300*150，使用`css`不会改变画布原始的宽高，而是会将其拉伸到你设置的`css`宽高，所以会出现变形的问题。

```js
// 设置为容器宽高
let { width, height } = this.$refs.container.getBoundingClientRect()
let canvas = this.$refs.canvas
canvas.width = width
canvas.height = height
```

绘图的api都是挂在`canvas`的绘图上下文中，所以先获取一下：

```js
this.ctx = canvas.getContext("2d")
```

`canvas`坐标系默认的原点在左上角，饼图的绘制一般都是在画布中间，所以每次绘制圆弧的时候圆心都要换算一下设置到画布的中心点，这个示例中只要换算一个中心点并不麻烦，但是如果在更复杂的场景，所有都要换算是很麻烦的，所以为了避免，可以使用`translate`方法将画布的坐标系原点设置到画布中心点：

```js
this.centerX = width / 2
this.centerY = height / 2
this.ctx.translate(this.centerX, this.centerY)
```

接下来需要计算一下饼图的半径，画的太满不太好看，所以暂定为画布区域短边一半的90%：

```js
this.radius = Math.min(width, height) / 2 * 0.9
```

最后看一下要渲染的数据的结构：

```js
this.data = [
    {
        name: '名称',
        num: 10,
        color: ''// 颜色
    },
    // ...
]
```



## 饼图

饼图其实就是一堆面积不一的扇形组成的一个圆，画圆和扇形都是使用`arc`方法，它有6个参数，分别是圆心x、圆心y、半径r、圆弧起点弧度、圆弧终点弧度、逆时针还是顺时针绘制。

扇形的面积代表数据的占比，可以用角度的占比来表示，那就需要转成弧度，角度转弧度公式为：`弧度=角度*(Math.PI/180)`。

```js
// 遍历数据进行转换，total是所有数据的数量总和
let curTotalAngle = 0
let r = Math.PI / 180
this.data.forEach((item, index) => {
    let curAngle = (item.num / total) * 360
    let cruEndAngle = curTotalAngle + curAngle
    this.$set(this.data[index], 'angle', [curTotalAngle, cruEndAngle])// 角度
    this.$set(this.data[index], 'radian', [curTotalAngle * r, cruEndAngle * r])// 弧度
    curTotalAngle += curAngle
});
```

转换为弧度之后再遍历`angleData`来进行扇形绘制：

```js
// 函数renderPie
this.data.forEach((item, index) => {
    this.ctx.beginPath()
    this.ctx.moveTo(0, 0)
    this.ctx.fillStyle = item.color
    let startRadian = item.radian[0] - Math.PI/2
    let endRadian = item.radian[1] - Math.PI/2
    this.ctx.arc(0, 0, this.radius, startRadian, endRadian)
    this.ctx.fill()
});
```

效果如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b31acb5067d4564837069e70bd35567~tplv-k3u1fbpfcp-watermark.image)

`beginPath`方法用来开始一段新的路径，它会把当前路径的所有子路径都给清除掉，否则调用`fill`方法闭合路径时会把所有的子路径都首尾连接起来，那不是我们要的。

另外这里使用`moveTo`方法将这个新路径的起点移到了坐标原点，为什么要这样可以先看不这样的效果：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ac0204ce701438c89dd22fdc24c9275~tplv-k3u1fbpfcp-watermark.image)

原因是因为`arc`方法只是绘制一段圆弧，所以把它的首尾相连就是上述效果，但是扇形是需要这段圆弧和圆心一起闭合，`arc`方法调用时如果当前路径上已经存在子路径会用一段线段把当前子路径的终点和这段圆弧的起点连接起来，所以我们先把路径的起点移到圆心，这样最后闭合现成的就是一个扇形。

至于为什么起始弧度和结束弧度都减了`Math.PI/2`，是因为0弧度是在x轴的正方向，也就是右边，但是一般我们认为的起点在顶部，所以减掉1/4圆让它的起点移到顶部。



## 动画

我们在使用`ECharts`饼图的时候会发现它渲染的时候是会有一小段动画的：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/691501bcb7c7420ea7471eb78a38ab99~tplv-k3u1fbpfcp-watermark.image)

用`canvas`实现动画的基本原理就是不断改变绘图数据，然后不断刷新画布，听起来像是废话，所以一种实现方式是动态修改当前绘制结束的圆弧的弧度，从0一直变化到`2*Math.PI`，这样就可以实现这个慢慢变多的效果，但是这里我们使用另外一种，用`clip`方法。

`clip`用来在当前路径中创建一个剪裁路径，剪裁之后，后续绘制的信息只会出现在该剪裁路径内。基于此，我们可以创建一个从0弧度变化到`2*Math.PI`弧度的扇形剪裁区域，即可实现这个动画效果。

先看一下清除画布的方法：

```js
this.ctx.clearRect(-this.centerX, -this.centerY, this.width, this.height)
```

`clearRect`方法用来清除以`(x,y)`为起点，宽`width`高`height`范围内的所有已经绘制的内容。清除原理就是将这个范围内的像素都设置成透明，因为原点被我们移到了画布中心，所以画布左上角是（-this.centerX, -this.centerY）。

开源社区有很多动画库可以选择，但是因为我们只需要一个简单的动画函数，引入一个库没必要，所以自己简单写一个就好了。

```js
// 动画曲线函数，更多函数可参考：http://robertpenner.com/easing/
// t: current time, b: begInnIng value, c: change In value, d: duration
const ease = {
    // 弹跳
    easeOutBounce(t, b, c, d) {
        if ((t /= d) < (1 / 2.75)) {
            return c * (7.5625 * t * t) + b;
        } else if (t < (2 / 2.75)) {
            return c * (7.5625 * (t -= (1.5 / 2.75)) * t + .75) + b;
        } else if (t < (2.5 / 2.75)) {
            return c * (7.5625 * (t -= (2.25 / 2.75)) * t + .9375) + b;
        } else {
            return c * (7.5625 * (t -= (2.625 / 2.75)) * t + .984375) + b;
        }
    },
    // 慢进慢出
    easeInOut(t, b, c, d) {
        if ((t /= d / 2) < 1) return c / 2 * t * t * t + b
        return c / 2 * ((t -= 2) * t * t + 2) + b
    }
}
/*
	动画函数
	from：起始值
	to：目标值
	dur：过渡时间，ms
	callback：实时回调函数
	done：动画结束的回调函数
	easing：动画曲线函数
*/
function move(from, to, dur = 500, callback = () => {}, done = () => {}, easing = 'easeInOut') {
    let difference = to - from
    let startTime = Date.now()
    let isStop = false
    let timer = null
    let run = () => {
        if (isStop) {
            return false
        }
        let curTime = Date.now()
        let durationTime = curTime - startTime
        // 调用缓动函数来计算当前的比例
        let ratio = ease[easing](durationTime, 0, 1, dur)
        ratio = ratio > 1 ? 1 : ratio
        let step = difference * ratio + from
        callback && callback(step)
        if (ratio < 1) {
            timer = window.requestAnimationFrame(run)
        } else {
            done && done()
        }
    }
    run()
    return () => {
        isStop = true
        cancelAnimationFrame(timer)
    }
}
```

有了动画函数就可以很方便实现扇形的变化：

```js
// 从-0.5到1.5的原因和上面绘制扇形时减去Math.PI/2一样
move(-0.5, 1.5, 1000, (cur) => {
    this.ctx.save()
    // 绘制扇形剪切路径
    this.ctx.beginPath()
    this.ctx.moveTo(0, 0)
    this.ctx.arc(
        0,
        0,
        this.radius,
        -0.5 * Math.PI,
        cur * Math.PI// 结束圆弧不断变大
    )
    this.ctx.closePath()
    // 剪切完后进行绘制
    this.ctx.clip()
    this.renderPie()
    this.ctx.restore()
});
```

效果如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae0181c59c9e41ebae8c491dccd342f7~tplv-k3u1fbpfcp-watermark.image)

这里使用了`save`和`restore`方法，`save`方法用来将当前的绘图状态保存起来，你在之后如果修改了状态再调用`restore`方法可以又恢复到之前保存的状态，这两个方法是通过栈来进行保存，所以可以保存多个，只要`restore`方法正确对应上，在`canvas`中，绘图状态包括：当前的变换矩阵、当前的剪切区域、当前的虚线列表，绘图样式属性。

这里要使用这两个方法是因为如果当前已经存在裁剪区域，再调用`clip`方法时会将剪切区域设置为当前裁剪区域和当前路径的交集，所以剪切区域可能会越来越小，保险起见，在使用`clip`方法时都将它放在`save`和`restore`方法之间。



## 鼠标移上的突出显示

`ECharts`的饼图还有一个效果就是鼠标移上去所在的扇形会突出显示，其实也是一个小动画，突出的原理实际上就是这个扇形的半径变大了，按之前的套路，只要把半径的变化值交给动画函数跑一下就可以了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/542908f8e7be4a75b3652a92145ce208~tplv-k3u1fbpfcp-watermark.image)

不过这之前需要先要知道鼠标移到了哪个扇形上，先给元素绑定一下鼠标移动事件：

```vue
<template>
  <div class="chartContainer" ref="container">
    <canvas ref="chart" @mousemove="onCanvasMousemove"></canvas>
  </div>
</template>
```

获取一个坐标点是否在某个路径内可以使用`isPointInPath`，该方法可以检测某个点是否在当前的路径内，注意，是当前路径。所以我们可以在之前的遍历绘制扇形的循环方法里加上这个检测：

```js
renderPie (checkHover, x, y) {
    let hoverIndex = null// ++
    this.data.forEach((item, index) => {
        this.ctx.beginPath()
        this.ctx.moveTo(0, 0)
        this.ctx.fillStyle = item.color
        let startRadian = item.radian[0] - Math.PI/2
        let endRadian = item.radian[1] - Math.PI/2
        this.ctx.arc(0, 0, this.radius, startRadian, endRadian)
        // this.ctx.fill();--
        // ++
        if (checkHover) {
            if (hoverIndex === null && this.ctx.isPointInPath(x, y)) {
                hoverIndex = index
            }
        } else {
            this.ctx.fill()
        }
    })
    // ++
    if (checkHover) {
        return hoverIndex
    }
}
```

那么在`onCanvasMousemove`方法里要做的就是计算一下上面的`(x,y)`，然后调用一下这个方法：

```js
onCanvasMousemove(e) {
    let rect = this.$refs.canvas.getBoundingClientRect()
    let x = e.clientX - rect.left
    let y = e.clientY - rect.top
    // 检测当前所在扇形
    this.curHoverIndex = this.getHoverAngleIndex(x, y)
}
```

获取到所在的扇形索引后就可以让该扇形的半径动起来，半径变大可以乘一个倍数，比如变大0.1倍，那我们就可以通过动画函数让这个倍数从0过渡到0.1，再修改上面的遍历绘制扇形方法里的半径值，不断刷新重绘即可。

不过在此之前，要先去上面定义的数据结构里加一个字段：

```js
this.data = [
    {
        name: '名称',
        num: 10,
        color: '',
        hoverDrawRatio: 0// 这个字段表示当前扇形绘制时的倍数
    },
    // ...
]
```

要给每个扇形都单独加一个倍数字段的原因是同一时刻不一定只有一个扇形的倍数在变化，比如我从一个扇形快速移到另一个扇形，这个扇形的半径在变大的同时前一个扇形的半径还在恢复，所以是会同时变化的。

```js
onCanvasMousemove(e) {
   	// ...
    // 检测当前所在扇形
    this.curHoverIndex = this.getHoverAngleIndex(x, y)
    // 让倍数动起来
    if (this.curHoverIndex !== null) {
        move(
            this.data[hoverIndex].hoverDrawRatio,// 默认是0
            0.1,
            300,
            (cur) => {
                // 实时修改该扇形的倍数
                this.data[hoverIndex].hoverDrawRatio = cur
                // 重新绘制
                this.renderPie()
            },
            null,
            "easeOutBounce"// 参考ECharts，这里选择弹跳动画
        )
    }
}
// 获取鼠标移到的扇形索引
getHoverAngleIndex(x, y) {
    this.ctx.save()
    let index = this.renderPie(true, x, y)
    this.ctx.restore()
    return index
}
```

接下来改造绘制函数：

```js
renderPie (checkHover, x, y) {
    let hoverIndex = null
    this.data.forEach((item, index) => {
        this.ctx.beginPath()
        this.ctx.moveTo(0, 0)
        this.ctx.fillStyle = item.color
        let startRadian = item.radian[0] - Math.PI/2
        let endRadian = item.radian[1] - Math.PI/2
        // this.ctx.arc(0, 0, this.radius, startRadian, endRadian)--
        // 半径从写死的修改成加上当前扇形的放大值
        let _radius = this.radius + this.radius * item.hoverDrawRatio
    	this.ctx.arc(0, 0, _radius, startRadian, endRadian)
        if (checkHover) {
            if (hoverIndex === null && this.ctx.isPointInPath(x, y)) {
                hoverIndex = index
            }
        } else {
            this.ctx.fill()
        }
    });
    if (checkHover) {
        return hoverIndex
    }
}
```

然而上面的代码并不会实现预期的效果，有个问题需要解决。在同一个扇形里面移动`onCanvasMousemove`会持续触发并检测到当前所在索引调用`move`方法，可能是一个动画还没结束，而且在同一个扇形里移动只要动画一次就够了，所以需要做个判断：

```js
onCanvasMousemove(e) {
   	// ...
    this.curHoverIndex = this.getHoverAngleIndex(x, y)
    if (this.curHoverIndex !== null) {
        // 增加一个字段来记录上一次所在的扇形索引
        if (this.lastHoverIndex !== this.curHoverIndex) {// ++
            this.lastHoverIndex = this.curHoverIndex// ++
            move(
                this.data[hoverIndex].hoverDrawRatio,
                0.1,
                300,
                (cur) => {
                    this.data[hoverIndex].hoverDrawRatio = cur
                    this.renderPie()
                },
                null,
                "easeOutBounce"
            )
        }
    } else {// ++
        this.lastHoverIndex = null
    }
}
```

最后加一下由大变回去的动画方法，遍历数据，判断哪个扇形当前的放大倍数不为0，就给它加个动画，这个方法的调用位置是在`onCanvasMousemove`函数里，因为当你从一个扇形移到另一个扇形，或从圆内部移到外部都需要判断是否要恢复：

```js
resume() {
    this.data.forEach((item, index) => {
        if (
            index !== this.curHoverIndex &&// 当前鼠标所在的扇形不需要恢复
            item.hoverDrawRatio !== 0 &&// 当前扇形放大倍数不为0代表需要恢复
            this.data[index].stop === null// 因为这个方法会在鼠标移动过程中不断调用，所以要判断一下当前扇形是否已经在动画中了，在的话就不需要重复进行了，stop字段同样需要在上述的数据结构里先添加一下
        ) {
            this.data[index].stop = move(
                item.hoverDrawRatio,
                0,
                300,
                (cur) => {
                    this.data[index].hoverDrawRatio = cur;
                    this.renderPie();
                },
                () => {
                    this.data[index].hoverDrawRatio = 0;
                    this.data[index].stop = null;
                },
                "easeOutBounce"
            );
        }
    });
},
```

效果如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af17c94c3bfc4a4ebada60239b0675c6~tplv-k3u1fbpfcp-watermark.image)



## 环图

环图其实就是饼图中间挖了个洞，同样可以使用`clip`方法来实现，具体就是创建一个圆环路径：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3621d37979a42298e21645f656a0b06~tplv-k3u1fbpfcp-watermark.image)

所谓圆环也就是一大一小两个圆，但是这样会存在两个区域，一个是小圆内部区域，一个是小圆和大圆之间的区域，那么`clip`方法怎么知道剪切哪个区域呢，`clip`方法其实是有参数的，`clip(fillRule)`，这个`fillRule`表示判断一个点是在路径内还是路径外的算法类型，默认是使用非零环绕原则，还有一个是奇偶环绕原则，非零环绕原则很简单，就是在某个区域向外画一条线段，这条线段与路径会有交叉点，和顺时针的线段交叉时加1，和逆时针线段交叉了减1， 最后看计数器是否是0，是0就不填充，非0就填充。

如果我们使用两个`arc`方法画两个圆形路径，这里我们需要填充的是这个圆环部分，所以从圆环里向外画一条线只有一个交叉点，那么肯定会被填充，但是从小圆内部画出的线段最终的计数器是1+1=2，不为0也会被填充，这样就不是圆环而是一个大圆了，所以需要通过`arc`方法最后一个参数来设置其中一个圆形路径为逆时针方向：

```js
clipPath() {
    this.ctx.beginPath()
    this.ctx.arc(0, 0, this.radiusInner, 0, Math.PI * 2)// 内圆顺时针
    this.ctx.arc(0, 0, this.radius, 0, Math.PI * 2, true)// 外圆逆时针
    this.ctx.closePath()
    this.ctx.clip()
}
```

这个方法在调用遍历绘制扇形的方法`renderPie`之前调用：

```js
// 包装成新函数，之前所有调用renderPie进行绘制的地方都替换成drawPie
drawPie() {
    this.clear()
    this.ctx.save()
    // 裁剪圆环区域
    this.clipPath()
    // 绘制圆环
    this.renderPie()
    this.ctx.restore()
}
```

这样会有个问题，就是这个剪切圆环的外圆半径是`radius`，而如果某个扇形放大了那么就显示不了了，所以需要实时遍历扇形数据来获取到当前最大的半径，可以使用计算属性来做这件事：

```js
{
    computed: {
        hoverRadius() {
            let max = null
            this.data.forEach((item) => {
                if (max === null) {
                    max = item.hoverDrawRatio
                } else {
                    if (item.hoverDrawRatio > max) {
                        max = item.hoverDrawRatio
                    }
                }
            })
            return this.radius + this.radius * max
        }
    }
}
```

效果如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f2b145a6795455e95363319ba491b88~tplv-k3u1fbpfcp-watermark.image)

可以看到上图有个bug，就是鼠标移到内圆里还是会触发凸出的动画效果，解决方法很简单，在之前的`getHoverAngleIndex`方法里我们先检查一下鼠标是否移到了内圆，是的话就不就行后续扇形检测了：

```js
getHoverAngleIndex(x, y) {
    this.ctx.save();
    // 移到内圆环不触发，创建一个内圆大小的路径，调用isPointInPath方法进行检测
    if (this.checkHoverInInnerCircle(x, y)) 
        return null;
    }
    let index = this.renderPie(true, x, y);
    this.ctx.restore();
    return index;
}
```



## 南丁格尔玫瑰图

最后再来实现一下南丁格尔玫瑰图，由一个叫南丁格尔的人分明的，是一种圆形的直方图，相当于把一个柱形图拉成一个圆形，用扇形的半径来表示数据的大小，实现上其实就是把环图里的扇形半径也通过占比来区分开。

要改造的是`renderPie`方法，绘制的半径由统一的半径乘上一个各自的占比即可：

```js
renderPie (checkHover, x, y) {
    let hoverIndex = null
    this.data.forEach((item, index) => {
        // ...
        // let _radius = this.radius + this.radius * item.hoverDrawRatio --
        // this.ctx.arc(0, 0, _radius, startRadian, endRadian)
        // ++
        // 该扇形和最大的扇形的大小比例换算成占圆环的比例
        let nightingaleRadius =
            (1 - item.num / this.max) * // 圆环减去该占后比剩下的部分
            (this.radius - this.radiusInner)// 圆环的大小
        let _radius = this.radius - nightingaleRadius// 外圆半径减去多出的部分
        let _radius = _radius + _radius * item.hoverDrawRatio
        this.ctx.arc(0, 0, _radius, startRadian, endRadian)
        // ...
    });
    // ...
}
```

效果如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b9df4243c984c4689ea1f177cc494d8~tplv-k3u1fbpfcp-watermark.image)



## 总结

本文通过一个简单的饼图来回顾了一下`canvas`的一些基础知识，`canvas`还有很多有用和高级的特性，比如`isPointInStroke`可以用来检测一个点是否在一条路径上，矩阵变换同样支持旋转和缩放，也可以用来处理图像等等，有兴趣的可以自行了解。

代码已上传到github：[https://github.com/wanglin2/pieChart](https://github.com/wanglin2/pieChart)

