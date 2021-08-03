---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight: juejin
---
## 开头

`TypeScript`已经出来很多年了，现在用的人也越来越多，毋庸置疑，它会越来越流行，但是我还没有用过，因为首先是项目上不用，其次是我对强类型并不敏感，所以纯粹的光看文档看不了几分钟就心不在焉，一直就被耽搁了。

但是，现在很多流行的框架都开始用`TypeScript`重构，很多文章的示例代码也变成`TypeScript`，所以这就很尴尬了，你不会就看不懂，所以好了，没得选了。

既然目前我的痛点是看源码看不懂，那不如就在看源码的过程中遇到不懂的`TypeScript`语法再去详细了解，这样可能比单纯看文档更有效，接下来我将在阅读[BetterScroll](https://github.com/ustbhuangyi/better-scroll)源码的同时恶补`TypeScript`。

`BetterScroll`是一个针对移动端的滚动库，使用纯`JavaScript`，2.0版本使用`TypeScript`进行了重构，通过插件化将功能进行了分离，核心只保留基本的滚动功能。

方便起见，后续`TypeScript`缩写为`TS`，`BetterScroll`缩写为`BS`。

`BS`的核心功能代码在`/packages/core/`文件夹下，结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3147823fc9db4cd0acbd1e8f3991a59b~tplv-k3u1fbpfcp-watermark.image)

`index.ts`文件只用来对外暴露接口，我们从`BScroll.ts`开始阅读。

## 入口类

```ts
interface PluginCtor {
  pluginName: string
  applyOrder?: ApplyOrder
  new (scroll: BScroll): any
}
```

`interface`接口用来定义值的结构，之后`TS`的类型检查器就会对值进行检查，上面的`PluginCtor`接口用来对`BS`的插件对象结构进行定义及限制，意思为需要一个必填的字符串类型插件名称`pluginName`，`?`的意思为可选，可有可不有的`ApplyOrder`类型的调用位置，找到`ApplyOrder`的定义：

```ts
export const enum ApplyOrder {
  Pre = 'pre',
  Post = 'post'
}
```

`enum`的意思是枚举，可以定义一些带名字的常量，使用枚举可以清晰的知道可选的选项是什么，枚举支持数字枚举和字符串枚举，数字枚举还有自增的功能，上述通过`const`来修饰的枚举称为常量枚举，常量枚举的特点是在编译阶段会被删除而直接内联到使用的地方。

回到接口，`interface`可以为类和实例来定义接口，这里有个`new`意味着这是为类定义的接口，这里我们就可以知道`BS`的插件主体需要是一个类，且有两个静态属性，构造函数入参是`BS`的实例，`any`代表任何类型。

再往下：

```ts
interface PluginsMap {
  [key: string]: boolean
}
```

这里同样是个接口定义，`[key: string]`的属性称作索引签名，因为`TS`会对对象字面量进行额外属性检查，即出现了接口里没有定义的属性时会认为是个错误，解决这个问题的其中一个方法就是在接口定义里增加索引签名。

```ts
type ElementParam = HTMLElement | string
```

`type`意为类型别名，相当于给一个类型起了一个别名，不会新建类型，是一种引用关系，使用的时候和接口差不多，但是有一些细微差别。

`|`代表联合类型，表示一个值可以是几种类型之一。

```ts
export interface MountedBScrollHTMLElement extends HTMLElement {
  isBScrollContainer?: boolean
}
```

接口是可以继承的，继承能从一个接口里复制成员到另一个接口里，增加可重用性。

```ts
export class BScrollConstructor<O = {}> extends EventEmitter {}
```

`<o = {}>`，`<>`称为泛型，即可以支持多种类型，不限制为具体的一种，为扩展提供了可能，也比使用`any`严谨，`<>`就像`()`一样，调用的时候传入类型，`<>`里的参数来接收，`<>`里的参数称为类型变量，比如下面的泛型函数：

```ts
function fn<T>(arg: T): T {}
fn<Number>(1)
```

表示入参和返回参数的类型都是`Number`，除了`<>`，入参里的`T`和返回参数类型的`T`可以理解为是占位符。

```ts
static plugins: PluginItem[] = []
```

`[]`代表数组类型，定义数组有两种方式：

```ts
let list: number[] = [1,2,3]// 1.元素类型后面跟上[]
let list: Array<number> = [1,2,3]// 2.使用数组泛型，Array<元素类型>
```

所以上面的意思是定义了一个元素类型是`PluginItem`的数组。

`BS`使用插件需要在`new BS`之前调用`use`方法，`use`是`BS`类的一个静态方法：

```ts
class BS {
    static use(ctor: PluginCtor) {
        const name = ctor.pluginName
        // 插件名称检查、插件是否已经注册检查...
        BScrollConstructor.pluginsMap[name] = true
        BScrollConstructor.plugins.push({
          name,
          applyOrder: ctor.applyOrder,
          ctor,
        })
        return BScrollConstructor
      }
}
```

`use`方法就是简单的把插件添加到`plugins`数组里。

```ts
class BS {
    constructor(el: ElementParam, options?: Options & O) {
        super([
            //注册的事件名称
        ])
    	const wrapper = getElement(el)// 获取元素
    	this.options = new OptionsConstructor().merge(options).process()// 参数合并
        if (!this.setContent(wrapper).valid) {
          return
        }
        this.hooks = new EventEmitter([
          // 注册的钩子名称
        ])
    	this.init(wrapper)
  	}
}
```

构造函数做的事情是注册事件，获取元素，参数合并处理，参数处理里进行了环境检测及浏览器兼容工作，以及进行初始化。`BS`本身继承了事件对象，实例派发的叫事件，这里又创建了一个事件对象的实例`hooks`，在`BS`里为了区分叫做钩子，普通用户更关注事件，而插件开发一般要更关注钩子。

`setContent`函数的作用是设置`BS`要处理滚动的`content`，BS`默认是将`wrapper`的第一个子元素作为`content`，也可以通过配置参数来指定。

```ts
class BS {
    private init(wrapper: MountedBScrollHTMLElement) {
        this.wrapper = wrapper
        // 创建一个滚动实例
        this.scroller = new Scroller(wrapper, this.content, this.options)
        // 事件转发
        this.eventBubbling()
        // 自动失焦
        this.handleAutoBlur()
        // 启用BS，并派发对应事件
        this.enable()
        // 属性和方法代理
        this.proxy(propertiesConfig)
        // 实例化插件，遍历BS类的plugins数组挨个进行实例化，并将插件实例以key：插件名，value：插件实例保存到BS实例的plugins对象上
        this.applyPlugins()
        // 调用scroller实例刷新方法，并派发刷新事件
        this.refreshWithoutReset(this.content)
        // 下面的用来设置初始滚动的位置
        const { startX, startY } = this.options
        const position = {
          x: startX,
          y: startY,
        }
        if (
          // 如果你的插件要修改初始滚动位置，那么可以监听这个事件
          this.hooks.trigger(this.hooks.eventTypes.beforeInitialScrollTo, position)
        ) {
          return
        }
        this.scroller.scrollTo(position.x, position.y)
      }
}
```

`init`方法里做了很多事情，一一来看：

```ts
{
    private eventBubbling() {
        bubbling(this.scroller.hooks, this, [
          this.eventTypes.beforeScrollStart,
          // 事件...
        ])
      }
}
// 事件转发
export function bubbling(source,target,events) {
  events.forEach(event => {
    let sourceEvent
    let targetEvent
    if (typeof event === 'string') {
      sourceEvent = targetEvent = event
    } else {
      sourceEvent = event.source
      targetEvent = event.target
    }
    source.on(sourceEvent, function(...args: any[]) {
      return target.trigger(targetEvent, ...args)
    })
  })
}
```

BS实例的构造函数里注册了一系列事件，有些是scroller实例派发的，所以需要监听scroller对应的事件来派发自己注册的事件，相当于事件转发。

```ts
{
    private handleAutoBlur() {
        if (this.options.autoBlur) {
          this.on(this.eventTypes.beforeScrollStart, () => {
            let activeElement = document.activeElement as HTMLElement
            if (
              activeElement &&
              (activeElement.tagName === 'INPUT' ||
                activeElement.tagName === 'TEXTAREA')
            ) {
              activeElement.blur()
            }
          })
        }
      }
}
```

配置项里有一个参数：autoBlur，如果设为true会监听即将滚动的事件来将当前页面上激活的元素（input、textarea）失去焦点，`document.activeElement`可以获取文档中当前获得焦点的元素。

另外这里出现了`as`，`TS`支持的数据类型有：boolean、number、string、T[]|Array<T>、元组、枚举enum、任意any、空void、undefined、null、永不存在的值的类型never、非原始类型object，有时候你会确切的知道某个值是什么类型，可能会比`TS`更准确，那么可以通过`as`来指明它的类型，这称作类型断言，这样`TS`就不再进行判断了。

```ts
{
    proxy(propertiesConfig: PropertyConfig[]) {
        propertiesConfig.forEach(({ key, sourceKey }) => {
          propertiesProxy(this, sourceKey, key)
        })
      }
}
```

插件会有一些自己的属性和方法，`proxy`方法用来代理到`BS`实例，这样可以直接通过`BS`的实例访问，`propertiesConfig`的定义如下：

```ts
export const propertiesConfig = [
  {
    sourceKey: 'scroller.scrollBehaviorX.currentPos',
    key: 'x'
  },
  // 其他属性和方法...
]
```

```ts
export function propertiesProxy(target,sourceKey,key) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return getProperty(this, sourceKey)
  }
  sharedPropertyDefinition.set = function proxySetter(val) {
    setProperty(this, sourceKey, val)
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

通过`defineProperty`来定义属性，需要注意的是`sourceKey`的格式都是需要能让`BS`的实例`this`通过`.`能访问到源属性才行，比如这里的`this.scroller.scrollBehaviorX.currentPos`可以访问到`scroller`实例的`currentPos`属性，如果是一个插件的话，你的`propertiesConfig`需要这样：

```ts
{
    sourceKey: 'plugins.myPlugin.xxx',
    key: 'xxx'
  }
```

`plugins`是`BS`实例上的一个属性，这样通过`this.plugins.myPlugin.xxx`就能访问到你的源属性，也就能够直接通过`this`修改到源属性的属性值。所以`setProperty`和`getProperty`的逻辑也就很简单了：

```ts
const setProperty = (obj, key, value) => {
 	let keys = key.split('.')
    // 一级一级进行访问
    for(let i = 0; i < keys.length - 1; i++) {
        let tmp = keys[i]
        if (!obj[tmp]){
            obj[tmp] = {}
        } 
        obj = obj[tmp]
    }
    obj[keys.pop()] = value
}
const getProperty = (obj,key) => {
  const keys = key.split('.')
  for (let i = 0; i < keys.length - 1; i++) {
    obj = obj[keys[i]]
    if (typeof obj !== 'object' || !obj) return
  }
  const lastKey = keys.pop()
  if (typeof obj[lastKey] === 'function') {
    return function () {
      return obj[lastKey].apply(obj, arguments)
    }
  } else {
    return obj[lastKey]
  }
}
```

获取属性时如果是函数的话要特殊处理，原因是如果你这么调用的话：

```ts
let bs = new BS()
bs.xxx()// 插件的方法
```

`xxx`方法虽然是插件的方法，但是这样调用的时候`this`是指向`bs`的，但是显然，`this`应该指向这个插件实例才对，所以需要使用`apply`来指定上下文。

除上述之外，`BS`实例还有几个方法：

```ts
class BS {
    // 重新计算，一般当DOM结构发生变化后需要手动调用
    refresh() {
        // 调用setContent方法，调用scroller实例的刷新方法，派发相关事件
      }

    // 启用BS
    enable() {
        this.scroller.enable()
        this.hooks.trigger(this.hooks.eventTypes.enable)
        this.trigger(this.eventTypes.enable)
    }

    // 禁用BS
    disable() {
        this.scroller.disable()
        this.hooks.trigger(this.hooks.eventTypes.disable)
        this.trigger(this.eventTypes.disable)
    }

    // 销毁BS
    destroy() {
        this.hooks.trigger(this.hooks.eventTypes.destroy)
        this.trigger(this.eventTypes.destroy)
        this.scroller.destroy()
    }
    
    // 注册事件
    eventRegister(names: string[]) {
        this.registerType(names)
    }
}
```

都很简单，就不细说了，总的来说实例化`BS`时大致做的事情时参数处理、设置滚动元素、实例化滚动类，代理事件及方法，接下来看核心的滚动类`/scroller/Scroller.ts`。

## 滚动类

```ts
export interface ExposedAPI {
  scrollTo(
    x: number,
    y: number,
    time?: number,
    easing?: EaseItem,
    extraTransform?: { start: object; end: object }
  ): void
}
```

上述为类定义了一个接口，`scrollTo`是实例的一个方法，定义了这个方法的入参及类型、返回参数。

```ts
export default class Scroller implements ExposedAPI {
    constructor(
        public wrapper: HTMLElement,
        public content: HTMLElement,
        options: BScrollOptions
      ) {}
}
```

`public`关键字代表公开，`public`声明的属性或方法可以在类的外部使用，对应的`private`关键字代表私有的，即在类的外部不能访问，比如：

```ts
class S {
    public name: string,
    private age: number
}
let s = new S()
s.name// 可以访问
s.age// 报错
```

另外还有一个关键字`protected`，声明的变量不能在类的外部使用，但是可以在继承它的子类的内部使用，所以这个关键字如果用在`constructor`上，那么这个类只能被继承，自身不能被实例化。

对于上面这个示例，它把成员的声明和初始化合并在构造函数的参数里，称作参数属性：

```ts
constructor(public wrapper: HTMLElement)
```

```ts
class Scroller {
	constructor(
        public wrapper: HTMLElement,
        public content: HTMLElement,
        options: BScrollOptions
    ) {
        // 注册事件
        this.hooks = new EventEmitter([
            // 事件... 
        ])
        // Behavior类主要用来存储管理滚动时的一些状态
        this.scrollBehaviorX = new Behavior()
        this.scrollBehaviorY = new Behavior()
        // Translater用来获取和设置css的transform的translate属性
        this.translater = new Translater()
        // BS支持使用css3 transition和requestAnimationFrame两种方式来做动画，createAnimater会根据配置来创建对应类的实例
        this.animater = createAnimater()
        // ActionsHandler用来绑定dom事件
        this.actionsHandler = new ActionsHandler()
        // ScrollerActions用来做真正的滚动控制
        this.actions = new ScrollerActions()
        // 绑定手机的旋转事件和窗口尺寸变化事件
        this.resizeRegister = new EventRegister()
        // 监听content的transitionend事件
        this.registerTransitionEnd()
        // 监听上述类的各种事件来执行各种操作
        this.init()
    }
}
```

上面是`Scroller`类简化后的构造函数，可以看到做了非常多的事情，`new`了一堆实例，这么多挨个打开看不出一会就得劝退，所以大致的知道每个类是做什么的后，我们来简单思考一下，要能实现一个最基本的滚动大概要做一些什么事，首先肯定要先获取一些基本信息，例如`wrapper`和`content`元素的尺寸信息，然后监听事件，比如触摸事件，然后判断是否需要滚动，怎么滚动，最后进行滚动，根据这个思路我们来挨个看一下。

### 初始信息计算

获取和计算尺寸信息的在`new Behavior`的时候，构造函数里会执行`refresh`方法，我们以`scrollBehaviorY`的情况来看：

```ts
refresh(content: HTMLElement) {
    // size：height、position：top
    const { size, position } = this.options.rect
    const isWrapperStatic =
          window.getComputedStyle(this.wrapper, null).position === 'static'
    // wrapper的尺寸信息
    const wrapperRect = getRect(this.wrapper)
    // wrapper的高
    this.wrapperSize = wrapperRect[size]
    // 设置content元素，如果有变化则复位一些数据
    this.setContent(content)
    // content元素的尺寸信息
    const contentRect = getRect(this.content)
    // content元素的高
    this.contentSize = contentRect[size]
    // content距wrapper的距离
    this.relativeOffset = contentRect[position]
    // getRect方法里获取普通元素信息用的是offset相关属性，所以top是相对于offsetParent来说的，如果wrapper没有定位那么content的offsetParent则还要在上层继续查找，那么top就不是相对于wrapper的距离，需要减去wrapper的offsetTop
    if (isWrapperStatic) {
        this.relativeOffset -= wrapperRect[position]
    }
	// 设置边界，即可以滚动的最大和最小距离
    this.computeBoundary()
	// 设置默认滚动方向
    this.setDirection(Direction.Default)
}

export function getRect(el: HTMLElement): DOMRect {
  if (el instanceof (window as any).SVGElement) {
    let rect = el.getBoundingClientRect()
    return {
      top: rect.top,
      left: rect.left,
      width: rect.width,
      height: rect.height,
    }
  } else {
    return {
      top: el.offsetTop,
      left: el.offsetLeft,
      width: el.offsetWidth,
      height: el.offsetHeight,
    }
  }
}
```

看一下`computeBoundary`方法，这个方法主要获取了能滚动的最大距离，也就是两个边界值：

```ts
computeBoundary() {
    const boundary: Boundary = {
        minScrollPos: 0,// 可以理解为translateY的最小值
        maxScrollPos: this.wrapperSize - this.contentSize,// 可以理解为translateY的最大值
    }
    // wrapper的高小于content的高，那么显然是需要滚动的
    if (boundary.maxScrollPos < 0) {
        // 因为content是相对于自身的位置进行偏移的，所以如果前面还有元素占了位置的话即使滚动了maxScrollPos的距离后还会有一部分是不可见的，需要继续向上滚动relativeOffset的距离
        boundary.maxScrollPos -= this.relativeOffset
        // 这里属实没看懂，但是一般offsetTop为0的话这里也不影响
        if (this.options.specifiedIndexAsContent === 0) {
            boundary.minScrollPos = -this.relativeOffset
        }
    }
    this.minScrollPos = boundary.minScrollPos
    this.maxScrollPos = boundary.maxScrollPos
    // 判断是否需要滚动
    this.hasScroll =
        this.options.scrollable && this.maxScrollPos < this.minScrollPos
    if (!this.hasScroll && this.minScrollPos < this.maxScrollPos) {
        this.maxScrollPos = this.minScrollPos
        this.contentSize = this.wrapperSize
    }
}
```

首先要搞明白的是滚动是作用在`content`元素上的，[https://better-scroll.github.io/examples/#/core/specified-content](https://better-scroll.github.io/examples/#/core/specified-content)，这个示例可以很清楚的看到，`wrapper`里非`content`的元素是不会动的。

### 事件监听处理

接下来就是监听事件，这个在`ActionsHandler`里，分`pc`和手机端绑定了鼠标和触摸两套事件，处理函数其实都是同一个，我们以触摸事件来看，有`start`触摸开始、`move`触摸中、`end`触摸结束三个事件处理函数。

```ts
private start(e: TouchEvent) {
    // 鼠标相关事件的type为1，触摸为2
    const _eventType = eventTypeMap[e.type]
	// 避免鼠标和触摸事件同时作用？
    if (this.initiated && this.initiated !== _eventType) {
      return
    }
    // 设置initiated的值
    this.setInitiated(_eventType)
	// 如果检查到配置了某些元素不需要响应滚动，这里直接返回
    if (tagExceptionFn(e.target, this.options.tagException)) {
      this.setInitiated()
      return
    }
	// 只允许鼠标左键单击
    if (_eventType === EventType.Mouse && e.button !== MouseButton.Left) return
	// 这里根据配置来判断是否要阻止冒泡和阻止默认事件
    this.beforeHandler(e, 'start')
	// 记录触摸开始的点距页面的距离，pageX和pageY会包括页面被卷去部分的长度
    let point = (e.touches ? e.touches[0] : e) as Touch
    this.pointX = point.pageX
    this.pointY = point.pageY
  }
```

触摸开始事件最主要的就是记录一下触摸点的位置。

```ts
private move(e: TouchEvent) {
    let point = (e.touches ? e.touches[0] : e) as Touch
    // 计算触摸移动的差值
    let deltaX = point.pageX - this.pointX
    let deltaY = point.pageY - this.pointY
    this.pointX = point.pageX
    this.pointY = point.pageY
	// 页面被卷去的长度
    let scrollLeft =
      document.documentElement.scrollLeft ||
      window.pageXOffset ||
      document.body.scrollLeft
    let scrollTop =
      document.documentElement.scrollTop ||
      window.pageYOffset ||
      document.body.scrollTop
	// 当前触摸的位置距离视口的位置，为什么不用clientX、clientY？
    let pX = this.pointX - scrollLeft
    let pY = this.pointY - scrollTop
	// 如果你快速滑动幅度过大的时候可能手指会滑出屏幕导致没有触发touchend事件，这里就是进行判断，当你的手指位置距离边界小于某个值时就自动调用end方法来结束本次滑动
    const autoEndDistance = this.options.autoEndDistance
    if (
      pX > document.documentElement.clientWidth - autoEndDistance ||
      pY > document.documentElement.clientHeight - autoEndDistance ||
      pX < autoEndDistance ||
      pY < autoEndDistance
    ) {
      this.end(e)
    }
  }
```

触摸中的方法主要做了两件事，记录和上次滑动的差值以及满足条件自动结束滚动。

```ts
private end(e: TouchEvent) {
    // 复位initiated的值，这样move事件就不会再响应
    this.setInitiated()
    // 派发事件
    this.hooks.trigger(this.hooks.eventTypes.end, e)
  }
```

### 滚动逻辑

以上仍只是绑定了事件，还没到滚动那一步，接下来看`ScrollerActions`，构造函数里调用了`bindActionsHandler`方法，这个方法里监听了刚才`actionsHandler`里绑定的那些事件：

```ts
private bindActionsHandler() {
    // [mouse|touch]触摸开始事件
    this.actionsHandler.hooks.on(
      this.actionsHandler.hooks.eventTypes.start,
      (e: TouchEvent) => {
        if (!this.enabled) return true
        return this.handleStart(e)
      }
    )
    // [mouse|touch]触摸中事件
    this.actionsHandler.hooks.on(
      this.actionsHandler.hooks.eventTypes.move,
      ({ deltaX, deltaY, e}) => {
        if (!this.enabled) return true
        return this.handleMove(deltaX, deltaY, e)
      }
    )
    // [mouse|touch]触摸结束事件
    this.actionsHandler.hooks.on(
      this.actionsHandler.hooks.eventTypes.end,
      (e: TouchEvent) => {
        if (!this.enabled) return true
        return this.handleEnd(e)
      }
    )
  }
```

接下来是上面三个事件对应的处理函数：

```ts
private handleStart(e: TouchEvent) {
    // 获取触摸开始的时间戳
    const timestamp = getNow()
    this.moved = false
    this.startTime = timestamp
    // directionLockAction主要是用来做方向锁定的，比如判断某次滑动时应该进行水平滚动还是垂直滚动等，reset方法是复位锁定的方向变量
    this.directionLockAction.reset()
	// start方法同样也是做一些初始化或复位工作，包括滑动的距离、滑动方向
    this.scrollBehaviorX.start()
    this.scrollBehaviorY.start()
	// 强制结束上次滚动
    this.animater.doStop()
	// 复位滚动开始的位置
    this.scrollBehaviorX.resetStartPos()
    this.scrollBehaviorY.resetStartPos()
  }
```

这个方法主要是做一系列的复位工作，毕竟是开启一次新的滚动。

```ts
private handleMove(deltaX: number, deltaY: number, e: TouchEvent) {
    // deltaX和deltaY记录的是move事件每次触发时和上一次的差值，getAbsDist方法是用来记录当前和触摸开始的绝对距离
    const absDistX = this.scrollBehaviorX.getAbsDist(deltaX)
    const absDistY = this.scrollBehaviorY.getAbsDist(deltaY)
    const timestamp = getNow()
	// 要么滑动距离大于阈值，要么在上次滑动结束后又立即滑动，否则不认为要进行滚动
    /**/
    	private checkMomentum(absDistX: number, absDistY: number, timestamp: number) {
            return (
              timestamp - this.endTime > this.options.momentumLimitTime &&
              absDistY < this.options.momentumLimitDistance &&
              absDistX < this.options.momentumLimitDistance
            )
          }
    /**/
    if (this.checkMomentum(absDistX, absDistY, timestamp)) {
      return true
    }
    // 这里用来根据eventPassthrough配置项来判断是否要进行锁定，保留原生滚动
    // 如果本次检测到你是进行水平滚动，那么水平方向上会进行锁定，如果你这个配置设置的也是horizontal，这个方法会返回true，就相当于这次不进行模拟滚动而直接使用原生滚动，如果你传的是vertical，就会调用e.preventDefault()来阻止原生滚动
    if (this.directionLockAction.checkMovingDirection(absDistX, absDistY, e)) {
      this.actionsHandler.setInitiated()
      return true
    }
	// 这个方法会把锁定的那个方向的另外一个方向的delta值设为0，即另外那个方向不进行滚动
    const delta = this.directionLockAction.adjustDelta(deltaX, deltaY)
    // move方法做了两件事，1是设置本次滑动的方向值，把右->左、下->上作为正向1，反之作为负向-1；2是调用阻尼方法，这个阻尼是啥意思呢，就是没到边界的话滑动的时候你能感觉到页面是跟你的手指同步滑动的，阻尼之后你就会感觉到有阻力，页面滑动变慢跟不上你的手指了：
    /**/
    	performDampingAlgorithm(delta: number, dampingFactor: number) {
    		// 滑动开始的位置加上本次滑动偏移量即当前滑动到的位置
            let newPos = this.currentPos + delta
            // 已经滑动到了边界
            if (newPos > this.minScrollPos || newPos < this.maxScrollPos) {
              if (
                (newPos > this.minScrollPos && this.options.bounces[0]) ||
                (newPos < this.maxScrollPos && this.options.bounces[1])
              ) {
              	// 阻尼原理很简单，将本次滑动的距离乘一个小于1的小数就可以了
                newPos = this.currentPos + delta * dampingFactor
              } else {
              	// 如果配置关闭了阻尼效果，那么本次滑动就到头了，滑不动了
                newPos =
                  newPos > this.minScrollPos ? this.minScrollPos : this.maxScrollPos
              }
            }
            return newPos
          }
    /**/
    const newX = this.scrollBehaviorX.move(delta.deltaX)
    const newY = this.scrollBehaviorY.move(delta.deltaY)
	// 无论是使用css3 transition还是requestAnimationFrame做动画，实际上改变的都是css的transform属性的值，这里的translate最终调用的是上述this.translater实例的translate方法
    /**/
    	//point:{x:10,y:10}
    	translate(point: TranslaterPoint) {
            let transformStyle = [] as string[]
            Object.keys(point).forEach((key) => {
              if (!translaterMetaData[key]) {
                return
              }
              // translateX/translateY
              const transformFnName = translaterMetaData[key][0]
              if (transformFnName) {
              	// px
                const transformFnArgUnit = translaterMetaData[key][1]
                // x，y的值
                const transformFnArg = point[key]
                transformStyle.push(
                  `${transformFnName}(${transformFnArg}${transformFnArgUnit})`
                )
              }
            })
            this.hooks.trigger(
              this.hooks.eventTypes.beforeTranslate,
              transformStyle,
              point
            )
            // 赋值
            this.style[style.transform as any] = transformStyle.join(' ')
            this.hooks.trigger(this.hooks.eventTypes.translate, point)
          }
    /**/
    // 可以看到直接调用这个方法是没有设置transition的值或是使用requestAnimationFrame来改变位移，所以是没有动画的，到这里content元素就已经会跟着你的触摸进行滚动了
    this.animater.translate({
      x: newX,
      y: newY
    })
	// 这个方法主要是用来重置startTime的值以及根据probeType配置来判断如何派发scroll事件
    /**/
    	private dispatchScroll(timestamp: number) {
            // 每momentumLimitTime时间派发一次事件
            if (timestamp - this.startTime > this.options.momentumLimitTime) {
              // 刷新起始时间和位置，这个用来判断是否要进行momentum动画
              this.startTime = timestamp
              // updateStartPos会将元素当前滚动到的新位置作为起始位置startPos
              this.scrollBehaviorX.updateStartPos()
              this.scrollBehaviorY.updateStartPos()
              if (this.options.probeType === Probe.Throttle) {
                this.hooks.trigger(this.hooks.eventTypes.scroll, this.getCurrentPos())
              }
            }
            // 实时派发事件
            if (this.options.probeType > Probe.Throttle) {
              this.hooks.trigger(this.hooks.eventTypes.scroll, this.getCurrentPos())
            }
          }
    /**/
    this.dispatchScroll(timestamp)
  }
```

到这个函数内容就会跟着我们的触摸开始滚动了，其实这样就可以结束了，但是呢，还有两件事要做，一是一般如果我们滑动一个东西，滑动较快的时候，即使手松开了物体也还会继续滚动一会，不会你一松开它也立马停下来，所以要判断是否是快速滑动以及如何进行这个松开后的动量动画；二是如果开启了回弹动画，这里需要判断是否要回弹。

### 动量动画及回弹动画

先来看触摸结束的处理函数：

```js
private handleEnd(e: TouchEvent) {
    if (this.hooks.trigger(this.hooks.eventTypes.beforeEnd, e)) {
        return
    }
    // 调用scrollBehaviorX和scrollBehaviorY的同名方法来获取当前currentPos的值
    const currentPos = this.getCurrentPos()
    // 更新本次的滚动方向
    this.scrollBehaviorX.updateDirection()
    this.scrollBehaviorY.updateDirection()
    if (this.hooks.trigger(this.hooks.eventTypes.end, e, currentPos)) {
        return true
    }
    // 更新元素位置到结束触摸点的位置
    this.animater.translate(currentPos)
    // 计算最后一次区间耗时
    this.endTime = getNow()
    const duration = this.endTime - this.startTime
    this.hooks.trigger(this.hooks.eventTypes.scrollEnd, currentPos, duration)
}
```

这个函数就派发了几个事件，具体做了什么还要找到订阅了这几个事件的地方，那么就要回到`Scroller.ts`，

`Scroller`类构造函数最后的`init`方法里会执行一系列事件的订阅，找到`end`事件的地方：

```js
actions.hooks.on(
    actions.hooks.eventTypes.end,
    (e: TouchEvent, pos: TranslaterPoint) => {
        this.hooks.trigger(this.hooks.eventTypes.touchEnd, pos)
        if (this.hooks.trigger(this.hooks.eventTypes.end, pos)) {
            return true
        }
        // 判断是否是点击操作
        if (!actions.moved) {
            this.hooks.trigger(this.hooks.eventTypes.scrollCancel)
            if (this.checkClick(e)) {
                return true
            }
        }
        // 这里这里，这个就是用来判断是否越界及进行调整的方法
        if (this.resetPosition(this.options.bounceTime, ease.bounce)) {
            this.animater.setForceStopped(false)
            return true
        }
    }
)
```

看`resetPosition`方法：

```js
resetPosition(time = 0, easing = ease.bounce) {
    // checkInBoundary方法用来返回边界值及是否刚好在边界，具体逻辑看下面
    const {
        position: x,
        inBoundary: xInBoundary,
    } = this.scrollBehaviorX.checkInBoundary()
    const {
        position: y,
        inBoundary: yInBoundary,
    } = this.scrollBehaviorY.checkInBoundary()
    // 如果都刚好在边界那么说明不需要回弹
    if (xInBoundary && yInBoundary) {
        return false
    }
	// 超过边界了那么就滚回去~（诶，你怎么骂人呢），scrollTo方法详见下面
    this.scrollTo(x, y, time, easing)
    return true
}

/*scrollBehavior的相关方法*/
checkInBoundary() {
    const position = this.adjustPosition(this.currentPos)
    // 如果边界值和本次位置一样那么说明刚好在边界
    const inBoundary = position === this.getCurrentPos()
    return {
        position,
        inBoundary,
    }
}

// 越界调整位置
adjustPosition(pos: number) {
    let roundPos = Math.round(pos)
    if (
        !this.hasScroll &&
        !this.hooks.trigger(this.hooks.eventTypes.ignoreHasScroll)
    ) {// 满足条件返回最小滚动距离
        roundPos = this.minScrollPos
    } else if (roundPos > this.minScrollPos) {// 越过最小滚动距离了则需要回弹到最小距离
        roundPos = this.minScrollPos
    } else if (roundPos < this.maxScrollPos) {// 超过最大滚动距离了则需要回弹到最大距离
        roundPos = this.maxScrollPos
    }
    return roundPos
}
/**/
```

上述的最后就是调用`scrollTo`方法进行滚动，那么接下来就来看动画相关的逻辑。

```js
scrollTo(
    x: number,
    y: number,
    time = 0,
    easing = ease.bounce,
    extraTransform = {
        start: {},
        end: {},
    }
) {
    // 根据是使用transition还是requestAnimationFrame来判断是使用css cubic-bezier还是函数
    /*
    bounce: {
        style: 'cubic-bezier(0.165, 0.84, 0.44, 1)',
        fn: function(t: number) {
          return 1 - --t * t * t * t
        }
      }
    */
    const easingFn = this.options.useTransition ? easing.style : easing.fn
    const currentPos = this.getCurrentPos()
	// 动画开始位置
    const startPoint = {
        x: currentPos.x,
        y: currentPos.y,
        ...extraTransform.start,
    }
    // 动画结束位置
    const endPoint = {
        x,
        y,
        ...extraTransform.end,
    }
    this.hooks.trigger(this.hooks.eventTypes.scrollTo, endPoint)
	// 起点终点相同当然就不需要动画了
    if (isSamePoint(startPoint, endPoint)) return
	// 调用动画方法
    this.animater.move(startPoint, endPoint, time, easingFn)
}
```

这个方法的最后终于调用了动画的方法，因为支持两种动画方法，所以我们先来简单思考一下这两种的原理分别是什么。

### 动画

使用css3的transition来做动画是很简单的，只要设置好过渡属性`transition`的值，接下来改变`transform`的值自己就会应用动画，`transition`是个简写属性，包含四个属性，一般来说我们主要设置它的`transition-property`（指定你要应用动画的css属性名称，如transform，不设置则默认应用到所有可以应用的属性）、`transition-duration`（过渡时间，必须要设置，不然为0没有过渡）、`transition-timing-function`（动画曲线）。

使用`requestAnimationFrame`的话就需要自己来设置计算每次的位置了，配合一些常用的动画曲线函数这个也是很简单的，比如上述的函数，更多函数可访问[http://robertpenner.com/easing/](http://robertpenner.com/easing/)：

```js
function(t: number) {
    return 1 - --t * t * t * t
}
```

你只要把动画已经进行了的时长和过渡时间的比例传入，返回的值你再和本次动画的距离相乘，即可得到此刻的位移。

接下来看具体的实现，需要先说明的是这两个类都继承了一个基类，因为它们存在很多的共同操作。

1.css3方式

```js
move(
    startPoint: TranslaterPoint,
    endPoint: TranslaterPoint,
    time: number,
    easingFn: string | EaseFn
) {
    // 设置一个pending变量，用来判断当前是否正在动画中
    this.setPending(time > 0)
    // 设置transition-timing-function属性
    this.transitionTimingFunction(easingFn as string)
    // 设置transition-property的值为transform
    this.transitionProperty()
    // 设置transition-duration属性
    this.transitionTime(time)
    // 调用上述提到过的this.translater的translate方法来设置元素的transform值
    this.translate(endPoint)
	// 如果时间不存在，那么在一个事件周期里里改变属性值不会触发transitionend事件，所以这里通过触发回流强制更新
    if (!time) {
        this._reflow = this.content.offsetHeight
        this.hooks.trigger(this.hooks.eventTypes.move, endPoint)
        this.hooks.trigger(this.hooks.eventTypes.end, endPoint)
    }
}
```

2.`requestAnimationFrame`方式

```js
move(
    startPoint: TranslaterPoint,
    endPoint: TranslaterPoint,
    time: number,
    easingFn: EaseFn | string
) {
    // time为0直接调用translate方法设置位置就可以了
    if (!time) {
        this.translate(endPoint)
        this.hooks.trigger(this.hooks.eventTypes.move, endPoint)
        this.hooks.trigger(this.hooks.eventTypes.end, endPoint)
        return
    }
    // 不为0再进行动画
    this.animate(startPoint, endPoint, time, easingFn as EaseFn)
}

private animate(
    startPoint: TranslaterPoint,
    endPoint: TranslaterPoint,
    duration: number,
    easingFn: EaseFn
) {
    let startTime = getNow()
    const destTime = startTime + duration
    // 动画方法，会被requestAnimationFrame递归调用
    const step = () => {
        let now = getNow()
        // 当前时间大于本次动画结束的时间表示动画结束了
        if (now >= destTime) {
            // 可能距目标值有一点小误差，手动设置一下提高准确度
            this.translate(endPoint)
            this.hooks.trigger(this.hooks.eventTypes.move, endPoint)
            this.hooks.trigger(this.hooks.eventTypes.end, endPoint)
            return
        }
		// 时间耗时比例
        now = (now - startTime) / duration
        // 调用缓动函数
        let easing = easingFn(now)
        const newPoint = {} as TranslaterPoint
        Object.keys(endPoint).forEach((key) => {
            const startValue = startPoint[key]
            const endValue = endPoint[key]
            // 得到本次动画的目标位置
            newPoint[key] = (endValue - startValue) * easing + startValue
        })
        // 执行滚动
        this.translate(newPoint)
        if (this.pending) {
            this.timer = requestAnimationFrame(step)
        }
    }
	// 设置标志位
    this.setPending(true)
    // 基本操作，开始新的定时器或requestAnimationFrame时先做一次清除操作
    cancelAnimationFrame(this.timer)
    // 开始动画
    step()
}
```

上面的代码里都只有设置`pending`为`true`，而没有重置为`false`的地方，聪明的你一定能想到肯定是通过事件订阅在其他地方进行重置了，是的，让我们回到`Scroller.ts`，`Scroller`类里面绑定了`content`元素的`transitionend`事件和订阅了`end`事件：

```js
// 这是transitionend的处理函数
private transitionEnd(e: TouchEvent) {
    if (e.target !== this.content || !this.animater.pending) {
        return
    }
    const animater = this.animater as Transition
    // 删除transition-duration的属性值
    animater.transitionTime()
	// 这里也调用了resetPosition来进行边界回弹，之前是在触摸结束后的end事件调用了，因为直接调用translate方法时是不会触发transitionend事件的，以及触摸结束后可能会有回弹动画，所以这里也需要调用
    if (!this.resetPosition(this.options.bounceTime, ease.bounce)) {
        this.animater.setPending(false)
    }
}
```

```js
this.animater.hooks.on(
    this.animater.hooks.eventTypes.end,
    (pos: TranslaterPoint) => {
        // 同上，边界回弹
        if (!this.resetPosition(this.options.bounceTime)) {
            this.animater.setPending(false)
            this.hooks.trigger(this.hooks.eventTypes.scrollEnd, pos)
        }
    }
)
```

当然，上述边界回弹的函数里最后动画完成后又会触发这两个事件，就又走到了`resetPosition`的判断逻辑，但是因为它们已经回弹完成在边界上了，所以会直接返回false。

回弹逻辑看完了，但是动量动画还是没看到，别急，上面说了一般是当你松开手指的时候才判断是否要进行动量运动，所以回到上面的`handleEnd`方法，发现最后触发了一个`scrollEnd`事件，在`Scroller`里找到订阅该事件的处理函数：

```ts
actions.hooks.on(
    actions.hooks.eventTypes.scrollEnd,
    (pos: TranslaterPoint, duration: number) => {
        // 这个duration=this.endTime - this.startTime，但是startTime在一次触摸中每超过momentumLimitTime都会进行重置的，所以不是从手指触摸到手指离开的总时间
        // 最后这段时间片段滚动的距离
        const deltaX = Math.abs(pos.x - this.scrollBehaviorX.startPos)
        const deltaY = Math.abs(pos.y - this.scrollBehaviorY.startPos)
		// 判断是否是轻拂动作，应该是为插件服务的，这里不管
        /**/
        private checkFlick(duration: number, deltaX: number, deltaY: number) {
            const flickMinMovingDistance = 1 // distinguish flick from click
            if (
                this.hooks.events.flick.length > 1 &&
                duration < this.options.flickLimitTime &&
                deltaX < this.options.flickLimitDistance &&
                deltaY < this.options.flickLimitDistance &&
                (deltaY > flickMinMovingDistance || deltaX > flickMinMovingDistance)
            ) {
                return true
            }
        }
        /**/
        if (this.checkFlick(duration, deltaX, deltaY)) {
            this.animater.setForceStopped(false)
            this.hooks.trigger(this.hooks.eventTypes.flick)
            return
        }
		// 判断是否进行momentum动画
        if (this.momentum(pos, duration)) {
            this.animater.setForceStopped(false)
            return
        }
    }
)
```

```ts
private momentum(pos: TranslaterPoint, duration: number) {
    const meta = {
        time: 0,
        easing: ease.swiper,
        newX: pos.x,
        newY: pos.y,
    }
    // 判断是否满足动量条件，满足则计算动量数据，也就是最后要滚动到的位置，这个方法代码较多，就不放出来了，反正做的事情时根据配置来判断是否满足动量条件，满足再根据配置判断是否在某个方向上允许回弹，最后再动用另一个方法momentum来计算动量数据,这个方法见下面
    const momentumX = this.scrollBehaviorX.end(duration)
    const momentumY = this.scrollBehaviorY.end(duration)
	// 做一下判断
    meta.newX = isUndef(momentumX.destination)
        ? meta.newX
    : (momentumX.destination as number)
    meta.newY = isUndef(momentumY.destination)
        ? meta.newY
    : (momentumY.destination as number)
    meta.time = Math.max(
        momentumX.duration as number,
        momentumY.duration as number
    )
    // 位置变了，那么意味着要进行动量动画
    if (meta.newX !== pos.x || meta.newY !== pos.y) {
        this.scrollTo(meta.newX, meta.newY, meta.time, meta.easing)
        return true
    }
}
```

```ts
// 计算动量数据
private momentum(
    current: number,
    start: number,
    time: number,
    lowerMargin: number,
    upperMargin: number,
    wrapperSize: number,
    options = this.options
) {
    // 最后滑动的时间片段
    const distance = current - start
    // 最后滑动的速度
    const speed = Math.abs(distance) / time
    const { deceleration, swipeBounceTime, swipeTime } = options
    const momentumData = {
        // 目标位置计算方式：手指松开后元素最后的位置+额外距离
        // deceleration代表减速度，默认值是0.0015，假如distance = 15px，time = 300ms，那么speed = 0.05px/ms，则speed / deceleration = 33，即从当前距离继续滑动33px，你速度越快或deceleration设置的越小，滑动的越远
        destination: current + (speed / deceleration) * (distance < 0 ? -1 : 1),
        duration: swipeTime,
        rate: 15,
    }
    // 超过最大滑动距离
    if (momentumData.destination < lowerMargin) {
        // 如果用户配置允许该方向回弹，那么再次计算动量距离，为什么？？否则最多只能滚动到最大距离
        momentumData.destination = wrapperSize
            ? Math.max(
            lowerMargin - wrapperSize / 4,
            lowerMargin - (wrapperSize / momentumData.rate) * speed
        )
        : lowerMargin
        momentumData.duration = swipeBounceTime
    } else if (momentumData.destination > upperMargin) {// 超过最小滚动距离，同上
        momentumData.destination = wrapperSize
            ? Math.min(
            upperMargin + wrapperSize / 4,
            upperMargin + (wrapperSize / momentumData.rate) * speed
        )
        : upperMargin
        momentumData.duration = swipeBounceTime
    }
    momentumData.destination = Math.round(momentumData.destination)
    return momentumData
}
```

动量逻辑其实也很简单，就是根据最后时刻的耗时和距离来进行一下判断，再根据一定算法来计算动量数据也就是最终要滚动到的位置，然后滚过去。

到这里，核心的滚动逻辑已经全部结束了，最后来看一下如何强制结束`transition`滚动，因为`requestAnimationFrame`结束很简单，调用一下`cancelAnimationFrame`就可以了。

```ts
doStop(): boolean {
    const pending = this.pending
    if (pending) {
        // 复位标志位
        this.setPending(false)
        // 获取content元素当前的translateX和translateY的值
        const { x, y } = this.translater.getComputedPosition()
        // 将transition-duration的值设为0
        this.transitionTime()
        // 设置到当前位置
        this.translate({ x, y })
    }
    return pending
}
```

首先获取到元素此刻的位置，然后删除过渡时间，最后再修改目标值为此刻的位置，因为不修改，即使你把过渡时间改回0了过渡动画仍然会继续，此时你强制修改一下位置，它立马就会结束。

## 例行总结

因为是第一次认真的阅读一份源码，所以可能会有很多问题，通篇就像在给这个源码加注释，而且因为是凭空阅读并没有通过运行代码进行断点调试，所以难免会存在错误。

首先说说`TypeScript`，后半部分基本没有再介绍过它，所以可以发现想要阅读一份`TypeScript`代码是并不难的，只要了解一些常用的语法基本就没有障碍了，但是离自己能熟练的使用那还是存在很远的距离，很多东西就是这样，你可以看的懂，但是你自己写就不会了，也没啥捷径，归根结底还是要多用多思考。

然后是`BetterScroll`，代码总体来说还是比较清晰的，因为是插件化，所以事件机制是少不了的，优点是功能解耦，各部分独立，缺点也显而易见，首先是每个类都有自己的事件，很多事件还是同名的，所以很容易看着看着就晕了，其次是因为事件订阅发布，很难清楚的理解事件流，所以这也是比如`vue`更提倡通过属性来显示传递和接收。

总的来说，这个库的核心滚动是一个很简单的功能，自己实现什么都不考虑的话一百多行代码可能也就够了，但是并不妨碍可以将它扩展成一个功能强大的库，这样要考虑的事情就比较多了，首先要考虑到各种边界情况，其次是要考虑兼容性，比如css样式，可能还会遇到特定机型的bug，代码如何组织也很重要，要尽量的复用，比如`BetterScroll`里两种动画方式就存在很多共同操作，那么就可以把这些提取到公共的父类里，又比如水平滚动和垂直滚动肯定也是大量代码都是一样的，所以也需要进行抽象提炼，因为设计成插件化，所以还要考虑插件的开发和集成，最后还需要完善的测试，所以一个优秀的开源项目都是不容易的。
