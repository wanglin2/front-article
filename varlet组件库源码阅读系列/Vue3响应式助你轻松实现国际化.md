> 本文为Varlet组件库源码主题阅读系列第十篇，也是最后一篇，读完本篇，可以了解到如何通过创建一个Vue3响应式对象就可以轻松实现国际化的需求。

[Varlet](https://github.com/varletjs/varlet)组件库支持多语言切换，使用也很简单：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/983fc456a42e4d85b3037cbbc1825cb3~tplv-k3u1fbpfcp-zoom-1.image)


本文会从源码角度来看一下它是如何实现的，希望给你提供一点思路。

如上图所示，主要就是提供了三个方法，不过在了解具体实现前先看一下组件中是如何使用多语言的。

# 组件中是如何使用多语言的

比如`Pagination`组件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a40c7d8becb04d31861f10d23155ce04~tplv-k3u1fbpfcp-zoom-1.image)


其中的`条`和`页`就需要支持多语言，源码中是这样的：

```html
<template>
	<span>{{ size }}{{ pack.paginationItem }} / {{ pack.paginationPage }}</span>
</template>
```

```ts
import { pack } from '../locale'
```

就是这么简单。

# pack是什么

上一小节中的`pack`是什么呢？为什么使用它就能随着多语言切换进行切换？其实很简单，`pack`就是一个`Vue3`的响应式对象：

```ts
const pack: Ref<Partial<T>> = ref({})
```

它的值就是多语言数据，响应式对象改变了模板显然会自动更新，那么切换语言也只需要修改`pack`的值即可。

# 语言扩展

先来看看语言扩展的函数`add`，一个多语言文件如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2cf26879d8f41bead836d3eea3d2818~tplv-k3u1fbpfcp-zoom-1.image)


其实就是一个对象。

`add`方法接收两个参数，一个多语言名称，另一个就是多语言对象：

```ts
// 保存所有多语言数据
const packs: Record<string, Partial<T>> = {}
// 扩展多语言
const add = (lang: string, pack: Partial<T> & { lang?: string }) => {
    pack.lang = lang
    packs[lang] = pack
}
```

非常简单，就是以多语言名称为`key`，多语言对象为`value`保存到`packs`对象上，至于`pack.lang = lang`这个是用来干什么的，笔者反正没看出来。

# 切换语言

扩展完语言后就可以使用`use`方法进行切换，接收一个参数，即要切换到的语言名称：

```ts
// 切换语言
const use = (lang: string) => {
    if (!packs[lang]) {
        console.warn(`The ${lang} does not exist. You can mount a language package using the add method`)
        return {}
    }

    pack.value = packs[lang]
}
```

同样非常简单，先判断一下要切换到的语言是否存在，存在的话就将该语言对象数据赋值给响应式变量`pack`，那么使用了该响应式变量的所有模板都会自动更新，达到多语言切换的效果。

# 语言合并

`Varlet`还提供了一个`merge`方法来合并某个语言数据，比如想覆盖多语言的某个字段的默认翻译那么就可以通过这个方法来实现：

```ts
Locale.merge('en-US', {
  dialogTitle: 'Hello'
})
```

```ts
// 语言合并
const merge = (lang: string, pack: Partial<T>) => {
    if (!packs[lang]) {
        console.warn(`The ${lang} does not exist. You can mount a language package using the add method`)
        return
    }

    packs[lang] = { ...packs[lang], ...pack }

    use(lang)
}
```

接收两个参数，要合并的多语言名称，以及要合并的对象，合并也就是覆盖修改保存在`packs`对象上的多语言对象，需要注意的是合并操作后还会自动切换到该语言。

# 总结

可以看到使用`Vue3`的响应式对象来实现国际化是非常简单的，各位如果有此需求的话不妨考虑以上实现。



