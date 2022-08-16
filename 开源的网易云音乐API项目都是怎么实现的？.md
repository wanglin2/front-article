上一篇文章[这个高颜值的开源第三方网易云音乐播放器你值得拥有](https://juejin.cn/post/7116521655844732936)介绍了一个开源的第三方网易云音乐播放器，这篇文章我们来详细了解一下其中使用到的网易云音乐`api`项目[NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi)的实现原理。

[NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi)使用`Node.js`开发，主要用到的框架和库有两个，一个Web应用开发框架[Express](https://www.expressjs.com.cn/)，一个请求库[Axios](https://www.axios-http.cn/)，这两个大家应该都很熟了就不过多介绍了。

## 创建express应用

项目的入口文件为`/app.js`：

```js
async function start() {
  require('./server').serveNcmApi({
    checkVersion: true,
  })
}
start()
```

调用了`/server.js`文件的`serveNcmApi`方法，让我们转到这个文件，`serveNcmApi`方法简化后如下：

```js
async function serveNcmApi(options) {
    const port = Number(options.port || process.env.PORT || '3000')
    const host = options.host || process.env.HOST || ''
    const app = await consturctServer(options.moduleDefs)
    const appExt = app
    appExt.server = app.listen(port, host, () => {
        console.log(`server running @ http://${host ? host : 'localhost'}:${port}`)
    })

    return appExt
}
```

主要是启动监听指定端口，所以创建应用的主要逻辑在`consturctServer`方法：

```js
async function consturctServer(moduleDefs) {
    // 创建一个应用
    const app = express()

    // 设置为true，则客户端的IP地址被理解为X-Forwarded-*报头中最左边的条目
    app.set('trust proxy', true)

    /**
   * 配置CORS & 预检请求
   */
    app.use((req, res, next) => {
        if (req.path !== '/' && !req.path.includes('.')) {
            res.set({
                'Access-Control-Allow-Credentials': true, // 跨域情况下，允许客户端携带验证信息，比如cookie，同时，前端发送请求时也需要设置withCredentials: true
                'Access-Control-Allow-Origin': req.headers.origin || '*', // 允许跨域请求的域名，设置为*代表允许所有域名
                'Access-Control-Allow-Headers': 'X-Requested-With,Content-Type', // 用于给预检请求(options)列出服务端允许的自定义标头，如果前端发送的请求中包含自定义的请求标头，且该标头不包含在Access-Control-Allow-Headers中，那么该请求无法成功发起
                'Access-Control-Allow-Methods': 'PUT,POST,GET,DELETE,OPTIONS', // 设置跨域请求允许的请求方法理想
                'Content-Type': 'application/json; charset=utf-8', // 设置响应数据的类型及编码
            })
        }
        // OPTIONS为预检请求，复杂请求会在发送真正的请求前先发送一个预检请求，获取服务器支持的Access-Control-Allow-xxx相关信息，判断后续是否有必要再发送真正的请求，返回状态码204代表请求成功，但是没有内容
        req.method === 'OPTIONS' ? res.status(204).end() : next()
    })
    // ...
}
```

首先创建了一个`Express`应用，然后设置为信任代理，在`Express`里获取`ip`一般是通过`req.ip`和`req.ips`，`trust proxy`默认值为`false`，这种情况下`req.ips`值是空的，当设置为`true`时，`req.ip`的值会从请求头`X-Forwarded-For`上取最左侧的一个值，`req.ips`则会包含`X-Forwarded-For`头部的所有`ip`地址。

`X-Forwarded-For`头部的格式如下：

```
X-Forwarded-For: client1, proxy1, proxy2
```

值通过一个 `逗号+空格` 把多个`ip`地址区分开，最左边的`client1`是最原始客户端的`ip`地址，代理服务器每成功收到一个请求，就把**请求来源`ip`地址**添加到右边。

以上面为例，这个请求通过了两台代理服务器：`proxy1`和`proxy2`。请求由`client1`发出，此时`XFF`是空的，到了`proxy1`时，`proxy1`把`client1`添加到`XFF`中，之后请求发往`proxy2`，通过`proxy2`的时候，`proxy1`被添加到`XFF`中，之后请求发往最终服务器，到达后`proxy2`被添加到`XFF`中。

但是伪造这个字段非常容易，所以当代理不可信时，这个字段也不一定可靠，不过正常情况下`XFF`中最后一个`ip`地址肯定是最后一个代理服务器的`ip`地址，这个会比较可靠。

随后设置了跨域响应头，这里的设置就是允许不同域名的网站也能请求成功的关键所在。

继续：

```js
async function consturctServer(moduleDefs) {
    // ...
    /**
   * 解析Cookie
   */
    app.use((req, _, next) => {
        req.cookies = {}
        //;(req.headers.cookie || '').split(/\s*;\s*/).forEach((pair) => { //  Polynomial regular expression //
        // 从请求头中读取cookie，cookie格式为：name=value;name2=value2...，所以先根据;切割为数组
        ;(req.headers.cookie || '').split(/;\s+|(?<!\s)\s+$/g).forEach((pair) => {
            let crack = pair.indexOf('=')
            // 没有值的直接跳过
            if (crack < 1 || crack == pair.length - 1) return
            // 将cookie保存到cookies对象上
            req.cookies[decode(pair.slice(0, crack)).trim()] = decode(
                pair.slice(crack + 1),
            ).trim()
        })
        next()
    })

    /**
   * 请求体解析和文件上传处理
   */
    app.use(express.json())
    app.use(express.urlencoded({ extended: false }))
    app.use(fileUpload())

    /**
   * 将public目录下的文件作为静态文件提供
   */
    app.use(express.static(path.join(__dirname, 'public')))

    /**
   * 缓存请求，两分钟内同样的请求会从缓存里读取数据，不会向网易云音乐服务器发送请求
   */
    app.use(cache('2 minutes', (_, res) => res.statusCode === 200))
    // ...
}
```

接下来注册了一些中间件，用来解析`cookie`、处理请求体等，另外还做了接口缓存，防止太频繁请求网易云音乐服务器导致被封掉。

继续：

```js
async function consturctServer(moduleDefs) {
    // ...
    /**
   * 特殊路由
   */
    const special = {
        'daily_signin.js': '/daily_signin',
        'fm_trash.js': '/fm_trash',
        'personal_fm.js': '/personal_fm',
    }

    /**
   * 加载/module目录下的所有模块，每个模块对应一个接口
   */
    const moduleDefinitions =
          moduleDefs ||
          (await getModulesDefinitions(path.join(__dirname, 'module'), special))
    // ...
}
```

接下来加载了`/module`目录下所有的模块：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5daee62151fd46f7830bff572e6dc2d7~tplv-k3u1fbpfcp-zoom-1.image)

每个模块代表一个对网易云音乐接口的请求，比如获取专辑详情的`album_detail.js`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a95221c2fdd24ad2963429b37f8f56c9~tplv-k3u1fbpfcp-zoom-1.image)

模块加载方法`getModulesDefinitions`如下：

```js
async function getModulesDefinitions(
  modulesPath,
  specificRoute,
  doRequire = true,
) {
  const files = await fs.promises.readdir(modulesPath)
  const parseRoute = (fileName) =>
    specificRoute && fileName in specificRoute
      ? specificRoute[fileName]
      : `/${fileName.replace(/\.js$/i, '').replace(/_/g, '/')}`
  // 遍历目录下的所有文件
  const modules = files
    .reverse()
    .filter((file) => file.endsWith('.js'))// 过滤出js文件
    .map((file) => {
      const identifier = file.split('.').shift()// 模块标识
      const route = parseRoute(file)// 模块对应的路由
      const modulePath = path.join(modulesPath, file)// 模块路径
      const module = doRequire ? require(modulePath) : modulePath// 加载模块

      return { identifier, route, module }
    })

  return modules
}
```

以刚才的`album_detail.js`模块为例，返回的数据如下：

```js
{ 
    identifier: 'album_detail', 
    route: '/album/detail', 
    module: () => {/*模块内容*/}
}
```

接下来就是注册路由：

```js
async function consturctServer(moduleDefs) { 
    // ...
    for (const moduleDef of moduleDefinitions) {
        // 注册路由
        app.use(moduleDef.route, async (req, res) => {
            // cookie也可以从查询参数、请求体上传来
            ;[req.query, req.body].forEach((item) => {
                if (typeof item.cookie === 'string') {
                    // 将cookie字符串转换成json类型
                    item.cookie = cookieToJson(decode(item.cookie))
                }
            })

            // 把cookie、查询参数、请求头、文件都整合到一起，作为参数传给每个模块
            let query = Object.assign(
                {},
                { cookie: req.cookies },
                req.query,
                req.body,
                req.files,
            )

            try {
                // 执行模块方法，即发起对网易云音乐接口的请求
                const moduleResponse = await moduleDef.module(query, (...params) => {
                    // 参数注入客户端IP
                    const obj = [...params]
                    // 处理ip，为了实现IPv4-IPv6互通，IPv4地址前会增加::ffff:
                    let ip = req.ip
                    if (ip.substr(0, 7) == '::ffff:') {
                        ip = ip.substr(7)
                    }
                    obj[3] = {
                        ...obj[3],
                        ip,
                    }
                    return request(...obj)
                })
                // 请求成功后，获取响应中的cookie，并且通过Set-Cookie响应头来将这个cookie设置到前端浏览器上
                const cookies = moduleResponse.cookie
                if (Array.isArray(cookies) && cookies.length > 0) {
                    if (req.protocol === 'https') {
                        // 去掉跨域请求cookie的SameSite限制，这个属性用来限制第三方Cookie，从而减少安全风险
                        res.append(
                            'Set-Cookie',
                            cookies.map((cookie) => {
                                return cookie + '; SameSite=None; Secure'
                            }),
                        )
                    } else {
                        res.append('Set-Cookie', cookies)
                    }
                }
                // 回复请求
                res.status(moduleResponse.status).send(moduleResponse.body)
            } catch (moduleResponse) {
                // 请求失败处理
                // 没有响应体，返回404
                if (!moduleResponse.body) {
                    res.status(404).send({
                        code: 404,
                        data: null,
                        msg: 'Not Found',
                    })
                    return
                }
                // 301代表调用了需要登录的接口，但是并没有登录
                if (moduleResponse.body.code == '301')
                    moduleResponse.body.msg = '需要登录'
                res.append('Set-Cookie', moduleResponse.cookie)
                res.status(moduleResponse.status).send(moduleResponse.body)
            }
        })
    }

    return app
}
```

逻辑很清晰，将每个模块都注册成一个路由，接收到对应的请求后，将`cookie`、查询参数、请求体等都传给对应的模块，然后请求网易云音乐的接口，如果请求成功了，那么处理一下网易云音乐接口返回的`cookie`，最后将数据都返回给前端即可，如果接口失败了，那么也进行对应的处理。

其中从请求的查询参数和请求体里获取`cookie`可能不是很好理解，因为`cookie`一般是从请求体里带过来，这么做应该主要是为了支持在`Node.js`里调用：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c94b5245d6cf43f2bd731236864530a1~tplv-k3u1fbpfcp-zoom-1.image)

请求成功后，返回的数据里如果存在`cookie`，那么会进行一些处理，首先如果是`https`的请求，那么会设置`SameSite=None; Secure`，`SameSite`是`Cookie`中的一个属性，用来限制第三方`Cookie`，从而减少安全风险。`Chrome 51` 开始新增这个属性，用来防止`CSRF`攻击和用户追踪，有三个可选值：`strict/lax/none`，默认为`lax`，比如在域名为`https://123.com`的页面里调用`https://456.com`域名的接口，默认情况下除了导航到`123`网址的`get`请求除外，其他请求都不会携带`123`域名的`cookie`，如果设置为`strict`更严格，完全不会携带`cookie`，所以这个项目为了方便跨域调用，设置为`none`，不进行限制，设置为`none`的同时需要设置`Secure`属性。

最后通过`Set-Cookie`响应头将`cookie`写入前端的浏览器即可。

# 发送请求

接下来看一下上面涉及到发送请求所使用的`request`方法，这个方法在`/util/request.js`文件，首先引入了一些模块：

```js
const encrypt = require('./crypto')
const axios = require('axios')
const PacProxyAgent = require('pac-proxy-agent')
const http = require('http')
const https = require('https')
const tunnel = require('tunnel')
const { URLSearchParams, URL } = require('url')
const config = require('../util/config.json')
// ...
```

然后就是具体发送请求的方法`createRequest`，这个方法也挺长的，我们慢慢来看：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        let headers = { 'User-Agent': chooseUserAgent(options.ua) }
        // ...
        })
}
```

函数会返回一个`Promise`，首先定义了一个请求头对象，并添加了`User-Agent`头，这个头部会保存浏览器类型、版本号、渲染引擎，以及操作系统、版本、CPU类型等信息，标准格式为： 

```
浏览器标识 (操作系统标识; 加密等级标识; 浏览器语言) 渲染引擎标识 版本信息
```

不用多说，伪造这个头显然是用来欺骗服务器，让它认为这个请求是来自浏览器，而不是同样也来自服务端。

默认写死了几个`User-Agent`头部随机进行选择：

```js
const chooseUserAgent = (ua = false) => {
    const userAgentList = {
        mobile: [
            'Mozilla/5.0 (iPhone; CPU iPhone OS 13_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.1.1 Mobile/15E148 Safari/604.1',
            'Mozilla/5.0 (Linux; Android 9; PCT-AL10) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.64 HuaweiBrowser/10.0.3.311 Mobile Safari/537.36',
            // ...
        ],
        pc: [
            'Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:80.0) Gecko/20100101 Firefox/80.0',
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:80.0) Gecko/20100101 Firefox/80.0',
            // ...
        ],
    }
    let realUserAgentList =
        userAgentList[ua] || userAgentList.mobile.concat(userAgentList.pc)
    return ['mobile', 'pc', false].indexOf(ua) > -1
        ? realUserAgentList[Math.floor(Math.random() * realUserAgentList.length)]
    : ua
}
```

继续看：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        // ...
        // 如果是post请求，修改编码格式
        if (method.toUpperCase() === 'POST')
            headers['Content-Type'] = 'application/x-www-form-urlencoded'
        // 伪造Referer头
        if (url.includes('music.163.com'))
            headers['Referer'] = 'https://music.163.com'
        // 设置ip头部
        let ip = options.realIP || options.ip || ''
        if (ip) {
            headers['X-Real-IP'] = ip
            headers['X-Forwarded-For'] = ip
        }
        // ...
    })
}
```

继续设置了几个头部字段，`Axios`默认的编码格式为`json`，而`POST`请求一般都会使用`application/x-www-form-urlencoded`编码格式。

`Referer`头代表发送请求时所在页面的`url`，比如在`https://123.com`页面内调用`https://456.com`接口，`Referer`头会设置为`https://123.com`，这个头部一般用来防盗链。所以伪造这个头部也是为了欺骗服务器这个请求是来自它们自己的页面。

接下来设置了两个`ip`头部，`realIP`需要前端手动传递：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27f2e9c3f95b4070a9187b306c8d8b25~tplv-k3u1fbpfcp-zoom-1.image)

继续：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        // ...
        // 设置cookie
        if (typeof options.cookie === 'object') {
            if (!options.cookie.MUSIC_U) {
                // 游客
                if (!options.cookie.MUSIC_A) {
                    options.cookie.MUSIC_A = config.anonymous_token
                }
            }
            headers['Cookie'] = Object.keys(options.cookie)
                .map(
                (key) =>
                encodeURIComponent(key) +
                '=' +
                encodeURIComponent(options.cookie[key]),
            )
                .join('; ')
        } else if (options.cookie) {
            headers['Cookie'] = options.cookie
        }
        // ...
    })
}
```

接下来设置`cookie`，分两种类型，一种是对象类型，这种情况`cookie`一般来源于查询参数或者请求体，另一种为字符串，这个就是正常情况下请求头带过来的。`MUSIC_U`应该就是登录后的`cookie`了，`MUSIC_A`应该是一个`token`，未登录情况下调用某些接口可能报错，所以会设置一个游客`token`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3731fab16a14f7f971241d8f58ff09b~tplv-k3u1fbpfcp-zoom-1.image)

继续：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        // ...
        if (options.crypto === 'weapi') {
            let csrfToken = (headers['Cookie'] || '').match(/_csrf=([^(;|$)]+)/)
            data.csrf_token = csrfToken ? csrfToken[1] : ''
            data = encrypt.weapi(data)
            url = url.replace(/\w*api/, 'weapi')
        } else if (options.crypto === 'linuxapi') {
            data = encrypt.linuxapi({
                method: method,
                url: url.replace(/\w*api/, 'api'),
                params: data,
            })
            headers['User-Agent'] =
                'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36'
            url = 'https://music.163.com/api/linux/forward'
        } else if (options.crypto === 'eapi') {
            const cookie = options.cookie || {}
            const csrfToken = cookie['__csrf'] || ''
            const header = {
                osver: cookie.osver, //系统版本
                deviceId: cookie.deviceId, //encrypt.base64.encode(imei + '\t02:00:00:00:00:00\t5106025eb79a5247\t70ffbaac7')
                appver: cookie.appver || '8.7.01', // app版本
                versioncode: cookie.versioncode || '140', //版本号
                mobilename: cookie.mobilename, //设备model
                buildver: cookie.buildver || Date.now().toString().substr(0, 10),
                resolution: cookie.resolution || '1920x1080', //设备分辨率
                __csrf: csrfToken,
                os: cookie.os || 'android',
                channel: cookie.channel,
                requestId: `${Date.now()}_${Math.floor(Math.random() * 1000)
                .toString()
                .padStart(4, '0')}`,
            }
            if (cookie.MUSIC_U) header['MUSIC_U'] = cookie.MUSIC_U
            if (cookie.MUSIC_A) header['MUSIC_A'] = cookie.MUSIC_A
            headers['Cookie'] = Object.keys(header)
                .map(
                (key) =>
                encodeURIComponent(key) + '=' + encodeURIComponent(header[key]),
            )
                .join('; ')
            data.header = header
            data = encrypt.eapi(options.url, data)
            url = url.replace(/\w*api/, 'eapi')
        }
        // ...
    })
}
```

这一段代码会比较难理解，笔者也没有看懂，反正大致呢这个项目使用了四种类型网易云音乐的接口：`weapi`、`linuxapi`、`eapi`、`api`，比如：

```
https://music.163.com/weapi/vipmall/albumproduct/detail
https://music.163.com/eapi/activate/initProfile
https://music.163.com/api/album/detail/dynamic
```

每种类型的接口请求参数、加密方式都不一样，所以需要分开单独处理：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57834308cc414ee2a2313cc06e830415~tplv-k3u1fbpfcp-zoom-1.image)

比如`weapi`：

```js
let csrfToken = (headers['Cookie'] || '').match(/_csrf=([^(;|$)]+)/)
data.csrf_token = csrfToken ? csrfToken[1] : ''
data = encrypt.weapi(data)
url = url.replace(/\w*api/, 'weapi')
```

将`cookie`中的`_csrf`值取出加到请求数据中，然后加密数据：

```js
const weapi = (object) => {
  const text = JSON.stringify(object)
  const secretKey = crypto
    .randomBytes(16)
    .map((n) => base62.charAt(n % 62).charCodeAt())
  return {
    params: aesEncrypt(
      Buffer.from(
        aesEncrypt(Buffer.from(text), 'cbc', presetKey, iv).toString('base64'),
      ),
      'cbc',
      secretKey,
      iv,
    ).toString('base64'),
    encSecKey: rsaEncrypt(secretKey.reverse(), publicKey).toString('hex'),
  }
}
```

查看其他加密算法：[crypto.js](https://github.com/Binaryify/NeteaseCloudMusicApi/blob/master/util/crypto.js)。

至于这些是怎么知道的呢，要么就是网易云音乐内部人士（基本不可能），要么就是进行逆向了，比如网页版的接口，打开控制台，发送请求，找到在源码中的位置， 打断点，查看请求数据结构，阅读压缩或混淆后的源码慢慢进行尝试，总之，向这些大佬致敬。

继续：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        // ...
        // 响应的数据结构
        const answer = { status: 500, body: {}, cookie: [] }
        // 请求配置
        let settings = {
            method: method,
            url: url,
            headers: headers,
            data: new URLSearchParams(data).toString(),
            httpAgent: new http.Agent({ keepAlive: true }),
            httpsAgent: new https.Agent({ keepAlive: true }),
        }
        if (options.crypto === 'eapi') settings.encoding = null
        // 配置代理
        if (options.proxy) {
            if (options.proxy.indexOf('pac') > -1) {
                settings.httpAgent = new PacProxyAgent(options.proxy)
                settings.httpsAgent = new PacProxyAgent(options.proxy)
            } else {
                const purl = new URL(options.proxy)
                if (purl.hostname) {
                    const agent = tunnel.httpsOverHttp({
                        proxy: {
                            host: purl.hostname,
                            port: purl.port || 80,
                        },
                    })
                    settings.httpsAgent = agent
                    settings.httpAgent = agent
                    settings.proxy = false
                } else {
                    console.error('代理配置无效,不使用代理')
                }
            }
        } else {
            settings.proxy = false
        }
        if (options.crypto === 'eapi') {
            settings = {
                ...settings,
                responseType: 'arraybuffer',
            }
        }
        // ...
    })
}
```

这里主要是定义了响应的数据结构、定义了请求的配置数据，以及针对`eapi`做了一些特殊处理，最主要是代理的相关配置。

`Agent`是`Node.js`的`HTTP`模块中的一个类，负责管理`http`客户端连接的持久性和重用。 它维护一个给定主机和端口的待处理请求队列，为每个请求重用单个套接字连接，直到队列为空，此时套接字要么被销毁，要么放入池中，在池里会被再次用于请求到相同的主机和端口，总之就是省去了每次发起`http`请求时需要重新创建套接字的时间，提高效率。

`pac`指代理自动配置，其实就是包含了一个`javascript`函数的文本文件，这个函数会决定是直接连接还是通过某个代理连接，比直接写死一个代理方便一点，当然需要配置的`options.proxy`是这个文件的远程地址，格式为：`'pac+【pac文件地址】+'`。`pac-proxy-agent`模块会提供一个`http.Agent`实现，它会根据指定的`PAC`代理文件判断使用哪个`HTTP`、`HTTPS` 或`SOCKS`代理，或者是直接连接。

至于为什么要使用`tunnel`模块，笔者搜索了一番还是没有搞懂，可能是解决`http`协议的接口请求网易云音乐的`https`协议接口失败的问题？知道的朋友可以评论区解释一下~

最后：

```js
const createRequest = (method, url, data = {}, options) => {
    return new Promise((resolve, reject) => {
        // ...
        axios(settings)
            .then((res) => {
                const body = res.data
                // 将响应的set-cookie头中的cookie取出，直接保存到响应对象上
                answer.cookie = (res.headers['set-cookie'] || []).map((x) =>
                	x.replace(/\s*Domain=[^(;|$)]+;*/, ''),// 去掉域名限制
                )
                try {
                    // eapi返回的数据也是加密的，需要解密
                    if (options.crypto === 'eapi') {
                        answer.body = JSON.parse(encrypt.decrypt(body).toString())
                    } else {
                        answer.body = body
                    }
                    answer.status = answer.body.code || res.status
                    // 统一这些状态码为200，都代表成功
                    if (
                        [201, 302, 400, 502, 800, 801, 802, 803].indexOf(answer.body.code) > -1
                    ) {
                        // 特殊状态码
                        answer.status = 200
                    }
                } catch (e) {
                    try {
                        answer.body = JSON.parse(body.toString())
                    } catch (err) {
                        answer.body = body
                    }
                    answer.status = res.status
                }
                answer.status =
                    100 < answer.status && answer.status < 600 ? answer.status : 400
            	// 状态码200代表成功，其他都代表失败
                if (answer.status === 200) resolve(answer)
                else reject(answer)
            })
            .catch((err) => {
                answer.status = 502
                answer.body = { code: 502, msg: err }
                reject(answer)
            })
    })
}
```

最后一步就是使用`Axios`发送请求了，处理了一下响应的`cookie`，保存到响应对象上，方便后续使用，另外处理了一些状态码，可以看到`try-catch`的使用比较多，至于为什么呢，估计要多尝试来能知道到底哪里会出错了，有兴趣的可以自行尝试。



# 总结

本文通过源码角度了解了一下[NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi)项目的实现原理，可以看到整个流程是比较简单的。无非就是一个请求代理，难的在于找出这些接口，并且逆向分析出每个接口的参数，加密方法，解密方法。最后也提醒一下，这个项目仅供学习使用，请勿从事商业行为或进行破坏版权行为~

