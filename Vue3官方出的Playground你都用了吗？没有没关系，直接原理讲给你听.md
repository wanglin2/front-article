相比`Vue2`，`Vue3`的官方文档中新增了一个在线`Playground`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/680404a519ac49bbafd28e75afd2ff2d~tplv-k3u1fbpfcp-zoom-1.image)

打开是这样的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d28c22ee4994adc848c7596001ccb66~tplv-k3u1fbpfcp-zoom-1.image)

相当于让你可以在线编写和运行`Vue`单文件组件，当然这个东西也是开源的，并且发布为了一个`npm`包，本身是作为一个`Vue`组件，所以可以轻松在你的`Vue`项目中使用：

```html
<script setup>
import { Repl } from '@vue/repl'
import '@vue/repl/style.css'
</script>

<template>
  <Repl />
</template>
```

用于`demo`编写和分享还是很不错的，尤其适合作为基于`Vue`相关项目的在线`demo`，目前很多`Vue3`的组件库都用了，仓库地址：[@vue/repl](https://github.com/vuejs/repl)。

`@vue/repl`有一些让人（我）眼前一亮的特性，比如数据存储在`url`中，支持创建多个文件，当然也存在一些限制，比如只支持`Vue3`，不支持使用`CSS`预处理语言，不过支持使用`ts`。

接下来会带领各位从头探索一下它的实现原理，需要说明的是我们会选择性的忽略一些东西，比如`ssr`相关的，有需要了解这方面的可以自行阅读源码。

首先下载该项目，然后找到测试页面的入口文件：

```ts
// test/main.ts
const App = {
  setup() {
    // 创建数据存储的store
    const store = new ReplStore({
      serializedState: location.hash.slice(1)
    })
	// 数据存储
    watchEffect(() => history.replaceState({}, '', store.serialize()))
	// 渲染Playground组件
    return () =>
      h(Repl, {
        store,
        sfcOptions: {
          script: {}
        }
      })
  }
}

createApp(App).mount('#app')
```

首先取出存储在`url`的`hash`中的文件数据，然后创建了一个`ReplStore`类的实例`store`，所有的文件数据都会保存在这个全局的`store`里，接下来监听`store`的文件数据变化，变化了会实时反映在`url`中，即进行实时存储，最后渲染组件`Repl`并传入`store`。

先来看看`ReplStore`类。

# 数据存储

```ts
// 默认的入口文件名称
const defaultMainFile = 'App.vue'
// 默认文件的内容
const welcomeCode = `
<script setup>
import { ref } from 'vue'

const msg = ref('Hello World!')
</script>

<template>
  <h1>{{ msg }}</h1>
  <input v-model="msg">
</template>
`.trim()

// 数据存储类
class ReplStore {
    constructor({
        serializedState = '',
        defaultVueRuntimeURL = `https://unpkg.com/@vue/runtime-dom@${version}/dist/runtime-dom.esm-browser.js`,
    }) {
        let files: StoreState['files'] = {}
        // 有存储的数据
        if (serializedState) {
            // 解码保存的数据
            const saved = JSON.parse(atou(serializedState))
            for (const filename in saved) {
                // 遍历文件数据，创建文件实例保存到files对象上
                files[filename] = new File(filename, saved[filename])
            }
        } else {
        // 没有存储的数据
            files = {
                // 创建一个默认的文件
                [defaultMainFile]: new File(defaultMainFile, welcomeCode)
            }
        }
        // Vue库的cdn地址，注意是运行时版本，即不包含编译模板的代码，也就是模板必须先被编译成渲染函数才行
        this.defaultVueRuntimeURL = defaultVueRuntimeURL
        // 默认的入口文件为App.vue
        let mainFile = defaultMainFile
        if (!files[mainFile]) {
            // 自定义了入口文件
            mainFile = Object.keys(files)[0]
        }
        // 核心数据
        this.state = reactive({
            mainFile,// 入口文件名称
            files,// 所有文件
            activeFile: files[mainFile],// 当前正在编辑的文件
            errors: [],// 错误信息
            vueRuntimeURL: this.defaultVueRuntimeURL,// Vue库的cdn地址
        })
        // 初始化import-map
        this.initImportMap()
    }
}
```

主要是使用`reactive`创建了一个响应式对象来作为核心的存储对象，存储的数据包括入口文件名称`mainFile`，一般作为根组件，所有的文件数据`files`，以及当前我们正在编辑的文件对象`activeFile`。

## 数据是如何存储在url中的

可以看到上面对`hash`中取出的数据`serializedState`调用了`atou`方法，用于解码数据，还有一个与之相对的`utoa`，用于编码数据。

大家或多或少应该都听过`url`有最大长度的限制，所以按照我们一般的想法，数据肯定不会选择存储到`url`上，但是`hash`部分的应该不受影响，并且`hash`数据也不会发送到服务端。

即便如此，`@vue/repl`在存储前还是先做了压缩的处理，毕竟`url`很多情况下是用来分享的，太长总归不太方便。

首先来看一下最开始提到的`store.serialize()`方法，用来序列化文件数据存储到`url`上：

```ts
class ReplStore {
    // 序列化文件数据
    serialize() {
        return '#' + utoa(JSON.stringify(this.getFiles()))
    }
    // 获取文件数据
    getFiles() {
        const exported: Record<string, string> = {}
        for (const filename in this.state.files) {
            exported[filename] = this.state.files[filename].code
        }
        return exported
    }

}
```

调用`getFiles`取出文件名和文件内容，然后转成字符串后调用`utoa`方法：

```ts
import { zlibSync, strToU8, strFromU8 } from 'fflate'

export function utoa(data: string): string {
  // 将字符串转成Uint8Array
  const buffer = strToU8(data)
  // 以最大的压缩级别进行压缩，返回的zipped也是一个Uint8Array
  const zipped = zlibSync(buffer, { level: 9 })
  // 将Uint8Array重新转换成二进制字符串
  const binary = strFromU8(zipped, true)
  // 将二进制字符串编码为Base64编码字符串
  return btoa(binary)
}
```

压缩使用了[fflate](https://github.com/101arrowz/fflate)，号称是目前最快、最小、最通用的纯`JavaScript`压缩和解压库。

可以看到其中`strFromU8`方法第二个参数传了`true`，代表转换成二进制字符串，这是必要的，因为`js`内置的`btoa`和`atob`方法不支持`Unicode`字符串，而我们的代码内容显然不可能只使用`ASCII`的`256`个字符，那么直接使用`btoa`编码就会报错：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c27873c53e094c53a7c38f0dfa8cfe41~tplv-k3u1fbpfcp-zoom-1.image)

详情：[https://base64.guru/developers/javascript/examples/unicode-strings](https://base64.guru/developers/javascript/examples/unicode-strings)。

看完了压缩方法再来看一下对应的解压方法`atou`：

```ts
import { unzlibSync, strToU8, strFromU8 } from 'fflate'

export function atou(base64: string): string {
    // 将base64转成二进制字符串
    const binary = atob(base64)
    // 检查是否是zlib压缩的数据，zlib header (x78), level 9 (xDA)
    if (binary.startsWith('\x78\xDA')) {
        // 将字符串转成Uint8Array
        const buffer = strToU8(binary, true)
        // 解压缩
        const unzipped = unzlibSync(buffer)
        // 将Uint8Array重新转换成字符串
        return strFromU8(unzipped)
    }
    // 兼容没有使用压缩的数据
    return decodeURIComponent(escape(binary))
}
```

和`utoa`稍微有点不一样，最后一行还兼容了没有使用`fflate`压缩的情况，因为`@vue/repl`毕竟是个组件，用户初始传入的数据可能没有使用`fflate`压缩，而是使用下面这种方式转`base64`的：

```js
function utoa(data) {
  return btoa(unescape(encodeURIComponent(data)));
}
```

## 文件类File

保存到`files`对象上的文件不是纯文本内容，而是通过`File`类创建的文件实例：

```ts
// 文件类
export class File {
  filename: string// 文件名
  code: string// 文件内容
  compiled = {// 该文件编译后的内容
    js: '',
    css: ''
  }

  constructor(filename: string, code = '', hidden = false) {
    this.filename = filename
    this.code = code
  }
}
```

这个类很简单，除了保存文件名和文件内容外，主要是存储文件被编译后的内容，如果是`js`文件，编译后的内容保存在`compiled.js`上，`css`显然就是保存在`compiled.css`上，如果是`vue`单文件，那么`script`和`template`会编译成`js`保存到`compiled.js`上，样式则会提取到`compiled.css`上保存。

这个编译逻辑我们后面会详细介绍。

## 使用import-map

在浏览器上直接使用`ESM`语法是不支持裸导入的，也就是下面这样不行：

```js
import moment from "moment";
```

导入来源需要是一个合法的`url`，那么就出现了[import-map](https://github.com/WICG/import-maps)这个提案，当然目前兼容性还不太好[import-maps](https://caniuse.com/import-maps)，不过可以`polyfill`:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e14a2baf18b64b298335088cf12f8551~tplv-k3u1fbpfcp-zoom-1.image)

这样我们就可以通过下面这种方式来使用裸导入了：

```html
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment.js",
  }
}
</script>

<script type="importmap">
import moment from "moment";
</script>
```

那么我们看一下`ReplStore`的`initImportMap`方法都做了 什么：

```ts
private initImportMap() {
    const map = this.state.files['import-map.json']
    if (!map) {
        // 如果还不存在import-map.json文件，就创建一个，里面主要是Vue库的map
        this.state.files['import-map.json'] = new File(
            'import-map.json',
            JSON.stringify(
                {
                    imports: {
                        vue: this.defaultVueRuntimeURL
                    }
                },
                null,
                2
            )
        )
    } else {
        try {
            const json = JSON.parse(map.code)
            // 如果vue不存在，那么添加一个
            if (!json.imports.vue) {
                json.imports.vue = this.defaultVueRuntimeURL
                map.code = JSON.stringify(json, null, 2)
            }
        } catch (e) {}
    }
}
```

其实就是创建了一个`import-map.json`文件用来保存`import-map`的内容。

接下来就进入到我们的主角`Repl.vue`组件了，模板部分其实没啥好说的，主要分为左右两部分，左侧编辑器使用的是`codemirror`，右侧预览使用的是`iframe`，主要看一下`script`部分：

```js
// ...
props.store.options = props.sfcOptions
props.store.init()
// ...
```

核心就是这两行，将使用组件时传入的`sfcOptions`保存到`store`的`options`属性上，后续编译文件时会使用，当然默认啥也没传，一个空对象而已，然后执行了`store`的`init`方法，这个方法就会开启文件编译。

# 文件编译

```ts
class ReplStore {
  init() {
    watchEffect(() => compileFile(this, this.state.activeFile))
    for (const file in this.state.files) {
      if (file !== defaultMainFile) {
        compileFile(this, this.state.files[file])
      }
    }
  } 
}
```

编译当前正在编辑的文件，默认为`App.vue`，并且当当前正在编辑的文件发生变化之后会重新触发编译。另外如果初始存在多个文件，也会遍历其他的文件进行编译。

执行编译的`compileFile`方法比较长，我们慢慢来看。

## 编译css文件

```ts
export async function compileFile(
store: Store,
 { filename, code, compiled }: File
) {
    // 文件内容为空则返回
    if (!code.trim()) {
        store.state.errors = []
        return
    }
    // css文件不用编译，直接把文件内容存储到compiled.css属性
    if (filename.endsWith('.css')) {
        compiled.css = code
        store.state.errors = []
        return
    }
    // ...
}
```

`@vue/repl`目前不支持使用`css`预处理语言，所以样式的话只能创建`css`文件，很明显`css`不需要编译，直接保存到编译结果对象上即可。

## 编译js、ts文件

继续：

```ts
export async function compileFile(){
    // ...
    if (filename.endsWith('.js') || filename.endsWith('.ts')) {
        if (shouldTransformRef(code)) {
            code = transformRef(code, { filename }).code
        }
        if (filename.endsWith('.ts')) {
            code = await transformTS(code)
        }
        compiled.js = code
        store.state.errors = []
        return
    }
    // ...
}
```

`shouldTransformRef`和`transformRef`两个方法是[@vue/reactivity-transform](https://github.com/vuejs/core/tree/main/packages/reactivity-transform)包中的方法，用来干啥的呢，其实`Vue3`中有个实验性质的提案，我们都知道可以使用`ref`来创建一个原始值的响应性数据，但是访问的时候需要通过`.value`才行，那么这个提案就是去掉这个`.value`，方式是不使用`ref`，而是使用`$ref`，比如：

```js
// $ref都不用导出，直接使用即可
let count = $ref(0)
console.log(count)
```

除了`ref`，还支持其他几个`api`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41fd98c59c3e43e3996e106d84977f2e~tplv-k3u1fbpfcp-zoom-1.image)

所以`shouldTransformRef`方法就是用来检查是否使用了这个实验性质的语法，`transformRef`方法就是用来将其转换成普通语法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ef55f1775454c91ab5e3c45b120608c~tplv-k3u1fbpfcp-zoom-1.image)

如果是`ts`文件则会使用`transformTS`方法进行编译：

```ts
import { transform } from 'sucrase'

async function transformTS(src: string) {
  return transform(src, {
    transforms: ['typescript']
  }).code
}
```

使用[sucrase](https://github.com/alangpierce/sucrase)转换`ts`语法（说句题外话，我喜欢看源码的一个原因之一就是总能从源码中发现一些有用的库或者工具），通常我们转换`ts`要么使用官方的`ts`工具，要么使用`babel`，但是如果对编译结果的浏览器兼容性不太关心的话可以使用`sucrase`，因为它超级快：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b59058a756f74d4cbc22d684751aea26~tplv-k3u1fbpfcp-zoom-1.image)

## 编译Vue单文件

继续回到`compileFile`方法：

```ts
import hashId from 'hash-sum'

export async function compileFile(){
    // ...
    // 如果不是vue文件，那么就到此为止，其他文件不支持
    if (!filename.endsWith('.vue')) {
        store.state.errors = []
        return
    }
    // 文件名不能重复，所以可以通过hash生成一个唯一的id，后面编译的时候会用到
    const id = hashId(filename)
    // 解析vue单文件
    const { errors, descriptor } = store.compiler.parse(code, {
        filename,
        sourceMap: true
    })
    // 如果解析出错，保存错误信息然后返回
    if (errors.length) {
        store.state.errors = errors
        return
    }
    // 接下来进行了两个判断，不影响主流程，代码就不贴了
    // 判断template和style是否使用了其他语言，是的话抛出错误并返回
    // 判断script是否使用了ts外的其他语言，是的话抛出错误并返回
    // ...
}
```

编译`vue`单文件的包是[@vue/compiler-sfc](https://github.com/vuejs/core/tree/main/packages/compiler-sfc)，从`3.2.13`版本起这个包会内置在`vue`包中，安装`vue`就可以直接使用这个包，这个包会随着`vue`的升级而升级，所以`@vue/repl`并没有写死，而是可以手动配置：

```ts
import * as defaultCompiler from 'vue/compiler-sfc'

export class ReplStore implements Store {
    compiler = defaultCompiler
      vueVersion?: string

    async setVueVersion(version: string) {
        this.vueVersion = version
        const compilerUrl = `https://unpkg.com/@vue/compiler-sfc@${version}/dist/compiler-sfc.esm-browser.js`
        const runtimeUrl = `https://unpkg.com/@vue/runtime-dom@${version}/dist/runtime-dom.esm-browser.js`
        this.pendingCompiler = import(/* @vite-ignore */ compilerUrl)
        this.compiler = await this.pendingCompiler
        // ...
    }
}
```

默认使用当前仓库的`compiler-sfc`，但是可以通过调用`store.setVueVersion`方法来设置指定版本的`vue`和`compiler`。

假设我们的`App.vue`的内容如下：

```html
<script setup>
import { ref } from 'vue'
const msg = ref('Hello World!')
</script>

<template>
  <h1>{{ msg }}</h1>
  <input v-model="msg">
</template>

<style>
  h1 {
    color: red;
  }
</style>
```

`compiler.parse`方法会将其解析成如下结果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d98d5edb7f3458a9e136f83ead8d303~tplv-k3u1fbpfcp-zoom-1.image)

其实就是解析出了其中的`script`、`template`、`style`三个部分的内容。

继续回到`compileFile`方法：

```ts
export async function compileFile(){
    // ...
    // 是否有style块使用了scoped作用域
    const hasScoped = descriptor.styles.some((s) => s.scoped)
	// 保存编译结果
    let clientCode = ''
    const appendSharedCode = (code: string) => {
        clientCode += code
    }
    // ...
}
```

`clientCode`用来保存最终的编译结果。

### 编译script

继续回到`compileFile`方法：

```ts
export async function compileFile(){
    // ...
    const clientScriptResult = await doCompileScript(
        store,
        descriptor,
        id,
        isTS
    )
    // ...
}
```

调用`doCompileScript`方法编译`script`部分，其实`template`部分也会被一同编译进去，除非你没有使用`<script setup>`语法或者手动配置了不要这么做：

```js
h(Repl, {
    sfcOptions: {
        script: {
            inlineTemplate: false
        }
    }
})
```

我们先忽略这种情况，看一下`doCompileScript`方法的实现：

```ts
export const COMP_IDENTIFIER = `__sfc__`

async function doCompileScript(
  store: Store,
  descriptor: SFCDescriptor,
  id: string,
  isTS: boolean
): Promise<[string, BindingMetadata | undefined] | undefined> {
  if (descriptor.script || descriptor.scriptSetup) {
    try {
      const expressionPlugins: CompilerOptions['expressionPlugins'] = isTS
        ? ['typescript']
        : undefined
      // 1.编译script
      const compiledScript = store.compiler.compileScript(descriptor, {
        inlineTemplate: true,// 是否编译模板并直接在setup()里面内联生成的渲染函数
        ...store.options?.script,
        id,// 用于样式的作用域
        templateOptions: {// 编译模板的选项
          ...store.options?.template,
          compilerOptions: {
            ...store.options?.template?.compilerOptions,
            expressionPlugins// 这个选项并没有在最新的@vue/compiler-sfc包的源码中看到，可能废弃了
          }
        }
      })
      let code = ''
      // 2.转换默认导出
      code +=
        `\n` +
        store.compiler.rewriteDefault(
          compiledScript.content,
          COMP_IDENTIFIER,
          expressionPlugins
        )
      // 3.编译ts
      if ((descriptor.script || descriptor.scriptSetup)!.lang === 'ts') {
        code = await transformTS(code)
      }
      return [code, compiledScript.bindings]
    } catch (e: any) {
      store.state.errors = [e.stack.split('\n').slice(0, 12).join('\n')]
      return
    }
  } else {
    return [`\nconst ${COMP_IDENTIFIER} = {}`, undefined]
  }
}
```

这个函数主要做了三件事，我们一一来看。

1.编译script

调用`compileScript`方法编译`script`，这个方法会处理`<script setup>`语法、`css`变量注入等特性，`css`变量注入指的是在`style`标签中使用`v-bind`绑定组件的`data`数据这种情况，详细了解[CSS 中的 v-bind]([单文件组件 CSS 功能 | Vue.js](https://cn.vuejs.org/api/sfc-css-features.html#v-bind-in-css))。

如果使用了`<script setup>`语法，且`inlineTemplate`选项传了`true`，那么会同时将`template`部分编译成渲染函数并内联到`setup`函数里面，否则`template`需要另外编译。

`id`参数用于作为`scoped id`，当`style`块中使用了`scoped`，或者使用了`v-bind`语法，都需要使用这个`id`来创建唯一的`class`类名、样式名。

编译结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2548dc29fdf24ee7a9c4800647afea01~tplv-k3u1fbpfcp-zoom-1.image)

可以看到模板部分被编译成了渲染函数并内联到了组件的`setup`函数内，并且使用`export default`默认导出组件。



2.转换默认导出

这一步会把前面得到的默认导出语句转换成变量定义的形式，使用的是`rewriteDefault`方法，这个方法接收三个参数：要转换的内容、变量名称、插件数组，这个插件数组是传给`babel`使用的，所以如果使用了`ts`，那么会传入`['typescript']`。

转换结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/487bc5cde93847c0910cd90e2b64c169~tplv-k3u1fbpfcp-zoom-1.image)

转成变量有什么好处呢，其实这样就可以方便的在对象上添加其他属性了，如果是`export default {}`的形式，如果想在这个对象上扩展一些属性要怎么做呢？正则匹配？转成`ast`树？好像可以，但不是很方便，因为都得操作源内容，但是转成变量就很简单了，只要知道定义的变量名称，就可以直接拼接如下代码：

```ts
__sfc__.xxx = xxx
```

不需要知道源内容是什么，想添加什么属性就直接添加。



3.编译ts

最后一步会判断是否使用了`ts`，是的话就使用前面提到过的`transformTS`方法进行编译。



编译完`script`回到`compileFile`方法：

```ts
export async function compileFile(){
  // ...
  // 如果script编译没有结果则返回
  if (!clientScriptResult) {
    return
  }
  // 拼接script编译结果
  const [clientScript, bindings] = clientScriptResult
  clientCode += clientScript
  // 给__sfc__组件对象添加了一个__scopeId属性
  if (hasScoped) {
    appendSharedCode(
      `\n${COMP_IDENTIFIER}.__scopeId = ${JSON.stringify(`data-v-${id}`)}`
    )
  }
  if (clientCode) {
    appendSharedCode(
      `\n${COMP_IDENTIFIER}.__file = ${JSON.stringify(filename)}` +// 给__sfc__组件对象添加了一个__file属性
      `\nexport default ${COMP_IDENTIFIER}`// 导出__sfc__组件对象
    )
    compiled.js = clientCode.trimStart()// 将script和template的编译结果保存起来
  }
  // ...
}
```

将`export default`转换成变量定义的好处来了，添加新属性很方便。

最后使用`export default`导出定义的变量即可。



### 编译template

前面已经提到过几次，如果使用了`<script setup>`语法且`inlineTemplate`选项没有设为`false`，那么无需自己手动编译`template`，如果要自己编译也很简单，调用一下`compiler.compileTemplate`方法编译即可，实际上前面只是`compiler.compileScript`方法内部帮我们调了这个方法而已，这个方法会将`template`编译成渲染函数，我们把这个渲染函数字符串也拼接到`clientCode`上，并且在组件选项对象，也就是前面一步编译`script`得到的`__sfc__`对象上添加一个`render`属性，值就是这个渲染函数：

```ts
let code =
    `\n${templateResult.code.replace(
      /\nexport (function|const) render/,
      `$1 render`
    )}` + `\n${COMP_IDENTIFIER}.render = render
```



### 编译style

继续回到`compileFile`方法，到这里`vue`单文件的`script`和`template`部分就已经编译完了，接下来会处理`style`部分：

```ts
export async function compileFile(){
  // ...
  let css = ''
  // 遍历style块
  for (const style of descriptor.styles) {
    // 不支持使用CSS Modules
    if (style.module) {
      store.state.errors = [
        `<style module> is not supported in the playground.`
      ]
      return
    }
	// 编译样式
    const styleResult = await store.compiler.compileStyleAsync({
      ...store.options?.style,
      source: style.content,
      filename,
      id,
      scoped: style.scoped,
      modules: !!style.module
    })
    css += styleResult.code + '\n'
  }
  if (css) {
    // 保存编译结果
    compiled.css = css.trim()
  } else {
    compiled.css = '/* No <style> tags present */'
  }
}
```

很简单，使用`compileStyleAsync`方法编译`style`块，这个方法会帮我们处理`scoped`、`module`以及`v-bind`语法。

到这里，文件编译部分就介绍完了，总结一下：

- 样式文件因为只能使用原生`css`，所以不需要编译
- `js`文件原本也不需要编译，但是有可能使用了实验性质的`$ref`语法，所以需要进行一下判断并处理，如果使用了`ts`那么需要进行编译
- `vue`单文件会使用`@vue/compiler-sfc`编译，`script`部分会处理`setup`语法、`css`变量注入等特性，如果使用了`ts`也会编译`ts`，最后的结果其实就是组件对象，`template`部分无论是和`script`一起编译还是单独编译，最后都会编译成渲染函数挂载到组件对象上，`style`部分编译后直接保存起来即可



# 预览

文件都编译完成了接下来是不是就可以直接预览了呢？很遗憾，并不能，为什么呢，因为前面文件编译完后得到的是普通的`ESM`模块，也就是通过`import `和`export`来导入和导出，比如除了`App.vue`外，我们还创建了一个`Comp.vue`文件，然后在`App.vue`中引入

```ts
// App.vue
import Comp from './Comp.vue'
```

乍一看好像没啥问题，但问题是服务器上并没有`./Comp.vue`文件，这个文件只是我们在前端模拟的，那么如果直接让浏览器发出这个模块请求肯定是失败的，并且我们模拟创建的这些文件最终都会通过一个个`<script type="module">`标签插入到页面，所以需要把`import`和`export`转换成其他形式。

## 创建iframe

预览部分会先创建一个`iframe`：

```ts
onMounted(createSandbox)

let sandbox: HTMLIFrameElement
function createSandbox() {
    // ...
    sandbox = document.createElement('iframe')
    sandbox.setAttribute(
        'sandbox',
        [
            'allow-forms',
            'allow-modals',
            'allow-pointer-lock',
            'allow-popups',
            'allow-same-origin',
            'allow-scripts',
            'allow-top-navigation-by-user-activation'
        ].join(' ')
    )
    // ...
}
```

创建一个`iframe`元素，并且设置了`sandbox`属性，这个属性可以控制`iframe`框架中的页面的一些行为是否被允许，详情[arrt-sand-box](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/iframe#attr-sandbox)。

```ts
import srcdoc from './srcdoc.html?raw'

function createSandbox() {
    // ...
    // 检查importMap是否合法
    const importMap = store.getImportMap()
    if (!importMap.imports) {
        importMap.imports = {}
    }
    if (!importMap.imports.vue) {
        importMap.imports.vue = store.state.vueRuntimeURL
    }
    // 向框架页面内容中注入import-map
    const sandboxSrc = srcdoc.replace(
        /<!--IMPORT_MAP-->/,
        JSON.stringify(importMap)
    )
    // 将页面HTML内容注入框架
    sandbox.srcdoc = sandboxSrc
    // 添加框架到页面
    container.value.appendChild(sandbox)
    // ...
}
```

`srcdoc.html`就是用于预览的页面，会先注入`import-map`的内容，然后通过创建的`iframe`渲染该页面。

```ts
let proxy: PreviewProxy
function createSandbox() {
    // ...
    proxy = new PreviewProxy(sandbox, {
        on_error:() => {}
        // ...
    })
    sandbox.addEventListener('load', () => {
        stopUpdateWatcher = watchEffect(updatePreview)
    })
}
```

接下来创建了一个`PreviewProxy`类的实例，最后在`iframe`加载完成时注册一个副作用函数`updatePreview`，这个方法内会处理文件并进行预览操作。

## 和iframe通信

`PreviewProxy`类主要是用来和`iframe`通信的：

```ts
export class PreviewProxy {
  constructor(iframe: HTMLIFrameElement, handlers: Record<string, Function>) {
    this.iframe = iframe
    this.handlers = handlers

    this.pending_cmds = new Map()

    this.handle_event = (e) => this.handle_repl_message(e)
    window.addEventListener('message', this.handle_event, false)
  }
}
```

`message`事件可以监听来自`iframe`的信息，向`iframe`发送信息是通过`postMessage`方法：

```ts
export class PreviewProxy {
  iframe_command(action: string, args: any) {
    return new Promise((resolve, reject) => {
      const cmd_id = uid++

      this.pending_cmds.set(cmd_id, { resolve, reject })

      this.iframe.contentWindow!.postMessage({ action, cmd_id, args }, '*')
    })
  }
}
```

通过这个方法可以向`iframe`发送消息，返回一个`promise`，发消息前会生成一个唯一的`id`，然后把`promise`的`resolve`和`reject`通过`id`保存起来，并且这个`id`会发送给`iframe`，当`iframe`任务执行完了会向父窗口回复信息，并且会发回这个`id`，那么父窗口就可以通过这个`id`取出`resove`和`reject`根据函数根据任务执行的成功与否决定调用哪个。

`iframe`向父级发送信息的方法：

```ts
// srcdoc.html
window.addEventListener('message', handle_message, false);

async function handle_message(ev) {
  // 取出任务名称和id
  let { action, cmd_id } = ev.data;
  // 向父级发送消息
  const send_message = (payload) => parent.postMessage( { ...payload }, ev.origin);
  // 回复父级，会带回id
  const send_reply = (payload) => send_message({ ...payload, cmd_id });
  // 成功的回复
  const send_ok = () => send_reply({ action: 'cmd_ok' });
  // 失败的回复
  const send_error = (message, stack) => send_reply({ action: 'cmd_error', message, stack });
  // 根据actiion判断执行什么任务
  // ...
}
```

## 编译模块进行预览

接下来看一下`updatePreview`方法，这个方法内会再一次编译文件，得到模块列表，其实就是`js`代码，然后将模块列表发送给`iframe`，`iframe`会动态创建`script`标签插入这些模块代码，达到更新`iframe`页面进行预览的效果。

```ts
async function updatePreview() {
  // ...
  try {
    // 编译文件生成模块代码
    const modules = compileModulesForPreview(store)
    // 待插入到iframe页面中的代码
    const codeToEval = [
      `window.__modules__ = {}\nwindow.__css__ = ''\n` +
        `if (window.__app__) window.__app__.unmount()\n` +
        `document.body.innerHTML = '<div id="app"></div>'`,
      ...modules,
      `document.getElementById('__sfc-styles').innerHTML = window.__css__`
    ]
    // 如果入口文件时Vue文件，那么添加挂载它的代码！
    if (mainFile.endsWith('.vue')) {
      codeToEval.push(
        `import { createApp } as _createApp } from "vue"
        const _mount = () => {
          const AppComponent = __modules__["${mainFile}"].default
          AppComponent.name = 'Repl'
          const app = window.__app__ = _createApp(AppComponent)
          app.config.unwrapInjectedRef = true
          app.config.errorHandler = e => console.error(e)
          app.mount('#app')
        }
        _mount()
      )`
    }
    // 给iframe页面发送消息，插入这些模块代码
    await proxy.eval(codeToEval)
  } catch (e: any) {
    // ...
  }
}
```

`codeToEval`数组揭示了预览的原理，`codeToEval`数组的内容最后是会发送到`iframe`页面中，然后动态创建`script`标签插入到页面进行运行的。

首先我们再添加一个文件`Comp.vue`：

```html
<script setup>
import { ref } from 'vue'
const msg = ref('我是子组件')
</script>

<template>
  <h1>{{ msg }}</h1>
</template>
```

然后在`App.vue`组件中引入：

```html
<script setup>
import { ref } from 'vue'
import Comp from './Comp.vue'// ++
const msg = ref('Hello World!')
</script>

<template>
  <h1>{{ msg }}</h1>
  <input v-model="msg">
  <Comp></Comp>// ++ 
</template>

<style>
  h1 {
    color: red;
  }
</style>
```

此时经过上一节【文件编译】处理后，`Comp.vue`的编译结果如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ebada0d671b4380b48026c59b0fd97e~tplv-k3u1fbpfcp-zoom-1.image)

`App.vue`的编译结果如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f99eb29c6604a959f634a0c77ea6ee9~tplv-k3u1fbpfcp-zoom-1.image)

`compileModulesForPreview`会再一次编译各个文件，主要是做以下几件事情：

1.将模块的导出语句`export`转换成属性添加语句，也就是把模块添加到`window.__modules__`对象上：

```ts
const __sfc__ = {
  __name: 'Comp',
  // ...
}

export default __sfc__
```

转换成：

```ts
const __module__ = __modules__["Comp.vue"] = { [Symbol.toStringTag]: "Module" }

__module__.default = __sfc__
```

2.将`import`了相对路径的模块`./`的语句转成赋值的语句，这样可以从`__modules__`对象上获取到指定模块：

```ts
import Comp from './Comp.vue'
```

转换成：

```ts
const __import_1__ = __modules__["Comp.vue"]
```

3.最后再转换一下导入的组件使用到的地方：

```ts
_createVNode(Comp)
```

转换成：

```ts
_createVNode(__import_1__.default)
```

4.如果该组件存在样式，那么追加到`window.__css__`字符串上：

```ts
if (file.compiled.css) {
    js += `\nwindow.__css__ += ${JSON.stringify(file.compiled.css)}`
}
```

此时再来看`codeToEval`数组的内容就很清晰了，首先创建一个全局对象`window.__modules__`、一个全局字符串`window.__css__`，如果之前已经存在`__app__`实例，说明是更新情况，那么先卸载之前的组件，然后在页面中创建一个`id`为`app`的`div`元素用于挂载`Vue`组件，接下来添加`compileModulesForPreview`方法编译返回的模块数组，这样这些组件运行时全局变量都已定义好了，组件有可能会往`window.__css__`上添加样式，所以当所有组件运行完后再将`window.__css__`样式添加到页面。

最后，如果入口文件是`Vue`组件，那么会再添加一段`Vue`的实例化和挂载代码。

`compileModulesForPreview`方法比较长，做的事情大致就是从入口文件开始，按前面的4点转换文件，然后递归所有依赖的组件也进行转换，具体的转换方式是使用`babel`将模块转换成`AST`树，然后使用[magic-string](https://github.com/Rich-Harris/magic-string)修改源代码，这种代码对于会的人来说很简单，对于没有接触过`AST`树操作的人来说就很难看懂，所以具体代码就不贴了，有兴趣查看具体实现的可以点击[moduleCompiler.ts](https://github.com/vuejs/repl/blob/main/src/output/moduleCompiler.ts#L82)。

`codeToEval`数组内容准备好了，就可以给预览的`iframe`发送消息了：

```ts
await proxy.eval(codeToEval)
```

`iframe`接收到消息后会先删除之前添加的`script`标签，然后创建新标签：

```ts
// scrdoc.html
async function handle_message(ev) {
    let { action, cmd_id } = ev.data;
    // ...
    if (action === 'eval') {
        try {
            // 移除之前创建的标签
            if (scriptEls.length) {
                scriptEls.forEach(el => {
                    document.head.removeChild(el)
                })
                scriptEls.length = 0
            }
			// 遍历创建script标签
            let { script: scripts } = ev.data.args
            if (typeof scripts === 'string') scripts = [scripts]
            for (const script of scripts) {
                const scriptEl = document.createElement('script')
                scriptEl.setAttribute('type', 'module')
                const done = new Promise((resolve) => {
                    window.__next__ = resolve
                })
                scriptEl.innerHTML = script + `\nwindow.__next__()`
                document.head.appendChild(scriptEl)
                scriptEls.push(scriptEl)
                await done
            }
        }
        // ...
    }
}
```

为了让模块按顺序挨个添加，会创建一个`promise`，并且把`resove`方法赋值到一个全局的属性`__next__`上，然后再在每个模块最后拼接上调用的代码，这样当插入一个`script`标签时，该标签的代码运行完毕会执行`window.__next__`方法，那么就会结束当前的`promise`，进入下一个`script`标签的插件，不得不说，还是很巧妙的。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/198d0882e8404026a566b2fef58c4a54~tplv-k3u1fbpfcp-zoom-1.image)


# 总结

本文从源码角度来看了一下`@vue/repl`组件的实现，其实忽略了挺多内容，比如`ssr`相关的、使用`html`作为入口文件、信息输出等，有兴趣的可以自行阅读源码。

因为该组件不支持运行`Vue2`，所以我的一个同事`fork`修改创建了一个`Vue2`的版本，有需求的可以关注一下[vue2-repl](https://github.com/Thy3634/vue2-repl)。

最后也推荐一下我的开源项目，也是一个在线`Playground`，也支持`Vue2`和`Vue3`单文件，不过更通用一些，但是不支持创建多个文件，有兴趣的可以关注一下[code-run](https://github.com/wanglin2/code-run)。















