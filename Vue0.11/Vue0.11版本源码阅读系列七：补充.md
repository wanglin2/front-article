---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

第一篇留了两个问题：

1.计算属性依赖的属性变化了是如何触发计算属性更新的

2.`watch`选项或`$watch`方法的原理是怎样的

本篇来分析一下这两个问题，另外简单看一下自定义元素是怎么渲染的。



# 计算属性

```html
<p v-text="showMessage + '我是不重要的字符串'"></p>
```

```js
{
    data: {
        message: 'Hello Vue.js!'
    },
    computed: {
        showMessage() { 
            return this.message.toUpperCase()
        }
    }
}
```

以这个简单的例子来说，首先计算属性也是会挂载到`vue`实例上成为实例的一个属性：

```js
for (var key in computed) {
    var userDef = computed[key]
    var def = {
        enumerable: true,
        configurable: true
    }
    if (typeof userDef === 'function') {
        def.get = _.bind(userDef, this)
        def.set = noop
    } else {
        def.get = userDef.get
            ? _.bind(userDef.get, this)
        : noop
        def.set = userDef.set
            ? _.bind(userDef.set, this)
        : noop
    }
    Object.defineProperty(this, key, def)
}
```

通过`this.xxx`访问计算属性时会调用我们定义的`computed`选项里面的函数。

其次在模板编译指令解析的阶段计算属性和普通属性并没有区别，这个`v-text`指令会创建一个`Directive`实例，这个`Directive`实例初始化时会以`showMessage + '我是不重要的字符串'`为唯一的标志创建一个`Watcher`实例，`v-text`指令的`update`方法会被这个`Watcher`实例所收集，添加到它的`cbs`数组里，`Watcher`实例化时会把自身赋值给`Observer.target`，随后对`showMessage + '我是不重要的字符串'`这个表达式求值，也就会调用到计算属性的函数`showMessage()`，这个函数调用后会引用所依赖的所有属性，这里也就是`message`，这会触发`message`的`getter`，这样这个`Watcher`实例就被添加到`message`的依赖收集对象`dep`里了，后续当`message`的值变化触发其`setter`后会遍历其`dep`里收集的`Watcher`实例，触发`Watcher`的`update`方法，最后会遍历`cbs`里添加的指令的`update`方法，这样这个依赖计算属性的指令就得到了更新。

值得注意的是在这个版本里，计算属性是没有缓存的，即使所依赖的值没有变化，重复引用计算属性的值也会重新执行我们定义的计算属性函数。



# 侦听器

`watch`选项声明的侦听器最后调用的也是`$watch`方法，在第一篇已经知道了`$watch`方法里主要就是创建了一个`Watcher`实例：

```js
// exp就是我们要侦听的数据，如：a、a.b
exports.$watch = function (exp, cb, deep, immediate) {
  var vm = this
  var key = deep ? exp + '**deep**' : exp
  var watcher = vm._userWatchers[key]
  var wrappedCb = function (val, oldVal) {
    cb.call(vm, val, oldVal)
  }
  if (!watcher) {
    watcher = vm._userWatchers[key] =
      new Watcher(vm, exp, wrappedCb, {
        deep: deep,
        user: true
      })
  } else {
    watcher.addCb(wrappedCb)
  }
}
```

对于`Watcher`我们现在已经很熟悉了，实例化的时候会把自己赋值给`Observer.target`，然后触发表达式的求值，也就是我们要侦听的属性，触发其`gettter`然后把该`Watcher`收集到它的依赖收集对象`dep`里，只要被收集就好办了，后续属性值变化后就会触发这个`Watcher`的更新，也就会触发上面的回调。



# 自定义组件的渲染

```html
<my-component></my-component>
```

```js
new Vue({
    el: '#app',
    components: {
        'my-component': {
            template: '<div>{{msg}}</div>',
            data() {
                return {
                    msg: 'hello world！'
                }
            }
        }
    }
})
```

在第一篇里我们提到了每个组件选项最后都会被创建成一个继承了`vue`的构造函数：

![image-20210114201622204](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/860f9c467c4841dd9e85a1489b9c304f~tplv-k3u1fbpfcp-zoom-1.image)

然后到模板编译阶段遍历到这个自定义元素会给它添加一个`v-component`属性：

```js
tag = el.tagName.toLowerCase()
component =
    tag.indexOf('-') > 0 &&
    options.components[tag]
if (component) {
    el.setAttribute(config.prefix + 'component', tag)
}
```

![image-20210115100403284](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a259d7d41f9406cb51c92d2bd81a50c~tplv-k3u1fbpfcp-zoom-1.image)

所以后续也是通过指令来处理这个自定义组件，接下来会生成链接函数，`component`属于`terminal`指令的一种：

![image-20210115100542982](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0b00b04225347a993feffbb0007d3cb~tplv-k3u1fbpfcp-zoom-1.image)

接下来就回到了正常的指令编译过程了，`_bindDir`方法会给`v-component`指令创建一个`Directive`实例，然后会调用`component`指令的`bind`方法：

```js
{
    bind: function () {
        // el就是我们的自定义元素my-component
        if (!this.el.__vue__) {
            // 创建一个注释元素替换掉该自定义元素
            this.ref = document.createComment('v-component')
            _.replace(this.el, this.ref)
            // 检查是否存在keep-alive选项
            this.keepAlive = this._checkParam('keep-alive') != null
            // 检查是否存在ref来引用该组件
            this.refID = _.attr(this.el, 'ref')
            if (this.keepAlive) {
                this.cache = {}
            }
            // 解析构造函数，也就是返回初始化时选项合并阶段生成的构造函数，expression这里是指令值my-component
            this.resolveCtor(this.expression)
            // 创建子实例
            var child = this.build()
            // 插入该子实例
            child.$before(this.ref)
            // 设置ref
            this.setCurrent(child)
        }
    }
} 
```

看`build`方法：

```js
{
    build: function () {
        // 如果有缓存直接返回
        if (this.keepAlive) {
            var cached = this.cache[this.ctorId]
            if (cached) {
                return cached
            }
        }
        var vm = this.vm
        if (this.Ctor) {
            var child = vm.$addChild({
                el: this.el,
                _asComponent: true,
                _host: this._host
            }, this.Ctor)// Ctor就是该组件的构造函数
            if (this.keepAlive) {
                this.cache[this.ctorId] = child
            }
            return child
        }
    }
}
```

这个方法用来创建子实例，调用了`$addChild`方法，简化后如下：

```js
exports.$addChild = function (opts, BaseCtor) {
    var parent = this
    // 父实例就是上述我们new Vue的实例
    opts._parent = parent
   	// 根组件也就是父实例的根组件
    opts._root = parent.$root
    // 创建一个该自定义组件的实例
    var child = new BaseCtor(opts)
    return child
}
```

上面两个方法主要就是创建了一个该组件构造函数的实例，因为组件构造函数继承了`vue`，所以之前的`new Vue`时做的初始化工作同样也都会走一遍，什么观察数据、遍历该自定义组件及其所有子元素进行模板编译绑定指令等等，因为我们传递了`template`选项，所以在第一篇里一带而过的方法`_compile`里在调用`compile`方法之前会先对这个进行处理：

```js
// 这里会把template模板字符串转成dom，原理很简单，创建一个文档片段，再创建一个div，之后再把模板字符串设为div的innserHTML，最后再把div里的元素都添加到文档片段里即可
el = transclude(el, options)
// 编译并链接其余的
compile(el, options)(this, el)
```

最后如果存在`keep-alive`则把该实例缓存一下，回到`bind`方法里的`child.$before(this.ref)`：

```js
exports.$before = function (target, cb, withTransition) {
    return insert(
        this, target, cb, withTransition,
        before, transition.before
    )
}
```

```js
function insert (vm, target, cb, withTransition, op1, op2) {
    // 获取目标元素，这里就是bind方法里创建的注释元素
    target = query(target)
    // 元素当前不在文档中
    var targetIsDetached = !_.inDoc(target)
    // 判断是否要使用过渡方式插入，如果元素不在文档中则会使用带过渡的方式插入
    var op = withTransition === false || targetIsDetached
    ? op1
    : op2
    // 如果目标元素当前已经插入文档以及该该组件没有挂载过就需要触发attached生命周期
    var shouldCallHook =
        !targetIsDetached &&
        !vm._isAttached &&
        !_.inDoc(vm.$el)
    // 插入文档
    op(vm.$el, target, vm, cb)
    if (shouldCallHook) {
        vm._callHook('attached')
    }
    return vm
}
```

`op`方法会调用`transition.before`方法把元素插入到文档中，关于过渡插入的详细分析请参考[vue0.11版本源码阅读系列六：过渡原理](https://juejin.cn/post/6918316620561514510)。

到这里组件就已经渲染完成了，`bind`方法里最后调用了`setCurrent`：

```js
{
    setCurrent: function (child) {
        this.childVM = child
        var refID = child._refID || this.refID
        if (refID) {
            this.vm.$[refID] = child
        }
    }
}
```

如果我们设置了引用比如：`<my-component v-ref="myComponent"></my-component>`，那么就可以通过`this.$.myComponent`访问到该子组件。

`keep-alive`的工作原理也很简单，就是返回之前的实例而不是创建新实例，这样所有的状态都还保存着。



# 总结

本系列到这里基本就结束了，我相信能看到这里的人不多，因为第一次写这种源码阅读的系列，总的来说有点乱，很多地方重点不是很突出，描述的可能也不是很详细，可能不是很让人看的下去，另外难免也会有错误，欢迎大家指出。

阅读源码是每个开发者都无法绕过去的必经之路，无论是为了提升自己还是为了面试，我们终归是要对自己每时每刻在用的东西有个更深的了解，这样对于使用来说也是有好处的，另外思考和学习别人优秀的编码思维，也能让自己变的更好。

不得不说阅读源码是挺枯燥和无聊的，也是有难度的，很容易让人心生退意，很多地方你不是非常的了解其作用的话是基本看不懂的，当然我们也不必执着于这些地方，也不用把所有地方都看完看懂，更好的方式还是带着问题去阅读，比如说我想搞懂某一个地方原理，那么你就去看这部分的代码就可以了，当你沉浸在里面也是别有一番意思的。

话不多说，白白~

