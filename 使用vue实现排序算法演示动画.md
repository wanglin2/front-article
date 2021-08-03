# 缘起

最近做的一个小需求涉及到排序，界面如下所示：


![](https://user-gold-cdn.xitu.io/2020/4/8/171590d9997cef06)

因为项目是使用vue的，所以实现方式很简单，视图部分不用管，本质上就是操作数组，代码如下：

```js
{
    // 上移
    moveUp (i) {
        // 把位置i的元素移到i-1上
      let tmp = this.form.replayList.splice(i, 1)
      this.form.replayList.splice(i - 1, 0, tmp[0])
    },

    // 下移
    moveDown (i) {
        // 把位置i的元素移到i+1上
      let tmp = this.form.replayList.splice(i, 1)
      this.form.replayList.splice(i + 1, 0, tmp[0])
    }
}
```

这样就可以正常的交换位置了，但是是突变的，没有动画，所以不明显，于是一个码农的自我修养（实际上是太闲）让我打开了vue的网站，看到了这个示例：[https://cn.vuejs.org/v2/guide/transitions.html#%E5%88%97%E8%A1%A8%E7%9A%84%E6%8E%92%E5%BA%8F%E8%BF%87%E6%B8%A1](https://cn.vuejs.org/v2/guide/transitions.html#%E5%88%97%E8%A1%A8%E7%9A%84%E6%8E%92%E5%BA%8F%E8%BF%87%E6%B8%A1)

![](https://user-gold-cdn.xitu.io/2020/4/8/171590ff520c9a7f?w=141&h=332&f=gif&s=12713)

这个示例我已看过多遍，但是一直没用过，这里刚好就是我要的效果，于是一通复制粘贴大法：

```html
<template>
    <transition-group name="flip-list" tag="p">
        <!--循环生成列表部分，略-->
    </transition-group>
</template>

<style>
.flip-list-move {
  transition: transform 0.5s;
}
</style>
```

这样就有交换的过渡效果了，如下：

![](https://user-gold-cdn.xitu.io/2020/4/8/17159105a31d3b9e?w=649&h=433&f=gif&s=39352)

嗯，舒服了很多，这个需求到这里就完了，但是事情并没有结束，我突然想到了以前看一些算法文章的时候通常会配上一些演示的动画，感觉跟这个很类似，那么是不是可以用这个来实现呢，当然是可以的。



# 实现算法演示动画

先写一下基本的布局和样式：

```html
<template>
  <div class="sortList">
      <transition-group name="flip-list" tag="p">
        <div
          class="item"
          v-for="item in list"
          :key="item.index"
          :style="{height: (item.value / max * 100) + '%'}"
        >
          <span class="value">{{item.value}}</span>
        </div>
      </transition-group>
    </div>
</template>

<style>
.flip-list-move {
  transition: transform 0.5s;
}
</style>
```

`list`是要排序的数组，当然是经过处理的，在真正的源数组上加上了唯一的`index`，因为要能正常过渡的话列表的每一项需要一个唯一的key：

```js
const arr = [10, 43, 23, 65, 343, 75, 100, 34, 45, 3, 56, 22]

export default {
  data () {
    return {
      list: arr.map((item, index) => {
        return {
          index,
          value: item
        }
      })
    }
  }
}
```

`max`是这个数组中最大的值，用来按比例显示高度：

```js
{
    computed: {
        max () {
            let max = 0
            arr.forEach(item => {
                if (item > max) {
                    max = item
                }
            })
            return max
        }
  }
}
```

其他样式可以自行发挥，显示效果如下：

![](https://user-gold-cdn.xitu.io/2020/4/8/1715910a44515cd2?w=553&h=311&f=png&s=4096)

简约而不简单~，现在万事俱备，只欠让它动起来，排序算法有很多，但是本人比较菜，所以就拿冒泡算法来举例，最最简单的冒泡排序算法如下：

```js
{
    mounted(){
        this.bubbleSort()
    },
    methods: {
        bubbleSort() {
          let len = this.list.length

          for (let i = 0; i < len; i++) {
            for (let j = 0; j < len - i - 1; j++) {
              if (this.list[j] > this.list[j + 1]) {  // 相邻元素两两对比
                let tmp = this.list[j]        // 元素交换
                this.$set(this.list, j, this.list[j + 1])
                this.$set(this.list, j + 1, tmp)
              }
            }
          }
        }
    }
}
```

但是这样写它是不会动的，瞬间就给你排好了：

![](https://user-gold-cdn.xitu.io/2020/4/8/17159161430d868a?w=545&h=311&f=png&s=4047)

试着加个延时：

```js
{
    mounted () {
        setTimeout(() => {
            this.bubbleSort()
        }, 1000)
    }
}
```

刷新看效果：

![](https://user-gold-cdn.xitu.io/2020/4/8/1715911a6965c983?w=541&h=301&f=gif&s=21743)

有动画了，不过这种不是我们要的，我们要的应该是下面这样的才对：

![](https://user-gold-cdn.xitu.io/2020/4/8/171591230c4af992?w=826&h=257&f=gif&s=466890)

所以来改造一下，因为for循环是只要开始执行就不会停的，所以需要把两个for循环改成两个函数，这样可以控制每个循环什么时候执行：

```js
{
    bubbleSort () {
      let len = this.list.length
      let i = 0
      let j = 0
      // 内层循环
      let innerLoop = () => {
        // 每个内层循环都执行完毕后再执行下一个外层循环
        if (j >= (len - 1 - i)) {
          j = 0
          i++
          outLoop()
          return false
        }
        if (this.list[j].value > this.list[j + 1].value) {
          let tmp = this.list[j]
          this.$set(this.list, j, this.list[j + 1])
          this.$set(this.list, j + 1, tmp)
        }
        // 动画是500毫秒，所以每隔800毫秒执行下一个内层循环
        setTimeout(() => {
          j++
          innerLoop()
        }, 800)
      }
      // 外层循环
      let outLoop = () => {
        if (i >= len) {
          return false
        }
        innerLoop()
      }
      outLoop()
    }
}
```

这样就实现了每一步的动画效果：

![](https://user-gold-cdn.xitu.io/2020/4/8/17159126f8ddd78e?w=541&h=301&f=gif&s=85560)

但是这样不太直观，因为有些相邻不用交换的时候啥动静也没有，不知道当前具体排到了哪两个，所以需要突出当前正在比较交换的两个元素，首先模板部分给当前正在比较的元素加一个类名，用来高亮显示：

```html
<div
     :class="{sortingHighlight: sorts.includes(item.index)}"
     >
    <span class="value">{{item.value}}</span>
</div>
```

js部分定义一个数组`sorts`来装载当前正在比较的两个元素的唯一的`index`值：

```js
{
    data() {
        return {
            sorts: []
        }
    },
    methods: {
        bubbleSort () {
            // ...
            // 内层循环
            let innerLoop = () => {
                // 每个内层循环都执行完毕后再执行下一个外层循环
                if (j >= (len - 1 - i)) {
                    // 清空数组
                    this.sorts = []
                    j = 0
                    i++
                    outLoop()
                    return false
                }
                // 将当前正在比较的两个元素的index装到数组里
                this.sorts = [this.list[j].index, this.list[j + 1].index]
                // ...
            }
            // 外层循环
            // ...
        }
    }
}
```

修改后效果如下：

![](https://user-gold-cdn.xitu.io/2020/4/8/1715912ba72ea958?w=541&h=301&f=gif&s=147483)

最后，再参考刚才别人的示例把已排序的元素也加上高亮：

```js
{
    data() {
        return {
            sorted: []
        }
    },
    methods: {
        bubbleSort () {
            // ...
            // 内层循环
            let innerLoop = () => {
                // 每个内层循环都执行完毕后再执行下一个外层循环
                if (j >= (len - 1 - i)) {
                    this.sorts = []
                    // 看这里，把排好的元素加到数组里就ok了
                    this.sorted.push(this.list[j].index)
                    j = 0
                    i++
                    outLoop()
                    return false
                }
                // ...
            }
            // 外层循环
            // ...
        }
    }
}
```

最终效果如下：

![](https://user-gold-cdn.xitu.io/2020/4/8/1715915482eac244?w=541&h=301&f=gif&s=147521)

接下来看一下选择排序，这是选择排序的算法：

```js
{
    selectSort() {
        for (let i = 0; i < len - 1; i++) {
            minIndex = i
            for (let j = i + 1; j < len; j++) {
                if (this.list[j].value < this.list[minIndex].value) {
                    minIndex = j
                }
            }
            tmp = this.list[minIndex]
            this.$set(this.list, minIndex, this.list[i])
            this.$set(this.list, i, tmp)
        }
    }
}
```

选择排序涉及到一个当前最小元素，所以需要新增一个高亮：

```html
<div
     :class="{minHighlight: min === item.index , sortingHighlight: sorts.includes(item.index), sortedHighlight: sorted.includes(item.index)}"
     >
    <span class="value">{{item.value}}</span>
</div>
```

```js
{
    data () {
        return {
            min: 0
        }
    },
    methods: {
        selectSort () {
            let len = this.list.length
            let i = 0; let j = i + 1
            let minIndex, tmp
			// 内层循环
            let innerLoop = () => {
                if (j >= len) {
                    // 高亮最后要交换的两个元素
                    this.sorts = [this.list[i].index, this.list[minIndex].index]
                    // 延时是用来给高亮一点时间
                    setTimeout(() => {
                        // 交换当前元素和比当前元素小的元素的位置
                        tmp = this.list[minIndex]
                        this.$set(this.list, minIndex, this.list[i])
                        this.$set(this.list, i, tmp)
                        this.sorted.push(this.list[i].index)
                        i++
                        j = i + 1
                        outLoop()
                    }, 1000)
                    return false
                }
                // 高亮当前正在寻找中的元素
                this.sorts = [this.list[j].index]
                // 找到比当前元素小的元素
                if (this.list[j].value < this.list[minIndex].value) {
                    minIndex = j
                    this.min = this.list[j].index
                }
                setTimeout(() => {
                    j++
                    innerLoop()
                }, 800)
            }
            let outLoop = () => {
                if (i >= len - 1) {
                    this.sorted.push(this.list[i].index)
                    return false
                }
                minIndex = i
                this.min = this.list[i].index
                innerLoop()
            }
            outLoop()
        }
    }
}
```

效果如下：


![](https://user-gold-cdn.xitu.io/2020/4/8/17159159e4a164fd?w=541&h=301&f=gif&s=151136)

其他的排序也是同样的套路，将for循环或while循环改写成可以控制的函数形式，然后可能需要稍微修改一下显示逻辑，如果你也有打算写排序文章的话现在就可以给自己加上动图展示了！



# 总结

之前看到这些动图的时候也有想过怎么实现，但是都没有深究，这次业务开发无意中也算找到了其中的一种实现方式，其实核心逻辑很简单，关键是很多时候没有想到可以这么做，这也许是框架带给我们的另一些好处吧。
