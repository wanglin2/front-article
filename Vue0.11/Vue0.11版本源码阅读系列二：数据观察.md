---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

上篇介绍了创建`vue`实例时大概做了一些什么事情，其中有一项是初始化数据，本篇来看一下数据观察具体是怎么做的。

`_initData`就是数据观察的起点了：

```js
exports._initData = function () {
  // 代理data到实例
  var data = this._data
  var keys = Object.keys(data)
  var i = keys.length
  var key
  while (i--) {
    key = keys[i]
    if (!_.isReserved(key)) {
      this._proxy(key)
    }
  }
  // 观察data数据
  Observer.create(data).addVm(this)
}
```

`_proxy`方法上一篇已经说过了，就是把`data`数据代理到`vue`实例上，可以通过`this.xx`访问到`this.data.xx`的数据，关键是`Observer`。

`create`是`Observer`类的静态方法，用来给一个数组或对象创建观察对象：

```js
Observer.create = function (value) {
  if (
    value &&
    value.hasOwnProperty('__ob__') &&
    value.__ob__ instanceof Observer
  ) {
    return value.__ob__
  } else if (_.isArray(value)) {
    return new Observer(value, ARRAY)
  } else if (
    _.isPlainObject(value) &&
    !value._isVue
  ) {
    return new Observer(value, OBJECT)
  }
}
```

从这里可以知道`vue`只会对数组和纯粹的对象进行观察，其他比如函数什么的是不会观察的，其主要逻辑是判断该属性是否已经观察过了，是的话就返回观察者对象，否则分别对数组和对象使用不同的标志来实例化观察对象。

来看`Observer`类：

```js
function Observer (value, type) {
  this.id = ++uid
  this.value = value
  this.deps = []
  // 将该观察实例设置到该对象或数组的一个属性，方便后面检查和使用
  _.define(value, '__ob__', this)
  if (type === ARRAY) {// 数组分支
    var augment = _.hasProto
      ? protoAugment
      : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value)
  } else if (type === OBJECT) {// 对象分支
    this.walk(value)
  }
}
```

初始化了一些属性，先看一下比较简单的对象分支：

```js
p.walk = function (obj) {
  var keys = Object.keys(obj)
  var i = keys.length
  var key, prefix
  while (i--) {
    key = keys[i]
    prefix = key.charCodeAt(0)
    if (prefix !== 0x24 && prefix !== 0x5F) { // 跳过 $ or _开头的私有属性
      this.convert(key, obj[key])
    }
  }
}
```

`walk`方法对对象的每个子属性遍历调用`convert`方法：

```js
p.convert = function (key, val) {
  var ob = this
  // 如果该属性的值也是个数组或对象，那么也需要进行观察，observe方法最终调用的也是Object.create方法
  var childOb = ob.observe(val)
  // 每个属性都会创建一个依赖收集实例，利用闭包来保存
  var dep = new Dep()
  // 该属性的观察实例添加到属性值的观察对象里
  if (childOb) {
    childOb.deps.push(dep)
  }
  Object.defineProperty(ob.value, key, {
    enumerable: true,
    configurable: true,
    get: function () {
      // 这里进行收集依赖，Observer.target是一个全局属性，是一个watcher实例，后续再细说，当引用该属性前把watcher实例赋值给这个全局属性，此处就能引用到，然后收集到该属性的dep实例列表里
      if (Observer.target) {
        Observer.target.addDep(dep)
      }
      return val
    },
    set: function (newVal) {
      if (newVal === val) return
      // 如果旧的值是对象或数组那么肯定也有对应的观察实例，所以需要从对应的观察实例里移除该属性的dep
      var oldChildOb = val && val.__ob__
      if (oldChildOb) {
        var oldDeps = oldChildOb.deps
        oldDeps.splice(oldDeps.indexOf(dep), 1)
      }
      val = newVal
      // 检查新值，新赋的值是对象或数组又需要进行递归创建观察实例
      var newChildOb = ob.observe(newVal)
      if (newChildOb) {
        newChildOb.deps.push(dep)
      }
      // 通知该属性的依赖进行更新
      dep.notify()
    }
  })
}
```

接下来看一下数组的分支：

```js
if (type === ARRAY) {
    var augment = _.hasProto
    ? protoAugment
    : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value)
}
```

`vue`修改了数组原型上的一些方法，比如：`push`、`shift`等等，原因是使用这些方法操作数组不会触发该属性的`setter`，所以`vue`就无法检测到变化进行更新，所以需要拦截这些方法进行修改。

这里使用了两种方法，如果浏览器支持`__proto__`，直接通过修改数组的`__proto__`来设置新的原型对象，如果不支持，则使用`Object.defineProperty`来覆盖添加修改后的数组方法。

```js
var arrayProto = Array.prototype
// 创建一个以数组原型对象为原型的新对象
var arrayMethods = Object.create(arrayProto)
;[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
.forEach(function (method) {
  // 缓存数组的原始方法
  var original = arrayProto[method]
  _.define(arrayMethods, method, function mutator () {
    //  这里将arguments拷贝了一份，避免将该对象直接传递给其他函数使用，可能对性能不利
    var i = arguments.length
    var args = new Array(i)
    while (i--) {
      args[i] = arguments[i]
    }
    // 调用原始方法
    var result = original.apply(this, args)
    // 获取该数组的观察实例
    var ob = this.__ob__
    // 获取新插入数组的值
    var inserted
    switch (method) {
      case 'push':
        inserted = args
        break
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 如果有新插入的值，那么对它递归进行观察
    if (inserted) ob.observeArray(inserted)
    // 通知依赖更新
    ob.notify()
    return result
  })
})
```

逻辑很简单，就是当调用了这些方法更新数组后观察新插入的数据，以及通知更新，这里是调用观察对象`ob`的更新方法`notify`：

```js
p.notify = function () {
  var deps = this.deps
  for (var i = 0, l = deps.length; i < l; i++) {
    deps[i].notify()
  }
}
```

通过上面的`convert`方法我们知道这个`deps`数组里收集的是该属性值对应的属性的依赖收集实例`dep`，有点绕：

```js
{
    data: {
        a: [1, 2, 3],
        b: {
            c: [4, 5, 6]
        }
    }
}
```

比如这个例子，忽略`b`的话，一共存在两个`Observer`实例，一个是属性`data`的值的，另一个是 `[1, 2, 3]`的， `[1, 2, 3]`的`Observer`实例的`deps`数组收集了`a`的`dep`，我们使用上述数组的方法更新了这个数组，会通知`a`的`dep`进行更新通知，这很容易理解，如果我们给`a`设置了新值，比如：`data.a = 2`是会触发`a`的`setter`的，里面会调用`a`的`dep`的`notify`方法，只是现在这个`a`的值变成了数组，数组变化了就相当于`a`变化了，但问题是数组变化并不会触发`a`的`setter`，所以就只能手动去调用`a`的`dep`的更新方法去通知`a`的依赖也去更新，但是，比如`c`的数组变化了，会通知`c`的依赖更新，但是不会向上再去通知`b`的依赖更新。

数组的原型方法修改完后就需要去遍历该数组的元素进行观察：

```js
p.observeArray = function (items) {
  var i = items.length
  while (i--) {
    this.observe(items[i])
  }
}
```

很简单，遍历数组调用`observe`方法。

到这里，就完成了对`data`上所有数据的观察了，总结一下，从`data`对象开始，给该对象创建一个观察实例，然后遍历它的子属性，值是数组或对象的话又创建对应的观察实例，然后再继续遍历它们的子属性，继续递归，直到把每个属性都转换成`getter`和`setter`。

在第一次渲染的时候会引用用到的值，也就是会触发对应属性的`getter`，引用前会把对应的`watcher`赋值到`Observer.target`属性，`JavaScript`代码执行是单线程的，所以同一时刻只会有一个`Observer.target`，所以只要某个属性的`getter`里获取到了此刻的`Observer.target`，那一定代表该`watcher`是依赖该属性的，那么就添加到该属性的依赖收集对象`dep`里，这里巧妙的使用闭包来保存每个属性的`dep`实例，后续如果该属性值变化了，那么会触发`setter`，如果新赋值是对象或数组又会递归进行观察，最后再通知该属性的所有依赖进行更新。

上面一直都提到了这个`dep`，现在来看一下：

```js
function Dep () {
  this.id = ++uid
  this.subs = []
}
var p = Dep.prototype
p.addSub = function (sub) {
  this.subs.push(sub)
}
p.removeSub = function (sub) {
  if (this.subs.length) {
    var i = this.subs.indexOf(sub)
    if (i > -1) this.subs.splice(i, 1)
  }
}
p.notify = function () {
  var subs = _.toArray(this.subs)
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

这个类很简单，这就是全部代码，功能是收集订阅者、删除订阅者以及遍历调用订阅者的`update`方法。

最后看一下修改数组和对象的辅助方法，如：`$set`、`$remove`等。

对于数组，直接使用索引设置数组项`vue`是不能检测到的，所以提供了`$set`方法：

```js
_.define(
  arrayProto,
  '$set',
  function $set (index, val) {
    if (index >= this.length) {
      this.length = index + 1
    }
    return this.splice(index, 1, val)[0]
  }
)
```

给数组的原型上添加了`$set`方法，调用`splice`方法来设置值，这个方法由于已经被重写过了，所以可以触发更新，我们完全可以直接使用`splice`方法。

对于对象，在`data`初始化后在添加新属性也是不能检测到的，在`0.11`版本提供各了`$add`方法：

```js
_.define(
  objProto,
  '$add',
  function $add (key, val) {
    if (this.hasOwnProperty(key)) return
    var ob = this.__ob__
    if (!ob || _.isReserved(key)) {
      this[key] = val
      return
    }
    ob.convert(key, val)
    if (ob.vms) {
      var i = ob.vms.length
      while (i--) {
        var vm = ob.vms[i]
        vm._proxy(key)
        vm._digest()
      }
    } else {
      ob.notify()
    }
  }
)
```

直接调用`convert`方法就可以了，设置完得通知更新，这里分了两种情况，如果设置的是`data`的根属性，那么需要把该属性代理到`vue`实例上，另外需要通知该实例及其所有子实例的`watcher`进行强制更新。如果不是根属性，那么调用所在对象的观察者实例的`notify`方法，通知对象对应的属性的订阅者进行更新。

数据观察到这里就结束了，但是现在还不知道，依赖到底是什么时候才进行收集的，`Observer.target`到底什么时候才会被赋值，如果数据更新了，`watcher`是什么，`watcher`又是怎么触发`DOM`更新以及怎么更新，问题还有很多，咱们下回再见。
