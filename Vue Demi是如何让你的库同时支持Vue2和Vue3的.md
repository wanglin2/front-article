# Vue Demi是什么

如果你想开发一个同时支持`Vue2`和`Vue3`的库可能想到以下两种方式：

1.创建两个分支，分别支持`Vue2`和`Vue3`

2.只使用`Vue2`和`Vue3`都支持的`API`

这两种方式都有缺点，第一种很麻烦，第二种无法使用`Vue3`新增的组合式 `API`，其实现在`Vue2.7+`版本已经内置支持组合式`API`，`Vue2.6`及之前的版本也可以使用[@vue/composition-api](https://github.com/vuejs/composition-api)插件来支持，所以完全可以只写一套代码同时支持`Vue2`和`3`。虽然如此，但是实际开发中，同一个`API`在不同的版本中可能导入的来源不一样，比如`ref`方法，在`Vue2.7+`中直接从`vue`中导入，但是在`Vue2.6-`中只能从`@vue/composition-api`中导入，那么必然会涉及到版本判断，[Vue Demi](https://github.com/vueuse/vue-demi)就是用来解决这个问题，使用很简单，只要从`Vue Demi`中导出你需要的内容即可：

```js
import { ref, reactive, defineComponent } from 'vue-demi'
```

`Vue-demi`会根据你的项目判断到底使用哪个版本的`Vue`，具体来说，它的策略如下：

- `<=2.6`: 从`Vue`和`@vue/composition-api`中导出
- `2.7`: 从`Vue`中导出（组合式`API`内置于`Vue 2.7`中）
- `>=3.0`: 从`Vue`中导出，并且还`polyfill`了两个`Vue 2`版本的`set`和`del` `API`

接下来从源码角度来看一下它具体是如何实现的。

# 基本原理

当我们使用`npm i vue-demi`在我们的项目里安装完以后，它会自动执行一个脚本：

```json
{
    "scripts": {
        "postinstall": "node ./scripts/postinstall.js"
    }
}
```

```js
// postinstall.js
const { switchVersion, loadModule } = require('./utils')

const Vue = loadModule('vue')

if (!Vue || typeof Vue.version !== 'string') {
  console.warn('[vue-demi] Vue is not found. Please run "npm install vue" to install.')
}
else if (Vue.version.startsWith('2.7.')) {
  switchVersion(2.7)
}
else if (Vue.version.startsWith('2.')) {
  switchVersion(2)
}
else if (Vue.version.startsWith('3.')) {
  switchVersion(3)
}
else {
  console.warn(`[vue-demi] Vue version v${Vue.version} is not suppported.`)
}
```

导入我们项目里安装的`vue`，然后根据不同的版本分别调用`switchVersion`方法。

先看一下`loadModule`方法：

```js
function loadModule(name) {
  try {
    return require(name)
  } catch (e) {
    return undefined
  }
}
```

很简单，就是包装了一下`require`，防止报错阻塞代码。

然后看一下`switchVersion`方法：

```js
function switchVersion(version, vue) {
  copy('index.cjs', version, vue)
  copy('index.mjs', version, vue)
  copy('index.d.ts', version, vue)

  if (version === 2)
    updateVue2API()
}
```

执行了`copy`方法，从函数名可以大概知道是复制文件，三个文件的类型也很清晰，分别是`commonjs`版本的文件、`ESM`版本的文件、`TS`类型定义文件。

另外还针对`Vue2.6`及一下版本执行了`updateVue2API`方法。

`updateVue2API`方法我们后面再看，先看一下`copy`方法：

```js
const dir = path.resolve(__dirname, '..', 'lib')

function copy(name, version, vue) {
  vue = vue || 'vue'
  const src = path.join(dir, `v${version}`, name)
  const dest = path.join(dir, name)
  let content = fs.readFileSync(src, 'utf-8')
  content = content.replace(/'vue'/g, `'${vue}'`)
  try {
    fs.unlinkSync(dest)
  } catch (error) { }
  fs.writeFileSync(dest, content, 'utf-8')
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db20b6428e954f63aa80024e7214b112~tplv-k3u1fbpfcp-zoom-1.image)


其实就是从不同版本的目录里复制上述三个文件到外层目录，其中还支持替换`vue`的名称，这当你给`vue`设置了别名时需要用到。

到这里，`Vue Demi`安装完后自动执行的事情就做完了，其实就是根据用户项目中安装的`Vue`版本，分别从三个对应的目录中复制文件作为`Vue Demi`包的入口文件，`Vue Demi`支持三种模块语法：

```json
{
    "main": "lib/index.cjs",
    "jsdelivr": "lib/index.iife.js",
    "unpkg": "lib/index.iife.js",
    "module": "lib/index.mjs",
    "types": "lib/index.d.ts"
}
```

默认入口为`commonjs`模块`cjs`文件，支持`ESM`的可以使用`mjs`文件，同时还提供了可以直接在浏览器上使用的`iife`类型的文件。

接下来看一下分别针对三种版本的`Vue`具体都做了什么。

# v2版本

`Vue2.6`版本只有一个默认导出：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/650f47b947fe4c799f86c0c20fb88a93~tplv-k3u1fbpfcp-zoom-1.image)


我们只看`mjs`文件，`cjs`有兴趣的可以自行阅读：

```js
import Vue from 'vue'
import VueCompositionAPI from '@vue/composition-api/dist/vue-composition-api.mjs'

function install(_vue) {
  _vue = _vue || Vue
  if (_vue && !_vue['__composition_api_installed__'])
    _vue.use(VueCompositionAPI)
}

install(Vue)
// ...
```

导入`Vue`和`VueCompositionAPI`插件，并且自动调用`Vue.use`方法安装插件。

继续：

```js
// ...
var isVue2 = true
var isVue3 = false
var Vue2 = Vue
var version = Vue.version

export {
    isVue2,
    isVue3,
    Vue,
    Vue2,
    version,
    install,
}

/**VCA-EXPORTS**/
export * from '@vue/composition-api/dist/vue-composition-api.mjs'
/**VCA-EXPORTS**/
```

首先导出了两个变量`isVue2`和`isVue3`，方便我们的库代码判断环境。

然后在导出`Vue`的同时，还通过`Vue2`的名称再导出了一遍，这是为啥呢，其实是因为`Vue2`的`API`都是挂载在`Vue`对象上，比如我要进行一些全局配置，那么只能这么操作：

```js
import { Vue, isVue2 } from 'vue-demi'

if (isVue2) {
  Vue.config.xxx
}
```

这样在`Vue2`的环境中没有啥问题，但是当我们的库处于`Vue3`的环境中时，其实是不需要导入`Vue`对象的，因为用不上，但是构建工具不知道，所以它会把`Vue3`的所有代码都打包进去，但是`Vue3`中很多我们没有用到的内容是不需要的，但是因为我们导入了包含所有`API`的`Vue`对象，所以无法进行去除，所以针对`Vue2`版本单独导出一个`Vue2`对象，我们就可以这么做：

```js
import { Vue2 } from 'vue-demi'

if (Vue2) {
  Vue2.config.xxx
}
```

然后后续你会看到在`Vue3`的导出中`Vue2`是`undefined`，这样就可以解决这个问题了。

接着导出了`Vue`的版本和`install`方法，意味着你可以手动安装`VueCompositionAPI`插件。

然后是导出`VueCompositionAPI`插件提供的`API`，也就是组合式`API`，但是可以看到前后有两行注释，还记得前面提到的`switchVersion`方法里针对`Vue2`版本还执行了`updateVue2API`方法，现在来看一看它做了什么事情：

```js
function updateVue2API() {
  const ignoreList = ['version', 'default']
  // 检查是否安装了composition-api
  const VCA = loadModule('@vue/composition-api')
  if (!VCA) {
    console.warn('[vue-demi] Composition API plugin is not found. Please run "npm install @vue/composition-api" to install.')
    return
  }
  // 获取除了version、default之外的其他所有导出
  const exports = Object.keys(VCA).filter(i => !ignoreList.includes(i))
  // 读取ESM语法的入口文件
  const esmPath = path.join(dir, 'index.mjs')
  let content = fs.readFileSync(esmPath, 'utf-8')
  // 将export * 替换成 export { xxx }的形式
  content = content.replace(
    /\/\*\*VCA-EXPORTS\*\*\/[\s\S]+\/\*\*VCA-EXPORTS\*\*\//m,
`/**VCA-EXPORTS**/
export { ${exports.join(', ')} } from '@vue/composition-api/dist/vue-composition-api.mjs'
/**VCA-EXPORTS**/`
    )
  // 重新写入文件
  fs.writeFileSync(esmPath, content, 'utf-8')
}
```

主要做的事情就是检查是否安装了`@vue/composition-api`，然后过滤出了`@vue/composition-api`除了`version`和`default`之外的所有导出内容，最后将：

```js
export * from '@vue/composition-api/dist/vue-composition-api.mjs'
```

的形式改写成：

```js
export { EffectScope, ... } from '@vue/composition-api/dist/vue-composition-api.mjs'
```

为什么要过滤掉`version`和`default`呢，`version`是因为已经导出了`Vue`的`version`了，所以会冲突，本来也不需要，`default`即默认导出，`@vue/composition-api`的默认导出其实是一个包含它的`install`方法的对象，前面也看到了，可以默认导入`@vue/composition-api`，然后通过`Vue.use`来安装，这个其实也不需要从`Vue Demi`导出，不然像下面这样就显得很奇怪：

```js
import VueCompositionAPI from 'vue-demi'
```

到这里，就导出所有内容了，然后我们就可以从`vue-demi`中导入各种需要的内容了，比如：

```js
import { isVue2, Vue, ref, reactive, defineComponent } from 'vue-demi'
```

# v2.7版本

接下来看一下是如何处理`Vue2.7`版本的导出的，和`Vue2.6`之前的版本相比，`Vue2.7`直接内置了`@vue/composition-api`，所以除了默认导出`Vue`对象外还导出了组合式`API`：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b27d17002bcb452894381d30241ba3d8~tplv-k3u1fbpfcp-zoom-1.image)


```js
import Vue from 'vue'

var isVue2 = true
var isVue3 = false
var Vue2 = Vue
var warn = Vue.util.warn

function install() {}

export { Vue, Vue2, isVue2, isVue3, install, warn }
// ...
```

和`v2`相比，导出的内容是差不多的，因为不需要安装`@vue/composition-api`，所以`install`是个空函数，区别在于还导出了一个`warn`方法，这个文档里没有提到，但是可以从过往的[issues](https://github.com/vueuse/vue-demi/issues?q=warn)中找到原因，大致就是`Vue3`导出了一个`warn`方法，而`Vue2`的`warn`方法在`Vue.util`对象上，所以为了统一手动导出，为什么`V2`版本不手动导出一个呢，原因很简单，因为这个方法在`@vue/composition-api`的导出里有。

继续：

```js
// ...
export * from 'vue'
// ...
```

导出上图中`Vue`所有的导出，包括`version`、组合式`API`，但是要注意这种写法不会导出默认的`Vue`，所以如果你像下面这样使用默认导入是获取不到`Vue`对象的：

```js
import Vue from 'vue-demi'
```

继续：

```js
// ...
// createApp polyfill
export function createApp(rootComponent, rootProps) {
  var vm
  var provide = {}
  var app = {
    config: Vue.config,
    use: Vue.use.bind(Vue),
    mixin: Vue.mixin.bind(Vue),
    component: Vue.component.bind(Vue),
    provide: function (key, value) {
      provide[key] = value
      return this
    },
    directive: function (name, dir) {
      if (dir) {
        Vue.directive(name, dir)
        return app
      } else {
        return Vue.directive(name)
      }
    },
    mount: function (el, hydrating) {
      if (!vm) {
        vm = new Vue(Object.assign({ propsData: rootProps }, rootComponent, { provide: Object.assign(provide, rootComponent.provide) }))
        vm.$mount(el, hydrating)
        return vm
      } else {
        return vm
      }
    },
    unmount: function () {
      if (vm) {
        vm.$destroy()
        vm = undefined
      }
    },
  }
  return app
}
```

和`Vue2`的`new Vue`创建`Vue`实例不一样，`Vue3`是通过`createApp`方法，`@vue/composition-api`插件`polyfill`了这个方法，所以针对`Vue2.7`，`Vue Demi`手动进行了`polyfill`。

到这里，针对`Vue2.7`所做的事情就结束了。

# v3版本

`Vue3`相比之前的版本，最大区别是不再提供一个单独的`Vue`导出：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2fce033bafb4b848c44e451a11c7f5b~tplv-k3u1fbpfcp-zoom-1.image)


```js
import * as Vue from 'vue'

var isVue2 = false
var isVue3 = true
var Vue2 = undefined

function install() {}

export {
  Vue,
  Vue2,
  isVue2,
  isVue3,
  install,
}
// ...
```

因为默认不导出`Vue`对象了，所以通过整体导入`import * as Vue`的方式把所有的导出都加载到`Vue`对象上，然后也可以看到导出的`Vue2`为`undefined`，`install`同样是个空函数。

继续：

```js
// ...
export * from 'vue'
// ...
```

没啥好说的，直接导出`Vue`的所有导出内容。

继续：

```js
// ...
export function set(target, key, val) {
  if (Array.isArray(target)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  target[key] = val
  return val
}

export function del(target, key) {
  if (Array.isArray(target)) {
    target.splice(key, 1)
    return
  }
  delete target[key]
}
```

最后`polyfill`了两个方法，这两个方法实际上是`@vue/composition-api`插件提供的，因为`@vue/composition-api`提供的响应性`API`实现上并没有使用`Proxy`代理，仍旧是基于`Vue2`的响应系统来实现的，所以`Vue2`中响应系统的限制仍旧还是存在的，所以需要提供两个类似`Vue.set`和`Vue.delete`方法用来给响应性数据添加或删除属性。

