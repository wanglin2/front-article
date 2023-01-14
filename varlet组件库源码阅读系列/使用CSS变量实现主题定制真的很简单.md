> 本文为Varlet组件库源码主题阅读系列第六篇，`Varlet`支持自定义主题及暗黑模式，本篇文章我们来详细看一下这两者的实现。



# 主题定制

`Varlet`是通过`css`变量来组织样式的，什么是`css`变量呢，其实很简单，首先声明自定义的`css`属性，随便声明在哪个元素上都可以，不过只有该元素的后代才能使用，所以如果要声明全局所有元素都能使用的话，可以设置到根伪类`:root`下：

```css
:root {
  --main-bg-color: red;
}
```

如代码所示，`css`变量的自定义属性是有要求的，需要以`--`开头。

然后在任何需要使用该样式的元素上通过`var()`函数调用即可：

```css
div {
  background-color: var(--main-bg-color);
}
```

只要更改了`--main-bg-color`属性的值，所有使用该样式变量的地方都会更新，所以主题定制靠的就是这个。

`Varlet`组件的样式变量总体分为两种：基本的、组件自身的。

公共的基本样式变量定义在`varlet-ui/src/styles/`目录下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b59d00ce658645c59ee8015210c3690c~tplv-k3u1fbpfcp-zoom-1.image)


每个组件都会引入这个文件，比如`Button`组件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/205eb5fddbee48c89c8b925cbbd0357d~tplv-k3u1fbpfcp-zoom-1.image)


除此之外每个组件也会有自身的变量，同样比如`Button`组件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd825ff48f73496eb08af9c037aff6e8~tplv-k3u1fbpfcp-zoom-1.image)


想要修改默认的值也很简单，直接覆盖即可。运行时动态更新样式也可以直接修改根节点的样式变量，此外`Varlet`也提供了一个组件来帮我们做这件事，接下来看看这个组件是怎么实现的。

## 组件式调用

组件式调用可以有范围性的定制组件样式，避免全局污染，使用示例：

```vue
<script setup>
import { ref, reactive } from 'vue'

const state = reactive({
  score: 5,
})

const styleVars = ref({
  '--rate-primary-color': 'var(--color-success)',
})
</script>

<template>
  <var-style-provider :style-vars="styleVars">
    <var-rate v-model="state.score" />
  </var-style-provider>
</template>
```

`StyleProvider`组件源码如下：

```vue
<script lang="ts">
import { defineComponent, h } from 'vue'
import { formatStyleVars } from '../utils/elements'
import { call, createNamespace } from '../utils/components'

const { n } = createNamespace('style-provider')

export default defineComponent({
  name: 'VarStyleProvider',
  props: {
    styleVars: {
      type: Object,
      default: () => ({}),
    },
  },
  setup(props, { slots }) {
    return () =>
      h(
        'div',
        {
          class: n(),
          style: formatStyleVars(props.styleVars),
        },
        call(slots.default)
      )
  },
})
</script>
```

实现很简单，就是创建一个`div`元素来包裹组件，然后将`css`变量设置到该`div`上，这样这些`css`变量只会影响它的子孙元素。

## 函数式调用

除了使用组件，也可以通过函数的方式使用，但是只能全局更新样式：

```vue
<script setup>
import { StyleProvider } from '@varlet/ui'

let rootStyleVars = null

const darkTheme = {
  '--color-primary': '#3f51b5'
}

const toggleRootTheme = () => {
  rootStyleVars = rootStyleVars ? null : darkTheme
  StyleProvider(rootStyleVars)
}
</script>

<template>
  <var-button type="primary" block @click="toggleRootTheme">切换根节点样式变量</var-button>
</template>
```

`StyleProvider`函数如下：

```ts
const mountedVarKeys: string[] = []

function StyleProvider(styleVars: StyleVars | null = {}) {
    // 删除之前设置的css变量
    mountedVarKeys.forEach((key) => document.documentElement.style.removeProperty(key))
    mountedVarKeys.length = 0
    // 将css变量设置到根元素上，并且添加到mountedVarKeys数组
    const styles: StyleVars = formatStyleVars(styleVars)
    Object.entries(styles).forEach(([key, value]) => {
        document.documentElement.style.setProperty(key, value)
        mountedVarKeys.push(key)
    })
}
```

实现也非常简单，直接将`css`变量设置到`html`节点上，同时会添加到一个数组里，用于删除操作。



# 暗黑模式

`Varlet`内置提供了暗黑模式的支持，使用方式为：

```vue
<script setup>
import dark from '@varlet/ui/es/themes/dark'
import { StyleProvider } from '@varlet/ui'

let currentTheme = null

const toggleTheme = () => {
  currentTheme = currentTheme ? null : dark
  StyleProvider(currentTheme)
}
</script>

<template>
  <var-button block @click="toggleTheme">切换主题</var-button>
</template>
```

也调用了前面的`StyleProvider`方法，所以实现原理也是通过`css`变量，其实就是内置了一套暗黑模式的`css`变量：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f155f55332b241ff9be35778baa4ef34~tplv-k3u1fbpfcp-zoom-1.image)



# 总结

可以发现使用`css`变量来实现主题定制和暗黑模式是非常简单的，兼容性也非常好，各位如果有涉及到换肤的需求都可以优先考虑使用。
