笔者开源了一个Web思维导图[mind-map](https://github.com/wanglin2/mind-map)，最近在优化背景图片效果的时候遇到了一个问题，页面上展示时背景图片是通过`css`使用`background-image`渲染的，而导出的时候实际上是绘制到`canvas`上导出的，那么就会有个问题，`css`的背景图片支持比较丰富的效果，比如通过`background-size`设置大小，通过`background-position`设置位置，通过`background-repeat`设置重复，但是`canvas`笔者只找到一个`createPattern()`方法，且只支持设置重复效果，那么如何在`canvas`里模拟一定的`css`背景效果呢，不要走开，接下来一起来试试。

首先要说明的是不会去完美完整`100%`模拟`css`的所有效果，因为`css`太强大了，属性值组合很灵活，且种类非常多，其中单位就很多种，所有只会模拟一些常见的情况，单位也只考虑`px`和`%`。

读完本文，你还可以顺便复习一下`canvas`的`drawImage`方法，以及`css`背景设置的几个属性的用法。

# canvas的drawImage()方法

总的来说，我们会使用`canvas`的`drawImage()`方法来绘制背景图片，先来大致看一下这个方法，这个方法接收的参数比较多：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e4ce1ed7b4e427e9542b895c422bd11~tplv-k3u1fbpfcp-zoom-1.image)


只有三个参数是必填的。

# 基本框架和工具方法

核心逻辑就是加载图片，然后使用`drawImage`方法绘制图片，无非是根据各种`css`的属性和值来计算`drawImage`的参数，所以可以写出下面的函数基本框架：

```js
const drawBackgroundImageToCanvas = (
  ctx,// canvas绘图上下文
  width,// canvas宽度
  height,// canvas高度
  img,// 图片url
  { backgroundSize, backgroundPosition, backgroundRepeat }// css样式，只模拟这三种
) => {
  // canvas的宽高比
  let canvasRatio = width / height
  // 加载图片
  let image = new Image()
  image.src = img
  image.onload = () => {
    // 图片的宽高及宽高比
    let imgWidth = image.width
    let imgHeight = image.height
    let imageRatio = imgWidth / imgHeight
    // 绘制图片
    // drawImage方法的参数值
    let drawOpt = {
        sx: 0,
        sy: 0,
        swidth: imgWidth,// 默认绘制完整图片
        sheight: imgHeight,
        x: 0,
        y: 0,
        width: imgWidth,// 默认不缩放图片
        height: imgHeight
    }
    // 根据css属性和值计算...
    // 绘制图片
    ctx.drawImage(image, drawOpt.sx, drawOpt.sy, drawOpt.swidth, drawOpt.sheight, drawOpt.x, drawOpt.y, drawOpt.width, drawOpt.height)
  }
}
```

接下来看几个工具函数。

```js
// 将以空格分隔的字符串值转换成成数字/单位/值数组
const getNumberValueFromStr = value => {
  let arr = String(value).split(/\s+/)
  return arr.map(item => {
    if (/^[\d.]+/.test(item)) {
        // 数字+单位
        let res = /^([\d.]+)(.*)$/.exec(item)
        return [Number(res[1]), res[2]]
    } else {
        // 单个值
        return item
    }
  })
}
```

`css`的属性值为字符串或数字类型，比如`100px 100% auto`，不方便直接使用，所以转换成`[[100, 'px'], [100, '%'], 'auto']`形式。

```js
// 缩放宽度
const zoomWidth = (ratio, height) => {
    // w / height = ratio
    return ratio * height
}

// 缩放高度
const zoomHeight = (ratio, width) => {
  // width / h = ratio
  return width / ratio
}
```

根据原比例和新的宽度或高度，计算缩放后的宽度或高度。

# 模拟background-size属性

默认`background-repeat`的值为`repeat`，我们先不考虑重复的情况，所以先把它设置成`no-repeat`。

`background-size` 属性用于设置背景图片的大小，可以接受四种类型的值，依次来模拟一下。

## length类型

设置背景图片的高度和宽度。第一个值设置宽度，第二个值设置高度。如果只给出一个值，第二个默认为 **auto**(自动)。

`css`样式如下：

```css
.cssBox {
    background-image: url('/1.jpg');
    background-repeat: no-repeat;
    background-size: 300px;
}
```

只设置一个值，那么代表背景图片显示的实际宽度，高度没有设置，那么会根据图片的长宽比自动缩放，效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cfb27fd952344d11b8fe3da4c3fa0d3c~tplv-k3u1fbpfcp-zoom-1.image)


在`canvas`中模拟很简单，需要传给`drawImage`方法四个参数：`img、x、y、width、height`，`img`代表图片，`x、y`代表在画布上放置图片的位置，没有特殊设置，显然就是`0、0`，`width、height`代表将图片缩放到指定大小，如果`background-size`只传了一个值，那么`width`直接设置成这个值，而`height`则根据图片的长宽比进行计算，如果传了两个值，那么分别把两个值传给`width、height`即可，另外需要对值为`auto`的进行一下处理，实现如下：

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundSize: '300px'
})

const drawBackgroundImageToCanvas = () =>{
    // ...
    image.onload = () => {
        // ...
        // 模拟background-size
        handleBackgroundSize({
            backgroundSize, 
            drawOpt, 
            imageRatio
        })
        // ...
    }
}

// 模拟background-size
const handleBackgroundSize = ({ backgroundSize, drawOpt, imageRatio }) => {
    if (backgroundSize) {
      // 将值转换成数组
      let backgroundSizeValueArr = getNumberValueFromStr(backgroundSize)
      // 两个值都为auto，那就相当于不设置
      if (backgroundSizeValueArr[0] === 'auto' && backgroundSizeValueArr[1] === 'auto') {
        return
      }
      // 图片宽度
      let newNumberWidth = -1
      if (backgroundSizeValueArr[0]) {
        if (Array.isArray(backgroundSizeValueArr[0])) {
            // 数字+单位类型
            drawOpt.width = backgroundSizeValueArr[0][0]
            newNumberWidth = backgroundSizeValueArr[0][0]
        } else if (backgroundSizeValueArr[0] === 'auto') {
            // auto类型，那么根据设置的新高度以图片原宽高比进行自适应
            if (backgroundSizeValueArr[1]) {
                drawOpt.width = zoomWidth(imageRatio, backgroundSizeValueArr[1][0])
            }
        }
      }
      // 设置了图片高度
      if (backgroundSizeValueArr[1] && Array.isArray(backgroundSizeValueArr[1])) {
        // 数字+单位类型
        drawOpt.height = backgroundSizeValueArr[1][0]
      } else if (newNumberWidth !== -1) {
        // 没有设置图片高度或者设置为auto，那么根据设置的新宽度以图片原宽高比进行自适应
        drawOpt.height = zoomHeight(imageRatio, newNumberWidth)
      }
    }
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95c1bf5f08f2444caf0098b3a0de9f3b~tplv-k3u1fbpfcp-zoom-1.image)


设置两个值的效果：

```css
background-size: 300px 400px;
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc5094dae5044cbeb7f6f17d51d63319~tplv-k3u1fbpfcp-zoom-1.image)


## percentage类型

将计算相对于背景定位区域的百分比。第一个值设置宽度百分比，第二个值设置的高度百分比。如果只给出一个值，第二个默认为**auto**(自动)。比如设置了`50% 80%`，意思是将图片缩放到背景区域的`50%`宽度和`80%`高度。

`css`样式如下：

```css
.cssBox {
    background-image: url('/1.jpg');
    background-repeat: no-repeat;
    background-size: 50% 80%;
}
```

实现也很简单，在前面的基础上判断一下单位是否是`%`，是的话就按照`canvas`的宽高来计算图片要显示的宽高，第二值没有设置或者为`auto`，跟之前一样也是根据图片的宽高比来自适应。

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundSize: '50% 80%'
})

handleBackgroundSize({
    backgroundSize,
    drawOpt,
    imageRatio,
    canvasWidth: width,// 传参新增canvas的宽高
    canvasHeight: height
})

// 模拟background-size
const handleBackgroundSize = ({ backgroundSize, drawOpt, imageRatio, canvasWidth, canvasHeight }) => {
  if (backgroundSize) {
    // ...
    // 图片宽度
    let newNumberWidth = -1
    if (backgroundSizeValueArr[0]) {
      if (Array.isArray(backgroundSizeValueArr[0])) {
        // 数字+单位类型
        if (backgroundSizeValueArr[0][1] === '%') {
            // %单位，则图片显示的高度为画布的百分之多少
            drawOpt.width = backgroundSizeValueArr[0][0] / 100 * canvasWidth
            newNumberWidth = drawOpt.width
        } else {
            // 其他都认为是px单位
            drawOpt.width = backgroundSizeValueArr[0][0]
            newNumberWidth = backgroundSizeValueArr[0][0]
        }
      } else if (backgroundSizeValueArr[0] === 'auto') {
        // auto类型，那么根据设置的新高度以图片原宽高比进行自适应
        if (backgroundSizeValueArr[1]) {
            if (backgroundSizeValueArr[1][1] === '%') {
                // 高度为%单位
                drawOpt.width = zoomWidth(imageRatio, backgroundSizeValueArr[1][0] / 100 * canvasHeight)
            } else {
                // 其他都认为是px单位
                drawOpt.width = zoomWidth(imageRatio, backgroundSizeValueArr[1][0])
            }
        }
      }
    }
    // 设置了图片高度
    if (backgroundSizeValueArr[1] && Array.isArray(backgroundSizeValueArr[1])) {
      // 数字+单位类型
      if (backgroundSizeValueArr[1][1] === '%') {
        // 高度为%单位
        drawOpt.height = backgroundSizeValueArr[1][0] / 100 * canvasHeight
      } else {
        // 其他都认为是px单位
        drawOpt.height = backgroundSizeValueArr[1][0]
      }
    } else if (newNumberWidth !== -1) {
      // 没有设置图片高度或者设置为auto，那么根据设置的新宽度以图片原宽高比进行自适应
      drawOpt.height = zoomHeight(imageRatio, newNumberWidth)
    }
  }
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0112bef2839e429da2bb21d315abe8c1~tplv-k3u1fbpfcp-zoom-1.image)


## cover类型

`background-size`设置为`cover`代表图片会保持原来的宽高比，并且缩放成将完全覆盖背景定位区域的最小大小，注意，图片不会变形。

`css`样式如下：

```css
.cssBox {
    background-image: url('/3.jpeg');
    background-repeat: no-repeat;
    background-size: cover;
}
```

这个实现也很简单，根据图片的宽高比和`canvas`的宽高比判断，到底是缩放图片的宽度和`canvas`的宽度一致，还是缩放图片的高度和`canvas`的高度一致。

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundSize: 'cover'
})

handleBackgroundSize({
    backgroundSize,
    drawOpt,
    imageRatio,
    canvasWidth: width,
    canvasHeight: height,
    canvasRatio// 参数增加canvas的宽高比
})

const handleBackgroundSize = ({
  backgroundSize,
  drawOpt,
  imageRatio,
  canvasWidth,
  canvasHeight,
  canvasRatio
}) => {
    // ...
    // 值为cover
    if (backgroundSizeValueArr[0] === 'cover') {
        if (imageRatio > canvasRatio) {
            // 图片的宽高比大于canvas的宽高比，那么图片高度缩放到和canvas的高度一致，宽度自适应
            drawOpt.height = canvasHeight
            drawOpt.width = zoomWidth(imageRatio, canvasHeight)
        } else {
            // 否则图片宽度缩放到和canvas的宽度一致，高度自适应
            drawOpt.width = canvasWidth
            drawOpt.height = zoomHeight(imageRatio, canvasWidth)
        }
        return
    }
    // ...
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2fb50ee85bf749c29b77ab7653e551d0~tplv-k3u1fbpfcp-zoom-1.image)


## contain类型

`background-size`设置为`contain`类型表示图片还是会保持原有的宽高比，并且缩放成适合背景定位区域的最大大小，也就是图片会显示完整，但是不一定会铺满背景的水平和垂直两个方向，在某个方向可能会有留白。

`css`样式：

```css
.cssBox {
    background-image: url('/1.jpg');
    background-repeat: no-repeat;
    background-size: contain;
}
```

实现刚好和`cover`类型的实现反过来即可，如果图片的宽高比大于`canvas`的宽高比，为了让图片显示完全，让图片的宽度和`canvas`的宽度一致，高度自适应。

```js
const handleBackgroundSize = () => {
    // ...
    // 值为contain
    if (backgroundSizeValueArr[0] === 'contain') {
        if (imageRatio > canvasRatio) {
            // 图片的宽高比大于canvas的宽高比，那么图片宽度缩放到和canvas的宽度一致，高度自适应
            drawOpt.width = canvasWidth
            drawOpt.height = zoomHeight(imageRatio, canvasWidth)
        } else {
            // 否则图片高度缩放到和canvas的高度一致，宽度自适应
            drawOpt.height = canvasHeight
            drawOpt.width = zoomWidth(imageRatio, canvasHeight)
        }
        return
    }
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0ce2375e7da447695e590173cd93430~tplv-k3u1fbpfcp-zoom-1.image)


到这里对`background-size`的模拟就结束了，接下来看看`background-position`。

# 模拟background-position属性

先看不设置`background-size`的情况。

`background-position`属性用于设置背景图像的起始位置，默认值为` 0% 0%`，它也支持几种不同类型的值，一一来看。

## percentage类型

第一个值设置水平位置，第二个值设置垂直位置。左上角是`0％0％`，右下角是`100％100％`，如果只设置了一个值，第二个默认为`50%`，比如设置为`50% 60%`，意思是将图片的`50% 60%`位置和背景区域的`50% 60%`位置进行对齐，又比如`50% 50%`，代表图片中心点和背景区域中心点重合。

`css`样式：

```css
.cssBox {
    background-image: url('/2.jpg');
    background-repeat: no-repeat;
    background-position: 50% 50%;
}
```

实现上我们只需要用到`drawImage`方法的`img`、`x、y`三个参数，图片的宽高不会进行缩放，根据比例分别算出在`canvas`和图片上对应的距离，他们的差值即为图片在`canvas`上显示的位置。

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundPosition: '50% 50%'
})

const drawBackgroundImageToCanvas = () => {
    // ...
    // 模拟background-position
    handleBackgroundPosition({
      backgroundPosition,
      drawOpt,
      imgWidth,
      imgHeight,
      canvasWidth: width,
      canvasHeight: height
    })
    // ...
}

// 模拟background-position
const handleBackgroundPosition = ({
  backgroundPosition,
  drawOpt,
  imgWidth,
  imgHeight,
  canvasWidth,
  canvasHeight
}) => {
  if (backgroundPosition) {
    // 将值转换成数组
    let backgroundPositionValueArr = getNumberValueFromStr(backgroundPosition)
    if (Array.isArray(backgroundPositionValueArr[0])) {
      if (backgroundPositionValueArr.length === 1) {
        // 如果只设置了一个值，第二个默认为50%
        backgroundPositionValueArr.push([50, '%'])
      }
      // 水平位置
      if (backgroundPositionValueArr[0][1] === '%') {
        // 单位为%
        let canvasX = (backgroundPositionValueArr[0][0] / 100) * canvasWidth
        let imgX = (backgroundPositionValueArr[0][0] / 100) * imgWidth
        // 计算差值
        drawOpt.x = canvasX - imgX
      }
      // 垂直位置
      if (backgroundPositionValueArr[1][1] === '%') {
        // 单位为%
        let canvasY = (backgroundPositionValueArr[1][0] / 100) * canvasHeight
        let imgY = (backgroundPositionValueArr[1][0] / 100) * imgHeight
        // 计算差值
        drawOpt.y = canvasY - imgY
      }
    }
  }
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43adca0cd4cd4562b3d8b6699ddccc11~tplv-k3u1fbpfcp-zoom-1.image)


## length类型

第一个值代表水平位置，第二个值代表垂直位置。左上角是`0 0`。单位可以是`px`或任何其他`css`单位，当然，我们只考虑`px`。如果仅指定了一个值，其他值将是`50％`。所以你可以混合使用`%`和`px`。

`css`样式：

```css
.cssBox {
    background-image: url('/2.jpg');
    background-repeat: no-repeat;
    background-position: 50px 150px;
}
```

这个实现更简单，直接把值传给`drawImage`的`x、y`参数即可。

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundPosition: '50px 150px'
})

// 模拟background-position
const handleBackgroundPosition = ({}) => {
    // ...
    // 水平位置
    if (backgroundPositionValueArr[0][1] === '%') {
        // ...
    } else {
        // 其他单位默认都为px
        drawOpt.x = backgroundPositionValueArr[0][0]
    }
    // 垂直位置
    if (backgroundPositionValueArr[1][1] === '%') {
        // ...
    } else {
        // 其他单位默认都为px
        drawOpt.y = backgroundPositionValueArr[1][0]
    }
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4ed2302d5e04af781fec316304699b3~tplv-k3u1fbpfcp-zoom-1.image)


## 关键词类型

也就是通过`left`、`top`之类的关键词进行组合，比如：`left top`、`center center`、`center bottom`等。可以看做是特殊的`%`值，所以我们只要写一个映射将这些关键词对应上百分比数值即可。

```css
.cssBox {
    background-image: url('/2.jpg');
    background-repeat: no-repeat;
    background-position: right bottom;
}
```

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundPosition: 'right bottom'
})

// 关键词到百分比值的映射
const keyWordToPercentageMap = {
  left: 0,
  top: 0,
  center: 50,
  bottom: 100,
  right: 100
}

const handleBackgroundPosition = ({}) => {
    // ...
    // 将关键词转为百分比
    backgroundPositionValueArr = backgroundPositionValueArr.map(item => {
      if (typeof item === 'string') {
        return keyWordToPercentageMap[item] !== undefined
          ? [keyWordToPercentageMap[item], '%']
          : item
      }
      return item
    })
    // ...
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52da9c729f1044f5aea150891541af19~tplv-k3u1fbpfcp-zoom-1.image)


## 和background-size组合

最后我们来看看和`background-size`组合使用会发生什么情况。

```css
.cssBox {
    background-image: url('/2.jpg');
    background-repeat: no-repeat;
    background-size: cover;
    background-position: right bottom;
}
```

```js
drawBackgroundImageToCanvas(ctx, width, height, this.img, {
    backgroundSize: 'cover',
    backgroundPosition: 'right bottom'
})
```

结果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e7020f8c2b242e7941bb3329dc6c888~tplv-k3u1fbpfcp-zoom-1.image)


不一致，这是为啥呢，我们来梳理一下，首先在处理`background-size`会计算出`drawImage`参数中的`width、height`，也就是图片在`canvas`中显示的宽高，而在处理`background-position`时会用到图片的宽高，但是我们传的还是图片的原始宽高，这样计算出来当然是有问题的，修改一下：

```js
// 模拟background-position
handleBackgroundPosition({
    backgroundPosition,
    drawOpt,
    imgWidth: drawOpt.width,// 改为传计算后的图片的显示宽高
    imgHeight: drawOpt.height,
    imageRatio,
    canvasWidth: width,
    canvasHeight: height,
    canvasRatio
})
```

现在再来看看效果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/403caf0e0a4f46fe913f9fe134cbad5d~tplv-k3u1fbpfcp-zoom-1.image)


# 模拟background-repeat属性

`background-repeat`属性用于设置如何平铺对象的`background-image`属性，默认值为`repeat`，也就是当图片比背景区域小时默认会向垂直和水平方向重复，另外还有几个可选值：

- `repeat-x`：只有水平位置会重复背景图像
- `repeat-y`：只有垂直位置会重复背景图像
- `no-repeat`：`background-image`不会重复

接下来我们实现一下这几种情况。

## no-repeat

首先判断图片的宽高是否都比背景区域大，是的话就不需要平铺，也就不用处理，另外值为`no-repeat`也不需要做处理：

```js
// 模拟background-repeat
handleBackgroundRepeat({
    backgroundRepeat,
    drawOpt,
    imgWidth: drawOpt.width,
    imgHeight: drawOpt.height,
    imageRatio,
    canvasWidth: width,
    canvasHeight: height,
    canvasRatio
})
```

可以看到这里我们传的图片的宽高也是经`background-size`计算后的图片显示宽高。

```js
// 模拟background-repeat
const handleBackgroundRepeat = ({
  backgroundRepeat,
  drawOpt,
  imgWidth,
  imgHeight,
  canvasWidth,
  canvasHeight,
}) => {
    if (backgroundRepeat) {
        // 将值转换成数组
        let backgroundRepeatValueArr = getNumberValueFromStr(backgroundRepeat)
        // 不处理
        if (backgroundRepeatValueArr[0] === 'no-repeat' || (imgWidth >= canvasWidth && imgHeight >= canvasHeight)) {
            return
        }
    }
}
```

## repeat-x

接下来增加对`repeat-x`的支持，当`canvas`的宽度大于图片的宽度，那么水平平铺进行绘制，绘制会重复调用`drawImage`方法，所以还需要再传递`ctx`、`image`参数给`handleBackgroundRepeat`方法，另外如果`handleBackgroundRepeat`方法里进行了绘制，原来的绘制方法就不用再调用了：

```js
// 模拟background-repeat
// 如果在handleBackgroundRepeat里进行了绘制，那么会返回true
let notNeedDraw = handleBackgroundRepeat({
    ctx,
    image,
    ...
})
if (!notNeedDraw) {
    drawImage(ctx, image, drawOpt)
}

// 根据参数绘制图片
const drawImage = (ctx, image, drawOpt) => {
  ctx.drawImage(
    image,
    drawOpt.sx,
    drawOpt.sy,
    drawOpt.swidth,
    drawOpt.sheight,
    drawOpt.x,
    drawOpt.y,
    drawOpt.width,
    drawOpt.height
  )
}
```

将绘制的方法提取成了一个方法，方便复用。

```js
const handleBackgroundRepeat = ({}) => {
    // ...
    // 水平平铺
    if (backgroundRepeatValueArr[0] === 'repeat-x') {
      if (canvasWidth > imgWidth) {
        let x = 0
        while (x < canvasWidth) {
          drawImage(ctx, image, {
            ...drawOpt,
            x
          })
          x += imgWidth
        }
        return true
      }
    }
    // ...
}
```

每次更新图片的放置位置`x`参数，直到超出`canvas`的宽度。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d27f12ee3d374f4ea31a44fcf2b0c987~tplv-k3u1fbpfcp-zoom-1.image)


## repeat-y

对`repeat-y`的处理也是类似的：

```js
const handleBackgroundRepeat = ({}) => {
    // ...
    // 垂直平铺
    if (backgroundRepeatValueArr[0] === 'repeat-y') {
      if (canvasHeight > imgHeight) {
        let y = 0
        while (y < canvasHeight) {
          drawImage(ctx, image, {
            ...drawOpt,
            y
          })
          y += imgHeight
        }
        return true
      }
    }
    // ...
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cc7639fcd0f45558f624108225e5dd3~tplv-k3u1fbpfcp-zoom-1.image)


## repeat

最后就是`repeat`值，也就是水平和垂直都进行重复：

```js
const handleBackgroundRepeat = ({}) => {
    // ...
    // 平铺
    if (backgroundRepeatValueArr[0] === 'repeat') {
      let x = 0
      while (x < canvasWidth) {
        if (canvasHeight > imgHeight) {
          let y = 0
          while (y < canvasHeight) {
            drawImage(ctx, image, {
              ...drawOpt,
              x,
              y
            })
            y += imgHeight
          }
        }
        x += imgWidth
      }
      return true
    }
}
```

从左到右，一列一列进行绘制，水平方向绘制到`x`超出`canvas`的宽度为止，垂直方向绘制到`y`超出`canvas`的高度为止。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95f8166f06af40abb58b871ddf7f7fc3~tplv-k3u1fbpfcp-zoom-1.image)


## 和background-size、background-position组合

最后同样看一下和前两个属性的组合情况。

`css`样式：

```css
.cssBox {
    background-image: url('/4.png');
    background-repeat: repeat;
    background-size: 50%;
    background-position: 50% 50%;
}
```

效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86837c5722004e83b96c837564826837~tplv-k3u1fbpfcp-zoom-1.image)


图片大小是正确的，但是位置不正确，`css`的做法应该是先根据`background-position`的值定位一张图片，然后再向四周进行平铺，而我们显然忽略了这种情况，每次都从`0 0`位置开始绘制。

知道了原理，解决也很简单，在`handleBackgroundPosition`方法中已经计算出了`x、y`，也就是没有平铺前第一张图片的放置位置：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d54da7bf02e45348d065cf37e5a8c5e~tplv-k3u1fbpfcp-zoom-1.image)


我们只要计算出左边和上边还能平铺多少张图片，把水平和垂直方向上第一张图片的位置计算出来，作为后续循环的`x、y`的初始值即可。

```js
const handleBackgroundRepeat = ({}) => {
    // 保存在handleBackgroundPosition中计算出来的x、y
    let ox = drawOpt.x
    let oy = drawOpt.y
    // 计算ox和oy能平铺的图片数量
    let oxRepeatNum = Math.ceil(ox / imgWidth)
    let oyRepeatNum = Math.ceil(oy / imgHeight)
    // 计算ox和oy第一张图片的位置
    let oxRepeatX = ox - oxRepeatNum * imgWidth 
    let oxRepeatY = oy - oyRepeatNum * imgHeight
    // 将oxRepeatX和oxRepeatY作为后续循环的x、y的初始值
    // ...
    // 平铺
    if (backgroundRepeatValueArr[0] === 'repeat') {
      let x = oxRepeatX
      while (x < canvasWidth) {
        if (canvasHeight > imgHeight) {
          let y = oxRepeatY
          // ...
        }
      }
    }
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eaf124b33d5a49779c0a31fa3c4b68a3~tplv-k3u1fbpfcp-zoom-1.image)


# 结尾

本文简单实现了一下在`canvas`中模拟`css`的`background-size`、`background-position`、`background-repeat`三个属性的部分效果，完整源码在[https://github.com/wanglin2/simulateCSSBackgroundInCanvas](https://github.com/wanglin2/simulateCSSBackgroundInCanvas)。
