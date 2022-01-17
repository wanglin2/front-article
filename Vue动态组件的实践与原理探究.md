我司有一个工作台搭建产品，允许通过拖拽小部件的方式来搭建一个工作台页面，平台内置了一些常用小部件，另外也允许自行开发小部件上传使用，本文会从实践的角度来介绍其实现原理。

> ps.本文项目使用`Vue CLI`创建，所用的`Vue`版本为`2.6.11`，`webpack`版本为`4.46.0`。



# 创建项目

首先使用`Vue CLI`创建一个项目，在`src`目录下新建一个`widgets`目录用来存放小部件：

![image-20211228135808675.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c26ce23d38ca405280df3ecd8efd8393~tplv-k3u1fbpfcp-watermark.image?)

一个小部件由一个`Vue`单文件和一个`js`文件组成：

![image-20211228135933206.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8af86fbee174079a719f420c82abb5c~tplv-k3u1fbpfcp-watermark.image?)

测试组件`index.vue`的内容如下：

```html
<template>
  <div class="countBox">
    <div class="count">{{ count }}</div>
    <div class="btn">
      <button @click="add">+1</button>
      <button @click="sub">-1</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'count',
  data() {
    return {
      count: 0,
    }
  },
  methods: {
    add() {
      this.count++
    },
    sub() {
      this.count--
    },
  },
}
</script>

<style lang="less" scoped>
.countBox {
  display: flex;
  flex-direction: column;
  align-items: center;

  .count {
    color: red;
  }
}
</style>
```

一个十分简单的计数器。

`index.js`用来导出组件：

```js
import Widget from './index.vue'

export default Widget

const config = {
    color: 'red'
}

export {
    config
}
```

除了导出组件，也支持导出配置。

项目的`App.vue`组件我们用来作为小部件的开发预览和测试，效果如下：

![image-20211228141656015.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/340549be1d2444b2994dba51aa72213d~tplv-k3u1fbpfcp-watermark.image?)

小部件的配置会影响包裹小部件容器的边框颜色。



# 打包小部件

假设我们的小部件已经开发完成了，那么接下来我们需要进行打包，把`Vue`单文件编译成`js`文件，打包使用的是`webpack`，首先创建一个`webpack`配置文件：

![image-20211228145100423.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd993bc56f9e432eb4413bd901ac49b2~tplv-k3u1fbpfcp-watermark.image?)

`webpack`的常用配置项为：`entry`、`output`、`module`、`plugins`，我们一一来看。

1.`entry`入口

入口显然就是各个小部件目录下的`index.js`文件，因为小部件数量是不定的，可能会越来越多，所以入口不能写死，需要动态生成：

```js
const path = require('path')
const fs = require('fs')

const getEntry = () => {
    let res = {}
    let files = fs.readdirSync(__dirname)
    files.forEach((filename) => {
        // 是否是目录
        let dir = path.join(__dirname, filename)
        let isDir = fs.statSync(dir).isDirectory
        // 入口文件是否存在
        let entryFile = path.join(dir, 'index.js')
        let entryExist = fs.existsSync(entryFile)
        if (isDir && entryExist) {
            res[filename] = entryFile
        }
    })
    return res
}

module.exports = {
    entry: getEntry()
}
```

2.`output`输出

因为我们开发完后还要进行测试，所以便于请求打包后的文件，我们把小部件的打包结果直接输出到`public`目录下：

```js
module.exports = {
    // ...
    output: {
        path: path.join(__dirname, '../../public/widgets'),
        filename: '[name].js'
    }
}
```

3.`module`模块

这里我们要配置的是`loader`规则：

- 处理`Vue`单文件我们需要`vue-loader`

- 编译`js`最新语法需要`babel-loader`
- 处理`less`需要`less-loader`

因为`vue-loader`和`babel-loader`相关的包`Vue`项目本身就已经安装了，所以不需要我们手动再安装，装一下处理`less`文件的`loader`即可：

```bash
npm i less less-loader -D
```

不同版本的`less-loader`对`webpack`的版本也有要求，如果安装出错了可以指定安装支持当前`webpack`版本的`less-loader`版本。

修改配置文件如下：

```js
module.exports = {
    // ...
    module: {
        rules: [
            {
                test: /\.vue$/,
                loader: 'vue-loader'
            },
            {
                test: /\.js$/,
                loader: 'babel-loader'
            },
            {
                test: /\.less$/,
                loader: [
                    'vue-style-loader',
                    'css-loader',
                    'less-loader'
                ]
            }
        ]
    }
}
```

4.`plugins` 插件

插件我们就使用两个，一个是`vue-loader`指定的，另一个是用来清空输出目录的：

```bash
npm i clean-webpack-plugin -D
```

修改配置文件如下：

```js
const { VueLoaderPlugin } = require('vue-loader')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

module.exports = {
    // ...
    plugins: [
        new VueLoaderPlugin(),
        new CleanWebpackPlugin()
    ]
}
```

`webpack`的配置就写到这里，接下来写打包的脚本文件：

![image-20211228152656241.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0c7022f6810475e882a41bfe761e39e~tplv-k3u1fbpfcp-watermark.image?)

我们通过`api`的方式来使用`webpack`：

```js
const webpack = require('webpack')
const config = require('./webpack.config')

webpack(config, (err, stats) => {
    if (err || stats.hasErrors()) {
        // 在这里处理错误
        console.error(err);
    }
    // 处理完成
    console.log('打包完成');
});
```

现在我们就可以在命令行输入`node src/widgets/build.js`进行打包了，嫌麻烦的话也可以配置到`package.json`文件里：

```json
{
    "scripts": {
        "build-widgets": "node src/widgets/build.js"
    }
}
```

运行完后可以看到打包结果已经有了：

![image-20211228153736948.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8312faae6554b3986c1f6a67d0cf709~tplv-k3u1fbpfcp-watermark.image?)

# 使用小部件

我们的需求是线上动态的请求小部件的文件，然后将小部件渲染出来。请求使用`ajax`获取小部件的`js`文件内容，渲染我们的第一想法是使用`Vue.component()`方法进行注册，但是这样是不行的，因为全局注册组件必须在根`Vue`实例创建之前发生。

所以这里我们使用的是`component`组件，`Vue`的`component`组件可以接受以注册组件的名字或一个组件的选项对象，刚好我们可以提供小部件的选项对象。

请求`js`资源我们使用`axios`，获取到的是`js`字符串，然后使用`new Function`动态进行执行获取导出的选项对象：

```js
// 点击加载按钮后调用该方法
async load() {
    try {
        let { data } = await axios.get('/widgets/Count.js')
        let run = new Function(`return ${data}`)
        let res = run()
        console.log(res)
    } catch (error) {
        console.error(error)
    }
}
```

正常来说我们能获取到导出的模块，可是居然报错了！

![image-20211228181924164.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3372fdc8e8354065b2793ca500f40b99~tplv-k3u1fbpfcp-watermark.image?)

说实话，笔者看不懂这是啥错，百度了一下也无果，但是经过一番尝试，发现把项目的`babel.config.js`里的预设由`@vue/cli-plugin-babel/preset`修改为`@babel/preset-env`后可以了，具体是为啥呢，反正我也不知道，当然，只使用`@babel/preset-env`可能是不够的，这就需要你根据实际情况再调整了。

不过后来笔者阅读`Vue CLI`官方文档时看到了下面这段话：

![image-20211228192729057.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad9f7de9e32e4482a3d2f89a6ea62de1~tplv-k3u1fbpfcp-watermark.image?)

直觉告诉我，肯定就是这个问题导致的了，于是把`vue.config.js`修改为如下：

```js
module.exports = {
  presets: [
    ['@vue/cli-plugin-babel/preset', {
      useBuiltIns: false
    }]
  ]
}
```

然后打包，果然一切正常（多看文档准没错），但是每次打包都要手动修改`babel.config.js`文件总不是一件优雅的事情，我们可以通过脚本在打包前修改，打包完后恢复，修改`build.js`文件：

```js
const path = require('path')
const fs = require('fs')

// babel.config.js文件路径
const babelConfigPath = path.join(__dirname, '../../babel.config.js')
// 缓存原本的配置
let originBabelConfig = ''

// 修改配置
const changeBabelConfig = () => {
    // 保存原本的配置
    originBabelConfig = fs.readFileSync(babelConfigPath, {
        encoding: 'utf-8'
    })
    // 写入新配置
    fs.writeFileSync(babelConfigPath, `
        module.exports = {
            presets: [
                ['@vue/cli-plugin-babel/preset', {
                    useBuiltIns: false
                }]
            ]
        }
    `)
}

// 恢复为原本的配置
const resetBabelConfig = () => {
    fs.writeFileSync(babelConfigPath, originBabelConfig)
}

// 打包前修改
changeBabelConfig()
webpack(config, (err, stats) => {
    // 打包后恢复
    resetBabelConfig()
    if (err || stats.hasErrors()) {
        console.error(err);
    }
    console.log('打包完成');
});
```

几行代码解放双手。现在来看看我们最后获取到的小部件导出数据：

![image-20211228182743099.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee67f95cf0f94a17b15ffa4ef63adbed~tplv-k3u1fbpfcp-watermark.image?)

小部件的选项对象有了，接下来把它扔给`component`组件即可：

```html
<div class="widgetWrap" v-if="widgetData" :style="{ borderColor: widgetConfig.color }">
    <component :is="widgetData"></component>
</div>
```

```js
export default {
    data() {
        return {
            widgetData: null,
            widgetConfig: null
        }
    },
    methods: {
        async load() {
            try {
                let { data } = await axios.get('/widgets/Count.js')
                let run = new Function(`return ${data}`)
                let res = run()
                this.widgetData = res.default
                this.widgetConfig = res.config
            } catch (error) {
                console.error(error)
            }
        }
    }
}
```

效果如下：

![2021-12-28-18-45-52.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2eb4e7d5f0045c4bdddc39f7df12442~tplv-k3u1fbpfcp-watermark.image?)

是不是很简单。



# 深入component组件

最后让我们从源码的角度来看看`component`组件是如何工作的，先来看看对于`component`组件最后生成的渲染函数长啥样：

![image-20211229135411191.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5898a68a576c44eca9828d52641784cf~tplv-k3u1fbpfcp-watermark.image?)

`_c`即`createElement`方法：

```js
vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
```

```js
function createElement (
  context,// 上下文，即父组件实例，即App组件实例
  tag,// 我们的动态组件Count的选项对象
  data,// {tag: 'component'}
  children,
  normalizationType,
  alwaysNormalize
) {
  // ...
  return _createElement(context, tag, data, children, normalizationType)
}
```

忽略了一些没有进入的分支，直接进入`_createElement`方法：

```js
function _createElement (
 context,
 tag,
 data,
 children,
 normalizationType
) {
    // ...
    var vnode, ns;
    if (typeof tag === 'string') {
        // ...
    } else {
        // 组件选项对象或构造函数
        vnode = createComponent(tag, data, context, children);
    }
    // ...
}
```

`tag`是个对象，所以会进入`else`分支，即执行`createComponent`方法：

```js
function createComponent (
 Ctor,
 data,
 context,
 children,
 tag
) {
    // ...
    var baseCtor = context.$options._base;

    // 选项对象: 转换成构造函数
    if (isObject(Ctor)) {
        Ctor = baseCtor.extend(Ctor);
    }
    // ...
}
```

`baseCtor`为`Vue`构造函数，`Ctor`即`Count`组件的选项对象，所以实际执行了`Vue.extend()`方法：

![image-20211229142121211.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be8f09bdc97e467b86df926f152d8954~tplv-k3u1fbpfcp-watermark.image?)

这个方法实际上就是以`Vue`为父类创建了一个子类：

![image-20211229142346598.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3a32fc9089b4c2aa1e1f992bfb5dcca~tplv-k3u1fbpfcp-watermark.image?)

继续看`createComponent`方法：

```js
// ...
// 返回一个占位符节点
var name = Ctor.options.name || tag;
var vnode = new VNode(
    ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
    data, undefined, undefined, undefined, context,
    { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
    asyncFactory
);

return vnode
```

最后创建了一个占位`VNode`：

![image-20211229142925592.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28b4a0079db24d799f1be1e7bec38c8c~tplv-k3u1fbpfcp-watermark.image?)

`createElement`方法最后会返回创建的这个`VNode`，渲染函数执行完生成了`VNode`树，下一步会将虚拟`DOM`树转换成真实的`DOM`，这一阶段没有必要再去看，因为到这里我们已经能发现在编译后，也就是将模板编译成渲染函数这个阶段，`component`组件就已经被处理完了，得到了下面这个创建`VNode`的方法：

```js
_c(_vm.widgetData,{tag:"component"})
```

如果我们传给`component`的`is`属性是一个组件的名称，那么在`createElement`方法里就会走下图的第一个`if`分支：

![image-20211229143615082.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/efd586a089674920b169757730b39c73~tplv-k3u1fbpfcp-watermark.image?)

也就是我们普通注册的组件会走的分支，如果我们传给`is`的是选项对象，相对于普通组件，其实就是少了一个根据组件名称查找选项对象的过程，其他和普通组件没有任何区别，至于模板编译阶段对它的处理也十分简单：

![image-20211229150134553.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b8ab16460df4d4199d6450a6bdf1820~tplv-k3u1fbpfcp-watermark.image?)

直接取出`is`属性的值保存到`component`属性上，最后在生成渲染函数的阶段：

![image-20211229150410598.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e44a7965ba08473ea428b26ddc86cc5d~tplv-k3u1fbpfcp-watermark.image?)

![image-20211229150453947.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c03b6478abd4b1d8615b57c88595a9e~tplv-k3u1fbpfcp-watermark.image?)

这样就得到了最后生成的渲染函数。

