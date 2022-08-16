# 调试

阅读源码最好的方式还是通过调试，否则很容易在源码里迷失方向，本文提供了一种方便的方式来调试：

```bash
git clone https://github.com/wanglin2/learn_vue_2.6.0.git
cd learn_vue_2.6.0
npx http-server -e js -c-1
```

然后访问：`http://【你的ip:端口】/examples/index.html`即可打开测试页面，测试文件在`/examples/`目录下，你可以修改`html`和`js`文件，然后刷新页面即可。



# 引入vue时的初始化工作

首先`Vue`是存在很多构建版本的，我们要阅读的是包含编译器和运行时的`UMD`版本，对应打包后的文件是`/dist/vue.js`，入口文件是在`/src/platforms/web/entry-runtime-with-compiler.js`，这个文件主要是定义了`Vue`在`web`环境下的`$mount`方法。`Vue`对象是从`./runtime/index`文件引入的，这个文件主要是添加一些特定平台的方法，比如一些工具方法：

```js
Vue.config.mustUseProp = mustUseProp
// ...
```

添加平台运行时指令(v-model、v-show)和组件（Transition、TransitionGroup）：

```js
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)
```

给原型添加`__patch__`和`$mount`方法：

```js
Vue.prototype.__patch__ = inBrowser ? patch : noop
Vue.prototype.$mount = function ...
```

`Vue`构造函数依旧是从其他文件引入的：`core/index.js`，这个文件里做的事情比较多，首先执行了`initGlobalAPI`方法，把全局配置对象挂载到`Vue`上：

```js
const configDef = {}
configDef.get = () => config
Object.defineProperty(Vue, 'config', configDef)
```

然后添加了一些内部使用的工具方法：

```js
Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
}
```

接着添加了`set`、`delete`等静态方法：

```js
Vue.set = set
Vue.delete = del
Vue.nextTick = nextTick
// 2.6版本起开始暴露响应方法
Vue.observable = (obj) => {
    observe(obj)
    return obj
}
```

然后添加了一个选项对象，并添加了三个属性，就是我们熟悉的`component`、`directives`、`filters`，并给`components`挂载了`keep-alive`组件：

```js
Vue.options = Object.create(null)
ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
})
Vue.options._base = Vue
// 挂载keep-alive组件
extend(Vue.options.components, builtInComponents)
```

`extend`是一个简单的浅拷贝方法，在`vue`内部的出镜率非常高：

```js
export function extend (to, _from) {
  for (const key in _from) {
    to[key] = _from[key]
  }
  return to
}
```

最后又添加了一堆静态方法：`Vue.use`、`Vue.mixin`、`Vue.extend`、`Vue.component`、`Vue.filter`、`Vue.directive`。

`Vue`构造函数依然不是在该文件定义的，让我们最后再跳到`core/instance/index.js`，在这里终于看到了`Vue`构造函数的庐山真面目：

```js
function Vue (options) {
  this._init(options)
}
```

构造函数很简洁，就调用了一个方法。

接下来是给原型添加一堆方法：

```js
// 添加_init方法
initMixin(Vue)
// 代理data和prop数据、定义$set、$delete、$watch方法 
stateMixin(Vue)
// 添加$on、$once、$off、$emit方法 
eventsMixin(Vue)
// 添加_update、$forceUpdate、$destroy方法
lifecycleMixin(Vue)
// 添加一些创建VNode的快捷方法、$nextTick、_render方法
renderMixin(Vue)
```

到这里，引入`Vue`时`Vue`做的事情就结束了，其实就是创建了一个构造函数，然后在到处扩展原型方法、静态方法、静态属性等等。
