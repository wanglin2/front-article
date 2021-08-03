> 博客：[http://lxqnsys.com](http://lxqnsys.com)
> 公众号：理想青年实验室

# 前言

好久不见，距离OpenLayers入门第一篇已经过了很久，为什么迟迟没有后续呢，主要有两个原因，一是因为近期项目里使用地图的部分比较少，二是因为很多时候即使功能做出来了，但是还是不能完全理解，不是很明白的东西除了贴代码之外也写不了啥，其实第一篇也是很基础很简单的，但是意外的是看的人是最多的，这让我意识到可能即使是贴一下代码对一些人也是有帮助的，这就是这一篇的主要目的，可能有一些地方会看不懂，但是不要问，问我也不知道，如果你恰好了解的话十分欢迎在评论里分享，感谢~

首先来分享一个我无意中找到的教程，[http://linwei.xyz/ol3-primer/index.html](http://linwei.xyz/ol3-primer/index.html)。虽然是基于v3版本介绍的，很多api可能变了，但还是值得一看，除了OpenLayers本身的介绍，还会有一些地理基础知识的分享，这种相对全面的中文教程真的很稀有，且看且珍惜。

接下来分享一些常用的在线地图瓦片资源：

1.高德瓦片，最大支持放大到20级，字体比较大，但是最近好像又只能到19级了。

```text
http://wprd0{1-4}.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scl=1&style=7
```

2.高德瓦片，最大支持放大到20级，颜色偏灰绿色。

```text
http://webst0{1-4}.is.autonavi.com/appmaptile?style=7&x={x}&y={y}&z={z}
```

3.高德瓦片，最大支持放大到18级，最常用的样式。

```text
http://webrd01.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scale=1&style=8
```

4.谷歌地图瓦片，最大支持放大到22级，颜色偏绿色。

```text
https://mt0.google.cn/vt/lyrs=m&hl=zh-CN&gl=cn&x={x}&y={y}&z={z}
```

# 绘制多边形

```js
import Feature from 'ol/Feature'
import Polygon from 'ol/geom/Polygon'
import { Vector as VectorSource } from 'ol/source'
import { Style, Stroke, Fill } from 'ol/style'
import { Vector as VectorLayer } from 'ol/layer'

// data为多边形每个点的经纬度坐标数组，[[120.11997452699472, 30.314227730637967],[120.11997452699472, 30.314227730637967],...]
function renderArea (data) {
    // 创建要素
    const features = [
        new Feature({
            geometry: new Polygon([data])// 使用多边形类型
        })
    ]
    // 创建矢量数据源
    const source = new VectorSource({
        features
    })
    // 创建样式
    const style = new Style({
        stroke: new Stroke({
            color: '#4C99F8',
            width: 3,
            lineDash: [5]
        }),
        fill: new Fill({
            color: 'rgba(255,255,255,0.1)'
        })
    })
    // 创建矢量图层
    const areaLayer = new VectorLayer({
        source,
        style,
        zIndex: 1
    })
    // 添加到地图实例
    map.addLayer(areaLayer)
}
```

多边形的绘制很简单，使用几何类型里的多边形类创建一个要素就可以了。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d3616c3234a493baea5b8279ff1c502~tplv-k3u1fbpfcp-zoom-1.image)

区域中间的名字显示可以通过`Overlay`叠加层来显示，主要是要计算一下显示的位置：

```js
import Overlay from 'ol/Overlay';
import { boundingExtent } from 'ol/extent';
import { getCenter } from 'ol/extent';
import { fromLonLat } from 'ol/proj';

// 获取一个多边形的四个边界点，data:[[120.11997452699472, 30.314227730637967],[120.11997452699472, 30.314227730637967],...]
function getExtent (data) {
    let minx = 99999999;
    let miny = 99999999;
    let maxx = -99999999999;
    let maxy = -99999999999;
    data.forEach((item) => {
        if (item[0] > maxx) {
            maxx = item[0];
        }
        if (item[0] < minx) {
            minx = item[0];
        }
        if (item[1] > maxy) {
            maxy = item[1];
        }
        if (item[1] < miny) {
            miny = item[1];
        }
    });
    return [Number(minx), Number(miny), Number(maxx), Number(maxy)];
}
// 也可以直接使用工具方法：boundingExtent
function getExtent (data) {
    return boundingExtent(data)
}
// 获取范围的中心点坐标
let center = getCenter(getExtent(data));
// 显示名称
let nameEl = document.createElement('div')
nameEl.className = 'name'
nameEl.innerHTML = '我是名称'
let nameOverlay = new Overlay({
    position: fromLonLat(center, 'EPSG:4326'),
    element: nameEl,
    offset: [0, 0],
    positioning: 'bottom-center'
})
map.addOverlay(nameOverlay)
```



# 绘制以米为单位的圆

```js
import Feature from 'ol/Feature'
import { circular } from 'ol/geom/Polygon'
import { Vector as VectorSource } from 'ol/source'
import { getPointResolution } from 'ol/proj'
import { METERS_PER_UNIT } from 'ol/proj/Units'
import Circle from 'ol/geom/Circle'
import { Style, Stroke, Fill } from 'ol/style'
import { Vector as VectorLayer } from 'ol/layer'

// 两种方式

// 1.使用 circular绘制
function renderRangeUseCircular (center, radius) {
    const features = []
    features.push(new Feature({
        geometry: circular(center, radius)
    }))
    return new VectorSource({
        features
    })
}

// 2.使用Circle绘制圆
function renderRangeUseCircle (center, projection = 'EPSG:4326', radius) {
    const view = map.getView()
    const resolutionAtEquator = view.getResolution()
    const pointResolution = getPointResolution(projection, resolutionAtEquator, center, METERS_PER_UNIT.m)
    const resolutionFactor = resolutionAtEquator / pointResolution
    radius = (radius / METERS_PER_UNIT.m) * resolutionFactor
    const circle = new Circle(center, radius)
    const circleFeature = new Feature(circle)
    const vectorSource = new VectorSource({
        projection: projection
    })
    vectorSource.addFeature(circleFeature)
    return vectorSource
}

// 绘制
function renderRange () {
    const source = renderRangeUseCircle(...params)
    // const source = renderRangeUseCircular(...params)
    const style = new Style({
        stroke: new Stroke({
            color: '#4C99F8',
            width: 3,
            lineDash: [5]
        }),
        fill: new Fill({
            color: 'rgba(76,153,248,0.3)'
        })
    })
    rangeLayer = new VectorLayer({
        source,
        style,
        zIndex: 2
    })
    map.addLayer(rangeLayer)
}
```

绘制圆有两种方式，分别是使用`circular`和`Circle`这两者有什么区别我也不太清楚，它们的入参基本一样，中心点和半径，文档上没有指出半径的单位，第二种方法是百度上搜到的，绘制完经测距测试后是准确的。

# 添加阴影效果

OpenLayers的样式对象并不支持直接设置阴影效果，所以需要获取到canvas的绘图上下文来自行添加，原理是监听图层的`prerender`（在一个图层渲染前触发）和`postrender`（在一个图层渲染后触发）`事件，修改`canvas`上下文的绘图样式，对整个图层都是有影响的，所以最好把要添加阴影的要素放到一个单独的图层里：

```js
import { Vector as VectorSource } from 'ol/source'
import { Style, Stroke, Fill } from 'ol/style'
import { Vector as VectorLayer } from 'ol/layer'

const source = new VectorSource({
    features: []
})
const style = new Style({
    stroke: new Stroke({
        color: '#437AF6',
        width: 2
    }),
    fill: new Fill({
        color: 'rgba(33,150,243,0.20)'
    })
})
let vectorLayer = new VectorLayer({
    source,
    style
})
// 添加阴影
vectorLayer.on('prerender', evt => {
    evt.context.shadowBlur = 4
    evt.context.shadowColor = 'rgba(0,0,0,0.20)'
})
vectorLayer.on('postrender', evt => {
    evt.context.shadowBlur = 0
    evt.context.shadowColor = 'rgba(0,0,0,0.20)'
})
map.addLayer(vectorLayer)
```

# 绘制带边框的线段

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5ebf6ec912c4f9f81728b416beea991~tplv-k3u1fbpfcp-zoom-1.image)

OpenLayers是不直接支持这种带边框的线段的，所以一种简单的方法是绘制两条线段叠加起来，上面的宽度比下面的低，就有边框效果了：

```js
import Polygon from 'ol/geom/Polygon'
import Feature from 'ol/Feature'
import { Vector as VectorSource } from 'ol/source'
import { Style, Stroke, Fill } from 'ol/style'
import { Vector as VectorLayer } from 'ol/layer'

// 创建多边形
const features = [
    new Feature({
        geometry: new Polygon([]),
    })
]

// 下层线段，用来做边框，宽度更宽
const source = new VectorSource({
    features
})
const style = new Style({
    stroke: new Stroke({
        color: '#53AA08',
        width: 8
    }),
    fill: new Fill({
        color: 'rgba(151,255,201,0.23)'
    })
})
areaLayer = new VectorLayer({
    source,
    style,
    zIndex: 1
})
map.addLayer(areaLayer)

// 上层线段，用来做中间部分，宽度较小
const source2 = new VectorSource({
    features
})
const style2 = new Style({
    stroke: new Stroke({
        color: '#2AE000',
        width: 2
    })
})
areaLayer2 = new VectorLayer({
    source: source2,
    style: style2,
    zIndex: 2
})
map.addLayer(areaLayer2)
```

本次分享就这么多了，下次见~
