---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

# css过渡

首先看一下这个版本的`vue` `css`过渡和动画的应用方式：

```html
<p class="msg" v-if="show" v-transition="expand">Hello!</p>
<p class="animated" v-if="show" v-transition="bounce">Look at me!</p>
```

```css
.msg {
    transition: all .3s ease;
    height: 30px;
    padding: 10px;
    background-color: #eee;
    overflow: hidden;
}
.msg.expand-enter, .msg.expand-leave {
    height: 0;
    padding: 0 10px;
    opacity: 0;
}

.animated {
    display: inline-block;
}

.animated.bounce-enter {
    animation: bounce-in .5s;
}

.animated.bounce-leave {
    animation: bounce-out .5s;
}

@keyframes bounce-in {
    0% {
        transform: scale(0);
    }

    50% {
        transform: scale(1.5);
    }

    100% {
        transform: scale(1);
    }
}

@keyframes bounce-out {
    0% {
        transform: scale(1);
    }

    50% {
        transform: scale(1.5);
    }

    100% {
        transform: scale(0);
    }
}
```

可以看到也是通过指令的方式，这个版本只有支持两个类，一个是进入的时候添加的`v-enter`，另一个是离开时候添加的`v-leave`。

先看一下这个指令：

```js
module.exports = {
    isLiteral: true,// 为true不会创建watcher实例
    bind: function () {
        this.update(this.expression)
    },
    update: function (id) {
        var vm = this.el.__vue__ || this.vm
        this.el.__v_trans = {
            id: id,
            // 这个版本的vue可以使用transitions选项来定义JavaScript动画
            fns: vm.$options.transitions[id]
        }
    }
}
```

这个指令不会创建`watcher`，因为指令的值要么是`css`的类名，要么是`JavaScript`动画选项的名称，都不需要进行观察。指令绑定时所做的事情就是给`el`元素添加了个自定义属性，保存了表达式的值，这里是`expand`、`JavaScript`动画函数，这里是`undefined`。

要触发动画需要修改`if`指令`show`的值，假设开始是`false`，我们把它改成`true`，这会触发`if`指令的`update`方法，根据第三篇[vue0.11版本源码阅读系列三：指令编译](https://juejin.cn/post/6918313229449953293)最后部分对`if`指令过程的解析我们知道在进入时调用了`transition.blockAppend(frag, this.end, vm)`，在离开时调用了`transition.blockRemove(this.start, this.end, this.vm)`，这里显然会调用`blockAppend`：

```js
// block是包含了if指令绑定元素的代码片段
// target是一个注释节点，在if指令绑定元素所在的位置
exports.blockAppend = function (block, target, vm) {
    // 代码片段的子节点
    var nodes = _.toArray(block.childNodes)
    for (var i = 0, l = nodes.length; i < l; i++) {
        apply(nodes[i], 1, function () {
            _.before(nodes[i], target)
        }, vm)
    }
}
```

遍历元素调用`apply`方法：

```js
var apply = exports.apply = function (el, direction, op, vm, cb) {
    var transData = el.__v_trans
    if (
        !transData ||// 没有过渡数据
        !vm._isCompiled ||// 当前实例没有调用过$mount方法插入到页面
        (vm.$parent && !vm.$parent._isCompiled)// 父组件没有插入到页面
    ) {// 上述情况不需要动画，直接跳过
        op()
        if (cb) cb()
        return
    }
    var jsTransition = transData.fns
    // JavaScript动画，下一小节再看
    if (jsTransition) {
        applyJSTransition(
            el,
            direction,
            op,
            transData,
            jsTransition,
            vm,
            cb
        )
    } else if (
        _.transitionEndEvent &&
        // 页面不可见的话不进行过渡
        !(doc && doc.hidden)
    ) {
        // css
        applyCSSTransition(
            el,
            direction,
            op,
            transData,
            cb
        )
    } else {
        // 不需要应用过渡
        op()
        if (cb) cb()
    }
}
```

这个方法会判断是应用`JavaScript`动画还是`css`动画，并分发给不同的函数处理。函数套娃，又套到了`applyCSSTransition`方法：

```js
module.exports = function (el, direction, op, data, cb) {
    var prefix = data.id || 'v'// 此处是expand
    var enterClass = prefix + '-enter'// expand-enter
    var leaveClass = prefix + '-leave'// expand-leave
    if (direction > 0) { // 进入
        // 给元素添加进入的类名
        addClass(el, enterClass)
        // op就是_.before(nodes[i], target)操作，这一步会把元素添加到页面上
        op()
        push(el, direction, null, enterClass, cb)
    } else { // 离开
        // 给元素添加离开的类名
        addClass(el, leaveClass)
        push(el, direction, op, leaveClass, cb)
    }
}
```

可以看到进入和离开的操作是有区别的，本次我们是把`show`的值改成`true`，所以会走`direction > 0`的分支，先给元素添加进入的类名，然后再把元素实际插入到页面上，最后调用`push`方法；

如果是离开的话会先给元素添加离开的类名，然后调用`push`方法；

看一下`push`方法：

```js
var queue = []
var queued = false
function push (el, dir, op, cls, cb) {
    queue.push({
        el  : el,
        dir : dir,
        cb  : cb,
        cls : cls,
        op  : op
    })
    if (!queued) {
        queued = true
        _.nextTick(flush)
    }
}
```

把本次的任务添加到队列，注册了个异步回调在下一帧执行，关于`nextTick`的详细分析请前往[vue0.11版本源码阅读系列五：批量更新是怎么做的](https://juejin.cn/post/6918316455142686734)。

`addClass`和`op`都是同步任务，会立即执行，如果此刻有多个被这个`if`指令控制的元素都会被依次添加到队列里，结果就是这些元素都会被添加到页面上，但是因为我们给进入的样式设置的是` height: 0;opacity: 0;`，所以是看不见的，这些同步任务执行完后才会去异步队列里把注册的`flush`方法拉出来执行：

```js
function flush () {
  // 这个方法用来触发强制回流，确保我们添加的expand-enter样式能生效，不过我试过不回流也能生效
  var f = document.documentElement.offsetHeight
  queue.forEach(run)
  queue = []
  queued = false
}
```

`flush`方法遍历刚才添加到`queue`里的任务对象调用`run `方法，因为进行了异步批量更新，所以同一时刻有多个元素动画也只会触发一次回流：

```js
function run (job) {
    var el = job.el
    var data = el.__v_trans
    var cls = job.cls
    var cb = job.cb
    var op = job.op
    // getTransitionType方法用来获取是transition过渡还是animation动画，原理是判断元素的style对象或者getComputedStyle()方法获取的样式对象里的transitionDuration或animationDuration属性是否存在以及是否为0s
    var transitionType = getTransitionType(el, data, cls)
    if (job.dir > 0) { // 进入
        if (transitionType === 1) {// transition过渡
            // 因为v-enter的样式是隐藏元素的样式，另外因为给元素设置了transition: all .3s ease，所以只要把这个类删除了自然就会应用过渡效果
            removeClass(el, cls)
            // 存在回调时才需要监听transitionend事件
            if (cb) setupTransitionCb(_.transitionEndEvent)
        } else if (transitionType === 2) {// animation动画
            // animation动画只要添加了v-enter类自行就会触发，需要做的只是监听animationend事件在动画结束后把这个类删除
            setupTransitionCb(_.animationEndEvent, function () {
                removeClass(el, cls)
            })
        } else {
            // 没有过渡
            removeClass(el, cls)
            if (cb) cb()
        }
    } else { // 离开
        // 离开动画很简单，两者都是只要添加了v-leave类就可以触发动画
        // 要做的只是在监听动画结束的事件把元素从页面删除和把类名从元素上删除
        if (transitionType) {
            var event = transitionType === 1
            ? _.transitionEndEvent
            : _.animationEndEvent
            setupTransitionCb(event, function () {
                op()
                removeClass(el, cls)
            })
        } else {
            op()
            removeClass(el, cls)
            if (cb) cb()
        }
    }
}
```

现在看一下当把`show`的值由`true`改成`false`时调用的`blockRemove`方法：

```js
// start和end是两个注释节点，包围了该if指令控制的所有元素
exports.blockRemove = function (start, end, vm) {
    var node = start.nextSibling
    var next
    while (node !== end) {
        next = node.nextSibling
        apply(el, -1, function () {
            _.remove(el)
        }, vm, cb)
        node = next
    }
}
```

遍历元素同样调用`apply`方法，只不过参数传了`-1`代表是离开。

到这里可以总结一下`vue`的`css`过渡：

1.进入

先给元素添加`v-enter`类，然后把元素插入到页面，最后创建一个任务添加到队列，如果有多个元素的话会一次性全部完成，然后在下一帧来执行刚才添加的任务：

1.1css过渡

`v-enter`类名里的样式一般是用来隐藏元素的，比如把元素的宽高设为`0`、透明度设为`0`等等，反正让人看不见就对了，要触发动画需要把这个类名删除了，所以这里的任务就是移除元素的`v-enter`类名，然后浏览器会自己应用过渡效果。

1.2css动画

`animation`不一样，`v-enter`类的样式一般是定义`animation`的属性值，比如：`animation: bounce-out .5s;`，只要添加了这个类名，就会开始动画，所以这里的任务是监听动画结束事件来移除元素的`v-enter`类名。

2.离开

`css`过渡和动画在离开时是一样的，都是给元素添加一个`v-leave`类就可以了，`v-leave`类要设置的样式一般和`v-enter`是一样的，除非进出效果就是要不一样，否则都是要让元素不可见，然后添加一个任务，因为样式上不可见了但元素实际上还是在页面上，所以最后的任务就是监听动画结束事件把元素真正的从页面上移除，当然，相应的`v-leave`类也是要 从元素上移除的。



# JavaScript动画

在这个版本要使用`JavaScript`进行动画过渡需要使用声明过渡选项：

```js
Vue.transition('fade', {
  beforeEnter: function (el) {
    // 元素插入文档之前调用，比如提取把元素变成不可见，否则会有闪屏的问题
  },
  enter: function (el, done) {
    // 元素已经插入到DOM，动画完成后需要手动调用done方法
    $(el)
      .css('opacity', 0)
      .animate({ opacity: 1 }, 1000, done)
    // 返回一个函数当动画取消时被调用
    return function () {
      $(el).stop()
    }
  },
  leave: function (el, done) {
    $(el).animate({ opacity: 0 }, 1000, done)
    return function () {
      $(el).stop()
    }
  }
})
```

就是定义三个钩子函数，定义了`JavaScript`过渡选项，在`transition`指令的`update`方法就能根据表达式获取到，这样就会走到上述`apply`方法里的`jsTransition`分支，调用`applyJSTransition`方法：

```js
module.exports = function (el, direction, op, data, def, vm, cb) {
    if (data.cancel) {
        data.cancel()
        data.cancel = null
    }
    if (direction > 0) { // 进入
        // 调用beforeEnter钩子
        if (def.beforeEnter) {
            def.beforeEnter.call(vm, el)
        }
        op()// 把元素插入到页面dom
        // 调用enter钩子
        if (def.enter) {
            data.cancel = def.enter.call(vm, el, function () {
                data.cancel = null
                if (cb) cb()
            })
        } else if (cb) {
            cb()
        }
    } else { // 离开
        // 调用leave钩子
        if (def.leave) {
            data.cancel = def.leave.call(vm, el, function () {
                data.cancel = null
                // 离开动画结束了从页面移除元素
                op()
                if (cb) cb()
            })
        } else {
            op()
            if (cb) cb()
        }
    }
}
```

跟`css`过渡相比，`JavaScript`过渡很简单，进入过渡就是在元素实际插入到页面前执行以下你的初始化方法，然后把元素插入到页面，接下来调用`enter`钩子随你怎么让元素运动，动画结束后再调一下`vue`注入的方法告诉`vue`动画结束了，离开过渡先调一下你的离开钩子，在你的动画结束后再把元素从页面上删除，逻辑很简单。

