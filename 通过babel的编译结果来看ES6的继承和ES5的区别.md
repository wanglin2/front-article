如果以前问我`ES5`的继承和`ES6`的继承有什么区别，我一定会自信的说没有区别，不过是语法糖而已，充其量也就是写法有区别，但是现在我会假装思考一下，然后说虽然只是语法糖，但也是有点小区别的，那么具体有什么区别呢，不要走开，下文更精彩！

本文会先回顾一下`ES5`的寄生组合式继承的实现，然后再看一下`ES6`的写法，最后根据`Babel`的编译结果来看一下到底有什么区别。



# ES5：寄生组合式继承

`js`有很多种继承方式，比如大家耳熟能详的`原型链继承`、`构造继承`、`组合继承`、`寄生继承`等，但是这些或多或少都有一些不足之处，所以笔者认为我们只要记住一种就可以了，那就是`寄生组合式继承`。

首先要明确继承到底要继承些什么东西，一共有三部分，一是实例属性/方法、二是原型属性/方法、三是静态属性/方法，我们分别来看。

先来看一下我们要继承的父类的函数：

```js
// 父类
function Sup(name) {
    this.name = name// 实例属性
}
Sup.type = '午'// 静态属性
// 静态方法
Sup.sleep =  function () {
    console.log(`我在睡${this.type}觉`)
}
// 实例方法
Sup.prototype.say = function() {
    console.log('我叫 ' + this.name)
}
```



## 继承实例属性/方法

要继承实例属性/方法，明显要执行一下`Sup`函数才行，并且要修改它的`this`指向，这使用`call`、`apply`方法都行：

```js
// 子类
function Sub(name, age) {
    // 继承父类的实例属性
    Sup.call(this, name)
    // 自己的实例属性
    this.age = age
}
```

![image-20210824173830421.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdaa9640982a4ec2b975c254a6f46250~tplv-k3u1fbpfcp-watermark.image)

能这么做的原理又是另外一道经典面试题：`new操作符都做了什么`，很简单，就`4`点：

1.创建一个空对象

2.把该对象的`__proto__`属性指向`Sub.prototype`

3.让构造函数里的`this`指向新对象，然后执行构造函数，

4.返回该对象

所以`Sup.call(this)`的`this`指的就是这个新创建的对象，那么就会把父类的实例属性/方法都添加到该对象上。



## 继承原型属性/方法

我们都知道如果一个对象它本身没有某个方法，那么会去它构造函数的原型对象上，也就是`__proto__`指向的对象上查找，如果还没找到，那么会去构造函数原型对象的`__proto__`上查找，这样一层一层往上，也就是传说中的原型链，所以`Sub`的实例想要能访问到`Sup`的原型方法，就需要把`Sub.prototype`和`Sup.prototype`关联起来，这有几种方法：

### 1.使用`Object.create`

```js
Sub.prototype = Object.create(Sup.prototype)
Sub.prototype.constructor = Sub
```

### 2.使用`__proto__`

```js
Sub.prototype.__proto__ = Sup.prototype
```

### 3.借用中间函数

```js
function Fn() {}
Fn.prototype = Sup.prototype
Sub.prototype = new Fn()
Sub.prototype.constructor = Sub
```

以上三种方法都可以，我们再来覆盖一下继承到的`Say`方法，然后在该方法里面再调用父类原型上的`say`方法：

```js
Sub.prototype.say = function () {
    console.log('你好')
    // 调用父类的该原型方法
    // this.__proto__ === Sub.prototype、Sub.prototype.__proto__ === Sup.prototype
    this.__proto__.__proto__.say.call(this)
    console.log(`今年${this.age}岁`)
}
```

![image-20210824182416678.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb0266d6b2b74f069655f1b0a9822064~tplv-k3u1fbpfcp-watermark.image)


## 继承静态属性/方法

也就是继承`Sup`函数本身的属性和方法，这个很简单，遍历一下父类自身的可枚举属性，然后添加到子类上即可：

```js
Object.keys(Sup).forEach((prop) => {
    Sub[prop] = Sup[prop]
})
```

![image-20210824182459876.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f0fa2e19db041119887e1dd97dbdecb~tplv-k3u1fbpfcp-watermark.image)


# ES6：使用class继承

接下来我们使用`ES6`的`class`关键字来实现上面的例子：

```js
// 父类
class Sup {
    constructor(name) {
        this.name = name
    }
    
    say() {
        console.log('我叫 ' + this.name)
    }
    
    static sleep() {
        console.log(`我在睡${this.type}觉`)
    }
}
// static只能设置静态方法，不能设置静态属性，所以需要自行添加到Sup类上
Sup.type = '午'
// 另外，原型属性也不能在class里面设置，需要手动设置到prototype上，比如Sup.prototype.xxx = 'xxx'

// 子类，继承父类
class Sub extends Sup {
    constructor(name, age) {
        super(name)
        this.age = age
    }
    
    say() {
        console.log('你好')
        super.say()
        console.log(`今年${this.age}岁`)
    }
}
Sub.type = '懒'
```

![image-20210824182650898.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03f2935b70f14463a9feb0eaa7a84c2a~tplv-k3u1fbpfcp-watermark.image)

可以看到一样的效果，使用`class`会简洁明了很多，接下来我们使用`babel`来把这段代码编译回`ES5`的语法，看看和我们写的有什么不一样，由于编译完的代码有200多行，所以不能一次全部贴上来，我们先从父类开始看：



## 编译后的父类

```js
// 父类
var Sup = (function () {
  function Sup(name) {
    _classCallCheck(this, Sup);

    this.name = name;
  }

  _createClass(
    Sup,
    [
      {
        key: "say",
        value: function say() {
          console.log("我叫 " + this.name);
        },
      },
    ],
    [
      {
        key: "sleep",
        value: function sleep() {
          console.log("\u6211\u5728\u7761".concat(this.type, "\u89C9"));
        },
      },
    ]
  );

  return Sup;
})(); // static只能设置静态方法，不能设置静态属性

Sup.type = "午"; // 子类，继承父类
// 如果我们之前通过Sup.prototype.xxx = 'xxx'设置了原型属性，那么跟静态属性一样，编译后没有区别，也是这么设置的
```

可以看到是个自执行函数，里面定义了一个`Sup`函数，`Sup`里面先调用了一个`_classCallCheck(this, Sup)`函数，我们转到这个函数看看：

```js
function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}
```

`instanceof`运算符是用来检测右边函数的`prototype`属性是否出现在左边的对象的原型链上，简单说可以判断某个对象是否是某个构造函数的实例，可以看到如果不是的话就抛错了，错误信息是`不能把一个类当做函数调用`，这里我们就发现第一个区别了：

>  区别1：ES5里的构造函数就是一个普通的函数，可以使用new调用，也可以直接调用，而ES6的class不能当做普通函数直接调用，必须使用new操作符调用

继续看自执行函数，接下来调用了一个`_createClass`方法：

```js
function _createClass(Constructor, protoProps, staticProps) {
  if (protoProps) _defineProperties(Constructor.prototype, protoProps);
  if (staticProps) _defineProperties(Constructor, staticProps);
  return Constructor;
}
```

该方法接收三个参数，分别是构造函数、原型方法、静态方法（注意不包含原型属性和静态属性），后面两个都是数组，数组里面每一项代表一个方法对象，不管是实例方法还是原型方法，都是通过`_defineProperties`方法设置，先来看该方法：

```js
function _defineProperties(target, props) {
  for (var i = 0; i < props.length; i++) {
    var descriptor = props[i];
    // 设置该属性是否可枚举，设为false则for..in、Object.keys遍历不到该属性
    descriptor.enumerable = descriptor.enumerable || false;
    // 默认可配置，即能修改和删除该属性
    descriptor.configurable = true;
    // 设为true时该属性的值能被赋值运算符改变
    if ("value" in descriptor) descriptor.writable = true;
    Object.defineProperty(target, descriptor.key, descriptor);
  }
}
```

可以看到它是通过`Object.defineProperty`方法来设置原型方法和静态方法，而且`enumerable`默认为`false`，这就来到了第二个区别：

> 区别2：ES5的原型方法和静态方法默认是可枚举的，而class的默认不可枚举，如果想要获取不可枚举的属性可以使用Object.getOwnPropertyNames方法

接下来看子类编译后的代码：



## 编译后的子类

```js
// 子类，继承父类
var Sub = (function (_Sup) {
  _inherits(Sub, _Sup);

  var _super = _createSuper(Sub);

  function Sub(name, age) {
    var _this;

    _classCallCheck(this, Sub);

    _this = _super.call(this, name);
    _this.age = age;
    return _this;
  }

  _createClass(Sub, [
    {
      key: "say",
      value: function say() {
        console.log("你好");

        _get(_getPrototypeOf(Sub.prototype), "say", this).call(this);

        console.log("\u4ECA\u5E74".concat(this.age, "\u5C81"));
      }
    }
  ]);

  return Sub;
})(Sup);

Sub.type = "懒";
```

同样也是一个自执行方法，把要继承的父类构造函数作为参数传进去了，进来先调用了`_inherits(Sub, _Sup)`方法，虽然`Sub`函数是在后面定义的，但是函数声明是存在提升的，所以这里是可以正常访问到的：

```js
function _inherits(subClass, superClass) {
  // 被继承对象的必须是一个函数或null
  if (typeof superClass !== "function" && superClass !== null) {
    throw new TypeError("Super expression must either be null or a function");
  }
  // 设置原型
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: { value: subClass, writable: true, configurable: true }
  });
  if (superClass) _setPrototypeOf(subClass, superClass);
}
```

这个方法先检查了父类是否合法，然后通过`Object.create`方法设置了子类的原型，这个和我们之前的写法是一样的，只是今天我才发现`Object.create`居然还有第二个参数，第二个参数必须是一个对象，对象的自有可枚举属性(即其自身定义的属性，而不是其原型链上的枚举属性)将为新创建的对象添加指定的属性值和对应的属性描述符。

这个方法的最后为我们揭晓了第三个区别：

> 区别3：子类可以直接通过`__proto__`找到父类，而ES5是指向`Function.prototype`：
>
> ES6：`Sub.__proto__ === Sup`
>
> ES5：`Sub.__proto__ === Function.prototype`

为啥会这样呢，看看`_setPrototypeOf`方法做了啥就知道了：

```js
function _setPrototypeOf(o, p) {
    _setPrototypeOf =
        Object.setPrototypeOf ||
        function _setPrototypeOf(o, p) {
            o.__proto__ = p;
            return o;
        };
    return _setPrototypeOf(o, p);
}
```

可以看到这个方法把`Sub.__proto__`设置为了`Sup`，这样同时也完成了静态方法和属性的继承，因为函数也是对象，自身没有的属性和方法也会沿着`__proto__`链查找。

`_inherits`方法过后紧接着调用了一个`_createSuper(Sub)`方法，拉出来看看：

```js
function _createSuper(Derived) {
    return function _createSuperInternal() {
        // ...
    };
}
```

这个函数接收子类构造函数，然后返回了一个新函数，我们先跳到后面的子类构造函数的定义：

```js
function Sub(name, age) {
    var _this;

    // 检查是否当做普通函数调用，是的话抛错
    _classCallCheck(this, Sub);

    _this = _super.call(this, name);
    _this.age = age;
    return _this;
}
```

同样是先检查了一下是否是使用`new`调用，然后我们发现这个函数返回了一个`_this`，前面介绍了`new`操作符都做了什么，我们知道会隐式创建一个对象，并且会把函数内的`this`指向该对象，如果没有显式的指定构造函数返回什么，那么就会默认返回这个新创建的对象，而这里显然是手动指定了要返回的对象，而这个`_this`来自于`_super`函数的执行结果，`_super`就是前面`_createSuper`返回的新函数：

```js
function _createSuper(Derived) {
    // _isNativeReflectConstruct会检查Reflect.construct方法是否可用
    var hasNativeReflectConstruct = _isNativeReflectConstruct();
    return function _createSuperInternal() {
        // _getPrototypeOf方法用来获取Derived的原型，也就是Derived.__proto__
        var Super = _getPrototypeOf(Derived),
            result;
        if (hasNativeReflectConstruct) {
            // NewTarget === Sub
            var NewTarget = _getPrototypeOf(this).constructor;
            // Reflect.construct的操作可以简单理解为：result = new Super(...arguments)，第三个参数如果传了则作为新创建对象的构造函数，也就是result.__proto__ === NewTarget.prototype，否则默认为Super.prototype
            result = Reflect.construct(Super, arguments, NewTarget);
        } else {
            result = Super.apply(this, arguments);
        }
        return _possibleConstructorReturn(this, result);
    };
}
```

`Super`代表的是`Sub.__proto__`，根据前面的继承操作，我们知道子类的`__proto__`指向了父类，也就是`Sup`，这里会优先使用`Reflect.construct`方法，相当于创建了一个父类的实例，并且这个实例的`__proto__`又指回了`Sub.prototype`，不得不说这个`api`真是神奇。

我们就不考虑降级情况了，那么最后会返回这个父类的实例对象。

回到`Sub`构造函数，`_this`指向的就是这个通过父类创建的实例对象，为什么要这么做呢，这其实就是第四个区别了，也是最重要的区别：

> 区别4：ES5的继承，实质是先创造子类的实例对象`this`，然后再执行父类的构造函数给它添加实例方法和属性(不执行也无所谓）。而ES6的继承机制完全不同，实质是先创造父类的实例对象`this`（当然它的`__proto__`指向的是子类的`prototype`），然后再用子类的构造函数修改`this`。

这就是为啥使用`class`继承在`constructor`函数里必须调用`super`，因为子类压根没有自己的`this`，另外不能在`super`执行前访问`this`的原因也很明显了，因为调用了`super`后，`this`才有值。

子类自执行函数的最后一部分也是给它设置原型方法和静态方法，这个前面讲过了，我们重点看一下实例方法编译后的结果：

```js
function say() {
    console.log("你好");

    _get(_getPrototypeOf(Sub.prototype), "say", this).call(this);

    console.log("\u4ECA\u5E74".concat(this.age, "\u5C81"));
}
```

猜你们也忘了编译前的原函数是啥样的了，请看：

```js
say() {
    console.log('你好')
    super.say()
    console.log(`今年${this.age}岁`)
}
```

在`ES6`的`class`里`super`有两种含义，当做函数调用的话它代表父类的构造函数，只能在`constructor`里面调用，当做对象使用时它指向父类的原型对象，所以`_get(_getPrototypeOf(Sub.prototype), "say", this).call(this)`这行大概相当于`Sub.prototype.__proto__.say.call(this)`，跟我们最开始写的`ES5`版本也差不多，但是显然在`class`的语法要简单很多。

到此，编译后的代码我们就分析的差不多了，不过其实还有一个区别不知道大家有没有发现，那就是为啥要使用自执行函数，一当然是为了封装一些变量，二其实是因为第五个区别：

> 区别5：class不存在变量提升，所以父类必须在子类之前定义

不信你把父类放到子类后面试试，不出意外会报错，你可能会觉得直接使用函数表达式也可以达到这样的效果，非也：

```js
// 会报错
var Sub = function(){ Sup.call(this) }
new Sub()
var Sup = function(){}

// 不会报错
var Sub = function(){ Sup.call(this) }
var Sup = function(){}
new Sub()
```

但是`Babel`编译后的无论你在哪里实例化子类，只要父类在它之后声明都会报错。



# 总结

本文通过分析`Babel`编译后的代码来总结了`ES5`和`ES6`继承的5个区别，可能还有一些其他的，有兴趣可以自行了解。

想要详细了解`class`知识可以看这篇教程[class继承](http://caibaojian.com/es6/class.html)。

示例代码在[https://github.com/wanglin2/es5-es5-inherit-example](https://github.com/wanglin2/es5-es5-inherit-example)。
