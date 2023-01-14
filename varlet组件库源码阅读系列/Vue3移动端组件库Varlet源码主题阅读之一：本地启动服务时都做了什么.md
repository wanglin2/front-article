> 本文为`Varlet`组件库源码主题阅读系列第一篇

`Vue`开源的组件库有很多，基本各个大厂都会做一个，所以作为个人再重复造轮子其实意义也不是很大，但是笔者对于如何设计一个`Vue`组件库还是挺感兴趣的。

不同的组件库架构肯定有所不同，不过大体思路应该都差不多，笔者在众多组件库中挑选了[Varlet](https://github.com/varletjs/varlet) 来进行剖析，`Varlet`是一个基于 `Vue3` 开发的 `Material` 风格的移动端组件库，本系列的文章会全面解析这个项目，需要说明的是，不会具体的看某个组件是怎么实现的，而是了解组件库整体的设计，以及按需引入、主题定制、屏幕适配、组件打包、`VsCode`属性高亮等比较有意思的话题，话不多说，开始吧。

> Varlet 版本为：1.27.20

# 项目结构

先克隆`Varlet`的项目：

```bash
git clone https://github.com/varletjs/varlet.git
```

初始结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f0e6b21e120433c9ba3b67bb5501aa2~tplv-k3u1fbpfcp-zoom-1.image)


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f62b0fd4c3d4b388137eaedc5c80036~tplv-k3u1fbpfcp-zoom-1.image)


`packages`目录下存在很多单独的包，我们后面都会进行介绍。

# 本地开发

根据官方文档的描述，我们可以使用下面的命令进行本地开发：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11910f9f97f741f08669e6605ba6ebf2~tplv-k3u1fbpfcp-zoom-1.image)


`Varlet`使用的包管理器是[pnpm](https://pnpm.io/zh/)，`pnpm`是一个速度快、节省磁盘空间的软件包管理器，如果没安装的话需要先安装一下：

```bash
npm install -g pnpm
```

`bootstrap`命令如下：

```bash
pnpm install && node scripts/bootstrap.mjs
```

先安装依赖，然后执行`bootstrap.mjs`文件：

```js
// bootstrap.mjs
import { buildCli, buildIcons, buildShared, buildUI, runTask } from './build.mjs'

;(async () => {
  await runTask('shared', buildShared)
  await Promise.all([runTask('cli', buildCli), runTask('icons', buildIcons)])
  await runTask('ui', buildUI)
})()
```

运行了一波任务，挨个来看，先看`runTask`方法：

```js
// build.mjs
import ora from 'ora'

export async function runTask(taskName, task) {
  const s = ora().start(`Building ${taskName}`)
  try {
    await task()
    s.succeed(`Build ${taskName} completed!`)
  } catch (e) {
    s.fail(`Build ${taskName} failed!`)
    console.error(e.toString())
  }
}
```

[ora](https://github.com/sindresorhus/ora)是一个命令行的`loading`工具，可以在终端显示好看的`loading`效果，然后就是执行传入的任务函数。

- `shared`任务：

```js
// build.mjs
import execa from 'execa'
import { resolve } from 'path'

const CWD = process.cwd()// 获取nodejs进程的当前工作目录，也就是项目的根目录
const PKG_SHARED = resolve(CWD, './packages/varlet-shared')// varlet-shared包的绝对路径
export const buildShared = () => execa('pnpm', ['build'], { cwd: PKG_SHARED })
```

[execa](https://github.com/sindresorhus/execa)是`nodejs`的[child_process](https://nodejs.org/api/child_process.html)的改进版本，返回的是一个`Promise`，`pnpm`运行命令可以省略`run`，直接`pnpm build`即可，所以上述这个任务就是在`varlet-shared`包的目录下执行`build`命令：

```
tsc && tsc -p tsconfig.cjs.json
```

使用两个配置文件执行了两次`tsc`，也就是将`src`目录下的`ts`文件分别编译成了`es`模块和`commonjs`模块：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fb8f091651948b284f2796713b9298c~tplv-k3u1fbpfcp-zoom-1.image)


- `cli`任务：

```js
// build.mjs

const PKG_CLI = resolve(CWD, './packages/varlet-cli')
export const buildCli = () => execa('pnpm', ['build'], { cwd: PKG_CLI })
```

到`varlet-cli`目录下执行`build`命令：

```
tsc
```

同样也是编译`ts`，这个包的入口为`./lib/index.js`，未编译前`lib`目录下只有这一个文件，显然其他文件都是缺失的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03348bc9656e43c2b0f97d52bacec2e4~tplv-k3u1fbpfcp-zoom-1.image)


需要先编译才能使用这个包，编译后结果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79b89f98c6eb4916bcfe241754bace7a~tplv-k3u1fbpfcp-zoom-1.image)


- `icons`任务：

```js
// build.mjs

const PKG_ICONS = resolve(CWD, './packages/varlet-icons')
export const buildIcons = () => execa('pnpm', ['build'], { cwd: PKG_ICONS })
```

进入`varlet-icons`目录下运行`build`命令：

```
varlet-icons build
```

`varlet-icons`命令的执行文件为同目录下的`varlet-icons/lib/index.js`，详细逻辑我们后面再说，先看一下运行结果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc29e03d537047e48252bd6cfb28090b~tplv-k3u1fbpfcp-zoom-1.image)


其实就是将`svg`文件编译成字体图标。

- `ui`任务：

```js
// build.mjs

const PKG_UI = resolve(CWD, './packages/varlet-ui')
export const buildUI = (noUmd) => execa('pnpm', ['compile', noUmd ? '--noUmd' : ''], { cwd: PKG_UI })
```

进入`varlet-ui`目录下执行`compile`命令，和前面几个任务不同，这个任务会接收一个参数，顾名思义，是否不要生成`umd`，但是我搜索了一下并没有找到有传`true`的情况：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12df1d6425bf4b70af8908e1e34a36a2~tplv-k3u1fbpfcp-zoom-1.image)


`compile`命令如下：

```
varlet-cli compile
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be47896139524d6c91afbb296a92da71~tplv-k3u1fbpfcp-zoom-1.image)


该命令的作用是打包`varlet`的组件，具体实现逻辑后面再看，先看一下运行结果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf4af9dedd454ce3b9ad78cde325643f~tplv-k3u1fbpfcp-zoom-1.image)


主要是编译组件，有三种产物：`es`模块、`commonjs`模块、`umd`模块。

启动前的任务都运行完了，接下来就可以进入`varlet-ui`目录启动服务了，启动命令`pnpm dev`：

```
varlet-cli dev
```

这个命令做的事情比较多，我们后面再详解，大体上呢会把`varlet-cli`目录下的`site`目录复制到`varket-ui`目录下，并且动态生成两个路由文件：`mobile.routes.ts`、`pc.routes.ts`：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47a0e173b0ee47f69ce6b5ca1c49da80~tplv-k3u1fbpfcp-zoom-1.image)


访问启动的页面：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b397efd9ef04c838198198b3ff406b5~tplv-k3u1fbpfcp-zoom-1.image)


报错了，原因很简单，笔者是`Windows`电脑，路径的分隔符是反斜杠，所以生成的部分路径`\`没有转换成`/`，而`\`和`.varlet`组合起来会被认为是转义字符：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/142d38f6713c4788a3a57abc71fd89a7~tplv-k3u1fbpfcp-zoom-1.image)


查看源码发现虽然已经使用了[slash](https://github.com/sindresorhus/slash)来转换`Windows`平台下的路径问题，但是不知道为啥没有生效：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba5af929a9a6404cb25728871056f8c4~tplv-k3u1fbpfcp-zoom-1.image)



![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19f5fcde0fc440deac3206bfeb621cfd~tplv-k3u1fbpfcp-zoom-1.image)


没办法，只能手动修复一下，我们使用下面这种方法来转换：

```js
import path from 'path'
xxx.split(path.sep).join('/')
```

修改完以后文档页面示例显示出来了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6fcca033111a4cb6ae60a354d7087028~tplv-k3u1fbpfcp-zoom-1.image)


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc3f657cc0aa45229d67f07ed8719d4d~tplv-k3u1fbpfcp-zoom-1.image)


右边手机模拟器里的组件导入的是`varlet-ui/src/xxx`目录下的，也就是开发目录下的组件，所以我们直接修改就可以在页面上查看效果了。

项目的本地启动部分就到这里，我们下篇再见~
