> 本文为Varlet组件库源码主题阅读系列第四篇，读完本篇，可以了解到如何使用`Vite`的`Api`接口来启动服务、如何动态生成多语言的页面路由。

`Varlet`的文档网站其实就是一个`Vue`项目，整体分成两个单独的页面：文档页面及手机预览页面。

网站源代码文件默认是放在`varlet-cli`目录下，也就是脚手架的包里：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c519735855743d09c8a25542108d49e~tplv-k3u1fbpfcp-zoom-1.image)


执行脚手架提供的`dev`命令时会把这个目录复制到`varlet-ui/.varlet`目录下，并且动态生成两个页面的路由配置文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74e85350c5104f42bcd7abc9ffbd023c~tplv-k3u1fbpfcp-zoom-1.image)


然后使用`Vite`启动服务。

# 启动命令

先来看一下`varlet-cli`提供的`dev`命令都做了些什么。

```js
// varlet-cli/src/index.ts
import { Command } from 'commander'
const program = new Command()

program
  .command('dev')
  .option('-f --force', 'Force dep pre-optimization regardless of whether deps have changed')
  .description('Run varlet development environment')
  .action(dev)
```

可以看到这个命令是用来运行`varlet`的开发环境的，还提供了一个参数，用来强制开启`Vite`的[依赖预构建](https://www.vitejs.net/guide/dep-pre-bundling.html)功能，处理函数是`dev`：

```ts
// varlet-cli/src/commands/dev.ts
export async function dev(cmd: { force?: boolean }) {
  process.env.NODE_ENV = 'development'
  // SRC_DIR：varlet-ui/src，即组件的源码目录
  ensureDirSync(SRC_DIR)
  await startServer(cmd.force)
}
```

设置了环境变量，确保组件源目录是否存在，最后调用了`startServer`方法：

```ts
// varlet-cli/src/commands/dev.ts
let server: ViteDevServer
let watcher: FSWatcher
async function startServer(force: boolean | undefined) {
    // 如果server实例已经存在了，那么代表是重启
    const isRestart = Boolean(server)
    // 先关闭之前已经存在的实例
    server && (await server.close())
    watcher && (await watcher.close())
    // 构建站点入口
    await buildSiteEntry()
}
```

# 构建站点项目

复制站点文件的操作就在`buildSiteEntry`方法里：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildSiteEntry() {
  getVarletConfig(true)
  await Promise.all([buildMobileSiteRoutes(), buildPcSiteRoutes(), buildSiteSource()])
}
```

主要执行了四个方法，先看`getVarletConfig`：

```ts
// varlet-cli/src/config/varlet.config.ts
export function getVarletConfig(emit = false): Record<string, any> {
  let config: any = {}
  // VARLET_CONFIG：varlet-ui/varlet.config.js，即varlet-ui组件库目录下的配置文件
  if (pathExistsSync(VARLET_CONFIG)) {
    // require方法导入后会进行缓存，下次同样的导入会直接使用缓存，所以当重新启动服务时需要先删除缓存
    delete require.cache[require.resolve(VARLET_CONFIG)]
    config = require(VARLET_CONFIG)
  }
  // 默认配置，varlet-cli/varlet.default.config.js
  delete require.cache[require.resolve('../../varlet.default.config.js')]
  const defaultConfig = require('../../varlet.default.config.js')
  // 合并配置
  const mergedConfig = merge(defaultConfig, config)

  if (emit) {
    const source = JSON.stringify(mergedConfig, null, 2)
    // SITE_CONFIG：resolve(CWD, '.varlet/site.config.json')
    // outputFileSyncOnChange方法会检查内容是否有变化，没有变化不会重新写入文件
    outputFileSyncOnChange(SITE_CONFIG, source)
  }
  return mergedConfig
}
```

这个方法主要是合并组件库目录`varlet-ui`下的配置文件和默认的配置文件，然后将合并后的配置写入到站点的目标目录`varlet-ui/.varlet/`下。

合并完配置后执行了三个`build`方法：

## 生成手机页面路由

1.`buildMobileSiteRoutes()`方法：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildMobileSiteRoutes() {
  const examples: string[] = await findExamples()
  // 拼接路由
  const routes = examples.map(
    (example) => `
  {
    path: '${getExampleRoutePath(example)}',
    // @ts-ignore
    component: () => import('${example}')
  }`
  )

  const source = `export default [\
    ${routes.join(',')}
]`
  // SITE_MOBILE_ROUTES：resolve(CWD, '.varlet/mobile.routes.ts')，站点的手机预览页面路由文件
  await outputFileSyncOnChange(SITE_MOBILE_ROUTES, source)
}
```

这个方法主要是构建手机预览页面的路由文件，路由其实就是路径到组件的映射，所以先获取了路由组件列表，然后按格式拼接路由的内容，最后写入文件。

`findExamples()`：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export function findExamples(): Promise<string[]> {
  // SRC_DIR：varlet-ui/scr目录，即组件库的源码目录
  // EXAMPLE_DIR_NAME：example，即每个组件的示例目录
  // DIR_INDEX：index.vue
  return glob(`${SRC_DIR}/**/${EXAMPLE_DIR_NAME}/${DIR_INDEX}`)
}
```

从组件库源码目录里获取每个组件的示例组件，每个组件都是一个单独的目录，目录下存在一个`example`示例文件目录，该目录下的`index.vue`即示例组件，比如按钮组件`Button`的目录及示例组件如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57ccc942f33842198d9f31ecefe6f4ea~tplv-k3u1fbpfcp-zoom-1.image)


这个方法获取到的是绝对路径，并不能用作路由的`path`，所以需要进行一下处理：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
const EXAMPLE_COMPONENT_NAME_RE = /\/([-\w]+)\/example\/index.vue/
export function getExampleRoutePath(examplePath: string): string {
  return '/' + examplePath.match(EXAMPLE_COMPONENT_NAME_RE)?.[1]
}
```

提取出`example`前面的一段，即组件的目录名称，也就是组件的名称，最后生成的路由数据如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8522cf0f467f48ae94e71b802c03b193~tplv-k3u1fbpfcp-zoom-1.image)


## 生成pc页面路由

2.`buildPcSiteRoutes()`方法：

`pc`页面的路由稍微会复杂一点：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildPcSiteRoutes() {
  const [componentDocs, rootDocs, rootLocales] = await Promise.all([
    findComponentDocs(),
    findRootDocs(),
    findRootLocales(),
  ])
}
```

获取了三类文件，第一种：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export function findComponentDocs(): Promise<string[]> {
  // SRC_DIR：varlet-ui/scr目录，即组件库的源码目录
  // DOCS_DIR_NAME：docs
  return glob(`${SRC_DIR}/**/${DOCS_DIR_NAME}/*.md`)
}
```

获取组件目录`varlet-ui/src/**/docs/*.md`文件，也就是获取每个组件的文档文件，比如`Button`组件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e3b335f3a6e4bbb8ec9d49588c84ef4~tplv-k3u1fbpfcp-zoom-1.image)


文档是`markdown`格式编写的。

第二种：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export function findRootDocs(): Promise<string[]> {
  // ROOT_DOCS_DIR：varlet-ui/docs
  return glob(`${ROOT_DOCS_DIR}/*.md`)
}
```

获取除组件文档外的其他文档，比如基本介绍、快速开始之类的。

第三种：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function findRootLocales(): Promise<string[]> {
  // 默认的语言
  const defaultLanguage = get(getVarletConfig(), 'defaultLanguage')
  // SITE：varlet-cli/site/
  // LOCALE_DIR_NAME：locale
  const baseLocales = await glob(`${SITE}/pc/pages/**/${LOCALE_DIR_NAME}/*.ts`)
}
```

获取默认的语言类型，默认是`zh-CN`，然后获取站点`pc`页面的`locale`文件，继续：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
const ROOT_LOCALE_RE = /\/pages\/([-\w]+)\/locale\/([-\w]+)\.ts/
export async function findRootLocales(): Promise<string[]> {
    // ...
    const filterMap = new Map()
    baseLocales.forEach((locale) => {
        // 解析出页面path及文件的语言类型
        const [, routePath, language] = locale.match(ROOT_LOCALE_RE) ?? []
        // SITE_PC_DIR：varlet-ui/.varlet/site/pc
        filterMap.set(routePath + language, slash(`${SITE_PC_DIR}/pages/${routePath}/locale/${language}.ts`))
    })
	return Promise.resolve(Array.from(filterMap.values()))
}
```

返回获取到`pc`站点的`locale`文件路径，也就是这些文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9ec54c72bde4facb7bb28eb50aebe37~tplv-k3u1fbpfcp-zoom-1.image)


目前只有一个`Index`页面，也就是站点的首页：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99aa92cff82741bdb2a750789644e8a1~tplv-k3u1fbpfcp-zoom-1.image)


回到`buildPcSiteRoutes()`方法，文件路径都获取完了，接下来肯定就是遍历生成路由配置了：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildPcSiteRoutes() {
  // ...
  // 生成站点页面路由
  const rootPagesRoutes = rootLocales.map(
    (rootLocale) => `
  {
    path: '${getRootRoutePath(rootLocale)}',
    // @ts-ignore
    component: () => import('${getRootFilePath(rootLocale)}')
  }\
`
  )
}
```

有多少种翻译，同一个组件就会生成多少种路由，对于站点首页来说，目前存在`en-US.ts`、`zh-CN`两种翻译文件，那么会生成下面两个路由：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be34711e646549dd9a052d4af667678b~tplv-k3u1fbpfcp-zoom-1.image)


继续：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildPcSiteRoutes() {
  // ...
  // 生成每个组件的文档路由
  const componentDocsRoutes = componentDocs.map(
    (componentDoc) => `
      {
        path: '${getComponentDocRoutePath(componentDoc)}',
        // @ts-ignore
        component: () => import('${componentDoc}')
      }`
  )
  // 生成其他文档路由
  const rootDocsRoutes = rootDocs.map(
    (rootDoc) => `
      {
        path: '${getRootDocRoutePath(rootDoc)}',
        // @ts-ignore
        component: () => import('${rootDoc}')
      }`
  )
}
```

接下来拼接了组件文档和其他文档的路由，同样也是存在几种翻译，就会生成几个路由：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef9c7d03a0ab430aadf6d3987c5a31bb~tplv-k3u1fbpfcp-zoom-1.image)


继续：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildPcSiteRoutes() {
  // ...
  const layoutRoutes = `{
    path: '/layout',
    // @ts-ignore
    component:()=> import('${slash(SITE_PC_DIR)}/Layout.vue'),
    children: [
      ${[...componentDocsRoutes, rootDocsRoutes].join(',')},
    ]
  }`
}
```

这个路由是干嘛的呢，其实就是真正的文档页面了：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7d1ea8047574f45a566a32730b00f5e~tplv-k3u1fbpfcp-zoom-1.image)


组件文档路由和其他文档路由都是它的子路由，`Layout.vue`组件提供了组件详情页面的基本骨架，包括页面顶部栏、左边的菜单栏，中间部分就是子路由的出口，即具体的文档，右侧通过`iframe`引入了手机预览页面。

最后导出路由配置及写入到文件即可：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildPcSiteRoutes() {
  // ...
  const source = `export default [\
  ${rootPagesRoutes.join(',')},
  ${layoutRoutes}
]`
  // SITE_PC_ROUTES：varlet-ui/.varlet/pc.routes.ts
  outputFileSyncOnChange(SITE_PC_ROUTES, source)
}
```

## 复制站点文件

3.`buildSiteSource()`方法：

```ts
// varlet-cli/src/compiler/compileSiteEntry.ts
export async function buildSiteSource() {
  return copy(SITE, SITE_DIR)
}
```

这个方法很简单，就是将站点的项目文件由`varlet-cli/site`目录复制到`varlet-ui/.varlet/site`目录下。

总结一下上述操作，就是将站点的源代码文件由`cli`包复制到`ui`包，然后动态生成站点项目的路由文件。整个站点分为两个页面`pc`、`mobile`，`pc`页面主要是提供文档展示及嵌入`mobile`页面，`mobile`页面用来展示各个组件的`demo`。



# 启动服务

项目准备就绪，接下来就是启动服务了，回到`startServer`方法：

```ts
// varlet-ui/src/commands/dev.ts
async function startServer(force: boolean | undefined) {
  await buildSiteEntry()
  // 获取合并后的配置
  const varletConfig = getVarletConfig()
  // 获取Vite的启动配置，部分配置来自于varletConfig
  const devConfig = getDevConfig(varletConfig)
  // 将是否强制进行依赖预构建配置合并到Vite配置
  const inlineConfig = merge(devConfig, force ? { server: { force: true } } : {})
}
```

生成`Vite`的启动配置，然后就可以启动服务了：

```ts
// varlet-ui/src/commands/dev.ts
async function startServer(force: boolean | undefined) {
    // ...
    // 启动Vite服务
    server = await createServer(inlineConfig)
    await server.listen()
    server.printUrls()
	// VARLET_CONFIG：varlet-ui/varlet.config.js
    // 监听用户配置文件，修改了就重新启动服务
    if (pathExistsSync(VARLET_CONFIG)) {
        watcher = chokidar.watch(VARLET_CONFIG)
        watcher.on('change', () => startServer(force))
    }
}
```

使用了`Vite`的[JavaScript API](https://www.vitejs.net/guide/api-javascript.html)来启动服务，并且当配置文件发送变化会重启服务。



# Vite配置

接下来详细看一下上一步启动服务时的`Vite`配置：

```ts
// varlet-cli/src/config/vite.config.ts
export const VITE_RESOLVE_EXTENSIONS = ['.vue', '.tsx', '.ts', '.jsx', '.js', '.less', '.css']

export function getDevConfig(varletConfig: Record<string, any>): InlineConfig {
  // 默认语言
  const defaultLanguage = get(varletConfig, 'defaultLanguage')
  // 端口
  const host = get(varletConfig, 'host')

  return {
    root: SITE_DIR,// 项目根目录：varlet-ui/.varlet/site
    resolve: {
      extensions: VITE_RESOLVE_EXTENSIONS,// 导入时想要省略的扩展名列表
      alias: {// 导入路径别名
        '@config': SITE_CONFIG,
        '@pc-routes': SITE_PC_ROUTES,
        '@mobile-routes': SITE_MOBILE_ROUTES,
      },
    },
    server: {// 设置要监听的端口号和ip地址
      port: get(varletConfig, 'port'),
      host: host === 'localhost' ? '0.0.0.0' : host,
    },
    publicDir: SITE_PUBLIC_PATH,// 作为静态资源服务的文件夹：varlet-ui/public
    // ...
  }
}
```

设置了一些基本配置，你可能会有个小疑问，站点项目明明是个多页面项目，但是上面似乎并没有配置任何多页面相关的内容，其实在`Vue Cli`项目中是需要修改入口配置的，但是在`Vite`项目中不需要，这可能就是开发环境不需要打包的一个好处吧，不过虽然开发环境不需要配置，但是最后打包的时候是需要的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3233062b2ef4f0d8812f977db9bc54b~tplv-k3u1fbpfcp-zoom-1.image)


接下来还配置了一系列的插件：

```ts
import vue from '@vitejs/plugin-vue'
import md from '@varlet/markdown-vite-plugin'
import jsx from '@vitejs/plugin-vue-jsx'
import { injectHtml } from 'vite-plugin-html'

export function getDevConfig(varletConfig: Record<string, any>): InlineConfig {
    // ...
    return {
        // ...
        plugins: [
            // 提供 Vue 3 单文件组件支持
            vue({
                include: [/\.vue$/, /\.md$/],
            }),
            md({ style: get(varletConfig, 'highlight.style') }),
            // 提供 Vue 3 JSX 支持
            jsx(),
            // 给html页面注入数据
            injectHtml({
                data: {
                    pcTitle: get(varletConfig, `pc.title['${defaultLanguage}']`),
                    mobileTitle: get(varletConfig, `mobile.title['${defaultLanguage}']`),
                    logo: get(varletConfig, `logo`),
                    baidu: get(varletConfig, `analysis.baidu`, ''),
                },
            }),
        ],
    }
}
```

一共使用了四个插件，其中的`md`插件是`Varlet`自己编写的，顾名思义，就是用来处理`md`文件的，具体逻辑我们下一篇再看。



# 打包

最后就是站点项目的打包了，使用的是`varlet-cli`提供的`build`命令：

```ts
// varlet-cli/src/index.ts
program.command('build').description('Build varlet site for production').action(build)
```

处理函数为`build`：

```ts
// varlet-cli/src/commands/build.ts
export async function build() {
  process.env.NODE_ENV = 'production'

  ensureDirSync(SRC_DIR)
  await buildSiteEntry()
  const varletConfig = getVarletConfig()
  // 获取Vite的打包配置
  const buildConfig = getBuildConfig(varletConfig)

  await buildVite(buildConfig)
}
```

逻辑很简单，先设置环境变量为生产环境，然后同样执行了`buildSiteEntry`方法，最后获取`Vite`的打包配置进行打包即可：

```ts
// varlet-cli/src/config/vite.config.ts
export function getBuildConfig(varletConfig: Record<string, any>): InlineConfig {
  const devConfig = getDevConfig(varletConfig)

  return {
    ...devConfig,
    base: './',// 公共基础路径
    build: {
      outDir: SITE_OUTPUT_PATH,// varlet-ui/site
      brotliSize: false,// 禁用 brotli 压缩大小报告
      emptyOutDir: true,// 输出目录不在root目录下，所以需要手动开启该配置来清空输出目录
      cssTarget: 'chrome61',// https://www.vitejs.net/config/#build-csstarget
      rollupOptions: {
        input: {// 多页面入口配置
          main: resolve(SITE_DIR, 'index.html'),
          mobile: resolve(SITE_DIR, 'mobile.html'),
        },
      },
    },
  }
}
```

在启动服务的配置基础上增加了几个打包相关的配置。

到这里文档站点的初始化、启动、构建办法就介绍完了，我们下一篇再见~
