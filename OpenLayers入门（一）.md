# OpenLayers简介

OpenLayers（[https://openlayers.org/](https://openlayers.org/)）是一个用来帮助开发Web地图应用的高性能的、功能丰富的JavaScript类库，可以满足几乎所有的地图开发需求。

有如下特点：

1. 支持任何XYZ瓦片资源，同时也支持OGC的WMTS规范的瓦片服务以及ArcGIS规范的瓦片服务
2. 支持矢量切片，包括pbf、GeoJSON、TopoJSON格式
3. 支持矢量图层，能渲染GeoJSON、TopoJSON、KML、GML和其他格式的矢量数据
4. 支持OGC制定的WMS、WFS等GIS网络服务规范
5. 支持在移动设备上运行
6. 可以通过css来为地图控件设置样式
7. 面向对象开发方式，在OpenLayers中万物皆对象

和另一个流行的地图库leaflet不同，openLayers完全是用面向对象的方式开发的，且几乎内置了所有地图开发需要的功能，而leaflet核心库只提供基本功能，其他功能都是通过第三方插件进行扩展。使用上来说leaflet更容易上手，OpenLayers上手难度比较大，所以业务可预见较为简单的建议采用leaflet。

OpenLayers虽然很强大，但是因为一切皆对象，所以使用起来很麻烦，再加上无比难看的文档，所以对新手极其不友好，这也是本系列文章的初衷，旨在基于实际业务开发的场景下来沉淀一些内容，来帮助新手使用OpenLayers。

这是本系列的第一篇，主要介绍地图的实例化、基本的要素操作，后续不定期更新。

本文基于OpenLayers v6+版本，代码基于Vue。



# 安装

```
npm i ol
```



# 实例化地图

要显示一个基本的地图首先需要提供一个容器，设置好宽高，然后引入OpenLayers，添加一个地图图层，地图服务可以使用内置的一个开源地图OSM，也可以使用其他的在线瓦片服务，比如：百度、高德、天地图、必应、谷歌等，具体服务地址可以自行百度，本文使用的是高德的服务，详情可参考：[https://www.jianshu.com/p/e34f85029fd7](https://www.jianshu.com/p/e34f85029fd7)。

```html
<div class="ol-map" ref="olMap"></div>
```

```js
import Map from 'ol/Map'
import View from 'ol/View'
import { Tile as TileLayer } from 'ol/layer'
import {XYZ, OSM} from 'ol/source'
import { fromLonLat } from 'ol/proj'

// fromLonLat方法能将坐标从经度/纬度转换为其他投影

// 使用内置的OSM
//const tileLayer = new TileLayer({
//    source: new OSM()
//})
// 使用高德
const tileLayer = new TileLayer({
    source: new XYZ({
        url: 'https://webrd01.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}'
    })
})
let map = new Map({
    layers: [tileLayer],
    view: new View({
        center: fromLonLat([120.771441, 30.756433]),//地图中心点
        zoom: 15,// 缩放级别
        minZoom: 0,// 最小缩放级别
        maxZoom: 18,// 最大缩放级别
        constrainResolution: true// 因为存在非整数的缩放级别，所以设置该参数为true来让每次缩放结束后自动缩放到距离最近的一个整数级别，这个必须要设置，当缩放在非整数级别时地图会糊
    }),
    target: this.$refs.olMap// DOM容器
})
```

这样就可以显示一个基本的地图：


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a686c736999a?w=576&h=395&f=png&s=197671)

可以拖动和缩放，但是不能旋转，如果需要支持旋转，需要加上旋转交互：

```js
import {
  defaults as defaultInteractions,
  DragRotateAndZoom,
} from 'ol/interaction'

let map = new Map({
    // ...
    interactions: defaultInteractions().extend([new DragRotateAndZoom()])
})
```

这样就可以按住shift键时通过鼠标来进行旋转地图。

OpenLayers有内置很多开箱即用的控件，常用的使用如下：

```js
import { defaults, FullScreen, MousePosition, ScaleLine } from 'ol/control'

let map = new Map({
    // ...
    controls: defaults().extend([
        new FullScreen(), // 全屏
        new MousePosition(), // 显示鼠标当前位置的经纬度
        new ScaleLine()// 显示比例尺
    ])
})
```

地图也有很多事件，可以监听所需要的事件来进行对应的操作，使用如下：

```js
map.on('moveend', e => {
    // console.log('地图移动', e)
})
map.on('rendercomplete', () => {
    // console.log('渲染完成')
})
map.on('click', e => {
    // console.log('地图点击', e)
})
```

显示地图基本就到这里，接下来看一些常见的使用场景。



# 显示要素

在地图上显示一些自定义元素可以说是最基本也是最常见的需求，如果要显示的元素结构或样式比较复杂，可以使用Overlay，它可以将DOM元素在地图上进行显示，并将随地图一起移动。

```js
import Overlay from 'ol/Overlay'

// 你可以给元素添加任意的内容或属性或样式，也可以给元素绑定事件
let el = document.createElement('div')
let marker = new Overlay({
    element: el,// 要显示的元素
    position: fromLonLat([longitude, latitude], 'EPSG:4326'),// 地图投影的位置
    offset: [-17, -17], // 元素显示的像素偏移量
    autoPan: true, // 自动移动地图以完整的显示元素
})
// 添加到地图
map.addOverlay(marker)
// 从地图上删除
map.removeOverlay(marker)
```

如果是显示一个小icon、多边形、线之类的需要使用矢量对象Feature，先看如何显示一个图片icon：

```js
import Feature from 'ol/Feature'
import Point from 'ol/geom/Point'
import { Vector as VectorLayer } from 'ol/layer'
import { Vector as VectorSource } from 'ol/source'
import { Style, Icon } from 'ol/style'

// 实例化要素
let feature = new Feature({
    geometry: new Point([120.12636255813723, 30.313142215804806])// 地理几何图形选用点几何
})
// 如果需要给要素附加一些自定义数据
feature.set('data', data)
// 设置样式，这里就是显示一张图片icon
feature.setStyle([
    new Style({
        image: new Icon({
          anchor: [0.5, 1],// 显示位置
          size: [18, 28],// 尺寸
          src: require('../../assets/images/mouse_location_ing.png')// 图片url
        })
    })
])
// 矢量源
let source = new VectorSource({
    features: [feature]
})
// 实例化的时候也可以不添加feature，后续通过方法添加：source.addFeatures([feature])
// 清空feature：source.clear()

// 矢量图层
let vector = new VectorLayer({
    source: source
})
// 样式除了可以设置在单个feature上，也可以统一设置在矢量图层上
/*
let vector = new VectorLayer({
    source: source,
    style: new Style({
        image: new Icon({
          anchor: [0.5, 1],// 显示位置
          size: [18, 28],// 尺寸
          src: require('../../assets/images/mouse_location_ing.png')// 图片url
        })
    })
})
*/
map.addLayer(vector)
```

上面就实现了添加一个icon要素到地图上，如果要添加多个的话实例化多个Feature就好了，效果如下：


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a68bf26e8ef7?w=574&h=389&f=png&s=129763)

有时还需要支持能拖动要素来修改它的位置，实现这个需要Translate交互的支持：

```js
import {Translate} from 'ol/interaction'
// ...

// ...
let translate = new Translate({
    layers: [vector]
})
map.addInteraction(translate)
// 可以监听一下拖动开始和结束的事件，拖动后的经纬度可以从e里面获取
translate.on('translateend', (e) => {
    console.log(e)
})
translate.on('translatestart', (e) => {
    console.log(e)
})
```

除了直接在地图上显示，也可以自己进行添加，即在鼠标点击的位置上添加一个要素，这需要使用到Draw交互：

```js
import { Draw } from 'ol/interaction'

let draw = new Draw({
    source: source,
    type: 'Point',
    style: new Style({
        image: new Icon({
          anchor: [0.5, 1],// 显示位置
          size: [18, 28],// 尺寸
          src: require('../../assets/images/mouse_location_ing.png')// 图片url
        })
    })
})
// 监听完成事件
draw.on('drawend', (e) => {
    console.log(e)
    // 如果只需要放置一个的话可以移除该交互，否则可以一直添加
    map.removeInteraction(draw)
    
})
map.addInteraction(draw)
```

因为icon多了的话不知道某个icon到底代表的是啥，所以常常需要给icon添加一个tooltip，当鼠标移上去的时候显示，怎么实现呢，其实tooltip本质上就是一个DOM元素，上面已经介绍过Overlay了，用它就可以实现，请看：

```html
<!--可以给元素设置一些样式-->
<div class="ol-popup" ref="olPopup">{{olPopupText}}</div>
```

```js
import Overlay from 'ol/Overlay'

// 创建Overlayer
this.tooltipOverlay = new Overlay({
    element: this.$refs.olPopup,
    positioning: 'bottom-center',// 根据position属性的位置来进行相对点位
    offset: [0, -30],// 在positioning之上再进行偏移
    autoPan: true
})
map.addOverlay(this.tooltipOverlay)

// 给地图绑定鼠标移动事件，检测鼠标位置所在是否存在feature，如果是目标feature的话就显示tooltip
map.on('pointermove', (e) => {
    this.olPopupText = ''
    map.forEachFeatureAtPixel(e.pixel, (f, layer) => {
        if (layer !== this.vectorLayer || !f.get('data')) {
            return false
        }
        this.olPopupText = f.get('data')
        this.tooltipOverlay.setPosition(f.getGeometry().getCoordinates())
    })
})
```

这样当鼠标移上去就会显示tooltip：


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a690004b20e9?w=567&h=390&f=png&s=191868)

接下来看看如何绘制多边形，绘制图形用的还是之前的Draw交互：

```js
import { Draw } from 'ol/interaction'

let source = new VectorSource()
let vector = new VectorLayer({
    source: source
})
map.addLayer(vector)
let draw = new Draw({
    source: source,
    type: 'Circle'
})
map.addInteraction(draw)
```

很简单，实例化一个Draw对象，设置一下type就可以了，上面设置的是Circle，绘制出来的是圆：


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a693f2906815?w=568&h=394&f=png&s=204169)

接下来看看正方形和长方形，在上面的例子之上修改：

```js
import { createRegularPolygon, createBox } from 'ol/interaction/Draw'

// createRegularPolygon方法执行后返回一个创建正方形的geometryFunction
// createBox方法执行后返回一个创建长方形的geometryFunction

let draw = new Draw({
    source: source,
    type: 'Circle',//没错，还是Circle
    geometryFunction: createBox()
})
```

其他类型只要设置对应的type就可以了，比如绘制不规则多边形为POLYGON，具体类型可以查看文档：[https://openlayers.org/en/latest/apidoc/module-ol_geom_GeometryType.html](https://openlayers.org/en/latest/apidoc/module-ol_geom_GeometryType.html)。

实际的使用场景还会存在需要修改存在的多边形的情况，需要用到Modify交互：

```js
import { Modify } from 'ol/interaction'

let modify = new Modify({
    source
})
map.addInteraction(modify)
```

现在就可以拖动多边形的端点来进行修改了。

以上对几何体的操作和显示用的都是自带的默认样式，如果有自定义样式需求的话可以通过style配置进行修改，对要素的基本使用就到这里。



# 获取地图当前区域的范围

为了性能考虑，如果是在地图上显示要素的话最好是只显示当前显示区域内的要素，要显示的数据一般从后端进行请求，那么可以把当前区域的范围发送给后端，后端只返回这个区域内的数据就好了，那么就需要获取当前的范围：

```js
// 获取当前地图区域上下左右四个点的经纬度
let range = map.getView().calculateExtent(map.getSize())
let state = {
    minLon: range[0],
    minLat: range[1],
    maxLon: range[2],
    maxLat: range[3],
    zoomLevel: map.getView().getZoom()// 当前缩放级别，缩放级别可用来判断是否要将要素聚合进行显示
}
```



# 再会

因为本人也是刚开始入门，所以可能存在一些不对的地方或有一些更好的实现方式，欢迎指出。

