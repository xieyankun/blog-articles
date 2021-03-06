---
title: 小程序多次点击最佳实践
comments: true
date: 2018-11-22 19:18:22
categories: 实战
tags: 小程序
img: https://user-images.githubusercontent.com/22571015/48899721-4e84f000-ee8b-11e8-873c-949cece53adc.png
---

# 多次点击

小程序没有好的优化事件处理机制，导致重复点击会触发多次（当我们快速点击的时候会多次执行，比如点击加载更多，进入下一页面的时候都会出现异常，多次调用，页面也会多次进入，对于体验）

## 问题来源：

- 来参加小程序活动详情加载更多用户的时候遇到的


## 使用 disabled

disabled 的限制比较多，只有对于有button按钮和支持的 disabled 的组件可以使用，使用 disabled 的方式来阻止多次点击在调试工具上可以，但是手机上亲测的情况是还是会有多次点击的情况，主要是因为click事件有250ms的延时，也就是说点击click后会在200ms后执行。所以，使用这个方法在快速点击的情况下还是会出现这样的问题，而且现在的状态为不可点击所以还要解除限制（不是好的可选方案）



## 配合使用 touch 事件

data里面定义3个属性

```javascript
touchStartTime: 0, // 触摸开始时间
touchEndTime: 0, // 触摸结束时间
lastTapTime: 0 // 最后一次单击事件点击发生时间
```

页面触发这3个事件

```html
<button
  type='primary'
  bindtap="newTapHandle"
  bindtouchstart="touchStart"
  bindtouchend="touchEnd"
>
  配合touch事件
</button>
```


methods里面添加3个方法

```javascript
touchStart(e) {
  console.log(e)
  this.touchStartTime = e.timeStamp;
},
touchEnd(e) {
  console.log(e)
  this.touchEndTime = e.timeStamp;
},
newTapHandle(e) {
  console.log(e);
  // 控制点击事件在350ms内触发，加这层判断是为了防止长按时会触发点击事件
  if (this.touchEndTime - this.touchStartTime < 350) {
    // 当前点击的时间
    var currentTime = e.timeStamp;
    var lastTapTime = this.lastTapTime;

    // 更新最后一次点击时间
    this.lastTapTime = currentTime;

    // 如果两次点击时间在250毫秒内，则认为是多次事件
    if (currentTime - lastTapTime > 250) {
      // do something 点击事件具体执行那个业务
      console.log('双击')
      wx.navigateTo({
        url: '/pages/logs/logs',
      })
    }
  }
},
```
这个方法基本呢个解决我们需要的多次点击问题，通过touch时间的开始和结束的时间间隔，判断点击的状态，但是这个方法还是有个弊端，就是第一次快速点击的时候不会执行里面的方法，而且，要借助touch事件来完成，通用性很差，无疑增加了逻辑的复杂程度。

## 函数截流

为什么会有函数截流？
在前端 JS 在处理 DOM 的 `scroll` `resize` `mousemove` 等事件会不间断触发。以 `scroll` 事件为例：

```javascript
const scrollFn = () => {
    console.log(0)
}

window.onscroll = scrollFn()

```
打开控制台我们会看在滚动的同时 scrollFn 函数会执行很多次。在操作DOM和其他交互的时候会非常影响性能，为了避免这个问题，我们一般需要使用一个定时器来对函数进行截流。看下面的截流方法

```javascript
function throttle(method, delay){
  let timer = null;
  return function() {
    clearTimeout(timer);
    timer = setTimeout(() => {
      method.apply(this, arguments);
    }, delay);
  }
}

window.onscroll = throttle(scrollFn(), 250)

```
其实就是利用 `setTimeout` 的异步队列和闭包，把需要执行的函数缓存起来，然后在指定的事件后执行。

> 函数截流: 基本思想是设置一个定时器，在指定时间间隔内运行代码时清除上一次的定时器，并设置另一个定时器，函数请求停止并超过时间间隔才会执行。

但是这种截流方式在事件不停止的情况下，将永远不会被执行，所以需要对这个截流函数做修改。

```javascript
function throttle(method, delay, duration){
  const timer = null;
  let begin = new Date();  

  return function() {                
    clearTimeout(timer);

    const current = new Date(); 

    if(current - begin >= duration){
      method.apply(this, arguments);
      begin = current;
    } else {
      timer = setTimeout(() => {
        method.apply(this, arguments);
      }, delay);
    }

  }
}

window.onscroll = throttle(scrollFn, 100, 500);

```

小程序中在跳转页面的时候多次点击的时候，我们希望它值执行一次,我们看看另一种实现方式不使用 `setTimeout` 我们不希望它延时执行，我们只是希望在第一次点击后的间隙时间内，只触发一次。


```javascript
// utils/index.js
export const throttle = (fn, gapTime) => {
  if (gapTime == null || gapTime == undefined) {
    gapTime = 1000
  }

  let _lastTime = null

  // 返回新的函数
  return function () {
    let _nowTime = + new Date()
    if (_nowTime - _lastTime > gapTime || !_lastTime) {
      fn.apply(this, arguments)   //将this和参数传给原函数
      _lastTime = _nowTime
    }
  }
}
```

小程序内使用：

```html
<button
  type='primary'
  bindtap='throttleHandle'
>
  使用函数截流
</button>
```
```javascript
// click.js
throttleHandle: throttle(function(e) {
  this.goLogs();
}, 250),
```

推荐使用最后一种方法