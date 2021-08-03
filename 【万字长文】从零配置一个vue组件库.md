# 简介

本文会从零开始配置一个`monorepo`类型的组件库，包括规范化配置、打包配置、组件库文档配置及开发一些提升效率的脚本等，`monorepo `不熟悉的话这里一句话介绍一下，就是在一个`git`仓库里包含多个独立发布的模块/包。

ps.本文涉及到的工具配置其实在平时开发中一般都不需要自己配置，我们使用的各种脚手架都帮我们搞定了，但是我们至少得大概知道都是什么意思以及为什么，说来惭愧，笔者作为一个三四年工龄的前端老人，基本没有自己动手配过，甚至没有去了解过，所以以下大部分工具都是笔者第一次使用，除了介绍如何配置也会讲到遇到的一些坑及解决方法，另外也会尽量去搞清楚每一个参数的意思及原理，有兴趣的请继续阅读吧~



# 使用lerna管理项目

首先每个组件都是一个独立的`npm`包，但是某个组件可能又依赖了另一个组件，这样如果这个组件有bug修改完后发布了新版本，需要手动到依赖它的组件里挨个进行升级再进行发布，这是一个繁琐且效率不高的过程，所以可以使用[leran](https://lerna.js.org/)工具来进行管理，`lerna`是一个专门用于管理带有多个包的`JavaScript`项目的工具，可以帮助进行`npm`发布及`git`上传。

首先全局安装`lerna`：

```bash
npm i -g lerna
```

然后进入仓库目录执行：

```bash
lerna init
```

这个命令用来创建一个新的`lerna`仓库或者升级一个现有仓库的`lerna`版本，`lerna`有两种使用模式：

1.固定模式，默认固定模式下所有包的主版本号和次版本都会使用`lerna.json`配置里的`version`字段定义的版本号，如果某一次只修改了其中一个或几个包，但修改了配置文件里的主版本号或次版本号，那么发布时所有的包都会统一升级到该版本并进行发布，单个的包如果想要发布只能修改修订版本号进行发布；

2.独立模式就是每个包使用独立的版本号。

自动生成的目录如下：

![image-20210324113741016.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44ebdf8def2644ba9fdad8ee49928677~tplv-k3u1fbpfcp-watermark.image)

可以看到没有`.gitignore`文件，所以手动创建一下，目前只需要忽略`node_modules`目录。

我们所有的包都会放在`packages`文件夹下，添加新包可以使用`lerna create xxx`命令（后面会通过脚本来生成），组件库推荐给包名增加一个统一的作用域`scope`，可以避免命名冲突，比如常见的`@vue/xxx`、`@babel/xxx`等，`npm`从`2.0`版本开始支持发布带作用域的包，默认的作用域是你的`npm`用户名，比如：`@username/package-name`，也可以使用`npm config set @scope-name:registry http://reg.example.com `来给你使用的`npm`仓库关联一个作用域。

给包添加依赖可以使用`lerna add module-1 --scope=module-2`命令，表示将`module-1`安装到`module-2`的依赖里，`learn`检查到如果依赖的包是本项目中的会直接链接过去：

![image-20210324153524048.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f19782a03f24b10a456cd884cd83c01~tplv-k3u1fbpfcp-watermark.image)

可以看到有个链接标志，`lerna add`默认也会执行`lerna bootstrap`的操作，即给所有的包安装依赖项。

当修改完成后需要发布时可以使用`lerna publish`命令，该命令会完成模块的发布及`git`上传工作，有个需要注意的点是带作用域的包使用`npm`发布时需要添加`--access public`参数，但是`lerna publish`不支持该参数，一个解决方法是在所有包的`package.json`文件里添加：

```json
{
    // ...
    "publishConfig": {
        "access": "publish" 
    }
}
```

![image-20210401191940050.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbd66b56a1394586bbf193158dca39ed~tplv-k3u1fbpfcp-watermark.image)

# 规范化配置

## eslint

`eslint`是一个配置化的`JavaScript`代码检查工具，通过该工具可以约束代码风格，以及检测一些潜在错误，做到在不同的开发者下能有一个统一风格的代码，常见的比如是否允许使用`==`、语句结尾是否去掉`;`等等，`eslint`的规则非常多，可以在这里查看[https://eslint.bootcss.com/docs/rules/](https://eslint.bootcss.com/docs/rules/) 。

`eslint`的所有规则都可单独配置是否开启，并且默认都是禁用的，所以如果要自己来挨个配置是比较麻烦的，但是它有个继承的配置，可以很方便的使用别人的配置，先来安装一下：

```bash
npm i eslint --save-dev
```

然后在`package.json`文件里加一个命令：

```json
{
    "scripts": {
		"lint:init": "eslint --init"
    }
}
```

之后在命令行输入`npm run lint:init `来创建一个`eslint`配置文件，根据你的情况回答完一些问题后就会生成一个默认配置，我生成的内容如下：

![image-20210325104955305.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2adb6dbec684dc0a2ed2e9f1a475abb~tplv-k3u1fbpfcp-watermark.image)

简单看一下各个字段的意思：

- `env`字段用来指定你代码所要运行的环境，比如是在浏览器环境下，还是node环境下，不同的环境下所对应的全局变量不一样，因为后续还要写`node`脚本，所以把`node:true`也加上；

- `parserOptions`表示所支持的语言选项，比如`JavaScript`的版本、是否启用`JSX`等，设置正确的语言选项可以让`eslint`确定什么是解析错误；

- `plugins`顾名思义是插件列表，比如你使用的是`react`，那么需要使用`react`的插件来支持`react`的语法，因为我用的是`vue`，所以使用了`vue`的插件，可以用来检测单文件的语法问题，插件的命名规则为`eslint-plugin-xxxx`，配置时前缀可以省略；

- `rules`就是规则配置列表，可以单独配置某个规则启用与否；

- `extends`就是上文所说的继承，这里使用了官方推荐的配置以及`vue`插件顺带提供的配置，配置命名一般为`eslint-config-xxx`，使用时前缀也可以省略，并且插件也可以顺带提供配置功能，引入规则一般为`plugin:plugin-name/xxx`，此外也可以选择使用其他一些比较出名的配置如`eslint-config-airbnb`；

和`.gitignore`一样，`eslint`也可以创建一个忽略配置文件`.eslintignore`，每一行都是一个`glob`模式来表示哪些路径要忽略：

```
node_modules
docs
dist
assets
```

接下来再去`package.json`文件里加上运行检查的命令：

```json
"scripts": {
    "lint": "eslint ./ --fix"
}
```

意思是检查当前目录下的所有文件，`--fix`表示允许`eslint`进行修复，但是能修自动复的问题很少，执行`npm run lint`，结果如下：

![image-20210325142455760.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/298c09acd89340d9837cdec303935713~tplv-k3u1fbpfcp-watermark.image)


## husky

目前只能手动去运行`eslint`检查，就算能约束自己每次提交代码前检查一下，也不一定能约束到其他人，没有强制的规范和没有规范没啥区别，所以最好在`git`提交前采取强制措施，这可以使用[Husky](https://github.com/typicode/husky/tree/v4)，这个工具可以方便的让我们在执行某个`git`命令前先执行特定的命令，我们的需求是在`git commit`之前进行`eslint`检查，这需要使用`pre-commit`钩子，`git`还有很多其他的钩子：[https://git-scm.com/docs/githooks](https://git-scm.com/docs/githooks)。

国际惯例，先安装：

```bash
npm i husky@4 --save-dev
```

然后在`package.json`文件里添加：

```json
{
    "husky": {
        "hooks": {
            "pre-commit": "npm run lint"
        }
    }
}
```

接着我尝试`git commit`，但是，没有效果。。。检查了`node`、`npm`和`git`的版本，均没有问题，然后我打开`git`的隐藏文件夹`.git/hooks`：

![image-20210325153241918.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/adec7fbf27784f72a49f1aca1c4a9262~tplv-k3u1fbpfcp-watermark.image)

发现目前的这些钩子文件后面还是带着`sample`后缀，如果想要某个钩子生效，这个后缀要去掉才行，但是这种操作显然不应该让我手动来干，那么只能重装`husky`试试，经过简单的测试，我发现`v5.x`版本也是不行的，但是`v3.0.0`及`v1.1.1`两个版本是生效的（笔者系统是windows10，可能和笔者电脑环境有关）：

![image-20210325154358368.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0c3b86ee95740199ed9d350c8d8769e~tplv-k3u1fbpfcp-watermark.image)

![image-20210325154606969.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0737c377772041459b8de5d904665596~tplv-k3u1fbpfcp-watermark.image)

这样如果检查到有错误就会终止`commit`操作，不过目前一般还会使用另外一个包`lint-staged`，这个包顾名思义，只检查`staged`状态下的文件，其他本次提交没有变动的文件就不用检查了，这是合理的也能提高检查速度，先安装：`npm i lint-staged --save-dev`，然后去`package.json`里配置一下：

```json
{
    "husky": {
        "hooks": {
            "pre-commit": "lint-staged"
        }
    },
    "lint-staged": {
        "*.{js,vue}": [
            "eslint --fix"
        ]
    }
}
```

首先`git`钩子执行的命令改成`lint-staged`，`lint-staged`字段的值是个对象，对象的`key`也是`glob`匹配模式，`value`可以是字符串或字符串数组，每个字符串代表一个可执行的命令，如果`lint-staged`发现当前存在`staged`状态的文件会进行匹配，如果某个规则匹配到了文件那么就会执行这个规则对应的命令，在执行命令的时候会把匹配到的文件作为参数列表传给此命令，比如：`exlint --fix xxx.js xxx.vue ...`，所以上面配置的意思就是如果在已暂存的文件里匹配到了`js`或`vue`文件就执行`eslint --fix xxx.js ... `，为啥命令不直接写`npm run lint`呢，因为`lint`命令里我们配置了`./`路径，那么仍将会检查所有文件。

执行效果如下，在上文的截图中可以看到一共有14个错误，但是本次我只修改了一个文件，所以只检查了这一个文件：

![image-20210325170358158.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3775ea3d2354d419d11cb0de1c7751b~tplv-k3u1fbpfcp-watermark.image)


## stylelint

[stylelint](https://stylelint.io/)和`eslint`十分相似，只不过是用来检查`css`语法的，除了`css`文件，同时也支持`scss`、`less`等`css`预处理语言，`stylelint`可能没`eslint`那么流行，不过本着学习的目的，咱们也尝试一下，毕竟组件库肯定少不了写样式，依旧先安装：`npm i stylelint stylelint-config-standard --save-dev`，`stylelint-config-standard`是推荐的配置文件，和`eslint-config-xxx`一样，也可以拿来继承，不喜欢这个规则也可以换其他的，接着创建一个配置文件`.stylelintrc`，输入以下内容：

```json
{
  "extends": "stylelint-config-standard"
}
```

创建一个忽略配置文件`.stylelintignore`，输入：

```
node_modules
```

最后在`package.json`中添加一行命令：

```json
{
	"scripts": {
        "style:lint": "stylelint packages/**/*.{css,less} --fix"
    }
}
```

检查`packages`目录下所有以`css`或`less`结尾的文件，并且可以的话自动进行修复，执行命令效果如下：

![image-20210325195041077.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/742e1b5e19f141f1b3fc107aa256b814~tplv-k3u1fbpfcp-watermark.image)

最后的最后和`eslint`一样，在`git commit`之前也加上自动进行检查，`package.json`文件修改如下：

```json
{
    "lint-staged": {
        "*.{css,less}": [
            "stylelint --fix"
        ]
    }
}
```



## commitlint

`commit`的内容对于了解一次提交做了什么来说是很重要的，`git commit`内容的标准格式其实是包含三部分的：`Header`、`Body`、`Footer`，其中`Header`部分是必填的，但是说实话对于我来说`Header`部分都懒得认真写，更不用说其他几部分了，所以靠自觉不行还是上工具吧，让我们在`git`的`commit-msg`钩子上加上对`commit`内容的检查功能，不符合规则就打回重写，安装一下校验工具[commitlint](https://commitlint.js.org/)：

```bash
npm i --save-dev @commitlint/config-conventional @commitlint/cli
```

同样也是一个工具，一个配置，通过继承的方式来使用，严重怀疑这些工具的开发者都是同一批人，接下来创建一个配置文件`commitlint.config.js`，输入如下内容：

```js
module.exports = {
    extends: ['@commitlint/config-conventional']
}
```

当然你也可以再单独配置你需要的规则，然后去`package.json`的`husky`部分配置钩子：

```json
{
    "husky": {
        "hooks": {
            "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
        }
    }
}
```

`commitlint`命令需要有输入参数，也就是我们输入的`commit message`，`-E`参数的意思如下：

![image-20210326144703482.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae01e5182c614db0ae96016521d5be4a~tplv-k3u1fbpfcp-watermark.image)

大意就是从环境变量里给定的文件里获取输入内容，这个环境变量看名字就知道是`husky`提供的，具体它是啥呢，咱们来简单看一下，首先打开`.git/hooks/commit-msg`文件，这个就是`commit-msg`钩子执行的`bash`脚本：

![image-20210326151212708.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a2aa54bc3c64d6fa0944a757132a82b~tplv-k3u1fbpfcp-watermark.image)

可以看到最后执行了`run.js`，参数分别为`hookName`及`gitParams`，`baseName "$0"`代表当前执行的脚本名称，也就是文件名`commit-msg`，`"$*"`代表所有的参数，`run.js`里又辗转反侧的最后调用了一个`run`方法：

```js
function run([, scriptPath, hookName = '', HUSKY_GIT_PARAMS], getStdinFn = get_stdin_1.default) {
    console.log('拦截', scriptPath, hookName, HUSKY_GIT_PARAMS)
    
    // ...
}
```

我们打印看一下参数都是啥：

![image-20210326152521041.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d1bbd0c265a46f5a694b4563322952c~tplv-k3u1fbpfcp-watermark.image)

可以看到`HUSKY_GIT_PARAMS`就是一个文件路径，这个文件里保存着我们这次输入的`commit message`的内容，接着`husky`会把它设置到环境变量里：

```js
const env = {};
if (HUSKY_GIT_PARAMS) {
    env.HUSKY_GIT_PARAMS = HUSKY_GIT_PARAMS;
}
if (['pre-push', 'pre-receive', 'post-receive', 'post-rewrite'].includes(hookName)) {
    // Wait for stdin
    env.HUSKY_GIT_STDIN = yield getStdinFn();
}
if (command) {
    console.log(`husky > ${hookName} (node ${process.version})`);
    execa_1.default.shellSync(command, { cwd, env, stdio: 'inherit' });
    return 0;
}
```

现在再看`commitlint -E HUSKY_GIT_PARAMS`就很容易理解了，`commitlint`会去读取`.git/COMMIT_EDITMSG`文件内容来检查我们输入的`commit message`是否符合规范。

![image-20210326154237132.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d29dc230c1bd4a81811950f41c0f2e8c~tplv-k3u1fbpfcp-watermark.image)

可以看到我们只输入了一个`1`的话就报错了。


## commitizen

上面提到一个标准的`commit message`是包含三部分的，详细看就是这样的：

```
<type>(<scope>): <subject>
空行
<body>
空行
<footer>
```

当你输入`git commit`时，就会出现一个命令行编辑器让你来输入，但是这个编辑器很不好用，没用过的话怎么保存都是个问题，所以可以使用[commitizen](https://github.com/commitizen/cz-cli)来进行交互式的输入，依次执行下列命令：

```bash
npm install commitizen -g

commitizen init cz-conventional-changelog --save-dev --save-exact
```

执行完后应该会自动在你的`package.json`文件里加上下列配置：

```json
{
    "config": {
        "commitizen": {
            "path": "./node_modules/cz-conventional-changelog"
        }
    }
}
```

然后你就可以使用`git cz`命令来代替`git commit`命令了，它会给你一些选项，以及询问你一些问题，如实输入即可：

![image-20210326162358483.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e7d5f2d1ac54cb7adb6c94cea80f6b4~tplv-k3u1fbpfcp-watermark.image)

但是这样`git commit`命令仍然是可用的，文档上说可以进行如下配置来将`git commit`转换为`git cz`：

```json
{
    "husky": {
        "hooks": {
            "prepare-commit-msg": "exec < /dev/tty && git cz --hook || true",
        }
    }
}
```

但是我尝试了不行，报`系统找不到指定的路径。`的错误，没找到原因和解决方法，如果你知道如何解决的话评论区见吧~强制不了，那只能加一句卑微的提示了：

```json
{
    "husky": {
        "hooks": {
            "prepare-commit-msg": "echo ----------------please use [git cz] command instead of [git commit]----------------"
        }
    }
}
```

![image-20210326170629882.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e23e730fb8847f1bad5f6f238e0481e~tplv-k3u1fbpfcp-watermark.image)


规范化的暂且就配置这么多，其他的比如代码美化可以使用[prettier](https://github.com/prettier/prettier)、生成提交日志的可以使用[conventional-changelog](https://github.com/conventional-changelog/conventional-changelog)或[standard-version](https://github.com/conventional-changelog/standard-version)，有需要的可以自行尝试。



# 打包配置

目前每个组件的结构都是类似下面这样的：

![image-20210329135733055.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f59b8c6f559a4e94bd34414be7caf810~tplv-k3u1fbpfcp-watermark.image)

`index.js`返回一个带`install`方法的对象，作为`vue`的插件，使用这个组件的方式如下：

```js
import ModuleX from 'module-x'
Vue.use(ModuleX)
```

组件库其实直接这么发布就可以了，如果`js`文件里使用了最新的语法，那么需要在使用该组件的项目里的`vue.config.js`里添加一下如下配置：

```js
{
    transpileDependencies: [
        'module-x'
    ]
}
```

因为默认情况下 `babel-loader` 会忽略所有 `node_modules` 中的文件，添加这个配置可以让Babel 显式转译这个依赖。

不过如果你硬想要打包后再进行发布也是可以的，我们增加一下打包的配置。

先安装一下相关的工具：

```bash
npm i webpack less less-loader css-loader style-loader vue-loader vue-template-compiler babel-loader @babel/core @babel/cli @babel/preset-env url-loader clean-webpack-plugin -D
```

因为比较多，就不挨个介绍了，应该还是比较清晰的，分别是用来解析样式文件、`vue`单文件、`js`文件及其他文件，可以根据你的实际情况增减。

先说一下打包目标，分别给每个包进行打包，打包结果输出到各自文件夹的`dist`目录下，我们使用`webpack`的`node API`来做：

```js
// ./bin/buildModule.js

const webpack = require('webpack')
const path = require('path')
const fs = require('fs-extra')
const {
    CleanWebpackPlugin
} = require('clean-webpack-plugin')
const {
    VueLoaderPlugin
} = require('vue-loader')

// 获取命令行参数，用来打包指定的包，否则打包packages目录下的所有包
const args = process.argv.slice(2)

// 生成webpack配置
const createConfigList = () => {
    const pkgPath = path.join(__dirname, '../', 'packages')
    // 根据是否传入了参数来判断要打的包
    const dirs = args.length  > 0 ? args : fs.readdirSync(pkgPath)
    // 给每个包生成一个webpack配置
    return dirs.map((item) => {
        return {
            // 入口文件为每个包里的index.js文件
            entry: path.join(pkgPath, item, 'index.js'),
            output: {
                filename: 'index.js',
                path: path.resolve(pkgPath, item, 'dist'),// 打包删除到dist文件夹下
                library: item,
                libraryTarget: 'umd',// 打包成umd模块
                libraryExport: 'default'
            },
            target: ['web', 'es5'],// webpack5默认打包生成的代码是包含const、let、箭头函数等es6语法的，所以需要设置一下生成es5的代码
            module: {
                rules: [
                    {
                        test: /\.css$/,
                        use: ['style-loader', 'css-loader']
                    },
                    {
                        test: /\.less$/,
                        use: ['style-loader', 'css-loader', 'less-loader']
                    },
                    {
                        test: /\.vue$/,
                        loader: 'vue-loader'
                    },
                    {
                        test: /\.js$/,
                        loader: 'babel-loader'
                    },
                    {
                        test: /\.(png|jpe?g|gif)$/i,
                        loader: 'url-loader',
                        options: {
                            esModule: false// 最新版本的file-loader默认使用es module的方式引入图片，最终生成的链接是个对象，所以如果是通过require方式引入图片就访问不了，可以通过该配置关掉
                        }
                    }
                ]
            },
            plugins: [
                new VueLoaderPlugin(),
                new CleanWebpackPlugin()
            ]
        }
    })
}

// 开始打包
webpack(createConfigList(), (err, stats) => {
    // 处理和结果处理...
})
```

然后运行命令`node ./bin/buildModule.js ` 即可打所有的包，或者`node ./bin/buildModule.js xxx xxx2 ...`来打你指定的包。

当然，这只是最简单的配置，实际上肯定还会遇到很多特定问题，比如：

- 如果依赖了其他基础组件库的话会比较麻烦，推荐这种情况就不要打包了，直接源码发布；

- 寻找文件时缺少`vue`扩展名，那么需要配置一下`webpack`的`resolve.extensions`；

- 使用了某些比较新的`JavaScript`语法或者用到`jsx`等，那么需要配置一下对应的`babel`插件或预设；

- 引用了`vue`、`jquery`等外部库，不可能直接打包进去，所以需要配置一下`webpack`的`externals`；

- 某个包可能有多个入口，换句话说也就是个别的包可能有特定的配置，那么可以在该包下面添加一个配置文件，然后上述生成配置的代码里可以读取该文件进行配置合并；

这些问题解决都不难，看一下报的错然后去搜索一下基本很容易就能解决，有兴趣的话也可以去本文的源码查看。

接下来做个小优化，因为`webpack`打包不是同时进行的，所以包的数量多了的话总时间就很慢，可以使用`parallel-webpack`这个插件来让它并行打包：

```bash
npm i parallel-webpack -D
```

因为它的`api`使用的是配置的文件路径，不能直接传递对象类型，所以需要修改一下上述的代码，改成导出一个配置的方式：

```js
// 文件名改成config.js

// ...

// 删除
// webpack(createConfigList(), (err, stats) => {
    // 处理和结果处理...
// })

// 增加导出语句
module.exports = createConfigList()
```

另外创建一个文件：

```js
// run.js

const run = require('parallel-webpack').run
const configPath = require.resolve('./config.js')

run(configPath, {
    watch: false,
    maxRetries: 1,
    stats: true
})
```

执行`node ./bin/run.js`即可执行，我简单计时了一下，节省了大约一半的时间。



# 组件文档配置

组件文档工具使用的是[VuePress](https://vuepress.vuejs.org/zh/)，如果跟我一样遇到了`webpack`版本冲突问题，可以选择在`./docs`目录下单独安装：

```bash
cd ./docs
npm init
npm install -D vuepress
```

`vuepress`的基本配置很简单，使用[默认主题](https://vuepress.vuejs.org/zh/theme/default-theme-config.html#%E9%A6%96%E9%A1%B5)按照教程配置即可，这里就不细说了，只说一下如何在文档里使用`packages`里的组件，先看一下当前目录结构：

![image-20210331103456876.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1be92ec84665475da1b981c356d7ce4c~tplv-k3u1fbpfcp-watermark.image)

`config.js`文件是`vuepress`的默认配置文件，打包选项、导航栏、侧边栏等等都在这里配置，`enhanceApp`是客户端应用的增强，在这里可以获取到`vue`实例，可以做一些应用启动的工作，比如注册组件等。

`zh/rate`是我添加的一个组件的文档，文档及示例内容都在文件夹下的`README.md`文件里，`vuepress`对`markdown`做了扩展，所以在`markdown`文件里可以使用像`vue`单文件一样包含`template`、`script`、`style`三个块，方便在文档里进行示例开发，组件需要先在`enhanceApp.js`文件里进行导入及注册，那么问题来了，我们是导入开发中的还是打包后的呢，小朋友才做选择，成年人都要，比如开发阶段我们就导入开发中的，开发完成了就导入打包后的，区别只是在于`package.json`里的`main`入口字段指向不同而已，比如我们先指向开发中的：

```js
// package.json

{
	"main": "index.js"
}
```

接下来去`enhanceApp.js`里导入及注册：

```js
import Rate from '@zf/rate'

export default ({
    Vue
}) => {
    Vue.use(Rate)
}
```

如果直接这样的话默认是会报错的，因为找不到这个包，此时我们的包也还没发布，所以也不能直接安装，那怎么办呢，办法应该有好几个，比如可以使用`npm link`来将包链接到这里，但是这样太麻烦，所以我选择修改一下`vuepress`的`webpack`配置，让它寻找包的时候顺便去找`packages`目录下找，另外也需要给`@zf`设置一下别名，显然我们的目录里并没有`@zf`，修改`webpack`的配置需要在`config.js`文件里操作：

```js
const path = require('path')

module.exports = {
    chainWebpack: (config) => {
        // 我们包存放的位置
        const pkgPath = path.resolve(__dirname, '../../../', 'packages')
        // 修改webpack的resolve.modules配置，解析模块时应该搜索的目录，先去packages，再去node_modules
        config.resolve
            .modules
            .add(pkgPath)
            .add('node_modules')
        // 修改别名resolve.alias配置
        config.resolve
            .alias
            .set('@zf', pkgPath)
    }
}
```

这样在`vuepress`里就可以正常使用我们的组件了，当你开发完成后就可以把这个包`package.json`的入口字段改成打包后的目录：

```json
// package.json

{
	"main": "dist/index.js"
}
```

其他基本信息、导航栏、侧边栏等可以根据你的需求进行配置，效果如下：

![image-20210331160539487.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2967849358684e349f09a49eb6407094~tplv-k3u1fbpfcp-watermark.image)

![image-20210331160554265.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/accdd8ce306e40d29d47c43f46e2e6e0~tplv-k3u1fbpfcp-watermark.image)

# 使用脚本新增组件

现在让我们来看一下新增一个组件都有哪些步骤：

1.给要新增的组件取个名字，然后使用`npm search xxx`来检查一下是否已存在，存在就换个名字；

2.在`packages`目录下创建文件夹，新建几个基本文件，通常来说是复制粘贴其他组件然后修改；

3.在`docs`目录下创建文档文件夹，新建`README.md`文件，文件内容一般也是通过复制粘贴；

4.修改`config.js`进行侧边栏配置（如果配置了侧边栏的话）、修改`enhanceApp.js`导入及注册组件；

这一套步骤下来虽然不难，但是繁琐，很容易漏掉某一步，上述这些事情其实特别适合让脚本来干，接下来就实现一下。



## 初始化工作

先在`./bin`目录下新建一个`add.js`文件，这个就是咱们要执行的脚本，首先它肯定要接收一些参数，简单起见这里只需要输入一个组件名，但是为了后续扩展方便，我们使用[inquirer](https://github.com/SBoudrias/Inquirer.js)来处理命令行输入，接收到输入的组件名称后自动进行一下是否已存在的校验：

```js
// add.js
const {
    exec
} = require('child_process')
const inquirer = require('inquirer')
const ora = require('ora')// ora是一个命令行loading工具
const scope = '@zf/'// 包的作用域，如果你的包没有作用域，那么则不需要

inquirer
    .prompt([{
            type: 'input',
            name: 'name',
            message: '请输入组件名称',
            validate(input) {
                // 异步验证需要调用这个方法来告诉inquirer是否校验完成
                const done = this.async();
                input = String(input).trim()
                if (!input) {
                    return done('请输入组件名称')
                }
                const spinner = ora('正在检查包名是否存在').start()
                exec(`npm search ${scope + input}`, (err, stdout) => {
                    spinner.stop()
                    if (err) {
                        done('检查包名是否存在失败，请重试')
                    } else {
                        if (/No matches/.test(stdout)) {
                            done(null, true)
                        } else {
                            done('该包名已存在，请修改')
                        }
                    }
                })
            }
        }
    ])
    .then(answers => {
    	// 命令行输入完成，进行其他操作
        console.log(answers)
    })
    .catch(error => {
        // 错误处理
    });
```

执行后效果如下：

![image-20210331165515073.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42baf97388f74e68a7a48b2fa1811021~tplv-k3u1fbpfcp-watermark.image)


## 使用模板创建

接下来在`packages`目录下自动生成文件夹及文件，在【打包配置】一节中可以看到一个基本的包一共有四个文件：`index.js`、`package.json`、`index.vue`以及`style.less`，首先在`./bin`目录下创建一个`template`文件夹，然后再新建这四个文件，基本内容可以先复制粘贴进去，其中`index.js`和`style.less`的内容不需要修改，所以直接复制到新组件的目录下即可：

```js
// add.js

const upperCamelCase = require('uppercamelcase')// 字符串-风格的转驼峰
const fs = require('fs-extra')

const templateDir = path.join(__dirname, 'template')// 模板路径

// 这个方法在上述inquirer的then方法里调用，参数为命令行输入的信息
const create = ({
    name
}) => {
    // 组件目录
    const destDir = path.join(__dirname, '../', 'packages', name)
    const srcDir = path.join(destDir, 'src')
    // 创建目录
    fs.ensureDirSync(destDir)
    fs.ensureDirSync(srcDir)
    // 复制index.js和style.less
    fs.copySync(path.join(templateDir, 'index.js'), path.join(destDir, 'index.js'))
    fs.copySync(path.join(templateDir, 'style.less'), path.join(srcDir, 'style.less'))
}
```

`index.vue`和`package.json`内容的部分信息需要动态注入，比如`index.vue`的组件名、`package.json`的包名，我们可以使用一个很简单的库[json-templater](https://github.com/lightsofapollo/json-templater)来以双大括号插值的方法来注入数据，以`package.json`为例：

```json
// ./bin/template/package.json
{
    "name": "{{name}}",
    "version": "1.0.0",
    "description": "",
    "main": "index.js",
    "scripts": {},
    "author": "",
    "license": "ISC"
}
  
```

`name`是我们要注入的数据，接下来读取模板的内容，然后注入并渲染，最后创建文件：

```js
// add.js

const upperCamelCase = require('uppercamelcase')// 字符串-风格的转驼峰
const render = require('json-templater/string')

// 渲染模板及创建文件
const renderTemplateAndCreate = (file, data = {}, dest) => {
    const templateContent = fs.readFileSync(path.join(templateDir, file), {
        encoding: 'utf-8'
    })
    const fileContent = render(templateContent, data)
    fs.writeFileSync(path.join(dest, file), fileContent, {
        encoding: 'utf-8'
    })
}

const create = ({
    name
}) => {
    // 组件目录
    // ...
    // 创建package.json
    renderTemplateAndCreate('package.json', {
        name: scope + name
    }, destDir)
    // index.vue
    renderTemplateAndCreate('index.vue', {
        name: upperCamelCase(name)
    }, srcDir)
}
```

到这里组件的目录及文件就创建完成了，文档的目录及文件也是一样，这里就不贴代码了。



## 使用AST修改

最后要修改的两个文件是`config.js`和`enhanceApp.js`，这两个文件虽然也可以向上面一样使用模板注入的方式，但是考虑到这两个文件修改的频率可能比较频繁，所以每次都得去模板里修改不太方便，所以我们换一种方式，使用`AST`，这样就不需要模板的占位符了。

先看`enhanceApp.js`，每增加一个组件，我们都需要在这里导入和注册：

```js
import Rate from '@zf/rate'

export default ({
    Vue
}) => {
    Vue.use(Rate)
    console.log(1)
}
```

思路很简单，把这个文件的源代码先转换成`AST`，然后在最后一个`import`语句后面插入新组件的导入语句，以及在最后一条`Vue.use`语句和`console.log`语句之间插入新组件的注册语句，最后再转换回源码写回到这个文件里，`AST`相关的操作可以使用`babel`的工具包：[@babel/parser](https://www.npmjs.com/package/@babel/parser)、[@babel/traverse](https://www.npmjs.com/package/@babel/traverse)、[@babel/generator](https://www.npmjs.com/package/@babel/generator)、[@babel/types](https://www.npmjs.com/package/@babel/types)。

### @babel/parser

把源代码转换成`AST`很简单：

```js
// add.js
const parse = require('@babel/parser').parse

// 更新enhanceApp.js
const updateEnhanceApp = ({
    name
}) => {
    // 读取文件内容
    const filePath = path.join(__dirname, '../', 'docs', 'docs', '.vuepress', 'enhanceApp.js')
    const code = fs.readFileSync(filePath, {
        encoding: 'utf-8'
    })
    // 转换成AST
    const ast = parse(code, {
        sourceType: "module"// 因为用到了`import`语法，所以指明把代码解析成module模式
    })
    console.log(ast)
}
```

生成的数据很多，所以命令行一般都显示不下去，可以去[https://astexplorer.net/](https://astexplorer.net/)这个网站上查看，选择`@babel/parser`的解析器即可。

![image-20210401152359630.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32ab932eddae44a5a9e120e298d8d037~tplv-k3u1fbpfcp-watermark.image)


### @babel/traverse

得到了`AST`树之后就需要修改这颗树，`@babel/traverse`用来遍历和修改树节点，这是整个过程中相对麻烦的一个步骤，如果不熟悉`AST`的基础知识和操作的话推荐先阅读一下这篇文档[babel-handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)。

接下来我们对着上面解析的截图来写一下添加`import`语句的代码：

```js
// add.js
const traverse = require('@babel/traverse').default
const t = require("@babel/types")// 这个包是一个工具包，用来检测某个节点的类型、创建新节点等

const updateEnhanceApp = ({
    name
}) => {
    // ...
    
    // traverse的第一个参数是ast对象，第二个是一个访问器，当遍历到某种类型的节点后会调用对应的函数
    traverse(ast, {
        // 遍历到了Program节点会执行该函数
        // 函数的第一个参数并不是节点本身，而是代表节点的路径，路径上会包含该节点和其他节点之间的关系信息，后续的一些操作也都是在路径上进行，如果要访问节点本身，可以访问path.node
        Program(nodePath) {
            let bodyNodesList = nodePath.node.body // 通过上图可以看到是个数组
            // 遍历节点找到最后一个import节点
            let lastImportIndex = -1
            for (let i = 0; i < bodyNodesList.length; i++) {
                if (t.isImportDeclaration(bodyNodesList[i])) {
                    lastImportIndex = i
                }
            }
            // 构建即将要插入的import语句的AST节点：import name from @zf/name
            // 节点类型及需要的参数可以在这里查看：https://babeljs.io/docs/en/babel-types
            // 如果不确定使用哪个类型的话可以在上述的https://astexplorer.net/网站上看一下某个语句对应的是什么
            const newImportNode = t.importDeclaration(
                [ t.ImportDefaultSpecifier(t.Identifier(upperCamelCase(name))) ], // name
                t.StringLiteral(scope + name)
            )
            // 当前没有import节点，则在第一个节点之前插入import节点
            if (lastImportIndex === -1) {
                let firstPath = nodePath.get('body.0')// 获取body的第一个节点的path
                firstPath.insertBefore(newImportNode)// 在该节点之前插入节点
            } else { // 当前存在import节点，则在最后一个import节点之后插入import节点
                let lastImportPath = nodePath.get(`body.${lastImportIndex}`)
                lastImportPath.insertAfter(newImportNode)
            }
        }
    });
}
```

接下来看一下添加`Vue.use`的代码，因为生成的`AST`树太长了，所以不方便截图，大家可以打开上面的网站输入示例代码后看生成的结构：

```js
// add.js

// ...

traverse(ast, {
    Program(nodePath) {},
    
    // 遍历到ExportDefaultDeclaration节点
    ExportDefaultDeclaration(nodePath) {
        let bodyNodesList = nodePath.node.declaration.body.body // 找到箭头函数节点的body，目前存在两个表达式节点
        // 下面的逻辑和添加import语句的逻辑基本一致，遍历节点找到最后一个vue.use类型的节点，然后添加一个新节点
        let lastIndex = -1
        for (let i = 0; i < bodyNodesList.length; i++) {
            let node = bodyNodesList[i]
            // 找到vue.use类型的节点
            if (
                t.isExpressionStatement(node) &&
                t.isCallExpression(node.expression) &&
                t.isMemberExpression(node.expression.callee) &&
                node.expression.callee.object.name === 'Vue' &&
                node.expression.callee.property.name === 'use'
            ) {
                lastIndex = i
            }
        }
        // 构建新节点：Vue.use(name)
        const newNode = t.expressionStatement(
            t.callExpression(
                t.memberExpression(
                    t.identifier('Vue'),
                    t.identifier('use')
                ),
                [ t.identifier(upperCamelCase(name))]
            )
        )
        // 插入新节点
        if (lastIndex === -1) {
            if (bodyNodesList.length > 0) {
                let firstPath = nodePath.get('declaration.body.body.0')
                firstPath.insertBefore(newNode)
            } else {// body为空的话需要调用`body`节点的pushContainer方法追加节点
                let bodyPath = nodePath.get('declaration.body')
                bodyPath.pushContainer('body', newNode)
            }
        } else {
            let lastPath = nodePath.get(`declaration.body.body.${lastIndex}`)
            lastPath.insertAfter(newNode)
        }
    }
});
```



### @babel/generator

`AST`树修改完成接下来就可以转回源代码了：

```js
//  add.js
const generate = require('@babel/generator').default

const updateEnhanceApp = ({
    name
}) => {
    // ...
    
    // 生成源代码
    const newCode = generate(ast)
}
```

效果如下：

![image-20210401164116584.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbb2630650b549a6885f53d1329c8bf6~tplv-k3u1fbpfcp-watermark.image)

可以看到使用`AST`进行简单的操作并不难，关键是要细心及耐心，找对节点。另外对`config.js`的修改也是大同小异，有兴趣的可以直接看源码。

到这里我们只要使用`npm run add`命令就可以自动化的创建一个新组件，可以直接进行组件开发了~



# 结尾

本文是笔者在改造自己组件库的一些过程记录，因为是第一次实践，难免会有错误或不合理的地方，欢迎指出，感谢阅读，再会~

示例代码仓库：[https://github.com/wanglin2/vue_components](https://github.com/wanglin2/vue_components)。
