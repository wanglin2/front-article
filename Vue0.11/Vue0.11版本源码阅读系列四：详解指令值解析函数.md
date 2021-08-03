---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight:
---

# 需求

首先该版本的`vue`指令值支持一下几种类型以及通过`dirParser.parse`要返回的数据：

1.实例属性：`message`，解析后应为：

```js
[
    {
        "raw":"message",
        "expression":"message"
    }
]
```

2.表达式：`message === 'show'`，解析后应为：

```js
[
    {
        "raw":"message === 'show'",
        "expression":"message === 'show'"
    }
]
```

3.三元表达式：`show ? true : false`，解析后应为：

```js
[
    {
        "raw":"message === 'show' ? true : false",
        "expression":"message === 'show' ? true : false"
    }
]
```

4.设置元素类名、样式、属性、事件的：`red:hasError,bold:isImportant,hidden:isHidden`，解析后应为：

```js
[
    {"arg":"red","raw":"red: hasError","expression":"hasError"},
    {"arg":"bold","raw":"bold: isImportant","expression":"isImportant"},
    {"arg":"hidden","raw":"hidden: isHidden","expression":"isHidden"}
]
```

或者：`top: top + 'px',left: left + 'px',background-color: 'rgb(0,0,' + bg + ')'`，解析后应为：

```js
[
    {"arg":"top","raw":"top: top + 'px'","expression":"top + 'px'"},
    {"arg":"left","raw":"left: left + 'px'","expression":"left + 'px'"},
    {"arg":"background-color","raw":"background-color: 'rgb(0,0,' + bg + ')'","expression":"'rgb(0,0,' + bg + ')'"}
]
```

5.双大括号插值的：`{{partialId}}`，解析后应为：

```js
[
    {
        "raw":"{{partialId}}",
        "expression":"{{partialId}}"
    }
]
```

6.指令值是数组或对象

`[1, 2, 3]`应解析为：

```js
[
    {
        "raw":"[1, 2, 3]",
        "expression":"[1, 2, 3]"
    }
]
```

`{arr: [1, 2, 3], value: 'abc'}`应解析为：

```js
[
    {
        "raw":"{arr: [1, 2, 3], value: 'abc'}",
        "expression":"{arr: [1, 2, 3], value: 'abc'}"
    }
]
```

7.过滤器：

`message | capitalize`应解析为：

```js
[
    {
        "expression":"message",
        "raw":"message | capitalize",
        "filters":[
            {
                "name":"capitalize",
                "args":null
            }
        ]
    }
]
```

带参数的`message | capitalize 4 5`应解析为：

```js
[
    {
        "expression":"message",
        "raw":"message | capitalize 4 5",
        "filters":[
            {
                "name":"capitalize",
                "args":["4","5"]
            }
        ]
    }
]
```

多个过滤器之间使用`|`进行分隔。

总结一下，就是如果是以逗号分隔的冒号表达式，则解析为：

```js
[
    {
		arg: 【冒号前的字符】,
        expression: 【冒号后的字符】,
        raw: 【原始值】
    },
    ...
]
```

带过滤器的会多一个`filters`字段。

其他一律解析为：

```js
[
    {
        expression: 【和原始值一样的值】,
        raw: 【原始值】
    }
]
```

# 实现

现在让我们从0开始写一个解析器：

```js
let dirs = []
function parse(s) {
    return dirs
}
```

## 简单的变量

首先支持最简单的第一种，实例属性或方法，根据上面的对照，基本原封不动返回即可：

```js
var str = ''
var dirs = []
var dir = {}
var begin = 0
var i = 0
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {}
  pushDir()
  return dirs
}
function pushDir () {
  dir.raw = str.slice(begin, i)
  dir.expression = str.slice(begin, i)
  dirs.push(dir)
}
```

可以看到完全就是为了得到目标值的一个多此一举的过程，下一步来支持逗号分隔的冒号表达式。

## 冒号表达式

先看就一个的情况，如`a:b`，遍历到的当前字符如果是冒号的话就把冒号之前的字符截取出来作为`arg`，冒号后的字符作为`expression`，`begin`变量是用来标记当前这个表达式的起点的，所以要截取冒号后的字符需要新增一个变量：

```js
var str = ''
var dirs = []
var dir = {}
var begin = 0
var argIndex = 0 // ++
var i = 0
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    // ++
    c = str.charCodeAt(i)
    if (c === 0x3A) {// 冒号:
      dir.arg = str.slice(begin, i).trim()
      argIndex = i + 1
    }
  }
  pushDir()
  return dirs
}
function pushDir () {
  dir.raw = str.slice(begin, i)
  dir.expression = str.slice(argIndex, i).trim()// ++
  dirs.push(dir)
}
```

接下来支持存在逗号的情况，逗号相当于一个表达式的结束，所以要进行一次`push`，另外`begin`、`argIndex`、`dir`变量都需要重置：

```js
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (c === 0x3A) {// 冒号:
      dir.arg = str.slice(begin, i).trim()
      argIndex = i + 1
    } else if (c === 0x2C) {// 逗号, ++
      pushDir()
      dir = {}
      begin = argIndex = i + 1
    }
  }
  pushDir()
  return dirs
}
```

## 三元表达式

接下来支持三元表达式，目前会把三元表达式的冒号前后部分分离调，会输出类似下面的结果：

```js
[
    {
        "arg":"show ? true",
        "raw":"show ? true : false",
        "expression":"false"
    }
]
```

所以检查到冒号的时候我们要判断一下这是否是个三元表达式，是的话就不截取：

```js
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (c === 0x3A) {// 冒号: ++
      var arg = str.slice(begin, i).trim()
      if (/^[^\?]+$/.test(arg)) {
        dir.arg = arg
        argIndex = i + 1
      }
    } else if (c === 0x2C) {
      pushDir()
      dir = {}
      begin = argIndex = i + 1
    }
  }
  pushDir()
  return dirs
}
```

判断一下冒号之前的字符里是否存在`?`，存在的话就代表是三元表达式，则不进行分割。

看一个特殊情况：`background-color: 'rgb(0,0,' + bg + ')'`

目前会解析成：

```js
[
    {
        "arg":"background-color",
        "raw":"background-color: 'rgb(0",
        "expression":"'rgb(0"
    },
    {
        "raw":"0",
        "expression":"0"
    },
    {
        "raw":"' + bg + ')'",
        "expression":"' + bg + ')'"
    }
]
```

原因就出在属性值里的逗号，如果属性值里存在逗号，那该属性值一定是被引号包围的，所以在单引号或双引号里的都要忽略，所以让我们新增两个变量来记录是否是在引号里：

```js
var inDouble = false // ++
var inSingle = false // ++
```

如果出现第一个引号，把标志设为true，然后中间字符都直接跳过，直到出现闭合的引号，才退出继续其他的判断：

```js
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (inDouble) {// 双引号还未闭合 ++
      if (c === 0x22) {// 出现了闭合引号
        inDouble = !inDouble
      }
    } else if (inSingle) {// 单引号还未闭合 ++
      if (c === 0x27) {// 出现了闭合引号
        inSingle = !inSingle
      }
    } else if (c === 0x3A) {
      // ...
    } else if (c === 0x2C) {
      // ...
    } else {// ++
      switch (c) {
        // 首次出现引号设置标志位
        case 0x22: inDouble = true; break // "
        case 0x27: inSingle = true; break // '      
        default:
          break;
      }
    }
  }
  pushDir()
  return dirs
}
```

## 数组或对象

数组或对象都需要原封不动的返回，因为带冒号和逗号目前都会被切割，对数组来说，字符都是被`[]`中括号包围的，所以在这区间的逗号要忽略掉，因为括号可能多重嵌套，所以增加一个变量来计数，出现左括号加1，出现右括号减1，为0就代表不在括号里：

```js
var square = 0// ++
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (inDouble) {} 
    else if (inSingle) {} 
    else if (c === 0x3A) {} 
    else if (c === 0x2C && square === 0) {// ++
      pushDir()
      dir = {}
      begin = argIndex = i + 1
    } else {
      switch (c) {
        case 0x22: inDouble = true; break 
        case 0x27: inSingle = true; break   
        case 0x5B: square++; break        // [ ++
        case 0x5D: square--; break        // ] ++
        default:
          break;
      }
    }
  }
  pushDir()
  return dirs
}
```

对象也是类似，但是对象多了冒号，冒号也会被截掉，所以需要和三元表达式一样进行判断：

```js
var curly = 0// ++
exports.parse = function (s) {
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (inDouble) {} 
    else if (inSingle) {} 
    else if (c === 0x3A) {
      var arg = str.slice(begin, i).trim()
      if (/^[^\?\{]+$/.test(arg)) {// ++ 正则表达式修改，如果出现了{代表可能是对象
        dir.arg = arg
        argIndex = i + 1
      }
    } else if (c === 0x2C && square === 0 && curly === 0) {// ++
      pushDir()
      dir = {}
      begin = argIndex = i + 1
    } else {
      switch (c) {
        // ...
        case 0x7B: curly++; break         // { ++
        case 0x7D: curly--; break         // } ++
        default:
          break;
      }
    }
  }
  pushDir()
  return dirs
}
```

## 过滤器

最后来看过滤器，过滤器使用管道符`|`，所以遍历到这个字符时推入过滤器，过滤器支持多个，第一个字符串代表表达式，后续`|`分隔的各代表一个过滤器，当出现第一个`|`时只能获取到该过滤器所被应用的值，也就是`expression`的值，需要继续遍历才知道具体的过滤器，如何判断是否是第一个`|`可以根据`expression`是否有值：

```js
exports.parse = function (s) {
    for (i = 0, l = str.length; i < l; i++) {
        c = str.charCodeAt(i)
        // ...
        else if (c === 0x7C) {// 管道符|
            if (dir.expression === undefined) {// 第一次出现|
                dir.expression = str.slice(argIndex, i).trim()// 截取第一个|前的字符来作为表达式的值
            }
        }
        // ...
    }
}
function pushDir () {
    dir.raw = str.slice(begin, i)
    if (dir.expression === undefined) {// ++ 这里也需要进行判断，如果有值代表已经被过滤器分支设置过了，这里就不需要设置
        dir.expression = str.slice(argIndex, i).trim()
    }
    dirs.push(dir)
}
```

假设只有一个过滤器的话继续遍历会直到结束，结束后会再调一次`pushDir`方法，所以修改一下这个方法，进行一次过滤器收集处理：

```js

function pushDir () {
  dir.raw = str.slice(begin, i)
  if (dir.expression === undefined) {
    dir.expression = str.slice(argIndex, i).trim()
  } else {// ++ 添加过滤器
    pushFilter()
  }
  dirs.push(dir)
}
function pushFilter () {
    // 这里要截取的字符串应该是|后面的，begin和argIndex字段都用不了，所以需要新增一个变量
}
```

新增一个变量用于记录当前过滤器的起始位置：

```js
var lastFilterIndex = 0 // ++
exports.parse = function (s) {
    for (i = 0, l = str.length; i < l; i++) {
        c = str.charCodeAt(i)
        // ...
        else if (c === 0x7C) {
            if (dir.expression === undefined) {
                dir.expression = str.slice(argIndex, i).trim()
            }
            lastFilterIndex = i + 1// ++
        }
        // ...
    }
}
```

因为过滤器支持带参数，参数和过滤器名之间用空格分隔，所以写一个正则来匹配一下：`/[^\s'"]+|'[^']+'|"[^"]+"/g`，参数除了是变量也可以是字符串，所以后面两个对引号的匹配是为了保证最后匹配的结果也是带引号的，否则：`capitalize 'abc'`和`capitalize  abc`最后匹配出来的都是：`["abc"]`，加上后面两个引号的匹配后则才是我们需要的：`["'abc'"]`。

```js
function pushFilter() {
  var exp = str.slice(lastFilterIndex, i).trim()
  if (exp) {
    var tokens = exp.match(/[^\s'"]+|'[^']+'|"[^"]+"/g)
    var filter = {}
    filter.name = tokens[0]
    filter.args = tokens.length > 1 ? tokens.slice(1) : null
    dir.filters = dir.filters || []
    dir.filters.push(filter)
  }
}
```

结果如下：

![image-20210108154353418](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bf259ced0f2445a8213308eb1f43aeb~tplv-k3u1fbpfcp-zoom-1.image)

接下来支持一下多个过滤器的情况，多个过滤器，则会出现多个`|`，所以又会走到`|`的`if`分支，非第一次出现的话不需要修改`expression`的值，直接`push`当前遍历到的过滤器即可：

```js
exports.parse = function (s) {
    for (i = 0, l = str.length; i < l; i++) {
        c = str.charCodeAt(i)
        // ...
        else if (c === 0x7C) {
            if (dir.expression === undefined) {
                dir.expression = str.slice(argIndex, i).trim()
            } else {// 非第一次出现直接push ++ 
                pushFilter()
            }
            lastFilterIndex = i + 1
        }
        // ...
    }
}
```

结果如下：

![image-20210108154842736](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1990b2ef7828414fa4ce0c341a0bf58b~tplv-k3u1fbpfcp-zoom-1.image)

最后也看一种特殊情况，就是`||`的情况，这是或的意思，所以肯定不能解析为过滤器，在`if`条件里增加一下判断，排除当前遍历到的`|`前一个或后一个字符也是`|`的情况：

```js
exports.parse = function (s) {
    for (i = 0, l = str.length; i < l; i++) {
        c = str.charCodeAt(i)
        // ...
        else if (c === 0x7C && str.charCodeAt(i - 1) !== 0x7C && str.charCodeAt(i + 1) !== 0x7C) {// ++ 
            // ...
        }
        // ...
    }
}
```

## 完成

到这里基本就完成了，完整代码如下：

```js
var str = ''
var dirs = []
var dir = {}
var begin = 0
var argIndex = 0 
var i = 0
var inDouble = false
var inSingle = false
var square = 0
var curly = 0
var lastFilterIndex = 0
function reset() {
  str = ''
  dirs = []
  dir = {}
  begin = 0
  argIndex = 0 
  i = 0
  inDouble = false
  inSingle = false
  square = 0
  curly = 0
  lastFilterIndex = 0
}
exports.parse = function (s) {
  reset()
  str = s
  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (inDouble) {// 双引号还未闭合
      if (c === 0x22) {// 出现了闭合引号
        inDouble = !inDouble
      }
    } else if (inSingle) {// 单引号还未闭合
      if (c === 0x27) {// 出现了闭合引号
        inSingle = !inSingle
      }
    } else if (c === 0x3A) {// 冒号:
      var arg = str.slice(begin, i).trim()
      if (/^[^\?\{]+$/.test(arg)) {
        dir.arg = arg
        argIndex = i + 1
      }
    } else if (c === 0x2C && square === 0 && curly === 0) {// 逗号,
      pushDir()
      dir = {}
      begin = argIndex = i + 1
    } else if (c === 0x7C && str.charCodeAt(i - 1) !== 0x7C && str.charCodeAt(i + 1) !== 0x7C) {// 管道符|
      if (dir.expression === undefined) {// 第一次出现|
        dir.expression = str.slice(argIndex, i).trim()
      } else {// 非第一次出现直接push
        pushFilter()
      }
      lastFilterIndex = i + 1
    } else {
      switch (c) {
        case 0x22: inDouble = true; break // "
        case 0x27: inSingle = true; break // '  
        case 0x5B: square++; break        // [
        case 0x5D: square--; break        // ]
        case 0x7B: curly++; break         // {
        case 0x7D: curly--; break         // }
        default:
          break;
      }
    }
  }
  pushDir()
  return dirs
}
function pushDir () {
  dir.raw = str.slice(begin, i)
  if (dir.expression === undefined) {// ++ 这里也需要进行判断，如果有值代表已经被过滤器分支设置过了，这里就不需要设置
    dir.expression = str.slice(argIndex, i).trim()
  } else {// ++ 添加过滤器
    pushFilter()
  }
  dirs.push(dir)
}
function pushFilter() {
  var exp = str.slice(lastFilterIndex, i).trim()
  if (exp) {
    var tokens = exp.match(/[^\s'"]+|'[^']+'|"[^"]+"/g)
    var filter = {}
    filter.name = tokens[0]
    filter.args = tokens.length > 1 ? tokens.slice(1) : null
    dir.filters = dir.filters || []
    dir.filters.push(filter)
  }
}
```

把上面的代码替换掉`vue`源码里的相关代码，测试了一下基本用例是能跑通的，但是可能还会有其他一些特殊场景没有照顾到，更完善的代码请自行阅读`vue`源码。
