笔者开源了一个`Web`思维导图[mind-map](https://github.com/wanglin2/mind-map)，数据默认是存储在`localstorage`里，如果想保存到本地文件，需要使用导出功能，下次打开再使用导入功能，编辑完如果又想保存到文件，那么又需要从重新导出覆盖原来的文件，不得不说，可以但不优雅，所以最近增加了直接编辑本地文件的能力，体验了一下，还是不错的，并且就是调调`API`的事情，很简单，何乐而不为。

主角就是[showOpenFilePicker](https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker)和[showSaveFilePicker](https://developer.mozilla.org/en-US/docs/Web/API/Window/showSaveFilePicker)两个`API`，笔者基于它俩开发了三个功能：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dae962aaa8ce427eb8539bce66b1f058~tplv-k3u1fbpfcp-zoom-1.image)


`新建`和`另存为`其实一样的，只不过一个保存的是空数据，一个是当前的数据，当创建或打开文件成功后，操作的时候数据会直接保存到本地文件里，不再需要进行手动的导出，这种体验其实就和本地编辑器没什么区别了。

# 打开

先来看看打开文件，调用的是`showSaveFilePicker`方法，返回一个`Promise`，选择文件成功了那么`Promise`的结果是一个数组，每一项代表一个文件的操作句柄：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a82496d1c4d43be83200721c386e4cb~tplv-k3u1fbpfcp-zoom-1.image)


如果要获取某个文件的内容或写入某个文件就需要通过这些文件句柄对象。如果没有选择或选择失败了`Promise`则会出错：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94a02a85fd184a63ac1fee5012580070~tplv-k3u1fbpfcp-zoom-1.image)


这个方法接收一个选项对象作为参数：

- `options.multiple`

布尔值，设置是否可以选择多个文件。

- `options.types`

一个数组，设置允许被选择的文件类型，数组每一项都是一个对象：

```js
{
	description: '',
	accept: {
		'': []
	}
}
```

`description`用于说明，好像没什么用，`accept`是个对象，`key`为[MIME type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types)，`value`为一个数组，代表允许的文件扩展名。

如果`MIME type`设置的很具体，比如`application/json`，那么`value`不传的话只能选择文件后缀为`.json`的文件，如果`value`设置了扩展名的话，则在默认的`.json`文件外还允许选择设置的扩展名的文件，比如设置为`['.smm']`，那么`.json`和`.smm`为后缀的文件都可以选择：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/833a8cafb7184e7fbff79dda4f0a4925~tplv-k3u1fbpfcp-zoom-1.image)


如果`MIME type`设置的比较宽泛的话，比如`application/*`，那么所有``MIME type``为`application`类型的文件都可以选择，就算`value`只设置了一个`.json`，其他类型的文件也是可以选择的，所以`value`的作用不是限制，而是扩充。

但是呢，这种限制可以轻松突破，只要点击扩展名打开下拉列表选择所有文件选项，那么还是想选什么文件就选什么文件，有朋友知道怎么解决的欢迎评论区留言。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/affac5fbd0d14f09922315c27839fd45~tplv-k3u1fbpfcp-zoom-1.image)


- `options.excludeAcceptAllOption`

布尔值，默认为`false`，即允许不配置`types`选项，支持选择所有文件，如果设为`true`，那么`types`选项不能为空，必须要限制一种文件类型。

笔者的思维导图文件格式使用的是`.json`，并且吃饱了撑的自己定义了一个格式`.smm`，其实就是`json`，并且同一时间只能编辑一个文件，那么打开文件的代码如下所示：

```js
let fileHandle = null
async openLocalFile() {
    try {
        let [ _fileHandle ] = await window.showOpenFilePicker({
            types: [
                {
                    description: '',
                    accept: {
                        'application/json': ['.smm']
                    }
                },
            ],
            excludeAcceptAllOption: true,
            multiple: false
        });
        if (!_fileHandle) {
            return
        }
        fileHandle = _fileHandle
        if (fileHandle.kind === 'directory') {
            this.$message.warning('请选择文件')
            return
        }
        this.readFile()
    } catch (error) {
        if (error.toString().includes('aborted')) {
            return
        }
        this.$message.warning('你的浏览器可能不支持哦')
    }
}
```

将文件句柄保存起来，接下来都会基于它来操作文件，先来看看文件句柄对象，它存在两个方法：

- `getFile()`

返回一个`Promise`，获取该句柄所对应的文件对象，其实就是我们常见的`File`对象：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ee189ead339424b9f3a226ed589c25c~tplv-k3u1fbpfcp-zoom-1.image)


- `createWritable()`

返回也是一个`Promise`，创建一个可以写入文件的文件流对象：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13285ccdaf2446149ac3c1a74f552d33~tplv-k3u1fbpfcp-zoom-1.image)


基于这两个方法我们就可以读取打开文件的内容及把新内容写入文件：

```js
// 读取文件
async readFile() {
    let file = await fileHandle.getFile();
    let fileReader = new FileReader();
    fileReader.onload = async () => {
        // fileReader.result
    }
    fileReader.readAsText(file);
}

// 写入文件
async writeLocalFile(content) {
    if (!fileHandle) {
        return;
    }
    let string = JSON.stringify(content);
    const writable = await fileHandle.createWritable();
    await writable.write(string);
    await writable.close();
}
```

页面内第一次调用`createWritable`方法浏览器会弹个窗询问用户是否允许：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3cb9586440da41078a9a769a87049b14~tplv-k3u1fbpfcp-zoom-1.image)


每调用一次`createWritable`方法都会在你的本地创建一个`.crswap`文件：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3720751daa345c0b62c693bcd550118~tplv-k3u1fbpfcp-zoom-1.image)


相当于一个临时文件，没有调用写入流`writable`的`close`方法前，调用它的`write`方法写入的内容默认都保存在这个文件，只有调用`close`以后才会更新到源文件，并且自动删除这个临时文件，另外页面关闭，也会删除这些文件。

写入流默认是空的，每调用一次`write`方法，都会在`.crswap`中追加内容，但是可以指定写入的位置：

```js
await writable.write({ type: "write", position: 0, data: string });
```

这样会从指定的字节数开始写入，注意是替换，而不是插入。

所以为了方便起见，最好还是创建、写入就关闭，再写再创建。

# 新建

新建调用的是`showSaveFilePicker`方法，也接收一个选项对象为参数，有两个选项和`showOpenFilePicker`方法是一样的，即`types`和`excludeAcceptAllOption`，之外还有一个选项：

- `suggestedName`

默认填充的文件名称，为空则创建文件时输入框就是空的。


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/346687a34f6945b5b1dae6e778dd92dd~tplv-k3u1fbpfcp-zoom-1.image)



![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62abb407f9d64ab6963050fca3d95c26~tplv-k3u1fbpfcp-zoom-1.image)


可以直接输入文件名创建新文件，也可以点击已经存在的文件进行替换。

创建成功返回的也是一个文件句柄，那么创建文件就很简单了：

```js
async createLocalFile(content) {
    try {
        let _fileHandle = await window.showSaveFilePicker({
            types: [{
                description: '',
                accept: {'application/json': ['.smm']},
            }],
            suggestedName: '思维导图'
        });
        if (!_fileHandle) {
            return;
        }
        const loading = this.$loading({
            lock: true,
            text: '正在创建文件',
            spinner: 'el-icon-loading',
            background: 'rgba(0, 0, 0, 0.7)'
        });
        fileHandle = _fileHandle;
        await this.writeLocalFile(content);
        await this.readFile();
        loading.close();
    } catch (error) {
        if (error.toString().includes('aborted')) {
            return
        }
        this.$message.warning('你的浏览器可能不支持哦');
    }
}
```

来看看实际效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a5b917eb1d84ea18dd3618ec247bcb8~tplv-k3u1fbpfcp-zoom-1.image)



# 总结

最后再来看看兼容性：


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97ccc561eda14ae0a6f40d3e7284c30b~tplv-k3u1fbpfcp-zoom-1.image)


因为目前还是实验性质，所以可以看到是一片红，但是因为我的本身也只是一个示例项目，所以问题不大，有胜于无。

另外这个特性目前也只能在`HTTPS`协议或`localhost`下才可用，其他情况下`window`对象是不存在这两个`API`的，所以需要做好错误处理。
