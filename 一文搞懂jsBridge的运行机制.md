我司的APP是一个典型的混合开发APP，内嵌的都是前端页面，前端页面要做到和原生的效果相似，就避免不了调用一些原生的方法，`jsBridge`就是`js`和`原生`通信的桥梁，本文不讲概念性的东西，而是通过分析一下我司项目中的`jsBridge`源码，来从前端角度大概了解一下它是怎么实现的。


# js调用方式

先来看一下，`js`是怎么来调用某个原生方法的，首先初始化的时候会调用`window.WebViewJavascriptBridge.init`方法：

```js
window.WebViewJavascriptBridge.init()
```

然后如果要调用某个原生方法可以使用下面的函数：

```js
function native (funcName, args = {}, callbackFunc, errorCallbackFunc) {
    // 校验参数是否合法
    if (args && typeof args === 'object' && Object.prototype.toString.call(args).toLowerCase() === '[object object]' && !args.length) {
        args = JSON.stringify(args);
    } else {
        throw new Error('args不符合规范');
    }
    // 判断是否是手机环境
    if (getIsMobile()) {
        // 调用window.WebViewJavascriptBridge对象的callHandler方法
        window.WebViewJavascriptBridge.callHandler(
            funcName,
            args,
            (res) => {
                res = JSON.parse(res);
                if (res.code === 0) {
                    return callbackFunc(res);
                } else {
                    return errorCallbackFunc(res);
                }
            }
        );
    }
}
```

传入要调用的方法名、参数和回调即可，它先校验了一下参数，然后会调用`window.WebViewJavascriptBridge.callHandler`方法。

此外也可以提供回调供原生调用：

```js
window.WebViewJavascriptBridge.registerHandler(funcName, callbackFunc);
```

接下来看一下`window.WebViewJavascriptBridge`对象到底是啥。


# 安卓

`WebViewJavascriptBridge.js`文件内是一个自执行函数，首先定义了一些变量：

```js
// 定义变量
var messagingIframe;
var sendMessageQueue = [];// 发送消息的队列
var receiveMessageQueue = [];// 接收消息的队列
var messageHandlers = {};// 消息处理器

var CUSTOM_PROTOCOL_SCHEME = 'yy';// 自定义协议
var QUEUE_HAS_MESSAGE = '__QUEUE_MESSAGE__/';

var responseCallbacks = {};// 响应的回调
var uniqueId = 1;
```

根据变量名简单翻译了一下，具体用处接下来会分析。接下来定义了`WebViewJavascriptBridge`对象：

```js
var WebViewJavascriptBridge = window.WebViewJavascriptBridge = {
    init: init,
    send: send,
    registerHandler: registerHandler,
    callHandler: callHandler,
    _fetchQueue: _fetchQueue,
    _handleMessageFromNative: _handleMessageFromNative
};
```

可以看到就是一个普通的对象，上面挂载了一些方法，具体方法暂时不看，继续往下：

```js
var doc = document;
_createQueueReadyIframe(doc);
```

调用了`_createQueueReadyIframe`方法：

```js
function _createQueueReadyIframe (doc) {
    messagingIframe = doc.createElement('iframe');
    messagingIframe.style.display = 'none';
    doc.documentElement.appendChild(messagingIframe);
}
```

这个方法很简单，就是创建了一个隐藏的`iframe`插入到页面，继续往下：

```js
// 创建一个Events类型（基础事件模块）的事件（Event）对象
var readyEvent = doc.createEvent('Events');
// 定义事件名为WebViewJavascriptBridgeReady
readyEvent.initEvent('WebViewJavascriptBridgeReady');
// 通过document来触发该事件
doc.dispatchEvent(readyEvent);
```

这里定义了一个自定义事件，并直接派发了，其他地方可以像通过监听原生事件一样监听该事件：

```js
document.addEventListener(
    'WebViewJavascriptBridgeReady',
    function () {
        console.log(window.WebViewJavascriptBridge)
    },
    false
);
```

这里的用处我理解就是当该`jsBridge`文件如果是在其他代码之后引入的话需要保证之前的代码能知道`window.WebViewJavascriptBridge`对象何时可用，如果规定该`jsBridge`必须要最先引入的话那么就不需要这个处理了。

到这里自执行函数就结束了，接下来看一下最开始的`init`方法：

```js
function init (messageHandler) {
    if (WebViewJavascriptBridge._messageHandler) {
        throw new Error('WebViewJavascriptBridge.init called twice');
    }
    // init调用的时候没有传参，所以messageHandler=undefined
    WebViewJavascriptBridge._messageHandler = messageHandler;
    // 当前receiveMessageQueue也只是一个空数组
    var receivedMessages = receiveMessageQueue;
    receiveMessageQueue = null;
    for (var i = 0; i < receivedMessages.length; i++) {
        _dispatchMessageFromNative(receivedMessages[i]);
    }
}
```

从初始化的角度来看，这个`init`方法似乎啥也没做。接下来我们来看`callHandler`方法，看看是如何调用安卓的方法的：

```js
function callHandler (handlerName, data, responseCallback) {
    _doSend({
        handlerName: handlerName,
        data: data
    }, responseCallback);
}
```

处理了一下参数又调用了`_doSend`方法：

```js
function _doSend (message, responseCallback) {
    // 如果提供了回调的话
    if (responseCallback) {
        // 生成一个唯一的回调id
        var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
        // 回调通过id存储到responseCallbacks对象上
        responseCallbacks[callbackId] = responseCallback;
        // 把该回调id添加到要发送给native的消息里
        message.callbackId = callbackId;
    }
    // 消息添加到消息队列里
    sendMessageQueue.push(message);
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

这个方法首先把调用原生方法时的回调函数通过生成一个唯一的`id`保存到最开始定义的`responseCallbacks`对象里，然后把该`id`添加到要发送的信息上，所以一个`message`的结构是这样的：

```js
{
    handlerName,
    data,
    callbackId
}
```

接着把该`message`添加到最开始定义的`sendMessageQueue`数组里，最后设置了`iframe`的`src`属性：`yy://__QUEUE_MESSAGE__/`，这其实就是一个自定义协议的`url`，我简单搜索了一下，`native`会拦截这个`url`来做相应的处理，到这里我们就走不下去了，因为不知道原生做了什么事情，简单搜索了一下，发现了这个库：[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)，我司应该是在这个库基础上修改的，结合了网上的一些文章后大概知道了，原生拦截到这个`url`后会调用`js`的`window.WebViewJavascriptBridge._fetchQueue`方法：

```js
function _fetchQueue () {
    // 把我们要发送的消息队列转成字符串
    var messageQueueString = JSON.stringify(sendMessageQueue);
    // 清空消息队列
    sendMessageQueue = [];
    // 安卓无法直接读取返回的数据，因此还是通过iframe的src和java通信
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
}
```

安卓拦截到`url`后，知道`js`给安卓发送消息了，所以主动调用`js`的`_fetchQueue`方法，取出之前添加到队列里的消息，因为无法直接读取`js`方法返回的数据，所以把格式化后的消息添加到`url`上，再次通过`iframe`来发送，此时原生又会拦截到`yy://return/_fetchQueue/`这个`url`，那么取出后面的消息，解析出要其中要执行的原生方法名和参数后执行对应的原生方法，当原生方法执行完后又会主动调用`js`的`window.WebViewJavascriptBridge._handleMessageFromNative`方法：

```js
function _handleMessageFromNative (messageJSON) {
    // 根据之前的init方法的逻辑我们知道receiveMessageQueue是会被设置为null的，所以会走else分支
    if (receiveMessageQueue) {
        receiveMessageQueue.push(messageJSON);
    } else {
        _dispatchMessageFromNative(messageJSON);
    }
}
```

看一下`_dispatchMessageFromNative`方法做了什么：

```js
function _dispatchMessageFromNative (messageJSON) {
    setTimeout(function () {
        // 原生发回的消息是字符串类型的，转成json
        var message = JSON.parse(messageJSON);
        var responseCallback;
        // java调用完成，发回的responseId就是我们之前发送给它的callbackId
        if (message.responseId) {
            // 从responseCallbacks对象里取出该id关联的回调方法
            responseCallback = responseCallbacks[message.responseId];
            if (!responseCallback) {
                return;
            }
            // 执行回调，js调用安卓方法后到这里顺利收到消息
            responseCallback(message.responseData);
            delete responseCallbacks[message.responseId];
        } else {
            // ...
        }
    });
}
```

`messageJSON`就是原生发回的消息，里面除了执行完原生方法后返回的相关信息外，还带着之前我们传给它的`callbackId`，所以我们可以通过这个`id`来在`responseCallbacks`里找到关联的回调并执行，本次`js`调用原生方法流程结束。但是，明显函数里还有不存在`id`时的分支，这里是用来干啥的呢，我们前面介绍的都是`js`调用原生方法，但是显然，原生也可以直接给`js`发消息，比如常见的拦截返回键功能，当原生监听到返回键事件后它会主动发送信息告诉前端页面，页面就可以执行对应的逻辑，这个`else`分支就是用来处理这种情况：

```js
function _dispatchMessageFromNative (messageJSON) {
    setTimeout(function () {
        if (message.responseId) {
            // ...
        } else {
            // 和我们传给原生的消息可以带id一样，原生传给我们的消息也可以带一个id，同时原生内部也会通过这个id关联一个回调
            if (message.callbackId) {
                var callbackResponseId = message.callbackId;
                //如果前端需要再给原生回消息的话那么就带上原生之前传来的id，这样原生就可以通过id找到对应的回调并执行
                responseCallback = function (responseData) {
                    _doSend({
                        responseId: callbackResponseId,
                        responseData: responseData
                    });
                };
            }
            // 我们并没有设置默认的_messageHandler，所以是undefined
            var handler = WebViewJavascriptBridge._messageHandler;
            // 原生发送的消息里面有处理方法名称
            if (message.handlerName) {
                // 通过方法名称去messageHandlers对象里查找是否有对应的处理方法
                handler = messageHandlers[message.handlerName];
            }
            try {
                // 执行处理方法
                handler(message.data， responseCallback);
            } catch (exception) {
                if (typeof console !== 'undefined') {
                    console.log('WebViewJavascriptBridge: WARNING: javascript handler threw.', message, exception);
                }
            }
        }
    });
}
```

比如我们要监听原生的返回键事件，我们先通过`window.WebViewJavascriptBridge`对象的方法注册一下：

```js
window.WebViewJavascriptBridge.registerHandler('onBackPressed', () => {
    // 做点什么...
})
```

`registerHandler`方法如下：

```js
function registerHandler (handlerName, handler) {
    messageHandlers[handlerName] = handler;
}
```

很简单，把我们要监听的事件名和方法都存储到`messageHandlers`对象上，然后如果原生监听到返回键事件后会发送如下结构的消息：

```js
{
    handlerName: 'onBackPressed'
}
```

这样就可以通过`handlerName`找到我们注册的函数进行执行了。

到此，安卓环境的`js`和原生互相调用的逻辑就结束了，总结一下就是：

1.`js`调用原生

生成一个唯一的`id`，把回调和`id`保存起来，然后将要发送的信息（带上本次生成的唯一id）添加到一个队列里，之后通过`iframe`发送一个自定义协议的请求，原生拦截到后调用`js`的`window.WebViewJavascriptBridge`对象的一个方法来获取队列的信息，解析出请求和参数后执行对应的原生方法，然后再把响应（带上前端传来的id）通过调用`js`的`window.WebViewJavascriptBridge`的指定方法传递给前端，前端再通过`id`找到之前存储的回调，进行执行。

2.原生调用`js`

首先前端需要事先注册要监听的事件，把事件名和回调保存起来，然后原生在某个时刻会调用`js`的`window.WebViewJavascriptBridge`对象的指定方法，前端根据返回参数的事件名找到注册的回调进行执行，同时原生也会传过来一个`id`，如果前端执行完相应逻辑后还要给原生回消息，那么要把该`id`带回去，原生根据该`id`来找到对应的回调进行执行。

可以看到，`js`和原生两边的逻辑都是一致的。



# ios

`ios`和安卓基本是一致的，部分细节上有点区别，首先是协议不一样，`ios`的是这样的：

```js
var CUSTOM_PROTOCOL_SCHEME_IOS = 'https';
var QUEUE_HAS_MESSAGE_IOS = '__wvjb_queue_message__';
```

然后`ios`初始化创建`iframe`的时候会发送一个请求：

```js
var BRIDGE_LOADED_IOS = '__bridge_loaded__';
function _createQueueReadyIframe (doc) {
    messagingIframe = doc.createElement('iframe');
    messagingIframe.style.display = 'none';
    if (isIphone()) {
        // 这里应该是ios需要先加载一下bridge
        messagingIframe.src = CUSTOM_PROTOCOL_SCHEME_IOS + '://' + BRIDGE_LOADED_IOS;
    }
    doc.documentElement.appendChild(messagingIframe);
}
```

再然后是`ios`获取我们的消息队列时不需要通过`iframe`，它能直接获取执行`js`函数返回的数据：

```js
function _fetchQueue () {
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    return messageQueueString;// 直接返回，不需要通过iframe
}
```

其他部分都是一样的。



# 总结

本文分析了一下`jsBridge`的源码，可以发现其实是个很简单的东西，但是平时可能就没有去认真了解过它，总想做一些”大“的事情，以至于沦为了一个”好高骛远“的人，希望各位不要像笔者一样。

另外本文分析的只是笔者公司的`jsBridge`实现，可能有不一样、更好或更新的实现，欢迎留言探讨。



