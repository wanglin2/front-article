初始化结束后，如果存在`el`属性，那么最后会进行挂载操作：

```js
if (vm.$options.el) {
    vm.$mount(vm.$options.el)
}
```

`$mount`方法是个区分平台的方法，`web`平台的如下：

```js
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && query(el)
  const options = this.$options
  // 解析 template/el，编译成渲染函数
  if (!options.render) {
    // 分情况获取模板
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      // 编译模板为渲染函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  return mount.call(this, el, hydrating)
}
```

`Vue`的模板是需要编译成渲染函数的，所以`$mount`的主要逻辑就是判断是否存在字符串模板，存在的话才会调用编译方法，否则如果已经是渲染函数了，那么直接调用`mount`方法：

```js
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

接下来先看一下比较简单的`mountComponent`方法。



# 挂载组件mountComponent

```js
export function mountComponent (
  vm,
  el,
  hydrating
) {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
  }
  callHook(vm, 'beforeMount')

  // 更新组件的方法
  let updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }

  // 我们把Watcher实例设置为vm._watcher属性值这个逻辑放在watcher的构造函数中，因为watcher的初始补丁可能调用$forceUpdate（例如，在子组件的挂载钩子中），这需要依赖于vm._watcher先被定义
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

`render`渲染函数其实就是一个返回虚拟`DOM`的函数，不存在的话会默认定义为一个返回空虚拟`DOM`的函数。接下来定义了一个`updateComponent`方法，这个方法是真正的挂载方法，会在初始化和更新的时候调用。接着创建了一个`Watcher`实例，`Watcher`实例化时会执行取值函数，也就是会执行上述的`updateComponent`方法，这个方法很明显先执行渲染函数生成`VNode`，然后通过`diff`算法进行打补丁更新页面，这样组件就渲染完成了，当后续模板中依赖的数据变化了，那么会通知该`watcher`，该`watcher`会重新调用取值函数，也就是会再次调用`updateComponent`方法，达到页面更新。

# 编译渲染函数

接下来看看模板编译为渲染函数的过程，首先声明，这个过程是非常复杂的，所以我们不会太深入细节（因为我看！不！懂！），只会大概看一下流程是怎么样的。

首先执行的是`compileToFunctions`方法：

```js
const { render, staticRenderFns } = compileToFunctions(template, {
    outputSourceRange: process.env.NODE_ENV !== 'production',
    shouldDecodeNewlines,// 是否会编码换行符，IE在属性值内会编码换行符，而其他浏览器不编码
    shouldDecodeNewlinesForHref,// 是否会在href属性中编码换行符
    delimiters: options.delimiters,// 改变纯文本插入分隔符，默认为['{{', '}}']，你可以改成比如['${', '}']
    comments: options.comments// 当设为 true 时，将会保留且渲染模板中的 HTML 注释。默认行为是舍弃它们
}, this)
options.render = render
options.staticRenderFns = staticRenderFns
```

传入了模板和配置项。

```js
import { baseOptions } from './options'
import { createCompiler } from '../../../compiler/index'

const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```

可以看到`compileToFunctions`方法是`createCompiler`函数返回的，它接受一个配置对象：

```js
export const createCompiler = createCompilerCreator(function baseCompile () { })
```

`createCompiler`方法又是执行`createCompilerCreator`函数返回的：

```js
export function createCompilerCreator (baseCompile) {
  return function createCompiler (baseOptions) {
      
    function compile (template, options) {}

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}
```

而真正的`compileToFunctions`又是通过`createCompileToFunctionFn`函数返回的，怎么样，头有没有晕？让我们画图看一下：

![Snipaste_2021-11-05_14-39-50.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba1178fd4bd6463d82b869f0d01f747b~tplv-k3u1fbpfcp-watermark.image?)

`Vue`设计的这么复杂肯定是有原因的，不妨来模拟一下。

## 模拟设计思路

最开始我们只写了一个`complie`方法，接收模板和选项，返回编译结果，结果是一个代码字符串，类似这样的：

```js
with(this){return _c('ul',_l((list),function(item){return _c('li',[_v(_s(item))])}),0)}
```

只生成字符串还不够，我们还需要将它转换成可执行的函数，于是我们又写了一个`createCompileToFunctionFn`方法，它接收`compile`作为参数，返回一个`compileToFunctions`函数。现在是这样的：

```js
const compile = (template, options) => {}
const compileToFunctions = createCompileToFunctionFn(compile)
compileToFunctions(template, options)
```

一切正常，然而有一天我们发现有一些基础选项`baseOptions`也需要传给`compile`函数，而且这些基础选项也不是固定的，可能不同的平台需要传不同的，我们肯定不能写死在`compile`函数里，通过参数传递的话那么每个使用`compile`的地方都得引入这个参数并传给它，显然不行，那么我们可以通过`偏函数`的方式来接收这个参数并返回`compile`：

```js
const createCompiler = (baseOptions) => {
    const compile = (template, options) => {
        console.log(baseOptions)
    }
    return {
        compile,
        compileToFunctions: createCompileToFunctionFn(compile)
    }
}
const { compileToFunctions } = createCompiler(baseOptions)
compileToFunctions(template, options)
```

喜大普奔，可是万万没想到又有一天我们发现核心的编译功能也要区分平台，而且和`baseOptions`还不太一样，同一个编译器可以接收不同的选项，怎么办，一不做二不休，我们给`createCompiler`再包一层：

```js
const createCompilerCreator = (baseCompile) => {
    return const createCompiler = (baseOptions) => {
        const compile = (template, options) => {
            console.log(baseCompile, baseOptions)
        }
        return {
            compile,
            compileToFunctions: createCompileToFunctionFn(compile)
        }
    }
}
const createCompiler = createCompilerCreator(baseCompile)
// 选项1
const { compileToFunctions } = createCompiler(baseOptions)
// 选项2
const { compileToFunctions } = createCompiler(baseOptions2)
compileToFunctions(template, options)
```

终于完美了，既具有扩展性，又消灭了重复，到这里，你应该就理解为什么`Vue`要这么设计了。

## 详解编译流程

`compileToFunctions`函数是`createCompileToFunctionFn`函数生成的，简化后的代码如下：

```js
export function createCompileToFunctionFn (compile) {
  const cache = Object.create(null)

  return function compileToFunctions (
    template,
    options,
    vm
  ) {
    options = extend({}, options

    // 检查缓存
    const key = options.delimiters
      ? String(options.delimiters) + template
      : template
    if (cache[key]) {
      return cache[key]
    }

    // 编译
    const compiled = compile(template, options)

    // 将代码转换为函数
    const res = {}
    res.render = createFunction(compiled.render)
    res.staticRenderFns = compiled.staticRenderFns.map(code => {
      return createFunction(code)
    })

    return (cache[key] = res)
  }
}
```

`Vue`的缓存使用真是无处不在，这个函数的主要功能就是调用`compile`函数进行编译，返回代码字符串，然后通过`createFunction`转换成函数：

```js
function createFunction (code) {
  try {
    return new Function(code)
  } catch (err) {
    return noop
  }
}
```

很简单，就是使用了`new Function`。

接下来看`compile`的实现：

```js
function compile (
 template,
 options
) {
    const finalOptions = Object.create(baseOptions)
    // 错误信息
    const errors = []
    // 提示信息
    const tips = []

    let warn = (msg, range, tip) => {
        (tip ? tips : errors).push(msg)
    }

    if (options) {
        // merge custom modules
        // 合并自定义模块
        if (options.modules) {
            finalOptions.modules =
                (baseOptions.modules || []).concat(options.modules)
        }
        // merge custom directives
        // 合并自定义指令
        if (options.directives) {
            finalOptions.directives = extend(
                Object.create(baseOptions.directives || null),
                options.directives
            )
        }
        // copy other options
        // 复制其他选项
        for (const key in options) {
            if (key !== 'modules' && key !== 'directives') {
                finalOptions[key] = options[key]
            }
        }
    }

    finalOptions.warn = warn
    // 调用基础编译器进行编译
    const compiled = baseCompile(template.trim(), finalOptions)
    compiled.errors = errors
    compiled.tips = tips
    return compiled
}
```

这个函数的实现也很清晰，先处理及合并参数，然后调用`baseCompile`进行编译。

```js
function baseCompile (
  template,
  options
) {
  // 解析成ast
  const ast = parse(template.trim(), options)
  // 优化
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  // 生成代码
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}
```

可以看到`Vue`的模板编译的核心操作就是三步：把模板解析为抽象语法树、优化、生成代码字符串，这和普通的编译操作是一致的。

总结一下，模板编译首先会把模板字符串编译为抽象语法树，然后进行优化，生成代码字符串，最后再将代码字符串转换成可执行的函数，这些函数的产出就是`VNode`。

`parse`的过程太复杂了，完全看不懂，跳过，我们来看看优化的过程：

```js
export function optimize (root, options) {
  if (!root) return
  isStaticKey = genStaticKeys(options.staticKeys || '')
  // 判断是否是保留标签，比如html、svg标签
  isPlatformReservedTag = options.isReservedTag || no
  // 第一遍处理：标记所有非静态节点
  markStatic(root)
  // 第二遍：标记静态根节点
  markStaticRoots(root, false)
}
```

优化的目的是检测出`AST`中的静态子树，这样在每次重新渲染时就可以不用再重新创建新节点，在打补丁的时候也可以完全跳过。

`genStaticKeys`方法会返回一个函数，用来检测某个`key`是否是静态的`key`：

```js
function genStaticKeys (keys) {
  return makeMap(
    'type,tag,attrsList,attrsMap,plain,parent,children,attrs,start,end,rawAttrsMap' +
    (keys ? ',' + keys : '')
  )
}
```

`AST`树`root`的结构大致如下：

![Snipaste_2021-11-05_16-56-47.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4872ddb35c146d49a29df1ee305518e~tplv-k3u1fbpfcp-watermark.image?)

第一次遍历会标记出所有的静态节点：

```js
function markStatic (node) {
  node.static = isStatic(node)
  if (node.type === 1) {
    // 不要将组件插槽内容设置为静态，用来避免：
    //    1.组件无法改变插槽节点
    //    2.静态插槽内容无法进行热更新加载
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== 'slot' &&
      node.attrsMap['inline-template'] == null
    ) {
      return
    }
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      // 有一个子节点不是静态节点，那么该节点就不是静态节点
      if (!child.static) {
        node.static = false
      }
    }
    // 存在v-if
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}
```

静态节点的判断依据还是挺复杂的，直接看代码：

```js
function isStatic (node) {
  if (node.type === 2) { // expression表达式
    return false
  }
  if (node.type === 3) { // text文本
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings 没有动态绑定
    !node.if && !node.for && // 没有 v-if 、 v-for 、 v-else
    !isBuiltInTag(node.tag) && // not a built-in 不是内置标签
    isPlatformReservedTag(node.tag) && // not a component 不是组件
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}

```

第二遍会找出静态根：

```js
function markStaticRoots (node, isInFor) {
  if (node.type === 1) {
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
    // 使节点符合静态根的条件，它应该有不仅仅是静态文本的子级，
    // 否则，花费的成本将超过收益，还不如直接刷新渲染。
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```

可以看到如果一个节点是静态节点、存在子节点，且子节点不能只有一个静态文本，那么就可以把它视为是静态根。这部分优化凑活看吧，反正笔者看的云里雾里。

生成代码`generate`也是一个很复杂的过程，同样也跳过。

将模板编译为渲染函数后，就会调用前面介绍的`mount`方法进行挂载。

# 挂载详解

挂载前面介绍过会通过渲染`watcher`来触发，也就是执行下面这个函数：

```js
let updateComponent = () => {
    vm._update(vm._render(), hydrating)
}
```

很明显会先执行`_render`方法，来看一下：

```js
Vue.prototype._render = function () {
    const vm = this
    const { render, _parentVnode } = vm.$options
    // 存在父VNode
    if (_parentVnode) {
        vm.$scopedSlots = normalizeScopedSlots(
            _parentVnode.data.scopedSlots,
            vm.$slots
        )
    }
    // 设置父vnode。这允许渲染函数访问占位符节点上的数据。
    vm.$vnode = _parentVnode
    // 执行render方法生成虚拟DOM
    let vnode = render.call(vm._renderProxy, vm.$createElement)
    // 如果返回的数组只包含一个节点，请允许它
    if (Array.isArray(vnode) && vnode.length === 1) {
        vnode = vnode[0]
    }
    // 设置父节点
    vnode.parent = _parentVnode
    return vnode
}
```

这个函数的核心就是执行了`render`函数，也就是上一步模板编译生成的，其他一些细节目前还不太好理解，后续如果遇到了对应的场景再说。

生成了`VNode`后会执行`_update`方法：

```js
Vue.prototype._update = function (vnode, hydrating) {
    const vm = this
    const prevEl = vm.$el
    // 旧的vnode
    const prevVnode = vm._vnode
    // 将该vm标记为当前活跃的实例
    const restoreActiveInstance = setActiveInstance(vm)
    // 新生成的vnode
    vm._vnode = vnode
    // Vue.prototype.__patch__ 是在入口处根据所使用的平台来注入的
    if (!prevVnode) {
      // 第一次渲染
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // 后续更新
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // 更新 __vue__ 引用
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // 如果父项是HOC（高阶组件），则也更新其$el
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // 调度程序调用更新的钩子，以确保在父级的更新钩子中更新子级。
  }
```

这个函数的核心是调用了`__patch__`方法来打补丁，也就是第一次渲染时根据`vnode`来生成`dom`，后续通过`vnode`的`diff`操作来更新`dom`，这部分的详细内容后面会单独进行介绍。

总结一下本文的内容，当第一次创建一个`Vue`实例时，如果存在模板，那么会进行编译操作，编译操作核心就是三步：将模板编译为`AST`、优化`AST`、根据`AST`生成代码，最后会编译为渲染函数，然后会给该实例创建一个渲染`watcher`，初始化时会调用更新函数，也就是先调用`render`方法生成`VNode`，然后调用`__patch__`方法来生成实际的`DOM`。

当然，本文只介绍了基本的流程，具体的细节都跳过了，有能力的朋友可以深入了解一下。
