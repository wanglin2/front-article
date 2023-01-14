> 本来只是好奇打包工具是如何转换ESM和CJS模块的，没想到带着这个问题阅读完编译的代码后，我的问题更多了。

目前主流的有两种模块语法，一是`Node.js`专用的`CJS`，另一种是浏览器和`Node.js`都支持的`ESM`，在`ESM`规范没有出来之前，`Node.js`的模块编写使用的都是`CJS`，但是现在`ESM`已经逐渐在替代`CJS`成为浏览器和服务器通用的模块解决方案。

那么问题来了，比如说我早期开发了一个`CJS`的包，现在想把它转成`ESM`语法用来支持在浏览器端使用，或者现在使用`ESM`开发的一个包，想转换成`CJS`语法用来支持老版的`Node.js`，转换工具有很多，比如`Webpack`、`esbuild`等，那么你有没有仔细看过它们的转换结果都是什么样的，没有没关系，本文就来一探究竟。

# ESM模块语法

先来简单过一下常用的`ESM`模块语法。

导出：

```js
// esm.js
export let name1 = '周杰伦'
// 等同于
let name2 = '朴树'
export {
	name2
}
// 重命名
export {
	name1 as name3
}
// 默认导出
// 一个模块只能有一个默认输出，因此export default命令只能使用一次
// 本质上，export default就是输出一个叫做default的变量或方法，所以可以直接一个值，导入时可以使用任意名称
export default '华语乐坛经典人物'
```

导入：

```js
// 具名导入
import title, { name1, name2, name3, name1 as name4 } from './esm.js';
// 整体导入
import title, * as names from './esm.js';
```



# CJS模块语法

`CJS`模块语法会更简单一点，导出：

```js
// 方式一
exports.name2 = '朴树'
// 等同于
module.exports.name1 = '周杰伦'

// 方式二
module.exports = {
    name1: '周杰伦',
    name2: '朴树'
}
```

导入：

```js
// 整体
const names = require('./cjs.js')
console.log(names)
// 解构
const { name1, name2 } = require('./cjs.js')
console.log(name1, name2)
```



从我们肉眼观察的结果，`CJS`的`exports.xxx`类似于`ESM`的`export let xxx`，`CJS`的`module.exports = xxx`类似于`ESM`的`export default xxx`，但是它们的导入形式是有所不同的，`ESM`的`import xxx`的`xxx`代表的只是`export default xxx`的值，如果没有默认导出，这样导入是会报错的，需要使用`import * as xxx`语法，但是`CJS`其实无论使用的是`exports.xxx = `还是`module.exports =`，实际上导出的都是`module.exports`这个属性最终的值，所以导入的也只是这个属性的值。

实际上，`CJS`和`ESM`有三个重大的差异：

- `CJS` 模块输出的是一个值的拷贝，`ESM` 模块输出的是值的引用
- `CJS` 模块是运行时加载，`ESM` 模块是编译时输出接口
- `CJS` 模块的`require()`是同步加载模块，`ESM` 模块的`import`命令是异步加载，有一个独立的模块依赖的解析阶段

那么，在它们两者互相转换的过程中，是如何处理这些差异的呢，接下来我们使用[esbuild](https://esbuild.github.io/)来进行转换，为什么不用`webpack`呢，无他，唯简单尔，看看它是如何处理的，安装：

```bash
npm install esbuild
```

增加一个执行转换的文件：

```js
// build.js
require("esbuild").buildSync({
  entryPoints: [''],// 待转换的文件
  outfile: "out.js",
  format: '',// 转换的目标格式，cjs、esm
});
```

然后我们在命令行输入`node ./build.js`命令即可看到转换结果被输出在`out.js`文件内。



# ESM转CJS

## 转换导出

待转换的内容：

```js
export let name1 = '周杰伦'
let name2 = '朴树'
export {
	name2
}
export {
	name1 as name3
}
export default '华语乐坛经典人物'
```

接下来看一下转换结果的代码，核心的导出语句如下：

```js
module.exports = __toCommonJS(esm_exports);
```

导出的数据是调用`__toCommonJS`方法返回的结果，先来看看参数`esm_exports`：

```js
var esm_exports = {};
__export(esm_exports, {
  default: () => esm_default,
  name1: () => name1,
  name2: () => name2,
  name3: () => name1
});
let name1 = "周杰伦";
let name2 = "朴树";
var esm_default = "华语乐坛经典人物";
```

先定义了一个空对象`esm_exports`，然后调用了`__export`方法：

```js
var __defProp = Object.defineProperty;
var __export = (target, all) => {
  // 遍历对象
  for (var name in all)
    // 给对象添加一个属性，并设置属性描述符的取值函数get为all对象上该属性对应的函数，那么该属性的值也就是该函数的返回值
    __defProp(target, name, { get: all[name], enumerable: true });
};
```

上面所做的事情就是给`esm_exports`对象添加了四个属性，这四个属性很明显就是我们使用`ESM`的`export`导出的所有变量，`export default`默认导出，本质上就是导出了一个叫做`default`的变量而已，没有什么特别的：

```js
export default a
// 等同于
export {
	a as default
}
```

所以默认导出的变量会定义成名为`default`的属性添加到这个对象上，这很明显，因为我们知道`CJS`的导出其实是`module.exports`属性的值，那么我们使用`ESM`导出了多个变量，只能都添加到一个对象上来导出，注意看其中两点：

1.添加属性没有直接使用`esm_exports.xxx`的方式来添加，而是使用`Object.defineProperty`方法，并且只给属性定义了取值函数`get`，没有定义赋值函数`set`，这意味着`esm_exports`的这个属性的值是不能被修改的，这其实是`CommonJS`和`ESM`的一个不同点：`ESM`导出的接口不能修改，而`CJS`可以。

所以下面这些`ESM`做法都是会报错的：

```js
import * as names from './esm.js';
names.name1 = '许巍';// 报错

import title, { name1, name2, name3, name1 as name4 } from './esm.js';
title = '许巍';// 报错
name1 = '许巍';// 报错
```

而`CJS`不会：

```js
const names = require('./cjs.js');
names.name1 = 1;// 成功

let { name1, name2 } = require("./cjs.js");
name1 = 1;// 成功
```

2.设置属性的描述符时没有直接使用`value`，比如：

```js
var __export = (target, all) => {
  for (var name in all)
    __defProp(target, name, { value: all[name], enumerable: true });
};

__export(esm_exports, {
  default: esm_default,
  name1: name1,
  name2: name2,
  name3: name1,
  setName1: setName1
});
```

而是定义了`get`取值函数，通过函数的形式返回同名变量的值，这其实又是一个不同点了：`CJS` 模块输出的是一个值的拷贝，`ESM` 模块输出的是值的引用。

比如在`ESM`模块中：

```js
// esm.js
export let name = '周杰伦'
export const setName = (newName) => {
    name = newName
}

// other.js
import { name, setName } form './esm.js'
console.log(name)// 周杰伦
setName('许巍')
console.log(name)// 许巍
```

可以看到导入地方的值也跟着变了，但是在`CJS`模块中就不会：

```js
// cjs.js
let name = '周杰伦'
const setName = (newName) => {
    name = newName
}
module.exports = {
    name,
    setName
}

// other.js
let { name, setName } = require("./cjs.js")
console.log(name)// 周杰伦
setName('许巍')
console.log(name)// 周杰伦
```

正是如此，所以才需要通过设置`get`函数来实时取值，否则转换成`CJS`后，变量的值只拷贝了一份，后续变化了都不会再更新。

回到这行：

```js
module.exports = __toCommonJS(esm_exports);
```

看完了`esm_exports`，接下来看看`__toCommonJS`方法：

```js
var __toCommonJS = (mod) => __copyProps(__defProp({}, "__esModule", { value: true }), mod);
```

首先创建了一个空对象，然后使用`Object.defineProperty`添加了一个`__esModule=true`的属性，这个属性是用于在导入的时候进行一些判断的，接下来调用了`__copyProps`方法：

```js
var __getOwnPropNames = Object.getOwnPropertyNames;// 返回一个指定对象的所有自身属性的属性名（包括不可枚举属性但不包括 Symbol 值作为名称的属性）组成的数组
var __hasOwnProp = Object.prototype.hasOwnProperty;// 返回一个布尔值，指示对象自身属性中是否具有指定的属性（也就是，是否有指定的键），该方法会忽略掉那些从原型链上继承到的属性
var __getOwnPropDesc = Object.getOwnPropertyDescriptor;// 返回指定对象上一个自有属性对应的属性描述符。（自有属性指的是直接赋予该对象的属性，不需要从原型链上进行查找的属性）

var __copyProps = (to, from, except, desc) => {
  if (from && typeof from === "object" || typeof from === "function") {
    for (let key of __getOwnPropNames(from))
      if (!__hasOwnProp.call(to, key) && key !== except)
        __defProp(to, key, { get: () => from[key], enumerable: !(desc = __getOwnPropDesc(from, key)) || desc.enumerable });
  }
  return to;
};
```

这个方法做的事情是把`from`对象的所有属性都在`to`对象上添加一份，不过如果`to`对象上存在同名属性则不会覆盖，会发生在如下这种情况：

```js
// cjs.js
export let foo = 1

// cjsUse.js
export * from './cjs.js'

export let foo = 2
```

存在同名导出，`cjsUse`模块会覆盖`cjs`模块的同名导出，所以最终导出的`foo=2`。

同时会设置新添加属性的属性描述符，设置取值函数`get`，返回值为`from`对象的该属性值，因为没有设置`get`，所以添加的属性值也是不能被修改的。

简单来说就是创建了一个新对象，把`esm_exports`的属性都添加到新对象上，但是访问该新对象的属性时实际上最终访问的还是`from`对象的该属性值，相对于一个代理对象，然后对外导出该新对象。

> 百思不得解啊1：为啥要创建一个新对象，而不是直接导出`esm_exports`对象呢？

另外我们可以发现，`ESM`的默认导出`CJS`是不支持的，在`ESM`中默认导出我们可以这么导入：

```js
import defaultValue from 'xxx'
```

但是转成`CJS`后不能这样导入：

```js
const defaultValue = require('xxx')
```

而是需要通过`.default`的形式才能获取到真正的`defaultValue`：

```js
const importData = require('xxx')
console.log(importData.default)
```

所以可以的话还是尽量少用默认导出吧。

## 转换导入

接下来看看导入的转换：

```js
import title, { name1, name2, name3, name1 as name4 } from "./esm.js";
console.log(title, name1, name2, name3, name4);
```

转换结果：

```js
var import_esm = __toESM(require("./esm.js"));
console.log(import_esm.default, import_esm.name1, import_esm.name2, import_esm.name3, import_esm.name1);
```

对导入的数据调用了`__toESM`方法：

```js
var __create = Object.create;// 创建一个新对象，使用现有的对象来作为新创建对象的原型（prototype）
var __defProp = Object.defineProperty;
var __getProtoOf = Object.getPrototypeOf;// 返回指定对象的原型（内部[[Prototype]]属性的值）

var __toESM = (mod, isNodeMode, target) => (
  // 导入的模块存在，则使用该模块的原型为原型创建一个新对象作为target
  (target = mod != null ? __create(__getProtoOf(mod)) : {}),
  // 将导入模块的属性拷贝到target对象上
  __copyProps(
    isNodeMode || !mod || !mod.__esModule
      ? __defProp(target, "default", { value: mod, enumerable: true })
      : target,
    mod
  )
);
```

> 百思不得解啊2：为啥要以导入模块的原型为原型来创建一个新对象呢？

> 百思不得解啊3：为啥导入也要创建一个新对象？

可以看到也创建了一个新对象，然后把导入模块的属性添加到这个新对象上，前面在转换导出的时候会给导出的对象添加一个`__esModule=true`的属性，这里就用到了，为`true`就代表该模块是`ESM`转换而成的`CJS`模块，否则就是原始的`CJS`模块，这样的话会给`target`对象添加一个`default`属性，值就是导入的数据，这是为啥呢，其实是为了兼容导入原始的`CJS`模块，比如：

```js
// 导出
export default class Person {}
// 导入
import Person from 'x'
new Person()
```

转换成`CJS`以后：

```js
// 导出
module.exports = {
    default: () => Person
}
// 导入
const res = require('x')
new res.default()
```

但是如果`x`模块不是由`ESM`转换而来的，本身就是一个`CJS`模块：

```js
module.exports = Person
```

那么`res`就是导出的类，再获取它的`default`属性显然是不对的，所以需要手动创建一个对象，并添加一个`default`属性来引用。



# CJS转ESM

## 转换导出

待转换的内容如下：

```js
module.exports.name1 = '周杰伦'
exports.name2 = '朴树'
```

转换结果如下：

```js
// ...
export default require_cjs();
```

为什么要转换成默认导出而不是具名导出呢，一是因为`require`本身就很类似`import xxx`默认导入语法，二是转成具名导出不方便，比如如下导出：

```js
const res = {
    name1: '周杰伦'
}
module.exports = res
if (Math.random() > 0.5) {
    res.name2 = '许巍'
} else {
    res.name3 = '朴树'
}
```

不实际执行代码压根不知道最终导出的是啥，所以具名导出就不可能，只能使用默认导出，这样我只管导出`module.exports`属性，至于它上面都有啥就不管了。

看看`require_cjs`方法：

```js
var __getOwnPropNames = Object.getOwnPropertyNames;
var __commonJS = (cb, mod) =>
  function __require() {
    return (
      mod ||
        (0, cb[__getOwnPropNames(cb)[0]])((mod = { exports: {} }).exports, mod),
      mod.exports
    );
  };
var require_cjs = __commonJS({
  "cjs.js"(exports, module) {
    module.exports.name1 = "\u5468\u6770\u4F26";
    exports.name2 = "\u6734\u6811";
  },
});
```

> 百思不得解啊4：为啥要搞成这么奇怪的格式，直接传函数不行吗？


因为`CJS`的导出就是使用在`module.exports`对象上添加属性，或者是重写`module.exports`属性，所以直接将原模块的代码放到一个函数里，然后通过参数的形式传入`module`对象和`exports`属性，这样无需关心代码都做了什么，只要最后导出`module.exports`属性即可，并且还增加了缓存的机制，这也是`CJS`的一个特性，即同一个模块，只有第一次导入时会去执行该模块的代码，然后获取到导出的数据后就会把它缓存起来，后续再导入这个模块会直接从缓存里获取导出数据，这也是`CJS`不同于`ESM`的特性。

## 转换导入

待转换的代码：

```js
const res = require('./cjs.js')
console.log(res);
```

转换结果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03e2222c2dee427da825e94ab5501cbe~tplv-k3u1fbpfcp-zoom-1.image)



报错了，提示目前不支持将`require`转换成`esm`，这是为啥呢，其实是因为`require`是同步的，运行时的，所以可以动态导入、条件导入，可以出现在非顶层，把它当做一个普通函数看待即可，但是`import`导入不行，它是静态编译的，必须出现在顶层，所以是无法转换的，那怎么办呢，很简单，只要把`require`干掉就行，也就是把所有模块都打包到同一个文件里，假设被引入的文件两个模块如下：

```js
// cjs.js
module.exports = {
    name1: '周杰伦',
    name2: '朴树'
}
```

```js
// cjs2.js
module.exports = {
    name3: '许巍',
    name4: '梁博'
}
```

导入它们的模块内容如下：

```js
const res = require('./cjs.js')
console.log(res);

const res2 = require('./cjs2.js')
console.log(res2);

module.exports = {
    res,
    res2
}
```

然后修改一下我们执行转换的`build.js`文件：

```js
// build.js
require("esbuild").buildSync({
  entryPoints: [''],
  outfile: "out.js",
  format: '',
  bundle: true// ++
});
```

然后再转换就不会报错了，结果如下：

```js
// ...
// cjs.js
var require_cjs = __commonJS({
  "cjs.js"(exports, module) {
    module.exports.name1 = "\u5468\u6770\u4F26";
    exports.name2 = "\u6734\u6811";
  }
});

// cjs2.js
var require_cjs2 = __commonJS({
  "cjs2.js"(exports, module) {
    module.exports = {
      name3: "\u8BB8\u5DCD",
      name4: "\u6881\u535A"
    };
  }
});

// cjsUse.js
var require_cjsUse = __commonJS({
  "cjsUse.js"(exports, module) {
    var res = require_cjs();
    console.log(res);
    var res2 = require_cjs2();
    console.log(res2);
    module.exports = {
      res,
      res2
    };
  }
});
export default require_cjsUse();
```

可以看到其实和转换导出的逻辑是一样的，每个模块的内容都会包裹到一个函数里，然后生成一个函数，执行这个函数时就会执行该模块的代码，然后导出的数据就会挂载到`module.exports`上，无论是模块内使用还是导出都可以。

# 总结

温馨提醒，本文的内容纯粹是笔者的个人观点，不一定保证正确~另外以上这些问题也可能没有所谓的原因，换一个转换工具，比如`babel`、`rollup`等可能又会生成不同的代码，有兴趣的自行尝试吧。

总结一下：

- `ESM`转`CJS`：所有导出的变量都挂载到一个对象上，然后`module.exports`该对象。导入的话会判断是经`ESM`转换的`CJS`模块，还是原始的`CJS`模块，都会先创建一个对象，原始`CJS`模块的话会添加一个`default`属性来保存导入的数据，非原始`CJS`模块的话会直接将属性拷贝到新对象上，最后这个新对象作为导入的结果。
- `CJS`转`CSM`：将模块的内容包裹到一个函数内，通过参数的形式传入`module`对象和`module.exports`属性，函数的执行结果为`module.exports`属性的值，并且通过高阶函数的形式来增加缓存导出的功能，转换导出的话直接`export default`该函数的执行结果，导入的话不能单独转换，需要都打包到同一个文件中，所以也就不存在转换后的`import`语句。
