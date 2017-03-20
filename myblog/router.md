# 前端路由

在之前的 `SUI-mobile Router浅析中` 解释到了SUI中路由模块的一个大致的运行模式。
在这篇文章中，就说说前端路由的一个实现原理

## 前端路由模式

前端路由有两种模式，分别是 `hash` 还有 `History`。兼容性上，`hash`可以兼容到ie8，而 `History` 只能兼容到ie9。

## hash

在前端路由中，经常能见到 `#!` 符号。在浏览器中，`#` 后面的默认是不解析的，所以很多功能可以通过 `#` 的变化来进行操作，例如markdown中的标题定位，就是通过 `#` 来实现。在这层中`#` 标识的是一个锚点，类似于一个定位。
而在前端路由中，这个`#!` 称之为`哈希（hash）` 。在`History API` 出现之前，前端路由都是通过`hash`实现的。例如，Vue的路由模块，就有`hash`模式的一个实现。

### onhashchange

在`hash`模式中，通过window注册一个`hashchange`事件，就可以监听哈希变化

```javascript
window.addEventListener('hashchange', function () {
    /* listen hash change */
    })
```

在回调函数中，可以传递一个事件参数，打印事件参数，可以发现有两个字段
`newURL` 和 `oldURL`。
两个URL都是完整的地址，可以通过对两个url的处理来实现路由的分发，也可以通过window.location.hash来获取当前的哈希值。

需要注意的是，在哈希中，查询字符串是被忽略的，需要自己手动加到`window.location.search`。

## Histroy 
`History` 是 html5的一个特性，主要通过把当前状态存入栈中，来实现页面的切换。SUI mobile以及Vue-router的history模式用的都是这种模式。兼容ie9及以上浏览器。

### history.pushState

浏览器提供了`history.pushState` 以及 `history.replaceState` 两个api，用来添加或者改变状态。这里介绍的是`history.pushState`
具体查看[History.pushState()](https://developer.mozilla.org/zh-CN/docs/Web/API/History/pushState)
```javascript
/*
 *  @param {object} state 用于存储当前的页面状态，可以通过popstate事件释出
 *  @param {string} title 
 *  @param {string} url 当前的url
 */
    history.pushState(state, title, url);
```
可以打开任意网站尝试比如百度
```javascript
    window.history.pushState({ 'page_id': 1, 'user_id': 5 }, null, "https://www.baidu.com/name/orange");
    //url: https://www.baidu.com/name/orange

    window.history.pushState({ 'page_id': 1,
                               'user_id': 5 },
                             'hello',
                              "?name=orange");
    //url: https://www.baidu.com?name=orange

    window.history.pushState({ 'page_id': 1,
                               'user_id': 5 },
                               'hello',
                               "name=orange");
    //url: https://www.baidu.com/name=orange

    window.history.pushState({ 'page_id': 1,
                               'user_id': 5 },
                               'hello',
                               "/name/orange");
    //url: https://www.baidu.com/name/orange

    window.history.pushState({ 'page_id': 1,
                               'user_id': 5 },
                               'hello',
                               "name/orange");
    //url: https://www.baidu.com/name/orange
```
因为url不支持跨域，把url改为 ```baidu.com``` 就会报错


### popstate 事件
页面后退时，会触发`popstate`事件
具体查看[popstate](https://developer.mozilla.org/en-US/docs/Web/Events/popstate)
```javascript
    // 页面后退，触发popstate
    window.addEventListener('popstate', function(e) {
            // e.state就是 pushState所存储的信息(state,title,url)
            // 路由切换后（比如sui）的动画操作就在这里触发
            console.log(e.state)
        })
```

## 手势滑动分析
从现在看，一般前端路由的页面切换渲染都在``popstate`` 或者``hashchange`` 事件中去执行，拥有手势返回的框架，例如Freamework7，是没有前端路由的，页面切换并不会改变路径（例如[HiApp](https://hi.dearb.me/build/)），只有build一个路径。具体的实现方式，还得继续研究。

