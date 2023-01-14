> 本文为Varlet组件库源码主题阅读系列第三篇，读完本篇，你可以了解到如何将`svg`图标转换成字体图标文件，以及如何设计一个简洁的Vue图标组件。

`Varlet`提供了一些常用的图标，图标都来自 [Material Design Icon](https://materialdesignicons.com/)。



# 转换SVG为字体图标

图标原文件是`svg`格式的，但最后是以字体图标的方式使用，所以需要进行一个转换操作。

处理图标的是一个单独的包，目录为`/packages/varlet-icons/`，提供了可执行文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33caa6fce82b401e9c80005a107ba5eb~tplv-k3u1fbpfcp-zoom-1.image)


打包命令为：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db88a2a2bf1940d0b21baeae78a0ffa2~tplv-k3u1fbpfcp-zoom-1.image)


接下来详细看一下`lib/index.js`文件都做了哪些事情。

```js
// lib/index.js
const commander = require('commander')

commander.command('build').description('Build varlet icons from svg').action(build)
commander.parse()
```

使用命令行交互工具[commander](https://github.com/tj/commander.js)提供了一个`build`命令，处理函数是`build`：

```js
// lib/index.js
const webfont = require('webfont').default
const { resolve } = require('path')
const CWD = process.cwd()
const SVG_DIR = resolve(CWD, 'svg')// svg图标目录
const config = require(resolve(CWD, 'varlet-icons.config.js'))// 配置文件

async function build() {
    // 从配置文件里取出相关配置
    const { base64, publicPath, namespace, fontName, fileName, fontWeight = 'normal', fontStyle = 'normal' } = config

    const { ttf, woff, woff2 } = await webfont({
        files: `${SVG_DIR}/*.svg`,// 要转换的svg图标
        fontName,// 字体名称，也就是css的font-family
        formats: ['ttf', 'woff', 'woff2'],// 要生成的字体图标类型
        fontHeight: 512,// 输出的字体高度（默认为最高输入图标的高度）
        descent: 64,// 修复字体的baseline
    })
}
```

`varlet-icons`的配置如下：

```js
// varlet-icons.config.js
module.exports = {
  namespace: 'var-icon',// css类名的命名空间
  fileName: 'varlet-icons',// 生成的文件名
  fontName: 'varlet-icons',// 字体名
  base64: true,
}
```

核心就是使用[webfont](https://github.com/itgalaxy/webfont)包将多个`svg`文件转换成字体文件，`webfont`的工作原理可以通过其文档上的依赖描述大致看出：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/638f9d547f254fbd8af5b462ad3a4d0c~tplv-k3u1fbpfcp-zoom-1.image)


使用[svgicons2svgfont](https://github.com/nfroidure/svgicons2svgfont)包将多个`svg`文件转换成一个`svg`字体文件，何为`svg`字体呢，就是类似下面这样的：

```svg
<?xml version="1.0" standalone="no"?> 
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd" >
<svg xmlns="http://www.w3.org/2000/svg">
<defs>
  <font id="geniconsfont" horiz-adv-x="24">
    <font-face font-family="geniconsfont"
      units-per-em="150" ascent="150"
      descent="0" />
    <missing-glyph horiz-adv-x="0" />
    <glyph glyph-name="播放"
      unicode="&#xE001;"
      horiz-adv-x="150" d=" M110.2968703125 71.0175129375C113.2343765625 72.853453875 113.2343765625 77.131555125 110.2968703125 78.967497L63.4218703125 111.779997C60.2997703125 113.7313095 56.25 111.486733875 56.25 107.8050045L56.25 42.1800046875C56.25 38.49827625 60.2997703125 36.253700625 63.4218703125 38.205013125L110.2968703125 71.0175129375z" />
    <glyph glyph-name="地址"
      unicode="&#xE002;"
      horiz-adv-x="150" d=" M0 150H150V0H0V150z M75 112.5C92.3295454375 112.5 106.25 98.9382045 106.25 83.119658125C106.25 58.529589375 83.238636375 42.758655625 75 37.5C66.7613636875 42.758655625 43.75 58.529589375 43.75 83.119658125C43.75 98.9382045 57.6704545625 112.5 75 112.5zM75 93.75C68.1250000625 93.75 62.5 88.1249999375 62.5 81.25C62.5 74.3750000625 68.1249999375 68.75 75 68.75C81.8749999375 68.75 87.5 74.3750000625 87.5 81.25C87.5 88.1249999375 81.8749999375 93.75 75 93.75z" />
  </font>
</defs>
</svg>
```

每个单独的`svg`文件都会转换成上面的一个`glyph`元素，所以上面这段`svg`定义了一个名为`geniconsfont`的字体，包含两个字符图形，我们可以通过`glyph`上定义的`Unicode`码来使用该字形，详细了解`svg`字体请阅读[SVG_fonts](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/SVG_fonts)。

同一个`Unicode`在前端的`html`、`css`、`js`中使用的格式是有所不同的，在`html/svg`中，格式为`&#dddd;`或`&#xhhhh;`，`&#`代表后面是四位`10`进制数值，`&#x`代表后面是四位`16`进制数值；在`css`中，格式为`\hhhh`，以反斜杠开头；在`js`中，格式为`\uhhhh`，以`\u`开头。

转换成`svg`字体后再使用几个字体转换库分别转换成各种类型的字体文件即可。

到这里字体文件就生成好了，不过事情并没有结束。

```js
// lib/index.js
const { writeFile, ensureDir, removeSync, readdirSync } = require('fs-extra')

const DIST_DIR = resolve(CWD, 'dist')// 打包的输出目录
const FONTS_DIR = resolve(DIST_DIR, 'fonts')// 输出的字体文件目录
const CSS_DIR = resolve(DIST_DIR, 'css')// 输出的css文件目录

// 先删除输出目录
removeSync(DIST_DIR)
// 创建输出目录
await Promise.all([ensureDir(FONTS_DIR), ensureDir(CSS_DIR)])
```

清空上次的成果物，创建指定目录，继续：

```js
// lib/index.js
const icons = readdirSync(SVG_DIR).map((svgName) => {
    const i = svgName.indexOf('-')
    const extIndex = svgName.lastIndexOf('.')

    return {
        name: svgName.slice(i + 1, extIndex),// 图标的名称
        pointCode: svgName.slice(1, i),// 图标的代码
    }
})

const iconNames = icons.map((iconName) => `  "${iconName.name}"`)
```

读取`svg`文件目录，遍历所有`svg`文件，从文件名中取出图标名称和图标代码。`svg`文件的名称是有固定格式的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4667f6c9d920427e9ce2a2630e38881b~tplv-k3u1fbpfcp-zoom-1.image)


`uFxxx`是图标的`Unicode`代码，后面的是图标名称，名称也就是我们最终使用时候的`css`类名，而这个`Unicode`实际上映射的就是字体中的某个图形，**字体其实就是一个“编码-字形（glyph）”映射表**，比如最终生成的`css`里的这个`css`类名：

```css
.var-icon-checkbox-marked-circle::before {
  content: "\F000";
}
```

`var-icon`是命名空间，防止冲突，通过伪元素显示`Unicode`为`F000`的字符。

这个约定是[svgicons2svgfont](https://github.com/nfroidure/svgicons2svgfont#cli-interface)规定的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0feef51cf7c143689cf179e44ab537ae~tplv-k3u1fbpfcp-zoom-1.image)


如果我们不自定义图标的`Unicode`，那么会默认从`E001`开始，在`Unicode`中，`E000-F8FF`的区间没有定义字符，用于给我们自行使用[private-use-area](https://unicode-table.com/cn/blocks/private-use-area/)：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75745942e2ca40d39c1519dd611d5a59~tplv-k3u1fbpfcp-zoom-1.image)


接下来就是生成`css`文件的内容了：

```js
// lib/index.js

  // commonjs格式：导出所有图标的css类名
  const indexTemplate = `\
module.exports = [
${iconNames.join(',\n')}
]
`

  // esm格式：导出所有图标的css类名
  const indexESMTemplate = `\
export default [
${iconNames.join(',\n')}
]
`

  // css文件的内容
  const cssTemplate = `\
@font-face {
  font-family: "${fontName}";
  src: url("${
    base64
      ? `data:application/font-woff2;charset=utf-8;base64,${Buffer.from(woff2).toString('base64')}`
      : `${publicPath}${fileName}-webfont.woff2`
  }") format("woff2"),
    url("${
      base64
        ? `data:application/font-woff;charset=utf-8;base64,${woff.toString('base64')}`
        : `${publicPath}${fileName}-webfont.woff`
    }") format("woff"),
    url("${
      base64
        ? `data:font/truetype;charset=utf-8;base64,${ttf.toString('base64')}`
        : `${publicPath}${fileName}-webfont.ttf`
    }") format("truetype");
  font-weight: ${fontWeight};
  font-style: ${fontStyle};
}

.${namespace}--set,
.${namespace}--set::before {
  position: relative;
  display: inline-block;
  font: normal normal normal 14px/1 "${fontName}";
  font-size: inherit;
  text-rendering: auto;
  -webkit-font-smoothing: antialiased;
}

${icons
  .map((icon) => {
    return `.${namespace}-${icon.name}::before {
  content: "\\${icon.pointCode}";
}`
  })
  .join('\n\n')}
`
```

很简单，拼接生成导出`js`文件及`css`文件的内容，最后写入文件即可：

```js
// lib/index.js
await Promise.all([
    writeFile(resolve(FONTS_DIR, `${fileName}-webfont.ttf`), ttf),
    writeFile(resolve(FONTS_DIR, `${fileName}-webfont.woff`), woff),
    writeFile(resolve(FONTS_DIR, `${fileName}-webfont.woff2`), woff2),
    writeFile(resolve(CSS_DIR, `${fileName}.css`), cssTemplate),
    writeFile(resolve(CSS_DIR, `${fileName}.less`), cssTemplate),
    writeFile(resolve(DIST_DIR, 'index.js'), indexTemplate),
    writeFile(resolve(DIST_DIR, 'index.esm.js'), indexESMTemplate),
])
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef0c56108763494e857fe74cca43b97e~tplv-k3u1fbpfcp-zoom-1.image)


我们只要引入`varlet-icons.css`或`less`文件即可使用图标。



# 图标组件

字体图标可以在任何元素上面直接通过对应的类名使用，不过`Varlet`也提供了一个图标组件[Icon](https://varlet-varletjs.vercel.app/#/zh-CN/icon)，支持字体图标也支持传入图片：

```html
<var-icon name="checkbox-marked-circle" />
<var-icon name="https://varlet-varletjs.vercel.app/cat.jpg" />
```

实现也很简单：

```html
<template>
  <component
    :is="isURL(name) ? 'img' : 'i'"
    :class="
      classes(
        n(),
        [isURL(name), n('image'), `${namespace}-${nextName}`],
      )
    "
    :src="isURL(name) ? nextName : null"
  />
</template>
```

通过`component`动态组件，根据传入的`name`属性判断是渲染`img`标签还是`i`标签，图片的话`nextName`就是图片`url`，否则`nextName`就是图标类名。

`n`方法用来拼接[BEM](https://bemcss.com/)风格的`css`类名，`classes`方法主要是用来支持三元表达式，所以上面的：

```
[isURL(name), n('image'), `${namespace}-${nextName}`]
```

其实是个三元表达式，为什么不直接使用三元表达式呢，我也不知道，可能是更方便一点吧。

```ts
const { n, classes } = createNamespace('icon')

export function createNamespace(name: string) {
  const namespace = `var-${name}`

  // 返回BEM风格的类名
  const createBEM = (suffix?: string): string => {
    if (!suffix) return namespace

    return suffix.startsWith('--') ? `${namespace}${suffix}` : `${namespace}__${suffix}`
  }

  // 处理css类数组
  const classes = (...classes: Classes): any[] => {
    return classes.map((className) => {
      if (isArray(className)) {
        const [condition, truthy, falsy = null] = className
        return condition ? truthy : falsy
      }

      return className
    })
  }

  return {
    n: createBEM,
    classes,
  }
}
```

支持设置图标大小：

```html
<var-icon name="checkbox-marked-circle" :size="26"/>
```

如果是图片则设置宽高，否则设置字号：

```html
<template>
  <component
    :style="{
      width: isURL(name) ? toSizeUnit(size) : null,
      height: isURL(name) ? toSizeUnit(size) : null,
      fontSize: toSizeUnit(size),
    }"
  />
</template>
```

支持设置颜色，当然只支持字体图标：

```html
<var-icon name="checkbox-marked-circle" color="#2979ff" />
```

```html
<template>
  <component
    :style="{
      color,
    }"
  />
</template>
```

支持图标切换动画，当设置了 `transition(ms)` 并通过图标的 `name` 切换图标时，可以触发切换动画：

```html
<script setup>
import { ref } from 'vue'

const name = ref('information')

const toggle = () => {
  name.value = name.value === 'information' 
    ? 'checkbox-marked-circle' 
    : 'information'
}
</script>

<template>
  <var-icon 
    :name="name" 
    :transition="300" 
    @click="toggle"
  />
</template>
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c4d9e04b0d6404ebd6bb236f3018066~tplv-k3u1fbpfcp-zoom-1.image)

具体的实现是监听`name`属性变化，然后添加一个改变元素属性的`css`类名，具体到这里是添加了一个设置缩小为`0`的类名`--shrinking`：

```less
.var-icon {
    &--shrinking {
        transform: scale(0);
    }
}
```

然后通过`css`的`transition`设置过渡属性，这样就会以动画的方式缩小为`0`，动画结束后再更新`nextName`为`name`属性的值，也就是变成新的图标，再把这个`css`类名去掉，则又会以动画的方式恢复为原来大小。

```html
<template>
  <component
    :class="
      classes(
        [shrinking, n('--shrinking')]
      )
    "
    :style="{
      transition: `transform ${toNumber(transition)}ms`,
    }"
  />
</template>
```

```ts
const nextName: Ref<string | undefined> = ref('')
const shrinking: Ref<boolean> = ref(false)

const handleNameChange = async (newName: string | undefined, oldName: string | undefined) => {
    const { transition } = props

    // 初始情况或没有传过渡时间则不没有动画
    if (oldName == null || toNumber(transition) === 0) {
        nextName.value = newName
        return
    }

    // 添加缩小为0的css类名
    shrinking.value = true
    await nextTick()
    // 缩小动画结束后去掉类名及更新icon
    setTimeout(() => {
        oldName != null && (nextName.value = newName)
        // 恢复为原本大小
        shrinking.value = false
    }, toNumber(transition))
}

watch(() => props.name, handleNameChange, { immediate: true })
```

图标组件的实现部分还是比较简单的，到这里图标部分的详解就结束了，我们下一篇再见~

