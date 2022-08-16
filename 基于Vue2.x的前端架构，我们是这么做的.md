通过`Vue CLI`可以方便的创建一个`Vue`项目，但是对于实际项目来说还是不够的，所以一般都会根据业务的情况来在其基础上添加一些共性能力，减少创建新项目时的一些重复操作，本着学习和分享的目的，本文会介绍一下我们`Vue`项目的前端架构设计，当然，有些地方可能不是最好的方式，毕竟大家的业务不尽相同，适合你的就是最好的。

除了介绍基本的架构设计，本文还会介绍如何开发一个`Vue CLI`插件和`preset`预设。

```
ps.本文基于Vue2.x版本，node版本16.5.0
```



# 创建一个基本项目

先使用`Vue CLI`创建一个基本的项目：

```bash
vue create hello-world
```

然后选择`Vue2`选项创建，初始项目结构如下：

![image-20220126101820738.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2b394bc81074f40b24e238c614ce53f~tplv-k3u1fbpfcp-watermark.image?)

接下来就在此基础上添砖加瓦。



# 路由

路由是必不可少的，安装`vue-router`：

```bash
npm install vue-router
```

修改`App.vue`文件：

```html
<template>
  <div id="app">
    <router-view />
  </div>
</template>

<script>
export default {
  name: 'App',
}
</script>

<style>
* {
  padding: 0;
  margin: 0;
  border: 0;
  outline: none;
}

html,
body {
  width: 100%;
  height: 100%;
}
</style>
<style scoped>
#app {
  width: 100%;
  height: 100%;
  display: flex;
}
</style>
```

增加路由出口，简单设置了一下页面样式。

接下来新增`pages`目录用于放置页面， 把原本`App.vue`的内容移到了`Hello.vue`：

![image-20220126140342614.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/305e127bbd484a688f5c3d110b8e4224~tplv-k3u1fbpfcp-watermark.image?)

路由配置我们选择基于文件进行配置，在`src`目录下新建一个`/src/router.config.js`：

```js
export default [
  {
    path: '/',
    redirect: '/hello',
  },
  {
    name: 'hello',
    path: '/hello/',
    component: 'Hello',
  }
]
```

属性支持`vue-router`构建选项[routes](https://router.vuejs.org/zh/api/#routes)的所有属性，`component`属性传的是`pages`目录下的组件路径，规定路由组件只能放到`pages`目录下，然后新建一个`/src/router.js`文件：

```js
import Vue from 'vue'
import Router from 'vue-router'
import routes from './router.config.js'

Vue.use(Router)

const createRoute = (routes) => {
    if (!routes) {
        return []
    }
    return routes.map((item) => {
        return {
            ...item,
            component: () => {
                return import('./pages/' + item.component)
            },
            children: createRoute(item.children)
        }
    })
}

const router = new Router({
    mode: 'history',
    routes: createRoute(routes),
})

export default router
```

使用工厂函数和`import`方法来定义动态组件，需要递归对子路由进行处理。最后，在`main.js`里面引入路由：

```js
// main.js
// ...
import router from './router'// ++
// ...
new Vue({
  router,// ++
  render: h => h(App),
}).$mount('#app')
```



# 菜单

我们的业务基本上都需要一个菜单，默认显示在页面左侧，我们有内部的组件库，但没有对外开源，所以本文就使用`Element`替代，菜单也通过文件来配置，新建`/src/nav.config.js`文件：

```js
export default [{
    title: 'hello',
    router: '/hello',
    icon: 'el-icon-menu'
}]
```

然后修改`App.vue`文件：

```html
<template>
  <div id="app">
    <el-menu
      style="width: 250px; height: 100%"
      :router="true"
      :default-active="defaultActive"
    >
      <el-menu-item
        v-for="(item, index) in navList"
        :key="index"
        :index="item.router"
      >
        <i :class="item.icon"></i>
        <span slot="title">{{ item.title }}</span>
      </el-menu-item>
    </el-menu>
    <router-view />
  </div>
</template>

<script>
import navList from './nav.config.js'
export default {
  name: 'App',
  data() {
    return {
      navList,
    }
  },
  computed: {
    defaultActive() {
      let path = this.$route.path
      // 检查是否有完全匹配的
      let fullMatch = navList.find((item) => {
        return item.router === path
      })
      // 没有则检查是否有部分匹配
      if (!fullMatch) {
        fullMatch = navList.find((item) => {
          return new RegExp('^' + item.router + '/').test(path)
        })
      }
      return fullMatch ? fullMatch.router : ''
    },
  },
}
</script>
```

效果如下：

![image-20220126145352732.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21b1d5fbf45f45a0873c5cdf2ceb8cf9~tplv-k3u1fbpfcp-watermark.image?)

当然，上述只是意思一下，实际的要复杂一些，毕竟这里连嵌套菜单的情况都没考虑。



# 权限

我们的权限颗粒度比较大，只控制到路由层面，具体实现就是在菜单配置和路由配置里的每一项都新增一个`code`字段，然后通过请求获取当前用户有权限的`code`，没有权限的菜单默认不显示，访问没有权限的路由会重定向到`403`页面。 

## 获取权限数据

权限数据随用户信息接口一起返回，然后存储到`vuex`里，所以先配置一下`vuex`，安装：

```bash
npm install vuex --save
```

新增`/src/store.js`：

```js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
    state: {
        userInfo: null,
    },
    actions: {
        // 请求用户信息
        async getUserInfo(ctx) {
            let userInfo = {
                // ...
                code: ['001'] // 用户拥有的权限
            }
            ctx.commit('setUserInfo', userInfo)
        }
    },
    mutations: {
        setUserInfo(state, userInfo) {
            state.userInfo = userInfo
        }
    },
})
```

在`main.js`里面先获取用户信息，然后再初始化`Vue`：

```js
// ...
import store from './store'
// ...
const initApp = async () => {
  await store.dispatch('getUserInfo')
  new Vue({
    router,
    store,
    render: h => h(App),
  }).$mount('#app')
}
initApp()
```

## 菜单

修改`nav.config.js`新增`code`字段：

```js
// nav.config.js
export default [{
    title: 'hello',
    router: '/hello',
    icon: 'el-icon-menu'
    code: '001',
}]
```

然后在`App.vue`里过滤掉没有权限的菜单：

```js
export default {
  name: 'App',
  data() {
    return {
      navList,// --
    }
  },
  computed: {
    navList() {// ++
      const { userInfo } = this.$store.state
      if (!userInfo || !userInfo.code || userInfo.code.length <= 0) return []
      return navList.filter((item) => {
        return userInfo.code.includes(item.code)
      })
    }
  }
}
```

这样没有权限的菜单就不会显示出来。

## 路由

修改`router.config.js`，增加`code`字段：

```js
export default [{
        path: '/',
        redirect: '/hello',
    },
    {
        name: 'hello',
        path: '/hello/',
        component: 'Hello',
        code: '001',
    }
]
```

`code`是自定义字段，需要保存到路由记录的`meta`字段里，否则最后会丢失，修改`createRoute`方法：

```js
// router.js
// ...
const createRoute = (routes) => {
    // ...
    return routes.map((item) => {
        return {
            ...item,
            component: () => {
                return import('./pages/' + item.component)
            },
            children: createRoute(item.children),
            meta: {// ++
                code: item.code
            }
        }
    })
}
// ...
```

然后需要拦截路由跳转，判断是否有权限，没有权限就转到`403`页面：

```js
// router.js
// ...
import store from './store'
// ...
router.beforeEach((to, from, next) => {
    const userInfo = store.state.userInfo
    const code = userInfo && userInfo.code && userInfo.code.length > 0 ? userInfo.code : []
    // 去错误页面直接跳转即可，否则会引起死循环
    if (/^\/error\//.test(to.path)) {
        return next()
    }
    // 有权限直接跳转
    if (code.includes(to.meta.code)) {
        next()
    } else if (to.meta.code) { // 路由存在，没有权限，跳转到403页面
        next({
            path: '/error/403'
        })
    } else { // 没有code则代表是非法路径，跳转到404页面
        next({
            path: '/error/404'
        })
    }
})
```

`error`组件还没有，新增一下：

```html
// pages/Error.vue

<template>
  <div class="container">{{ errorText }}</div>
</template>

<script>
const map = {
  403: '无权限',
  404: '页面不存在',
}
export default {
  name: 'Error',
  computed: {
    errorText() {
      return map[this.$route.params.type] || '未知错误'
    },
  },
}
</script>

<style scoped>
.container {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 50px;
}
</style>
```

接下来修改一下`router.config.js`，增加错误页面的路由，及增加一个测试无权限的路由：

```js
// router.config.js

export default [
    // ...
    {
        name: 'Error',
        path: '/error/:type',
        component: 'Error',
    },
    {
        name: 'hi',
        path: '/hi/',
        code: '无权限测试，请输入hi',
        component: 'Hello',
    }
]
```

因为这个`code`用户并没有，所以现在我们打开`/hi`路由会直接跳转到`403`路由：

![2022-02-10-14-01-59.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/855718d7b09845f69ae1df2f67e86042~tplv-k3u1fbpfcp-watermark.image?)

# 面包屑

和菜单类似，面包屑也是大部分页面都需要的，面包屑的组成分为两部分，一部分是在当前菜单中的位置，另一部分是在页面操作中产生的路径。第一部分的路径因为可能会动态的变化，所以一般是通过接口随用户信息一起获取，然后存到`vuex`里，修改`store.js`：

```js
// ...
async getUserInfo(ctx) {
    let userInfo = {
        code: ['001'],
        breadcrumb: {// 增加面包屑数据
            '001': ['你好'],
        },
    }
    ctx.commit('setUserInfo', userInfo)
}
// ...
```

第二部分的在`router.config.js`里面配置：

```js
export default [
    //...
    {
        name: 'hello',
        path: '/hello/',
        component: 'Hello',
        code: '001',
        breadcrumb: ['世界'],// ++
    }
]
```

`breadcrumb`字段和`code`字段一样，属于自定义字段，但是这个字段的数据是给组件使用的，组件需要获取这个字段的数据然后在页面上渲染出面包屑菜单，所以保存到`meta`字段上虽然可以，但是在组件里面获取比较麻烦，所以我们可以设置到路由记录的`props`字段上，直接注入为组件的`props`，这样使用就方便多了，修改`router.js`：

```js
// router.js
// ...
const createRoute = (routes) => {
    // ...
    return routes.map((item) => {
        return {
            ...item,
            component: () => {
                return import('./pages/' + item.component)
            },
            children: createRoute(item.children),
            meta: {
                code: item.code
            },
            props: {// ++
                breadcrumbObj: {
                    breadcrumb: item.breadcrumb,
                    code: item.code
                } 
            }
        }
    })
}
// ...
```

这样在组件里声明一个`breadcrumbObj`属性即可获取到面包屑数据，可以看到把`code`也一同传过去了，这是因为还要根据当前路由的`code`从用户接口获取的面包屑数据中取出该路由`code`对应的面包屑数据，然后把两部分的进行合并，这个工作为了避免让每个组件都要做一遍，我们可以写在一个全局的`mixin`里，修改`main.js`：

```js
// ...
Vue.mixin({
    props: {
        breadcrumbObj: {
            type: Object,
            default: () => null
        }
    },
    computed: {
        breadcrumb() {
            if (!this.breadcrumbObj) {
                return []
            }
            let {
                code,
                breadcrumb
            } = this.breadcrumbObj
            // 用户接口获取的面包屑数据
            let breadcrumbData = this.$store.state.userInfo.breadcrumb
            // 当前路由是否存在面包屑数据
            let firstBreadcrumb = breadcrumbData && Array.isArray(breadcrumbData[code]) ? breadcrumbData[code] : []
            // 合并两部分的面包屑数据
            return firstBreadcrumb.concat(breadcrumb || [])
        }
    }
})

// ...
initApp()
```

最后我们在`Hello.vue`组件里面渲染一下面包屑：

```html
<template>
  <div class="container">
    <el-breadcrumb separator="/">
      <el-breadcrumb-item v-for="(item, index) in breadcrumb" :key="index">{{item}}</el-breadcrumb-item>
    </el-breadcrumb>
    // ...
  </div>
</template>
```

![image-20220210152155551.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e11e7423483d461eaa9cf46197011edf~tplv-k3u1fbpfcp-watermark.image?)

当然，我们的面包屑是不需要支持点击的，如果需要的话可以修改一下面包屑的数据结构。



# 接口请求

接口请求使用的是`axios`，但是会做一些基础配置、拦截请求和响应，因为还是有一些场景需要直接使用未配置的`axios`，所以我们默认创建一个新实例，先安装：

```bash
npm install axios
```

然后新建一个`/src/api/`目录，在里面新增一个`httpInstance.js`文件：

```js
import axios from 'axios'

// 创建一个新实例
const http = axios.create({
    timeout: 10000,// 超时时间设为10秒
    withCredentials: true,// 跨域请求时是否需要使用凭证，设置为需要
    headers: {
        'X-Requested-With': 'XMLHttpRequest'// 表明是ajax请求
    },
})

export default http
```

然后增加一个请求拦截器：

```js
// ...
// 请求拦截器
http.interceptors.request.use(function (config) {
    // 在发送请求之前做些什么
    return config;
}, function (error) {
    // 对请求错误做些什么
    return Promise.reject(error);
});
// ...
```

其实啥也没做，先写出来，留着不同的项目按需修改。

最后增加一个响应拦截器：

```js
// ...
import { Message } from 'element-ui'
// ...
// 响应拦截器
http.interceptors.response.use(
    function (response) {
        // 对错误进行统一处理
        if (response.data.code !== '0') {
            // 弹出错误提示
            if (!response.config.noMsg && response.data.msg) {
                Message.error(response.data.msg)
            }
            return Promise.reject(response)
        } else if (response.data.code === '0' && response.config.successNotify && response.data.msg) {
            // 弹出成功提示
            Message.success(response.data.msg)
        }
        return Promise.resolve({
            code: response.data.code,
            msg: response.data.msg,
            data: response.data.data,
        })
    },
    function (error) {
        // 登录过期
        if (error.status === 403) {
            location.reload()
            return
        }
        // 超时提示
        if (error.message.indexOf('timeout') > -1) {
            Message.error('请求超时，请重试！')
        }
        return Promise.reject(error)
    },
)
// ...
```

我们约定一个成功的响应（状态码为200）结构如下：

```js
{
    code: '0',
    msg: 'xxx',
    data: xxx
}
```

`code`不为`0`即使状态码为`200`也代表请求出错，那么弹出错误信息提示框，如果某次请求不希望自动弹出提示框的话也可以禁止，只要在请求时加上配置参数`noMsg: true`即可，比如：

```js
axios.get('/xxx', {
    noMsg: true
})
```

请求成功默认不弹提示，需要的话可以设置配置参数`successNotify: true`。

状态码在非`[200,300)`之间的错误只处理两种，登录过期和请求超时，其他情况可根据项目自行修改。

# 多语言

多语言使用[vue-i18n](https://github.com/kazupon/vue-i18n)实现，先安装：

```bash
npm install vue-i18n@8
```

`vue-i18n`的`9.x`版本支持的是`Vue3`，所以我们使用`8.x`版本。

然后创建一个目录`/src/i18n/`，在目录下新建`index.js`文件用来创建`i18n`实例：

```js
import Vue from 'vue'
import VueI18n from 'vue-i18n'

Vue.use(VueI18n)
const i18n = new VueI18n()

export default i18n
```

除了创建实例其他啥也没做，别急，接下来我们一步步来。

我们的总体思路是，多语言的源数据在`/src/i18n/`下，然后编译成`json`文件放到项目的`/public/i18n/`目录下，页面的初始默认语言也是和用户信息接口一起返回，页面根据默认的语言类型使用`ajax`请求`public`目录下的对应`json`文件，调用`VueI18n`的方法动态进行设置。

这么做的目的首先是方便修改页面默认语言，其次是多语言文件不和项目代码打包到一起，减少打包时间，按需请求，减少不必要的资源请求。

接下来我们新建页面的中英文数据，目录结构如下：

![image-20220211103104133.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/254c77256292448caad47205be5b9d7f~tplv-k3u1fbpfcp-watermark.image?)

比如中文的`hello.json`文件内容如下（忽略笔者的低水平翻译~）：

![image-20220211103928440.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf479b9315954631b71330e6c87335e9~tplv-k3u1fbpfcp-watermark.image?)

在`index.js`文件里导入`hello.json`文件及`ElementUI`的语言文件，并合并导出：

```js
import hello from './hello.json'
import elementLocale from 'element-ui/lib/locale/lang/zh-CN'

export default {
    hello,
    ...elementLocale
}
```

为什么是`...elementLocale`呢，因为传给`Vue-i18n`的多语言数据结构是这样的：

![image-20220211170320562.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e80f61b677dd4422823068a6de899dd7~tplv-k3u1fbpfcp-watermark.image?)

我们是把`index.js`的整个导出对象作为`vue-i18n`的多语言数据的，而`ElementUI`的多语言文件是这样的：

![image-20220211165917570.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/874388091b61462ca628f26fecc204c9~tplv-k3u1fbpfcp-watermark.image?)

所以我们需要把这个对象的属性和`hello`属性合并到一个对象上。

接下来我们需要把它导出的数据到写到一个`json`文件里并输出到`public`目录下，这可以直接写个`js`脚本文件来做这个事情，但是为了和项目的源码分开我们写成一个`npm`包。

## 创建一个npm工具包

我们在项目的平级下创建一个包目录，并使用`npm init`初始化：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a63b1d985384fdcbe41f48c500c3ddc~tplv-k3u1fbpfcp-watermark.image?)

命名为`-tool`的原因是后续可能还会有类似编译多语言这种需求，所以取一个通用名字，方便后面增加其他功能。

命令行交互工具使用[Commander.js](https://github.com/tj/commander.js)，安装：

```bash
npm install commander
```

然后新建入口文件`index.js`：

```js
#!/usr/bin/env node

const {
    program
} = require('commander');

// 编译多语言文件
const buildI18n = () => {
    console.log('编译多语言文件');
}

program
    .command('i18n') // 添加i18n命令
    .action(buildI18n)

program.parse(process.argv);
```

因为我们的包是要作为命令行工具使用的，所以文件第一行需要指定脚本的解释程序为`node`，然后使用`commander`配置了一个`i18n`命令，用来编译多语言文件，后续如果要添加其他功能新增命令即可，执行文件有了，我们还要在包的`package.json`文件里添加一个`bin`字段，用来指示我们的包里有可执行文件，让`npm`在安装包的时候顺便给我们创建一个符号链接，把命令映射到文件。

```js
// hello-tool/package.json
{
    "bin": {
        "hello": "./index.js"
    }
}
```

因为我们的包还没有发布到`npm`，所以直接链接到项目上使用，先在`hello-tool`目录下执行：

```bash
npm link
```

然后到我们的`hello world`目录下执行：

```bash
npm link hello-tool
```

现在在命令行输入`hello i18n`试试：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75d70a61a59540d595d7696adb2979e3~tplv-k3u1fbpfcp-watermark.image?)

## 编译多语言文件

接下来完善`buildI18n`函数的逻辑，主要分三步：

1.清空目标目录，也就是`/public/i18n`目录

2.获取`/src/i18n`下的各种多语言文件导出的数据

3.写入到`json`文件并输出到`/public/i18n`目录下

代码如下：

```js
const path = require('path')
const fs = require('fs')
// 编译多语言文件
const buildI18n = () => {
    // 多语言源目录
    let srcDir = path.join(process.cwd(), 'src/i18n')
    // 目标目录
    let destDir = path.join(process.cwd(), 'public/i18n')
    // 1.清空目标目录，clearDir是一个自定义方法，递归遍历目录进行删除
    clearDir(destDir)
    // 2.获取源多语言导出数据
    let data = {}
    let langDirs = fs.readdirSync(srcDir)
    langDirs.forEach((dir) => {
        let dirPath = path.join(srcDir, dir)
        // 读取/src/i18n/xxx/index.js文件，获取导出的多语言对象，存储到data对象上
        let indexPath = path.join(dirPath, 'index.js')
        if (fs.statSync(dirPath).isDirectory() && fs.existsSync(indexPath)) {
            // 使用require加载该文件模块，获取导出的数据
            data[dir] = require(indexPath)
        }
    })
    // 3.写入到目标目录
    Object.keys(data).forEach((lang) => {
        // 创建public/i18n目录
        if (!fs.existsSync(destDir)) {
            fs.mkdirSync(destDir)
        }
        let dirPath = path.join(destDir, lang)
        let filePath = path.join(dirPath, 'index.json')
        // 创建多语言目录
        if (!fs.existsSync(dirPath)) {
            fs.mkdirSync(dirPath)
        }
        // 创建json文件
        fs.writeFileSync(filePath, JSON.stringify(data[lang], null, 4))
    })
    console.log('多语言编译完成');
}
```

代码很简单，接下来我们运行命令：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e94c4de91ba44eba7987234bfc304b7~tplv-k3u1fbpfcp-watermark.image?)


报错了，提示不能在模块外使用`import`，其实新版本的`nodejs`已经支持`ES6`的模块语法了，可以把文件后缀换成`.mjs`，或者在`package.json`文件里增加`type=module`字段，但是都要做很多修改，这咋办呢，有没有更简单的方法呢？把多语言文件换成`commonjs`模块语法？也可以，但是不太优雅，不过好在`babel`提供了一个[@babel/register](https://www.babeljs.cn/docs/babel-register)包，可以把`babel`绑定到`node`的`require`模块上，然后可以在运行时进行即时编译，也就是当`require('/src/i18n/xxx/index.js')`时会先由`babel`进行编译，编译完当然就不存在`import`语句了，先安装：

```bash
npm install @babel/core @babel/register @babel/preset-env
```

然后新建一个`babel`配置文件：

```js
// hello-tool/babel.config.js
module.exports = {
  'presets': ['@babel/preset-env']
}
```

最后在`hello-tool/index.js`文件里使用：

```js
const path = require('path')
const {
    program
} = require('commander');
const fs = require('fs')
require("@babel/register")({
    configFile: path.resolve(__dirname, './babel.config.js'),
})
// ...
```

接下来再次运行命令：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d29a912d15a4fd6bfdd6dfa023a2df9~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2dd3b5205ccc490b97baf517fb03a2ad~tplv-k3u1fbpfcp-watermark.image?)

可以看到编译完成了，文件也输出到了`public`目录下，但是`json`文件里存在一个`default`属性，这一层显然我们是不需要的，所以`require('i18n/xxx/index.js')`时我们存储导出的`default`对象即可，修改`hello-tool/index.js`：

```js
const buildI18n = () => {
    // ...
    langDirs.forEach((dir) => {
        let dirPath = path.join(srcDir, dir)
        let indexPath = path.join(dirPath, 'index.js')
        if (fs.statSync(dirPath).isDirectory() && fs.existsSync(indexPath)) {
            data[dir] = require(indexPath).default// ++
        }
    })
    // ...
}
```

效果如下：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4148ee50488a48f096e52cba05e00815~tplv-k3u1fbpfcp-watermark.image?)

## 使用多语言文件

首先修改一下用户接口的返回数据，增加默认语言字段：

```js
// /src/store.js
// ...
async getUserInfo(ctx) {
    let userInfo = {
        // ...
        language: 'zh_CN'// 默认语言
    }
    ctx.commit('setUserInfo', userInfo)
}
// ...
```

然后在`main.js`里面获取完用户信息后立刻请求并设置多语言：

```js
// /src/main.js
import { setLanguage } from './utils'// ++
import i18n from './i18n'// ++

const initApp = async () => {
  await store.dispatch('getUserInfo')
  await setLanguage(store.state.userInfo.language)// ++
  new Vue({
    i18n,// ++
    router,
    store,
    render: h => h(App),
  }).$mount('#app')
}
```

`setLanguage`方法会请求多语言文件并切换：

```js
// /src/utils/index.js
import axios from 'axios'
import i18n from '../i18n'

// 请求并设置多语言数据
const languageCache = {}
export const setLanguage = async (language = 'zh_CN') => {
    let languageData = null
    // 有缓存，使用缓存数据
    if (languageCache[language]) {
        languageData = languageCache[language]
    } else {
        // 没有缓存，发起请求
        const {
            data
        } = await axios.get(`/i18n/${language}/index.json`)
        languageCache[language] = languageData = data
    }
    // 设置语言环境的 locale 信息
    i18n.setLocaleMessage(language, languageData)
    // 修改语言环境
    i18n.locale = language
}
```

然后把各个组件里显示的信息都换成`$t('xxx')`形式，当然，菜单和路由都需要做相应的修改，效果如下：

![2022-02-12-11-01-36.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6af6a0ab571f4e618f4ac92165e3c059~tplv-k3u1fbpfcp-watermark.image?)

可以发现`ElementUI`组件的语言并没有变化，这是当然的，因为我们还没有处理它，修改很简单，`ElementUI`支持自定义`i18n`的处理方法：

```js
// /src/main.js
// ...
Vue.use(ElementUI, {
  i18n: (key, value) => i18n.t(key, value)
})
// ...
```


![image-20220212111252574.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18d67a4f51e84bc6a194b7fcf6ead11c~tplv-k3u1fbpfcp-watermark.image?)

## 通过CLI插件生成初始多语言文件

最后还有一个问题，就是项目初始化时还没有多语言文件怎么办，难道项目创建完还要先手动运行命令编译一下多语言？有几种解决方法：

1.最终一般会提供一个项目脚手架，所以默认的模板里我们就可以直接加上初始的多语言文件；

2.启动服务和打包时先编译一下多语言文件，像这样：

```json
"scripts": {
    "serve": "hello i18n && vue-cli-service serve",
    "build": "hello i18n && vue-cli-service build"
  }
```

3.开发一个`Vue CLI`插件来帮我们在项目创建完时自动运行一次多语言编译命令；

接下来简单实现一下第三种方式，同样在项目同级新建一个插件目录，并创建相应的文件（注意插件的命名规范）：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfac5d2c70a44a18b5740db873c8f02d~tplv-k3u1fbpfcp-watermark.image?)

根据插件开发规范，`index.js`为`Service`插件的入口文件，`Service`插件可以修改`webpack`配置，创建新的 `vue-cli service`命令或者修改已经存在的命令，我们用不上，我们的逻辑在`generator.js`里，这个文件会在两个场景被调用：

1.项目创建期间，`CLI`插件被作为项目创建`preset`的一部分被安装时

2.项目创建完成时通过`vue add`或`vue invoke`单独安装插件时调用

我们需要的刚好是在项目创建时或安装该插件时自动帮我们运行多语言编译命令，`generator.js`需要导出一个函数，内容如下：

```js
const {
    exec
} = require('child_process');

module.exports = (api) => {
    // 为了方便在项目里看到编译多语言的命令，我们把hello i18n添加到项目的package.json文件里，修改package.json文件可以使用提供的api.extendPackage方法
    api.extendPackage({
        scripts: {
            buildI18n: 'hello i18n'
        }
    })
    // 该钩子会在文件写入硬盘后调用
    api.afterInvoke(() => {
        // 获取项目的完整路径
        let targetDir = api.generator.context
        // 进入项目文件夹，然后运行命令
        exec(`cd ${targetDir} && npm run buildI18n`, (error, stdout, stderr) => {
            if (error) {
                console.error(error);
                return;
            }
            console.log(stdout);
            console.error(stderr);
        });
    })
}
```

我们在`afterInvoke`钩子里运行编译命令，因为太早运行可能依赖都还没有安装完成，另外我们还获取了项目的完整路径，这是因为通过`preset`配置插件时，插件被调用时可能不在实际的项目文件夹，比如我们在`a`文件夹下通过该命令创建`b`项目：

```bash
vue create b
```

插件被调用时是在`a`目录，显然`hello-i18n`包是被安装在`b`目录，所以我们要先进入项目实际目录然后运行编译命令。

接下来测试一下，先在项目下安装该插件：

```bash
npm install --save-dev file:完整路径\vue-cli-plugin-i18n
```

然后通过如下命令来调用插件的生成器：

```bash
vue invoke vue-cli-plugin-i18n
```

效果如下：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/567d39b506d14133a4078c5913540154~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98815d75714c435db7717b273b0c8bc9~tplv-k3u1fbpfcp-watermark.image?)

可以看到项目的`package.json`文件里面已经注入了编译命令，并且命令也自动执行生成了多语言文件。

# Mock数据

`Mock`数据推荐使用[Mock](https://github.com/nuysoft/Mock)，使用很简单，新建一个`mock`数据文件：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/895231962cd442ba8b28d7c9bd2ec173~tplv-k3u1fbpfcp-watermark.image?)

然后在`/api/index.js`里引入：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29d9035d09664a1a8ce0bdca42b61561~tplv-k3u1fbpfcp-watermark.image?)

就这么简单，该请求即可被拦截：

![image-20220212150450209.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e64a5747fb074b56ae02a65f34c68fcb~tplv-k3u1fbpfcp-watermark.image?)


# 规范化

有关规范化的配置，比如代码风格检查、`git`提交规范等，笔者之前写过一篇组件库搭建的文章，其中一个小节详细的介绍了配置过程，可移步：[【万字长文】从零配置一个vue组件库-规范化配置小节](https://juejin.cn/post/6946171543013556255#heading-2)。

# 其他

## 请求代理

本地开发测试接口请求时难免会遇到跨域问题，可以配置一下`webpack-dev-server`的代理选项，新建`vue.config.js`文件：

```js
module.exports = {
    devServer: {
        proxy: {
            '^/api/': {
                target: 'http://xxx:xxx',
                changeOrigin: true
            }
        }
    }
}
```

## 编译node_modules内的依赖

默认情况下`babel-loader`会忽略所有`node_modules`中的文件，但是有些依赖可能是没有经过编译的，比如我们自己编写的一些包为了省事就不编译了，那么如果用了最新的语法，在低版本浏览器上可能就无法运行了，所以打包的时候也需要对它们进行编译，要通过`Babel`显式转译一个依赖，可以在这个`transpileDependencies`选项配置，修改`vue.config.js`：

```js
module.exports = {
    // ...
    transpileDependencies: ['your-package-name']
}
```

## 环境变量

需要环境变量可以在项目根目录下新建`.env`文件，需要注意的是如果要通过插件渲染`.`开头的模板文件，要用`_`来替代点，也就是`_env`，最终会渲染为`.`开头的文件。

# 脚手架

当我们设计好了一套项目结构后，肯定是作为模板来快速创建项目的，一般会创建一个脚手架工具来生成，但是`Vue CLI`提供了`preset`（预设）的能力，所谓`preset`指的是一个包含创建新项目所需预定义选项和插件的 `JSON`对象，所以我们可以创建一个`CLI`插件来创建模板，然后创建一个`preset`，再把这个插件配置到`preset`里，这样使用`vue create`命令创建项目时使用我们的自定义`preset`即可。

## 创建一个生成模板的CLI插件

新建插件目录如下：

![image-20220212162638048.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb9221d7c2db47f592d5c35b13bde9d4~tplv-k3u1fbpfcp-watermark.image?)

可以看到这次我们创建了一个`generator`目录，因为我们需要渲染模板，而模板文件就会放在这个目录下，新建一个`template`目录，然后把我们前文配置的项目结构完整的复制进去（不包括package.json）：



![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e563ed4c7e948bfa7cb3d8e65c1d687~tplv-k3u1fbpfcp-watermark.image?)

现在我们来完成`/generator/index.js`文件的内容：

1.因为不包括`package.json`，所以我们要修改`vue`项目默认的`package.json`，添加我们需要的东西，使用的就是前面提到的`api.extendPackage`方法：

```js
// generator/index.js

module.exports = (api) => {
    // 扩展package.json
    api.extendPackage({
        "dependencies": {
            "axios": "^0.25.0",
            "element-ui": "^2.15.6",
            "vue-i18n": "^8.27.0",
            "vue-router": "^3.5.3",
            "vuex": "^3.6.2"
        },
        "devDependencies": {
            "mockjs": "^1.1.0",
            "sass": "^1.49.7",
            "sass-loader": "^8.0.2",
            "hello-tool": "^1.0.0"// 注意这里，不要忘记把我们的工具包加上
        }
    })
}
```

添加了一些额外的依赖，包括我们前面开发的`hello-tool`。

2.渲染模板

```js
module.exports = (api) => {
    // ...
    api.render('./template')
}
```

`render`方法会渲染`template`目录下的所有文件。



## 创建一个自定义preset

插件都有了，最后让我们来创建一下自定义`preset`，新建一个`preset.json`文件，把我们前面写的`template`插件和`i18n`插件一起配置进去：

```json
{
    "plugins": {
        "vue-cli-plugin-template": {
            "version": "^1.0.0"
        },
        "vue-cli-plugin-i18n": {
            "version": "^1.0.0"
        }
    }
}
```

同时为了测试这个`preset`，我们再创建一个空目录：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8416f3cfa1bd474f95ee6b8ff3adfb9a~tplv-k3u1fbpfcp-watermark.image?)

然后进入`test-preset`目录运行`vue create`命令时指定我们的`preset`路径即可：

```bash
vue create --preset ../preset.json my-project
```

效果如下：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6318f16586994369a3cc80cf51d6bb3e~tplv-k3u1fbpfcp-watermark.image?)



![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74896a683c9e425ea5b8bf1eb2636b87~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b6820d4161e474b92d8d19f599c5d6a~tplv-k3u1fbpfcp-watermark.image?)

## 远程使用preset

`preset`本地测试没问题了就可以上传到仓库里，之后就可以给别人使用了，比如笔者上传到了这个仓库：[https://github.com/wanglin2/Vue_project_design](https://github.com/wanglin2/Vue_project_design)，那么你可以这么使用：

```bash
vue create --preset wanglin2/Vue_project_design project-name
```

# 总结

如果有哪里不对的或是更好的，评论区见~
