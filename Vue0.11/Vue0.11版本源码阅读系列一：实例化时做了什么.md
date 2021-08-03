---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

各位，你们没有看错，现在是2021年，vue3.0都已经出来很长一段时间了，而本系列将要带各位阅读的是0.11版本，也就是`vue`最早的正式版本，发布时间大概是六七年前，那时，嗯，太久远，都忘了我那时候在干什么，原因是2.0和3.0已经是一个很完善的框架了，代码量也很大，作为一个没啥源码阅读经验的老菜鸟，我不认为我有这个能力去看懂它，但同时又很想进一步的去看看它的真面目，思来想去，有两种思路，一是找到2.0或3.0的最早提交版本，然后一步一步的看它新增了什么，二是看它的早期版本，众所周知，早期版本一般都比较简单，最后决定先拿最早的版本练练手。

需要先说明的是0.11版本和2.x甚至是1.x语法区别都是很大的，但是核心思想是一致的，所以我们主要聚焦于响应式原理、模板编译等问题，具体的`api`不是咱们的重点，此外，这个版本因为实在太早了，所以没有虚拟节点，没有`diff`算法，想看这些的可以看看这位大神的系列文章：[https://github.com/answershuto/learnVue](https://github.com/answershuto/learnVue)和他的小册：[https://juejin.cn/book/6844733705089449991](https://juejin.cn/book/6844733705089449991)，话不多说，开始吧。

# 跑起来

0.11版本官方文档：[https://011.vuejs.org/guide/index.html](https://011.vuejs.org/guide/index.html)，仓库分支：[https://github.com/vuejs/vue/tree/0.11](https://github.com/vuejs/vue/tree/0.11)。

目录结构如下：

![image-20201229111517784](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f675a00600647a1bb8d8b47c4308ff2~tplv-k3u1fbpfcp-zoom-1.image)

看起来是不是挺清晰挺简单的，第一件事是要能把它跑起来，便于打断点进行调试，但是构建工具用的是`grunt`，不会，所以简单的使用`webpack`来配置一下：

1.安装：`npm install webpack webpack-cli webpack-dev-server html-webpack-plugin clean-webpack-plugin --save-dev`，注意要去看看`package.json`里面是不是已经有`webpack`了，有的话记得删了，不然版本不对。

2.在`/src`目录下新建一个`index.js`文件，用来作为我们的测试文件，输入：

```js
import Vue from './vue'
new Vue({
    el: '#app',
    data: {
        message: 'Hello Vue.js!'
    }
})
```

3.在`package.json`文件同级目录下新建一个`index.html`，输入：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>demo</title>
</head>
<body>
    <div id="app">
        <p>{{message}}</p>
        <input v-model="message">
    </div>
</body>
</html>
```

4.在`package.json`文件同级目录下新建一个`webpack`配置文件`webpack.config.js`，输入：

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = {
  mode: 'development',
  entry: {
    index: './src/index.js'
  },
  devtool: 'inline-source-map',
  output: {
    filename: '[name].bundle.js',
    path: path.resolve(__dirname, 'dist'),
  },
  devServer: {
    contentBase: './dist',
    hot: true
  },
  plugins: [
    new CleanWebpackPlugin({ cleanStaleWebpackAssets: false }),
    new HtmlWebpackPlugin({
      template: 'index.html'
    }),
  ],
};
```

5.最后配置一下`package.json`的执行命令：

```js
{
    "scripts": {
        "start": "webpack serve --hot only --host 0.0.0.0
    },
}
```

这样在命令行输入`npm start`就可以启动一个带热更新的服务了：

![image-20201229145416657](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b57cd08ca944488a83f7016fbdfe3e43~tplv-k3u1fbpfcp-zoom-1.image)

也可以直接克隆我的仓库[https://github.com/wanglin2/vue_v0.11_analysis](https://github.com/wanglin2/vue_v0.11_analysis)，已经配置好了并且翻译了英文注释。

# 构造函数

`Vue`的初始化工作主要是给`Vue`的构造函数和原型挂载方法和属性。

添加静态方法：

```js
function Vue (options) {
  this._init(options)
}
extend(Vue, require('./api/global'))
```

添加静态属性：

```js
Vue.options = {
  directives  : require('./directives'),
  filters     : require('./filters'),
  partials    : {},
  transitions : {},
  components  : {}
}
```

添加原型方法：

```js
var p = Vue.prototype
extend(p, require('./instance/init'))
extend(p, require('./instance/events'))
extend(p, require('./instance/scope'))
extend(p, require('./instance/compile'))
extend(p, require('./api/data'))
extend(p, require('./api/dom'))
extend(p, require('./api/events'))
extend(p, require('./api/child'))
extend(p, require('./api/lifecycle'))
```

`extend`方法很简单，就是一个浅拷贝函数：

```js
exports.extend = function (to, from) {
  for (var key in from) {
    to[key] = from[key]
  }
  return to
}
```

实例代理`data`属性：

```js
Object.defineProperty(p, '$data', {
  get: function () {
    return this._data
  },
  set: function (newData) {
    this._setData(newData)
  }
})
```

`_data`就是创建`vue`实例时传入的`data`数据对象。

构造函数里只调用了`_init`方法，这个方法首先定义了一堆后续需要使用的属性，包括公开的和私有的，然后会进行选项合并、初始化数据观察、初始化事件和生命周期，这之后就会调用`created`生命周期方法，如果传递了`$el`属性，接下来就会开始编译。

# 选项合并

```js
options = this.$options = mergeOptions(
    this.constructor.options,
    options,
    this
)
```

`constructor.options`就是上一节提到的那些静态属性，接下来看`mergeOptions`方法：

```js
guardComponents(child.components)
```

首先调用了`guardComponents`方法，这个方法用来处理我们传入的`components`选项，这个属性是用来注册组件的，比如：

```js
new Vue({
	components: {
        'to-do-list': {
            //...
        }
    }
})
```

组件其实也是个`vue`实例，所以这个方法就是用来把它转换成`vue`实例：

```js
function guardComponents (components) {
  if (components) {
    var def
    for (var key in components) {
      def = components[key]
      if (_.isPlainObject(def)) {
        def.name = key
        components[key] = _.Vue.extend(def)
      }
    }
  }
}
```

`isPlainObject`方法用来判断是不是纯粹的原始的对象类型：

```js
var toString = Object.prototype.toString
exports.isPlainObject = function (obj) {
  return toString.call(obj) === '[object Object]'
}
```

`vue`创建可复用组件调用的是静态方法`extend`，用来创建`Vue`构造函数的子类，为啥不直接`new Vue`呢？`extend`做了啥特殊操作呢？不要走开，接下来更精彩。

其实`extend`如字面意思继承，其实返回的也是个构造函数，因为我们知道组件是可复用的，如果直接`new`一个实例，那么即使在多处使用这个组件，实际上都是同一个，数据什么的都是同一份，修改一个影响所有，显然是不行的。

如果不使用继承的话，就相当于每使用一次该组件，就需要使用该组件选项去实例化一个新的`vue`实例，貌似也可以，所以给每个组件都创建一个构造函数可能是方便扩展和调试吧。

```js

exports.extend = function (extendOptions) {
  extendOptions = extendOptions || {}
  var Super = this
  // 创建子类构造函数
  var Sub = createClass(
    extendOptions.name ||
    Super.options.name ||
    'VueComponent'
  )
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
  // 这里也调用了mergeOptions方法
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super
  Sub.extend = Super.extend
  // 添加静态方法，如：directive、filter、transition等注册方法，以及component方法
  createAssetRegisters(Sub)
  return Sub
}
```

可以看到这个方法其实就是个类继承方法，一般我们创建子类会直接定义一个方法来当做子类的构造函数，如：

```js
function Par(name){
    this.name = name
}
Par.prototype.speak = function (){
    console.log('我叫' + this.name)
}
function Child(name){
    Par.call(this, name)
}
Child.prototype = new Par()
```

但是`Vue`这里使用的是`new Function`的方式：

```js
function createClass (name) {
  return new Function(
    'return function ' + _.classify(name) +
    ' (options) { this._init(options) }'
  )()
}
```

注释里的解释是：`This gives us much nicer output when logging instances in the console.`大意是方便在控制台打印。

回到选项合并方法：

```js
var key
if (child.mixins) {
    for (var i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
    }
}
```

因为每个`mixins`都可包含全部的选项，所以需要递归合并。

```js
for (key in parent) {
    merge(key)
}
for (key in child) {
    if (!(parent.hasOwnProperty(key))) {
        merge(key)
    }
}
function merge (key) {
    var strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
}
return options
```

然后是合并具体的属性，对不同的属性`vue`调用了不同的合并策略方法，有兴趣的可自行阅读。

# 初始化数据观察

选项参数合并完后紧接着调用了`_initScope`方法：

```js
exports._initScope = function () {
  this._initData()
  this._initComputed()
  this._initMethods()
  this._initMeta()
}
```

该方法又调用了四个方法，一一来看。

`_initData`方法及后续请移步第二篇：[vue0.11版本源码阅读系列二：数据观察](https://juejin.cn/post/6918314954986618887)。

`_initComputed`用来初始化计算属性：

```js
function noop () {}
exports._initComputed = function () {
  var computed = this.$options.computed
  if (computed) {
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
  }
}
```

设置计算属性的`gettter`和`setter`，然后定义到实例上成为实例的一个属性，我们都知道计算属性所依赖的数据变化了它也会跟着变化，根据上述代码，似乎不太明显，但是很容易理解的一点是通过`this.xxx`在任何时候引用计算属性它是会执行对应的函数的，所以拿到的值肯定是最新的，问题就是使用了计算属性的模板如何知道要更新，目前看不出来，后续再说。

`bind`方法用来设置函数的上下文对象，一般有：`call`、`apply`、`bind`三种方法，第三种方法执行后会返回一个新函数，这里`vue`使用`apply`简单模拟了一下`bind`方法，原因是比原生更快，缺点是不如原生完善：

```js
exports.bind = function (fn, ctx) {
  return function () {
    return fn.apply(ctx, arguments)
  }
}
```

`_initMethods`就比较简单了，把方法都代理到`this`上，更方便使用：

```js
exports._initMethods = function () {
  var methods = this.$options.methods
  if (methods) {
    for (var key in methods) {
      this[key] = _.bind(methods[key], this)
    }
  }
}
```

上述方法都使用`bind`方法把函数的上下文设置为`vue`实例，这样才能在函数里访问到实例上的其他方法或属性，这就是为什么不能使用箭头函数的原因，因为箭头函数没有自己的`this`。

# 初始化事件

`_initEvents`方法会遍历`watch`选项并调用`$watch`方法来观察数据，所以直接看`$watch`方法：

```js
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
  if (immediate) {
    wrappedCb(watcher.value)
  }
  return function unwatchFn () {
    watcher.removeCb(wrappedCb)
    if (!watcher.active) {
      vm._userWatchers[key] = null
    }
  }
}
```

检查要观察的表达式是否已经存在，存在则追加该回调函数，否则创建并存储一个新的`watcher`实例，最后返回一个方法用来解除观察，所以要想理解最终的原理，还是得后续再看`Watcher`的实现。

这一步结束后就会触发`created`生命周期方法：`this._callHook('created')`：

```js
exports._callHook = function (hook) {
  var handlers = this.$options[hook]
  if (handlers) {
    for (var i = 0, j = handlers.length; i < j; i++) {
      handlers[i].call(this)
    }
  }
  this.$emit('hook:' + hook)
}
```

最后如果传了挂载元素，则会立即开始编译，编译相关请阅读：[vue0.11版本源码阅读系列三：指令编译](https://juejin.cn/post/6918313229449953293)。

