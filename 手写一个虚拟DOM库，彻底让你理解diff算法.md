「本文已参与好文召集令活动，点击查看：[后端、大前端双赛道投稿，2万元奖池等你挑战！](https://juejin.cn/post/6978685539985653767)」

所谓虚拟`DOM`就是用`js`对象来描述真实`DOM`，它相对于原生`DOM`更加轻量，因为真正的`DOM`对象附带有非常多的属性，另外配合虚拟`DOM`的`diff`算法，能以最少的操作来更新`DOM`，除此之外，也能让`Vue`和`React`之类的框架支持除浏览器之外的其他平台，本文会参考知名的[snabbdom](https://github.com/snabbdom/snabbdom)库来手写一个简易版的，配合图片示例一步步完成代码，一定让你彻底理解虚拟`DOM`的`patch`及`diff`算法。



# 创建虚拟DOM对象

虚拟`DOM`（下文称`VNode`）就是使用`js`的普通对象来描述`DOM`的类型、属性、子元素等信息，一般通过名为`h`的函数来创建，为了纯粹的理解`VNode`的`patch`过程，我们先不考虑元素的属性、样式、事件等，只考虑节点类型及节点内容，看一下此时的`VNode`结构：

```js
{
    tag: '',// 元素标签
    children: [],// 子元素
    text: '',// 子元素是文本节点的话，保存文本
    el: null// 对应的真实dom
}
```

`h`函数根据接收的参数返回该对象即可：

```js
export const h = (tag, children) => {
    let text = ''
    let el
    // 子元素是文本节点
    if (typeof children === 'string' || typeof children === 'number') {
        text = children
        children = undefined
    } else if (!Array.isArray(children)) {
        children = undefined
    }
    return {
        tag, // 元素标签
        children, // 子元素
        text, // 文本子节点的文本
        el// 真实dom
    }
}
```

比如我们要创建一个`div`的`VNode`可以这样使用：

```js
h('div', '我是文本')
h('div', [h('span')])
```



# 详解patch过程

`patch`函数是我们的主函数，主要用来进行新旧`VNode`的对比，找到差异来更新实际`DOM`，它接收两个参数，第一个参数可以是`DOM`元素或者是`VNode`，表示旧的`VNode`，第二参数表示新的`VNode`，一般只有第一次调用时才会传`DOM`元素，如果第一个参数为`DOM`元素的话我们直接忽略它的子元素把它转为一个`VNode`：

```js
export const patch = (oldVNode, newVNode) => {
    // dom元素
    if (!oldVNode.tag) {
        let el = oldVNode
        el.innerHTML = ''
        oldVNode = h(oldVNode.tagName.toLowerCase())
        oldVNode.el = el
    }
}
```

接下来新旧两个`VNode`就可以进行比较了：

```js
export const patch = (oldNode, newNode) => {
    // ...
    patchVNode(oldVNode, newVNode)
    // 返回新的vnode
  	return newVNode
}
```

在`patchVNode`方法里我们对新旧`VNode`进行比较及更新`DOM`。

首先如果两个`VNode`的类型不同，那么不用比较，直接使用新的`VNode`替换旧的：

```js
const patchVNode = (oldNode, newNode) => {
    if (oldVNode === newVNode) {
        return
    }
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        // ...
    } else { // 类型不同那么根据新的VNode创建新的dom节点，然后插入新节点，移除旧节点
        let newEl = createEl(newVNode)
        let parent = oldVNode.el.parentNode
        parent.insertBefore(newEl, oldVNode.el)
        parent.removeChild(oldVNode.el)
    }
}
```

`createEl`方法用来递归的把`VNode`转换成真实的`DOM`节点：

```js
const createEl = (vnode) => {
    let el = document.createElement(vnode.tag)
    vnode.el = el
    // 创建子节点
    if (vnode.children && vnode.children.length > 0) {
        vnode.children.forEach((item) => {
            el.appendChild(createEl(item))
        })
    }
    // 创建文本节点
    if (vnode.text) {
        el.appendChild(document.createTextNode(vnode.text))
    }
    return el
}
```

如果类型相同，那么就要根据其子节点的情况来判断进行哪种操作。

如果新节点只有一个文本子节点，那么移除旧节点的所有子节点（如果有的话），创建一个文本子节点：

```js
const patchVNode = (oldVNode, newVNode) => {
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        // 元素类型相同，那么旧元素肯定是进行复用的
        let el = newVNode.el = oldVNode.el
        // 新节点的子节点是文本节点
        if (newVNode.text) {
            // 移除旧节点的子节点
            if (oldVNode.children) {
                oldVNode.children.forEach((item) => {
                    el.removeChild(item.el)
                })
            }
            // 文本内容不相同则更新文本
            if (oldVNode.text !== newVNode.text) {
                el.textContent = newVNode.text
            }
        } else {
            // ...
        }
    } else { // 不同使用newNode替换oldNode
        // ...
    }
}
```

如果新节点的子节点非文本节点，那也有几种情况：

1.新节点不存在子节点，而旧节点存在，那么移除旧节点的子节点;

2.新节点不存在子节点，旧节点存在文本节点，那么移除该文本节点;

3.新节点存在子节点，旧节点存在文本节点，那么移除该文本节点，然后插入新节点;

4.新旧节点都有子节点的话那么就需要进入到`diff`阶段;

```js
const patchVNode = (oldVNode, newVNode) => {
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        // ...
        // 新节点的子节点是文本节点
        if (newVNode.text) {
            // ...
        } else {// 新节点不存在文本节点
            // 新旧节点都存在子节点，那么就要进行diff
            if (oldVNode.children && newVNode.children) {
                diff(el, oldVNode.children, newVNode.children)
            } else if (oldVNode.children) {// 新节点不存在子节点，那么移除旧节点的所有子节点
                oldVNode.children.forEach((item) => {
                    el.removeChild(item.el)
                })
            } else if (newVNode.children) {// 新节点存在子节点
                // 旧节点存在文本节点则移除
                if (oldVNode.text) {
                    el.textContent = ''
                }
                // 添加新节点的子节点
                newVNode.children.forEach((item) => {
                    el.appendChild(createEl(item))
                })
            } else if (oldVNode.text) {// 新节点啥也没有，旧节点存在文本节点
                el.textContent = ''
            }
        }
    } else { // 不同使用newNode替换oldNode
        // ...
    }
}
```

如果当新旧节点都存在非文本的子节点的话，那么就要进入到著名的`diff`阶段了，`diff`算法的目的主要是用来尽可能复用旧的节点，以减小`DOM`操作的开销。



# 图解diff算法

首先最简单的`diff`显然是同位置的新旧节点两两比较，但是在`WEB`场景下，倒序、排序、换位都是经常有可能发生的，所以同位置比较很多时候都很低效，无法满足这种常见场景，各种所谓的`diff`算法就是用来尽量能检查出这些情况，然后进行复用，`snabbdom`里的`diff`算法是一种双端比较的策略，同时从新旧节点的两端向中间开始比较，每一轮都会进行四次比较，所以需要四个指针，如下图：

![image-20210629144314119.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/167618b16b2749378675eb4f40f7eaf1~tplv-k3u1fbpfcp-watermark.image)

即上述四个位置的排列组合：`oldStartIdx`与`newStartIdx`、`oldStartIdx`与`newEndIdx`、`oldEndIdx`与`newStartIdx`、`oldEndIdx`与`newEndIdx`，每当发现所比较的两个节点可能可以复用的话，那么就对这两个节点进行`patch`和相应操作，并更新指针进入下一轮比较，那怎么判断两个节点是否能复用呢？这就需要使用到`key`了，因为光看是否是同类型的节点是远远不够的，因为同一个列表基本上类型都是一样的，那就跟从头开始的两两比较没有区别了，先修改一下我们的`h`函数：

```js
export const h = (tag, data = {}, children) => {
    // ...
    let key
    // 文本节点
    // ...
    if (data && data.key) {
        key = data.key
    }
    return {
        // ...
        key
    }
}
```

现在创建`VNode`的时候可以传入`key`：

```js
h('div', {key: 1}, '我是文本')
```

比较的终止条件也很明显，其中一个列表已经比较完了，也就是`oldStartIdx>oldEndIdx`或`newStartIdx>newEndIdx`，先把算法基本框架写一下：

```js
// 判断两个节点是否可进行复用
const isSameNode = (a, b) => {
    return a.key === b.key && a.tag === b.tag
}

// 进行diff
const diff = (el, oldChildren, newChildren) => {
    // 位置指针
    let oldStartIdx = 0
    let oldEndIdx = oldChildren.length - 1
    let newStartIdx = 0
    let newEndIdx = newChildren.length - 1
    // 节点指针
    let oldStartVNode = oldChildren[oldStartIdx]
    let oldEndVNode = oldChildren[oldEndIdx]
    let newStartVNode = newChildren[newStartIdx]
    let newEndVNode = newChildren[newEndIdx]
    
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {

        } else if (isSameNode(oldStartVNode, newEndVNode)) {

        } else if (isSameNode(oldEndVNode, newStartVNode)) {

        } else if (isSameNode(oldEndVNode, newEndVNode)) {

        }
    }
}
```

新增了四个变量用来保存四个位置的节点，接下来以上图为例来完善代码。

第一轮会发现`oldEndVNode`与`newEndVNode`是可复用节点，那么对它们进行`patch`，因为都在最后的位置，所以不需要移动`DOM`节点，更新指针即可：

```js
const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {} 
        else if (isSameNode(oldStartVNode, newEndVNode)) {} 
        else if (isSameNode(oldEndVNode, newStartVNode)) {} 
        else if (isSameNode(oldEndVNode, newEndVNode)) {
            patchVNode(oldEndVNode, newEndVNode)
            // 更新指针
            oldEndVNode = oldChildren[--oldEndIdx]
            newEndVNode = newChildren[--newEndIdx]
        }
    }
}
```

此时的位置信息如下：

![image-20210629144737773.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c52b27d5038346aba7690a0178a079ff~tplv-k3u1fbpfcp-watermark.image)

下一轮会发现`oldStartIdx`与`newEndIdx`是可复用节点，那么对`oldStartVNode`和`newEndVNode`两个节点进行`patch`，同时该节点在新列表里的位置是当前比较区间的最后一个，所以需要把`oldStartIdx`的真实`DOM`移动到旧列表当前比较区间的最后，也就是`oldEndVNode`之后：

![image-20210629145205542.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2b2d73351a74235986152b652b248f9~tplv-k3u1fbpfcp-watermark.image)

```js
const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {} 
        else if (isSameNode(oldStartVNode, newEndVNode)) {
            patchVNode(oldStartVNode, newEndVNode)
            // 把节点移动到oldEndVNode之后
            el.insertBefore(oldStartVNode.el, oldEndVNode.el.nextSibling)
            // 更新指针
            oldStartVNode = oldChildren[++oldStartIdx]
            newEndVNode = newChildren[--newEndIdx]
        } 
        else if (isSameNode(oldEndVNode, newStartVNode)) {} 
        else if (isSameNode(oldEndVNode, newEndVNode)) {}
    }
}
```

这轮以后位置如下：

![image-20210629145308700.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6048d157ddef440487f0f289d6acb0f3~tplv-k3u1fbpfcp-watermark.image)

下一轮比较很明显`oldStartVNode`与`newStartVNode`是可复用节点，那么对它们进行`patch`，因为都在第一个位置，所以也不需要移动节点，更新指针即可：

```js
const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {
            patchVNode(oldStartVNode, newStartVNode)
            // 更新指针
            oldStartVNode = oldChildren[++oldStartIdx]
            newStartVNode = newChildren[++newStartIdx]
        } 
        else if (isSameNode(oldStartVNode, newEndVNode)) {} 
        else if (isSameNode(oldEndVNode, newStartVNode)) {} 
        else if (isSameNode(oldEndVNode, newEndVNode)) {}
    }
}
```

这轮过后位置如下：

![image-20210629145420143.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2082e2e3b346459982942bc7614aae9a~tplv-k3u1fbpfcp-watermark.image)

再下一轮会发现`oldEndVNode`与`newStartVNode`是可复用节点，在新的列表里位置变成了当前比较区间的第一个，所以`patch`完后需要把节点移动到`oldStartVNode`的前面：

```js
const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {} 
        else if (isSameNode(oldStartVNode, newEndVNode)) {} 
        else if (isSameNode(oldEndVNode, newStartVNode)) {
            patchVNode(oldEndVNode, newStartVNode)
            // 把oldEndVNode节点移动到oldStartVNode前
            el.insertBefore(oldEndVNode.el, oldStartVNode.el)
            // 更新指针
            oldEndVNode = oldChildren[--oldEndIdx]
            newStartVNode = newChildren[++newStartIdx]
        } 
        else if (isSameNode(oldEndVNode, newEndVNode)) {}
    }
}
```

这轮后位置如下：

![image-20210629145632152.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bf459a8903a44cfb5c42533cefafca1~tplv-k3u1fbpfcp-watermark.image)

再下一轮会发现四次比较都没有发现可以复用的节点，这咋办呢，因为最终我们需要让旧列表变成新列表，所以当前的`newStartVNode`如果在旧列表里没找到可复用的，需要直接创建一个新节点插进去，但是我们一眼就看到了旧节点里有`c`节点，只是不在此轮比较的四个位置上，那么我们可以直接在旧的列表里搜索，找到了就进行`patch`，并且把该节点移动到当前比较区间的第一个，也就是`oldStartIdx`之前，这个位置空下来了就置为`null`，后续遍历到就跳过，如果没找到，那么说明这丫节点真的是新增的，直接创建该节点插入到`oldStartIdx`之前即可：

```js
// 在列表里找到可以复用的节点
const findSameNode = (list, node) => {
    return list.findIndex((item) => {
        return item && isSameNode(item, node)
    })
}

const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        // 某个位置的节点为null跳过此轮比较，只更新指针
        if (oldStartVNode === null) {
            oldStartVNode = oldChildren[++oldStartIdx]
        } else if (oldEndVNode === null) {
            oldEndVNode = oldChildren[--oldEndIdx]
        } else if (newStartVNode === null) {
            newStartVNode = oldChildren[++newStartIdx]
        } else if (newEndVNode === null) {
            newEndVNode = oldChildren[--newEndIdx]
        }
        else if (isSameNode(oldStartVNode, newStartVNode)) {} 
        else if (isSameNode(oldStartVNode, newEndVNode)) {} 
        else if (isSameNode(oldEndVNode, newStartVNode)) {} 
        else if (isSameNode(oldEndVNode, newEndVNode)) {}
        else {
            let findIndex = findSameNode(oldChildren, newStartVNode)
            // newStartVNode在旧列表里不存在，那么是新节点，创建并插入之
            if (findIndex === -1) {
                el.insertBefore(createEl(newStartVNode), oldStartVNode.el)
            } else {// 在旧列表里存在，那么进行patch，并且移动到oldStartVNode前
                let oldVNode = oldChildren[findIndex]
                patchVNode(oldVNode, newStartVNode)
                el.insertBefore(oldVNode.el, oldStartVNode.el)
                // 原位置空了置为null
                oldChildren[findIndex] = null
            }
            // 更新指针
            newStartVNode = newChildren[++newStartIdx]
        }
    }
}
```

具体到我们的示例上，在旧的列表里找到了，所以这轮过后位置信息如下：


![image-20210629154956514.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66792a1b073e4950b1b3379594154916~tplv-k3u1fbpfcp-watermark.image)

再下一轮比较和上轮一样，会进入搜索的分支，并且找到了`d`，所以也是`path`加移动节点，本轮过后如下：

![image-20210629155900787.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6962636a426b4127b1d259add6084e3b~tplv-k3u1fbpfcp-watermark.image)

因为`newStartIdx`大于`newEndIdx`，所以`while`循环就结束了，但是我们发现旧的列表里多了`g`和`h`节点，这两个在新列表里没有，所以需要把它们移除，反过来，如果新的列表里多了旧列表里没有的节点，那么就创建和插入之：

```js
const diff = (el, oldChildren, newChildren) => {
    // ...
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isSameNode(oldStartVNode, newStartVNode)) {} 
        else if (isSameNode(oldStartVNode, newEndVNode)) {} 
        else if (isSameNode(oldEndVNode, newStartVNode)) {} 
        else if (isSameNode(oldEndVNode, newEndVNode)) {}
        else {}
    }
    // 旧列表里存在新列表里没有的节点，需要删除
    if (oldStartIdx <= oldEndIdx) {
        for(let i = oldStartIdx; i <= oldEndIdx; i++) {
            oldChildren[i] && el.removeChild(oldChildren[i].el)
        }
    } else if (newStartIdx <= newEndIdx) {// 新列表里存在旧列表没有的节点，创建和插入
        // 在newEndVNode的下一个节点前插入，如果下一个节点不存在，那么insertBefore方法会执行appendChild的操作
        let before = newChildren[newEndIdx + 1] ? newChildren[newEndIdx + 1].el : null
        for(let i = newStartIdx; i <= newEndIdx; i++) {
            el.insertBefore(createEl(newChildren[i]), before)
        }
    }
}
```

以上就是双端`diff`的全过程，是不是还挺简单，画个图就十分容易理解了。



# 属性的更新

其他属性都通过`data`参数传入，先修改一下`h`函数：

```js
export const h = (tag, data = {}, children) => {
  // ...
  return {
    // ...
    data
  }
}

```



## 类名

类名通过`data`选项的`class`字段传递，比如：

```js
h('div',{
    class: {
        btn: true
    }
}, '文本')
```

类名的更新在`patchVNode`方法里进行，当两个节点的类型一样，那么更新类名，替换的话就相当于设置类名：

```js
// 更新节点类名
const updateClass = (el, newVNode) => {
    el.className = ''
    if (newVNode.data && newVNode.data.class) {
        let className = ''
        Object.keys(newVNode.data.class).forEach((cla) => {
            if (newVNode.data.class[cla]) {
                className += cla + ' '
            }
        })
        el.className = className
    }
}

const patchVNode = (oldVNode, newVNode) => {
    // ...
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        let el = newVNode.el = oldVNode.el
        // 更新类名
        updateClass(el, newVNode)
        // ...
    } else { // 不同使用newNode替换oldNode
        let newEl = createEl(newVNode)
        // 更新类名
        updateClass(newEl, newVNode)
        // ...
    }
}
```

逻辑很简单，直接把旧节点的类名替换成`newVNode`的类名。



## 样式

样式属性使用`data`的`style`字段传入：

```js
h('div',{
    style: {
        fontSize: '30px'
    }
}, '文本')
```

更新的时机和类名的位置一致：

```js
// 更新节点样式
const updateStyle = (el, oldVNode, newVNode) => {
  let oldStyle = oldVNode.data.style || {}
  let newStyle = newVNode.data.style || {}
  // 移除旧节点里存在新节点里不存在的样式
  Object.keys(oldStyle).forEach((item) => {
    if (newStyle[item] === undefined || newStyle[item] === '') {
      el.style[item] = ''
    }
  })
  // 添加旧节点不存在的新样式
  Object.keys(newStyle).forEach((item) => {
    if (oldStyle[item] !== newStyle[item]) {
      el.style[item] = newStyle[item]
    }
  })
}

const patchVNode = (oldVNode, newVNode) => {
    // ...
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        let el = newVNode.el = oldVNode.el
        // 更新样式
        updateStyle(el, oldVNode, newVNode)
        // ...
    } else {
        let newEl = createEl(newVNode)
        // 更新样式
        updateStyle(el, null, newVNode)
        // ...
    }
}
```



## 其他属性

其他属性保存在`data`的`attr`字段上，更新方式及位置和样式的完全一致：

```js
// 更新节点属性
const updateAttr = (el, oldVNode, newVNode) => {
    let oldAttr = oldVNode && oldVNode.data.attr ? oldVNode.data.attr : {}
    let newAttr = newVNode.data.attr || {}
    // 移除旧节点里存在新节点里不存在的属性
    Object.keys(oldAttr).forEach((item) => {
        if (newAttr[item] === undefined || newAttr[item] === '') {
            el.removeAttribute(item)
        }
    })
    // 添加旧节点不存在的新属性
    Object.keys(newAttr).forEach((item) => {
        if (oldAttr[item] !== newAttr[item]) {
            el.setAttribute(item, newAttr[item])
        }
    })
}

const patchVNode = (oldVNode, newVNode) => {
    // ...
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        let el = newVNode.el = oldVNode.el
        // 更新属性
        updateAttr(el, oldVNode, newVNode)
        // ...
    } else {
        let newEl = createEl(newVNode)
        // 更新属性
        updateAttr(el, null, newVNode)
        // ...
    }
}
```



## 事件

最后来看一下事件的更新，事件与其他属性不同的是如果删除一个节点的话需要把它的事件先全部解绑，否则可能会存在内存泄漏的问题，那么就需要在各个移除节点的时机都先解绑事件：

```js
// 移除某个VNode对应的dom的所有事件
const removeEvent = (oldVNode) => {
  if (oldVNode && oldVNode.data && oldVNode.data.event) {
    Object.keys(oldVNode.data.event).forEach((item) => {
      oldVNode.el.removeEventListener(item, oldVNode.data.event[item])
    })
  }
}

// 更新节点事件
const updateEvent = (el, oldVNode, newVNode) => {
  let oldEvent = oldVNode && oldVNode.data.event ? oldVNode.data.event : {}
  let newEvent = newVNode.data.event || {}
  // 解绑不再需要的事件
  Object.keys(oldEvent).forEach((item) => {
    if (newEvent[item] === undefined || oldEvent[item] !== newEvent[item]) {
      el.removeEventListener(item, oldEvent[item])
    }
  })
  // 绑定旧节点不存在的新事件
  Object.keys(newEvent).forEach((item) => {
    if (oldEvent[item] !== newEvent[item]) {
      el.addEventListener(item, newEvent[item])
    }
  })
}

const patchVNode = (oldVNode, newVNode) => {
    // ...
    // 元素标签相同，进行patch
    if (oldVNode.tag === newVNode.tag) {
        // 元素类型相同，那么旧元素肯定是进行复用的
        let el = newVNode.el = oldVNode.el
        // 更新事件
        updateEvent(el, oldVNode, newVNode)
        // ...
    } else {
        let newEl = createEl(newVNode)
        // 移除旧节点的所有事件
        removeEvent(oldNode)
        // 更新事件
        updateEvent(newEl, null, newVNode)
        // ...
    }
}
// 其他还有几处需要添加removeEvent()，有兴趣请看源码
```

以上属性的更新逻辑都比较粗糙，仅用于参考，可以参考[snabbdom](https://github.com/snabbdom/snabbdom)的源码自行完善。



# 总结

以上代码实现了一个简单的虚拟`DOM`库，详细分解了`patch`过程和`diff`的过程，如果需要用在非浏览器平台上，只要把`DOM`相关的操作抽象成接口，不同平台上使用不同的接口即可，完整代码在[https://github.com/wanglin2/VNode-Demo](https://github.com/wanglin2/VNode-Demo)。

