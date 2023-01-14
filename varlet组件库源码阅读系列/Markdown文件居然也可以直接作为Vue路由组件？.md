> 本文为Varlet组件库源码主题阅读系列第五篇，读完本文你可以了解到如何通过编写一个`Vite`插件来支持使用`md`文件直接作为路由组件。

之前[文档站点的搭建]()里我们介绍了路由的动态生成逻辑，其中说到了文档是使用`Markdown`格式编写的，并且还直接在路由文件里使用`md`文件作为路由组件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c85da6bf68044403b5a76a9cc92aaaa7~tplv-k3u1fbpfcp-zoom-1.image)


路由就是路径到组件的映射，这个组件显然指的是`Vue`组件，`Vue`组件是一个包含特定选项的`JavaScript`对象，我们平常开发一般使用的是`Vue`单文件，单文件最终也会被编译成选项对象，这个工作是[@vitejs/plugin-vue](https://github.com/vitejs/vite/tree/main/packages/plugin-vue)做的，显然这个插件并不能处理`Markdown`文件，那么最终也就无法生成正确的`Vue`组件。

解决方法就是编写一个[Vite插件](https://www.vitejs.net/guide/api-plugin.html)，指定在`@vitejs/plugin-vue`插件之前调用，将`.md`文件的内容转换为`Vue`单文件的格式，然后配置`@vitejs/plugin-vue`插件，让它顺便也处理一下扩展名为`.md`的文件，因为已经转换成`Vue`单文件的语法格式了，所以它可以正常处理，接下来从源码角度来详细了解一下。



# Vite配置

之前的文章里详细介绍了启动服务时的`Vite`配置，这里只看一下涉及到的插件部分：

```ts
// varlet-cli/src/config/vite.config.ts
import vue from '@vitejs/plugin-vue'
import md from '@varlet/markdown-vite-plugin'

export function getDevConfig(varletConfig: Record<string, any>): InlineConfig {
    return {
        plugins: [
            vue({
                include: [/\.vue$/, /\.md$/],// vue插件默认只处理.vue文件，通过该参数配置让其也处理一下.md文件
            }),
            md({ style: get(varletConfig, 'highlight.style') }),// 使用md文件转换插件，使用插件时可以传入参数
        ]
    }
}
```

# markdown-vite-plugin插件

插件代码在`varlet-markdown-vite-plugin`目录，一个`Vite`插件就是一个函数，接收使用时传入的参数，最终返回一个对象。`Vite`插件扩展了`Rollup`的接口，并且带有一些` Vite` 独有的配置项，配置项类型基本就是两种，一种是属性，一种是钩子函数，插件的主要逻辑都在钩子函数里，`Rollup`和`Vite`提供了构建和编译时各个时机的钩子，插件可以根据功能选择对应的钩子。

`Vite`插件文档：[插件API](https://www.vitejs.net/guide/api-plugin.html)。

`Rollup`插件文档：[plugin-development](https://rollupjs.org/guide/en/#plugin-development)。

```js
// varlet-markdown-vite-plugin/index.js
function VarletMarkdownVitePlugin(options) {
  return {
    name: 'varlet-markdown-vite-plugin',// 插件名称
    enforce: 'pre',// 插件调用顺序
    // Rollup钩子，转换文件内容
    transform(source, id) {
      if (!/\.md$/.test(id)) {
        return
      }
      try {
        return markdownToVue(source, options)
      } catch (e) {
        this.error(e)
        return ''
      }
    },
    // Vite钩子，用于热更新
    async handleHotUpdate(ctx) {
      if (!/\.md$/.test(ctx.file)) return
      const readSource = ctx.read
      ctx.read = async function () {
        return markdownToVue(await readSource(), options)
      }
    },
  }
}
module.exports = VarletMarkdownVitePlugin
```

以上就是这个插件的函数，返回了一个对象，`name`属性为插件的名称，必填，用于信息和错误输出时的提示；`enforce`用于指定钩子的调用顺序：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44a33aa1120b4b4c870f33d88eb6eb22~tplv-k3u1fbpfcp-zoom-1.image)


`vue`插件没有指定，所以`md`插件会在其之前调用，保证到它这里`.md`文件的内容已经转换完毕。

接下来配置了两个钩子函数，我们详细来看。

## md文件内容转换

[transform](https://rollupjs.org/guide/en/#transform)是`Rollup`提供的构建阶段的钩子，可以在这个钩子内转换文件的内容，先判断文件后缀是否是`.md`，不是的话就不进行处理，是的话调用了`markdownToVue`方法：

```js
// varlet-markdown-vite-plugin/index.js
function markdownToVue(source, options) {
    const { source: vueSource, imports, components } = extractComponents(source)
    // ...
}
```

### 支持在md文件中引入Vue组件

`source`即文件内容，进来先调用了`extractComponents`方法，这个方法是干嘛的呢，是用来支持在`md`文件里引入`Vue`组件的，比如布局组件中的`Row`组件的文档：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d0f174ffab2404597454a58ee288a3d~tplv-k3u1fbpfcp-zoom-1.image)


引入了`Responsive.vue`组件，最终在页面上的渲染效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ffb883f08784dc7b6eb0cf963adc4d2~tplv-k3u1fbpfcp-zoom-1.image)


知道了它的作用后我们再来看一下实现：

```js
// varlet-markdown-vite-plugin/index.js
function extractComponents(source) {
  const componentRE = /import (.+) from ['"].+['"]/
  const importRE = /import .+ from ['"].+['"]/g
  const vueRE = /```vue((.|\r|\n)*?)```/g
  const imports = []
  const components = []
  
  // 替换```vue....```的内容
  source = source.replace(vueRE, (_, p1) => {
    // 解析出import语句列表
    const partImports = p1.match(importRE)
    const partComponents = partImports?.map((importer) => {
      // 去除换行符
      importer = importer.replace(/(\n|\r)/g, '')
      // 解析出导入的组件名
      const component = importer.replace(componentRE, '$1')
      // 收集导入语句及导入的组件
      !imports.includes(importer) && imports.push(importer)
      !components.includes(component) && components.push(component)
      // 返回使用组件的字符串
      return `<${kebabCase(component)} />`
    })
    return partComponents ? `<div class="varlet-component-preview">${partComponents.join('\n')}</div>` : ''
  })
  return {
    imports,
    components,
    source,
  }
}
```

以前面的为例，`source`为：

```markdown
xxx

​```vue
import BasicExample from '../example/Responsive.vue'
​```

xxx
```

匹配到`vueRE`，`p1`为：

```
import BasicExample from '../example/Responsive.vue'
```

使用`importRE`正则匹配后可以得到`partImports`数组：

```js
[
    `import BasicExample from '../example/Responsive.vue'`
]
```

遍历这个数组，然后解析出`component`为`BasicExample`，将导入语句及组件名称收集起来，然后拼接模板字符串为：

```html
<div class="varlet-component-preview">
    <basic-example />
</div>
```

最后这个模板字符串会替换掉`source`内`vueRE`匹配到的内容。

### 代码高亮

让我们继续回到`markdownToVue`方法：

```js
// varlet-markdown-vite-plugin/index.js
const markdown = require('markdown-it')

function markdownToVue(source, options) {
    // ...
    const md = markdown({
        html: true,// 允许存在html标签，这是必要的，因为前面处理Vue组件最后会生成html标签
        typographer: true,// 允许替换一些特殊字符，https://github.com/markdown-it/markdown-it/blob/master/lib/rules_core/replacements.js
        highlight: (str, lang) => highlight(str, lang, options.style),// 代码高亮，str为要高亮的代码，lang为语言种类
    })
}
```

使用[markdown-it](https://github.com/markdown-it/markdown-it)解析`markdown`，并且使用了`highlight`属性自定义代码语法高亮：

```js
// varlet-markdown-vite-plugin/index.js
const hljs = require('highlight.js')

function highlight(str, lang, style) {
  let link = ''

  if (style) {
    link = '<link class="hljs-style" rel="stylesheet" href="' + style + '"/>'
  }

  if (lang && hljs.getLanguage(lang)) {
    return (
      '<pre class="hljs"><code>' +
      link +
      hljs.highlight(str, { language: lang, ignoreIllegals: true }).value +
      '</code></pre>'
    )
  }

  return ''
}
```

代码高亮使用的是[highlight.js](https://github.com/search?q=highlight.js)，最开始使用`md`插件时我们传入了参数：

```js
{ style: get(varletConfig, 'highlight.style') }
```

这个用于设置`highlight.js`的主题，一个主题就是一个`css`文件，`highlight.js`内置了非常多的主题：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/582b615be45345a4bd94c210d4260b6f~tplv-k3u1fbpfcp-zoom-1.image)


默认配置如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/250206fdf1ba473187881da8ee214e2a~tplv-k3u1fbpfcp-zoom-1.image)


所以当指定了主题的话会创建一个`link`标签来加载对应的主题样式，否则就使用默认的，默认主题定义在`/site/pc/Layout.vue`组件内：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3fc37c668a44f5c9e3225cea6e7b956~tplv-k3u1fbpfcp-zoom-1.image)


这么做的好处是可以使用`css`变量，当页面切换暗黑模式时代码主题也可以跟着变化。

### 处理生成的html

继续看`markdownToVue`方法：

```js
// varlet-markdown-vite-plugin/index.js
function markdownToVue(source, options) {
    // ...
    let templateString = htmlWrapper(md.render(vueSource))
  	templateString = templateString.replace(/process.env/g, '<span>process.env</span>')
}
```

调用`render`方法将`markdown`编译成`html`，然后调用了`htmlWrapper`方法：

```js
// varlet-markdown-vite-plugin/index.js
function htmlWrapper(html) {
  const hGroup = html.replace(/<h3/g, ':::<h3').replace(/<h2/g, ':::<h2').split(':::')
  const cardGroup = hGroup
    .map((fragment) => (fragment.includes('<h3') ? `<div class="card">${fragment}</div>` : fragment))
    .join('')
  return cardGroup.replace(/<code>/g, '<code v-pre>')
}
```

前两行做的事情就是把`h3`标签之后，`h2`标签之前的内容都用类名为`card`的`div`包裹起来，目的是为了在页面上显示一个个块的效果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af5b8f86c88142ada8a396763b4b5a99~tplv-k3u1fbpfcp-zoom-1.image)


最后一行是给`code`标签添加了一个`v-pre`指令，这个指令用来跳过该元素及其所有子元素的编译，因为文档的代码示例难免会涉及到`Vue`模板语法的示例，如果不跳过，直接就被编译了。

### 引入代码块组件

继续`markdownToVue`方法：

```js
// varlet-markdown-vite-plugin/index.js
function markdownToVue(source, options) {
    // ...
    templateString = injectCodeExample(templateString)
}
```

又调用了`injectCodeExample`方法：

```js
// varlet-markdown-vite-plugin/index.js
function injectCodeExample(source) {
  const codeRE = /(<pre class="hljs">(.|\r|\n)*?<\/pre>)/g

  return source.replace(codeRE, (str) => {
    const flags = [
      '// playground-ignore\n',
      '<span class="hljs-meta">#</span><span class="bash"> playground-ignore</span>\n',
      '<span class="hljs-comment">// playground-ignore</span>\n',
      '<span class="hljs-comment">/* playground-ignore */</span>\n',
      '<span class="hljs-comment">&lt;!-- playground-ignore --&gt;</span>\n',
    ]

    const attr = flags.some((flag) => str.includes(flag)) ? 'playground-ignore' : ''

    str = flags.reduce((str, flag) => str.replace(flag, ''), str)

    // 引入var-site-code-example组件
    return `<var-site-code-example ${attr}>${str}</var-site-code-example>`
  })
}
```

`Varlet`提供了在线`playground`的功能：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7725e821f0584f359c65fccab3c36a39~tplv-k3u1fbpfcp-zoom-1.image)


可以直接从文档的代码块进行跳转：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b846baafdac44d5a2c7bc8ca8d51129~tplv-k3u1fbpfcp-zoom-1.image)


但不是所有代码块都需要，比如：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1332576afb3469b92c74fe997583cd1~tplv-k3u1fbpfcp-zoom-1.image)


所以就通过在文档上增加一个注释来注明忽略：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f87711a670f9415eac2c0d733e2512be~tplv-k3u1fbpfcp-zoom-1.image)


`injectCodeExample`方法就会检查是否存在这个标志，存在的话就给`var-site-code-example`组件传递一个不显示这个跳转按钮的属性，`var-site-code-example`组件的路径为`/site/components/code-example/CodeExample.vue`，用来提供代码块的展开收起、复制、跳转`playground`的功能。

### 组装Vue单文件的格式

最后就是按照`Vue`单文件的格式进行拼接了：

```js
// varlet-markdown-vite-plugin/index.js
function markdownToVue(source, options) {
    // ...
    return `
        <template>
			<div class="varlet-site-doc">${templateString}</div>
		</template>

        <script>
            ${imports.join('\n')}

            export default {
              components: {
                ${components.join(',')}
              }
            }
        </script>
  	`
}
```

把转换得到的`html`内容添加到`template`标签内，把解析出的组件导入语句添加到`script`标签内，并且进行注册，转换成这种格式后，后续`vue`插件就可以正常处理了。

## 热更新

除了`transform`钩子，还使用到了[handleHotUpdate](https://www.vitejs.net/guide/api-plugin.html#handlehotupdate)钩子，这个钩子是`Vite`提供的，用来执行自定义的热更新处理，这个钩子接收一个上下文对象：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4279c8de4c74929a8fcbb2667eee56a~tplv-k3u1fbpfcp-zoom-1.image)


`file`是发生变化的文件，`read`是读取这个文件内容的方法，`varlet-markdown-vite-plugin`插件重写了这个方法：

```js
// varlet-markdown-vite-plugin/index.js
function VarletMarkdownVitePlugin(options) {
  return {
    async handleHotUpdate(ctx) {
      if (!/\.md$/.test(ctx.file)) return

      const readSource = ctx.read
      ctx.read = async function () {
        return markdownToVue(await readSource(), options)
      }
    },
  }
}
```

目的和前面一样，就是把`markdown`语法转换成`Vue`单文件语法，`vue`插件也使用了这个钩子和`read`方法：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7236c99c2f934f1fb966a333f386ab02~tplv-k3u1fbpfcp-zoom-1.image)


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/392e3d75b47d4ab88d050daf5c7e7835~tplv-k3u1fbpfcp-zoom-1.image)


同样因为这个插件是在`vue`插件之前调用的，所以到了`vue`插件使用的就是被转换的`read`方法，就能在热更新时顺利处理`.md`文件。

处理`markdown`的插件就介绍到这里，我们下一篇再见，拜拜~
