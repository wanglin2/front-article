# 场景复现

笔者最近给自己的项目[CodeRun](https://github.com/wanglin2/code-run)增加了一个直接在浏览器上使用`ES`模块的功能，之前使用一个包前需要先找到它的在线`CDN`地址然后引进来，就像这样：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ed7b225837e46899f202b0ec5542302~tplv-k3u1fbpfcp-zoom-1.image)

现在可以直接这样：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5e36c239f804753acceec417440f68d~tplv-k3u1fbpfcp-zoom-1.image)

那么这是怎么实现的呢，很简单，使用[Skypack](https://www.skypack.dev/)，上图中的导入语句实际上最终会变成这样：

```js
import rough from 'https://cdn.skypack.dev/roughjs'
```

这个转换是通过`babel`实现的，我们可以写个`babel`插件，当访问到`import`语句时，判断如果是”裸“导入就拼接上`Skypack`的地址：

```js
// 转换导入语句
const transformJsImport = (jsStr) => {
	return window.Babel.transform(jsStr, {
		plugins: [
			parseJsImportPlugin()
		]
	}).code
}

// 修改import from语句
const parseJsImportPlugin = () => {
    return function (babel) {
        let t = babel.types
        return {
            visitor: {
                ImportDeclaration(path) {
		    // 是裸导入则替换该节点
                    if (isBareImport(path.node.source.value)) {
                        path.replaceWith(t.importDeclaration(
                            path.node.specifiers,
                            t.stringLiteral(`https://cdn.skypack.dev/${path.node.source.value}`)
                        ))
                    }
                }
            }
        }
    }
}

// 检查是否是裸导入
// 合法的导入格式有：http、./、../、/
const isBareImport = (source) => {
    return !(/^https?:\/\//.test(source) || /^(\/|\.\/|\.\.\/)/.test(source));
}
```

此外，还需要给`script`标签添加一个`type="module"`的属性，因为浏览器默认不会把`script`当做`ES`模块，只有设置了这个属性才能使用模块语法。

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7217634aba284e5d8652b2d1dec1dcde~tplv-k3u1fbpfcp-zoom-1.image)

# Skypack

`Skypack`本质上是一个`CDN`服务，但是和传统`CDN`服务有点不一样，传统的`CDN`只是给你提供一个文件的固定访问地址，你要使用哪个包，需要自己去这个包的发布文件中找到其中你要的那个文件。

早期大部分包提供的都是`IIFE`或者`commonjs`规范的模块，我们需要通过`link`或`script`标签引入，但是现在基本上所有的现代浏览器都原生支持`ES`模块，所以我们可以直接在浏览器上使用模块语法。如果使用传统的`CDN`服务，那么首先就需要某个包它提供了`ES`模块的文件，然后我们再从`CDN`里找到该`ES`版本的文件地址，再进行使用，如果某个包没有提供`ES`版本，那么我们就无法直接在浏览器上以模块的方式导入它，而`Skypack`是专门为现代浏览器设计的，它会自动帮我们进行转换，我们只要告诉它我们要导入的包名，即使这个包提供的是`commonjs`版本的文件，`Skypack`返回的也会是`ES`模块，所以我们就可以直接在浏览器上以模块的方式导入了。

## 基本使用

它的使用方式很简单：

```
https://cdn.skypack.dev/PACKAGE_NAME
```

只要拼接上你需要导入的包名即可，比如我们要导入`moment`：

```js
import moment from 'https://cdn.skypack.dev/moment';
console.log(moment().format());
```

如果要导入的包名有作用域，也只要把作用域带上就行，比如要导入`@wanglin1994/markjs`：

```js
import Markjs from "https://cdn.skypack.dev/@wanglin1994/markjs";
new Markjs();
```

## 指定版本

`Skypack`会根据我们提供的包名去`npm`上进行实时的查询，并返回包的最新版本，就像我们平时执行`npm install PACKAGE_NAME`一样，如果你需要导入指定的版本，那么也可以指定版本号，它遵循`semver`（`Semantic Version`（语义化版本））规范，你可以像下面这样导入指定的版本：

```
https://cdn.skypack.dev/react@16.13.1   // 匹配 react v16.13.1
https://cdn.skypack.dev/react@16      // 匹配 react 16.x.x 最新版本
https://cdn.skypack.dev/react@16.13    // 匹配 react 16.13.x 最新版本
https://cdn.skypack.dev/react@~16.13.0  // 匹配 react v16.13.x 最新版本
https://cdn.skypack.dev/react@^16.13.0  // 匹配 react v16.x.x  最新版本
```

## 指定导出包或指定导出文件

默认情况下，`Skypack`会返回包主入口点指定的文件，也就是`package.json`的`main`字段或`module`字段对应的文件，但是有时候这可能并不是我们需要的，以`vue@2`为例：

![图片名称](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32502d33bdbe463e82317972d9bdf6a8~tplv-k3u1fbpfcp-zoom-1.image)

可以看到页面输出是一片空白，这是为什么呢，让我们打开`vue2.6.14`版本的`npm`包，首先可以看到`dist`目录里提供了很多文件：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/513b5ce16af84e0ba54013d9437050a8~tplv-k3u1fbpfcp-zoom-1.image)

根据`package.json`可以看到它的主入口为：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16aea3de589c437e9f65413b0e12a101~tplv-k3u1fbpfcp-zoom-1.image)

指向的文件都只包含运行时，也就是不包含编译器，所以它没有在浏览器编译模板的能力，所以它就把`{{message}}`内容给忽略了，我们要导入的应该是`vue.esm.browser.js`或`vue.esm.browser.min.js`：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc83d97a573242459271956ba3f84317~tplv-k3u1fbpfcp-zoom-1.image)

`Skypack`也支持让我们导入指定的文件：

```js
import Vue from 'https://cdn.skypack.dev/vue@2.6.11/dist/vue.esm.browser.js'
```

在包名后面拼接上路径即可：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59b6237c667841eab1538a5840993518~tplv-k3u1fbpfcp-zoom-1.image)

以这种方式虽然可以加载到我们指定的文件，但是有一个很大的限制，就是如果要加载的文件不是`ES`模块，比如是`commonjs`模块，那么`Skypack`是不会自动对文件进行转换的，只有以按包名称(主入口)使用时才会进行处理。

## css文件

有些包不仅提供了`js`文件，还提供了`css`文件，常见于各种组件库，比如`element-ui`，示例如下：

```html
<div id="app">
    <div>{{title}}</div>
    <el-button type="success">成功按钮</el-button>
    <el-button type="primary" icon="el-icon-edit" circle></el-button>
    <el-input v-model="input" placeholder="请输入内容"></el-input>
</div>
```

```js
import Vue from 'vue@2.6.11/dist/vue.esm.browser.js'
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';

Vue.use(ElementUI);

new Vue({
  el: '#app',
  data() {
    return {
      title: 'Element UI',
      input: ''
    }
  }
})
```

我们直接在`js`里面导入`element-ui`的`css`文件，在我们平常的开发中这是很正常的，不过在浏览器上的运行结果如下：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd3d3975d5094a958d415e981856e69c~tplv-k3u1fbpfcp-zoom-1.image)

显然是无法在`ES`模块里直接导入`css`，所以我们需要把`css`通过传统样式的方式引入：

```css
@import 'element-ui/lib/theme-chalk/index.css'
```

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4d4622a8cf44816aa57e5b89a730c3b~tplv-k3u1fbpfcp-zoom-1.image)

## 固定url

以包名称进行导入虽然方便，但因为每次都是返回最新版本，所以很可能出现不兼容的问题，在实际生产环境中是需要导入特定版本的，`Skypack`会自动生成固定的`URL`：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/086ea72c39c04093993ad3f5b2fdde22~tplv-k3u1fbpfcp-zoom-1.image)

生产环境我们只要替换成图中划线的两个`URL`之一即可。

## 存在的问题

`Skypack`看起来很不错，然而理想是美好的，现实是残酷的。

首先第一个问题就是国内的网络访问`Skypack`的服务一言难尽，反正笔者使用时一会能请求到一会请求不到，非常不稳定。

第二个问题就是有些复杂的包可能会失败，比如`dayjs`、`vue`、`element-plus`等包的最新版本笔者尝试发现`Skypack`均编译失败了：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22505c1b74294a989fe5f333025003ee~tplv-k3u1fbpfcp-zoom-1.image)

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4825ce9b3c45479ab855443b60ba57a1~tplv-k3u1fbpfcp-zoom-1.image)

反正笔者目前使用下来发现失败概率还是很高的，你得不停的尝试不同的版本不同的文件，十分麻烦。

第三个问题笔者遇到的是`css`里面使用了在线字体，无法正常加载：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7233e601fb0b407a928e83b547582a07~tplv-k3u1fbpfcp-zoom-1.image)

鉴于以上问题，所以想用在实际生产环境中还是算了吧。

# 动手实现一个简单版

最后让我们用`nodejs`来实现一个超级简单版本的`Skypack`。

## 起个服务

创建一个新项目，在项目根目录新建一个`index.html`文件，用来测试`ES`模块，然后使用`Koa`搭建一个服务，安装：

```bash
npm i koa @koa/router koa-static
```

```js
const Koa = require("koa");
const Router = require("@koa/router");
const serve = require('koa-static');

// 创建应用
const app = new Koa();

// 静态文件服务
app.use(serve('.'));

// 路由
const router = new Router();
app.use(router.routes()).use(router.allowedMethods())

router.get("/(.*)", (ctx, next) => {
  ctx.body = ctx.url;
  next();
});

app.listen(3000);
console.log('服务启动成功！');
```

当我们访问`/index.html`即可访问`demo`页面：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6aabe73c1ba442938eaf9db47fc6e070~tplv-k3u1fbpfcp-zoom-1.image)

访问其他路径即可获取到访问的`url`：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9eecdaeb0fe3427a9e35dbe4f918acdf~tplv-k3u1fbpfcp-zoom-1.image)

## 下载npm包

先不考虑带作用域的包，我们暂且认为路径的第一段就是要下载的包名，然后我们使用`npm install`命令下载包（有其他更好的方式欢迎在评论区留言~）：

```js
const { execSync } = require('child_process');
const fs = require("fs");
const path = require("path");

router.get("/(.*)", async (ctx, next) => {
  let pkg = ctx.url.slice(1).split('/')[0];// 包名，比如vue@2.6
  let [pkgName] = pkg.split('@');// 去除版本号，获取纯包名
  if (pkgName) {
    try {
      // 该包没有安装过
      if (!checkIsInstall(pkgName)) {
        // 安装包
        execSync('npm i ' + pkg);
      }
    } catch (error) {
      ctx.throw(400, error.message);
    }
  }
  next();
});

// 检查某个包是否已安装过，暂不考虑版本问题
const checkIsInstall = (name) => {
  let dest = path.join("./node_modules/", name);
  try {
    fs.accessSync(dest, fs.constants.F_OK);
    return true;
  } catch (error) {
    return false;
  }
};
```

这样当我们访问`/moment`时如果没有安装这个包就会进行安装，已经安装了则直接跳过。

## 处理commonjs模块

我们可以读取下载的包的`package.json`文件，满足以下条件则代表是`commonjs`模块：

1.`type`字段不存在或者值为`commonjs`

2.不存在`module`字段

```js
const path = require("path");
const fs = require("fs");

router.get("/(.*)", async (ctx, next) => {
  let pkg = ctx.url.slice(1).split("/")[0];
  let [pkgName] = pkg.split("@");
  if (pkgName) {
    try {
      if (!checkIsInstall(pkgName)) {
        execSync("npm i " + pkg);
      }
      // 读取package.json
      let modulePkg = readPkg(pkgName);
      // 判断是否是commonjs模块
      let res = isCommonJs(modulePkg);
      ctx.body = '是否是commonjs模块：' + res;
    } catch (error) {
      ctx.throw(400, error.message);
    }
  }
  next();
});

// 读取指定模块的package.json文件
const readPkg = (name) => {
  return JSON.parse(fs.readFileSync(path.join('./node_modules/', name, 'package.json'), 'utf8'));
};

// 判断是否是commonjs模块
const isCommonJs = (pkg) => {
  return (!pkg.type || pkg.type === 'commonjs') && !pkg.module;
}
```

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/814ee76d5c114043ba7227370f485e53~tplv-k3u1fbpfcp-zoom-1.image)

`commonjs`模块显然是无法作为`ES`模块被加载的，所以需要先转换成`ES`模块，转换我们可以使用[esbuild](https://esbuild.github.io/)。

代码如下：

```bash
npm install esbuild
```

```js
const { transformSync } = require("esbuild");
router.get("/(.*)", async (ctx, next) => {
  let pkg = ctx.url.slice(1).split("/")[0];
  let [pkgName] = pkg.split("@");
  if (pkgName) {
    try {
      if (!checkIsInstall(pkgName)) {
        execSync("npm i " + pkg);
      }
      let modulePkg = readPkg(pkgName);
      let res = isCommonJs(modulePkg);
      // 是commonjs模块
      if (res) {
        ctx.type = 'text/javascript';
        // 转换成es模块
        ctx.body = commonjsToEsm(pkgName, modulePkg);
      }
    } catch (error) {
      ctx.throw(400, error.message);
    }
  }
  next();
});

// commonjs模块转换为esm
const commonjsToEsm = (name, pkg) => {
  let file = fs.readFileSync(path.join('./node_modules/', name, pkg.main), 'utf8');
  return transformSync(file, {
    format: 'esm'
  }).code;
}
```

`moment`未转换前的源码如下：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5d173c57520436ab0b46559b6dc5dcb~tplv-k3u1fbpfcp-zoom-1.image)

转换后如下：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2a84d25c54a4dafa8c3b18e6fb17eef~tplv-k3u1fbpfcp-zoom-1.image)

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f46db4831d4463782eeb1b54d832578~tplv-k3u1fbpfcp-zoom-1.image)

我们在`index.html`文件里测试一下，新增下面代码：

```html
<div id="app"></div>
<script type="module">
	import moment from '/moment';
	document.getElementById('app').innerHTML = moment().format('YYYY-MM-DD');
</script>
```

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4342c9cd3ee4bd29333f083905e4105~tplv-k3u1fbpfcp-zoom-1.image)

## 处理ES模块

`ES`模块会比较复杂一些，因为可能一个模块中又导入了另一个模块，首先我们来支持一下导入包中的指定文件，比如我们要导入`dayjs/esm/index.js`，当导入指定路径时我们就不进行`commonjs`检测了，直接默认为`ES`模块：

```js
router.get("/(.*)", async (ctx, next) => {
  let urlArr = ctx.url.slice(1).split("/");// 切割路径
  let pkg = urlArr[0]; // 包名
  let pkgPathArr = urlArr.slice(1); // 包中的路径
  let [pkgName] = pkg.split("@"); // 指定了版本号
  if (pkgName) {
    try {
      if (!checkIsInstall(pkgName)) {
        execSync("npm i " + pkg);
      }
      if (pkgPathArr.length <= 0) {
        let modulePkg = readPkg(pkgName);
        let res = isCommonJs(modulePkg);
        if (res) {
          ctx.type = "text/javascript";
          ctx.body = commonjsToEsm(pkgName, modulePkg);
        } else {
          // es模块
          ctx.type = "text/javascript";
          // 默认入口
          ctx.body = handleEsm(pkgName, [modulePkg.module || modulePkg.main]);
        }
      } else {
        // es模块
        ctx.type = "text/javascript";
        // 指定入口
        ctx.body = handleEsm(pkgName, pkgPathArr);
      }
    } catch (error) {
      ctx.throw(400, error.message);
    }
  }
  next();
});
```

我们知道当我们导入`js`文件时是可以省略文件后缀的，比如`import xxx from 'xxx/xxx'`，所以我们要检查是否省略了，省略了需要补上，`handleEsm`函数如下：

```js
// 处理es模块
const handleEsm = (name, paths) => {
  // 如果没有文件扩展名，则默认为`.js`后缀
  let last = paths[paths.length - 1];
  if (!/\.[^.]+$/.test(last)) {
    paths[paths.length - 1] = last + '.js';
  }
  let file = fs.readFileSync(
    path.join("./node_modules/", name, ...paths),
    "utf8"
  );
  return transformSync(file, {
    format: "esm",
  }).code;
};
```

`dayjs/esm/index.js`这个文件里面又引入了其他文件：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f14162120d6e4b969f77e75d74a6334b~tplv-k3u1fbpfcp-zoom-1.image)

每个`import`语句浏览器会发出一个对应的请求，让我们修改一下`index.html`进行测试：

```html
<script type="module">
	import dayjs from '/dayjs/esm/index.js';
	document.getElementById('app').innerHTML = dayjs().format('YYYY-MM-DD HH:mm:ss');
</script>
```

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2e4f01239dd4026b54e49bf388484f5~tplv-k3u1fbpfcp-zoom-1.image)

可以看到确实每个`import`语句都发出了一个对应的请求，页面运行结果如下：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f6216967d214381a0496274c2f44cd1~tplv-k3u1fbpfcp-zoom-1.image)

写到这里你可能会发现其实无需判断是否是`commonjs`模块，都交给`esbuild`处理就行了，让我们精简一下代码：

```js
router.get("/(.*)", async (ctx, next) => {
  let urlArr = ctx.url.slice(1).split("/");
  let pkg = urlArr[0];
  let pkgPathArr = urlArr.slice(1);
  let [pkgName] = pkg.split("@");
  if (pkgName) {
    try {
      if (!checkIsInstall(pkgName)) {
        execSync("npm i " + pkg);
      }
      let modulePkg = readPkg(pkgName);
      ctx.type = "text/javascript";
      ctx.body = handleEsm(pkgName, pkgPathArr.length <= 0 ? [modulePkg.module || modulePkg.main] : pkgPathArr);
    } catch (error) {
      ctx.throw(400, error.message);
    }
  }
  next();
});
```

## 打包到一个文件里

以`axios`的入口文件为例：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2637af8eeb364be7b40bc72f7a4c0074~tplv-k3u1fbpfcp-zoom-1.image)

使用`esbuild`的`transformSync`方法编译后的结果为：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5180efc91045459b9d8286a2cf780383~tplv-k3u1fbpfcp-zoom-1.image)

可以看到`require`方法还是存在，并没有把`require`的内容都打包进来，这样的`es`模块是无法使用的，如果需要把依赖都打包到一个文件内我们就不能使用`transformSync`方法了，需要使用`buildSync`，这个方法执行的是文件的编译，就是输入输出都是文件的形式。

```js
const { buildSync } = require("esbuild");
// 处理es模块
const handleEsm = (name, paths) => {
  const outfile = path.join("./node_modules/", name, "esbuild_output.js");
  // 检查是否已经编译过了
  if (checkIsExist(outfile)) {
    return fs.readFileSync(outfile, "utf8");
  }
  // 如果没有文件扩展名，则默认为`.js`后缀
  let last = paths[paths.length - 1];
  if (!/\.[^.]+$/.test(last)) {
    paths[paths.length - 1] = last + ".js";
  }
  // 编译文件
  buildSync({
    entryPoints: [path.join("./node_modules/", name, ...paths)],// 输入
    format: "esm",
    bundle: true,
    outfile,// 输出
  });
  return fs.readFileSync(outfile, "utf8");
};

// 检查某个文件是否存在
const checkIsExist = (file) => {
  try {
    fs.accessSync(file, fs.constants.F_OK);
    return true;
  } catch (error) {
    return false;
  }
};
```

再让我们`axios`编译后的结果：

![【粘贴图片】](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17ca8d073abf45d3919ec7b7bf18c5b2~tplv-k3u1fbpfcp-zoom-1.image)

# 总结

本文介绍了一下`Skypack`的使用，以及写了一个简单版的`ES`模块`CDN`服务，如果你用过`vitejs`，就会发现这就是它所做的事情之一，当然`vite`的实现要复杂的多。

`demo`的源代码地址[https://github.com/wanglin2/ES_Modules_CDN](https://github.com/wanglin2/ES_Modules_CDN)。
