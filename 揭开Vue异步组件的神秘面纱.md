# 简介

在大型应用里，有些组件可能一开始并不显示，只有在特定条件下才会渲染，那么这种情况下该组件的资源其实不需要一开始就加载，完全可以在需要的时候再去请求，这也可以减少页面首次加载的资源体积，要在`Vue`中使用异步组件也很简单：

```vue
// AsyncComponent.vue
<template>
  <div>我是异步组件的内容</div>
</template>

<script>
export default {
    name: 'AsyncComponent'
}
</script>
```

```vue
// App.vue
<template>
  <div id="app">
    <AsyncComponent v-if="show"></AsyncComponent>
    <button @click="load">加载</button>
  </div>
</template>

<script>
export default {
  name: 'App',
  components: {
    AsyncComponent: () => import('./AsyncComponent'),
  },
  data() {
    return {
      show: false,
    }
  },
  methods: {
    load() {
      this.show = true
    },
  },
}
</script>
```

我们没有直接引入`AsyncComponent`组件进行注册，而是使用`import()`方法来动态的加载，`import()`是[ES2015 Loader 规范](https://whatwg.github.io/loader/) 定义的一个方法，`webpack`内置支持，会把`AsyncComponent`组件的内容单独打成一个`js`文件，页面初始不会加载，点击加载按钮后才会去请求，该方法会返回一个`promise`，接下来，我们从源码角度详细看看这一过程。

通过本文，你可以了解`Vue`对于异步组件的处理过程以及`webpack`的资源加载过程。

# 编译产物

首先我们打个包，生成了三个`js`文件：

![image-20211214194854431.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edefb0ffe5344409a2c58230fab5c6a2~tplv-k3u1fbpfcp-watermark.image?)

第一个文件是我们应用的入口文件，里面包含了`main.js`、`App.vue`的内容，另外还包含了一些`webpack`注入的方法，第二个文件就是我们的异步组件`AsyncComponent`的内容，第三个文件是其他一些公共库的内容，比如`Vue`。

然后我们看看`App.vue`编译后的内容：

![image-20211224161447196.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c367b0786604afc984da33216f2e0fd~tplv-k3u1fbpfcp-watermark.image?)

上图为`App`组件的选项对象，可以看到异步组件的注册方式，是一个函数。

![image-20211224161252075.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/355789b0d5124cf7aafe3bd6a52cb058~tplv-k3u1fbpfcp-watermark.image?)

上图是`App.vue`模板部分编译后的渲染函数，当`_vm.show`为`true`的时候，会执行`_c('AsyncComponent')`，否则执行`_vm._e()`，创建一个空的`VNode`，`_c`即`createElement`方法：

```js
vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
```

接下来看看当我们点击按钮后，这个方法的执行过程。

# createElement方法

```js
function createElement (
  context,
  tag,
  data,
  children,
  normalizationType,
  alwaysNormalize
) {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children;
    children = data;
    data = undefined;
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE;
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```

`context`为`App`组件实例，`tag`就是`_c`的参数`AsyncComponent`，其他几个参数都为`undefined`或`false`，所以这个方法的两个`if`分支都没走，直接进入`_createElement`方法：

```js
function _createElement (
 context,
 tag,
 data,
 children,
 normalizationType
) {
    // 如果data是被观察过的数据
    if (isDef(data) && isDef((data).__ob__)) {
        return createEmptyVNode()
    }
    // v-bind中的对象语法
    if (isDef(data) && isDef(data.is)) {
        tag = data.is;
    }
    // tag不存在，可能是component组件的:is属性未设置
    if (!tag) {
        return createEmptyVNode()
    }
    // 支持单个函数项作为默认作用域插槽
    if (Array.isArray(children) &&
        typeof children[0] === 'function'
       ) {
        data = data || {};
        data.scopedSlots = { default: children[0] };
        children.length = 0;
    }
    // 处理子节点
    if (normalizationType === ALWAYS_NORMALIZE) {
        children = normalizeChildren(children);
    } else if (normalizationType === SIMPLE_NORMALIZE) {
        children = simpleNormalizeChildren(children);
    }
    // ...
}
```

上述逻辑在我们的示例中都不会进入，接着往下看：

```js
function _createElement (
 context,
 tag,
 data,
 children,
 normalizationType
) {
    // ...
    var vnode, ns;
    // tag是字符串
    if (typeof tag === 'string') {
        var Ctor;
        ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag);
        if (config.isReservedTag(tag)) {
            // 是否是保留元素，比如html元素或svg元素
            if (false) {}
            vnode = new VNode(
                config.parsePlatformTagName(tag), data, children,
                undefined, undefined, context
            );
        } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
            // 组件
            vnode = createComponent(Ctor, data, context, children, tag);
        } else {
            // 其他未知标签
            vnode = new VNode(
                tag, data, children,
                undefined, undefined, context
            );
        }
    } else {
        // tag是组件选项或构造函数
        vnode = createComponent(tag, data, context, children);
    }
    // ...
}
```

对于我们的异步组件，`tag`为`AsyncComponent`，是个字符串，另外通过`resolveAsset`方法能找到我们注册的`AsyncComponent`组件：

```js
function resolveAsset (
  options,// App组件实例的$options
  type,// components
  id,
  warnMissing
) {
  if (typeof id !== 'string') {
    return
  }
  var assets = options[type];
  // 首先检查本地注册
  if (hasOwn(assets, id)) { return assets[id] }
  var camelizedId = camelize(id);
  if (hasOwn(assets, camelizedId)) { return assets[camelizedId] }
  var PascalCaseId = capitalize(camelizedId);
  if (hasOwn(assets, PascalCaseId)) { return assets[PascalCaseId] }
  // 本地没有，则在原型链上查找
  var res = assets[id] || assets[camelizedId] || assets[PascalCaseId];
  if (false) {}
  return res
}
```

`Vue`会把我们的每个组件都先创建成一个构造函数，然后再进行实例化，在创建过程中会进行选项合并，也就是把该组件的选项和父构造函数的选项进行合并：

![image-20211227112643613.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91e3059cd5d9411887e482fe8b3881ce~tplv-k3u1fbpfcp-watermark.image?)

上图中，子选项是`App`的组件选项，父选项是`Vue`构造函数的选项对象，对于`components`选项，会以父类的该选项值为原型创建一个对象，然后把子类本身的选项值作为属性添加到该对象上，最后这个对象作为子类构造函数的`options.components`的属性值：

![image-20211227113823227.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7abad93250cf4d02817585cf9e46e53e~tplv-k3u1fbpfcp-watermark.image?)

![image-20211227113909991.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0114cce57ed843ccbe51f2a69cd350e8~tplv-k3u1fbpfcp-watermark.image?)

![image-20211227113657329.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2811b631b35d4fcf9a3d516c52d52884~tplv-k3u1fbpfcp-watermark.image?)

然后在组件实例化时，会以构造函数的`options`对象作为原型创建一个对象，作为实例的`$options`：

![image-20211227135444816.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f910896573d4433aa3fcbab9e1dbb95c~tplv-k3u1fbpfcp-watermark.image?)

所以`App`实例能通过`$options`从它的构造函数的`options.components`对象上找到`AsyncComponent`组件：

![image-20211227140124998.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aae4646109b14e26b934e7903e42aeb4~tplv-k3u1fbpfcp-watermark.image?)

可以发现就是我们前面看到过的编译后的函数。

接下来会执行`createComponent`方法：

```js
function createComponent (
 Ctor,
 data,
 context,
 children,
 tag
) {
    // ...
    // 异步组件
    var asyncFactory;
    if (isUndef(Ctor.cid)) {
        asyncFactory = Ctor;
        Ctor = resolveAsyncComponent(asyncFactory, baseCtor);
        if (Ctor === undefined) {
            return createAsyncPlaceholder(
                asyncFactory,
                data,
                context,
                children,
                tag
            )
        }
    }
    // ...
}
```

接着又执行了`resolveAsyncComponent`方法：

```js
function resolveAsyncComponent (
 factory,
 baseCtor
) {
     // ...
    var owner = currentRenderingInstance;
    if (owner && !isDef(factory.owners)) {
        var owners = factory.owners = [owner];
        var sync = true;
        var timerLoading = null;
        var timerTimeout = null

        ;(owner).$on('hook:destroyed', function () { return remove(owners, owner); });
        var forceRender = function(){}
        var resolve = once(function(){})
        var reject = once(function(){})
        // 执行异步组件的函数
        var res = factory(resolve, reject);
    }
     // ...
}
```

到这里终于执行了异步组件的函数，也就是下面这个：

```js
function AsyncComponent() {
    return __webpack_require__.e( /*! import() */ "chunk-1f79b58b").then(__webpack_require__.bind(null, /*! ./AsyncComponent */ "c61d"));
}
```

欲知`res`是什么，我们就得看看这几个`webpack`的函数是干什么的。

# 加载组件资源

## __webpack_require__.e方法

先看`__webpack_require__.e`方法：

```js
__webpack_require__.e = function requireEnsure(chunkId) {
    var promises = [];
    // 已经加载的chunk
    var installedChunkData = installedChunks[chunkId];
    if (installedChunkData !== 0) { // 0代表已经加载
      // 值非0即代表组件正在加载中，installedChunkData[2]为promise对象
      if (installedChunkData) {
        promises.push(installedChunkData[2]);
      } else {
        // 创建一个promise，并且把两个回调参数缓存到installedChunks对象上
        var promise = new Promise(function (resolve, reject) {
          installedChunkData = installedChunks[chunkId] = [resolve, reject];
        });
        // 把promise对象本身也添加到缓存数组里
        promises.push(installedChunkData[2] = promise);
        // 开始发起chunk请求
        var script = document.createElement('script');
        var onScriptComplete;
        script.charset = 'utf-8';
        script.timeout = 120;
        // 拼接chunk的请求url
        script.src = jsonpScriptSrc(chunkId);
        var error = new Error();
        // chunk加载完成/失败的回到
        onScriptComplete = function (event) {
          script.onerror = script.onload = null;
          clearTimeout(timeout);
          var chunk = installedChunks[chunkId];
          if (chunk !== 0) {
            // 如果installedChunks对象上该chunkId的值还存在则代表加载出错了
            if (chunk) {
              var errorType = event && (event.type === 'load' ? 'missing' : event.type);
              var realSrc = event && event.target && event.target.src;
              error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')';
              error.name = 'ChunkLoadError';
              error.type = errorType;
              error.request = realSrc;
              chunk[1](error);
            }
            installedChunks[chunkId] = undefined;
          } 
        };
        // 设置超时时间
        var timeout = setTimeout(function () {
          onScriptComplete({
            type: 'timeout',
            target: script
          });
        }, 120000);
        script.onerror = script.onload = onScriptComplete;
        document.head.appendChild(script);
      }
    }
    return Promise.all(promises);
  };
```

这个方法虽然有点长，但是逻辑很简单，首先函数返回的是一个`promise`，如果要加载的`chunk`未加载过，那么就创建一个`promise`，然后缓存到`installedChunks`对象上，接下来创建`script`标签来加载`chunk`，唯一不好理解的是`onScriptComplete`函数，因为在这里面判断该`chunk`在`installedChunks`上的缓存信息不为`0`则当做失败处理了，问题是前面才把`promise`信息缓存过去，也没有看到哪里有进行修改，要理解这个就需要看看我们要加载的`chunk`的内容了：

![image-20211227153327294.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03b228c8fe0e45518ce82abf014508fe~tplv-k3u1fbpfcp-watermark.image?)

可以看到代码直接执行了，并往`webpackJsonp`数组里添加了一项：

```js
window["webpackJsonp"] = window["webpackJsonp"] || []).push([["chunk-1f79b58b"],{..}])
```

看着似乎也没啥问题，其实`window["webpackJsonp"]`的`push`方法被修改过了：

```js
var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
jsonpArray.push = webpackJsonpCallback;
var parentJsonpFunction = oldJsonpFunction;
```

被修改成了`webpackJsonpCallback`方法：

```js
function webpackJsonpCallback(data) {
    var chunkIds = data[0];
    var moreModules = data[1];
    var moduleId, chunkId, i = 0,
        resolves = [];
    for (; i < chunkIds.length; i++) {
        chunkId = chunkIds[i];
        if (Object.prototype.hasOwnProperty.call(installedChunks, chunkId) && installedChunks[chunkId]) {
            // 把该chunk的promise的resolve回调方法添加到resolves数组里
            resolves.push(installedChunks[chunkId][0]);
        }
        // 标记该chunk已经加载完成
        installedChunks[chunkId] = 0;
    }
    // 将该chunk的module数据添加到modules对象上
    for (moduleId in moreModules) {
        if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
            modules[moduleId] = moreModules[moduleId];
        }
    }
    // 执行原本的push方法
    if (parentJsonpFunction) parentJsonpFunction(data);
    // 执行resolve函数
    while (resolves.length) {
        resolves.shift()();
    }
}
```

这个函数会取出该`chunk`加载的`promise`的`resolve`函数，然后将它在`installedChunks`上的信息标记为`0`，代表加载成功，所以在后面执行的`onScriptComplete`函数就可以通过是否为`0`来判断是否加载失败。最后会执行`resolve`函数，这样前面`__webpack_require__.e`函数返回的`promise`状态就会变为成功。

让我们再回顾一下`AsyncComponent`组件的函数：

```js
function AsyncComponent() {
    return __webpack_require__.e( /*! import() */ "chunk-1f79b58b").then(__webpack_require__.bind(null, /*! ./AsyncComponent */ "c61d"));
}
```

`chunk`加载完成后会执行`__webpack_require__`方法。

## `__webpack_require__`方法

这个方法是`webpack`最重要的方法，用来加载模块：

```js
function __webpack_require__(moduleId) {
    // 检查模块是否已经加载过了
    if (installedModules[moduleId]) {
        return installedModules[moduleId].exports;
    }
    // 创建一个新模块，并缓存
    var module = installedModules[moduleId] = {
        i: moduleId,
        l: false,
        exports: {}
    };
    // 执行模块函数
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    // 标记模块加载状态
    module.l = true;
    // 返回模块的导出
    return module.exports;
}
```

所以上面的`__webpack_require__.bind(null, /*! ./AsyncComponent */ "c61d")`其实是去加载了`c61d`模块，这个模块就在我们刚刚请求回来的`chunk`里：

![image-20211227161841023.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2368fc689d54b48a4030594538f2dc8~tplv-k3u1fbpfcp-watermark.image?)

这个模块内部又会去加载它依赖的模块，最终返回的结果为：

![image-20211227162447114.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ddd10f5efce426cbaccc4051351483c~tplv-k3u1fbpfcp-watermark.image?)

其实就是`AsyncComponent`的组件选项。

# 回到createElement方法

回到前面的`resolveAsyncComponent`方法：

```js
var res = factory(resolve, reject);
```

现在我们知道这个`res`其实就是一个未完成的`promise`，`Vue`并没有等待异步组件加载完成，而是继续向后执行：

```js
if (isObject(res)) {
    if (isPromise(res)) {
        // () => Promise
        if (isUndef(factory.resolved)) {
            res.then(resolve, reject);
        }
    }
}

return factory.resolved
```

把定义的`resolve`和`reject`函数作为参数传给`promise` `res`，最后返回了`factory.resolved`，这个属性并没有被设置任何值，所以是`undefined`。

接下来回到`createComponent`方法：

```js
Ctor = resolveAsyncComponent(asyncFactory, baseCtor);
if (Ctor === undefined) {
    // 返回异步组件的占位符节点，该节点呈现为注释节点，但保留该节点的所有原始信息。
    // 这些信息将用于异步服务端渲染。
    return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
    )
}
```

因为`Ctor`是`undefined`，所以会执行`createAsyncPlaceholder`方法返回一个占位符节点：

```js
function createAsyncPlaceholder (
  factory,
  data,
  context,
  children,
  tag
) {
  // 创建一个空的VNode，其实就是注释节点
  var node = createEmptyVNode();
  // 保留组件的相关信息
  node.asyncFactory = factory;
  node.asyncMeta = { data: data, context: context, children: children, tag: tag };
  return node
}
```

最后让我们再回到`_createElement`方法：

```js
// ...
vnode = createComponent(Ctor, data, context, children, tag);
// ...
return vnode
```

很简单，对于异步节点，直接返回创建的注释节点，最后把虚拟节点转换成真实节点，会实际创建一个注释节点：

![image-20211227181319356.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aba98508cb4a4bdb8d0bf8ab68ee4dd3~tplv-k3u1fbpfcp-watermark.image?)

现在让我们来看看`resolveAsyncComponent`函数里面定义的`resolve`，也就是当`chunk`加载完成后会执行的：

```js
var resolve = once(function (res) {d
    // 缓存结果
    factory.resolved = ensureCtor(res, baseCtor);
    // 非同步解析时调用
    // (SSR会把异步解析为同步)
    if (!sync) {
        forceRender(true);
    } else {
        owners.length = 0;
    }
});
```

`res`即`AsyncComponent`的组件选项，`baseCtor`为`Vue`构造函数，会把它们作为参数调用`ensureCtor`方法：

```js
function ensureCtor (comp, base) {
  if (
    comp.__esModule ||
    (hasSymbol && comp[Symbol.toStringTag] === 'Module')
  ) {
    comp = comp.default;
  }
  return isObject(comp)
    ? base.extend(comp)
    : comp
}
```

可以看到实际上是调用了`extend`方法：

![image-20211227182323558.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/206b33157dc848708eee7ec2fae630f8~tplv-k3u1fbpfcp-watermark.image?)

前面也提到过，`Vue`会把我们的组件都创建一个对应的构造函数，就是通过这个方法，这个方法会以`baseCtor`为父类创建一个子类，这里就会创建`AsyncComponent`子类：

![image-20211227182849384.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d112693235941cb92abc873320f9c5c~tplv-k3u1fbpfcp-watermark.image?)

子类创建成功后会执行`forceRender`方法：

```js
var forceRender = function (renderCompleted) {
    for (var i = 0, l = owners.length; i < l; i++) {
        (owners[i]).$forceUpdate();
    }

    if (renderCompleted) {
        owners.length = 0;
        if (timerLoading !== null) {
            clearTimeout(timerLoading);
            timerLoading = null;
        }
        if (timerTimeout !== null) {
            clearTimeout(timerTimeout);
            timerTimeout = null;
        }
    }
};
```

`owners`里包含着`App`组件实例，所以会调用它的`$forceUpdate`方法，这个方法会迫使 `Vue` 实例重新渲染，也就是重新执行渲染函数，进行虚拟`DOM`的`diff`和`path`更新。

所以会重新执行`App`组件的渲染函数，那么又会执行前面的`createElement`方法，又会走一遍我们前面提到的那些过程，只是此时`AsyncComponent`组件已经加载成功并创建了对应的构造函数，所以对于`createComponent`方法，这次执行`resolveAsyncComponent`方法的结果不再是`undefined`，而是`AsyncComponent`组件的构造函数：

```js
Ctor = resolveAsyncComponent(asyncFactory, baseCtor);

function resolveAsyncComponent (
 factory,
 baseCtor
) {
    if (isDef(factory.resolved)) {
        return factory.resolved
    }
}
```

接下来就会走正常的组件渲染逻辑：

```js
var name = Ctor.options.name || tag;
var vnode = new VNode(
    ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
    data, undefined, undefined, undefined, context,
    { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
    asyncFactory
);

return vnode
```

可以看到对于组件其实也是创建了一个`VNode`，具体怎么把该组件的`VNode`渲染成真实`DOM`不是本文的重点就不介绍了，大致就是在虚拟`DOM`的`diff`和`patch`过程中如果遇到的`VNode`是组件类型，那么会`new`一个该组件的实例关联到`VNode`上，组件实例化和我们`new Vue()`没有什么区别，都会先进行选项合并、初始化生命周期、初始化事件、数据观察等操作，然后执行该组件的渲染函数，生成该组件的`VNode`，最后进行`patch`操作，生成实际的`DOM`节点，子组件的这些操作全部完成后才会再回到父组件的`diff`和`patch`过程，因为子组件的`DOM`已经创建好了，所以插入即可，更详细的过程有兴趣可自行了解。

以上就是本文全部内容。

