这一篇我们来看一下`Vue`核心的响应性原理，在上一篇我们知道了初始化时`Vue`会把`data`选项的数据递归的转换成响应性的数据，具体来说就是给数组和对象创建一个关联的`Observer`实例，然后对于数组，会拦截它的所有方法，以此来监听数组的变化，对于普通对象，会遍历它自身所有可枚举的属性，将其转换成`setter`和`getter`形式，以此来监听某个属性的变化，为什么要这么做呢，我们来看一下。

首先简单介绍一下挂载的逻辑，如果我们传递了模板字符串，那么会进行编译，编译模板这个过程非常复杂，反正最后会将其转换为渲染函数，如果直接是渲染函数了那么就没有这个过程，像我们平时使用`Vue`单文件的形式开发的话，在打包阶段其实就已经编译好了，渲染函数就是返回虚拟`DOM`（也就是`VNode`）的函数，`VNode`怎么转换成实际的`DOM`呢，其中涉及到虚拟`DOM`的`patch`算法，后面会介绍，只要记得这个过程会获取模板里使用到的数据。接着还会给每个`Vue`实例创建一个`Watcher`实例，这部分的代码简化如下：

```js
// 更新组件的方法
let updateComponent = () => {
    // 执行渲染函数产出VNode，然后调用_update方法进行打补丁
    vm._update(vm._render(), hydrating)
}

new Watcher(vm, updateComponent, noop, {
    before () {
        if (vm._isMounted && !vm._isDestroyed) {
            callHook(vm, 'beforeUpdate')
        }
    }
}, true /* isRenderWatcher */)
```

定义了一个更新方法，然后传给`Watcher`，看`Watcher`类的实现之前我们先看一下`Dep`类的实现。

# Dep类

`dep`是一个可观察对象，可以有多个指令订阅它：

```js
let uid = 0
export default class Dep {
  static target;
  id;
  subs;

  constructor () {
    this.id = uid++
    this.subs = []
  }
}
```

定义了两个实例属性，一个静态属性，`subs`数组就是用来存放订阅它的对象，有四个实例方法：

1.添加订阅者

```js
addSub (sub) {
    this.subs.push(sub)
}
```

2.删除订阅者

```js
removeSub (sub) {
    remove(this.subs, sub)
}
```

3.依赖收集

```js
depend () {
    if (Dep.target) {
        Dep.target.addDep(this)
    }
}
```

4.通知订阅者更新

```js
notify () {
    // 只通知此刻存在的订阅者，调用其update方法
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
    }
}
```

# Watcher类

`dep`的订阅者其实就是`Watcher`实例：

```js
export default class Watcher {
  constructor (
    vm,
    expOrFn,
    cb,
    options,
    isRenderWatcher
  ) {
    this.vm = vm
    // 如果是渲染watcher，那么会将该Watcher实例添加到Vue实例的_watcher属性中
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // 用于批处理的uid
    this.active = true
    this.dirty = this.lazy // lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = ''
    // 从表达式中解析出getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
      }
    }
    // 执行取值方法，计算当前的值
    this.value = this.lazy
      ? undefined
      : this.get()
  }
}
```

`watcher`做的事情主要是解析表达式、依赖收集、并在表达式的值发生改变时触发回调。`Vue`把给组件创建的`Watcher`实例称为`渲染watcher`。

构造函数里先定义了一堆变量，然后判断表达式的类型，是函数的话那么直接使用该函数作为取值函数，否则调用`parsePath`方法来解析路径，并返回一个取值函数，这是一个简单的解析方法，只支持对`.`分隔的字符串进行解析：

```js
const bailRE = new RegExp(`[^${unicodeLetters}.$_\\d]`)
export function parsePath (path) {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    // 遍历路径，一层一层取值
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

比如实例的`data`结构如下：

```js
let vm = new Vue({
    data: {
        a: {
            b: {
                c: 1
            }
        }
    }
})
```

那么当我们使用表达式`a.b.c`来创建`Watcher`实例的话：

```js
new Watcher(vm, 'a.b.c', () => {})
```

那么以`vm`为上下文执行`parsePath`方法返回的函数时就能成功获取到值`1`。

构造函数的最后，如果`lazy`不为`true`的话会立即执行取值函数`get`，计算表达式的当前值。

## 依赖收集

具体到开头的挂载过程，表达式`expOrFn`就是`updateComponent`函数，也就是`Watcher`实例的`this.getter`，它会在`get`函数里调用：

```js
get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
        value = this.getter.call(vm, vm)
    } catch (e) {} finally {
        // "touch"每个属性，以便它们都作为依赖项进行跟踪，以便进行深入观察
        if (this.deep) {
            traverse(value)
        }
        popTarget()
        this.cleanupDeps()
    }
    return value
}
```

首先执行了`pushTarget`方法：

```js
// 当前正在计算执行中的目标watcher
// 这是全局唯一的，因为一次只能同时执行一个观察者。
Dep.target = null
const targetStack = []

export function pushTarget (target) {
  targetStack.push(target)
  Dep.target = target
}
```

`Dep.target`是一个全局变量，这一步相当于把当前的这个`Watcher`实例赋值给了`Dep.target`属性。

随后执行了`getter`函数，也就是执行`updateComponent`函数，在`vm._render`函数中会获取我们定义在`data`中的数据（如果模板里使用到了的话），此时，就会触发对应数据的`getter`函数。

比如：

```html
<div>{{a}}</div>
```

```js
new Vue({
    data: {
        a: 1
    }
})
```

在执行渲染函数的时候会执行`vm.a`来获取`a`的值，属性`a`已经被转换成了`getter`和`setter`的形式：

```js
Object.defineProperty(obj, key, {
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      // Dep.target就是当前的Watcher实例
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
    }
})
```

所以会触发属性`a`的`get`函数执行，此时`Dep.target`就是当前的`Watcher`实例，那么会执行`dep.depend`方法，`dep`是属性`a`的依赖收集对象，`depend`方法内会执行`Dep.target`也就是当前的`Watcher`实例的`addDep`方法：

```js
export default class Watcher {
    addDep (dep) {
        const id = dep.id
        if (!this.newDepIds.has(id)) {
            this.newDepIds.add(id)
            this.newDeps.push(dep)
            if (!this.depIds.has(id)) {
                dep.addSub(this)
            }
        }
    }
}
```

可以看到这里把属性`a`的`dep`添加到了`newDeps `集合中，为什么不直接添加到`deps`中呢？而要区分新旧呢？这其实要结合`cleanupDeps`方法一起看，这个方法在`get`方法的最后调用了，用来清理依赖项：

```js
export default class Watcher {
    cleanupDeps () {
        let i = this.deps.length
        // 如果之前依赖某个dep，此时不依赖了，那么要从对应dep里删除该watcher
        while (i--) {
            const dep = this.deps[i]
            if (!this.newDepIds.has(dep.id)) {
                dep.removeSub(this)
            }
        }
        // newDepIds转为depIds，并清空旧的depIds
        let tmp = this.depIds
        this.depIds = this.newDepIds
        this.newDepIds = tmp
        this.newDepIds.clear()
        // newDeps转为deps，并清空旧的deps
        tmp = this.deps
        this.deps = this.newDeps
        this.newDeps = tmp
        this.newDeps.length = 0
    }
}
```

现在应该很清楚为什么要区分新旧了，因为当前`watcher`之前的依赖项，现在可能已经不依赖了，那么要找出不再依赖的`dep`，并从中删除当前`watcher`，所以需要一个新旧对比的过程。

`addDep`方法执行完后，属性`a`的`dep`里会收集到当前的渲染`watcher`，渲染`watcher`里也会保存属性`a`的`dep`。

回到`a`的`getter`函数，`depend`函数执行完后，接着：

```js
if (childOb) {
    childOb.dep.depend()
    if (Array.isArray(value)) {
        dependArray(value)
    }
}
```

如果属性`a`的值是数组或对象的话，那么也会创建一个关联的`Observer`实例：

```js
let childOb = !shallow && observe(val)
```

这里相当于当前的渲染`watcher`也会同时订阅`a`的值对应的`dep`对象，比如下面这种情况：

```js
new Vue({
    data: {
        a: {
            b: 1
        }
    }
})
```

渲染`watcher`的`deps`数组一共会收集到两个`dep`实例，一个是属性`a`的，一个是`a`的值的。

最后对于`a`的值为数组的情况还调用了`dependArray`方法：

```js
function dependArray(value ) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```

递归遍历数组，对所有对象都进行依赖收集。

总结一下，当我们创建一个`Vue`实例时，内部会创建一个`渲染watcher`，`watcher`实例化时会把自身赋值给全局的`Dep.target`，然后执行取值函数，也就是会执行渲染函数，如果模板里引用了`data`的数据，那么就会触发对`data`数据的读取，相应的会触发对应属性的`getter`，在`getter`里会进行依赖收集，也就是会把`Dep.target`属性指向的`watcher`收集为依赖添加到它的`dep`实例中，同样该`watcher`也会保存该`dep`，如果该属性存在子`observer`，那么也会进行依赖收集，如果该属性的值是数组类型，还会递归遍历数组，给每个对象类型的数组项都进行依赖收集。

接下来回到`Watcher`实例的`get`方法，执行完取值方法把依赖收集完毕后：

```js
if (this.deep) {
    traverse(value)
}
```

如果`deep`选项配置为`true`的话，那么会对值调用`traverse`方法，对于实例化`Vue`时，该选项是`false`，所以并不会走这里，我们以一个其他例子来看：

```js
new Vue({
    data: {
        obj: {
            a: {
                b: 1
            }
        }
    },
    created() {
        this.$watch('obj', () => {
            console.log('变了')
        })
        setTimeout(() => {
          this.obj.a.b = 2
        }, 2000);
    }
})
```

这个例子中，两秒后我们修改了`obj.a.b`的属性值，它并不会触发回调，当我们`deep`设为`true`时，就会触发：

```js
this.$watch('obj', () => {
    console.log('变了')
}, {
    deep: true
})
```

这就是上面的`traverse`函数做的事情，这里的取值函数返回的`value`就是`obj`的值。

```js
const seenObjects = new Set()
export function traverse (val) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}
```

调用了`_traverse`方法：

```js
export function traverse (val) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}

function _traverse (val, seen) {
  let i, keys
  const isA = Array.isArray(val)
  // 非数组和对象就返回
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (val.__ob__) {
    // 这里我猜测是避免循环引用
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  // 递归
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

其实就是一个简单的递归函数，递归遍历了所有的数组项和对象的`key`，为什么这样就能实现监听深层值的变化呢，很简单，因为读取了所有的属性，也就是触发了它们的`getter`函数，所以所有属性的`dep`都收集了当前的`watcher`实例，那么当然如何一层里的任何一个属性修改了都会触发该`watcher`实例的更新。

回到`get`函数，接下来的逻辑：

```js
popTarget()
this.cleanupDeps()
```

`get`函数的开始调用了`pushTarget`，依赖收集完毕当然要撤销这个操作：

```js
export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

`cleanupDeps`方法前面已经说过，不再赘述。

到这里，`Watcher`实例化的过程就已经结束了，现在我们的依赖都已经收集完毕，那么它们有什么用呢？

## 触发依赖更新

例子如下：

```js
new Vue({
  el: '#app',
  template: `
    <ul>
      <li v-for="item in list">{{item}}</li>
    </ul>
  `,
  data: {
    list: [1, 2, 3]
  },
  created() {
    setTimeout(() => {
      this.list = [4, 5, 6]
    }, 5000);
  }
})
```

先来考考大家，这个例子里面的渲染`watcher`会收集到几个`dep`呢？

<details>
  <summary>点击查看答案</summary>
  <p>2个，list属性一个，list的属性值数组一个</p>
</details>

五秒后我们修改了`list`的值，显然会触发它的`setter`：

```js
Object.defineProperty(obj, key, {
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
```

首先更新属性值，然后如果新的值是数组或对象的话那么会调用`observe`方法来将它转成响应式的，这样当我们修改这个新数组或新对象本身时才能触发更新。

最后通知依赖更新，这里的依赖就是渲染`watcher`，`notify`方法里会调用`watcher`的`update`方法：

```js
export default class Watcher {
    update () {
        if (this.lazy) {
            this.dirty = true
        } else if (this.sync) {
            this.run()
        } else {
            queueWatcher(this)
        }
    }
}
```

可以看到有三个分支，如果`lazy`为`true`的话那么只设置一下`dirty`属性，这个属性具体有什么作用后面我们如果遇到了再说，如果是同步的，那么会直接执行`run`函数：

```js
run () {
    if (this.active) {
        const value = this.get()
        if (
            value !== this.value ||
            // 即使值相同，深度的watcher和对象/数组上的watcher也应该触发，因为值可能已经发生了变化。
            isObject(value) ||
            this.deep
        ) {
            // 设置为新值
            const oldValue = this.value
            this.value = value
            // 用户的watcher，即通过$watch方法或watch选项设置的
            if (this.user) {
                try {
                    this.cb.call(this.vm, value, oldValue)
                } catch (e) {
                    handleError(e, this.vm, `callback for watcher "${this.expression}"`)
                }
            } else {
                this.cb.call(this.vm, value, oldValue)
            }
        }
    }
}
```

这个函数的核心就是调用了`get`方法更新当前表达式的值，然后当值有变化则会调用回调函数。

当也不是同步的`watcher`，那么会执行`queueWatcher`方法，这个涉及到`Vue`的调度功能，也就是异步批量执行的功能，我们会单独开一篇文章来介绍。

总结一下如何触发依赖更新，当依赖收集完毕后，如果我们更新了某个属性的值，那么该属性值的`getter`函数会被执行，就会通知保存在该属性值的`dep`里的所有依赖，调用它们的`update`方法进行更新。

对于我们开头的例子，也就是会调用`渲染watcher`的`update`方法，重新执行取值函数，也就是会执行`updateComponent`方法来重新执行渲染函数，并进行打补丁更新组件，达到数据更新页面自动更新的效果。
