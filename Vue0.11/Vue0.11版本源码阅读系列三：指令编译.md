---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

因为`vue`指令很多，功能也很多，所以会有很多针对一些情况的特殊处理，这些逻辑如果不是对`vue`很熟悉的话一时间是看不懂的，所以我们只看一些基本逻辑。

# compile

创建`vue`实例时当传递了参数`el`或者手动调用`$mount`方法即可启动模板编译过程，`$mount`方法里调用了`_compile`方法，简化之后其实调用的是`compile(el, options)(this, el)`，`compile`也简化后代码如下：

```js
function compile (el, options, partial, transcluded) {
  var nodeLinkFn = compileNode(el, options)
  var childLinkFn = el.hasChildNodes()
      ? compileNodeList(el.childNodes, options)
      : null
  
  function compositeLinkFn (vm, el) 
    var childNodes = _.toArray(el.childNodes)
    if (nodeLinkFn) nodeLinkFn(vm.$parent, el)
    if (childLinkFn) childLinkFn(vm.$parent, childNodes)
  }

  return compositeLinkFn
}
```

该方法会根据实例的一些状态来判断处理某个部分使用哪个方法，因为代码极大的简化了所以不是很明显。

先来看`compileNode`方法，这个方法会会对普通节点和文本节点调用不同的方法，只看普通节点：

```js
function compileElement (el, options) {
  var linkFn, tag, component
  // 检查是否是自定义元素，也就是子组件
  if (!el.__vue__) {
    tag = el.tagName.toLowerCase()
    component =
      tag.indexOf('-') > 0 &&
      options.components[tag]
    // 是自定义组件则给元素设置一个属性标志
    if (component) {
      el.setAttribute(config.prefix + 'component', tag)
    }
  }
   // 如果是自定义组件或者元素有属性的话
  if (component || el.hasAttributes()) {
    // 检查 terminal 指令
    linkFn = checkTerminalDirectives(el, options)
    // 如果不是terminal，建立正常的链接功能
    if (!linkFn) {
      var dirs = collectDirectives(el, options)
      linkFn = dirs.length
        ? makeNodeLinkFn(dirs)
        : null
    }
  }
  return linkFn
}
```

`terminal` 指令有三种：`repeat`、`if`、`'component`：

```js
var terminalDirectives = [
  'repeat',
  'if',
  'component'
]
function skip () {}
skip.terminal = true
function checkTerminalDirectives (el, options) {
  // v-pre指令是用来告诉vue跳过编译该元素及其所有子元素
  if (_.attr(el, 'pre') !== null) {
    return skip
  }
  var value, dirName
  for (var i = 0; i < 3; i++) {
    dirName = terminalDirectives[i]
    if (value = _.attr(el, dirName)) {
      return makeTerminalNodeLinkFn(el, dirName, value, options)
    }
  }
}
```

顺便一提的是`attr`方法，这个方法其实是专门用来获取`vue`的自定义属性的，也就是`v-`开头的属性，为什么我们在模板里写的带`v-`前缀的属性在最终渲染的元素上没有呢，就是因为在这个方法里把它给移除了：

```js
exports.attr = function (node, attr) {
  attr = config.prefix + attr
  var val = node.getAttribute(attr)
  // 如果该自定义指令存在，则把它从元素上删除
  if (val !== null) {
    node.removeAttribute(attr)
  }
  return val
}
```

`makeTerminalNodeLinkFn`方法：

```js
function makeTerminalNodeLinkFn (el, dirName, value, options) {
  // 解析指令值
  var descriptor = dirParser.parse(value)[0]
  // 获取该指令的指令方法，vue内置了很多指令处理方法，都在/src/directives/文件夹下
  var def = options.directives[dirName]
  var fn = function terminalNodeLinkFn (vm, el, host) {
    // 创建并把指令绑定到元素
    vm._bindDir(dirName, el, descriptor, def, host)
  }
  fn.terminal = true
  return fn
}
```

`parse`方法用来解析指令的值，请移步文章：[vue0.11版本源码阅读系列四：详解指令值解析函数](https://juejin.cn/post/6918315630374584334)，比如指令值为`click: a = a + 1 | uppercase`，处理完最后会返回这样的信息：

```json
{
    arg: 'click',
    expression: 'a = a + 1',
    filters: [
        { name: 'uppercase', args: null }
    ]
}
```

`_bindDir`方法会创建一个指令实例：

```js
exports._bindDir = function (name, node, desc, def, host) {
  this._directives.push(
    new Directive(name, node, this, desc, def, host)
  )
}
```

所以`linkFn`以及`nodeLinkFn`就是这个`_bindDir`的包装函数。

对于非`terminal`指令，调用的是`collectDirectives`方法，这个方法会遍历元素的所有属性`attributes`，如果是`v-`前缀的`vue`指令会被定义为下列格式的对象：

```js
{
    name: dirName,// 去除了v-前缀的指令名
    descriptors: dirParser.parse(attr.value),// 指令值解析后的数据
    def: options.directives[dirName],// 该指令对应的处理方法
    transcluded: transcluded
}
```

非`vue`指令的属性如果存在动态绑定，也会进行处理，在该版本`vue`里的动态绑定是使用双大括号插值的，和2.x的使用`v-bind`不一样。

如：`<div class="{{error}}"></div>`，所以会通过正则来匹配判断是否存在动态绑定，最终返回下列格式的数据：

```js
{
    def: options.directives.attr,
    _link: allOneTime// 是否所有属性都是一次性差值
    ? function (vm, el) {// 一次性的话后续不需要更新
        el.setAttribute(name, vm.$interpolate(value))
    }
    : function (vm, el) {// 非一次性的话如果依赖的响应数据变化了也需要改变
        var value = textParser.tokensToExp(tokens, vm)
        var desc = dirParser.parse(name + ':' + value)[0]
        vm._bindDir('attr', el, desc, def)
    }
}
```

`collectDirectives`方法最终会返回一个上面对象组成的数组，然后调用`makeNodeLinkFn`为每个指令创建一个绑定函数：

```js
function makeNodeLinkFn (directives) {
  return function nodeLinkFn (vm, el, host) {
    var i = directives.length
    var dir, j, k, target
    while (i--) {
      dir = directives[i]
      if (dir._link) {
        dir._link(vm, el)
      } else {// v-前缀的指令
        k = dir.descriptors.length
        for (j = 0; j < k; j++) {
          vm._bindDir(dir.name, el,
            dir.descriptors[j], dir.def, host)
        }
      }
    }
  }
}
```

总结一下`compileNode`的作用就是遍历元素上的属性，分别给其创建一个指令绑定函数，这个指令函数后续调用时会创建一个`Directive`实例，这个类后续再看。

如果该元素存在子元素的话会调用`compileNodeList`方法，子元素又有子元素的话又会继续调用，其实就是递归所有子元素调用`compileNode`方法。

`compile`方法最后返回了`compositeLinkFn`方法，这个方法被立即执行了，这个方法里调用了刚才生成的`nodeLinkFn`和`childLinkFn`方法，执行结果就是会把所有的元素及子元素的指令进行绑定，也就是给元素上的某个属性或者说指令都创建了一个`Directive`实例。

# Directive

指令这个类主要做的事是把`DOM`和数据绑定起来，实例化的时候会调用指令的`bind`方法，同时会实例化一个`Watcher`实例，后续数据更新的时候会调用指令的`update`方法。

```js
function Directive (name, el, vm, descriptor, def, host) {
  this.name = name
  this.el = el
  this.vm = vm
  this.raw = descriptor.raw
  this.expression = descriptor.expression
  this.arg = descriptor.arg
  this.filters = _.resolveFilters(vm, descriptor.filters)
  this._host = host
  this._locked = false
  this._bound = false
  this._bind(def)
}
```

构造函数定义一些属性以及调用了`_bind`方法，`resolveFilters`方法会把过滤器以`getter`和`setter`分别收集到一个数组里，便于后续循环调用：

```js
exports.resolveFilters = function (vm, filters, target) {
  var res = target || {}
  filters.forEach(function (f) {
    var def = vm.$options.filters[f.name]
    if (!def) return
    var args = f.args
    var reader, writer
    if (typeof def === 'function') {
      reader = def
    } else {
      reader = def.read
      writer = def.write
    }
    if (reader) {
      if (!res.read) res.read = []
      res.read.push(function (value) {
        return args
          ? reader.apply(vm, [value].concat(args))
          : reader.call(vm, value)
      })
    }
    if (writer) {
      if (!res.write) res.write = []
      res.write.push(function (value, oldVal) {
        return args
          ? writer.apply(vm, [value, oldVal].concat(args))
          : writer.call(vm, value, oldVal)
      })
    }
  })
  return res
}
```

`_bind`方法：

```js
p._bind = function (def) {
  if (typeof def === 'function') {
    this.update = def
  } else {// 这个版本的vue指令有这几个钩子方法：bind、update、unbind
    _.extend(this, def)
  }
  this._watcherExp = this.expression
  // 如果该指令存在bind方法，此时进行调用
  if (this.bind) {
    this.bind()
  }
  if (this._watcherExp && this.update){
    var dir = this
    var update = this._update = function (val, oldVal) {
        dir.update(val, oldVal)
    }
    // 使用原始表达式作为标识符，因为过滤器会让同一个arg变成不同的观察者
    var watcher = this.vm._watchers[this.raw]
    if (!watcher) {
      // 该表达式未创建过watcher，则实例化一个
      watcher = this.vm._watchers[this.raw] = new Watcher(
        this.vm,
        this._watcherExp,
        update,
        {
          filters: this.filters
        }
      )
    } else {// 存在则把更新函数添加进入
      watcher.addCb(update)
    }
    this._watcher = watcher
    if (this._initValue != null) {// 带初始值的情况，见于v-model的情况
      watcher.set(this._initValue)
    } else if (this.update) {// 其他的会调用update方法，所以bind方法调用后紧接着会调用update方法
      this.update(watcher.value)
    }
  }
  this._bound = true
}
```

到这里可以知道实例化`Directive`的时候会调用指令的`bind`钩子函数，一般是做一些初始化工作，然后会对该指令初始化一个`Watcher`实例，这个实例会用来做依赖收集，最后非`v-model`的情况会立即调用指令的`update`方法，`watcher`实例化的时候会计算表达式的值，所以此时得到的`value`就是最新的。

# Watcher

`Watcher`实例用来解析表达式和收集依赖项，并在表达式的值变化时触发回调更新。第一篇里提到的`$watch`方法也是使用该类实现的。

```js
function Watcher (vm, expression, cb, options) {
  this.vm = vm
  this.expression = expression
  this.cbs = [cb]
  this.id = ++uid
  this.active = true
  options = options || {}
  this.deep = !!options.deep
  this.user = !!options.user
  this.deps = Object.create(null)
  if (options.filters) {
    this.readFilters = options.filters.read
    this.writeFilters = options.filters.write
  }
  // 将表达式解析为getter/setter
  var res = expParser.parse(expression, options.twoWay)
  this.getter = res.get
  this.setter = res.set
  this.value = this.get()
}
```

构造函数的逻辑很简单，声明一些变量、将表达式解析为`getter`和`setter`的类型，比如：`a.b`解析后的`get`为：

```js
function anonymous(o){
    return o.a.b
}
```

`set`为：

```js
function set(obj, val){
    Path.se(obj, path, val)
}
```

简单的说就是生成两个函数，一个用来给实例`this`设置值，一个用来获取实例`this`上的值，具体的解析逻辑比较复杂，有机会再详细分析或者可自行阅读源码：`/src/parsers/path.js`。

最后调用了`get`方法：

```js
p.get = function () {
  this.beforeGet()
  var vm = this.vm
  var value
  // 调用取值方法
  value = this.getter.call(vm, vm)
  // “触摸”每个属性，以便它们都作为依赖项进行跟踪，以便进行深入观察
  if (this.deep) {
    traverse(value)
  }
  // 应用过滤器函数
  value = _.applyFilters(value, this.readFilters, vm)
  this.afterGet()
  return value
}
```

在调用取值函数前调用了`beforeGet`方法：

```js
p.beforeGet = function () {
  Observer.target = this
  this.newDeps = {}
}
```

到这里我们知道了第二篇[vue0.11版本源码阅读系列二：数据观察](https://juejin.cn/post/6918314954986618887)里提到的`Observer.target`是什么了，逻辑也可以串起来，`vue`在数据观察时对每个属性进行了拦截，在`getter`里会判断`Observer.target`是否存在，存在的话会把`Observer.target`对应的`watcher`实例收集到该属性的依赖对象实例`dep`里：

```js
if (Observer.target) {
    Observer.target.addDep(dep)
}
```

`beforeGet`后紧接着就调用了该表达式的取值函数，会触发对应属性的`getter`。

`addDep`方法：

```js
p.addDep = function (dep) {
  var id = dep.id
  if (!this.newDeps[id]) {
    this.newDeps[id] = dep
    if (!this.deps[id]) {
      this.deps[id] = dep
      // 收集该watcher实例到该属性的依赖对象里
      dep.addSub(this)
    }
  }
}
```

`afterGet`用来做一些复位和清理工作：

```js
p.afterGet = function () {
  Observer.target = null
  for (var id in this.deps) {
    if (!this.newDeps[id]) {// 删除本次依赖收集时已经不依赖的属性
      this.deps[id].removeSub(this)
    }
  }
  this.deps = this.newDeps
}
```

`traverse`方法用来深度遍历所有嵌套属性，这样已转换的所有嵌套属性都会作为依赖项进行收集，也就是该表达式的`watcher`会被该属性及其所有后代属性的`dep`对象收集，这样某个后代属性的值变了也会触发更新：

```js
function traverse (obj) {
  var key, val, i
  for (key in obj) {
    val = obj[key]// 就是这里，获取一下该属性即可触发getter，此时Observer.target属性还是该watcher
    if (_.isArray(val)) {
      i = val.length
      while (i--) traverse(val[i])
    } else if (_.isObject(val)) {
      traverse(val)
    }
  }
}
```

如果某个属性的值后续发生变化根据第一篇我们知道在属性`setter`函数里会调用订阅者的`update`方法，这个订阅者就是`Watcher`实例，看一下这个方法：

```js
p.update = function () {
  if (!config.async || config.debug) {
    this.run()
  } else {
    batcher.push(this)
  }
}
```

正常情况下是走`else`分支的，`batcher`会以异步和批量的方式来更新，但是最后也调用了`run`方法，所以先来看一下这个方法：

```js
p.run = function () {
  if (this.active) {
    // 获取表达式的最新值
    var value = this.get()
    if (
      value !== this.value ||
      Array.isArray(value) ||
      this.deep
    ) {
      var oldValue = this.value
      this.value = value
      var cbs = this.cbs
      for (var i = 0, l = cbs.length; i < l; i++) {
        cbs[i](value, oldValue)
		// 某个回调删除了其他的回调的情况，目前属实不了解
        var removed = l - cbs.length
        if (removed) {
          i -= removed
          l -= removed
        }
      }
    }
  }
}
```

逻辑很简单，遍历调用该`watcher`实例所有指令的`update`方法，指令会完成页面的更新工作。

批量更新请移步文章[vue0.11版本源码阅读系列五：批量更新是怎么做的](https://juejin.cn/post/6918316455142686734)。

到这里模板编译的过程就结束了，接下来以一个指令的视角来看一下具体过程。



# 以if指令来看一下全过程

模板如下：

```html
<div id="app">
    <div v-if="show">我出来了</div>
</div>
```

`JavaScript`代码如下：

```js
window.vm = new Vue({
    el: '#app',
    data: {
        show: false
    }
})
```

在控制台输入`window.vm.show = true`这个`div`就会显示出来。

根据上面的分析，我们知道对于`v-if`这个指令最终肯定调用了`_bindDir`方法：

![image-20210111194832587](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64a8f1f160724f24bd7bff7e22df433c~tplv-k3u1fbpfcp-zoom-1.image)

进入`Directive`后在`_bind`里调用了`if`指令的`bind`方法，该方法简化后如下：

```js
{
    bind: function () {
        var el = this.el
        if (!el.__vue__) {
            // 创建了两个注释元素把我们要显示隐藏的div给替换了，效果见下图
            this.start = document.createComment('v-if-start')
            this.end = document.createComment('v-if-end')
            _.replace(el, this.end)
            _.before(this.start, this.end)
        }
    }
}
```

![image-20210111200014709](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eeb82a7d11914e61a78c348537d620e4~tplv-k3u1fbpfcp-zoom-1.image)

可以看到`bind`方法做的事情是用两个注释元素把这个元素从页面上给替换了。 `bind`方法之后就是给这个指令创建`watcher`：

![image-20210111201925172](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebc1d0ce68584e53a765a0480ef1237a~tplv-k3u1fbpfcp-zoom-1.image)

接下来在`watcher`里给`Observer.target`赋值及进行取值操作，触发了`show`属性的`getter`：

![image-20210112093317760](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/173e9030e98d4757a05af479356be010~tplv-k3u1fbpfcp-zoom-1.image)

依赖收集完后会调用`if`指令的`update`方法，看一下这个方法：

```js
{
    update: function (value) {
        if (value) {
            if (!this.unlink) {
                var frag = templateParser.clone(this.template)
                this.compile(frag)
            }
        } else {
            this.teardown()
        }
    }
}
```

因为我们的初始值为`false`，所以走`else`分支调用了`teardown`方法：

```js
{
    teardown: function () {
        if (!this.unlink) return
        transition.blockRemove(this.start, this.end, this.vm)
        this.unlink()
        this.unlink = null
    }
}
```

本次`unlink`其实并没有值，所以就直接返回了，但是假如有值的话，`teardown`方法首先使用会使用`transition`类来移除元素，然后解除该指令的绑定。

现在让我们在控制台输入`window.vm.show = true`，这会触发`show`的`setter`：

![image-20210112101502685](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11c75e8b40f941c5bc9800915e076d1c~tplv-k3u1fbpfcp-zoom-1.image)

然后会调用`show`属性的`dep` 的`notify`方法，`dep`的订阅者里目前就只有`if`指令的`watcher`，所以会调用`watcher`的`update`方法，最终调用到`if`指令的`update`方法，此时的值为`true`，所以会走到`if`分支里，`unlink`也没有值，所以会调用`compile`方法：

![image-20210112102302256](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dad08edc1e94f5396e14dcf927e4314~tplv-k3u1fbpfcp-zoom-1.image)

```js
{
    compile: function (frag) {
        var vm = this.vm
        transition.blockAppend(frag, this.end, vm)
    }
}
```

忽略了部分编译过程，可以看到使用看`transition`类来显示元素。这个过渡类我们将在[vue0.11版本源码阅读系列六：过渡原理](https://juejin.cn/post/6918316620561514510)里详细了解。



# 总结

可以发现在这个早期版本里没有所谓的虚拟`DOM`，没有`diff`算法，模板编译就是遍历元素及元素上的属性，给每个属性创建一个指令实例，对同样的指令表达式创建一个`watcher`实例，指令实例提供`update`方法给`watcher`，`watcher`会触发表达式里所有被观察属性的`getter`，然后`watcher`就会被这些属性的依赖收集实例`dep`收集起来，当属性值变化时会触发`setter`，在`setter`里会遍历`dep`里所有的`watcher`，调用更新方法，也就是指令实例提供的`update`方法，也就是最终指令对象的`update`方法完成页面更新。

当然，这部分的代码还是比较复杂的，远没有本文所说的这么简单，各种递归调用，各种函数重载，反复调用，让人看的云里雾里，有兴趣的还请自行阅读。

