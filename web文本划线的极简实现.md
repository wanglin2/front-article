---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

# 开篇

文本划线是目前逐渐流行的一个功能，不管你是小说阅读网站，还是卖教程的的网站，一般都会有记笔记或者评论的功能，传统的做法都是在文章底部加一个评论区，优点是简单，统一，缺点是不方便对文章的某一段或一句话进行针对性的评论，所以出现了划线及评论的需求，目前我见到的产品有划线功能的有：微信阅读APP、极客时间：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3221f40cf06d456a8487eeeab4cb09f0~tplv-k3u1fbpfcp-watermark.image)

InfoQ写作平台：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63d22eb7ed4b46c7a9536b19662a3c7d~tplv-k3u1fbpfcp-watermark.image)

等等，这个功能看似简单，实际上难点还是很多的，比如如何高性能的对各种复杂的文本结构划线、如何尽可能少的存储数据、如何精准的回显划线、如何处理重复划线、如何应对文本后续编辑的情况等等。

作为一个前端搬砖工，每当看到一个有意思的小功能时我都想自己去把它做出来，但是看了仅有的几篇相关文章之后，发现，不会😓，这些文章介绍的都只是一个大概思路，看完让人感觉好像会了，但是细想就会发现很多问题，只能去看源码，看源码总是费时的，还不一定能看懂。想要实现一个生产可用的难度还是很大的，所以本文退而求其次，单纯的写一个`demo`开心开心。`demo`效果请点击：[http://lxqnsys.com/#/demo/textUnderline](http://lxqnsys.com/#/demo/textUnderline)。

# 总体思路

总体思路很简单，遍历选区内的所有文本，切割成单个字符，给每个字符都包裹上划线元素，重复划线的话就在最深层继续包裹，事件处理的话从最深的元素开始。

存储的方式是记录该划线文本外层第一个非划线元素的标签名和索引，以及字符在其内所有字符里总的偏移量。

回显的方式是获取到上述存储数据对应的元素，然后遍历该元素的字符添加划线元素。

# 实现

## HTML结构

```html
<div class="article" ref="article"></div>
```

文本内容就放在上述的`div`里，我从掘金小册里随便挑选了一篇文章，把它的`html`结构原封不动的复制粘贴进去：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f289f332523b483cb2b23aa74bcaa043~tplv-k3u1fbpfcp-watermark.image)


## 显示tooltip

首先要做的是在选区上显示一个划线按钮，这个很简单，我们监听一下`mouseup`事件，然后获取一下选区对象，调用它的`getBoundingClientRect`方法获取位置信息，然后设置到我们的`tooltip`元素上：

```js
document.addEventListener('mouseup', this.onMouseup)

onMouseup () {
    // 获取Selection对象，里面可能包含多个`ranges`（区域）
    let selObj = window.getSelection()
    // 一般就只有一个Range对象
    let range = selObj.getRangeAt(0)
    // 如果选区起始位置和结束位置相同，那代表没有选到任何东西
    if (range.collapsed) {
        return
    }
    this.range = range.cloneRange()
    this.tipText = '划线'
    this.setTip(range)
}

setTip (range) {
    let { left, top, width } = range.getBoundingClientRect()
    this.tipLeft = left + (width - 80) / 2
    this.tipTop = top - 40
    this.showTip = true
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47cdf3dae38641d090489297ca5378f8~tplv-k3u1fbpfcp-watermark.image)


## 划线

给`tooltip`绑定一下点击事件，点击后需要获取到选区内的所有文本节点，先看一下`Range`对象的结构：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d8afb4a0a5f4b33905962b652c5fd02~tplv-k3u1fbpfcp-watermark.image)

简单介绍一下：

`collapsed`属性表示开始和结束的位置是否相同；

`commonAncestorContainer`属性返回包含`startContainer`和`endContainer`的公共父节点；

`endContainer`属性返回包含`range`终点的节点，通常是文本节点；

`endOffset`返回`range`终点在`endContainer`内的位置的数字；

`startContainer`属性返回包含`range`起点的节点，通常是文本节点；

`startContainer`返回`range`起点在`startContainer`内的位置的数字；

所以目标是要遍历`startContainer`和`endContainer`两个节点之间的所有节点来收集文本节点，受限于笔者匮乏的算法和数据结构知识，只能选择一个投机取巧的方法，遍历`commonAncestorContainer`节点，然后使用`range`对象的`isPointInRange()`方法来检测当前遍历的节点是否在选区范围内，这个方法需要注意的两个点地方，一个是`isPointInRange()`方法目前不支持`IE`，二是首尾节点需要单独处理，因为首尾节点可能部分在选区内，这样这个方法是返回`false`的。

```js
mark () 
  this.textNodes = []
  let { commonAncestorContainer, startContainer, endContainer } = this.range
  this.walk(commonAncestorContainer, (node) => {
    if (
      node === startContainer ||
      node === endContainer ||
      this.range.isPointInRange(node, 0)
    ) {// 起始和结束节点，或者在范围内的节点，如果是文本节点则收集起来
      if (node.nodeType === 3) {
        this.textNodes.push(node)
      }
    }
  })
  this.handleTextNodes()
  this.showTip = false
  this.tipText = ''
}
```

`walk`是一个深度优先遍历的函数：

```js
walk (node, callback = () => {}) {
    callback(node)
    if (node && node.childNodes) {
        for (let i = 0; i < node.childNodes.length; i++) {
            this.walk(node.childNodes[i], callback)
        }
    }
}
```

获取到选区范围内的所有文本节点后就可以切割字符进行元素替换：

```js
handleTextNodes () {
    // 生成本次的唯一id
    let id = ++this.idx
    // 遍历文本节点
    this.textNodes.forEach((node) => {
        // 范围的首尾元素需要判断一下偏移量，用来截取字符
        let startOffset = 0
        let endOffset = node.nodeValue.length
        if (
            node === this.range.startContainer &&
            this.range.startOffset !== 0
        ) {
            startOffset = this.range.startOffset
        }
        if (node === this.range.endContainer && this.range.endOffset !== 0) {
            endOffset = this.range.endOffset
        }
        // 替换该文本节点
        this.replaceTextNode(node, id, startOffset, endOffset)
    })
    // 序列化进行存储，获取刚刚生成的所有该id的划线元素
    this.serialize(this.$refs.article.querySelectorAll('.mark_id_' + id))
}
```

如果是首节点，且`startOffset`不为0，那么`startOffset`之前的字符不需要添加划线包裹元素，如果是尾节点，且`endOffset`不为0，那么`endOffset`之后的字符不需要划线，中间的其他所有文本都需要进行切割及划线：

```js
replaceTextNode (node, id, startOffset, endOffset) {
    // 创建一个文档片段用来替换文本节点
    let fragment = document.createDocumentFragment()
    let startNode = null
    let endNode = null
    // 截取前一段不需要划线的文本
    if (startOffset !== 0) {
        startNode = document.createTextNode(
            node.nodeValue.slice(0, startOffset)
        )
    }
    // 截取后一段不需要划线的文本
    if (endOffset !== 0) {
        endNode = document.createTextNode(node.nodeValue.slice(endOffset))
    }
    startNode && fragment.appendChild(startNode)
    // 切割中间的所有文本
    node.nodeValue
        .slice(startOffset, endOffset)
        .split('')
        .forEach((text) => {
        // 创建一个span标签用来作为划线包裹元素
        let textNode = document.createElement('span')
        textNode.className = 'markLine mark_id_' + id
        textNode.setAttribute('data-id', id)
        textNode.textContent = text
        fragment.appendChild(textNode)
    })
    endNode && fragment.appendChild(endNode)
    // 替换文本节点
    node.parentNode.replaceChild(fragment, node)
}
```

效果如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17149bd2b47946a4b9c6a2147fb3b677~tplv-k3u1fbpfcp-watermark.image)

此时`html`结构：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e450a33eb214733b72e149924ebaeb5~tplv-k3u1fbpfcp-watermark.image)


## 序列化存储

一次性的划线是没啥用的，那还不如在文章上面盖一个`canvas`元素，给用户一个自由画布，所以还需要进行保存，下次打开还能重新显示之前画的线。

存储的关键是要能让下次还能定位回去，参考其他文章介绍的方法，本文选择的是存储划线元素外层的第一个非划线元素的标签名，以及在指定节点范围内的同类型元素里的索引，以及该字符在该非划线元素里的总的字符偏移量。描述起来可能有点绕，看代码：

```js
serialize (markNodes) {
    // 选择article元素作为根元素，这样的好处是页面的其他结构如果改变了不影响划线元素的定位
    let root = this.$refs.article
    // 遍历刚刚生成的本次划线的所有span节点
    markNodes.forEach((markNode) => {
        // 计算该字符离外层第一个非划线元素的总的文本偏移量
        let offset = this.getTextOffset(markNode)
        // 找到外层第一个非划线元素
        let { tagName, index } = this.getWrapNode(markNode, root)
        // 保存相关数据
        this.serializeData.push({
          tagName,
          index,
          offset,
          id: markNode.getAttribute('data-id')
        })
    })
}
```

计算字符离外层第一个非划线元素的总的文本偏移量的思路是先算获取同级下之前的兄弟元素的总字符数，再依次向上遍历父元素及其之前的兄弟节点的总字符数，直到外层元素：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2916447562304033bd67666fccf7bcc1~tplv-k3u1fbpfcp-watermark.image)

```js
getTextOffset (node) {
    let offset = 0
    let parNode = node
    // 遍历直到外层第一个非划线元素
    while (parNode && parNode.classList.contains('markLine')) {
        // 获取前面的兄弟元素的总字符数
        offset += this.getPrevSiblingOffset(parNode)
        parNode = parNode.parentNode
    }
    return offset
}
```

获取前面的兄弟元素的总字符数：

```js
getPrevSiblingOffset (node) {
    let offset = 0
    let prevNode = node.previousSibling
    while (prevNode) {
        offset +=
            prevNode.nodeType === 3
            ? prevNode.nodeValue.length
        : prevNode.textContent.length
        prevNode = prevNode.previousSibling
    }
    return offset
}
```

获取外层第一个非划线元素在上面获取字符数的方法里其实已经有了：

```js
getWrapNode (node, root) {
  	// 找到外层第一个非划线元素
    let wrapNode = node.parentNode
    while (wrapNode.classList.contains('markLine')) {
        wrapNode = wrapNode.parentNode
    }
    let wrapNodeTagName = wrapNode.tagName
    // 计算索引
    let wrapNodeIndex = -1
    // 使用标签选择器获取所有该标签元素
    let els = root.getElementsByTagName(wrapNodeTagName)
    els = [...els].filter((item) => {// 过滤掉划线元素
      return !item.classList.contains('markLine');
    }).forEach((item, index) => {// 计算当前元素在其中的索引
      if (wrapNode === item) {
        wrapNodeIndex = index
      }
    })
    return {
        tagName: wrapNodeTagName,
        index: wrapNodeIndex
    }
}
```

最后存储的数据示例如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d304df41d1940079d0d31fae4b995d4~tplv-k3u1fbpfcp-watermark.image)


## 反序列化显示

显示就是根据上面存储的数据把线画上，遍历上面的数据，先根据`tagName`和`index`获取到指定元素，然后遍历该元素下的所有文本节点，根据`offset`找到需要划线的字符：

```js
deserialization () {
    let root = this.$refs.article
    // 遍历序列化的数据
    markData.forEach((item) => {
        // 获取到指定元素
        let els = root.getElementsByTagName(item.tagName)
        els = [...els].filter((item) => {// 过滤掉划线元素
          return !item.classList.contains('markLine');
        })
        let wrapNode = els[item.index]
        let len = 0
        let end = false
        // 遍历该元素所有节点
        this.walk(wrapNode, (node) => {
            if (end) {
                return
            }
            // 如果是文本节点
            if (node.nodeType === 3) {
                // 如果当前文本节点的字符数+之前的总数大于offset，说明要找的字符就在该文本内
                if (len + node.nodeValue.length > item.offset) {
                    // 计算在该文本里的偏移量
                    let startOffset = item.offset - len
                    // 因为我们是切割到单个字符，所以总长度也就是1
                    let endOffset = startOffset + 1
                    this.replaceTextNode(node, item.id, startOffset, endOffset)
                    end = true
                }
                // 累加字符数
                len += node.nodeValue.length
            }
        })
    })
}
```

结果如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f584a39b05c94317b01678f8eee004b3~tplv-k3u1fbpfcp-watermark.image)


## 删除划线

删除划线很简单，我们监听一下点击事件，如果目标元素是划线元素，那么获取一下所有该`id`的划线元素，创建一个`range`，显示一下`tooltip`，然后点击后把该划线元素删除即可。

```js
// 显示取消划线的tooltip
showCancelTip (e) {
    let tar = e.target
    if (tar.classList.contains('markLine')) {
        e.stopPropagation()
        e.preventDefault()
        // 获取划线id
        this.clickId = tar.getAttribute('data-id')
        // 获取该id的所有划线元素
        let markNodes = document.querySelectorAll('.mark_id_' + this.clickId)
        // 选择第一个和最后一个文本节点来作为range边界
        let startContainer = markNodes[0].firstChild
        let endContainer = markNodes[markNodes.length - 1].lastChild
        this.range = document.createRange()
        this.range.setStart(startContainer, 0)
        this.range.setEnd(
          endContainer,
          endContainer.nodeValue.length
        )
        this.tipText = '取消划线'
        this.setTip(this.range)
    }
}
```

点击了取消按钮后遍历该`id`的所有划线节点，进行元素替换：

```js
cancelMark () {
    this.showTip = false
    this.tipText = ''
    let markNodes = document.querySelectorAll('.mark_id_' + this.clickId)
    // 遍历所有划线街道
    for (let i = 0; i < markNodes.length; i++) {
        let item = markNodes[i]
        // 如果还有子节点，也就是其他id的划线元素
        if (item.children[0]) {
            let node = item.children[0].cloneNode(true)
            // 子节点替换当前节点
            item.parentNode.replaceChild(node, item)
        } else {// 否则只有文本的话直接创建一个文本节点来替换
            let textNode = document.createTextNode(item.textContent)
            item.parentNode.replaceChild(textNode, item)
        }
    }
    // 从序列化数据里删除该id的数据
    this.serializeData = this.serializeData.filter((item) => {
        return item.id !== this.clickId
    })
}
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdb0a1d4daca456dae3928a92a0e3a25~tplv-k3u1fbpfcp-watermark.image)


## 缺点

到这里这个极简划线就结束了，现在来看一下这个极简的方法有什么缺点.

首先毋庸置疑的就是如果划线字符很多，重复划线很多次，那么会生成非常多的`span`标签及嵌套层次，节点数量是影响页面性能的一个大问题。

第二个问题是需要存储的数据也会很大，增加存储成本和网络传输时间：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c575386f4b614205bcd694f353eac6be~tplv-k3u1fbpfcp-watermark.image)

这可以通过把字段名字压缩一下，改成一个字母，另外可以把连续的字符合并一下来稍微优化一下，但是然并卵。

第三个问题是如其名，文本划线，真的是只能给文本进行划线，其他的图片上面的就不行了：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1104c5b6921a45c7aab1678410e0816e~tplv-k3u1fbpfcp-watermark.image)

第四个问题是无法应对如果划线后文章被修改了，`html`结构变化了的问题。

这几个问题个个扎心，导致它只能是个`demo`。

## 稍微优化一下

很容易想到的一个优化方法是不要把字符单个切割，整块包裹不就好了吗，道理是这个道理：

```js
replaceTextNode (node, id, startOffset, endOffset) {
    // ...
    startNode && fragment.appendChild(startNode)

    // 改成直接包裹整块文本
    let textNode = document.createElement('span')
    textNode.className = 'markLine mark_id_' + id
    textNode.setAttribute('data-id', id)
    textNode.textContent = node.nodeValue.slice(startOffset, endOffset)
    fragment.appendChild(textNode)
    
    endNode && fragment.appendChild(endNode)
    // ...
}
```

这样序列化时需要增加一个长度的字段：

```js
let textLength = markNode.textContent.length
if (textLength > 0) {// 过滤掉长度为0的空字符，否则会有不可预知的问题
	this.serializeData.push({
      tagName,
      index,
      offset,
      length: textLength,// ++
      id: markNode.getAttribute('data-id')
  })
}
```

这样序列化后的数据量会大大减少：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b83dbb5e5864bb8a456cc553a0a5fcc~tplv-k3u1fbpfcp-watermark.image)

接下来反序列化也需要修改，字符长度不定的话就可能跨文本节点了：

```js
deserialization () {
    let root = this.$refs.article
    markData.forEach((item) => {
        let wrapNode = root.getElementsByTagName(item.tagName)[item.index]
        let len = 0
        let end = false
        let first = true
        let _length = item.length
        this.walk(wrapNode, (node) => {
            if (end) {
                return
            }
            if (node.nodeType === 3) {
                let nodeTextLength = node.nodeValue.length
                if (len + nodeTextLength > _offset) {
                    // startOffset之前的文本不需要划线
                    let startOffset = (first ? item.offset - len : 0)
                    first = false
                    // 如果该文本节点剩余的字符数量小于划线文本的字符长度的话代表该文本节点还只是划线文本的一部分，还需要到下一个文本节点里去处理
                    let endOffset = startOffset + (nodeTextLength - startOffset >= _length ? _length : nodeTextLength - startOffset)
                    this.replaceTextNode(node, item.id, startOffset, endOffset)
                    // 长度需要减去之前节点已经处理掉的长度
                    _length = _length - (nodeTextLength - startOffset)
                    // 如果剩余要处理的划线文本的字符数量为0代表已经处理完了，可以结束了
                    if (_length <= 0) {
                      end = true
                    }
                  }
                len += nodeTextLength
            }
        })
    })
}
```

最后取消划线也需要修改，因为子节点可能就不是只有单纯的一个划线节点或文本节点了，需要遍历全部子节点：

```js
cancelMark () {
    this.showTip = false
    this.tipText = ''
    let markNodes = document.querySelectorAll('.mark_id_' + this.clickId)
    for (let i = 0; i < markNodes.length; i++) {
        let item = markNodes[i]
        let fregment = document.createDocumentFragment()
        for (let j = 0; j < item.childNodes.length; j++) {
            fregment.appendChild(item.childNodes[j].cloneNode(true))
        }
        item.parentNode.replaceChild(fregment, item)
    }
    this.serializeData = this.serializeData.filter((item) => {
        return item.id !== this.clickId
    })
}
```

现在再来看一下效果：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/159ce99ea71746a2ae13bd5dfd29a474~tplv-k3u1fbpfcp-watermark.image)

`html`结构：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cac5a8527f4b4dec87128811612655f0~tplv-k3u1fbpfcp-watermark.image)

可以看到无论是序列化的数据还是`DOM`结构都已经简洁了很多。

但是，如果文档结构很复杂或者多次重复划线最终产生的节点和数据还是比较大的。

# 总结

本文介绍了一个实现`web`文本划线功能的极简实现，最初的想法是通过切割成单个字符来进行包裹，这样的优点是十分简单，缺点也很明显，产生的序列号数据很大、修改的`DOM`结构很复杂，在文章及`demo`的写作过程中经过实践，发现直接包裹整块文字也并不会带来太多问题，但是却能减少和优化很多要存储的数据和`DOM`结构，所以很多时候，想当然是不对的，最后想说，数据结构和算法真的很重要😭。

示例代码在：[https://github.com/wanglin2/textUnderline](https://github.com/wanglin2/textUnderline)。

参考文章：

1.[如何用JS实现“划词高亮”的在线笔记功能？](https://juejin.cn/post/6844903827745832967)

2.[「划线高亮」和「插入笔记」—— 不止是前端知识点](https://juejin.cn/post/6870058781527506952)
