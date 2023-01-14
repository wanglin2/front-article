[Vite](https://vitejs.dev/)是什么就不用笔者多说了，用过`Vue`的朋友肯定都知道，本文会通过手写一个非常简单的乞丐版`Vite`来了解一下`Vite`的基本实现原理，参考的是`Vite`最早的版本（`vite-1.0.0-rc.5`版本，`Vue`版本为`3.0.0-rc.10`）实现的，现在已经是`3.x`的版本了，为什么不直接参考最新的版本呢，因为一上来就看这种比较完善的工具源码比较难看懂，反正笔者不行，所以我们可以先从最早的版本来窥探一下原理，能力强的朋友可以忽略~

本文会分为上下两篇，上篇主要讨论如何成功运行项目，下篇主要讨论热更新。



# 前端测试项目

前端测试项目结构如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a890cdd58dc4fe28947659a2430173b~tplv-k3u1fbpfcp-zoom-1.image)


`Vue`组件使用的是`Options Api` ，不涉及到`css`预处理语言、`ts`等`js`语言，所以是一个非常简单的项目，我们的目标很简单，就是要写一个`Vite`服务让这个项目能运行起来！



# 搭建基本服务

`vite`服务的基本结构如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11323f0f5c3c4871baea17b937ba6688~tplv-k3u1fbpfcp-zoom-1.image)


首先让我们来起个服务，`HTTP`应用框架我们使用[connect](https://github.com/senchalabs/connect)：

```js
// app.js
const connect = require("connect");
const http = require("http");

const app = connect();

app.use(function (req, res) {
  res.end("Hello from Connect!\n");
});

http.createServer(app).listen(3000);
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bbeadc2de53498ab3e4c34d63aade18~tplv-k3u1fbpfcp-zoom-1.image)


接下来我们需要做的就是拦截各种类型的请求来进行不同的处理。



# 拦截html

项目访问的入口地址是`http://localhost:3000/index.html`，所以接到的第一个请求就是`html`文件的请求，我们暂时直接返回`html`文件的内容即可：

```js
// app.js
const path = require("path");
const fs = require("fs");

const basePath = path.join("../test/");
const typeAlias = {
  js: "application/javascript",
  css: "text/css",
  html: "text/html",
  json: "application/json",
};

app.use(function (req, res) {
  // 提供html页面
  if (req.url === "/index.html") {
    let html = fs.readFileSync(path.join(basePath, "index.html"), "utf-8");
    res.setHeader("Content-Type", typeAlias.html);
    res.statusCode = 200;
    res.end(html);
  } else {
    res.end('')
  }
});
```

现在访问页面肯定还是一片空白，因为页面发起的`main.js`的请求我们还没有处理，`main.js`的内容如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e54ad5dc8e594f8fbbfed0390d96d131~tplv-k3u1fbpfcp-zoom-1.image)



# 拦截js请求

`main.js`请求需要做一点处理，因为浏览器是不支持裸导入的，所以我们要转换一下裸导入的语句，将`import xxx from 'xxx'`转换为`import xxx from '/@module/xxx'`，然后再拦截`/@module`请求，从`node_modules`里获取要导入的模块进行返回。

解析导入语句我们使用[es-module-lexer](https://github.com/guybedford/es-module-lexer)：

```js
// app.js
const { init, parse: parseEsModule } = require("es-module-lexer");

app.use(async function (req, res) {
    if (/\.js\??[^.]*$/.test(req.url)) {
        // js请求
        let js = fs.readFileSync(path.join(basePath, req.url), "utf-8");
        await init;
        let parseResult = parseEsModule(js);
        // ...
    }
});
```

解析的结果为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8bc224a9040d42c198e58271352060f5~tplv-k3u1fbpfcp-zoom-1.image)


解析结果为一个数组，第一项也是个数组代表导入的数据，第二项代表导出，`main.js`没有，所以是空的。`s`、`e`代表导入来源的起止位置，`ss`、`se`代表整个导入语句的起止位置。

接下来我们检查当导入来源不是`.`或`/`开头的就转换为`/@module/xxx`的形式：

```js
// app.js
const MagicString = require("magic-string");

app.use(async function (req, res) {
    if (/\.js\??[^.]*$/.test(req.url)) {
        // js请求
        let js = fs.readFileSync(path.join(basePath, req.url), "utf-8");
        await init;
        let parseResult = parseEsModule(js);
        let s = new MagicString(js);
        // 遍历导入语句
        parseResult[0].forEach((item) => {
            // 不是裸导入则替换
            if (item.n[0] !== "." && item.n[0] !== "/") {
                s.overwrite(item.s, item.e, `/@module/${item.n}`);
            }
        });
        res.setHeader("Content-Type", typeAlias.js);
        res.statusCode = 200;
        res.end(s.toString());
    }
});
```

修改`js`字符串我们使用了[magic-string](https://github.com/Rich-Harris/magic-string)，从这个简单的示例上你应该能发现它的魔法之处，就是即使字符串已经变了，但使用原始字符串计算出来的索引修改它也还是正确的，因为索引还是相对于原始字符串。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9327ec9c174141b6a4c1a8636ca14d61~tplv-k3u1fbpfcp-zoom-1.image)


可以看到`vue`已经成功被修改成`/@module/vue`了。

紧接着我们需要拦截一下`/@module`请求：

```js
// app.js
const { buildSync } = require("esbuild");

app.use(async function (req, res) {
    if (/^\/@module\//.test(req.url)) {
        // 拦截/@module请求
        let pkg = req.url.slice(9);
        // 获取该模块的package.json
        let pkgJson = JSON.parse(
            fs.readFileSync(
                path.join(basePath, "node_modules", pkg, "package.json"),
                "utf8"
            )
        );
        // 找出该模块的入口文件
        let entry = pkgJson.module || pkgJson.main;
        // 使用esbuild编译
        let outfile = path.join(`./esbuild/${pkg}.js`);
        buildSync({
            entryPoints: [path.join(basePath, "node_modules", pkg, entry)],
            format: "esm",
            bundle: true,
            outfile,
        });
        let js = fs.readFileSync(outfile, "utf8");
        res.setHeader("Content-Type", typeAlias.js);
        res.statusCode = 200;
        res.end(js);
    }
})
```

我们先获取了包的`package.json`文件，目的是找出它的入口文件，然后读取并使用[esbuild](https://esbuild.github.io/)进行转换，当然`Vue`是有`ES模块`的产物的，但是可能有的包没有，所以直接就统一处理了。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/884b1f7a7b864d66a71cc1c9f3fbd779~tplv-k3u1fbpfcp-zoom-1.image)



# 拦截css请求

`css`请求有两种，一种来源于`link`标签，一种来源于`import`方式，`link`标签的`css`请求我们直接返回`css`即可，但是`import`的`css`直接返回是不行的，`ES模块`只支持`js`，所以我们需要转成`js`类型，主要逻辑就是手动把`css`插入页面，所以这两种请求我们需要分开处理。

为了能区分`import`请求，我们修改一下前面拦截`js`的代码，把每个导入来源都加上`?import`查询参数：

```js
// ...
	// 遍历导入语句
    parseResult[0].forEach((item) => {
      // 不是裸导入则替换
      if (item.n[0] !== "." && item.n[0] !== "/") {
        s.overwrite(item.s, item.e, `/@module/${item.n}?import`);
      } else {
        s.overwrite(item.s, item.e, `${item.n}?import`);
      }
    });
//...
```

拦截`/@module`的地方也别忘了修改：

```js
// ...
let pkg = removeQuery(req.url.slice(9));// 从/@module/vue?import中解析出vue
// ...

// 去除url的查询参数
const removeQuery = (url) => {
  return url.split("?")[0];
};
```

这样`import`的请求就都会带上一个标志：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e4bcd5f9f054d2d8b9316a24e29f8e8~tplv-k3u1fbpfcp-zoom-1.image)


然后根据这个标志来分别处理`css`请求：

```js
// app.js

app.use(async function (req, res) {
    if (/\.css\??[^.]*$/.test(req.url)) {
        // 拦截css请求
        let cssRes = fs.readFileSync(
            path.join(basePath, req.url.split("?")[0]),
            "utf-8"
        );
        if (checkQueryExist(req.url, "import")) {
            // import请求，返回js文件
            cssRes = `
                const insertStyle = (css) => {
                    let el = document.createElement('style')
                    el.setAttribute('type', 'text/css')
                    el.innerHTML = css
                    document.head.appendChild(el)
                }
                insertStyle(\`${cssRes}\`)
                export default insertStyle
            `;
            res.setHeader("Content-Type", typeAlias.js);
        } else {
            // link请求，返回css文件
            res.setHeader("Content-Type", typeAlias.css);
        }
        res.statusCode = 200;
        res.end(cssRes);
    }
})

// 判断url的某个query名是否存在
const checkQueryExist = (url, key) => {
  return new URL(path.resolve(basePath, url)).searchParams.has(key);
};
```

如果是`import`导入的`css`那么就把它转换为`js`类型的响应，然后提供一个创建`style`标签并插入到页面的方法，并且立即执行，那么这个`css`就会被插入到页面中，一般这个方法会被提前注入页面。

如果是`link`标签的`css`请求直接返回`css`即可。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/640157e842ad4ada951db86a34e29ffe~tplv-k3u1fbpfcp-zoom-1.image)



# 拦截vue请求

最后，就是处理`Vue`单文件的请求了，这个会稍微复杂一点，处理`Vue`单文件我们使用`@vue/compiler-sfc`的`3.0.0-rc.10`版本，首先需要把`Vue`单文件的`template`、`js`、`style`三部分解析出来：

```js
// app.js
const { parse: parseVue } = require("@vue/compiler-sfc");

app.use(async function (req, res) {
    if (/\.vue\??[^.]*$/.test(req.url)) {
    // Vue单文件
    let vue = fs.readFileSync(
      path.join(basePath, removeQuery(req.url)),
      "utf-8"
    );
    let { descriptor } = parseVue(vue);
  }
})
```

然后再分别解析三部分，`template`和`css`部分会转换成一个`import`请求。

## 处理`js`部分

```js
// ...
const { compileScript, rewriteDefault } = require("@vue/compiler-sfc");

let code = "";
// 处理js部分
let script = compileScript(descriptor);
if (script) {
    code += rewriteDefault(script.content, "__script");
}
```

`rewriteDefault`方法用于将`export default`转换为一个新的变量定义，这样我们可以注入更多数据，比如：

```js
// 转换前
let js = `
    export default {
        data() {
            return {}
        }
    }
`

// 转换后
let js = `
    const __script = {
        data() {
            return {}
        }
    }
`

//然后可以给__script添加更多属性，最后再手动添加到导出即可
js += `\n__script.xxx = xxx`
js += `\nexport default __script`
```

## 处理`template`部分

```js
// ...
// 处理模板
if (descriptor.template) {
    let templateRequest = removeQuery(req.url) + `?type=template`;
    code += `\nimport { render as __render } from ${JSON.stringify(
        templateRequest
    )}`;
    code += `\n__script.render = __render`;
}
```

将模板转换成了一个`import`语句，然后获取导入的`render`函数挂载到`__script`上，后面我们会拦截这个`type=template`的请求，返回模板的编译结果。

## 处理`style`部分

```js
// ...
// 处理样式
if (descriptor.styles) {
    descriptor.styles.forEach((s, i) => {
        const styleRequest = removeQuery(req.url) + `?type=style&index=${i}`;
        code += `\nimport ${JSON.stringify(styleRequest)}`
    })
}
```

和模板一样，样式也转换成了一个单独的请求。

最后导出`__script`并返回数据：

```js
// ...
// 导出
code += `\nexport default __script`;
res.setHeader("Content-Type", typeAlias.js);
res.statusCode = 200;
res.end(code);
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ebc7d11cd3a4cb29c461c1a9872a4eb~tplv-k3u1fbpfcp-zoom-1.image)


可以看到`__script`其实就是一个`Vue`的组件选项对象，模板部分编译的结果就是组件的渲染函数`render`，相当于把`js`和模板部分组合成一个完整的组件选项对象。

## 处理模板请求

当`Vue`单文件的请求`url`存在`type=template`参数，我们就编译一下模板然后返回：

```js
// app.js
const { compileTemplate } = require("@vue/compiler-sfc");

app.use(async function (req, res) {
    if (/\.vue\??[^.]*$/.test(req.url)) {
        // vue单文件
        // 处理模板请求
        if (getQuery(req.url, "type") === "template") {
            // 编译模板为渲染函数
            code = compileTemplate({
                source: descriptor.template.content,
            }).code;
            res.setHeader("Content-Type", typeAlias.js);
            res.statusCode = 200;
            res.end(code);
            return;
        }
        // ...
    }
})

// 获取url的某个query值
const getQuery = (url, key) => {
  return new URL(path.resolve(basePath, url)).searchParams.get(key);
};
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d5536cbd9d44eee9f867a9a7a64a607~tplv-k3u1fbpfcp-zoom-1.image)


## 处理样式请求

样式和前面我们拦截样式请求一样，也需要转换成`js`然后手动插入到页面：

```js
// app.js
const { compileTemplate } = require("@vue/compiler-sfc");

app.use(async function (req, res) {
    if (/\.vue\??[^.]*$/.test(req.url)) {
        // vue单文件
    }
    // 处理样式请求
    if (getQuery(req.url, "type") === "style") {
        // 获取样式块索引
        let index = getQuery(req.url, "index");
        let styleContent = descriptor.styles[index].content;
        code = `
            const insertStyle = (css) => {
                let el = document.createElement('style')
                el.setAttribute('type', 'text/css')
                el.innerHTML = css
                document.head.appendChild(el)
            }
            insertStyle(\`${styleContent}\`)
            export default insertStyle
        `;
        res.setHeader("Content-Type", typeAlias.js);
        res.statusCode = 200;
        res.end(code);
        return;
    }
})
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e02a72b5396a4f379a7bfd9aceff5558~tplv-k3u1fbpfcp-zoom-1.image)


样式转换为`js`的这个逻辑因为有两个地方用到了，所以我们可以提取成一个函数：

```js
// app.js
// css to js
const cssToJs = (css) => {
  return `
    const insertStyle = (css) => {
        let el = document.createElement('style')
        el.setAttribute('type', 'text/css')
        el.innerHTML = css
        document.head.appendChild(el)
    }
    insertStyle(\`${css}\`)
    export default insertStyle
  `;
};
```

## 修复单文件的裸导入问题

单文件内的`js`部分也可以导入模块，所以也会存在裸导入的问题，前面介绍了裸导入的处理方法，那就是先替换导入来源，所以单文件的`js`部分解析出来以后我们也需要进行一个替换操作，我们先把替换的逻辑提取成一个公共方法：

```js
// 处理裸导入
const parseBareImport = async (js) => {
  await init;
  let parseResult = parseEsModule(js);
  let s = new MagicString(js);
  // 遍历导入语句
  parseResult[0].forEach((item) => {
    // 不是裸导入则替换
    if (item.n[0] !== "." && item.n[0] !== "/") {
      s.overwrite(item.s, item.e, `/@module/${item.n}?import`);
    } else {
      s.overwrite(item.s, item.e, `${item.n}?import`);
    }
  });
  return s.toString();
};
```

然后编译完`js`部分后立即处理一下：

```js
// 处理js部分
let script = compileScript(descriptor);
if (script) {
    let scriptContent = await parseBareImport(script.content);// ++
    code += rewriteDefault(scriptContent, "__script");
}
```

另外，编译后的模板部分代码也会存在一个裸导入`Vue`，也需要处理一下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea5ed17946994f6eb2cdbb87a6bcbefd~tplv-k3u1fbpfcp-zoom-1.image)


```js
// 处理模板请求
if (
    new URL(path.resolve(basePath, req.url)).searchParams.get("type") ===
    "template"
) {
    code = compileTemplate({
        source: descriptor.template.content,
    }).code;
    code = await parseBareImport(code);// ++
    res.setHeader("Content-Type", typeAlias.js);
    res.statusCode = 200;
    res.end(code);
    return;
}
```

# 处理静态文件

`App.vue`里面引入了两张图片：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc9c5a55ab0e47a699215adfa485658f~tplv-k3u1fbpfcp-zoom-1.image)


编译后的结果为：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fee4c96f6f74134ba8b099f91ffcb47~tplv-k3u1fbpfcp-zoom-1.image)


`ES模块`只能导入`js`文件，所以静态文件的导入，响应结果也需要是`js`：

```js
// vite/app.js
app.use(async function (req, res) {
    if (isStaticAsset(req.url) && checkQueryExist(req.url, "import")) {
        // import导入的静态文件
        res.setHeader("Content-Type", typeAlias.js);
        res.statusCode = 200;
        res.end(`export default ${JSON.stringify(removeQuery(req.url))}`);
    }
})

// 检查是否是静态文件
const imageRE = /\.(png|jpe?g|gif|svg|ico|webp)(\?.*)?$/;
const mediaRE = /\.(mp4|webm|ogg|mp3|wav|flac|aac)(\?.*)?$/;
const fontsRE = /\.(woff2?|eot|ttf|otf)(\?.*)?$/i;
const isStaticAsset = (file) => {
  return imageRE.test(file) || mediaRE.test(file) || fontsRE.test(file);
};
```

`import`导入的静态文件处理很简单，直接把静态文件的`url`字符串作为默认导出即可。

这样我们又会收到两个静态文件的请求：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74803632893a4601ae3bad8e1c5945ea~tplv-k3u1fbpfcp-zoom-1.image)


简单起见，没有匹配到以上任何规则的我们都认为是静态文件，使用[serve-static](https://github.com/expressjs/serve-static)来提供静态文件服务即可：

```js
// vite/app.js
const serveStatic = require("serve-static");

app.use(async function (req, res, next) {
    if (xxx) {
        // xxx
    } else if (xxx) {
        // xxx
        // ...
    } else {
        next();// ++
    }
})

// 静态文件服务
app.use(serveStatic(path.join(basePath, "public")));
app.use(serveStatic(path.join(basePath)));
```

静态文件服务的中间件放到最后，这样没有匹配到的路由就会走到这里，到这一步效果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3302ddcd08f4ef195593db953971da7~tplv-k3u1fbpfcp-zoom-1.image)


可以看到页面已经被加载出来。

下一篇我们会介绍一下热更新的实现，See you later~
