> 本文为Varlet组件库源码主题阅读系列第八篇，读完本篇，可以了解到移动端页面如何适配各种尺寸的屏幕，包括pc端，另外如何将触摸事件转换成鼠标事件。

# 移动端适配

开发移动端页面，我们通常都会按照一个固定宽度的设计稿来做，但是实际上的手机屏幕尺寸五花八门，如果不进行适配的话会比较影响使用体验。

`Varlet`组件库的设计就是基于`375px`宽度的设计稿，然后使用[postcss-px-to-viewport](https://github.com/evrone/postcss-px-to-viewport)进行移动端适配，这个`PostCSS`插件会将`px`单位转换成`vw`单位，`1vw`等于`1/100`的视口宽度，所以使用`vw`作为单位就会随着视口的宽度进行变化达到适配不同机型的效果。

`px`转`vw`也很简单，假设某个元素的宽高为`100px`，设计稿宽度为`375px`，那么视口也就相当于是`375px`，那么`1vw = 375 / 100 =  3.75px`，那么`100px / 3.75px = 26.66vw `，公式如下：

```
vw = px / (viewportSize / 100)
```

接下来我们从零创建一个`Vite`项目来看一下`postcss-px-to-viewport`插件的使用。

创建项目：

```bash
npm init vite@latest
```

根据选项创建一个`Vue`的项目，然后写一个非常简单的按钮：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ec63135ce644882ab7d6ff089a35a35~tplv-k3u1fbpfcp-zoom-1.image)


接下来安装依赖和启动服务，效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50cb4fb70a534ecc83f9a141e7d260fc~tplv-k3u1fbpfcp-zoom-1.image)


假设我们的设计稿就是`375px`，那么我们切换到尺寸更大一点的机型看看：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61265d35647b4f3192319d5ba814281c~tplv-k3u1fbpfcp-zoom-1.image)


直接上`iPad`，可以看到按钮尺寸没有变，但是因为屏幕变大了而显得按钮太小了，这显然是不够友好的，接下来我们就配置一下`postcss-px-to-viewport`插件。

这个插件本身是一个`PostCSS`的插件，所以首先要支持`PostCss`，在`Vite`项目中使用`PostCSS`很简单，只要项目中包含有效的`PostCSS` 配置，`Vite`就会自动使其应用于所有导入的`CSS`，所以我们要做的就是增加一个`PostCSS` 配置，参考`postcss-px-to-viewport`插件文档，先安装：

```bash
npm install postcss-px-to-viewport
```

然后创建`postcss.config.js`文件，写入如下内容：

```js
module.exports = {
  plugins: {
    "postcss-px-to-viewport": {
      // 需要转换的单位
      unitToConvert: "px",
      // 设计稿的视口宽度
      viewportWidth: 375,
      // 单位转换后保留的精度
      unitPrecision: 4,
    },
  },
};
```

再次启动服务看看效果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ee09eecfd0441fd83d79a66111b5665~tplv-k3u1fbpfcp-zoom-1.image)


报错了，虽然不知道为什么会把这个配置文件也当成`ES Module`解析，但是解决方法很简单，把后缀名改成`.cjs`即可，再次重启：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31005990449a4fc3a5b6eab271b2d895~tplv-k3u1fbpfcp-zoom-1.image)


可以看到按钮变大了，单位也由我们书写的`px`变成了`vw`。



# 桌面端适配

这个适配指的不是尺寸，因为前面已经使用`vw`解决了尺寸的适配问题，这里主要是指事件，具体来说是我们在移动端使用的交互事件一般是`touch`事件，但是桌面端肯定不支持，所以为了让我们的移动端组件库不至于在桌面端完全无法使用，需要将`touch`事件转成`mouse`事件。

`Varlet`使用的是`@varlet/touch-emulator`这个包来实现的，使用也很简单，安装：

```bash
npm i @varlet/touch-emulator
```

导入：

```js
import '@varlet/touch-emulator'
```

接下来修改一下我们上面的示例，给按钮增加一个`touchstart`事件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2c1917e232b4b4fba2dfdf9a0799340~tplv-k3u1fbpfcp-zoom-1.image)


然后分别在模拟器和非模拟器环境下单击一下按钮：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b041cbd284243e0a5cc3d44d20a9bba~tplv-k3u1fbpfcp-zoom-1.image)



![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/280bfb030e1f4b01b0da55432328b516~tplv-k3u1fbpfcp-zoom-1.image)


显然，非模拟器环境下单击是没有效果的，接下来配置一下` @varlet/touch-emulator`，再次查看非模拟器环境下的点击效果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/439f5565d3384c299c0ddd68cdb63d02~tplv-k3u1fbpfcp-zoom-1.image)


可以看到成功触发了。

接下来就来窥探一下` @varlet/touch-emulator`都做了些什么。

```js
// 判断是否是浏览器环境
const inBrowser = typeof window !== 'undefined'
// 判断该环境是否支持touch事件
const supportTouch = inBrowser && 'ontouchstart' in window
// ...
```

首先进行了一下环境判断，如果不满足这两个条件就不需要做任何处理。

```js
// ...
if (inBrowser && !supportTouch) {
  createTouchEmulator()
}
// ...
```

满足条件则调用`createTouchEmulator`方法：

```js
// ...
function createTouchEmulator() {
  window.addEventListener('mousedown', (event) => onMouse(event, 'touchstart'), true)
  window.addEventListener('mousemove', (event) => onMouse(event, 'touchmove'), true)
  window.addEventListener('mouseup', (event) => onMouse(event, 'touchend'), true)
}
// ...
```

监听了三个鼠标事件，分别对应三个`touch`事件，注意`addEventListener`方法第三个参数都传了`true`，这个参数默认是`false`，表示在事件冒泡的阶段调用事件处理函数，传`true`就表示在事件捕获的阶段调用事件处理函数，举个栗子，比如我们给页面上的一个`div`也绑定了`mousedown`事件，然后当我们鼠标在这个`div`上按下，如果是冒泡阶段，那么`div`的事件函数会先被调用，如果是捕获阶段，那么`window`的事件函数会先被调用，所以这里传`true`笔者猜测是因为如果是冒泡阶段触发的话，某个元素的可能会阻止冒泡，那么就不会触发`window`上绑定的这几个事件了。

这几个处理方法内都调用了`onMouse`方法：

```js
// ...
let initiated = false
let eventTarget
function onMouse(mouseEvent, touchType) {
  // 事件类型、事件目标
  const { type, target } = mouseEvent
  // mousedown = true（mousedown事件）
  //             false（mouseup事件）
  //             保持（mousemove事件）
  initiated = isMousedown(type) ? true : isMouseup(type) ? false : initiated
  // 如果是鼠标移动事件且鼠标没有按下则返回
  if (isMousemove(type) && !initiated) return
  // 判断是否要更新事件目标
  if (isUpdateTarget(type)) eventTarget = target
  // 手动构造对应的touch事件并触发
  triggerTouch(touchType, mouseEvent)
  // 如果鼠标松开了则清除保存的事件目标
  if (isMouseup(type)) eventTarget = null
}

const isMousedown = (eventType) => eventType === 'mousedown'
const isMousemove = (eventType) => eventType === 'mousemove'
const isMouseup = (eventType) => eventType === 'mouseup'
// ...
```

这个方法首先根据鼠标事件的类型设置了`initiated`变量，记录鼠标的按下状态，如果是鼠标移动事件且鼠标没有按下，那么个方法会直接返回，因为`touch`事件都需要先按下才会触发，然后调用了`isUpdateTarget`方法判断是否要更新事件目标：

```js
const isUpdateTarget = (eventType) =>
  isMousedown(eventType) || !eventTarget || (eventTarget && !eventTarget.dispatchEvent)
```

鼠标按下显然对应的是`touchstart`，触发的第一个`touch`事件，事件目标肯定也是新的，所以需要更新，理论上不同手指的事件目标是可能不一样的，但是由于桌面端鼠标事件只能有一个，所以直接用一个变量保存即可。

`eventTarget`不存在当然也需要更新，但是笔者觉得这种情况应该不会出现，因为`touchstart`或者说是`mousedown`事件肯定是最先被触发的，`eventTarget`应该已经有值了。

第三个条件笔者也没有理解，按理说只要是`DOM`元素应该都会有`dispatchEvent`方法。

接下来调用了`triggerTouch`方法：

```js
// ...
function triggerTouch(touchType, mouseEvent) {
  const { altKey, ctrlKey, metaKey, shiftKey } = mouseEvent;
  // bubbles：该事件是否冒泡
  // cancelable：该事件能否被取消
  const touchEvent = new Event(touchType, { bubbles: true, cancelable: true });
  // 设置几个键的按下标志
  touchEvent.altKey = altKey;
  touchEvent.ctrlKey = ctrlKey;
  touchEvent.metaKey = metaKey;
  touchEvent.shiftKey = shiftKey;
  // 设置三种类型的触摸点对象数据
  touchEvent.touches = getActiveTouches(mouseEvent);
  touchEvent.targetTouches = getActiveTouches(mouseEvent);
  touchEvent.changedTouches = createTouchList(mouseEvent);
  // 派发事件
  eventTarget.dispatchEvent(touchEvent);
}
// ...
```

先手动创建一个对应类型的`touch`[Event](https://developer.mozilla.org/zh-CN/docs/Web/API/Event)对象，设置该事件支持冒泡，然后设置了相关按键的按下状态，笔者也是才知道`TouchEvent`事件是需要这几个属性的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45b4516e231d4f03868b5b52885f5f71~tplv-k3u1fbpfcp-zoom-1.image)


然后设置触摸点数据，一共有三种类型：

- `touches`：当前屏幕上所有触摸点的列表
- `targetTouches`：当前对象上所有触摸点的列表
- `changedTouches`：涉及当前(引发)事件的触摸点的列表

移动端触摸点是可能存在多个的，比如我同时好几个手指一起触摸，可以通过这三个列表进行区分，同样举个栗子，比如我给一个`div`绑定了三个`touch`事件，第一次我一个手指触摸到`div`上，此时这三个列表的值是一样的，就是第一个手指的触摸点，然后我第二个手指也开始触摸，但是不是触摸到`div`上，而是其他元素上，那么此时`touches`列表会包含两个手指的触摸点，`targetTouches`列表只会包含第一个手指的触摸点，`changedTouches`列表则为第二个手指的触摸点。手指全部松开后，这三个列表都将为空。

但是在桌面端，鼠标触摸点显然只有一个，所以这三个列表其实都是相同的。

`touches`和`targetTouches`都调用了`getActiveTouches`方法获取：

```js
// ...
function getActiveTouches(mouseEvent) {
  const { type } = mouseEvent;
  if (isMouseup(type)) return createTouchList();
  return updateTouchList(mouseEvent);
}
// ...
```

松开事件`touchList`是空的，所以返回一个空列表即可，调用的是`createTouchList`方法：

```js
// ...
function createTouchList() {
  const touchList = [];

  touchList.item = function (index) {
    return this[index] || null;
  };

  return touchList;
}
// ...
```

原生的[TouchList](https://developer.mozilla.org/zh-CN/docs/Web/API/TouchList)对象存在一个`item`方法，返回列表中以指定值作为索引的 [`Touch`](https://developer.mozilla.org/zh-CN/docs/Web/API/Touch) 对象，所以使用数组来代表`TouchList`需要自行提供一个同名方法。

其他事件类型则会调用`updateTouchList`方法：

```js
// ...
function updateTouchList(mouseEvent) {
  const touchList = createTouchList();

  touchList.push(new Touch(eventTarget, 1, mouseEvent));
  return touchList;
}
// ...
```

同样先创建了一个`touchList`，然后创建了一个[Touch](https://developer.mozilla.org/zh-CN/docs/Web/API/Touch)实例添加进去，这个`Touch`类定义如下，模拟的是原生的[Touch](https://developer.mozilla.org/zh-CN/docs/Web/API/Touch)对象：

```js
// ...
function Touch(target, identifier, mouseEvent) {
  const { clientX, clientY, screenX, screenY, pageX, pageY } = mouseEvent;

  this.identifier = identifier;
  this.target = target;
  this.clientX = clientX;
  this.clientY = clientY;
  this.screenX = screenX;
  this.screenY = screenY;
  this.pageX = pageX;
  this.pageY = pageY;
}
// ...
```

`changedTouches`直接调用的是`createTouchList`方法，显然无论何时返回的都是空的列表，这个似乎是有点问题的，因为前面说了，只有一个触摸点的话这三个列表的值应该都是一样的。

最后在事件目标上进行了事件的派发。

总结一下，整体所做的事情就是监听鼠标的三个事件，然后手动创建对应的`touch`事件对象，最后在事件目标元素上进行派发即可。
