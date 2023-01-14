上一篇[Vite入门从手写一个乞丐版的Vite开始（上）](https://juejin.cn/post/7142878515380092959)我们已经成功的将页面渲染出来了，这一篇我们来简单的实现一下热更新的功能。

所谓热更新就是修改了文件，不用刷新页面，页面的某个部分就自动更新了，听着似乎挺简单的，但是要实现一个很完善的热更新还是很复杂的，要考虑的情况很多，所以本文只会实现一个最基础的热更新效果。



# 创建WebSocket连接

浏览器显然是不知道文件有没有修改的，所以需要后端进行推送，我们先来建立一个`WebSocket`连接。

```js
// app.js
const server = http.createServer(app);
const WebSocket = require("ws");

// 创建WebSocket服务
const createWebSocket = () => {
    // 创建一个服务实例
    const wss = new WebSocket.Server({ noServer: true });// 不用额外创建http服务，直接使用我们自己创建的http服务

    // 接收到http的协议升级请求
    server.on("upgrade", (req, socket, head) => {
        // 当子协议为vite-hmr时就处理http的升级请求
        if (req.headers["sec-websocket-protocol"] === "vite-hmr") {
            wss.handleUpgrade(req, socket, head, (ws) => {
                wss.emit("connection", ws, req);
            });
        }
    });

    // 连接成功
    wss.on("connection", (socket) => {
        socket.send(JSON.stringify({ type: "connected" }));
    });

    // 发送消息方法
    const sendMsg = (payload) => {
        const stringified = JSON.stringify(payload, null, 2);

        wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(stringified);
            }
        });
    };

    return {
        wss,
        sendMsg,
    };
};
const { wss, sendMsg } = createWebSocket();

server.listen(3000);
```

`WebSocket`和我们的服务共用一个`http`请求，当接收到`http`协议的升级请求后，判断子协议是否是`vite-hmr`，是的话我们就把创建的`WebSocket`实例连接上去，这个子协议是自己定义的，通过设置子协议，单个服务器可以实现多个`WebSocket` 连接，就可以根据不同的协议处理不同类型的事情，服务端的`WebSocket`创建完成以后，客户端也需要创建，但是客户端是不会有这些代码的，所以需要我们手动注入，创建一个文件`client.js`：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5000fa3c9964297ad9850d00b308a61~tplv-k3u1fbpfcp-zoom-1.image)


```js
// client.js

// vite-hmr代表自定义的协议字符串
const socket = new WebSocket("ws://localhost:3000/", "vite-hmr");

socket.addEventListener("message", async ({ data }) => {
  const payload = JSON.parse(data);
});
```

接下来我们把这个`client.js`注入到`html`文件，修改之前`html`文件拦截的逻辑：

```js
// app.js
const clientPublicPath = "/client.js";

app.use(async function (req, res, next) {
    // 提供html页面
    if (req.url === "/index.html") {
        let html = readFile("index.html");
        const devInjectionCode = `\n<script type="module">import "${clientPublicPath}"</script>\n`;
        html = html.replace(/<head>/, `$&${devInjectionCode}`);
        send(res, html, "html");
    }
})
```

通过`import`的方式引入，所以我们需要拦截一下这个请求：

```js
// app.js
app.use(async function (req, res, next) {
    if (req.url === clientPublicPath) {
        // 提供client.js
        let js = fs.readFileSync(path.join(__dirname, "./client.js"), "utf-8");
        send(res, js, "js");
    }
})
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7833bc946d8f4d47a100b2c5328669a9~tplv-k3u1fbpfcp-zoom-1.image)


可以看到已经连接成功。



# 监听文件改变

接下来我们要初始化一下对文件修改的监听，监听文件的改变使用[chokidar](https://github.com/paulmillr/chokidar)：

```js
// app.js
const chokidar = require(chokidar);

// 创建文件监听服务
const createFileWatcher = () => {
  const watcher = chokidar.watch(basePath, {
    ignored: [/node_modules/, /\.git/],
    awaitWriteFinish: {
      stabilityThreshold: 100,
      pollInterval: 10,
    },
  });
  return watcher;
};
const watcher = createFileWatcher();

watcher.on("change", (file) => {
    // file文件修改了
})
```



# 构建导入依赖图

为什么要构建依赖图呢，很简单，比如一个模块改变了，仅仅更新它自己肯定还不够，依赖它的模块都需要修改才对，要做到这一点自然要能知道哪些模块依赖它才行。

```js
// app.js
const importerMap = new Map();
const importeeMap = new Map();

// map ： key -> set
// map ： 模块 -> 依赖该模块的模块集合
const ensureMapEntry = (map, key) => {
  let entry = map.get(key);
  if (!entry) {
    entry = new Set();
    map.set(key, entry);
  }
  return entry;
};
```

需要用到的变量和函数就是上面几个，`importerMap`用来存放`模块`到`依赖它的模块`之间的映射；`importeeMap`用来存放`模块`到`该模块所依赖的模块`的映射，主要作用是用来删除不再依赖的模块，比如`a`一开始依赖`b`和`c`，此时`importerMap`里面存在`b -> a`和`c -> a`的映射关系，然后我修改了一下`a`，删除了对`c`的依赖，那么就需要从`importerMap`里面也同时删除`c -> a`的映射关系，这时就可以通过`importeeMap`来获取到之前的`a -> [b, c]`的依赖关系，跟此次的依赖关系`a -> [b]`进行比对，就可以找出不再依赖的`c`模块，然后在`importerMap`里删除`c -> a`的依赖关系。

接下来我们从`index.html`页面开始构建依赖图，`index.html`内容如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1092a4df161741bdac624c443c63219d~tplv-k3u1fbpfcp-zoom-1.image)


可以看到它依赖了`main.js`，修改拦截`html`的方法：

```js
// app.js
app.use(async function (req, res, next) {
    // 提供html页面
    if (req.url === "/index.html") {
        let html = readFile("index.html");
        // 查找模块依赖图
        const scriptRE = /(<script\b[^>]*>)([\s\S]*?)<\/script>/gm;
        const srcRE = /\bsrc=(?:"([^"]+)"|'([^']+)'|([^'"\s]+)\b)/;
        // 找出script标签
        html = html.replace(scriptRE, (matched, openTag) => {
            const srcAttr = openTag.match(srcRE);
            if (srcAttr) {
                // 创建script到html的依赖关系
                const importee = removeQuery(srcAttr[1] || srcAttr[2]);
                ensureMapEntry(importerMap, importee).add(removeQuery(req.url));
            }
            return matched;
        });
        // 注入client.js
        // ...
    }
})
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84902309baa44981a66df0d0d1f41f79~tplv-k3u1fbpfcp-zoom-1.image)


接下来我们需要分别修改`js`的拦截方法，注册依赖关系；修改`Vue`单文件的拦截方法，注册`js`部分的依赖关系，因为上一篇文章里我们已经把转换裸导入的逻辑都提取成一个公共函数`parseBareImport`了，所以我们只要修改这个函数就可以了：

```js
// 处理裸导入
// 增加了importer入参，req.url
const parseBareImport = async (js, importer) => {
    await init;
    let parseResult = parseEsModule(js);
    let s = new MagicString(js);
    importer = removeQuery(importer);// ++
    parseResult[0].forEach((item) => {
        let url = "";
        if (item.n[0] !== "." && item.n[0] !== "/") {
            url = `/@module/${item.n}?import`;
        } else {
            url = `${item.n}?import`;
        }
        s.overwrite(item.s, item.e, url);
        // 注册importer模块所以依赖的模块到它的映射关系
        ensureMapEntry(importerMap, removeQuery(url)).add(importer);// ++
    });
    return s.toString();
};
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f1b93e25e1840c38129ce935b14a9a7~tplv-k3u1fbpfcp-zoom-1.image)


再来增加一下前面提到的去除不再依赖的关系的逻辑：

```js
// 处理裸导入
const parseBareImport = async (js, importer) => {
    // ...
    importer = removeQuery(importer);
    // 上一次的依赖集合
    const prevImportees = importeeMap.get(importer);// ++
    // 这一次的依赖集合
    const currentImportees = new Set();// ++
    importeeMap.set(importer, currentImportees);// ++
    parseResult[0].forEach((item) => {
        // ...
        let importee = removeQuery(url);// ++
        // url -> 依赖
        currentImportees.add(importee);// ++
        // 依赖 -> url
        ensureMapEntry(importerMap, importee).add(importer);
    });
    // 删除不再依赖的关系++
    if (prevImportees) {
        prevImportees.forEach((importee) => {
            if (!currentImportees.has(importee)) {
                // importer不再依赖importee，所以要从importee的依赖集合中删除importer
                const importers = importerMap.get(importee);
                if (importers) {
                    importers.delete(importer);
                }
            }
        });
    }
    return s.toString();
};
```



# Vue单文件的热更新

先来实现一下`Vue`单文件的热更新，先监听一下`Vue`单文件的改变事件：

```js
// app.js
// 监听文件改变
watcher.on("change", (file) => {
  if (file.endsWith(".vue")) {
    handleVueReload(file);
  }
});
```

如果修改的文件是以`.vue`结尾，那么就进行处理，怎么处理呢，`Vue`单文件会解析成`js`、`template`、`style`三部分，我们把解析数据缓存起来，当文件修改了以后会再次进行解析，然后分别和上一次的解析结果进行比较，判断单文件的哪部分发生变化了，最后给浏览器发送不同的事件，由前端页面来进行不同的处理，缓存我们使用[lru-cache](https://github.com/isaacs/node-lru-cache)：

```js
// app.js
const LRUCache = require("lru-cache");

// 缓存Vue单文件的解析结果
const vueCache = new LRUCache({
  max: 65535,
});
```

然后修改一下`Vue`单文件的拦截方法，增加缓存：

```js
// app.js
app.use(async function (req, res, next) {
    if (/\.vue\??[^.]*$/.test(req.url)) {
        // ...
        // vue单文件
        let descriptor = null;
        // 如果存在缓存则直接使用缓存
        let cached = vueCache.get(removeQuery(req.url));
        if (cached) {
            descriptor = cached;
        } else {
            // 否则进行解析，并且将解析结果进行缓存
            descriptor = parseVue(vue).descriptor;
            vueCache.set(removeQuery(req.url), descriptor);
        }
        // ...
    }
})
```

然后就来到`handleVueReload`方法了：

```js
// 处理Vue单文件的热更新
const handleVueReload = (file) => {
  file = filePathToUrl(file);
};

// 处理文件路径到url
const filePathToUrl = (file) => {
  return file.replace(/\\/g, "/").replace(/^\.\.\/test/g, "");
};
```

我们先转换了一下文件路径，因为监听到的是本地路径，和请求的`url`是不一样的：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/227511f6e7b541f592a04ca6596291d5~tplv-k3u1fbpfcp-zoom-1.image)


```js
const handleVueReload = (file) => {
  file = filePathToUrl(file);
  // 获取上一次的解析结果
  const prevDescriptor = vueCache.get(file);
  // 从缓存中删除上一次的解析结果
  vueCache.del(file);
  if (!prevDescriptor) {
    return;
  }
  // 解析
  let vue = readFile(file);
  descriptor = parseVue(vue).descriptor;
  vueCache.set(file, descriptor);
};
```

接着获取了一下缓存数据，然后进行了这一次的解析，并更新缓存，接下来就要判断哪一部分发生了改变。

## 热更新template

我们先来看一下比较简单的模板热更新：

```js
const handleVueReload = (file) => {
    // ...
    // 检查哪部分发生了改变
    const sendRerender = () => {
        sendMsg({
            type: "vue-rerender",
            path: file,
        });
    };
    // template改变了发送rerender事件
    if (!isEqualBlock(descriptor.template, prevDescriptor.template)) {
        return sendRerender();
    }
}

// 判断Vue单文件解析后的两个部分是否相同
function isEqualBlock(a, b) {
    if (!a && !b) return true;
    if (!a || !b) return false;
    if (a.src && b.src && a.src === b.src) return true;
    if (a.content !== b.content) return false;
    const keysA = Object.keys(a.attrs);
    const keysB = Object.keys(b.attrs);
    if (keysA.length !== keysB.length) {
        return false;
    }
    return keysA.every((key) => a.attrs[key] === b.attrs[key]);
}
```

逻辑很简单，当`template`部分发生改变后向浏览器发送一个`rerender`事件，带上修改模块的`url`。

现在我们来修改一下`HelloWorld.vue`的`template`看看：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/492ec5b429fe47d5bcc64a0bed3b00fa~tplv-k3u1fbpfcp-zoom-1.image)


可以看到已经成功收到了消息。

接下来需要修改一下`client.js`文件，增加收到`vue-rerender`消息后的处理逻辑。

文件更新了，浏览器肯定需要请求一下更新的文件，`Vite`使用的是`import()`方法，但是这个方法`js`本身是没有的，另外笔者没有找到是哪里注入的，所以加载模块的逻辑只能自己来简单实现一下：

```js
// client.js
// 回调id
let callbackId = 0;
// 记录回调
const callbackMap = new Map();
// 模块导入后调用的全局方法
window.onModuleCallback = (id, module) => {
  document.body.removeChild(document.getElementById("moduleLoad"));
  // 执行回调
  let callback = callbackMap.get(id);
  if (callback) {
    callback(module);
  }
};

// 加载模块
const loadModule = ({ url, callback }) => {
  // 保存回调
  let id = callbackId++;
  callbackMap.set(id, callback);
  // 创建一个模块类型的script
  let script = document.createElement("script");
  script.type = "module";
  script.id = "moduleLoad";
  script.innerHTML = `
        import * as module from '${url}'
        window.onModuleCallback(${id}, module)
    `;
  document.body.appendChild(script);
};
```

因为要加载的都是`ES`模块，直接请求是不行的，所以创建一个`type`为`module`的`script`标签，来让浏览器加载，这样请求都不用自己发，只要把想办法获取到模块的导出就行了，这个也很简单，创建一个全局函数即可，这个很像`jsonp`的原理。

接下来就可以处理`vue-rerender`消息了：

```js
// app.js
socket.addEventListener("message", async ({ data }) => {
    const payload = JSON.parse(data);
    handleMessage(payload);
});

const handleMessage = (payload) => {
    switch (payload.type) {
        case "vue-rerender":
            loadModule({
                url: payload.path + "?type=template&t=" + Date.now(),
                callback: (module) => {
                    window.__VUE_HMR_RUNTIME__.rerender(payload.path, module.render);
                },
            });
            break;
    }
};
```

就这么简单，我们来修改一下`HelloWorld.vue`文件的模板来看看：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/922552ef594d4a95b6037e57414d780c~tplv-k3u1fbpfcp-zoom-1.image)


可以看到没有刷新页面，但是更新了，接下来详细解释一下原理。

因为我们修改的是模板部分，所以请求的`url`为`payload.path + "?type=template`，这个源于上一篇文章里我们请求`Vue`单文件的模板部分是这么设计的，为什么要加个时间戳呢，因为不加的话浏览器认为这个模块已经加载过了，是不会重新请求的。

模板部分的请求结果如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7acbbdd349eb452ab3335918c85a6e0d~tplv-k3u1fbpfcp-zoom-1.image)


导出了一个`render`函数，这个其实就是`HelloWorld.vue`组件的渲染函数，所以我们通过`module.render`来获取这个函数。

`__VUE_HMR_RUNTIME__.rerender`这个函数是哪里来的呢，其实来自于`Vue`，`Vue`非生产环境的源码会提供一个`__VUE_HMR_RUNTIME__`对象，顾名思义就是用于热更新的，有三个方法：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63c3b7a501f149ebbfc2b4e9a9b84e31~tplv-k3u1fbpfcp-zoom-1.image)


`rerender`就是其中一个：

```js
function rerender(id, newRender) {
    const record = map.get(id);
    if (!record)
        return;
    Array.from(record).forEach(instance => {
        if (newRender) {
            instance.render = newRender;// 1
        }
        instance.renderCache = [];
        isHmrUpdating = true;
        instance.update();// 2
        isHmrUpdating = false;
    });
}
```

核心代码就是上面的`1、2`两行，直接用新的渲染函数覆盖组件旧的渲染函数，然后触发组件更新就达到了热更新的效果。

另外要解释一下其中涉及到的`id`，需要热更新的组件会被添加到`map`里，那怎么判断一个组件是不是需要热更新呢，也很简单，给它添加一个属性即可：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49bd4a441aa74412826c439286ebfa56~tplv-k3u1fbpfcp-zoom-1.image)


在`mountComponent`方法里会判断组件是否存在`__hmrId`属性，存在则认为是需要进行热更新的，那么就添加到`map`里，注册方法如下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d93199c24b2a4417a8e4ad059625cf01~tplv-k3u1fbpfcp-zoom-1.image)


这个`__hmrId`属性需要我们手动添加，所以需要修改一下之前拦截`Vue`单文件的方法：

```js
// app.js
app.use(async function (req, res, next) {
    if (/\.vue\??[^.]*$/.test(req.url)) {
        // vue单文件
        // ...
        // 添加热更新标志
        code += `\n__script.__hmrId = ${JSON.stringify(removeQuery(req.url))}`;// ++
        // 导出
        code += `\nexport default __script`;
        // ...
    }
})
```

## 热更新js

趁热打铁，接下来看一下`Vue`单文件中的`js`部分发生了修改怎么进行热更新。

基本套路是一样的，检查两次的`js`部分是否发生了修改了，修改了则向浏览器发送热更新消息：

```js
// app.js
const handleVueReload = (file) => {
    const sendReload = () => {
        sendMsg({
            type: "vue-reload",
            path: file,
        });
    };
    // js部分发生了改变发送reload事件
    if (!isEqualBlock(descriptor.script, prevDescriptor.script)) {
        return sendReload();
    }
}
```

`js`部分发生改变了就发送一个`vue-reload`消息，接下来修改`client.js`增加对这个消息的处理逻辑：

```js
// client.js
const handleMessage = (payload) => {
  switch (payload.type) {
    case "vue-reload":
      loadModule({
        url: payload.path + "?t=" + Date.now(),
        callback: (module) => {
          window.__VUE_HMR_RUNTIME__.reload(payload.path, module.default);
        },
      });
      break;
  }
}
```

和模板热更新很类似，只不过是调用`reload`方法，这个方法会稍微复杂一点：

```js
function reload(id, newComp) {
    const record = map.get(id);
    if (!record)
        return;
    Array.from(record).forEach(instance => {
        const comp = instance.type;
        if (!hmrDirtyComponents.has(comp)) {
            // 更新原组件
            extend(comp, newComp);
            for (const key in comp) {
                if (!(key in newComp)) {
                    delete comp[key];
                }
            }
            // 标记为脏组件，在虚拟DOM树patch的时候会直接替换
            hmrDirtyComponents.add(comp);
            // 重新加载后取消标记组件
            queuePostFlushCb(() => {
                hmrDirtyComponents.delete(comp);
            });
        }
        if (instance.parent) {
            // 强制父实例重新渲染
            queueJob(instance.parent.update);
        }
        else if (instance.appContext.reload) {
            // 通过createApp()装载的根实例具有reload方法
            instance.appContext.reload();
        }
        else if (typeof window !== 'undefined') {
            window.location.reload();
        }
    });
}
```

通过注释应该能大概看出来它的原理，通过强制父实例重新渲染、调用根实例的`reload`方法、通过标记为脏组件等等方式来重新渲染组件达到更新的效果。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3104aaf5b2b4fbcafbc7c6852a7fe4d~tplv-k3u1fbpfcp-zoom-1.image)


## style热更新

样式更新的情况比较多，除了修改样式本身，还有作用域修改了、使用到了`CSS`变量等情况，简单起见，我们只考虑修改了样式本身。

根据上一篇的介绍，`Vue`单文件中的样式也是通过`js`类型发送到浏览器，然后动态创建`style`标签插入到页面，所以我们需要能删除之前添加的标签，这就需要给添加的`style`标签增加一个`id`了，修改一下上一篇文章里我们编写的`insertStyle`方法：

```js
// app.js
// css to js
const cssToJs = (css, id) => {
  return `
    const insertStyle = (css) => {
		// 删除之前的标签++
		if ('${id}') {
          let oldEl = document.getElementById('${id}')
          if (oldEl) document.head.removeChild(oldEl)
        }
        let el = document.createElement('style')
        el.setAttribute('type', 'text/css')
		el.id = '${id}' // ++
        el.innerHTML = css
        document.head.appendChild(el)
    }
    insertStyle(\`${css}\`)
    export default insertStyle
  `;
};
```

给`style`标签增加一个`id`，然后添加之前先删除之前的标签，接下来需要分别修改一下`css`的拦截逻辑增加`removeQuery(req.url)`作为`id`；以及`Vue`单文件的`style`部分的拦截请求，增加`removeQuery(req.url) + '-' + index`作为`id`，要加上`index`是因为一个`Vue`单文件里可能有多个`style`标签。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61be8c1146004be28e32f714de388828~tplv-k3u1fbpfcp-zoom-1.image)


接下来继续修改`handleVueReload`方法：

```js
// app.js
const handleVueReload = (file) => {
    // ...
    // style部分发生了改变
    const prevStyles = prevDescriptor.styles || []
    const nextStyles = descriptor.styles || []
    nextStyles.forEach((_, i) => {
        if (!prevStyles[i] || !isEqualBlock(prevStyles[i], nextStyles[i])) {
            sendMsg({
                type: 'style-update',
                path: `${file}?import&type=style&index=${i}`,
            })
        }
    })
}
```

遍历新的样式数据，根据之前的进行对比，如果某个样式块之前没有或者不一样那就发送`style-update`事件，注意`url`需要带上`import`及`type=style`参数，这是上一篇里我们规定的。

`client.js`也要配套修改一下：

```js
// client.js
const handleMessage = (payload) => {
    switch (payload.type) {
        case "style-update":
            loadModule({
                url: payload.path + "&t=" + Date.now(),
            });
            break; 
    }
}
```

很简单，加上时间戳重新加载一下样式文件即可。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d1f55f9cb664de2bab6f4584147a91e~tplv-k3u1fbpfcp-zoom-1.image)


不过还有个小问题，比如原来有两个`style`块，我们删掉了一个，目前页面上还是存在的，比如一开始存在两个`style`块：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7b7c62f97564f54a1db6ff5259f84b2~tplv-k3u1fbpfcp-zoom-1.image)



删掉第二个`style`块，也就是设置背景颜色的那个：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e10b0c04fb724483a54afd160bdf4b7e~tplv-k3u1fbpfcp-zoom-1.image)


可以看到还是存在，我们是通过索引来添加的，所以更新后有多少个样式块，就会从头覆盖之前已经存在的多少个样式块，最后多出来的是不会被删除的，所以需要手动删除不再需要的标签：

```js
// app.js
const handleVueReload = (file) => {
    // ...
    // 删除已经被删掉的样式块
    prevStyles.slice(nextStyles.length).forEach((_, i) => {
        sendMsg({
            type: 'style-remove',
            path: file,
            id: `${file}-${i + nextStyles.length}`
        })
    })
}
```

发送一个`style-remove`事件，通知页面删除不再需要的标签：

```js
// client.js
const handleMessage = (payload) => {
    switch (payload.type) {
        case "style-remove":
            document.head.removeChild(document.getElementById(payload.id));
            break;
    }
}
```


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9dcab0237cd449089064fb803e6a4c96~tplv-k3u1fbpfcp-zoom-1.image)


可以看到被成功删掉了。



# 普通js文件的热更新

最后我们来看一下非`Vue`单文件，普通`js`文件更新后要怎么处理。

增加一个处理`js`热更新的函数：

```js
// app.js
// 监听文件改变
watcher.on("change", (file) => {
  if (file.endsWith(".vue")) {
    handleVueReload(file);
  } else if (file.endsWith(".js")) {// ++
    handleJsReload(file);// ++
  }
});
```

普通`js`热更新就需要用到前面的依赖图数据了，如果监听到某个`js`文件修改了，先判断它是否在依赖图中，不是的话就不用管，是的话就递归获取所有依赖它的模块，因为所有模块的最上层依赖肯定是`index.html`，如果只是简单的获取所有依赖模块再更新，那么每次都相当于要刷新整个页面了，所以我们规定如果检查到某个依赖是`Vue`单文件，那么就代表支持热更新，否则就相当于走到死胡同，需要刷新整个页面。

```js
// 处理js文件的热更新
const handleJsReload = (file) => {
  file = filePathToUrl(file);
  // 因为构建依赖图的时候有些是以相对路径引用的，而监听获取到的都是绝对路径，所以稍微兼容一下
  let importers = getImporters(file);
  // 遍历直接依赖
  if (importers && importers.size > 0) {
    // 需要进行热更新的模块
    const hmrBoundaries = new Set();
    // 递归依赖图获取要更新的模块
    const hasDeadEnd = walkImportChain(importers, hmrBoundaries);
    const boundaries = [...hmrBoundaries];
    // 无法热更新，刷新整个页面
    if (hasDeadEnd) {
      sendMsg({
        type: "full-reload",
      });
    } else {
      // 可以热更新
      sendMsg({
        type: "multi",// 可能有多个模块，所以发送一个multi类型的消息
        updates: boundaries.map((boundary) => {
          return {
            type: "vue-reload",
            path: boundary,
          };
        }),
      });
    }
  }
};

// 获取模块的直接依赖模块
const getImporters = (file) => {
  let importers = importerMap.get(file);
  if (!importers || importers.size <= 0) {
    importers = importerMap.get("." + file);
  }
  return importers;
};
```

递归获取修改的`js`文件的依赖模块，判断是否支持热更新，支持则发送热更新事件，否则发送刷新整个页面事件，因为可能同时要更新多个模块，所以通过`type=multi`来标识。

看一下递归的方法`walkImportChain`：

```js
// 递归遍历依赖图
const walkImportChain = (importers, hmrBoundaries, currentChain = []) => {
  for (const importer of importers) {
    if (importer.endsWith(".vue")) {
      // 依赖是Vue单文件那么支持热更新，添加到热更新模块集合里
      hmrBoundaries.add(importer);
    } else {
      // 获取依赖模块的再上层用来模块
      let parentImpoters = getImporters(importer);
      if (!parentImpoters || parentImpoters.size <= 0) {
        // 如果没有上层依赖了，那么代表走到死胡同了
        return true;
      } else if (!currentChain.includes(importer)) {
        // 通过currentChain来存储已经遍历过的模块
        // 递归再上层的依赖
        if (
          walkImportChain(
            parentImpoters,
            hmrBoundaries,
            currentChain.concat(importer)
          )
        ) {
          return true;
        }
      }
    }
  }
  return false;
};
```

逻辑很简单，就是递归遇到`Vue`单文件就停止，否则继续遍历，直到顶端，代表走到死胡同。

最后再来修改一下`client.js`：

```js
// client.js
socket.addEventListener("message", async ({ data }) => {
  const payload = JSON.parse(data);
  // 同时需要更新多个模块
  if (payload.type === "multi") {// ++
    payload.updates.forEach(handleMessage);// ++
  } else {
    handleMessage(payload);
  }
});
```

如果消息类型是`multi`，那么就遍历`updates`列表依次调用处理方法：

```js
// client.js
const handleMessage = (payload) => {
    switch (payload.type) {
        case "full-reload":
            location.reload();
            break;
    }
}
```

`vue-rerender`事件之前已经有了，所以只需要增加一个刷新整个页面的方法即可。

测试一下，`App.vue`里面引入一个`test.js`文件：

```html
// App.vue
<script>
import test from "./test.js";

export default {
  data() {
    return {
      text: "",
    };
  },
  mounted() {
    this.text = test();
  },
};
</script>

<template>
  <div>
    <p>{{ text }}</p>
  </div>
</template>
```

`test.js`又引入了`test2.js`：

```js
// test.js
import test2 from "./test2.js";

export default function () {
  let a = test2();
  let b = "我是测试1";
  return a + " --- " + b;
}

// test2.js
export default function () {
    return '我是测试2'
}
```

接下来修改`test2.js`测试效果：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9b839f6d42d49ef81ecfa708d027fb8~tplv-k3u1fbpfcp-zoom-1.image)


可以看到重新发送了请求，但是页面并没有更新，这是为什么呢，其实还是缓存问题：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc6198dd029947ff99bad560f4522d7e~tplv-k3u1fbpfcp-zoom-1.image)


`App.vue`导入的两个文件之前已经请求过了，所以浏览器会直接使用之前请求的结果，并不会重新发送请求，这要怎么解决呢，很简单，可以看到请求的`App.vue`的`url`是带了时间戳的，所以我们可以检查请求模块的`url`是否存在时间戳，存在则把它依赖的所有模块路径也都带上时间戳，这样就会触发重新请求了，修改一下模块路径转换方法`parseBareImport`：

```js
// app.js
// 处理裸导入
const parseBareImport = async (js, importer) => {
    // ...
    // 检查模块url是否存在时间戳
    let hast = checkQueryExist(importer, "t");// ++
    // ...
    parseResult[0].forEach((item) => {
        let url = "";
        if (item.n[0] !== "." && item.n[0] !== "/") {
            url = `/@module/${item.n}?import${hast ? "&t=" + Date.now() : ""}`;// ++
        } else {
            url = `${item.n}?import${hast ? "&t=" + Date.now() : ""}`;// ++
        }
        // ...
    })
    // ...
}
```

再来测试一下：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22e8a4123bfa41d08e815c85bc2adfdf~tplv-k3u1fbpfcp-zoom-1.image)


可以看到成功更新了。最后我们再来测试运行刷新整个页面的情况，修改一下`main.js`文件即可：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0b0298cb112405681825e57f677ab48~tplv-k3u1fbpfcp-zoom-1.image)


# 总结

本文参考`Vite-1.0.0-rc.5`版本写了一个非常简单的`Vite`，简化了非常多的细节，旨在对`Vite`及热更新有一个基础的认识，其中肯定有不合理或错误之处，欢迎指出~

示例代码在：[https://github.com/wanglin2/vite-demo](https://github.com/wanglin2/vite-demo)。

