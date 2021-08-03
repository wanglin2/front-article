---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: jzman
highlight:
---

在通读了vue的官网文档后，我记录下了如下这些相对于2.x的变化之处。

# 1.创建应用实例的变化

之前一般是这样：

```js
let app = new Vue({
	// ...一些选项
	template: '',// 字符串模板
	render: h => h(App)// 单文件情况下
})
let vm = app.$mount('#app')
app === vm// true
```

而现在变成这样：

```js
import { createApp } from 'vue'
import App from './App.vue'
let app = createApp({
    // ...组件选项
})
let app = createApp(App)// 单文件情况下
let vm = app.mount('#app')
app === vm // false
```

改成这样的最主要原因是为了避免对`Vue`的全局配置会影响每个创建的实例。



# 2.`data`选项变化

之前在非组件的情况下创建实例可以使用对象，但是现在所有情况下都只能使用一个返回对象的函数。



# 3.生命周期变化

`beforeDestroy`=>`beforeUnmount`，`destroyed`=>`unmounted`，另外新增了两个生命周期`renderTracked`和`renderTriggered`，用来跟踪虚拟DOM重新渲染。



# 4.事件监听支持多个处理函数

在3.0中`v-on`指令可以绑定多个处理函数：

```vue
<button @click="one(),two(),three($event)"></button>
```

```js
export default {
    methods: {
        one(){},
        two(){},
        three(){}
    }
}
```

绑定多个函数时必须使用内联函数调用方式，即不能只写一个函数名。



# 5.实例多了一个数据选项：`emits`

显式声明该组件能触发的自定义事件，就像`props`属性一样，可以是简单的字符串数组，也可以是对象，同样的，对象类型的话可以用来定义校验，使用方法如下：

```js
export default {
    emits: ['change', 'select'],// 数组类型
    emits: {// 对象类型
        change: null,// 没有验证函数
        select: (arg) => {// 接收this.$emit('select', ..args)的args参数
            return true// 返回true或false代表事件参数是否有效，校验失败事件还是能正常触发，但是控制台会弹出一行警告信息
        }
    },
    methods: {
        emit() {
            this.$emit('change')
            this.$emit('select', 1, 2, 3)
        }
    }
}
```

该声明是可选的。



# 6.新增了`v-is`指令

这个指令用来承担2.x版本里的特殊attribute`is`的部分功能。

在2.x里`is`可用在两个场景下，一是用于动态组件`component`来切换要渲染的组件，二是用于在使用DOM模板时的一些HTML元素的限制，比如`ul`元素里只能出现`li`元素，这样当`ul`里使用自定义组件时浏览器会认为是无效内容，此时可以使用`is`属性：

```html
<ul>
    <!--<my-component></my-component> x这样不行-->
    <li is="my-component"></li>
</ul>
```

而在3.0版本`is`只能用在`component`上，上述功能需要使用`v-is`来代替：

```html
<ul>
    <li v-is="'my-component'"></li>
</ul>
```

注意上述的单引号是必须的。



# 7.未声明的`emits`

因为新增了类似`props`的选项`emits`，如果某些传递给组件的属性并没有在`props`声明，那么可以通过`$attrs`属性来访问，事件监听器也一样：

```html
<!--父组件-->
<sub-component @change="change" @select="select"></sub-component>
```

```js
// 子组件
export default {
    emits: ['change'],
    created(){
        console.log(this.$attrs)// { onSelect: () => {}  }
    },
}
```

另外，在2.x中这些未声明的`props`或`emits`会直接继承到该组件的根节点上，比如：

```html
<!--父组件-->
<sub-component class="warn"></sub-component>
```

```html
<!--子组件-->
<div class="info"></div>
```

```html
<!--实际渲染结果-->
<div class="info warn"></div>
```

但在3.x中组件支持多个根节点，当出现多个根节点时，属性将不会主动继承，需要手动给需要继承属性的组件进行绑定，如果一个都没绑定的话vue会给出警告：

```vue
<template>
	<my-momponent class="bar" @change="change"></my-component>
</template>
```

```vue
<template>
	<div v-bind="$attrs"></div>
	<div></div>
</template>
```



# 8.`v-model`的变化

在2.x中给一个组件自定义`v-model`一般是这样的：

```js
export default {
    model: {// v-model默认是利用名为value的prop及input事件，可使用model选项来修改
        prop: 'checked',
        event: 'change'
    },
    props: {
        checked: Boolean
    },
    methods: {
        emit() {
            this.$emit('change', true)
        }
    }
}
/*
<my-component v-model="checked"></my-component>
*/
```

在3.x中`v-model`指令多了一个参数，比如：`v-model:value="value"`，所以就不需要使用`model`选项了，`vue`会直接利用`value`属性及事件名`update:value`：

```js
export default {
    props: {
        checked: Boolean
    },
    methods: {
        emit() {
            this.$emit('update:checked', true)
        }
    }
}
/*
<my-component v-model:checked="checked"></my-component>
*/
```

当然你也可以省略`value`，这样会默认绑定到名为`modelValue`的`prop`上：

```js
export default {
    props: {
        modelValue: Boolean
    },
    methods: {
        emit() {
            this.$emit('update:modelValue', true)
        }
    }
}
/*
<my-component v-model="checked"></my-component>
*/
```

这样的一个好处是可以绑定多个`v-model`：

```js
export default {
    props: {
        modelValue: Number,
        checked: Boolean,
        value: String
    },
    methods: {
        emit() {
            this.$emit('update:modelValue', 1)
            this.$emit('update:checked', true)
            this.$emit('update:value', 'abc')
        }
    }
}
/*
<my-component v-model="count" v-model:checked="checked" v-model:value="value"></my-component>
*/
```

最后一点是3.x支持自定义`v-model`的修饰符，大致就是修饰符也能通过`props`获取到，然后可以根据修饰符存在与否进行一些对应的数据格式化操作：

```js
/*
<my-component v-model.double="count" v-model:count2.double="count2"></my-component>
*/

export default {
    props: {
        modelValue: Number,
        count2: Number,
        modelModifiers: Object,// 没有参数的v-model的修饰符数据，名称为modelModifiers，对象格式：{double: true}，如果修饰符不存在为undefined
        count2Modifiers: Object// 带参数的v-model的修饰符数据名称为：参数+"Modifiers"，对象格式：{double: true}，如果修饰符不存在为undefined
    },
    methods: {
        emit() {
            // 在这里可以根据modelModifiers和count2Modifiers的值来判断是否要进行一些数据操作
            this.$emit('update:modelValue', xxx)
            this.$emit('update:value', xxx)
        }
    }
}
```



# 9.响应式`provide/reject`

`provide/reject`默认是没有响应性的，父组件的`provide`值变化了，子组件使用`reject`接收的值不会相应更新，在2.0中，想要使它变成可响应比较麻烦，下面这种方式是不行的，父组件的`count`变化了子组件的`count`并不会变化：

```vue
<template>
	<div>{{count}}</div>
</template>
<script>
export default {
    inject: ['count']
}
</script>
```

```js
export default {
    provide() {
        return {
            count: this.count
        }
    },
    data: {
        count: 0
    }
}
```

`vue`2.x文档里有个提示：

>  提示：`provide` 和 `inject` 绑定并不是可响应的。这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的 property 还是可响应的。

后半句我的理解是如果`provide`返回的对象的属性值是一个可响应对象的话，那么是可以的，比如：

```js
export default {
    provide() {
        return {
            count: this.countObj
        }
    },
    data: {
        countObj: {
            value: 0
        }
    }
}
```

这样的话修改`countObj.value`的值，子组件会相应的更新，但是如果想像上面那样依赖`count`的值，即使你使用`computed`也是不行的：

```js
export default {
    provide() {
        return {
            count: this.countObj
        }
    },
    data: {
        count: 0
    },
    computed: {
        countObj() {
            return {
                value: this.count
            };
        }
    }
}
```

那么就只能使用`watch`和`Vue.observable`方法来配合实现：

```js
let countState = Vue.observable({ value: 0 });

export default {
    provide() {
        return {
            count: countState
        };
    },
    data: {
        count: 0
    },
    watch: {
        count(newVal) {
            countState.value = newVal
        }
    }
}
```

但是在3.x中就比较简单了，可以直接使用组合式api里的`computed`方法：

```js
import {computed} from 'vue'

export default {
    provide() {
        return {
            count: computed(() => {
                return this.count
            })
        };
    },
    data: {
        count: 0
    }
}
```

后面这些在子组件里使用的时候都需要访问`count.value`属性。



# 10.异步组件

在2.x中，异步组件一般使用如下方法定义：

```js
// 全局
Vue.component('async-component', () => import('./my-async-component'))
// 局部
{
    components: {
        'async-component': () => import('./my-async-component')
    }
}
```

在3.x中新增了一个函数`defineAsyncComponent`来做这件事情：

```js
import { defineAsyncComponent } from 'vue'
const AsyncComp = defineAsyncComponent(() =>
  import('./components/AsyncComponent.vue')
)

// 全局
app.component('async-component', AsyncComp)
// 组件内
{
    components: {
		'AsyncComponent': AsyncComp
    }
}
```



# 11.过渡class的变化

3.x和2.x一样，仍然有6个class，意义完全一样，唯一的变化只有`v-enter`->`v-enter-from`、`v-leave`->`v-leave-from`两个名字以及`enter-class`->`enter-from-class`、`leave-class`->`leave-from-class`两个自定义类名的变化。



# 12.自定义指令变化

在2.x中提供了`bind`、`inserted`、`update`、`componentUpdated`、`unbind`五个指令，在3.x中新增了一个，一共有六个：

`beforeMount`（指令第一次绑定到元素并且还未挂载到父组件上调用，对应于`bind`，用来进行一些初始化操作）

`mounted`（绑定元素的父组件被挂载时调用，对应`inserted`，但是`inserted`的描述里说仅保证父组件存在但不一定被插入到文档中，`mounted`的描述里没有这句话）

`beforeUpdate`（在包含该组件的虚拟节点被更新前调用，对应`update`）

`updated`（在包含该组件的虚拟节点及其所有子组件的虚拟节点都更新后调用，对应`componentUpdated`）

`beforeUnmount`（在卸载绑定元素的父组件前调用，为新增钩子）

`unmounted`（指令与元素解除绑定且父组件已经卸载时调用，对应`unbind`）

总的来说改名后的自定义钩子和vue本身的生命周期钩子趋于一致。



# 13.新增Teleport

在2.x中有一个常见的痛点：

```html
<div>
    <Dialog></Dialog>
    <Loading></Loading>
</div>
```

在上述组件里包含了两个子组件，像这种弹窗或loading组件一般都是希望它们的DOM节点直接挂在body元素下，这样在样式尤其是层级上比较好控制，但是实际渲染出来是在这个div节点下的，那么就只能把这两个组件移到body下，但是逻辑上这两个组件又是属于该组件，所以就比较不爽。

在3.x中新增了`teleport`组件可以用来解决这个问题：

```html
<div>
    <teleport to="body">
    	<Dialog></Dialog>
    </teleport>
    <teleport to="#xxx">
    	<Loading></Loading>
    </teleport>
</div>
```

直接将需要提到外层的组件放到`teleport`标签里，通过`to`属性来指定要挂载到的元素，`to`可以是有效的元素查询选择器，比如id选择器，类选择器等。



# 14.渲染函数的变化

在2.x中使用`render`函数需要使用注入的方法来创建虚拟节点，示例如下：

```js
Vue.component('my-component', {
    render(createElement) {
        return createElement('div', '我是文本')
    }
})
```

在3.x中使用vue对象的静态方法来实现：

```js
Vue.component('my-component', {
    render() {
        return Vue.h('div', '我是文本')
    }
})
```

`h`函数接收的参数和`createElement`基本都是`tag`、`props`、`children`，但是`props`结构发生了很大变化，比如事件绑定：

```js
Vue.component('my-component', {
    render(createElement) {
        return createElement('div', {
            on: {
                'click': this.clickCallback
            }
        })
    }
})
Vue.component('my-component', {
    render() {
        return Vue.h('div', {
            onClick: this.clickCallback
        })
    }
})
```

在2.x中不支持`v-model`，`3.x`中已经支持了，其他变化之处也很大，需要读者自己去详细了解，这一节的官方文档应该还需要完善，`props`的具体描述并未看到，但是大致的改变就是更加扁平化，比如2.x的结构：

```js
{
  class: ['xxx', 'xxx'],
  style: { color: '#fff' },
  attrs: { id: 'xxx' },
  domProps: { innerHTML: 'xxx' },
  on: { click: onClick },
  key: 'xxx'
}
```

在3.x中变成这样：

```js
{
    class: ['xxx', 'xxx'],
  	style: { color: '#fff' },
    id: 'xxx',
    innerHTML: 'xxx',
    onClick: onClick,
    key: 'xxx'
}
```



# 15.插件开发的变化

在2.x中注册插件时调用插件的`install`方法时会注入`Vue`对象和参数对象，在3.x中因为将`Vue`上的全局属性和方法都移到了由`createApp`方法创建的实例`app`上，所以注册插件需要在`createApp`方法执行之后，另外注入功能时也会有一些细微的变化。



# 16.去掉了过滤器选项

在3.x中可以使用方法来实现该功能。



# 17.响应性原理变化

众所周知，在2.x中是使用`Object.defineProperty`来实现数据响应的，在3.x默认使用`ES6`的`Proxy`来实现，并且在`IE`浏览器上使用`Object.defineProperty`进行降级。

另外在3.x中增加了很多可以用来给数据增加响应行功能的方法，比如：

```js
// 非原始值
import {reactive} from 'vue'
// 响应式状态
const state = reactive({
    count: 1
})

// 原始值
import {ref} from 'vue'

// 响应式状态
const count = ref(0)
console.log(count.value)
```

此外还新增了`computed`、`watch`等等可以直接使用的方法，这些方法一般在使用组合式api的情况下使用。



# 18.新增响应式和组合式api

这个已经有非常多的文章详细的介绍它了，可以在掘金上搜索或直接去官网上看，此处不赘述。



# 19.ref的变化

在2.x中`ref`是用来访问组件实例或者是DOM元素的属性：

```html
<div ref="div">
    <ul>
    	<li v-for="item in list" ref="liList"></li>
    </ul>
	<MyComponent ref="component"></MyComponent>
</div>
```

```js
export default {
    mounted() {
        console.log(this.$refs.div, this.$refs.component)
        console.log(this.$refs.liList)// liList会自动是一个数组
    }
}
```

其中当在循环里使用`ref`是不明确的，尤其是存在嵌套循环，所以在3.x中`ref`支持绑定到一个函数：

```html
<div ref="div">
    <ul>
    	<li v-for="item in list" :ref="setLiList"></li>
    </ul>
	<MyComponent ref="component"></MyComponent>
</div>
```

```js
export default {
    data() {
        return {
            liList: []
        }
    }
    mounted() {
        console.log(this.$refs.div, this.$refs.component)
        console.log(this.liList)
    },
    methods: {
        setLiList(el) {
            this.liList.push(el)
        }
    }
}
```



# 20.Vue-Router变化

`vue-router`升级到了新版本，安装命令为：`npm install vue-router@4`。

接下来使用一个简单的例子看一下2.x和3.x的区别：

```js
// 2.x
import Vue from 'vue'
import VueRouter from 'vue-router'
Vue.use(VueRouter)
const routes = [
    // ...
]
const router = new VueRouter({
    // ...一些选项配置
    routes
})
const app = new Vue({
    router
}).$mount('#app')
```

```js
// 3.x
import Vue from 'vue'
import VueRouter from 'vue-router@4'
const routes = [
    // ...
]
const router = VueRouter.createRouter({
    // ...一些选项配置
    routes
})
const app = Vue.createApp({})
app.use(router)
app.mount('#app')
```

除了创建路由的方式有变化外，其他也有很多细节变化，以及如何在组合式api中使用，笔者没看完，请自行阅读`vue-router`文档。



# 21.Vuex变化

除路由外，官方的状态管理库`vuex`也配套升级了新版本，安装：`npm install vuex@next --save`。

同样以一个十分简单的例子看一下初始化的变化：

```js
// 2.x
import Vue from 'vue'
import Vuex from 'vuex'
Vue.use(Vuex)
const store = new Vuex.Store({
	state: {
        count: 0
    },
    mutations: {},
    actions: {},
    // ...
})
new Vue({
    store
})
```

```js
// 3.x
import {createApp} from 'vue'
import {createStore} from 'vuex'
const store = createStore({
    state() {
        return {
            count:0
        }
    },
    mutations: {},
    actions: {},
    // ...
})
const app = createApp({})
app.use(store)
```

`vuex`的api基本没有大的变化，更多的可以去了解一下如何在组合式api中使用。



# 22.其他变化一览

- `$attrs`里也包含`class`和`style`

- 移除了`$children`，如需访问子组件请使用`ref`

- 移除了`Vue`实例的`$on`、`$emit`、`$once`方法，之前常见的使用方式现在需要自己实现或者使用其他事件库：

  ```js
  import Vue from 'vue'
  Vue.prototype.$bus = new Vue()
  ```

  这一常见操作完全被干掉了，因为现在要增加全局功能的话需要通过应用实例的`globalProperties`属性：

  ```js
  app.config.globalProperties.$bus = new OtherEvent()
  ```

- 支持多个根节点：

  ```html
  <template>
  	<div></div>
      <Header></Header>
  </template>
  ```

- 一些2.x的全局api都改成使用导出的方式进行使用，比如：`import {nextTick} from 'vue'`，这样可以利于构建工具去掉无用代码

- 使用`template`组件进行循环操作时，`key`属性可以需要直接设置在`template`标签上：

  ```html
  <template>
  	<template v-for="item in list" :key="item.id"></template>
  </template>
  ```



以上大部分内容在`vue`的官方升级指南中也提到了，有兴趣的也可以直接去看官方文档：[https://v3.vuejs.org/guide/migration/introduction.html](https://v3.vuejs.org/guide/migration/introduction.html)，以及中文版：[https://v3.cn.vuejs.org/guide/migration/introduction.html](https://v3.cn.vuejs.org/guide/migration/introduction.html)，如果有任何错误的话欢迎指出。

