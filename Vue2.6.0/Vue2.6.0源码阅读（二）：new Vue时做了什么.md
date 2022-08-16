上一篇我们看了引入`Vue`时都有哪些操作，这一篇我们来看一下`new`一个`Vue`实例时会发生什么，测试代码如下：

```html
<div id="app"></div>
```

```js
const app = new Vue({
    el: '#app',
    template: `<h1>{{text}}</h1>`,
    data: {
        text: 'hello'
    }
})
```

根据上一篇的最后我们知道`Vue`的构造函数里只调用了一个`_init`方法，来看看这个函数都做了什么：

```js
Vue.prototype._init = function (options) {
    // vue里把vue实例都叫做vm
    const vm = this
    // 一个uid
    vm._uid = uid++
    // 避免自身被观察的标志
    vm._isVue = true
    // 合并选项
    if (options && options._isComponent) {
      // 这里针对组件的我们暂时不看
      // 优化内部组件实例化，因为动态选项合并非常慢，并且没有任何内部组件选项需要特殊处理。
      initInternalComponent(vm, options)
    } else {
      // 合并选项
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    // 引用自己
    vm._renderProxy = vm
    vm._self = vm
    // ...
}
```

前半部分主要做的是事情就是合并选项，我们具体来看。

# 合并选项

首先调用了`resolveConstructorOptions(vm.constructor)`，`vm.constructor`就是`Vue`构造函数：

```js
export function resolveConstructorOptions (Ctor) {
  // 上一节我们知道Vue构造函数的options上面一共初始化了四个属性：components、directives、filters、_base
  let options = Ctor.options
  // 这里super属性为false，所以会直接跳过，我们后续再深入这里
  if (Ctor.super) {
    // ...
  }
  return options
}
```

接下来看`mergeOptions`方法：

```js
export function mergeOptions (
  parent,
  child,
  vm
){
  // 我们的options是对象，所以这个分支也跳过
  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  // ...
}
```

这里又调用了三个方法，我们一一来看。

1.`normalizeProps`方法主要是用来处理我们传入的`props`选项，因为`props`可以传入数组类型的也可以传入对象类型的，最后都会被格式为对象格式：

```js
function normalizeProps (options, vm) {
    const props = options.props
    if (!props) return
    const res = {}
    let i, val, name
    // 如果是字符串数组形式的话会转成{val:{ type: null }}
    if (Array.isArray(props)) {
        i = props.length
        while (i--) {
            val = props[i]
            if (typeof val === 'string') {
                name = camelize(val)
                res[name] = { type: null }
            }
        }
    } else if (isPlainObject(props)) {
        for (const key in props) {
            val = props[key]
            name = camelize(key)
            // val可以是一个类型，比如String、Number，也可以是一个对象，如{type: Number, default: 0}
            res[name] = isPlainObject(val)
                ? val
            : { type: val }
        }
    }
    options.props = res
}
```

`camelize`方法用来将连字符分隔的字符串转为驼峰式：

```js
const camelizeRE = /-(\w)/g
export const camelize = cached((str) => {
  return str.replace(camelizeRE, (_, c) => c ? c.toUpperCase() : '')
})
```

所以同一个属性名可以使用驼峰式，也可以使用`-`连接式。

另外可以看到`camelize`是使用`cached`方法调用后返回的一个函数，这主要是为了缓存计算结果，如果某个计算之前计算过了，下次会直接返回缓存的结果，就不用再次计算了，`cached`函数如下：

```js
export function cached (fn) {
  // 通过闭包来保存一个缓存对象
  const cache = Object.create(null)
  return (function cachedFn (str) {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  })
}
```

`isPlainObject`方法用来判断一个对象是否是普通的对象字面量：

```js
export function isPlainObject (obj) {
  return _toString.call(obj) === '[object Object]'
}
```

2.`normalizeInject`方法顾名思义就是用来处理`inject`选项的，该选项是和`provide`选项配合使用的，用来向子孙后代组件注入一个依赖，`inject`可以是一个字符串数组或一个对象：

```js
{
    // 数组
    inject: ['foo'],
    // 对象方式1
    inject: {
        foo: { default: 'foo' }
    },
    // 对象方式2
    inject: {
        foo: {
          from: 'bar',// 从一个不同名字的 property 注入
          default: 'foo'
        }
    },
    // 对象方式3
    inject: {
        foo: 'foo'
    }
}
```

该方法会把上述几种类型都规范化为同样的类型：

```js
function normalizeInject (options) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  if (Array.isArray(inject)) {
    // 字符串数组类型，转化为{ from: val }
    for (let i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] }
    }
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val)// 值也是个对象的话
        : { from: val }// 值是个字符串
    }
  }
}
```

3.最后一个`normalizeDirectives`用来处理指令选项，指令选项可以传一个对象，也可以传一个函数：

```js
// 对象格式
directives: {
    focus: {
        inserted: function (el) {
            el.focus()
        },
        // 其他钩子
    }
}
// 函数格式，该函数会在bind 和 update时调用
directives: {
    focus (el, binding) {
        el.style.backgroundColor = binding.value
    }
}
```

函数格式是一种简写，会在`bind`和`update`两个钩子执行，所以会把它也转成钩子对象的形式：

```js
function normalizeDirectives (options) {
    const dirs = options.directives
    if (dirs) {
        for (const key in dirs) {
            const def = dirs[key]
            if (typeof def === 'function') {
                dirs[key] = { bind: def, update: def }
            }
        }
    }
}
```

接下来回到`mergeOptions`方法，继续往下走：

```js
export function mergeOptions (){
  // ...
  
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  // ...
}
```

`_base`属性目前我们只知道`Vue`构造函数的选项上是有的，而且就是`Vue`构造函数自身，我们传的选项显然没有，所以会进入这个分支，`extends`选项是用来继承另一个组件，官方文档上说是为了方便扩展单文件组件的，和`mixins`类似，反正笔者没使用过，如果传了该选项，那么会把它和`parent`选项先进行合并。`mixins`选项我们应该很熟悉，各个组件的公共逻辑可能会通过`mixins`提取出来，因为`mixins`可以包含所有选项，里面再套`mixin`也是可以的，所以通过递归来合并，这些都会在我们组件自身的选项之前合并，所以这就是为什么`mixin`里的生命周期钩子会在组件自身的钩子之前调用。

继续看`mergeOptions`方法：

```js
const options = {}
let key
// 先遍历父选项进行合并
for (key in parent) {
    mergeField(key)
}
// 再遍历子选项里存在父选项里不存在的属性
for (key in child) {
    if (!hasOwn(parent, key)) {
        mergeField(key)
    }
}
// 合并操作
function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
}
return options
```

最后就是遍历所有选项，然后执行合并操作，如果某个选项有特定的合并策略，那么`strats[key]`可以获取到，否则就执行默认的合并策略。

## 默认的合并策略

我们先看默认的合并策略：

```js
const defaultStrat = function (parentVal, childVal) {
    return childVal === undefined
        ? parentVal
    : childVal
}
```

默认的合并策略很简单，子选项的值不存在那么就使用父选项的。

## data/provide选项的合并

```js
strats.data = function (
 parentVal,
 childVal,
 vm
) {
    if (!vm) {
        if (childVal && typeof childVal !== 'function') {
            process.env.NODE_ENV !== 'production' && warn(
                'The "data" option should be a function ' +
                'that returns a per-instance value in component ' +
                'definitions.',
                vm
            )
            return parentVal
        }
        return mergeDataOrFn(parentVal, childVal)
    }
    return mergeDataOrFn(parentVal, childVal, vm)
}
```

如果`vm`不存在，那么就相当于不是通过`new`来调用，可能是定义一个组件，这时候如果传的`data`选项不是一个函数就会警告，并且直接忽略，返回默认值，相信用`vue`的人都很清楚这个规则。然后会调用`mergeDataOrFn`这个方法：

```js
export function mergeDataOrFn (
 parentVal,
 childVal,
 vm
) {
    if (!vm) {
        // Vue.extend合并, 必须都是函数
        if (!childVal) {
            return parentVal
        }
        if (!parentVal) {
            return childVal
        }
        // 当父选项和子选项都提供了
        // 我们需要返回一个函数，该函数返回两个函数的合并结果
        // 这里不需要检查parentVal是否是函数，因为它必须是一个函数才能传递以前的合并。
        return function mergedDataFn () {
            return mergeData(
                typeof childVal === 'function' ? childVal.call(this, this) : childVal,
                typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
            )
        }
    } else {
        // 实例化的时候合并
        return function mergedInstanceDataFn () {
            const instanceData = typeof childVal === 'function'
            ? childVal.call(vm, vm)
            : childVal
            const defaultData = typeof parentVal === 'function'
            ? parentVal.call(vm, vm)
            : parentVal
            if (instanceData) {
                return mergeData(instanceData, defaultData)
            } else {
                return defaultData
            }
        }
    }
}
```

同样也分了两个分支，且最后返回的都是一个函数，也就是说真正的合并时机并不是在这里。`vm`不存在则代表是执行`Vue.extend`合并，也就是定义组件的时候，这种情况下，父选项和子选项都必须是函数，所以当两个选项都存在时，会先执行这两个函数，然后对两个对象再调用`mergeData`方法进行合并。`vm`存在的时候则代表是创建`vue`实例，是函数的话就执行一下，当子选项存在最后也是调用`mergeData`来合并：

```js
function mergeData (to, from) {
    if (!from) return to
    let key, toVal, fromVal

    const keys = hasSymbol
    ? Reflect.ownKeys(from)
    : Object.keys(from)
    // 遍历from对象的keys
    for (let i = 0; i < keys.length; i++) {
        key = keys[i]
        // __ob__是代表该对象被观察过了的标志，不需要合并
        if (key === '__ob__') continue
        toVal = to[key]
        fromVal = from[key]
        // 如果to对象没有该属性，那么直接添加
        if (!hasOwn(to, key)) {
            // set方法会判断目标对象是否是一个响应式对象，是的话添加的新属性也会是响应式的，否则就是单纯添加一个属性
            set(to, key, fromVal)
        } else if (// 否则如果两个的值不一样，且都是对象{}的时候递归进行合并
            toVal !== fromVal &&
            isPlainObject(toVal) &&
            isPlainObject(fromVal)
        ) {
            mergeData(toVal, fromVal)
        }
    }
    return to
}
```

`to`代表是子选项/实例选项，`from`代表是父选项或默认选项，这个方法会遍历父选项/默认选项的所有属性，然后合并到子选项/实例选项上，合并策略也很简单，当某个属性子选项没有那么直接添加，当父子都有，那么判断它们的值是否是一个普通的对象，是的话就递归进行合并，否则就直接使用子选项的值。

`provide`选项同样也可以接收一个对象或一个返回对象的函数，所以它们的合并策略是一样的：

```js
strats.provide = mergeDataOrFn
```

## 生命周期的合并

```js
const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

所有生命周期的合并策略都是一样的，看`mergeHook`：

```js
function mergeHook (
  parentVal,
  childVal
) {
  // childVal不存在，返回parentVal
  // childVal存在，且parentVal也存在，那么两个数组进行合并
  //              且parentVal不存在，把childVal格式化成数组类型
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}
```

合并策略也很简单，会把父子选项/默认选项实例选项的钩子函数都保存到一个数组里，父/默认选项的优先，最后会使用`dedupeHooks`方法做一个去重操作：

```js
function dedupeHooks (hooks) {
  const res = []
  for (let i = 0; i < hooks.length; i++) {
    if (res.indexOf(hooks[i]) === -1) {
      res.push(hooks[i])
    }
  }
  return res
}
```

## 合并资源选项

```js
const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```

这三个选项的合并策略也是一样的：

```js
function mergeAssets (
  parentVal,
  childVal,
  vm,
  key
) {
  const res = Object.create(parentVal || null)
  if (childVal) {
    return extend(res, childVal)
  } else {
    return res
  }
}
```

创建了一个以父选项的值为`__proto__`的对象，如果子选项没有值，直接返回该对象，否则调用`extend`方法来合并属性。

可以看到这里的属性合并有点不一样，不是直接把父子选项的数据合并到同一个对象上，而是把父选项的值挂载到子选项值的原型上。

## 合并watch选项

```js
strats.watch = function (
  parentVal,
  childVal,
  vm,
  key
) {
  // 当父子选项不同时存在
  if (!childVal) return Object.create(parentVal || null)
  if (!parentVal) return childVal
  const ret = {}
  extend(ret, parentVal)
  // 遍历子选项对象的属性
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```

`watch`选项合并也不是覆盖操作，而是会把父子选项的值都保存，比如父子选项都观察了一个`a`变量，那么它的两个回调函数都会合并到一个数组里，父选项的回调优先。

## 其他一些值为对象的选项合并

```js
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal,
  childVal,
  vm,
  key
) {
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```

这几个选项值都是对象的属性合并都是一样的，且很简单，就是创建一个新对象，然后把父子选项值的属性都添加上去，如果有重复，那么子的属性会覆盖父的。

# 初始化操作

回到`_init`方法，合并选项结束后，接下来会初始化一堆东西：

```js
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm)
initState(vm)
initProvide(vm)
callHook(vm, 'created')
```

现在我们大致可以知道，`beforeCreate`生命周期前做的事情有合并选项、初始化生命周期、初始化事件、初始化渲染，`beforeCreate`和`created`两个生命周期之间做的事情有初始化依赖注入、初始化状态，这个状态里面其实包含了很多我们熟悉的`props`、`methods`、`data`、`computed`、`watch`等。接下来一一来看：

## 初始化生命周期

```js
export function initLifecycle (vm) {
  const options = vm.$options

  // 找到第一个非抽象的父级，然后将该实例添加到其$children属性中，这样我们就可以在父级上通过该属性找到它的子组件
  // 抽象组件有transition、keep-alive
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  // 关联父组件
  vm.$parent = parent
  // 当前组件树的根vue实例，如果没有父实例，那么就是自己
  vm.$root = parent ? parent.$root : vm
  // 当前实例的直接子组件
  vm.$children = []
  // dom元素和组件实例
  vm.$refs = {}
  // 下面这些属性在后面用到时再来了解
  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

初始化生命周期方法主要做的事情是关联父子组件，然后定义了一些对外或者对内的 相关变量。

## 初始化事件

```js
export function initEvents (vm) {
  // 创建了一个保存事件的对象
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // 初始化父附加事件
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```

初始化事件先定义了两个变量，然后判断是否存在父组件的附加事件，这个具体是做什么的暂时还不知道，后面再看。

## 初始化渲染

```js
export function initRender (vm) {
  vm._vnode = null // 子树的根节点
  vm._staticTrees = null // v-once 缓存树
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // 父树中的占位符节点
  const renderContext = parentVnode && parentVnode.context
  // 处理slot
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // 给当前实例绑定一个createElement方法
  // 这样我们就可以在其中获得适当的渲染上下文。
  // 参数列表: tag, data, children, normalizationType, alwaysNormalize
  // 内部版本由从模板编译的渲染函数使用
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // 公开版本，用于用户编写渲染函数时使用
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

   // $attrs 和 $listeners 用于更方便的创建高阶组件 HOC
  // 它们需要是响应性的，这样使用它们的HOC才会始终得到更新
  const parentData = parentVnode && parentVnode.data
  defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
  defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
}
```

初始化渲染方法也是先定义了几个变量，然后处理了一下插槽，接着定义了两个变量用于引用创建虚拟`dom`节点的`createElement`方法，最后，给实例添加了两个响应性的属性`$attrs`、`$listeners`，`defineReactive`方法的实现我们放到下一篇去看。

## 触发生命周期

触发声明周期的方法如下：

```js
callHook(vm, 'beforeCreate')
```

让我们看看`callhook`方法的实现：

```js
export function callHook (vm, hook) {
  // #7573 调用生命周期钩子时禁止依赖收集
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  // 此处暂时不知道其用处
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

调用生命周期钩子不允许进行依赖收集，通过`pushTarget`来实现：

```js
Dep.target = null
const targetStack = []

export function pushTarget (target) {
  targetStack.push(target)
  Dep.target = target
}
```

不带参数执行，相当于设置`Dep.target = undefined`，所以即使触发了某个属性的`getter`，也不会收集到任何`watcher`。

接下来遍历生命周期的钩子函数，通过`invokeWithErrorHandling`方法执行：

```js
export function invokeWithErrorHandling (
  handler,
  context,
  args,
  vm,
  info
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)
    // 处理函数返回值是promise的情况
    if (res && !res._isVue && isPromise(res)) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```

其实就是把函数放在`try catch`里执行，这样能捕捉错误，详细的错误捕捉会在后面单独写一篇文章来介绍。

最后执行了`popTarget`：

```js
export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

也就是恢复到上一个`watcher`。

## 初始化依赖注入

`initInjections`是在`initProvide`之前执行的，不过为了好理解，我们先看`initProvide`：

```js
export function initProvide (vm) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```

很简单，给`vue`实例添加了一个`_provided`属性，用来存放`provide`对象。

接下来看`initInjections`：

```js
export function initInjections (vm) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      defineReactive(vm, key, result[key])
    })
    toggleObserving(true)
  }
}
```

先执行了`resolveInject`方法：

```js
export function resolveInject (inject, vm) {
  if (inject) {
    const result = Object.create(null)
    const keys = hasSymbol
      ? Reflect.ownKeys(inject)
      : Object.keys(inject)
    // 遍历inject对象的属性，每个属性代表是注入的一个依赖
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      // #6574 跳过被观察过的注入对象...
      if (key === '__ob__') continue
      const provideKey = inject[key].from
      let source = vm
      // 从当前实例，依次往上查找，直到某个实例的provide提供了对应的inject
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      // 如果一个实例都没有
      if (!source) {
        // 如果存在default属性，那么使用default的属性值
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        }
      }
    }
    return result
  }
}
```

这个方法其实就是求出注入的每个`inject`属性的值，如果有值的话，接下来执行了`toggleObserving(false)`：

```js
export let shouldObserve = true

export function toggleObserving(value) {
  shouldObserve = value
}
```

也就是把`shouldObserve`标志位设为`false`，这个标志位如果为`false`的话那么调用`observe`方法时将不会对传入的对象进行观察，因为接下来紧接着把注入的属性都添加到了`vue`实例上：

```js
Object.keys(result).forEach(key => {
    defineReactive(vm, key, result[key])
})
```

`defineReactive`方法是给一个对象添加一个响应性的属性的，如果该属性的值也是对象或数组的话就会调用`observe`方法来观察该数组，所以就是为了不进行这一步操作，从`vue`的文档中我们也可以看到：

>提示：`provide` 和 `inject` 绑定并不是可响应的。这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的 property 还是可响应的。

## 初始化状态

`initState`里面就包含着`vue`的核心，数据观察：

```js
export function initState(vm) {
  vm._watchers = []// 声明了一个存放watcher的数组
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */ )
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

很清晰，分别处理了`props`、`methods`、`data`、`computed`、`watch`，接下来一一来看：

### 初始化props

```js
function initProps(vm, propsOptions) {
  // propsData：创建实例时传递 props。主要作用是方便测试，只用于 new 创建的实例中
  const propsData = vm.$options.propsData || {}
  // 存储props
  const props = vm._props = {}
  // 缓存prop的key，这样在后面更新prop时可以通过遍历数组，而不是枚举动态对象的属性
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // 根实例的props需要被观察
  if (!isRoot) {
    toggleObserving(false)
  }
  // 遍历所有prop
  for (const key in propsOptions) {
    keys.push(key)
    // 校验props，返回其默认值
    const value = validateProp(key, propsOptions, propsData, vm)
    defineReactive(props, key, value)
    // 在Vue.extend（）期间，静态prop已在组件的原型上代理。我们只需要在这里代理实例化时定义的prop
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

先定义了几个内部变量，然后对于非根实例，也是关闭了`observe`观察的标志位，然后遍历`props`，`validateProp`方法用来确定该`prop`的默认值，然后将该`prop`响应性的添加到`vm._props`对象上，因为`shouldObserve`设为`false`了，所以不会对`value`进行观察，最后，对于新增的`prop`，会将其访问代理到`_props`上，也就是当我们访问`this.xxx`时，实际访问的是`this._props.xxx`：

```js
export function proxy(target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

### 初始化方法

```js
function initMethods(vm, methods) {
  for (const key in methods) {
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
  }
}
```

`initMethods`方法很简单，先判断是否是函数，是的话先绑定一下它的`this`，然后把该方法添加到实例上即可。我们知道在`methods`的函数里访问`this`都是指向当前`vue`实例的，就是通过这个`bind`方法来绑定的，接下来看一下它的实现：

```js
export const bind = Function.prototype.bind
  ? nativeBind
  : polyfillBind
```

如果当前浏览器支持原生的`bind`方法，那么直接使用原生方法：

```js
function nativeBind (fn, ctx) {
  return fn.bind(ctx)
}
```

如果不支持，就只能`polyfill`一个：

```js
function polyfillBind (fn, ctx) {
  function boundFn (a) {
    const l = arguments.length
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }

  boundFn._length = fn.length
  return boundFn
}
```

可以看到就是一个最简单的实现方式，参数大于一个时，使用`apply`方法调用，否则使用`call`方法调用。这个方法其实没有太大的存在必要，因为几乎所有浏览器都原生支持`bind`方法，这里只是为了能兼容之前的版本，以让它支持在`PhantomJS 1.x`的环境里运行。

### 初始化data

这部分会在下一篇文章里详细介绍。

### 初始化计算属性

```js
function initComputed(vm, computed) {
  // 保存用于计算属性的watcher
  const watchers = vm._computedWatchers = Object.create(null)

  for (const key in computed) {
    // 计算属性支持两种写法：普通函数、对象形式：{ get: Function, set: Function }
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get

    // 创建一个内部的 watcher 
    watchers[key] = new Watcher(
      vm,
      getter || noop,
      noop,
      {
        lazy: true
      }
    )

    // 组件定义的计算属性已在组件原型上定义。我们只需要在这里定义实例化时定义的计算属性。
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
}
```

可以看到为每个计算属性都创建了一个`watcher`实例，`watcher`的作用我们后续再说，然后调用了`defineComputed`方法，这块涉及到计算属性的缓存原理，我们后续作为一篇单独的文章进行介绍。

### 初始化watch

```js
function initWatch(vm, watch) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

遍历`watch`选项对象的`key`，依次调用`createWatcher`方法：

```js
function createWatcher(
  vm,
  expOrFn,
  handler,
  options
) {
  // {handler: '', deep...}形式
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  // 回调是字符串的话，那么一般指methods里的方法
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

适配了两种非函数的形式，最后调用了`$watch`方法，这个方法是`vue`原型上的一个方法，用于观察 `vue` 实例上的一个表达式或者一个函数计算结果的变化：

```js
Vue.prototype.$watch = function (
 expOrFn,
 cb,
 options
) {
    const vm = this
    if (isPlainObject(cb)) {
        // createWatcher方法会从对象里解析出cb，options，然后又会重新调用$watch方法
        return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // 指定了immediate为true，那么立刻执行一次回调
    if (options.immediate) {
        try {
            cb.call(vm, watcher.value)
        } catch (error) {
            handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
        }
    }
    // 返回一个用于解除观察的方法
    return function unwatchFn() {
        watcher.teardown()
    }
}
```

这个方法逻辑也很简单，主要就是实例化了一个`Watcher`实例，`Watcher`的相关内容后续再说，然后判断是否要立即执行一次回调，最后返回一个解除观察的方法。

## 挂载

最后，如果提供了`el`选项，那么会进行挂载：

```js
if (vm.$options.el) {
    vm.$mount(vm.$options.el)
}
```

挂载的详细内容也会在后面单独讨论。

`new Vue`的过程到这里就结束了，注意，本文考虑的只是最最最基础的使用方式的实例化过程，当你使用的`vue`功能越多，实例化要做的事情也会更多，这些我们都后面再看，毕竟，路漫漫其修远兮。

