持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第2天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

> 本文为`Varlet`组件库源码主题阅读系列第二篇，读完本篇，你可以了解到如何将一个Vue3组件库打包成各种格式

[上一篇](https://juejin.cn/post/7151661990925041677)里提到了启动服务前会先进行一下组件库的打包，运行的命令为：

```
varlet-cli compile
```

显然是`varlet-cli`包提供的一个命令：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0516d6295ef4fd9adc6a99793297e96~tplv-k3u1fbpfcp-zoom-1.image)


处理函数为`compile`，接下来我们详细看一下这个函数都做了什么。

```ts
// varlet-cli/src/commands/compile.ts
export async function compile(cmd: { noUmd: boolean }) {
    process.env.NODE_ENV = 'compile'
    await removeDir()
    // ...
}

// varlet-cli/src/commands/compile.ts
export function removeDir() {
    // ES_DIR：varlet-ui/es
    // LIB_DIR：varlet-ui/lib
    // HL_DIR：varlet-ui/highlight
    // UMD_DIR：varlet-ui/umd
    return Promise.all([remove(ES_DIR), remove(LIB_DIR), remove(HL_DIR), remove(UMD_DIR)])
}
```

首先设置了一下当前的环境变量，然后清空相关的输出目录。

```ts
// varlet-cli/src/commands/compile.ts
export async function compile(cmd: { noUmd: boolean }) {
    // ...
    process.env.TARGET_MODULE = 'module'
    await runTask('module', compileModule)

    process.env.TARGET_MODULE = 'esm-bundle'
    await runTask('esm bundle', () => compileModule('esm-bundle'))

    process.env.TARGET_MODULE = 'commonjs'
    await runTask('commonjs', () => compileModule('commonjs'))

    process.env.TARGET_MODULE = 'umd'
    !cmd.noUmd && (await runTask('umd', () => compileModule('umd')))
}
```

接下来依次打包了四种类型的产物，方法都是同一个`compileModule`，这个方法后面会详细分析。

# 组件的基本组成

以`Button`组件为例看一下未打包前的组件结构：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a96823127834fcca74d78517f81f21e~tplv-k3u1fbpfcp-zoom-1.image)


一个典型组件的构成主要是四个文件：

>.less：样式
>
>.vue：组件
>
>index.ts：导出组件，提供组件注册方法
>
>props.ts：组件的props定义 

样式部分`Varlet`使用的是`less`语言，样式比较少的话会直接内联写到`Vue`单文件的`style`块中，否则会单独创建一个样式文件，比如图中的`button.less`，每个组件除了引入自己本身的样式外，还会引入一些基本样式、其他组件的样式：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/240b176838ff466caa786fe3032283a2~tplv-k3u1fbpfcp-zoom-1.image)


`index.ts`文件用来导出组件，提供组件的注册方法：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4110509145f64942990184be824add2c~tplv-k3u1fbpfcp-zoom-1.image)


`props.ts`文件用来声明组件的`props`类型：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c98420e8d9464091a403b4d151ec1190~tplv-k3u1fbpfcp-zoom-1.image)


有的组件没有使用`.vue`，而是`.tsx`，也有些组件会存在其他文件，比如有些组件就还存在一个`provide.ts`文件，用于向子孙组件注入数据。

# 打包的整体流程

首先大致过一遍整体的打包流程，主要函数为`compileModule`：

```ts
// varlet-cli/src/compiler/compileModule.ts
export async function compileModule(modules: 'umd' | 'commonjs' | 'esm-bundle' | boolean = false) {
  if (modules === 'umd') {
    // 打包umd格式
    await compileUMD()
    return
  }

  if (modules === 'esm-bundle') {
    // 打包esm-bundle格式
    await compileESMBundle()
    return
  }
    
  // 打包commonjs和module格式
  // 打包前设置一下环境变量
  process.env.BABEL_MODULE = modules === 'commonjs' ? 'commonjs' : 'module'
  // 输出目录
  // ES_DIR：varlet-ui/es
  // LIB_DIR：varlet-ui/lib
  const dest = modules === 'commonjs' ? LIB_DIR : ES_DIR
  // SRC_DIR：varlet-ui/src，直接将组件的源码目录复制到输出目录
  await copy(SRC_DIR, dest)
  // 读取输出目录
  const moduleDir: string[] = await readdir(dest)
  // 遍历打包每个组件
  await Promise.all(
    // 遍历每个组件目录
    moduleDir.map((filename: string) => {
      const file: string = resolve(dest, filename)
      if (isDir(file)) {
        // 在每个组件目录下新建两个样式入口文件
        ensureFileSync(resolve(file, './style/index.js'))
        ensureFileSync(resolve(file, './style/less.js'))
      }
      // 打包组件
      return isDir(file) ? compileDir(file) : null
    })
  )
  // 遍历varlet-ui/src/目录，找出所有存在['index.vue', 'index.tsx', 'index.ts', 'index.jsx', 'index.js']这些文件之一的目录
  const publicDirs = await getPublicDirs()
  // 生成整体的入口文件
  await (modules === 'commonjs' ? compileCommonJSEntry(dest, publicDirs) : compileESEntry(dest, publicDirs))
}
```

`umd`和`esm-bundle`两种格式都会把所有内容都打包到一个文件，用的是`Vite`提供的方法进行打包。

`commonjs`和`module`是单独打包每个组件，不会把所有组件的内容都打包到一起，`Vite`没有提供这个能力，所以需要自行处理，具体操作为：

- 先把组件源码目录`varlet/src/`下的所有组件文件都复制到对应的输出目录下；
- 然后在输出目录遍历每个组件目录：
  - 创建两个样式的导出文件；
  - 删除不需要的目录、文件（测试、示例、文档）；
  - 分别编译`Vue`单文件、`ts`文件、`less`文件；
- 全部打包完成后，遍历所有组件，动态生成整体的导出文件；

以`compileESEntry`方法为例看一下整体导出文件的生成：

```ts
// varlet-cli/src/compiler/compileScript.ts
export async function compileESEntry(dir: string, publicDirs: string[]) {
  const imports: string[] = []
  const plugins: string[] = []
  const constInternalComponents: string[] = []
  const cssImports: string[] = []
  const lessImports: string[] = []
  const publicComponents: string[] = []
  // 遍历组件目录名称
  publicDirs.forEach((dirname: string) => {
    // 连字符转驼峰式
    const publicComponent = bigCamelize(dirname)
    // 收集组件名称
    publicComponents.push(publicComponent)
    // 收集组件导入语句
    imports.push(`import ${publicComponent}, * as ${publicComponent}Module from './${dirname}'`)
    // 收集内部组件导入语句
    constInternalComponents.push(
      `export const _${publicComponent}Component = ${publicComponent}Module._${publicComponent}Component || {}`
    )
    // 收集插件注册语句
    plugins.push(`${publicComponent}.install && app.use(${publicComponent})`)
    // 收集样式导入语句
    cssImports.push(`import './${dirname}/style'`)
    lessImports.push(`import './${dirname}/style/less'`)
  })

  // 拼接组件注册方法
  const install = `
function install(app) {
  ${plugins.join('\n  ')}
}
`

  // 拼接导出入口index.js文件的内容，注意它是不包含样式的
  const indexTemplate = `\
${imports.join('\n')}\n
${constInternalComponents.join('\n')}\n
${install}
export {
  install,
  ${publicComponents.join(',\n  ')}
}

export default {
  install,
  ${publicComponents.join(',\n  ')}
}
`
  
  // 拼接css导入语句
  const styleTemplate = `\
${cssImports.join('\n')}
`

  // 拼接umdIndex.js文件，这个文件是用于后续打包umd和esm-bundle格式时作为打包入口，注意它是包含样式导入语句的
  const umdTemplate = `\
${imports.join('\n')}\n
${cssImports.join('\n')}\n
${install}
export {
  install,
  ${publicComponents.join(',\n  ')}
}

export default {
  install,
  ${publicComponents.join(',\n  ')}
}
`

  // 拼接less导入语句
  const lessTemplate = `\
${lessImports.join('\n')}
`
  // 将拼接的内容写入到对应文件
  await Promise.all([
    writeFile(resolve(dir, 'index.js'), indexTemplate, 'utf-8'),
    writeFile(resolve(dir, 'umdIndex.js'), umdTemplate, 'utf-8'),
    writeFile(resolve(dir, 'style.js'), styleTemplate, 'utf-8'),
    writeFile(resolve(dir, 'less.js'), lessTemplate, 'utf-8'),
  ])
}
```



# 打包成module和commonjs格式

打包成`umd`和`esm-bundle`两种格式依赖`module`格式的打包产物，而打包成`module`和`commonjs`两种格式是同一套逻辑，所以我们先来看看是如何打包成这两种格式的。

这两种格式就是单独打包每个组件，生成单独的入口文件和样式文件，然后再生成一个统一的导出入口，不会把所有组件的内容都打包到同一个文件，方便按需引入，去除不需要的内容，减少文件体积。

打包每个组件的`compileDir`方法：

```ts
// varlet-cli/src/compiler/compileModule.ts
export async function compileDir(dir: string) {
  // 读取组件目录
  const dirs = await readdir(dir)
  // 遍历组件目录下的文件
  await Promise.all(
    dirs.map((filename) => {
      const file = resolve(dir, filename)
      // 删除组件目录下的__test__目录、example目录、docs目录
      ;[TESTS_DIR_NAME, EXAMPLE_DIR_NAME, DOCS_DIR_NAME].includes(filename) && removeSync(file)
      // 如果是.d.ts文件或者是style目录（前面为样式入口文件创建的目录）直接返回
      if (isDTS(file) || filename === STYLE_DIR_NAME) {
        return Promise.resolve()
      }
      // 编译文件
      return compileFile(file)
    })
  )
}
```

删除了不需要的目录，然后针对需要编译的文件调用了`compileFile`方法：

```ts
// varlet-cli/src/compiler/compileModule.ts
export async function compileFile(file: string) {
  isSFC(file) && (await compileSFC(file))// 编译vue文件
  isScript(file) && (await compileScriptFile(file))// 编译js文件
  isLess(file) && (await compileLess(file))// 编译less文件
  isDir(file) && (await compileDir(file))// 如果是目录则进行递归
}
```

分别处理三种文件，让我们一一来看。

## 编译Vue单文件

```ts
// varlet-cli/src/compiler/compileSFC.ts
import { parse } from '@vue/compiler-sfc'

export async function compileSFC(sfc: string) {
    // 读取Vue单文件内容
    const sources: string = await readFile(sfc, 'utf-8')
    // 使用@vue/compiler-sfc包解析单文件
    const { descriptor } = parse(sources, { sourceMap: false })
    // 取出单文件的每部分内容
    const { script, scriptSetup, template, styles } = descriptor
    // Varlet暂时不支持setup语法
    if (scriptSetup) {
        logger.warning(
            `\n Varlet Cli does not support compiling script setup syntax\
\n  The error in ${sfc}`
        )
        return
    }
    // ...
}
```

使用[@vue/compiler-sfc](https://github.com/vuejs/core/tree/main/packages/compiler-sfc)包来解析`Vue`单文件，`parse`方法可以解析出`Vue`单文件中的各个块，针对各个块，`@vue/compiler-sfc`包都提供了相应的编译方法，后续都会涉及到。

```ts
// varlet-cli/src/compiler/compileSFC.ts
import hash from 'hash-sum'

export async function compileSFC(sfc: string) {
    // ...
    // scoped
    // 检查是否存在scoped作用域的样式块
    const hasScope = styles.some((style) => style.scoped)
    // 将单文件的内容进行hash生成id
    const id = hash(sources)
    // 生成样式的scopeId
    const scopeId = hasScope ? `data-v-${id}` : ''
    // ...
}
```

这一步主要是检查`style`块是否存在作用域块，存在的话会生成一个作用域`id`，作为`css`的作用域，防止和其他样式冲突，这两个`id`相关的编译方法需要用到。

```ts
// varlet-cli/src/compiler/compileSFC.ts
import { compileTemplate } from '@vue/compiler-sfc'

export async function compileSFC(sfc: string) {
    // ...
    if (script) {
        // template
        // 编译模板为渲染函数
        const render =
              template &&
              compileTemplate({
                  id,
                  source: template.content,
                  filename: sfc,
                  compilerOptions: {
                      scopeId,
                  },
              })
	// 注入render函数
        let { content } = script
        if (render) {
            const { code } = render
            content = injectRender(content, code)
        }
        // ...
    }
}
```

使用`@vue/compiler-sfc`包的`compileTemplate`方法将解析出的模板部分编译为渲染函数，然后调用`injectRender`方法将渲染函数注入到`script`中：

```ts
// varlet-cli/src/compiler/compileSFC.ts
const NORMAL_EXPORT_START_RE = /export\s+default\s+{/
const DEFINE_EXPORT_START_RE = /export\s+default\s+defineComponent\s*\(\s*{/

export function injectRender(script: string, render: string): string {
  if (DEFINE_EXPORT_START_RE.test(script.trim())) {
    return script.trim().replace(
      DEFINE_EXPORT_START_RE,
      `${render}\nexport default defineComponent({
  render,\
    `
    )
  }
  if (NORMAL_EXPORT_START_RE.test(script.trim())) {
    return script.trim().replace(
      NORMAL_EXPORT_START_RE,
      `${render}\nexport default {
  render,\
    `
    )
  }
  return script
}
```

兼容两种导出方式，以一个小例子来看一下，比如生成的渲染函数为：

```js
export function render(_ctx, _cache) {
    // ...
}
```

`script`的内容为：

```js
export default defineComponent({
    name: 'VarButton',
    // ...
})
```

注入`render`后`script`的内容变成了：

```js
export function render(_ctx, _cache) {
    // ...
}
export default defineComponent({
    render,
    name: 'VarButton',
    /// ...
})
```

其实就是把渲染函数的内容和`script`的内容合并了，`script`其实就是组件的选项对象，所以同时也把组件的渲染函数添加到组件对象上。

继续`compileSFC`方法：

```ts
// varlet-cli/src/compiler/compileSFC.ts
import { compileStyle } from '@vue/compiler-sfc'

export async function compileSFC(sfc: string) {
    // ...
    if (script) {
        // ...
        // script
        // 编译js
        await compileScript(content, sfc)
        // style
        // 编译样式
        for (let index = 0; index < styles.length; index++) {
          const style: SFCStyleBlock = styles[index]
          // replaceExt方法接收文件名称，比如xxx.vue，然后使用第二个参数替换文件名称的扩展名，比如处理完会返回xxxSfc.less
          const file = replaceExt(sfc, `Sfc${index || ''}.${style.lang || 'css'}`)
          // 编译样式块
          let { code } = compileStyle({
            source: style.content,
            filename: file,
            id: scopeId,
            scoped: style.scoped,
          })
          // 去除样式中的导入语句
          code = extractStyleDependencies(file, code, STYLE_IMPORT_RE, style.lang as 'css' | 'less', true)
          // 将解析后的样式写入文件
          writeFileSync(file, clearEmptyLine(code), 'utf-8')
          // 如果样式块是less语言，那么同时也编译成css文件
          style.lang === 'less' && (await compileLess(file))
        }
    }
}
```

调用了`compileScript`方法编译`script`内容，这个方法我们下一小节再说。然后遍历`style`块，每个块都会生成相应的样式文件，比如`Button.vue`组件存在一个`less`语言的`style`块


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0f2837312ab418e8f2021d521fa86f4~tplv-k3u1fbpfcp-zoom-1.image)


那么会生成一个`ButtonSfc.less`，因为是`less`，所以同时也会再编译生成一个`ButtonSfc.css`文件，当然这两个样式文件里只包括内联在`Vue`单文件中的样式，不包括使用`@import`导入的样式，所以生成的这两个样式文件都是空的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f63c2af9639453080bb6a2bd1f69071~tplv-k3u1fbpfcp-zoom-1.image)


编译样式块使用的是`@vue/compiler-sfc`的`compileStyle`方法，它会帮我们处理`<style scoped>`, `<style module> `以及`css`变量注入的问题。

`extractStyleDependencies`方法会提取并去除样式中的导入语句：

```ts
// varlet-cli/src/compiler/compileStyle.ts
import { parse, resolve } from 'path'

export function extractStyleDependencies(
  file: string,
  code: string,
  reg: RegExp,//     /@import\s+['"](.+)['"]\s*;/g
  expect: 'css' | 'less',
  self: boolean
) {
  const { dir, base } = parse(file)
  // 用正则匹配出样式导入语句
  const styleImports = code.match(reg) ?? []
  // 这两个文件是之前创建的
  const cssFile = resolve(dir, './style/index.js')
  const lessFile = resolve(dir, './style/less.js')
  const modules = process.env.BABEL_MODULE
  // 遍历导入语句
  styleImports.forEach((styleImport: string) => {
    // 去除导入源的扩展名及处理导入的路径，因为index.js和less.js两个文件和Vue单文件不在同一个层级，所以导入的相对路径需要修改一下
    const normalizedPath = normalizeStyleDependency(styleImport, reg)
    // 将导入语句写入创建的两个文件中
    smartAppendFileSync(
      cssFile,
      modules === 'commonjs' ? `require('${normalizedPath}.css')\n` : `import '${normalizedPath}.css'\n`
    )
    smartAppendFileSync(
      lessFile,
      modules === 'commonjs' ? `require('${normalizedPath}.${expect}')\n` : `import '${normalizedPath}.${expect}'\n`
    )
  })
  // 上面已经把Vue单文件中style块内的导入语句提取出去了，另外之前也提到了每个style块本身也会创建一个样式文件，所以导入这个文件的语句也需要追加进去：
  if (self) {
    smartAppendFileSync(
      cssFile,
      modules === 'commonjs'
        ? `require('${normalizeStyleDependency(base, reg)}.css')\n`
        : `import '${normalizeStyleDependency(base, reg)}.css'\n`
    )
    smartAppendFileSync(
      lessFile,
      modules === 'commonjs'
        ? `require('${normalizeStyleDependency(base, reg)}.${expect}')\n`
        : `import '${normalizeStyleDependency(base, reg)}.${expect}'\n`
    )
  }
  // 去除样式中的导入语句
  return code.replace(reg, '')
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c88942d2160249a283d4a24916aad77e~tplv-k3u1fbpfcp-zoom-1.image)


到这里，一共生成了四个文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1900b0feea8a4a37a12b7fadcac02d10~tplv-k3u1fbpfcp-zoom-1.image)




## 编译less文件

`script`部分的编译比较复杂，我们最后再看，先看一下`less`文件的处理。

```ts
// varlet-cli/src/compiler/compileStyle.ts
import { render } from 'less'

export async function compileLess(file: string) {
  const source = readFileSync(file, 'utf-8')
  const { css } = await render(source, { filename: file })

  writeFileSync(replaceExt(file, '.css'), clearEmptyLine(css), 'utf-8')
}
```

很简单，使用`less`包将`less`编译成`css`，然后写入文件即可，到这里又生成了一个`css`文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00fda53e552c4a52b9e36ffddd37c65d~tplv-k3u1fbpfcp-zoom-1.image)



## 编译script文件

`script`部分，主要是`ts`、`tsx`文件，`Varlet`大部分组件是使用`Vue`单文件编写的，不过也有少数组件使用的是`tsx`，编译调用了`compileScriptFile`方法：

```ts
// varlet-cli/src/compiler/compileScript.ts
export async function compileScriptFile(file: string) {
  const sources = readFileSync(file, 'utf-8')

  await compileScript(sources, file)
}
```

读取文件，然后调用`compileScript`方法，前面`Vue`单文件中解析出来的`script`部分内容调用的也是这个方法。

### 兼容模块导入

```ts
// varlet-cli/src/compiler/compileScript.ts
export async function compileScript(script: string, file: string) {
  const modules = process.env.BABEL_MODULE
  // 兼容模块导入
  if (modules === 'commonjs') {
    script = moduleCompatible(script)
  }
  // ...
}
```

首先针对`commonjs`做了一下兼容处理：

```ts
// varlet-cli/src/compiler/compileScript.ts
export const moduleCompatible = (script: string): string => {
  const moduleCompatible = get(getVarletConfig(), 'moduleCompatible', {})
  Object.keys(moduleCompatible).forEach((esm) => {
    const commonjs = moduleCompatible[esm]
    script = script.replace(esm, commonjs)
  })
  return script
}
```

替换一些导入语句，`Varlet`组件开发是基于`ESM`规范的，使用其他库时导入的肯定也是`ESM`版本，所以编译成`commonjs`模块时需要修改成对应的`commonjs`版本，`Varlet`引入的第三方库不多，主要就是`dayjs`：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1406f2b4b73b44cf9e710a69d17a33e3~tplv-k3u1fbpfcp-zoom-1.image)


### 使用babel编译

继续`compileScript`方法：

```ts
// varlet-cli/src/compiler/compileScript.ts
import { transformAsync } from '@babel/core'

export async function compileScript(script: string, file: string) {
  // ...
  // 使用babel编译js
  let { code } = (await transformAsync(script, {
    filename: file,// js内容对应的文件名，babel插件会用到
  })) as BabelFileResult
  // ...
}
```

接下来使用[@babel/core](https://babeljs.io/docs/en/babel-core)包编译`js`内容，`transformAsync`方法会使用本地的配置文件，因为打包命令是在`varlet-ui/`目录下运行的，所以`babel`会在这个目录下寻找配置文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9cb70f4ccb24218a20517b6773c9922~tplv-k3u1fbpfcp-zoom-1.image)


编译成`module`还是`commonjs`格式的判断也在这个配置中，有关配置的详解，有兴趣的可以阅读最后的附录小节。

### 提取样式导入语句

继续`compileScript`方法：

```ts
// varlet-cli/src/compiler/compileScript.ts
export const REQUIRE_CSS_RE = /(?<!['"`])require\(\s*['"](\.{1,2}\/.+\.css)['"]\s*\);?(?!\s*['"`])/g
export const REQUIRE_LESS_RE = /(?<!['"`])require\(\s*['"](\.{1,2}\/.+\.less)['"]\s*\);?(?!\s*['"`])/g
export const IMPORT_CSS_RE = /(?<!['"`])import\s+['"](\.{1,2}\/.+\.css)['"]\s*;?(?!\s*['"`])/g
export const IMPORT_LESS_RE = /(?<!['"`])import\s+['"](\.{1,2}\/.+\.less)['"]\s*;?(?!\s*['"`])/g

export async function compileScript(script: string, file: string) {
    // ...
    code = extractStyleDependencies(
        file,
        code as string,
        modules === 'commonjs' ? REQUIRE_CSS_RE : IMPORT_CSS_RE,
        'css'
    )
    code = extractStyleDependencies(
        file,
        code as string,
        modules === 'commonjs' ? REQUIRE_LESS_RE : IMPORT_LESS_RE,
        'less'
    )
    // ...
}
```

`extractStyleDependencies`方法前面已经介绍了，所以这一步的操作就是提取并去除`script`内的样式导入语句。

### 转换其他导入语句

```ts
// varlet-cli/src/compiler/compileScript.ts
export async function compileScript(script: string, file: string) {
    // ...
    code = replaceVueExt(code as string)
    code = replaceTSXExt(code as string)
    code = replaceJSXExt(code as string)
    code = replaceTSExt(code as string)
    // ...
}
```

这一步的操作是把`script`中的各种类型的导入语句都修改为导入`.js`文件，因为这些文件最后都会被编译成`js`文件，比如`button/index.ts`文件内导入了`Button.vue`组件：

```ts
import Button from './Button.vue'
// ...
```

转换后会变成：

```ts
import Button from './Button.js'
// ...
```

继续：

```ts
// varlet-cli/src/compiler/compileScript.ts
export async function compileScript(script: string, file: string) {
    // ...
    removeSync(file)
    writeFileSync(replaceExt(file, '.js'), code, 'utf8')
}
```

最后就是把处理完的`script`内容写入文件。

到这里`.vue`，`.ts`、`.tsx`文件都已处理完毕：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba54b86cc1ed4e24addb9c31fa6007f1~tplv-k3u1fbpfcp-zoom-1.image)


### 小节

到这里，打包成`module`和`commonjs`格式就完成了，总结一下所做的事情：

- `less`文件直接使用`less`包编译成同名的`css`文件；
- `ts`、`tsx`等文件使用`babel`编译成`js`文件；提取并去除其中的样式导入语句，并将该样式导入语句写入单独的文件、修改`.vue`、`.ts`等类型的导入语句来源为对应的编译后的`js`路径；
- `Vue`单文件使用`@vue/compiler-sfc`解析并对各个块分别使用对应的函数进行编译；每个`style`块也会提取并去除其中的样式导入语句，并将该导入语句写入单独的文件，剩下的样式内容会分别创建一个对应的样式文件，如果是`less`块，同时会编译并创建一个同名的`css`文件；`template`的编译结果会合并到`script`内，然后`script`的内容会重复上一步`ts`文件的处理逻辑；
- 所有组件都编译完了，再动态创建整体的导出文件，一共生成了四个文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b02f0f4c8164df2a517205abb6331a5~tplv-k3u1fbpfcp-zoom-1.image)


# 打包成esm-bundle

打包成`esm-bundle`格式调用的是`compileESMBundle`方法：

```ts
// varlet-cli/src/compiler/compileModule.ts
import { build } from 'vite'

export function compileESMBundle() {
  return new Promise<void>((resolve, reject) => {
    const config = getESMBundleConfig(getVarletConfig())

    build(config)
      .then(() => resolve())
      .catch(reject)
  })
}
```

`getVarletConfig`方法会把`varlet-cli/varlet.default.config.js`和`varlet-ui/varlet.config.js`两个配置进行合并，看一下`getESMBundleConfig`方法：

```ts
// varlet-cli/src/config/vite.config.js
export function getESMBundleConfig(varletConfig: Record<string, any>): InlineConfig {
  const name = get(varletConfig, 'name')// name默认为Varlet
  const fileName = `${kebabCase(name)}.esm.js`// 输出文件名，varlet.esm.js

  return {
    logLevel: 'silent',
    build: {
      emptyOutDir: true,// 清空输出目录
      lib: {// 指定构建为库
        name,// 库暴露的全局变量
        formats: ['es'],// 构建格式
        fileName: () => fileName,// 打包出口
        entry: resolve(ES_DIR, 'umdIndex.js'),// 打包入口
      },
      rollupOptions: {// 传给rollup的配置
        external: ['vue'],// 外部化处理不需要打包进库的依赖
        output: {
          dir: ES_DIR,// 输出目录，ES_DIR：varlet-ui/es
          exports: 'named',// 既存在命名导出，也存在默认导出，所以设置为named，详情：https://rollupjs.org/guide/en/#outputexports
          globals: {// 在umd构建模式下为外部化的依赖提供一个全局变量
            vue: 'Vue',
          },
        },
      },
    },
    plugins: [clear()],
  }
}
```

其实就是使用如上的配置来调用`Vite`的`build`方法进行打包，可参考[库模式](https://www.vitejs.net/guide/build.html#library-mode)，可以看到打包入口为前面打包`module`格式时生成的`umdIndex.js`文件。

因为`Vite`开发环境使用的是`esbuild`，生产环境打包使用的是`rollup`，所以想要深入玩转`Vite`，这几个东西都需要了解，包括各自的配置选项、插件开发等，还是不容易的。

打包完成后会在`varlet-ui/es/`目录下生成两个文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ac405ce391e41d99cde7720b07b63e3~tplv-k3u1fbpfcp-zoom-1.image)





# 打包成umd格式

打包成`umd`格式调用的是`compileUMD`方法：

```ts
// varlet-cli/src/compiler/compileModule.ts
import { build } from 'vite'

export function compileUMD() {
  return new Promise<void>((resolve, reject) => {
    const config = getUMDConfig(getVarletConfig())

    build(config)
      .then(() => resolve())
      .catch(reject)
  })
}
```

整体和打包`esm-bundle`是一样的，只不过获取的配置不一样：

```ts
// varlet-cli/src/config/vite.config.js
export function getUMDConfig(varletConfig: Record<string, any>): InlineConfig {
  const name = get(varletConfig, 'name')// name默认为Varlet
  const fileName = `${kebabCase(name)}.js`// 将驼峰式转换成-连接

  return {
    logLevel: 'silent',
    build: {
      emptyOutDir: true,
      lib: {
        name,
        formats: ['umd'],// 设置为umd
        fileName: () => fileName,
        entry: resolve(ES_DIR, 'umdIndex.js'),// ES_DIR：varlet-ui/es，打包入口
      },
      rollupOptions: {
        external: ['vue'],
        output: {
          dir: UMD_DIR,// 输出目录，UMD_DIR：varlet-ui/umd
          exports: 'named',
          globals: {
            vue: 'Vue',
          },
        },
      },
    },
    // 使用了两个插件，作用如其名
    plugins: [inlineCSS(fileName, UMD_DIR), clear()],
  }
}
```

大部分配置是一样的，打包入口同样也是`varlet-ui/es/umdIndex.js`，打包结果会在`varlet-ui/umd/`目录下生成一个`varlet.js`文件，`Varlet`和其他组件库稍微有点不一样的地方是它把样式也都打包进了`js`文件，省去了使用时需要再额外引入样式文件的麻烦，这个操作是`inlineCSS`插件做的，这个插件也是`Varlet`自己编写的，代码也很简单：

```ts
// varlet-cli/src/config/vite.config.js
function inlineCSS(fileName: string, dir: string): PluginOption {
  return {
    name: 'varlet-inline-css-vite-plugin',// 插件名称
    apply: 'build',// 设置插件只在构建时被调用
    closeBundle() {// rollup钩子，打包完成后调用的钩子
      const cssFile = resolve(dir, 'style.css')
      if (!pathExistsSync(cssFile)) {
        return
      }
      const jsFile = resolve(dir, fileName)
      const cssCode = readFileSync(cssFile, 'utf-8')
      const jsCode = readFileSync(jsFile, 'utf-8')
      const injectCode = `;(function(){var style=document.createElement('style');style.type='text/css';\
style.rel='stylesheet';style.appendChild(document.createTextNode(\`${cssCode.replace(/\\/g, '\\\\')}\`));\
var head=document.querySelector('head');head.appendChild(style)})();`
      // 将【动态将样式插入到页面】的代码插入到js代码内
      writeFileSync(jsFile, `${injectCode}${jsCode}`)
      // 将该样式文件复制到varlet-ui/lib/style.css文件里
      copyFileSync(cssFile, resolve(LIB_DIR, 'style.css'))
      // 删除样式文件
      removeSync(cssFile)
    },
  }
}
```

这个插件所做的事情就是在打包完成后，读取生成的`style.css`文件，然后拼接一段`js`代码，这段代码会把样式动态插入到页面，然后把这段`js`合并到生成的`js`文件中，这样就不用自己手动引入样式文件了。

同时，也会把样式文件复制一份到`lib`目录下，也就是`commonjs`产物的目录。

最后再回顾一下这个打包顺序：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91b1a16b6d644155a0442ca5604c094e~tplv-k3u1fbpfcp-zoom-1.image)


你会发现这个顺序是有原因的，`ems-bundle`的打包入口依赖`module`的产物，`umd`打包会给`commonjs`复制一份样式文件，所以打包`umd`需要在`commonjs`后面。



# 附录：babel配置详解

上文编译`script`、`ts`、`tsx`内容使用的是`babel`，提到了会使用本地的配置文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5bb856aab45475c867db74e74fba0dc~tplv-k3u1fbpfcp-zoom-1.image)


主要就是配置了一个`presets`，`presets`即`babel`的[预设](https://babeljs.io/docs/en/presets)，作用是方便使用一些共享配置，可以简单了解为包含了一组插件，`babel`的转换是通过各种插件进行的，所以使用预设可以免去自己配置插件，可以使用本地的预设，也可以使用发布在`npm `包里的预设，预设可以传递参数，比如上图，使用的是`@varlet/cli`包里附带的一个预设：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ee2684494ad47b7bd55bb0d26310e0f~tplv-k3u1fbpfcp-zoom-1.image)


预设其实就是一个`js`文件，导出一个函数，这个函数可以接受两个参数，`api`可以访问`babel`自身导出的所有模块，同时附带了一些配置文件指定的`api`，`options`为使用预设时传入的参数，这个函数需要返回一个对象，这个对象就是具体的配置。

```ts
// varlet-cli/src/config/babel.config.ts
module.exports = (api?: ConfigAPI, options: PresetOption = {}) => {
  if (api) {
    // 设置不要缓存该配置，每次都执行函数重新获取
    api.cache.never()
  }
  // 判断打包格式
  const isCommonJS = process.env.NODE_ENV === 'test' || process.env.BABEL_MODULE === 'commonjs'
  return {
    presets: [
      [
        require.resolve('@babel/preset-env'),
        {
          // 编译为commonjs模块类型时需要将ESM模块语法转换成commonjs模块语法，否则保留ESM模块语法
          modules: isCommonJS ? 'commonjs' : false,
          loose: options.loose,// 是否允许@babel/preset-env预设中配置的插件开启松散转换，https://cloud.tencent.com/developer/article/1418101
        },
      ],
      require.resolve('@babel/preset-typescript'),
      require('./babel.sfc.transform'),
    ],
    plugins: [
      [
        require.resolve('@vue/babel-plugin-jsx'),
        {
          enableObjectSlots: options.enableObjectSlots,
        },
      ],
    ],
  }
}
export default module.exports
```

又配置了三个预设，无限套娃，[@babel/preset-env](https://babeljs.io/docs/en/babel-preset-env)预设是一个智能预设，会根据你的目标环境自动判断需要转换哪些语法，`@babel/preset-typescript`用来支持`ts`语法，`babel.sfc.transform`是`varlet`自己编写的，用来转换`Vue`单文件。

还配置了一个[babel-plugin-jsx](https://github.com/vuejs/babel-plugin-jsx)插件，用来在`Vue`中支持`JSX`语法。

预设和插件的应用顺序是有规定的：

- 插件在预设之前运行
- 多个插件按从第一个到最后一个顺序运行
- 多个预设按从最后一个到第一个顺序运行

基于此我们可以大致窥探一下整个转换流程，首先运行插件`@vue/babel-plugin-jsx`转换`JSX`语法，然后运行预设`babel.sfc.transform`：

```ts
// varlet-cli/src/config/babel.sfc.transform.ts
import { readFileSync } from 'fs'
import { declare } from '@babel/helper-plugin-utils'

module.exports = declare(() => ({
  overrides: [
    {
      test: (file: string) => {
        if (/\.vue$/.test(file)) {
          const code = readFileSync(file, 'utf8')
          return code.includes('lang="ts"') || code.includes("lang='ts'")
        }

        return false
      },
      plugins: ['@babel/plugin-transform-typescript'],
    },
  ],
}))
```

通过`babel`的[overrides](https://babeljs.io/docs/en/options#overrides)选项来根据条件注入配置，当处理的是`Vue`单文件的内容，并且使用的是`ts`语法，那么就会注入一个插件[@babel/plugin-transform-typescript](https://babeljs.io/docs/en/babel-plugin-transform-typescript)，用于转换`ts`语法，非`Vue`单文件会忽略这个配置，进入下一个`preset`：[@babel/preset-typescript](https://babeljs.io/docs/en/babel-preset-typescript)，这个预设也包含了前面的`@babel/plugin-transform-typescript`插件，但是这个预设只会在`.ts`文件才会启用`ts`插件，所以前面才需要自行判断`Vue`单文件并手动配置`ts`插件，`ts`语法转换完毕后最后会进入`@babel/preset-env`，进行`js`语法的转换。





