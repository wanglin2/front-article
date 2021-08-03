
前几天一个简单的下载图片的需求折腾了我后端大佬好几天，最终还是需要前端来搞，开始说不行的笔者最后又行了，所以趁着这个机会来总结一下下载图片到底有多少种方法。



# 先起个服务

使用[expressjs](https://expressjs.com/)起个简单的后端服务，先安装：

```bash
mkdir demo
cd demo
npm init
npm install express --save// v4.17.1
```

然后创建一个`app.js`文件，输入：

```js
const express = require('express')
const app = express()
app.get('/', (req, res) => {
    res.send('hello world')
})
app.listen(3000, () => {
    console.log('服务启动完成')
})
```

然后在命令行输入：`node app.js`，访问`http://localhost:3000/`，页面显示`hello world`即表示服务启动成功。

接下来分别模拟几种情况：

- 情况1.静态图片

创建一个`public`文件夹，随便拷贝一张图片比如`test.jpg`进去，然后添加以下代码：

```js
// ...
app.use(express.static('./public'))
// app.listen...
```

浏览器访问`http://localhost:3000/test.jpg`即可看到该图片。

- 情况2.读取图片文件，以流的形式返回

```js
app.get('/getFileStream', (req, res) => {
    const fileName = req.query.name
    const stream = fs.createReadStream(path.resolve('./public/' + fileName))
    stream.pipe(res)
})
```

浏览器访问`http://localhost:3000/getFileStream?name=test.jpg`即可访问到该图片。

- 情况3.读取图片文件返回流并添加`Content-Disposition`响应头

`Content-Disposition`响应头是`MIME`协议的扩展，用来告诉浏览器如何处理服务器发送的文件，有三种取值：

```
Content-Disposition: inline// 如果浏览器能直接打开该文件会直接打开，否则触发保存
Content-Disposition: attachment// 告诉浏览器以附件的形式发送，会直接触发保存，会以接口的名字作为默认的文件名
Content-Disposition: attachment; filename="xxx.jpg"// 告诉浏览器以附件的形式发送，会直接触发保存，filename的值作为默认的文件名
```

```js
app.get('/getAttachmentFileStream', (req, res) => {
    const fileName = req.query.name
    // attachment方法实际上设置了两个响应头的值：
    /*
        Content-Disposition: attachment; filename="【文件名】"
		Content-Type: 【文件MIME类型】
    */
    res.attachment(fileName); 
    const stream = fs.createReadStream(path.resolve('./public/' + fileName))
    stream.pipe(res)
})
```

- 情况4.动态生成图片返回流

我们以生成二维码为例，使用[qr-image](https://github.com/alexeyten/qr-image)这个库来创建二维码，添加以下代码：

```js
const qr = require('qr-image')
app.get('/createQrCode', (req, res) => {
    // 生成二维码只读流
    const data = qr.image(req.query.text, {
        type: 'png'
    });
    data.pipe(res)
})
```

- 情况5.返回`base64`字符串

```js
app.get('/createBase64QrCode', (req, res) => {
    const data = qr.image(req.query.text, {
        type: 'png'
    });
    const chunks = []
    let size = 0
    data.on('data', (chunk) => {
        chunks.push(chunk)
        size += chunk.length
    })
    data.on('end', () => {
        const data = Buffer.concat(chunks, size)
        const base64 = `data:image/png;base64,` + data.toString('base64')
        res.send(base64)
    })
})
```

- 情况6.上述几种情况的post请求方式

```js
// 解析json类型的请求体
app.use(express.json())
// 解析urlencoded类型的请求体
app.use(express.urlencoded())
app.post('/getFileStream', (req, res) => {
    const fileName = req.body.name
    const stream = fs.createReadStream(path.resolve('./public/' + fileName))
    stream.pipe(res)
})
app.post('/getAttachmentFileStream', (req, res) => {
    const fileName = req.body.name
    res.attachment(fileName);
    const stream = fs.createReadStream(path.resolve('./public/' + fileName))
    stream.pipe(res)
})
app.post('/createQrCode', (req, res) => {
    const data = qr.image(req.body.text, {
        type: 'png'
    });
    data.pipe(res)
})
```



# 一.a标签下载

`a`标签`html5`版本新增了`download`属性，用来告诉浏览器下载该`url`，而不是导航到它，可以带属性值，用来作为保存文件时的文件名，尽管说有同源限制，但是我实际测试时非同源的也是可以下载的。

对于没有设置`Content-Disposition`响应头或者设置为`inline`的图片来说，因为图片对于浏览器来说是属于能打开的文件，所以并不会触发下载，而是直接打开，浏览器不能预览的文件无论有没有`Content-Disposition`头都会触发保存：

```html
<!-- 直接打开 -->
<a href="/test.jpg" download="test.jpg" target="_blank">jpg静态资源</a>
<!-- 触发保存 -->
<a href="/test.zip" download="test.pdf" target="_blank">zip静态资源</a>
<!-- 触发保存 -->
<a href="https://www.7-zip.org/a/7z1900-x64.exe" download="test.zip" target="_blank">三方exe静态资源</a>
<!-- 直接打开 -->
<a href="/createQrCode?text=http://lxqnsys.com/" download target="_blank">二维码流</a>
<!-- 直接打开 -->
<a href="/getFileStream?name=test.jpg" download target="_blank">jpg流</a>
<!-- 触发保存 -->
<a href="/getFileStream?name=test.zip" download target="_blank">zip流</a>
<!-- 触发保存 -->
<a href="/getAttachmentFileStream?name=test.jpg" download target="_blank">附件jpg流</a>
<!-- 触发保存 -->
<a href="/getAttachmentFileStream?name=test.zip" download target="_blank">附件zip流</a>
```

所以说如果想用`a`标签下载图片，那么要让后端加上`Content-Disposition`响应头，另外也必须以流的形式返回，跨域图片符合这个要求也可以下载，即使响应没有允许跨域的头，但是静态图片即使添加了这个头也是直接打开：

```js
// 经测试，浏览器仍然直接打开图片
app.use(express.static('./public', {
    setHeaders(res) {
        res.attachment()
    }
}))
```

和`a`标签方式类似的还可以使用`location.href`：

```js
location.href = '/test.jpg'
location.href = '/test.zip'
```

行为和`a`标签完全一致。

这两种方式的缺点也很明显，一是不支持`post`等其他方式的请求，二是需要后端支持。



# 二.`base64`格式下载

`a`标签支持`data:`协议的`URL`，利用这个可以让后端返回`base64`格式的字符串，然后使用`download`属性进行下载：

```vue
<template>
    <a :href="base64Img" download target="_blank">base64字符串</a>
</template>
<script>
import axios from 'axios'
export default {
  data () {
    return {
      base64Img: ''
    }
  },
  async created () {
    let { data } = await axios.get('/createBase64QrCode?text=http://lxqnsys.com/')
    this.base64Img = data
  }
}
</script>
```

这个方式就随便`get`还是`post`请求了，缺点是`base64`字符串可能会非常大，传输慢以及浪费流量，另外当然也得后端支持，需要同域或允许跨域。



# 三.`blob`格式下载

还是`a`标签，它还支持`blob:`协议的`URL`，利用这个可以把响应类型设置为`blob`，然后和`base64`一样扔给`a`标签：

```vue
<template>
    <a :href="blobData" download target="_blank">blob</a>
</template>
<script>
import axios from 'axios'
export default {
  data () {
    return {
      blobData: null,
      blobDataName: ''
    }
  },
  async created () {
    let { data } = await axios.get('/test.jpg', {
      responseType: 'blob'
    })
    const blobData = URL.createObjectURL(data)
    this.blobData = blobData
  }
}
</script>
```

这个方式需要和上述几个需要通过`ajax`请求的一样，都需要后端可控，即图片同域或支持跨域。



# 四.使用`canvas`下载

这个方法其实和方法二和方法三是类似的，只是相当于把图片请求方式换了一下：

```vue
<template>
	<a :href="canvasBase64Img" download target="_blank">canvas base64字符串</a>
	<a :href="canvasBlobImg" download target="_blank">canvas blob</a>
</template>

<script>
    export default {
        data () {
            return {
                canvasBase64Img: '',
                canvasBlobImg: null
            }
        },
        created () {
            const img = new Image()
            // 跨域图片需要添加这个属性，否则画布被污染了无法导出图片
            img.setAttribute('crossOrigin', 'anonymous')
            img.onload = () => {
                let canvas = document.createElement('canvas')
                canvas.width = img.width
                canvas.height = img.height
                let ctx = canvas.getContext('2d')
                // 图片绘制到canvas里
                ctx.drawImage(img, 0, 0, img.width, img.height)
                // 1.data:协议
                let data = canvas.toDataURL()
                this.canvasBase64Img = data
                // 2.blob:协议
                canvas.toBlob((blob) => {
                    const blobData = URL.createObjectURL(blob)
                    this.canvasBlobImg = blobData
                })
            }
            img.src = '/createQrCode?text=http://lxqnsys.com/'
        }
    }
</script>
```

`img`标签是可以跨域的，但是跨域的图片绘制到`canvas`里后无法导出，浏览器会报错，可以给`img`添加`crossOrigin`属性，但是，如果图片没有允许跨域的头加了也没用。



# 五.表单形式下载

对于`post`请求方式下载图片的话，除了使用上述的方法二和方法三之外，还可以使用`form`表单：

```vue
<template>
    <el-button type="primary" @click="formType">from表单下载</el-button>
  </div>
</template>

<script>
export default {
  methods: {
    formType () {
      // 创建一个隐藏的表单
      const form = document.createElement('form')
      form.style.display = 'none'
      form.action = '/getAttachmentFileStream'
      // 发送post请求
      form.method = 'post'
      form.target = '_blank'
      document.body.appendChild(form)
      const params = {
        name: 'test.jpg'
      }
      // 创建input来传递参数
      for (let key in params) {
        let input = document.createElement('input')
        input.type = 'hidden'
        input.name = key
        input.value = params[key]
        form.appendChild(input)
      }
      form.submit()
      form.remove()
    }
  }
}
</script>
```

使用该方式，图片流的响应头需要设置`Content-Disposition`，否则浏览器也是直接打开图片，有该响应头的话跨域图片也可以下载，即使图片不允许跨域。



# 六.`ifrmae`下载

`document.execCommand`有一个`SaveAs`命令，可以触发浏览器的另存为行为，利用这个可以把图片加载到`iframe`里，然后通过`iframe`的`document`来触发该命令：

```vue
<template>
	<el-button type="primary" @click="iframeType">iframe下载</el-button>
</template>

<script>
    export default {
        methods: {
            iframeType () {
                const iframe = document.createElement('iframe')
                iframe.style.display = 'none'
                iframe.onload = () => {
                    iframe.contentWindow.document.execCommand('SaveAs')
                    document.body.removeChild(iframe)
                }
                iframe.src = '/createQrCode?text=http://lxqnsys.com/'
                document.body.appendChild(iframe)
            }
        }
    }
</script>
```

图片必须要是同源的，这种方式了解一下就行，因为它只在`IE`里被支持。



# 小结

本文简单分析了一下前端下载图片的各种方式，各位可以根据实际需求进行选择，除了最后一种方法，其余方法均未在`IE`上测试，有需要的可以自行测试。

`demo`代码在[https://github.com/wanglin2/download-image-demo](https://github.com/wanglin2/download-image-demo)。
