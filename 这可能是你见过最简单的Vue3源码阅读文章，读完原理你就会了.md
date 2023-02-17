`Vue3`官网中有下面这样一张图，基本展现出了`Vue3`的渲染原理：

![render pipeline](https://cn.vuejs.org/assets/render-pipeline.03805016.png)

本文会从源码角度来草率的看一下`Vue3`的运行全流程，旨在加深对上图的理解，从下面这个很简单的使用示例开始：

```js
import { createApp, ref } from "vue";

createApp({
  template: `
        <div class="card">
            <button type="button" @click="count++">count is {{ count }}</button>
        </div>
    `,
  setup() {
    const count = ref(0);
    return {
      count,
    };
  },
}).mount("#app");
```

通过`createApp`方法创建应用实例，传了一个组件的选项对象，包括模板`template`、组合式 `API` 的入口`setup`函数，在`setup`函数里使用`ref`创建了一个响应式数据，然后`return`给模板使用，最后调用实例的`mount`方法将模板渲染到`id`为`app`的元素内。后续只要修改`count`的值页面就会自动刷新，麻雀虽小，但是也代表了`Vue`的核心。

首先调用了`createApp`方法：

```js
const createApp = ((...args) => {
    const app = createRenderer(rendererOptions).createApp(...args);
    return app;
});
```

通过`createRenderer`创建了一个渲染器，`rendererOptions`是一个对象，上面主要是操作`DOM`的方法：

```js
{
    insert: (child, parent, anchor) => {
        parent.insertBefore(child, anchor || null);
    },
    //...
}
```

这么做主要是方便跨平台，比如在其他非浏览器环境，可以替换成对应的节点操作方法。

```js
function createRenderer(options) {
    return baseCreateRenderer(options);
}

function baseCreateRenderer(options, createHydrationFns) {
    // ...
    return {
        render,
        hydrate,
        createApp: createAppAPI(render, hydrate)
    };
}
```

`baseCreateRenderer`方法非常长，包含了渲染器的所有方法，比如`mount`、`patch`等，`createApp`是通过`createAppAPI`方法调用返回的：

```js
function createAppAPI(render, hydrate) {
    return function createApp(rootComponent, rootProps = null) {
        if (!isFunction(rootComponent)) {
            rootComponent = Object.assign({}, rootComponent);
        }
        const context = createAppContext();
        let isMounted = false;
        const app = (context.app = {
            _uid: uid$1++,
            _component: rootComponent,
            _props: rootProps,
            _container: null,
            _context: context,
            _instance: null,
            version,
            get config() {},
            set config() {},
            use(){},
            mixin(){},
            component(){},
            directive(){},
            mount(){},
            unmount(){},
            provide(){}
        });
        return app;
    }
}
```

这个就是最终的`createApp`方法，所谓的应用实例`app`其实就是一个对象，我们传进去的组件选项作为根组件存储在`_component`属性上，另外还可以看到应用实例提供的一些方法，比如注册插件的`use`方法，挂载实例的`mount`方法等。

`context`其实也是一个普通对象：

```js
function createAppContext() {
    return {
        app: null,
        config: {
            isNativeTag: NO,
            performance: false,
            globalProperties: {},
            optionMergeStrategies: {},
            errorHandler: undefined,
            warnHandler: undefined,
            compilerOptions: {}
        },
        mixins: [],
        components: {},
        directives: {},
        provides: Object.create(null),
        optionsCache: new WeakMap(),
        propsCache: new WeakMap(),
        emitsCache: new WeakMap()
    };
}
```

这个上下文对象会保存在应用实例和根`VNode`上，可能是后续渲染时会用到。

接下来看一下创建实例后挂载的`mount`方法：

```js
mount(rootContainer, isHydrate, isSVG) {
    // 没有挂载过
    if (!isMounted) {
        // 创建虚拟DOM
        const vnode = createVNode(rootComponent, rootProps);
        vnode.appContext = context;
        // 渲染
        render(vnode, rootContainer, isSVG);
        isMounted = true;
        // 实例和容器元素互相关联
        app._container = rootContainer;
        rootContainer.__vue_app__ = app;
        // 返回根组件的实例
        return getExposeProxy(vnode.component) || vnode.component.proxy;
    }
}
```

主要就是做了两件事，创建虚拟`DOM`，然后渲染。

`createVNode`方法：

```js
const createVNode = _createVNode;
function _createVNode(type, props = null, children = null, patchFlag = 0, dynamicProps = null, isBlockNode = false) {
    const shapeFlag = isString(type)
        ? 1 /* ShapeFlags.ELEMENT */
        : isSuspense(type)
            ? 128 /* ShapeFlags.SUSPENSE */
            : isTeleport(type)
                ? 64 /* ShapeFlags.TELEPORT */
                : isObject(type)
                    ? 4 /* ShapeFlags.STATEFUL_COMPONENT */
                    : isFunction(type)
                        ? 2 /* ShapeFlags.FUNCTIONAL_COMPONENT */
                        : 0;
    return createBaseVNode(type, props, children, patchFlag, dynamicProps, shapeFlag, isBlockNode, true);
}
```

`createVNode`方法会根据组件的类型生成一个标志，后续会通过这个标志做一些优化之类的处理。我们传的是一个组件选项，也就是一个普通对象，`shapeFlag`的值为`4`。

然后调用了`createBaseVNode`方法：

```js
function createBaseVNode(type, props = null, children = null, patchFlag = 0, dynamicProps = null, shapeFlag = type === Fragment ? 0 : 1 /* ShapeFlags.ELEMENT */, isBlockNode = false, needFullChildrenNormalization = false) {
    const vnode = {
        __v_isVNode: true,
        __v_skip: true,
        type,
        props,
        key: props && normalizeKey(props),
        ref: props && normalizeRef(props),
        scopeId: currentScopeId,
        slotScopeIds: null,
        children,
        component: null,
        suspense: null,
        ssContent: null,
        ssFallback: null,
        dirs: null,
        transition: null,
        el: null,
        anchor: null,
        target: null,
        targetAnchor: null,
        staticCount: 0,
        shapeFlag,
        patchFlag,
        dynamicProps,
        dynamicChildren: null,
        appContext: null,
        ctx: currentRenderingInstance
    };
    return vnode;
}
```

可以看到返回的虚拟`DOM`也是一个普通对象，我们传进去的组件选项会存储在`type`属性上。

虚拟`DOM`创建完后就会调用`render`方法将虚拟`DOM`渲染为实际的`DOM`节点，`render`方法是通过参数传给`createAppAPI`的：

```js
const render = (vnode, container, isSVG) => {
    if (vnode == null) {
        // 卸载
        if (container._vnode) {
            unmount(container._vnode, null, null, true);
        }
    }
    else {
        // 首次渲染或者更新
        patch(container._vnode || null, vnode, container, null, null, null, isSVG);
    }
    flushPreFlushCbs();
    flushPostFlushCbs();
    container._vnode = vnode;
};
```

如果要渲染的新`VNode`不存在，那么从容器元素上获取之前`VNode`进行卸载，否则调用`patch`方法进行打补丁，如果是首次渲染，`container._vnode`不存在，那么直接将新`VNode`渲染为`DOM`元素即可，否则会对比新旧`VNode`，使用`diff`算法进行打补丁，`Vue2`中使用的是双端`diff`算法，`Vue3`中使用的是快速`diff`算法。

打补丁结束后清空了两个回调队列，可以看到事件队列还分为前后两个，那么我们常用的`nextTick`方法注册的回调在哪个队列呢，实际上，两个都不在：

```js
const resolvedPromise = Promise.resolve();
let currentFlushPromise = null;

function nextTick(fn) {
    const p = currentFlushPromise || resolvedPromise;
    return fn ? p.then(this ? fn.bind(this) : fn) : p;
}
```

`Promise.resolve()`方法会创建一个`Resolved`状态的`Promise`对象。

`nextTick`方法就是这么简单，如果`currentFlushPromise`有值，那么使用这个`Promise`注册回调，否则使用默认的`resolvedPromise`将回调放到微任务队列。

`currentFlushPromise`会在调用`queueFlush`方法时赋值，也就是生成一个新的`Promise`对象：

```js
function queueFlush() {
    if (!isFlushing && !isFlushPending) {
        isFlushPending = true;
        currentFlushPromise = resolvedPromise.then(flushJobs);
    }
}
```

`flushJobs`和前面的`flushPreFlushCbs`方法里冲刷的都是`queue`队列，而`flushPostFlushCbs`方法里冲刷的是`pendingPostFlushCbs`队列，`flushJobs`方法在冲刷完`queue`队列后才会冲刷`pendingPostFlushCbs`队列。而如果是冲刷中调用`nextTick`添加的回调会在这两个队列都清空后才会执行。

扯远了，回到`render`方法，接下来看看`render`方法里调用的`patch`方法：

```js
const patch = (n1, n2, container, anchor = null, parentComponent = null, parentSuspense = null, isSVG = false, slotScopeIds = null, optimized = (process.env.NODE_ENV !== 'production') && isHmrUpdating ? false : !!n2.dynamicChildren) => {
    	// 新旧VNode相同直接返回
        if (n1 === n2) {
            return;
        }
    	// 如果新旧VNode的类型不同，那么也不需要打补丁了，直接卸载旧的，挂载新的
        if (n1 && !isSameVNodeType(n1, n2)) {
            anchor = getNextHostNode(n1);
            unmount(n1, parentComponent, parentSuspense, true);
            n1 = null;
        }
        const { type, ref, shapeFlag } = n2;
        switch (type) {
              case Text:
                // ...
                break;
              // ...
              default:
                // ...
                else if (shapeFlag & 6 /* ShapeFlags.COMPONENT */) {
                    processComponent(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized);
                }
                // ...
        }
}
```

`patch`方法就是用来打补丁更新实际`DOM`的，`switch`里面根据`VNode`的类型不同做的处理也不同，因为我们的例子传的是一个组件选项对象，所以会走`processComponent`处理分支：

```js
const processComponent = (n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized) => {
    // 如果旧的VNode不存在，那么调用挂载方法
    if (n1 == null) {
        mountComponent(n2, container, anchor, parentComponent, parentSuspense, isSVG, optimized);
    }
    // 新旧都存在，那么进行更新操作
    else {
        updateComponent(n1, n2, optimized);
    }
};
```

根据是否存在旧的`VNode`判断是调用挂载方法还是更新方法，先看`mountComponent`方法：

```js
const mountComponent = (initialVNode, container, anchor, parentComponent, parentSuspense, isSVG, optimized) => {
    const instance = (initialVNode.component = createComponentInstance(initialVNode, parentComponent, parentSuspense));
    setupComponent(instance);
    setupRenderEffect(instance, initialVNode, container, anchor, parentSuspense, isSVG, optimized);
}
```

首先调用`createComponentInstance`方法创建组件实例，返回的其实也是一个普通对象：

```js
function createComponentInstance(vnode, parent, suspense) {
    const type = vnode.type;
    const appContext = (parent ? parent.appContext : vnode.appContext) || emptyAppContext;
    const instance = {
        uid: uid++,
        vnode,
        type,
        parent,
        appContext,
        // 还有非常多属性
        // ...
    }
    return instance;
}
```

然后调用了`setupComponent`方法：

```js
function setupComponent(instance, isSSR = false) {
    const { props, children } = instance.vnode;
    const isStateful = instance.vnode.shapeFlag & 4;
    initProps(instance, props, isStateful, isSSR);
    initSlots(instance, children);
    const setupResult = isStateful
        ? setupStatefulComponent(instance, isSSR)
        : undefined;
    return setupResult;
}
```

初始化`props`和`slots`，然后如果`shapeFlag`为`4`会调用`setupStatefulComponent`方法，前面说了我们传的组件选项对应的`shapeFlag`就是`4`，所以会走`setupStatefulComponent`方法：

```js
function setupStatefulComponent(instance, isSSR) {
    const { setup } = Component;
    if (setup) {
        const setupResult = callWithErrorHandling(setup, instance, 0, [instance.props, setupContext]);
        handleSetupResult(instance, setupResult, isSSR);
    }
}
```

在这个方法里会调用组件选项的`setup`方法，这个函数中返回的对象会暴露给模板和组件实例，看一下`handleSetupResult`方法：

```js
function handleSetupResult(instance, setupResult, isSSR) {
    if (isFunction(setupResult)) {
        instance.render = setupResult;
    } else if (isObject(setupResult)) {
        instance.setupState = proxyRefs(setupResult);
    }
    finishComponentSetup(instance, isSSR);
}
```

如果`setup`返回的是一个函数，那么这个函数会直接被作为渲染函数。否则如果返回的是一个对象，会使用`proxyRefs`将这个对象转为`Proxy`代理的响应式对象。

最后又调用了`finishComponentSetup`方法：

```js
function finishComponentSetup(instance, isSSR) {
    const Component = instance.type;
    if (!instance.render) {
        if (!isSSR && compile && !Component.render) {
            const template = Component.template ||
                  resolveMergedOptions(instance).template;
            if (template) {
                const { isCustomElement, compilerOptions } = instance.appContext.config;
                const { delimiters, compilerOptions: componentCompilerOptions } = Component;
                const finalCompilerOptions = extend(extend({
                    isCustomElement,
                    delimiters
                }, compilerOptions), componentCompilerOptions);
                Component.render = compile(template, finalCompilerOptions);
            }
        }
        instance.render = (Component.render || NOOP);
    }
}
```

这个函数主要是判断组件是否存在渲染函数`render`，如果不存在则判断是否存在`template`选项，我们传的组件选项显然是没有`render`属性，而是传的模板`template`，所以会使用`compile`方法来将模板编译成渲染函数。

回到`mountComponent`方法，最后调用了`setupRenderEffect`，这个方法很重要：

```js
const setupRenderEffect = (instance, initialVNode, container, anchor, parentSuspense, isSVG, optimized) => {
    // 组件更新方法
    const componentUpdateFn = () => {}
    // 创建一个effect
    const effect = (instance.effect = new ReactiveEffect(componentUpdateFn, () => queueJob(update), instance.scope));
    // 调用effect的run方法执行componentUpdateFn方法
    const update = (instance.update = () => effect.run());
    update();
}
```

这一步就涉及到`Vue3`的响应式原理了，核心就是使用`Proxy`拦截数据，然后在属性读取时将属性和读取该属性的函数（称为副作用函数）关联起来，然后在更新该属性时取出该属性关联的副作用函数出来执行，详细的内容网上已经有非常多的文章了，有兴趣的可以自己搜一搜，或者直接看源码也是可以的。

简化后的`ReactiveEffect`类就是这样的：

```js
let activeEffect;
class ReactiveEffect {
    constructor(fn, scheduler = null, scope) {
        this.fn = fn;
    }
    run() {
        activeEffect = this;
        try {
            return this.fn();
        } finally {
            activeEffect = null
        }
    } 
}
```

执行它的`run`方法时会把自身赋值给全局的`activeEffect`变量，然后执行副作用函数时如果读取了`Proxy`代理后的对象的某个属性时就会将对象、属性和这个`ReactiveEffect`示例关联存储起来，如果属性发生改变，会取出关联的`ReactiveEffect`实例，执行它的`run`方法，达到自动更新的目的。

我们使用的是`ref`方法创建的数据，`ref`方法返回的响应式数据虽然不是通过`Proxy`代理的，但是读取修改操作同样是会被拦截的，和`Proxy`代理的数据拦截时做的事情是一样的。

接下来看看传给它的组件更新方法`componentUpdateFn`：

```js
const componentUpdateFn = () => {
    // 组件没有挂载过
    if (!instance.isMounted) {
        const subTree = (instance.subTree = renderComponentRoot(instance));
        patch(null, subTree, container, anchor, instance, parentSuspense, isSVG);
        initialVNode.el = subTree.el;
        instance.isMounted = true;
    } else {// 组件已经挂载过
        const nextTree = renderComponentRoot(instance);
        patch(prevTree, nextTree, hostParentNode(prevTree.el), getNextHostNode(prevTree), instance, parentSuspense, isSVG);
        next.el = nextTree.el;
    }
}
```

组件无论是首次挂载，还是更新，做的事情核心是一样的，先调用`renderComponentRoot`方法生成组件模板的虚拟`DOM`，然后调用`patch`方法打补丁。

```js
function renderComponentRoot(instance) {
    const { type: Component, vnode, proxy, withProxy, props, propsOptions: [propsOptions], slots, attrs, emit, render, renderCache, data, setupState, ctx, inheritAttrs } = instance;
    let result = render.call(proxyToUse, proxyToUse, renderCache, props, setupState, data, ctx)
    return result
}
```

`renderComponentRoot`核心就是调用组件的渲染函数`render`方法生成组件模板的虚拟`DOM`，然后扔给`patch`方法更新就好了。

看完了`mountComponent`方法，再来看看`updateComponent`方法：

```js
const updateComponent = (n1, n2, optimized) => {
    const instance = (n2.component = n1.component);
    if (shouldUpdateComponent(n1, n2, optimized)) {
        // 需要更新
        instance.next = n2;
        instance.update();
    }else {
        // 不需要更新
        n2.el = n1.el;
        instance.vnode = n2;
    }
}
```

先调用`shouldUpdateComponent`方法判断组件是否需要更新，大致是通过是否存在过渡效果、是否存在动态`slots`、`props`是否发生改变、子节点是否发改变等来判断。

如果需要更新，那么会执行`instance.update`方法，这个方法就是前面`setupRenderEffect`方法里保存的`effect.run`方法，所以最终执行的也是`componentUpdateFn`方法。

到这里，从我们创建实例到页面渲染，再到更新的全流程就讲完了，总结一下，大致就是:

1.每个`Vue`组件都需要产出一份虚拟`DOM`，也就是组件的`render`函数的返回值，`render`函数你可以直接手写，也可以通过`template`传递模板字符串，由`Vue`内部来编译成渲染函数，平常我们开发时写的`Vue`单文件，最终也会编译成普通的`Vue`组件选项对象；

2.`render`函数会作为副作用函数执行，也就是如果在模板中使用到了响应式数据（所谓响应式数据就是能拦截到它的各种读取、修改操作），那么响应式数据和属性会与`render`函数关联起来，那么当响应式数据被修改以后，就能找到依赖它的`render`函数，那么就可以通知依赖的组件进行更新；

2.有了虚拟`DOM`之后，`Vue`内部的渲染器就能将它渲染成真实的`DOM`，如果是更新的情况，也就是存在新旧两个虚拟`DOM`，那么`Vue`会通过比较，必要时会使用`diff`算法进行高效的更新真实`DOM`；

所以只要你实现一个渲染器，能将虚拟`DOM`渲染成真实`DOM`，并且能高效的根据新旧虚拟`DOM`对比完成更新，再实现一个编译器，能将模板编译成渲染函数，最后再基于`Proxy`实现一个响应系统就可以实现一个`Vue3`了，是不是很简单，心动不如行动，下一个框架等你来创造！





