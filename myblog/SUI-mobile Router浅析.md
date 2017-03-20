# SUI-mobile Router浅析

[TOC]

## 封装方式

SUI-mobile基于Framework7实现的，这一点从Router模块与其他模块的封装方式可以看得出来。

 **Picker模块**
 
```javascript
var Picker = function(params) {
    var p = this;
    var defaults={
    //基本设置
    }
    p.setValue = function(arrValues, transition) {
        // setValue 代码块
    }
}
```
以Picker模块为例，所有的封装都基于this来实现，其他模块类似

 **Router模块**
 
```javascript
 var EVENTS = {
    // 事件名称
 }
 // 一些获取路径以及检测函数
 var Util = {
    getUrlFragment: function(url) {
        // getUrlFragment 代码块
    },
    // 以下类似
 }
// Router 的构造函数
 var Router = function(){}

// 通过prototype来进行Router封装
Router.prototype._init = function () {}

// 以下雷同_

```
 Router 的封装是通过prototype，最后通过
```javascript
  var  router = $.router =new Router();
```
 绑定在了zepto上

## 返回和前进的实现（重点）
在这，先讲的是在SUI中，返回和前进的实现。
在Router构造函数中，有一段代码
```javascript
window.addEventListener('popstate', this._onPopState.bind(this));
```
在这句代码中，注册了一个`popstate`事件，SUI的回退和前进都是基于这个实现的

在讲`popstate`，先来说明下`返回`和`前进`按钮，它其实是简单封装了`history.back`和`history.go`，并没有什么特殊。

但是在触发`history.back`和`history.go`的时候，就会触发`popstate`事件
```javascript
Router.prototype._onPopState = function(event) {
    var state = event.state;
    //代码忽略
    if (state.id < lastState.id) {
        this._back(state, lastState);
     } else {
        this._forward(state, lastState);
    }
}
```

在`popstate`触发的`_onPopState`方法里，有一个event的回调，如果你进入一个页面在按返回触发这个方法然后log出来，是这样的
```javascript
{
    id:3,
    pageId:"page-home",
    url: [obj]// 存储各种路径
}
```

那么这些数据是怎么来的呢，就是不管是内嵌页面切换还是外链页面切换都会进入一个`_pushNewState`
```javascript
Router.prototype._pushNewState = function(url, sectionId) {
        var state = {
            id: this._getNextStateId(),
            pageId: sectionId,
            url: Util.toUrlObject(url)
        };

        theHistory.pushState(state, '', url);
        this._saveAsCurrentState(state);
        this._incMaxStateId();
    };
```
通过`theHistory.pushState(state, '', url);`即`window.history.pushState`把信息存入state，这个state的信息只有在触发`history.back`和`history.go`才会从回调中取得。

## 内嵌页面和外链页面的实现与区别

在SUI中，规定要把页面放在`page-group`类下。
在点击外链的时候，跳入`_switchToDocument`，会触发Ajax，获取链接页面，并取得其中的`page-group`类的内容，然后跳入`_doSwitchDocument`渲染在新的页面，刷新href,然后`pushState`。
因此，其实整个的“跳转”并不是传统意义上的跳转页面，只是同个页面Ajax请求后刷新的假象。

点击链接后，跳入`Router.prototype.load`，并传入url

**Router.prototype.load**

```javascript
Router.prototype.load = function(url, ignoreCache, $link) {
        // 默认使用缓存
        if (ignoreCache === undefined) {
            ignoreCache = false;
        }
        // 检查base url是否相同，相同则锚点切换, 否则 Ajax 切换
        if (this._isTheSameDocument(location.href, url)) {
            this._switchToSection(Util.getUrlFragment(url), $link);
        } else {
            // 在外链切换，多了这么一项动作，是存储当前的页面和地址
            this._saveDocumentIntoCache($(document), location.href);
            this._switchToDocument(url, ignoreCache);
        }
    };
```


**_saveDocumentIntoCache**

```javascript
     Router.prototype._saveDocumentIntoCache = function(doc, url) {
        // 代码忽略
        this.cache[urlAsKey] = {
            $doc: $doc,
            $content: $doc.find('.' + routerConfig.sectionGroupClass)
        };
    }
```
在这里，点击外链的时候，把整个旧页面存入了`$.router.cache`中，在返回的时候就会调用里面的dom渲染旧页面。这一点加上SUI对于页面回退的实现方法，是整个滑动返回实现的难点所在。

加载完页面紧接着是获取当前页面$currentDoc为然后渲染新页面为$newDoc，接着进入`_animateDocument`。
如果是返回的话，$newDoc则在`$.router.cache`中获取，然后渲染为$newDoc
```javascript
// 四个参数为 当前页面，新页面，可见块，切换方向
this._animateDocument($currentDoc, $newDoc, $visibleSection, direction);
```

**内嵌页面切换**

内嵌页面的切换就没有那么多的步骤，只是获取id拿到那段section渲染为`newPage`，当前页面为`currentPage`然后传入`_animateSection`接着再传入`_animateElement`做class切换就结束了。

**back&&forward**

在上面所述的back和forward，其实动画的实现方式也是一样，只是多了一层判断是否为外链之间的切换，其他的也是通过渲染成`newPage`和`currentPage`作为参数传入`_animateSection`或者`_animateDocument`处理就能形成切换了。

## 滑动返回实现评估

之前见过Framework7的滑动返回，它都是基于内嵌页面的层叠关系来实现的，并且没有路由跳转，单纯在本页面的切换。
在SUI中，想要实现滑动返回需要解决的最大的是`路由`和`state`。基于SUI做的话，除了解决CSS方面的问题，还有保留`路由`和`state`去实现的话，暂时还没有想到很好的办法。
