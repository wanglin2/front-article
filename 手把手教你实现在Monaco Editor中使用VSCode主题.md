# 背景

笔者开源了一个小项目[code-run](https://github.com/wanglin2/code-run)，类似`codepen`的一个工具，其中代码编辑器使用的是微软的[Monaco Editor](https://microsoft.github.io/monaco-editor/)，这个库是直接从`VSCode`的源码中生成的，只不过是做了一点修改让它支持在浏览器中运行，但是功能基本是和`VSCode`一样强大的，所以在笔者看来`Monaco Editor`等于`VSCode`的编辑器核心。

另外笔者是一个颜控，不管做什么项目，都热衷于配套一些好看的皮肤、主题，所以`Moncao Editor`仅仅内置了三种主题是远远满足不了笔者需求的，况且还都很丑，于是结合`Monaco Editor`和`VSCode`的关系就很自然的想到，能不能直接复用`VSCode`的主题，接下来就给大家介绍一下笔者的探索之路。

> ps.想直接了解如何实现的可以跳转到【具体实现】小节。



# 基本使用

先看一下`Monaco Editor`的基本使用，首先安装：

```bash
npm install monaco-editor
```

然后引入：

```js
import * as monaco from 'monaco-editor'

// 创建一个js编辑器
const editor = monaco.editor.create(document.getElementById('container'), {
    value: ['function x() {', '\tconsole.log("Hello world!");', '}'].join('\n'),
    language: 'javascript',
    theme: 'vs'
})
```

这样就可以在`container`元素上创建一个`js`语言的编辑器，并且使用了内置的`vs-dark`主题。如果遇到报错或者语法提示不生效，那么可能需要配置一下`worker`文件的路径，可以参考官方示例[browser-esm-webpack](https://github.com/microsoft/monaco-editor-samples/tree/main/browser-esm-webpack)。



# 自定义主题

`Monaco Editor`支持自定义主题，方法如下：

```js
// 定义主题
monaco.editor.defineTheme(themeName, themeData)
// 使用定义的主题
monaco.editor.setTheme(themeName)
```

`themeName`是要自定义的主题名称，比如`OneDarkPro`，`themeData`是一个对象，即主题数据，基本结构如下：

```js
{
    base: 'vs',// 要继承的基础主题，即内置的三个：vs、vs-dark、hc-black
    inherit: false,// 是否继承
    rules: [// 高亮规则，即给代码里不同token类型的代码设置不同的显示样式
        { token: '', foreground: '000000', background: 'fffffe' }
    ],
    colors: {// 非代码部分的其他部分的颜色，比如背景、滚动条等
        [editorBackground]: '#FFFFFE'
    }
}
```

`rules`里面就是用来给代码进行高亮的，常见的`token`有`string`（字符串）、`comment`（注释）、`keyword`（关键词）等等，完整的请移步[themes.ts](https://github.com/microsoft/vscode/blob/main/src/vs/editor/standalone/common/themes.ts)，这些`token`是怎么确定的呢，`Monaco Editor`内置了一个语法着色器[Monarch](https://microsoft.github.io/monaco-editor/monarch.html)，本质是通过正则表达式来匹配，然后给匹配到的内容命名为一个`token`。

可以直接在编辑器中查看代码某块对应的`token`，按`F1`或鼠标右键点击`Command Palette`，然后再找到并点击`Developer: Inspect Tokens`，接下来鼠标点哪一块代码，就会显示对应的信息，包括`token`类型，当前应用的颜色等。



# 踩坑

最开始的想法很简单，直接找到`VSCode`的主题文件，然后通过自定义主题来使用。



## 获取`VSCode`主题文件

有两种方法，如果某个主题已经在你的`VSCode`里安装并正在使用的话，那么可以按`F1`或`Command/Control + Shift + P`或鼠标右键点击`Command Palette/命令面板`，接着找到并点击`Developer:Generate Color Theme From Current Setting/开发人员:使用当前设置生成颜色主题`，然后`VSCode`就会生成一份`json`数据，保存即可。

如果某个主题没有安装的话，那么可以去[vscode主题商店](https://marketplace.visualstudio.com/search?target=VSCode&category=Themes&sortBy=Installs)搜索该主题，进入主题详情页面后点击右侧的`Download Extension`按钮即可下载该主题，下载完成后找到刚才下载的文件，文件应该是以`.vsix`结尾的，直接把该后缀改成`.zip`，然后解压缩，最后打开里面的`/extension/themes/`文件夹，里面的`.json`文件即主题文件，打开该文件复制`json`数据即可。



## 把`VSCode`主题转换成`Monaco Editor`主题格式

上一步过后你应该可以发现`VSCode`主题的格式是这样的：

```json
{
	"$schema": "vscode://schemas/color-theme",
	"type": "dark",
	"colors": {
		"activityBar.background": "#282c34"
    },
    "tokenColors": [
        {
			"scope": "variable.other.generic-type.haskell",
			"settings": {
				"foreground": "#C678DD"
			}
		}，
        {
			"scope": [
				"punctuation.section.embedded.begin.php",
				"punctuation.section.embedded.end.php"
			],
			"settings": {
				"foreground": "#BE5046"
			}
		}
    ]
}  
```

跟`Monaco Editor`的主题格式有一点区别，那是不是可以写一个转换方法把它转换成下面这样呢：

```js
{
    base: 'vs',
    inherit: false,
    rules: [
        { token: 'variable.other.generic-type.haskell', foreground: '#C678DD' },
        { token: 'punctuation.section.embedded.begin.php', foreground: '#BE5046' },
        { token: 'punctuation.section.embedded.end.php', foreground: '#BE5046' }
    ],
    colors: {
        "activityBar.background": "#282c34"
    }
}
```

当然可以，这也不难，但是最后当你使用这个自定义的主题后会发现，没有效果，为什么呢，去[Monarch](https://microsoft.github.io/monaco-editor/monarch.html)看一下对应语言的解析配置后就会发现，压根就没有`VSCode`主题里定义的这些`token`，有效果才奇怪，那怎么办呢，自己扩展这个解析的配置吗，笔者最开始就是这么做的，写正则表达式嘛，应该也不是很难，为此，笔者还把`Monarch`文档完整翻译了一遍[Monarch中文](https://github.com/wanglin2/front-article/blob/main/Monarch%E4%B8%AD%E6%96%87%E6%96%87%E6%A1%A3.md)，但是当笔者在`VSCode`里看到如下效果时：

![image-20210918142132745.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20b39a45d94d441fbe22f3fb4438919c~tplv-k3u1fbpfcp-watermark.image?)

果断放弃，这显然是要进行语义分析才行，否则谁知道`abc`是个变量。

其实在`VSCode`里语法高亮使用的是`TextMate`，而在`Monaco Editor`里使用的是`Monarch`，两者压根不是一个东西，为什么`Monaco Editor`不使用`TextMate`，而是要开发一个新的东西呢，原因是`VSCode`使用的是[vscode-textmate](https://github.com/Microsoft/vscode-textmate)来解析`TextMate`语法，这个库依赖一个`Oniguruma`正则表达式库，而这个正则表达式库是使用`C`语言开发的，当然不支持在浏览器上运行。



## 退而求其次

既然`VSCode`的主题不能直接使用，那么就只能能用多少用多少，因为`Monaco Editor`内置的主题`token`就只有那么多，那么把它所有的`token`颜色换成`VSCode`的主题颜色不就行了吗，虽然语义高亮没有，但是总比默认主题好看。实现也很简单，首先`colors`部分的基本可以直接使用，而`token`部分可以通过上面介绍的方法`Developer: Inspect Tokens`在`VSCode`里找到对应代码块的颜色，复制到`Monaco Editor`主题的对应`token`上即可，比如笔者转换后的`OneDarkPro`的实际效果如下：

![image-20210918143406409.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f1e827fd88b4d3ea1b7bb2cae12f67e~tplv-k3u1fbpfcp-watermark.image?)

在`VSCode`里的效果如下：

![image-20210918143427581.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e7609917fdb44fd8947cc0924c21613~tplv-k3u1fbpfcp-watermark.image?)

只可粗看，不要细究。

这个事情也有人已经做了，可以参考这个仓库[monaco-themes](https://github.com/brijeshb42/monaco-themes)，里面帮你转换了一些常见的主题，可以拿来直接使用。



## 新的曙光

就在笔者已经放弃在`Monaco Editor`中直接使用`VSCode`主题的想法后，无意间发现[codesandbox](https://codesandbox.io/)和[leetcode](https://leetcode-cn.com/)两个网站中的编辑器主题效果和`VSCode`中基本一致，而且可以明显的看到在`leetcode`中切换主题请求的文件：

![image-20210918161935357.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2702baa33fca4fe1bf300bb2099bf734~tplv-k3u1fbpfcp-watermark.image?)

基本和`VSCode`主题格式是一样的，这就说明在`Monaco Editor`中使用`VSCode`主题是可以实现的，那么问题就变成了怎么实现。



# 实现

不得不说，这方面资料真的很少，相关文章基本没有，百度搜索结果里只有一两个相关的链接，不过也足以解决问题了，相关链接详见文章尾部。

主要使用的是[monaco-editor-textmate](https://github.com/NeekSandhu/monaco-editor-textmate)这个工具（所以除了百度谷歌之外，`github`也是一个很重要的搜索引擎啊），先安装：

```bash
npm i monaco-editor-textmate
```

`npm`应该会同时帮你再安装[monaco-textmate](https://github.com/NeekSandhu/monaco-textmate)、[onigasm](https://github.com/NeekSandhu/onigasm)、`monaco-editor`这几个包，`monaco-editor`自不必说，我们自己都装了，其他两个可以自行检查一下，如果没有的话需要自行安装。

## 工具介绍

简单介绍一下这几个包。

### onigasm

这个库就是用来解决上述浏览器不支持`C`语言编写的`Oniguruma`的问题，解决方法是把`Oniguruma`编译为[WebAssembly](https://webassembly.org/)，`WebAssembly`是一种中间格式，可以把非`js`代码编译成`.wasm`格式的文件，然后浏览器就可以加载并运行它了，`WebAssembly`已经是`WEB`的标准之一了，随着时间的推移，相信兼容性也不是问题。

### monaco-textmate

这个库是在`VSCode`使用的`vscode-textmate`库的基础上修改的， 以便让它在浏览器上使用。主要作用是解析`TextMate`语法，这个库依赖前面的`onigasm`。

### monaco-editor-textmate

这个库的主要作用是帮我们把`monaco-editor`和`monaco-textmate`关联起来，内部首先会加载对应语言的`TextMate`语法文件，然后调用[monaco.languages.setTokensProvider](https://microsoft.github.io/monaco-editor/api/modules/monaco.languages.html#settokensprovider)方法来自定义语言的`token`解析器。

看一下它的使用示例：

```js
import { loadWASM } from 'onigasm'
import { Registry } from 'monaco-textmate'
import { wireTmGrammars } from 'monaco-editor-textmate'
export async function liftOff() {
    await loadWASM(`path/to/onigasm.wasm`)
    const registry = new Registry({
        getGrammarDefinition: async (scopeName) => {
            return {
                format: 'json',
                content: await (await fetch(`static/grammars/css.tmGrammar.json`)).text()
            }
        }
    })
    const grammars = new Map()
    grammars.set('css', 'source.css')
    grammars.set('html', 'text.html.basic')
    grammars.set('typescript', 'source.ts')
    monaco.editor.defineTheme('vs-code-theme-converted', {});
    var editor = monaco.editor.create(document.getElementById('container'), {
        value: [
            'html, body {',
            '    margin: 0;',
            '}'
        ].join('\n'),
        language: 'css',
        theme: 'vs-code-theme-converted'
    })
    await wireTmGrammars(monaco, registry, grammars, editor)
}
```



## 具体实现

看完前面的使用示例后，接下来我们详细看一下如何使用。

### 加载onigasm

首先我们要做的是加载`onigasm`的`wasm`文件，这个文件需要首先被加载，且加载一次就可以了，所以我们在编辑器初始化前进行加载：

```js
import { loadWASM } from 'onigasm'
const init = async () => {
    await loadWASM(`${base}/onigasm/onigasm.wasm`)
    // 创建编辑器...
}
init()
```

`onigasm.wasm`文件可以在`/node_modules/onigasm/lib/`目录下找到，然后复制到项目的`/public/onigasm/`目录下，这样可以通过`http`进行请求。

### 创建作用域映射

接下来创建语言`id`到作用域名称的映射：

```js
const grammars = new Map()
grammars.set('css', 'source.css')
```

其他语言的作用域名称可以在[各种语言的语法列表](https://github.com/microsoft/vscode/tree/94c9ea46838a9a619aeafb7e8afd1170c967bb55/extensions)这里找到，比如想知道`css`的作用域名称，我们进入`css`目录，然后打开`package.json`文件，可以看到其中有一个`grammars`字段：

```js
"grammars": [
    {
        "language": "css",
        "scopeName": "source.css",
        "path": "./syntaxes/css.tmLanguage.json",
        "tokenTypes": {
            "meta.function.url string.quoted": "other"
        }
    }
]
```

`language`就是语言`id`，`scopeName`就是作用域名称。常见的如下：

```js
const scopeNameMap = {
    html: 'text.html.basic',
    pug: 'text.pug',
    css: 'source.css',
    less: 'source.css.less',
    scss: 'source.css.scss',
    typescript: 'source.ts',
    javascript: 'source.js',
    javascriptreact: 'source.js.jsx',
    coffeescript: 'source.coffee'
}
```

### 注册语法映射

再接着注册`TextMate`的语法映射关系，这样可以通过作用域名称来加载并创建对应的语法：

```js
import {
    Registry
} from 'monaco-textmate'

// 创建一个注册表，可以从作用域名称来加载对应的语法文件
const registry = new Registry({
    getGrammarDefinition: async (scopeName) => {
        return {
            format: 'json',// 语法文件格式，有json、plist
            content: await (await fetch(`${base}grammars/css.tmLanguage.json`)).text()
        }
    }
})
```

语法文件和前面的作用域名称一样，也是在[各种语言的语法列表](https://github.com/microsoft/vscode/tree/94c9ea46838a9a619aeafb7e8afd1170c967bb55/extensions)这里找，同样以`css`语言为例，还是看它的`package.json`的`grammars`字段：

```js
"grammars": [
    {
        "language": "css",
        "scopeName": "source.css",
        "path": "./syntaxes/css.tmLanguage.json",
        "tokenTypes": {
            "meta.function.url string.quoted": "other"
        }
    }
]
```

`path`字段就是对应的语法文件的路径，我们把这些`json`文件复制到项目的`/public/grammars/`目录下，这样就可以通过`fetch`来请求到。

### 定义主题

前面介绍过，`Monaco Editor`的主题格式和`VSCode`的格式是有点不一样的，所以需要进行转换，转换可以自己实现，也可以直接使用[monaco-vscode-textmate-theme-converter](https://github.com/Nishkalkashyap/monaco-vscode-textmate-theme-converter)这个工具，它可以同时转换多个本地文件：

```js
// convertTheme.js
const converter = require('monaco-vscode-textmate-theme-converter')
const path = require('path')

const run = async () => {
    try {
        await converter.convertThemeFromDir(
            path.resolve(__dirname, './vscodeThemes'), 
            path.resolve(__dirname, '../public/themes')
        );
    } catch (error) {
        console.log(error)
    }
}
run()
```

运行`node ./convertTheme.js`命令后，就会把你放在`vscodeThemes`目录下所有`VSCode`的主题文件转换成`Monaco Editor`的主题文件并输出到`public/themes`目录下，然后我们在代码里直接通过`fetch`来请求主题文件并使用`defineTheme`方法定义主题即可：

```js
// 请求OneDarkPro主题文件
const themeData = await (
    await fetch(`${base}themes/OneDarkPro.json`)
).json()
// 定义主题
monaco.editor.defineTheme('OneDarkPro', themeData)
```

### 设置token解析器

经过前面这些准备工作，最后一步要做的是设置`Monaco Editor`的`token`解析器，默认使用的是内置的`Monarch`，我们要换成`TextMate`的解析器，也就是`monaco-editor-textmate`做的事情：

```js
import {
    wireTmGrammars
} from 'monaco-editor-textmate'
import * as monaco from 'monaco-editor'

let editor = monaco.editor.create(document.getElementById('container'), {
    value: [
        'html, body {',
        '    margin: 0;',
        '}'
    ].join('\n'),
    language: 'css',
    theme: 'OneDarkPro'
})

await wireTmGrammars(monaco, registry, grammars, editor)
```

### 问题1

上一步后应该可以看到`VSCode`的主题在`Monaco Editor`上生效了，但是多试几次可能会发现偶尔会失效，原因是`Monaco Editor`内置的语言是延迟加载的，并且加载完后也会同样注册一个`token`解析器，所以会把我们的给覆盖掉，详见`issue`：[setTokensProvider unable to override existing tokenizer](https://github.com/Microsoft/monaco-editor/issues/884)。

一种解决方法是去除内置的语言，这可以使用[monaco-editor-webpack-plugin](https://github.com/microsoft/monaco-editor-webpack-plugin)。

安装：

```bash
npm install monaco-editor-webpack-plugin -D
```

`Vue`项目配置如下：

```js
// vue.config.js
const MonacoWebpackPlugin = require('monaco-editor-webpack-plugin')

module.exports = {
    configureWebpack: {
        plugins: [
            new MonacoWebpackPlugin({
                languages: []
            })
        ]
    }
}
```

`languages`选项用来指定要包含的语言，我们直接设为空，啥也不要。

然后修改`Monaco Editor`的引入方式为：

```js
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api'
```

最后需要手动注册我们需要的语言，因为所有内置语言都被去除了嘛，比如我们要使用`js`语言的话：

```js
monaco.languages.register({id: 'javascript'})
```

这种方法虽然可以完美解决该问题，但是很大的一个副作用是语法提示不生效了，因为只有包含了内置的`html`、`css`、`typescript`时才会去加载对应的`worker`文件，没有语法提示笔者也是无法接受的，所以最后笔者使用了一种比较`low`的`hack`方式：

```js
// 插件配置
new MonacoWebpackPlugin({
    languages: ['css', 'html', 'javascript', 'less', 'pug', 'scss', 'typescript', 'coffee']
})

// 注释掉语言注册语句
// monaco.languages.register({id: 'javascript'})

// 当worker文件被加载了后再wire
let hasGetAllWorkUrl = false
window.MonacoEnvironment = {
    getWorkerUrl: function (moduleId, label) {
        hasGetAllWorkUrl = true
        if (label === 'json') {
            return './monaco/json.worker.bundle.js'
        }
        if (label === 'css' || label === 'scss' || label === 'less') {
            return './monaco/css.worker.bundle.js'
        }
        if (label === 'html' || label === 'handlebars' || label === 'razor') {
            return './monaco/html.worker.bundle.js'
        }
        if (label === 'typescript' || label === 'javascript') {
            return './monaco/ts.worker.bundle.js'
        }
        return './monaco/editor.worker.bundle.js'
    },
}
// 循环检测
let loop = () => {
    if (hasGetAllWorkUrl) {
        Promise.resolve().then(async () => {
            await wireTmGrammars(monaco, registry, grammars, editor)
        })
    } else {
        setTimeout(() => {
            loop()
        }, 100)
    }
}
loop()
```

### 问题2

笔者遇到的另外一个问题是，转换后有些主题的默认颜色并未设置，所以都是黑色，很丑：

![image-20210924105525593.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbc20920c6f44b8c8db2d447e4c4a6eb~tplv-k3u1fbpfcp-watermark.image?)

这个问题的解决方法是可以给主题的`rules`数组添加一个空的`token`，用来作为没有匹配到的默认`token`：

```js
{
    "rules": [
        {
            "foreground": "#abb2bf",
            "token": ""
        }
     ]
}
```

`foreground`的色值可以取`colors`选项里的`editor.foreground`的值，要手动修改每个色值比较麻烦，可以在之前的转换主题的步骤里顺便进行，会在下一个问题里一起解决。

### 问题3 

[monaco-vscode-textmate-theme-converter](https://github.com/Nishkalkashyap/monaco-vscode-textmate-theme-converter)这个包本质算是`nodejs`环境下的工具，所以想在纯前端环境下使用不太方便，另外它对于非标准`json`格式的`VSCode`主题转换时会报错，因为很多主题格式是`.jsonc`，内容是带有很多注释的，所以都需要自己先进行检查并修改，不是很方便，基于这两个问题，笔者`fork`了它的代码，然后修改并分成了两个包，分别对应`nodejs`和`浏览器`环境，详见[https://github.com/wanglin2/monaco-vscode-textmate-theme-converter](https://github.com/wanglin2/monaco-vscode-textmate-theme-converter)。

所以我们可以替换掉`monaco-vscode-textmate-theme-converter`，改成安装笔者的：

```bash
npm i vscode-theme-to-monaco-theme-node -D
```

使用方式基本是一样的：

```js
// 只要修改引入为笔者的包即可
const converter = require('vscode-theme-to-monaco-theme-node')
const path = require('path')

const run = async () => {
    try {
        await converter.convertThemeFromDir(
            path.resolve(__dirname, './vscodeThemes'), 
            path.resolve(__dirname, '../public/themes')
        );
    } catch (error) {
        console.log(error)
    }
}
run()
```

现在就可以直接转换`.jsonc`文件，而且输出统一为`.json`文件，另外内部会自动添加一个空的`token`作为没有匹配到的默认`token`，效果如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98c617e85c1840c5aa3700e41b9fd783~tplv-k3u1fbpfcp-watermark.image?)

### 最佳实践

`VSCode`主题除了代码主题外，一般还包含编辑器其他部分的主题，比如标题栏、状态栏、侧边栏、按钮等等，所以我们也可以在页面应用这些样式，达到整个页面的主题也能随编辑器代码主题一起切换的效果，这样能让页面整体更加协调，具体的实现上，我们可以使用`CSS`变量，先把页面所有涉及到的颜色都定义成`CSS`变量，然后在切换主题时根据主题的`colors`选项里的指定字段来更新变量即可，具体使用哪个字段来对应页面的哪个部分可以根据实际情况来确定，`VSCode`主题的所有可配置项可以在[theme-color](https://code.visualstudio.com/api/references/theme-color)这里找到。效果如下：

![2021-09-27-10-46-47.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9ff875738fb4ef1bf95104a9a994319~tplv-k3u1fbpfcp-watermark.image?)

# 总结

本文完整详细的介绍了笔者对于`Monaco Editor`编辑器主题的探索，希望能给有主题定制需求的小伙伴们一点帮助，完整的代码请参考本项目源码：[code-run](https://github.com/wanglin2/code-run)。



# 参考链接

文章：[monaco使用vscode相关语法高亮在浏览器上显示](https://segmentfault.com/a/1190000024525323) 

文章：[codesandbox是如何解决主题的问题](https://codesandbox.io/post/introducing-themes)

文章：[闲谈Monaco Editor-自定义语言之Monarch](https://zhuanlan.zhihu.com/p/52316626)

讨论：[如何在Monaco Editor中使用VSC主题？](https://stackoverflow.com/questions/65959169/how-to-use-a-vsc-theme-in-monaco-editor)

讨论：[使用WebAssembly来支持TextMate语法](https://github.com/microsoft/monaco-editor/issues/1915)
