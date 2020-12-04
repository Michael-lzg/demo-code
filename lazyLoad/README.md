# 图片懒加载

在类电商类项目，往往存在大量的图片，如 banner 广告图，菜单导航图，美团等商家列表头图等。图片众多以及图片体积过大往往会影响页面加载速度，造成不良的用户体验，所以进行图片懒加载优化势在必行。

### 为什么要进行图片懒加载

我们先来看一下页面启动时加载的图片信息。

<img src="./img/2.png">

如图所示，这个页面启动时加载了几十张图片（甚至更多），而这些图片请求几乎是并发的，在 Chrome 浏览器，最多支持的并发请求次数是有限的，其他的请求会推入到队列中等待或者停滞不前，直到上轮请求完成后新的请求才会发出。所以相当一部分图片资源请求是需要排队等待时间的。

在上面可以看出，有部分图片达到几百 kB，设置 2M(这锅必须运营背，非得上传高清大图不可？)，直接导致了加载时间过长。

<img src="./img/1.png">

针对以上情况，进行图片懒加载有以下优点：

1. 减少资源的加载，页面启动只加载首页的图片，这样能明显减少了服务器的压力和流量，也能够减小浏览器的负担。
2. 防止并发加载的资源过多会阻塞 js 的加载，影响网站的正常使用。
3. 能提升用户的体验，不妨设想下，用户打开页面的时候，如果页面上所有的图片都需要加载，由于图片数目较大，等待时间很长这就严重影响用户体验。

### 图片懒加载的原理

图片懒加载的原理主要是判断当前图片是否到了可视区域这一核心逻辑实现的

1. 拿到所有的图片 img dom 。
2. 遍历每个图片判断当前图片是否到了可视区范围内。
3. 如果到了就设置图片的 src 属性。
4. 绑定 window 的 scroll 事件，对其进行事件监听。

### 传统方法

传统的图片懒加载主要是通过绑定 window 的 scroll 事件，进行事件监听，判断当前图片是否到了可视区域。

我们先来看下页面结构

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Lazyload</title>
    <style>
      img {
        display: block;
        margin-bottom: 50px;
        height: 200px;
        width: 400px;
      }
    </style>
  </head>
  <body>
    <img src="./img/default.png" data-src="./img/1.jpg" />
    <img src="./img/default.png" data-src="./img/2.jpg" />
    <img src="./img/default.png" data-src="./img/3.jpg" />
    <img src="./img/default.png" data-src="./img/4.jpg" />
    <img src="./img/default.png" data-src="./img/5.jpg" />
    <img src="./img/default.png" data-src="./img/6.jpg" />
    <img src="./img/default.png" data-src="./img/7.jpg" />
    <img src="./img/default.png" data-src="./img/8.jpg" />
    <img src="./img/default.png" data-src="./img/9.jpg" />
    <img src="./img/default.png" data-src="./img/10.jpg" />
  </body>
</html>
```

先获取所有图片的 dom，通过 document.body.clientHeight 获取可视区高度，再使用 element.getBoundingClientRect() API 直接得到元素相对浏览的 top 值， 遍历每个图片判断当前图片是否到了可视区范围内。代码如下：

```js
function lazyload() {
  let viewHeight = document.body.clientHeight //获取可视区高度
  let imgs = document.querySelectorAll('img[data-src]')
  imgs.forEach((item, index) => {
    if (item.dataset.src === '') return

    // 用于获得页面中某个元素的左，上，右和下分别相对浏览器视窗的位置
    let rect = item.getBoundingClientRect()
    if (rect.bottom >= 0 && rect.top < viewHeight) {
      item.src = item.dataset.src
      item.removeAttribute('data-src')
    }
  })
}
```

最后给 window 绑定 onscroll 事件

```js
window.addEventListener('scroll', lazyload)
```

主要就完成了一个图片懒加载的操作了。但是这样存在较大的性能问题，因为 scroll 事件会在很短的时间内触发很多次，严重影响页面性能，为了提高网页性能，我们需要一个节流函数来控制函数的多次触发，在一段时间内（如 200ms）只执行一次回调。

下面实现一个节流函数

```js
function throttle(fn, delay) {
  let timer
  let prevTime
  return function (...args) {
    const currTime = Date.now()
    const context = this
    if (!prevTime) prevTime = currTime
    clearTimeout(timer)

    if (currTime - prevTime > delay) {
      prevTime = currTime
      fn.apply(context, args)
      clearTimeout(timer)
      return
    }

    timer = setTimeout(function () {
      prevTime = Date.now()
      timer = null
      fn.apply(context, args)
    }, delay)
  }
}
```

然后修改一下 srcoll 事件

```js
window.addEventListener('scroll', throttle(lazyload, 200))
```

### IntersectionObserver

通过上面例子的实现，我们要实现懒加载都需要去监听 scroll 事件，尽管我们可以通过函数节流的方式来阻止高频率的执行函数，但是我们还是需要去计算 scrollTop，offsetHeight 等属性，有没有简单的不需要计算这些属性的方式呢，答案就是 IntersectionObserver。

IntersectionObserver 是一个新的 API，可以自动"观察"元素是否可见，Chrome 51+ 已经支持。由于可见（visible）的本质是，目标元素与视口产生一个交叉区，所以这个 API 叫做"交叉观察器"。我们来看一下它的用法：

```js
var io = new IntersectionObserver(callback, option)

// 开始观察
io.observe(document.getElementById('example'))

// 停止观察
io.unobserve(element)

// 关闭观察器
io.disconnect()
```

IntersectionObserver 是浏览器原生提供的构造函数，接受两个参数：callback 是可见性变化时的回调函数，option 是配置对象（该参数可选）。  

目标元素的可见性变化时，就会调用观察器的回调函数 callback。callback 一般会触发两次。一次是目标元素刚刚进入视口（开始可见），另一次是完全离开视口（开始不可见）。

```js
var io = new IntersectionObserver((entries) => {
  console.log(entries)
})
```
callback 函数的参数（entries）是一个数组，每个成员都是一个 IntersectionObserverEntry 对象。举例来说，如果同时有两个被观察的对象的可见性发生变化，entries 数组就会有两个成员。

* time：可见性发生变化的时间，是一个高精度时间戳，单位为毫秒
* target：被观察的目标元素，是一个 DOM 节点对象
* isIntersecting: 目标是否可见
* rootBounds：根元素的矩形区域的信息，getBoundingClientRect()方法的返回值，如果没有根元素（即直接相对于视口滚动），则返回 null
* boundingClientRect：目标元素的矩形区域的信息
* intersectionRect：目标元素与视口（或根元素）的交叉区域的信息
* intersectionRatio：目标元素的可见比例，即 intersectionRect 占 boundingClientRect 的比例，完全可见时为 1，完全不可见时小于等于 0