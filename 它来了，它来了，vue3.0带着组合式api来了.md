# 开头

9月19号，万众期待的vue3.0如期而至，事实上很多人很早就已经体验过了，但是本人比较懒，再加上没有正式发布也不能在项目上使用，所以一直没去尝试，只在一些零星的文章上看到了它大概会有一些什么新功能，现在它的正式发布意味着已经推荐在新项目上使用它了，毕竟相对于2.0，它的优点还是非常明显的，虽然现在配套的一些工具还没完成，但是在一些小项目上使用应该没有什么问题。

目前我还没有实际的去使用过它，本文是在看完其提案文档、api文档及一些文章后进行总结的，难免会有问题，此外，本文跟其他类似的介绍vue组合式api的文章没有什么区别，如果看过其他的本文可以忽略~。

根据官方的说法，vue3.0的变化包括性能上的改进、更小的 bundle 体积、对 TypeScript 更好的支持、用于处理大规模用例的全新 API，全新的api指的就是本文主要要说的组合式api。

如果你的项目还没有或无法升级到3.0版本，不要遗憾，也是可以使用的，2.x 的版本，可以通过 `@vue/composition-api`插件来在项目中使用组合式api，这个插件主要提供了vue3.0对象里新增的一些接口，在以后升级到3.0，只需把引用从该插件改到vue对象就好了，目前也有一些同时支持2.x和3.x版本的实用的工具库（如`vueuse`，`vue-composable`），所以放心大胆的去使用它吧。

一种新东西的出现肯定有它的原因，组合式api在react里出现的更早，也就是react hooks，vue引入的原因肯定不是因为react有所以我也得有，抄袭就更谈不上了，框架的发展肯定都是奔着更先进的方向去的，比如之前的双向绑定、虚拟DOM等等。

我个人在之前2.x的使用上主要有以下两点问题：

1.对于稍大一点的项目，复用是很重要的，使用这种模块化的框架不复用，那是极大的浪费，在2.x中，vue复用的方式主要是组件、mixin，如果不涉及到模板和样式的逻辑上的复用一般都会用mixin，但是mixin用起来是很痛苦的，因为代码提取到mixin里后你基本想不起它里面有啥了，存在多个mixin你甚至不知道来自于哪个，每次都需要打开mixin文件看，而且很容易会造成重复声明的问题。

2.虽说一个组件最好只做一件事，但是实际上这是比较困难的，组件或多或少都做了好几件事，因为vue的书写方式是选项组织式，数据和数据在一块，方法和方法在一块，其他的属性也是一样，所以一个相对独立的功能被分散在到处，很难区分，提取复用逻辑也很麻烦，这在一个vue文件代码量很大的时候是很可怕的。

组合式api的出现就能解决以上两个问题，此外，它也对TypeScript类型推导更加友好。

在具体使用上，对vue单文件来说，模板部分和样式部分基本和以前没有区别，组合式api主要影响的是逻辑部分。来看一个最简单的例子：

```vue
<template>
	<div @click="increment">{{data.count}}</div>
</template>
<script>
import { reactive } from 'vue'
export default {
    setup() {
        const data = reactive({
            count: 0
        })
        function increment() {
            data.count++
        }
        return {
            data,
            increment
        }
    }
}
</script>
```

实现的效果就是点击数字，数字自增，可以看到很明显的一个变化是多了一个setup函数，之前熟悉的`data`、`methods`属性都不见了，需要注意的是在模板里要用到的数据都要在`setup`函数里显式返回。

例子虽小，但是已经能看出它是怎么解决上面的问题了，比如你想复用这个计数，只需要把：

```js
const data = reactive({
    count: 0
})
function increment() {
    data.count++
}
```

提取成一个函数就可以了，实际上就是把原来分散在各个选项里的东西都放到一起，再加上没有了`this`，这样每个独立的功能都可以提取成函数，为了进行区分，函数名一般以`use`开头，如`useCount`。`setup`函数实际上就成了以类方式组织代码里的`constructor`函数或一般的`init`函数，用来调用各个`use`函数。

如果你还不是很习惯这种方式的话，实际上2.x的选项方式仍然可以继续使用，组合式api会先于选项进行解析，所以组合式api里不能访问选项里的属性，`setup`函数返回的属性可以在选项里通过`this`来访问，但是最好还是不要混用。

# 常用api介绍

## setup

```js
export default {
    setup() {
        return {}
    }
}
```

`setup`函数是新增的一个选项，用来使用组合式api，它会在`beforeCreate`生命周期钩子之前被调用，一些知识点：

1.可以返回一个对象，对象的属性会合并到模板渲染的上下文中；

2.可以返回一个函数；

3.第一个参数是`props`，注意使用`props`不能解构，第二个参数是一个上下文对象，暴露了一些其他常用属性；

## reactive

```js
import { reactive } from 'vue'
const data = reactive({
  count: 0,
})
```

`reactive`函数使一个对象变成响应式，返回的`data`基本相当于以前的`data`选项。

## watchEffect 

```js
import { reactive, watchEffect } from 'vue'
const data = reactive({
  count: 0,
})
watchEffect(() => {
    alert(this.count)
})
```

`watchEffect`用来监听数据的变化，它会立即执行一次，之后会追踪函数里面用到的所有响应式状态，当变化后会重新执行该回调函数。

## computed 

```js
import { reactive, computed } from 'vue'
const data = reactive({
  count: 0,
})
const double = computed(() => {
    return data.count * 2
})
```

相当于之前的computed选项，创建一个依赖于其他状态的状态，需要注意的是返回的是一个对象，称作`ref`：

```js
{
    value: xxx//这个才是你需要的值
}
```

引用的实际值是该对象的`value`属性，为什么要这么做原因很简单，因为说到底`computed`只是个函数，如果返回的是个基本类型的值，比如这里返回的是个0，那么即使后续`data.count`改变了，`double`仍然是0，因为JavaScript是基本类型是按值传递的，而对象是按引用传递的，所以这样获取到的`value`始终是最新的值。

除了可以传`getter`函数，也可以传一个带有`setter`和`getter`属性的对象创建一个可修改的计算值。

## ref

```js
import { ref } from 'vue'
let count = ref(0)
```

除了通过`computed`返回`ref`，也可以直接创建，但是也可以直接通过`reactive`来实现响应式：

```js
let data = reactive({
    count: 0
})
```

`ref`和`reactive`存在一定的相似性，所以需要完全理解它们才能高效的在各种场景下选择不同的方式，它们之间最明显的区别是`ref`使用的时候需要通过`.value`来取值，`reactive`不用。

它们的区别也可以这么理解，`ref`是使某一个数据提供响应能力，而`reactive`是为包含该数据的一整个对象提供响应能力。

在模板里使用`ref`和嵌套在响应式对象里时不需要通过`.value`，会自己解开：

```vue
<template>
	<div>{{count}}</div>
</template>
<script>
let data = reactive({
    double: count * 2
})
</script>
```

除了响应式`ref`还有一个引用DOM元素的`ref`，2.x里面是通过`this.$refs.xxx`来引用，但是在`setup`里面没有`this`，所以也是通过创建一个`ref`来使用：

```vue
<template>
	<div ref="node"></div>
</template>
<script>
import { ref, onMounted } from 'vue'
export default {
    setup() {
        const node = ref(null)
        onMounted(() => {
            console.log(node.value)// 此处就是dom元素
        })
        return {
            node
        }
    }
}
</script>
```



## toRefs

如果你从一个组合函数里返回了响应式对象，在使用函数里把它解构进行使用是会丢失响应性的，解决这个问题你可以把原封不动的返回：

```js
import { reactive } from 'vue'
function useCounter () {
    let data = reactive({
        count: 0
    })
    return data
}
export default {
    setup() {
        // let {count} = useCounter()，这样是不行的
        // return {，这样也是不行的
        //    ...useCounter()
        // }
        return {
            count: useCounter()//这样才行
        }
    }
}
```

除此之外还有一个办法就是使用`toRefs`，`toRefs`会把一个响应式对象的每个属性都转换为一个`ref`：

```js
import { toRefs, reactive } from 'vue'
function useCounter () {
    let data = reactive({
        count: 0
    })
    return toRefs(data)
}
export default {
    setup() {
        let {count} = useCounter()
        return {
            count
        }
    }
}
```

在`setup`函数里返回的数据使用`toRefs`转换一下是有好处的：

```js
import { toRefs, reactive } from 'vue'
export default {
    setup() {
        let data = reactive({
            count: 0
        })
        return {
            data
        }
    }
}
```

如果这样的话在模板里使用需要通过`data.count`来引用`count`的值，如果使用`toRefs`的话可以直接使用`count`：

```vue
<script>
import { toRefs, reactive } from 'vue'
export default {
    setup() {
        let data = reactive({
            count: 0
        })
        return {
            ...toRefs(data)
        }
    }
}
</script>
<template>
	<div>{{count}}</div>
</template>
```



## 生命周期函数

```js
import { onUpdated, onMounted } from 'vue'
export default {
    setup() {
        onUpdated(() => {
            console.log(1)
        }
        onMounted(() => {
            console.log(1)
        }
    }
}
```

只需要将之前的生命周期改成onXXX的形式即可，需要注意的是`created`、`beforeCreate`两个钩子被删除了，生命周期函数只能在`setup`函数里使用。

除此之外，还有一些其他api，建议详细通读一遍官方文档，然后在实际使用中加以巩固。

# 结尾

使用组合式api还是需要一点时间来适应的，首先需要能区分`ref`和`reactive`，不要在基本类型和应用类型之间搞混、响应式和非响应式对象之间迷路，其次就是如何拆分好每一个use函数，组合式api带来了更好的代码组织方式，但也更容易把代码写的更难以维护，比如`setup`函数巨长。

简单总结一下升级思路，`data`选项里的数据通过`reactive`进行声明，通过`...toRefs()`返回；`computed`、`mounted`等选项通过对应的`computed`、`onMounted`等函数来进行替换；`methods`里的函数随便在哪声明，只要在`setup`函数里返回即可。

通过这一套组合式api，使用范围其实已经不限于在`vue`里，完全可以拎出来用在其他地方：

```js
import { reactive, watchEffect } from '@vue/composition-api'
let data = reactive({
    count: 0
})
watchEffect(() => {
    renderTemplate(data)//你的模板渲染函数
})
```

好了，不说了，我要去尝试一下它了。



参考文章

https://vue-composition-api-rfc.netlify.app/zh/
