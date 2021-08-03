---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

在第三篇[vue0.11版本源码阅读系列三：指令编译](https://juejin.cn/post/6918313229449953293)里我们知道如果某个属性的值变化了，会调用依赖该属性的`watcher`的`update`方法：

```js
p.update = function () {
  if (!config.async || config.debug) {
    this.run()
  } else {
    batcher.push(this)
  }
}

```

它没有直接调用指令的`update`方法，而是交给了`batcher`，本篇来看一下这个`batcher`做了什么。

顾名思义，`batcher`是批量的意思，所以就是批量更新，为什么要批量更新呢，先看一下下面的情况：

```html
<div v-if="show">我出来了</div>
<div v-if="show && true">我也是</div>
```

```js
window.vm.show = true
window.vm.show = false
```

比如有两个指令依赖同一个属性或者连续修改某个属性，如果不进行批量异步更新，那么就会多次修改dom，这显然是没必要的，看下面两个动图能更直观的感受到：

没有进行批量异步更新的时候：

![2021-01-12-17-01-46](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5221291cd6074b14bb4d29df8c8c4af0~tplv-k3u1fbpfcp-zoom-1.image)

进行了批量异步更新：

![2021-01-12-17-02-21](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e1c4eba1be04aebbc7092f2f40b3ec6~tplv-k3u1fbpfcp-zoom-1.image)

能清晰的发现通过异步更新能跳过中间不必要的渲染以达到优化性能的效果。

接下来看一下具体实现，首先是`push`函数：

```js
// 定义了两个队列，一个用来存放用户的watcher，一个用来存放指令更新的watcher
var queue = []
var userQueue = []
var has = {}
var waiting = false
var flushing = false
exports.push = function (job) {
  // job就是watcher实例
  var id = job.id
  // 在没有flushing的情况下has[id]用来跳过同一个watcher的重复添加
  if (!id || !has[id] || flushing) {
    has[id] = 1
    // 首先要说明的是通过$watch方法或者watch选项生成的watcher代表是用户的，user属性为true
    // 这里注释说在执行任务中用户的watcher可能会触发非user的指令更新，所以要立即更新这个被触发的指令，否则flushing这个变量是不需要的
    if (flushing && !job.user) {
      job.run()
      return
    }
    // 根据指令的类型添加到不同的队列里
    ;(job.user ? userQueue : queue).push(job)
    // 上个队列未被清空前不会创建新队列
    if (!waiting) {
      waiting = true
      _.nextTick(flush)
    }
  }
}
```

`push`方法做的事情是把`watcher`添加到队列`quene`里，然后如果没有扔过`flush`给`nextTick`或者上次扔给`nextTick`的`flush`方法已经被执行了，就再给它一个。

`flush`方法用来遍历队列里的`watcher`并调用其`run`方法，`run`方法最终会调用指令的`update`方法来更新页面。

```js
function flush () {
  flushing = true
  run(queue)
  run(userQueue)
  // 清空队列和复位变量
  reset()
}
function run (queue) {
  // 循环执行watcher实例的run方法，run方法里会遍历该watcher实例的指令队列并执行指令的update方法
  for (var i = 0; i < queue.length; i++) {
    queue[i].run()
  }
}
```

接下来就是`nextTick`方法了：

```js
exports.nextTick = (function () {
  var callbacks = []
  var pending = false
  var timerFunc
  function handle () {
    pending = false
    var copies = callbacks.slice(0)
    callbacks = []
    for (var i = 0; i < copies.length; i++) {
      copies[i]()
    }
  }
  // 支持MutationObserver接口的话使用MutationObserver
  if (typeof MutationObserver !== 'undefined') {
    var counter = 1
    var observer = new MutationObserver(handle)
    var textNode = document.createTextNode(counter)
    observer.observe(textNode, {
      characterData: true// 设为 true 表示监视指定目标节点或子节点树中节点所包含的字符数据的变化
    })
    timerFunc = function () {
      counter = (counter + 1) % 2// counter会在0和1两者循环变化
      textNode.data = counter// 节点变化会触发回调handle，
    }
  } else {// 否则使用定时器
    timerFunc = setTimeout
  }
  return function (cb, ctx) {
    var func = ctx
      ? function () { cb.call(ctx) }
      : cb
    callbacks.push(func)
    if (pending) return
    pending = true
    timerFunc(handle, 0)
  }
})()
```

这是个自执行函数，一般用来定义并保存一些局部变量，返回了一个函数，就是`nextTick`方法本法了，`flush`方法会被`push`到`callbacks`数组里，我们常用的方法`this.$nextTick(() => {xxxx})`也会把回调添加到这个数组里，这里也有一个变量`pending`来控制重复添加的问题，最后添加到事件循环的队列里的是`handle`方法。

批量很容易理解，都放到一个队列里，最后一起执行就是批量执行了，但是要理解`MutationObserver`的回调或者`setTimeout`的回调为什么能异步调用就需要先来了解一下`JavaScript`语言里的事件循环`Event Loop`的原理了。

简单的说就是因为`JavaScript`是单线程的，所以任务需要排队进行执行，前一个执行完了才能执行后面一个，但有些任务比较耗时而且没必要等着，所以可以先放一边，先执行后面的，等到了可以执行了再去执行它，比如有些`IO`操作，像常见的鼠标键盘事件注册、`Ajax`请求、`settimeout`定时器、`Promise`回调等。所以会存在两个队列，一个是同步队列，也就是主线程，另一个是异步队列，刚才提到的那些事件的回调如果可以被执行了都会被放在异步队列里，当主线程上的任务执行完毕后会把异步队列的任务取过来进行执行，所以同步代码总是在异步代码之前执行，执行完了后又会去检查异步队列，这样不断循环就是`Event Loop`。

但是异步任务里其实还是分两种，一种叫宏任务，常见的为：`setTimeout`、`setInterval`，另一种叫微任务，常见的如：`Promise`、`MutationObserver`。微任务会在宏任务之前执行，即使宏任务的回调先被添加到队列里。

现在可以来分析一下异步更新的原理，就以开头提到的例子来说：

```html
<div v-if="show">我出来了</div>
<div v-if="show && true">我也是</div>
```

```js
window.vm.show = true
window.vm.show = false
```

因为有两个指令都依赖了`show`，表达式不一样，所以会有两个`watcher`，这两个`watcher`都会被`show`属性的`dep`收集，所以每修改一次`show`的值都会触发这两个`watcher`的更新，也就是会调两次`batcher.push(this)`方法，第一次调用后会执行`_.nextTick(flush)`注册一个回调，连续两次修改`show`的值，会调用四次上述提到的`batcher.push(this)`方法，因为重复添加的被过滤掉了，所以最后会有两个`watcher`被添加到队列里，以上这些操作都是同步任务，所以是连续被执行完的，等这些同步任务都被执行完了后就会把刚才注册的回调`handle`拿过来执行，也就是会一次性执行刚才添加的两个`watcher`：

![image-20210112200127418](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33c9ded98bf94c1f97eebbb45521a99e~tplv-k3u1fbpfcp-zoom-1.image)

以上就是`vue`异步更新的全部内容。

