去年写了一篇文章[手写一个虚拟DOM库，彻底让你理解diff算法](https://juejin.cn/post/6984939221681176607)介绍虚拟`DOM`的`patch`过程和`diff`算法过程，当时使用的是双端`diff`算法，今年看到了`Vue3`使用的已经是快速`diff`算法，所以也想写一篇来记录一下，但是肯定已经有人写过了，所以就在想能不能有点不一样的，上次的文章主要是通过画图来一步步展示`diff`算法的每一种情况和过程，所以就在想能不能改成动画的形式，于是就有了这篇文章。当然目前的实现还是基于双端`diff`算法的，后续会补充上快速`diff`算法。

传送门：[双端Diff算法动画演示](https://wanglin2.github.io/VNode_visualization_demo/)。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a68b3f19661e4b9e8eb335f94c20e90e~tplv-k3u1fbpfcp-zoom-1.image)


界面就是这样的，左侧可以输入要比较的新旧`VNode`列表，然后点击启动按钮就会以动画的形式来展示从头到尾的过程，右侧是水平的三个列表，分别代表的是新旧的`VNode`列表，以及当前的真实`DOM`列表，`DOM`列表初始和旧的`VNode`列表一致，算法结束后会和新的`VNode`列表一致。

需要说明的是这个动画只包含`diff`算法的过程，不包含`patch`过程。

先来回顾一下双端`diff`算法的函数：

```js
const diff = (el, oldChildren, newChildren) => {
  // 指针
  let oldStartIdx = 0
  let oldEndIdx = oldChildren.length - 1
  let newStartIdx = 0
  let newEndIdx = newChildren.length - 1
  // 节点
  let oldStartVNode = oldChildren[oldStartIdx]
  let oldEndVNode = oldChildren[oldEndIdx]
  let newStartVNode = newChildren[newStartIdx]
  let newEndVNode = newChildren[newEndIdx]
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (oldStartVNode === null) {
      oldStartVNode = oldChildren[++oldStartIdx]
    } else if (oldEndVNode === null) {
      oldEndVNode = oldChildren[--oldEndIdx]
    } else if (newStartVNode === null) {
      newStartVNode = oldChildren[++newStartIdx]
    } else if (newEndVNode === null) {
      newEndVNode = oldChildren[--newEndIdx]
    } else if (isSameNode(oldStartVNode, newStartVNode)) { // 头-头
      patchVNode(oldStartVNode, newStartVNode)
      // 更新指针
      oldStartVNode = oldChildren[++oldStartIdx]
      newStartVNode = newChildren[++newStartIdx]
    } else if (isSameNode(oldStartVNode, newEndVNode)) { // 头-尾
      patchVNode(oldStartVNode, newEndVNode)
      // 把oldStartVNode节点移动到最后
      el.insertBefore(oldStartVNode.el, oldEndVNode.el.nextSibling)
      // 更新指针
      oldStartVNode = oldChildren[++oldStartIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else if (isSameNode(oldEndVNode, newStartVNode)) { // 尾-头
      patchVNode(oldEndVNode, newStartVNode)
      // 把oldEndVNode节点移动到oldStartVNode前
      el.insertBefore(oldEndVNode.el, oldStartVNode.el)
      // 更新指针
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    } else if (isSameNode(oldEndVNode, newEndVNode)) { // 尾-尾
      patchVNode(oldEndVNode, newEndVNode)
      // 更新指针
      oldEndVNode = oldChildren[--oldEndIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else {
      let findIndex = findSameNode(oldChildren, newStartVNode)
      // newStartVNode在旧列表里不存在，那么是新节点，创建插入
      if (findIndex === -1) {
        el.insertBefore(createEl(newStartVNode), oldStartVNode.el)
      } else { // 在旧列表里存在，那么进行patch，并且移动到oldStartVNode前
        let oldVNode = oldChildren[findIndex]
        patchVNode(oldVNode, newStartVNode)
        el.insertBefore(oldVNode.el, oldStartVNode.el)
        // 将该VNode置为空
        oldChildren[findIndex] = null
      }
      newStartVNode = newChildren[++newStartIdx]
    }
  }
  // 旧列表里存在新列表里没有的节点，需要删除
  if (oldStartIdx <= oldEndIdx) {
    for (let i = oldStartIdx; i <= oldEndIdx; i++) {
      removeEvent(oldChildren[i])
      oldChildren[i] && el.removeChild(oldChildren[i].el)
    }
  } else if (newStartIdx <= newEndIdx) {
    let before = newChildren[newEndIdx + 1] ? newChildren[newEndIdx + 1].el : null
    for (let i = newStartIdx; i <= newEndIdx; i++) {
      el.insertBefore(createEl(newChildren[i]), before)
    }
  }
}
```

该函数具体的实现步骤可以参考之前的文章，本文就不再赘述。

我们想让这个`diff`过程动起来，首先要找到动画的对象都有哪些，从函数的参数开始看，首先`oldChildren`和 `newChildren`两个`VNode`列表是必不可少的，可以通过两个水平的列表表示，然后是四个指针，这是双端`diff`算法的关键，我们通过四个箭头来表示，指向当前所比较的节点，然后就开启循环了，循环中新旧`VNode`列表其实基本上是没啥变化的，我们实际操作的是`VNode`对应的真实`DOM`元素，包括`patch`打补丁、移动、删除、新增等等操作，所以我们再来个水平的列表表示当前的真实`DOM`列表，最开始肯定是和旧的`VNode`列表是对应的，通过`diff`算法一步步会变成和新的`VNode`列表对应。

再来回顾一下创建`VNode`对象的`h`函数：

```js
export const h = (tag, data = {}, children) => {
  let text = ''
  let el
  let key
  // 文本节点
  if (typeof children === 'string' || typeof children === 'number') {
    text = children
    children = undefined
  } else if (!Array.isArray(children)) {
    children = undefined
  }
  if (data && data.key) {
    key = data.key
  }
  return {
    tag, // 元素标签
    children, // 子元素
    text, // 文本节点的文本
    el, // 真实dom
    key,
    data
  }
}
```

我们输入的`VNode`列表数据会使用`h`函数来创建成`VNode`对象，所以可以输入的最简单的结构如下：

```json
[
  {
    tag: 'div',
    children: '文本节点的内容',
    data: {
      key: 'a'
    }
  }
]
```

输入的新旧`VNode`列表数据会保存在`store`中，可以通过如下方式获取到：

```js
// 输入的旧VNode列表
store.oldVNode
// 输入的新VNode列表
store.newVNode
```

接下来定义相关的变量：

```js
// 指针列表
const oldPointerList = ref([])
const newPointerList = ref([])
// 真实DOM节点列表
const actNodeList = ref([])
// 新旧节点列表
const oldVNodeList = ref([])
const newVNodeList = ref([])
// 提示信息
const info = ref('')
```

指针的移动动画可以使用`css`的`transition`属性来实现，只要修改指针元素的`left`值即可，真实`DOM`列表的移动动画可以使用`Vue`的列表过渡组件[TransitionGroup](https://cn.vuejs.org/guide/built-ins/transition-group.html)来轻松实现，模板如下：

```html
<div class="playground">
  <!-- 指针 -->
  <div class="pointer">
    <div
         class="pointerItem"
         v-for="item in oldPointerList"
         :key="item.name"
         :style="{ left: item.value * 120 + 'px' }"
         >
      <div class="pointerItemName">{{ item.name }}</div>
      <div class="pointerItemValue">{{ item.value }}</div>
      <img src="../assets/箭头_向下.svg" alt="" />
    </div>
  </div>
  <div class="nodeListBox">
    <!-- 旧节点列表 -->
    <div class="nodeList">
      <div class="name" v-if="oldVNodeList.length > 0">旧的VNode列表</div>
      <div class="nodes">
        <TransitionGroup name="list">
          <div
               class="nodeWrap"
               v-for="(item, index) in oldVNodeList"
               :key="item ? item.data.key : index"
               >
            <div class="node">{{ item ? item.children : '空' }}</div>
          </div>
        </TransitionGroup>
      </div>
    </div>
    <!-- 新节点列表 -->
    <div class="nodeList">
      <div class="name" v-if="newVNodeList.length > 0">新的VNode列表</div>
      <div class="nodes">
        <TransitionGroup name="list">
          <div
               class="nodeWrap"
               v-for="(item, index) in newVNodeList"
               :key="item.data.key"
               >
            <div class="node">{{ item.children }}</div>
          </div>
        </TransitionGroup>
      </div>
    </div>
    <!-- 提示信息 -->
    <div class="info">{{ info }}</div>
  </div>
  <!-- 指针 -->
  <div class="pointer">
    <div
         class="pointerItem"
         v-for="item in newPointerList"
         :key="item.name"
         :style="{ left: item.value * 120 + 'px' }"
         >
      <img src="../assets/箭头_向上.svg" alt="" />
      <div class="pointerItemValue">{{ item.value }}</div>
      <div class="pointerItemName">{{ item.name }}</div>
    </div>
  </div>
  <!-- 真实DOM列表 -->
  <div class="nodeList act" v-if="actNodeList.length > 0">
    <div class="name">真实DOM列表</div>
    <div class="nodes">
      <TransitionGroup name="list">
        <div
             class="nodeWrap"
             v-for="item in actNodeList"
             :key="item.data.key"
             >
          <div class="node">{{ item.children }}</div>
        </div>
      </TransitionGroup>
    </div>
  </div>
</div>
```

双端`diff`算法过程中是不会修改新的`VNode`列表的，但是旧的`VNode`列表是有可能被修改的，也就是当首尾比较没有找到可以复用的节点，但是通过直接在旧的`VNode`列表中搜索找到了，那么会移动该`VNode`对应的真实`DOM`，移动后该`VNode`其实就相当于已经被处理过了，但是该`VNode`的位置又是在当前指针的中间，不能直接被删除，所以只好置为空`null`，所以可以看到模板中有处理这种情况。

另外我们还创建了一个`info`元素用来展示提示的文字信息，作为动画的描述。

但是这样还是不够的，因为每个旧的`VNode`是有对应的真实`DOM`元素的，但是我们输入的只是一个普通的`json`数据，所以模板还需要新增一个列表，作为旧的`VNode`列表的关联节点，这个列表只要提供节点引用即可，不需要可见，所以把它的`display`设为`none`：

```js
// 根据输入的旧VNode列表创建元素
const _oldVNodeList = computed(() => {
  return JSON.parse(store.oldVNode)
})
// 引用DOM元素
const oldNode = ref(null)
const oldNodeList = ref([])
```

```html
<!-- 隐藏 -->
<div class="hide">
  <div class="nodes" ref="oldNode">
    <div
         v-for="(item, index) in _oldVNodeList"
         :key="index"
         ref="oldNodeList"
         >
      {{ item.children }}
    </div>
  </div>
</div>
```

然后当我们点击启动按钮，就可以给我们的三个列表变量赋值了，并使用`h`函数创建新旧`VNode`对象，然后传递给打补丁的`patch`函数就可以开始进行比较更新实际的`DOM`元素了：

```js
const start = () => {
  nextTick(() => {
    // 表示当前真实的DOM列表
    actNodeList.value = JSON.parse(store.oldVNode)
    // 表示旧的VNode列表
    oldVNodeList.value = JSON.parse(store.oldVNode)
    // 表示新的VNode列表
    newVNodeList.value = JSON.parse(store.newVNode)
    nextTick(() => {
      let oldVNode = h(
        'div',
        { key: 1 },
        JSON.parse(store.oldVNode).map((item, index) => {
          // 创建VNode对象
          let vnode = h(item.tag, item.data, item.children)
          // 关联真实的DOM元素
          vnode.el = oldNodeList.value[index]
          return vnode
        })
      )
      // 列表的父节点也需要关联真实的DOM元素
      oldVNode.el = oldNode.value
      let newVNode = h(
        'div',
        { key: 1 },
        JSON.parse(store.newVNode).map(item => {
          return h(item.tag, item.data, item.children)
        })
      )
      // 调用patch函数进行打补丁
      patch(oldVNode, newVNode)
    })
  })
}
```

可以看到我们输入的新旧`VNode`列表是作为一个节点的子节点的，这是因为只有当比较的两个节点都存在非文本节点的子节点时才需要使用`diff`算法来高效的更新他们的子节点，当`patch`函数运行完后你可以打开控制台查看隐藏的`DOM`列表，会发现是和新的`VNode`列表保持一致的，那么你可能要问，为什么不直接用这个列表来作为真实`DOM`列表呢，还要自己额外创建一个`actNodeList`列表，其实是可以，但是`diff`算法过程中是使用`insertBefore`等方法来移动真实`DOM`节点的，所以不好加过渡动画，只会看到节点瞬间换位置，不符合我们的动画需求。

到这里效果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/990954f0df5a4826916ddeb90edc4bee~tplv-k3u1fbpfcp-zoom-1.image)


接下来我们先把指针搞出来，我们创建一个处理函数对象，这个对象上会挂载一些方法，用于在`diff`算法过程中调用，在函数中更新相应的变量。

```js
const handles = {
  // 更新指针
  updatePointers(oldStartIdx, oldEndIdx, newStartIdx, newEndIdx) {
    oldPointerList.value = [
      {
        name: 'oldStartIdx',
        value: oldStartIdx
      },
      {
        name: 'oldEndIdx',
        value: oldEndIdx
      }
    ]
    newPointerList.value = [
      {
        name: 'newStartIdx',
        value: newStartIdx 
      },
      {
        name: 'newEndIdx',
        value: newEndIdx
      }
    ]
  },
}
```

然后我们就可以在`diff`函数中通过`handles.updatePointers()`更新指针了：

```js
const diff = (el, oldChildren, newChildren) => {
  // 指针
  // ...
  handles.updatePointers(oldStartIdx, oldEndIdx, newStartIdx, newEndIdx)
  // ...
}
```

这样指针就出来了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/504f9addb3da4d819ab60eb64d2ccfab~tplv-k3u1fbpfcp-zoom-1.image)


然后在`while`循环中会不断改变这四个指针，所以在循环中也需要更新：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  // ...
  handles.updatePointers(oldStartIdx, oldEndIdx, newStartIdx, newEndIdx)
}
```

但是这样显然是不行的，为啥呢，因为循环也就一瞬间就结束了，而我们希望每次都能停留一段时间，很简单，我们写个等待函数：

```js
const wait = t => {
  return new Promise(resolve => {
    setTimeout(
      () => {
        resolve()
      },
      t || 3000
    )
  })
}
```

然后我们使用`async/await`语法，就可以轻松在循环中实现等待了：

```js
const diff = async (el, oldChildren, newChildren) => {
  // ...
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // ...
    handles.updatePointers(oldStartIdx, oldEndIdx, newStartIdx, newEndIdx)
    await wait()
  }
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b77f68b169241c481d780522fe3614a~tplv-k3u1fbpfcp-zoom-1.image)


接下来我们新增两个变量，来突出表示当前正在比较的两个`VNode`：

```js
// 当前比较中的节点索引
const currentCompareOldNodeIndex = ref(-1)
const currentCompareNewNodeIndex = ref(-1)

const handles = {
  // 更新当前比较节点
  updateCompareNodes(a, b) {
    currentCompareOldNodeIndex.value = a
    currentCompareNewNodeIndex.value = b
  }
}
```

```html
<div
     class="nodeWrap"
     v-for="(item, index) in oldVNodeList"
     :key="item ? item.data.key : index"
     :class="{
         current: currentCompareOldNodeIndex === index,
     }"
     >
  <div class="node">{{ item ? item.children : '空' }}</div>
</div>
<div
     class="nodeWrap"
     v-for="(item, index) in newVNodeList"
     :key="item.data.key"
     :class="{
         current: currentCompareNewNodeIndex === index,
     }"
     >
  <div class="node">{{ item.children }}</div>
</div>
```

给当前比较中的节点添加一个类名，用来突出显示，接下来还是一样，需要在`diff`函数中调用该函数，但是，该怎么加呢：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if // ...
    } else if (isSameNode(oldStartVNode, newStartVNode)) {
      // ...
      oldStartVNode = oldChildren[++oldStartIdx]
      newStartVNode = newChildren[++newStartIdx]
    } else if (isSameNode(oldStartVNode, newEndVNode)) {
      // ...
      oldStartVNode = oldChildren[++oldStartIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else if (isSameNode(oldEndVNode, newStartVNode)) {
      // ...
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    } else if (isSameNode(oldEndVNode, newEndVNode)) {
      // ...
      oldEndVNode = oldChildren[--oldEndIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else {
      // ...
      newStartVNode = newChildren[++newStartIdx]
    }
```

我们想表现出头尾比较的过程，其实就在这些`if`条件中，也就是要在每个`if`条件中停留一段时间，那么可以直接这样吗：

```js
const isSameNode = async () => {
  // ...
  handles.updateCompareNodes()
  await wait()
}

if (await isSameNode(oldStartVNode, newStartVNode))
```

很遗憾，我尝试了不行，那么只能改写成其他形式了：

```js
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  let stop = false
  let _isSameNode = false
  if (oldStartVNode === null) {
    callbacks.updateInfo('')
    oldStartVNode = oldChildren[++oldStartIdx]
    stop = true
  }
  // ...
  if (!stop) {
    callbacks.updateInfo('头-头比较')
    callbacks.updateCompareNodes(oldStartIdx, newStartIdx)
    _isSameNode = isSameNode(oldStartVNode, newStartVNode)
    if (_isSameNode) {
      callbacks.updateInfo(
        'key值相同，可以复用，进行patch打补丁操作。新旧节点位置相同，不需要移动对应的真实DOM节点'
      )
    }
    await wait()
  }
  if (!stop && _isSameNode) {
    // ...
    oldStartVNode = oldChildren[++oldStartIdx]
    newStartVNode = newChildren[++newStartIdx]
    stop = true
  }
  // ...
}
```

我们使用一个变量来表示是否进入到了某个分支，然后把检查节点是否能复用的结果也保存到一个变量上，这样就可以通过不断检查这两个变量的值来判断是否需要进入到后续的比较分支中，这样比较的逻辑就不在`if`条件中了，就可以使用`await`了，同时我们还使用`updateInfo`增加了提示语：

```js
const handles = {
  // 更新提示信息
  updateInfo(tip) {
    info.value = tip
  }
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1bf0816a2ed4f22af8934ba5c3d73cd~tplv-k3u1fbpfcp-zoom-1.image)


接下来看一下节点的移动操作，当头（`oldStartIdx`对应的`oldStartVNode`节点）尾（`newEndIdx`对应的`newEndVNode`节点）比较发现可以复用时，在打完补丁后需要将`oldStartVNode`对应的真实`DOM`元素移动到`oldEndVNode`对应的真实`DOM`元素的位置，也就是插入到`oldEndVNode`对应的真实`DOM`的后面一个节点的前面：

```js
if (!stop && _isSameNode) {
  // 头-尾
  patchVNode(oldStartVNode, newEndVNode)
  // 把oldStartVNode节点移动到最后
  el.insertBefore(oldStartVNode.el, oldEndVNode.el.nextSibling)
  // 更新指针
  oldStartVNode = oldChildren[++oldStartIdx]
  newEndVNode = newChildren[--newEndIdx]
  stop = true
}
```

那么我们可以在`insertBefore`方法移动完真实的`DOM`元素后紧接着调用一下我们模拟列表的移动节点的方法：

```js
if (!stop && _isSameNode) {
  // ...
  el.insertBefore(oldStartVNode.el, oldEndVNode.el.nextSibling)
  callbacks.moveNode(oldStartIdx, oldEndIdx + 1)
  // ...
}
```

我们要操作的实际上是代表真实`DOM`节点的`actNodeList`列表，那么关键是要找到具体是哪个，首先头尾的四个节点指针它们表示的是在新旧`VNode`列表中的位置，所以我们可以根据`oldStartIdx`和`oldEndIdx`获取到`oldVNodeList`中对应位置的`VNode`，然后通过`key`值在`actNodeList`列表中找到对应的节点，进行移动、删除、插入等操作：

```js
const handles = {
  // 移动节点
  moveNode(oldIndex, newIndex) {
    let oldVNode = oldVNodeList.value[oldIndex]
    let newVNode = oldVNodeList.value[newIndex]
    let fromIndex = findIndex(oldVNode)
    let toIndex = findIndex(newVNode)
    actNodeList.value[fromIndex] = '#'
    actNodeList.value.splice(toIndex, 0, oldVNode)
    actNodeList.value = actNodeList.value.filter(item => {
      return item !== '#'
    })
  }
}

const findIndex = (vnode) => {
  return !vnode
    ? -1
    : actNodeList.value.findIndex(item => {
        return item && item.data.key === vnode.data.key
      })
}
```

其他的插入节点和删除节点也是类似的：

插入节点：

```js
const handles = {
  // 插入节点
  insertNode(newVNode, index, inNewVNode) {
    let node = {
      data: newVNode.data,
      children: newVNode.text
    }
    let targetIndex = 0
    if (index === -1) {
      actNodeList.value.push(node)
    } else {
      if (inNewVNode) {
        let vNode = newVNodeList.value[index]
        targetIndex = findIndex(vNode)
      } else {
        let vNode = oldVNodeList.value[index]
        targetIndex = findIndex(vNode)
      }
      actNodeList.value.splice(targetIndex, 0, node)
    }
  }
}
```

删除节点：

```js
const handles = {
  // 删除节点
  removeChild(index) {
    let vNode = oldVNodeList.value[index]
    let targetIndex = findIndex(vNode)
    actNodeList.value.splice(targetIndex, 1)
  }
}
```

这些方法在`diff`函数中的执行位置其实就是执行`insertBefore`、`removeChild`方法的地方，具体可以本文源码，这里就不在具体介绍了。

另外还可以凸显一下已经结束比较的元素、即将被添加的元素、即将被删除的元素等等，最终效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5f0e074330740b4a329ad2020c32dd2~tplv-k3u1fbpfcp-zoom-1.image)



时间原因，目前只实现了双端`diff`算法的效果，后续会增加上快速`diff`算法的动画过程，有兴趣的可以点个关注哟~

仓库：[https://github.com/wanglin2/VNode_visualization](https://github.com/wanglin2/VNode_visualization)。
