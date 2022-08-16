上一节我们大致看了一下实例化`Vue`时所做的事情，其中初始化`data`选项的部分我们跳过了，这一篇我们详细来了解一下。

# 初始化data选项

先看一下初始化`data`的方法：

```js
function initData(vm) {
  let data = vm.$options.data
  // 获取data对象
  data = vm._data = typeof data === 'function' ?
    getData(data, vm) :
    data || {}
  // 检查是否是普通对象，不是则重置为空对象
  if (!isPlainObject(data)) {
    data = {}
  }
  // 代理data到实例上
  const keys = Object.keys(data)
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // 观察data
  observe(data, true /* asRootData */ )
}
```

从上一篇的`data`选项合并部分我们知道合并后最终返回的是一个函数`mergedInstanceDataFn`，真正的合并是在这个函数内，所以这里会调用`getData`方法：

```js
export function getData(data, vm) {
  // #7573 当执行data的getter时禁止依赖收集，pushTarget上一篇已经介绍过，本质就是把Dep.target设为undefined
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```

执行`data`函数，最后会返回我们上面的`data`对象。接下来遍历`data`对象的属性，使用`proxy`方法将`_data`的属性访问代理到实例`vm`上，这样我们就可以直接通过`this.xxx`来访问到`data`对象的数据了。

最后执行了`observe`函数，这个方法就是用来开启数据观察的方法。

# 数据观察

```js
export function observe(value, asRootData) {
  // 如果不是对象或者是虚拟dom对象则返回
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob
  // 存在__ob__属性则代表该对象之前已经观察过了
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&// 当前允许进行观察
    (Array.isArray(value) || isPlainObject(value)) &&// 只允许对数组和简单对象进行观察
    Object.isExtensible(value) &&// 并且该对象是可开展的，即可以给它添加新属性
    !value._isVue// 最后它不能是Vue实例
  ) {
    ob = new Observer(value)// 创建一个观察者实例
  }
  // 统计有多少个Vue实例对象将该对象作为根数据
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

`observe`方法会判断一个对象是否已经被观察过了，如果没有的话，那么当该数据是数组或简单对象的话会给它创建一个`Observer`实例，接下来看`Observer`类：

```js
export class Observer {
  value;
  dep;
  vmCount; // 使用该对象作为根数据的vm数量

  constructor(value) {
    // 目标对象
    this.value = value
    // 实例化一个依赖收集对象
    this.dep = new Dep()
    // vm数量
    this.vmCount = 0
    // 给目标对象添加一个被观察过了的标志
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // 如果浏览器支持使用__proto__属性
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
}
```

`Observer`构造函数定义了几个变量，然后给目标对象添加了一个被观察的标志位，使用了`def`方法，这个方法也是源码中很常见的一个方法，来看看：

```js
export function def (obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,// 是否可枚举
    writable: true,// 可写
    configurable: true// 可配置、可删除
  })
}
```

其实就是使用`Object.defineProperty`方法来给对象定义属性，同时可以配置属性描述符。

`Observer`构造函数中的`dep`实例是用来收集依赖的，后面再看，接下来区分了数组和对象两种类型，我们一一来看。

## 观察数组

1.如果浏览器支持使用`__proto__`属性时，会调用`protoAugment`方法：

```js
function protoAugment(target, src) {
  target.__proto__ = src
}

protoAugment(value, arrayMethods)
```

一个对象的`__proto__`属性会指向其构造函数的`prototype`对象，所以相当于把数组对象的原型对象由`Array.prototype`修改为`arrayMethods`。

`arrayMethods`顾名思义就是数组的方法：

```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
```

创建了一个以`Array.prototype`为`__proto__`属性对象的对象，大致相当于`new Array()`生成的对象。

```js
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

methodsToPatch.forEach(function (method) {
  // 缓存原始方法
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    // 执行数组原始方法
    const result = original.apply(this, args)
    // 该数组对象的观察者实例
    const ob = this.__ob__
    // 获取新插入的数据
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 遍历新插入的数据进行观察
    if (inserted) ob.observeArray(inserted)
    // 触发更新
    ob.dep.notify()
    return result
  })
})
```

总结来说，`arrayMethods`对象包含了数组的所有方法，且这些方法都是重写后的数组方法，方法内部会先执行数组原始方法，然后如果当前数组的操作会插入新数据的话那么会对新插入的数据也进行观察，最后，如果有该数组的依赖的话，会通知这些依赖更新。

2.如果不支持`__proto__`的话那么会调用`copyAugment`方法：

```js
function copyAugment(target, src, keys ) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}

copyAugment(value, arrayMethods, arrayKeys)
```

这个方法其实相当于使用`Object.defineProperty`给数组对象本身添加方法，我们都知道访问一个对象的属性或方法，如果对象本身存在，那么就不会去原型对象上查找，所以相当于把数组原始方法都给屏蔽了。

至于为什么要重写数组的方法，当然是为了能监听到数组的变化了，重写完数组的方法后，接下来会调用`observeArray`方法：

```js
this.observeArray(value)

observeArray(items ) {
    for (let i = 0, l = items.length; i < l; i++) {
        observe(items[i])
    }
}
```

很简单，遍历数组，依次观察数组的每一项，`observe`方法我们前面介绍过了，它只会对数组和普通对象创建观察者对象，所以如果某一项是数组，那么又会再调用`observeArray`方法，其实就是递归的进行遍历观察。

## 观察对象

对象的话就直接调用`walk`方法：

```js
this.walk(value)

walk(obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
        defineReactive(obj, keys[i])
    }
}
```

遍历对象自身可枚举的属性，然后依次调用`defineReactive`方法，这个方法前面也看到过，现在来看一下它的实现：

```js
export function defineReactive(
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  // 实例化一个dep，用来收集依赖，通过闭包保存
  const dep = new Dep()
  // 获取该属性原来的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // 如果该属性是不可配置的，那么直接返回
  if (property && property.configurable === false) {
    return
  }
  // 保存属性原有的set和get
  const getter = property && property.get
  const setter = property && property.set
  // 如果没有传递val，且getter不存在或setter存在，那么val取当前对象上的值
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  // 如果该属性的值又是一个对象或数组，那么也需要递归进行观察
  let childOb = !shallow && observe(val)
  // 定义get和set
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      // 依赖收集
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val
      // 值没有变化则直接返回
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      // #7981: 对于不带setter的访问器属性
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 观察新的值
      childOb = !shallow && observe(newVal)
      // 触发更新
      dep.notify()
    }
  })
}
```

这个方法比较长，但做的事情比较简单，首先通过闭包保存一个依赖收集实例`dep`，然后重新定义对象的该属性，转换成`get`和`set`的形式，如果该属性原本就存在`getter`和`setter`存取描述符，那么仍然会使用原来的方法，否则通过闭包来使用`val`变量进行该属性值的维护。

在`get`函数里，也就是读取该属性时，会进行依赖收集，在`set`函数里，也就是设置该属性时会触发依赖更新，这部分的内容我们后面再会。

总结一下数据观察的操作，首先从根`data`对象开始，深度优先进行遍历，如果是普通数组和对象的话，会给其创建一个关联的`Observer`实例，同时会给它创建一个依赖收集实例`dep`，紧接着如果它是数组，那么会拦截它的数组方法，然后遍历数组项依次进行观察，也就是重复前面的逻辑，如果是对象，那么会把它自身的所有可枚举属性都转换成`getter`和`setter`的形式，当然也会对它的值进行观察，同样是重复前面的逻辑。这些操作结束后，就相当于把整个`data`都转换成一个响应式对象了，当我们操作这个对象时，无论是给数组添加元素还是给对象设置新值`Vue`都能监听到，监听到用来干什么，后面再细说。
