# Redux 源码浅析

最近一段时间公司比较闲下来，故抽空学了``react`` + ``redux``。在``react``方面，鉴于有``Vue``的经验，很多东西概念上还是很统一的，例如``Virtual Dom`` ``props`` ``JSX``等。区别在于``react``没有像``Vue``那样那么多的api，更多的是用纯粹的JavaScript去解决，在这一点我觉得我更喜欢``react``多些。

当然，``react``更加的函数式，对于``state``的变更一定要通过``setState``这一点可以看出，``react``对于``state``不允许有任何的``副作用``。而相对与``react``的简单，更困扰我的是``redux``，它更加函数式，从代码中，随处可见的``高阶函数``、``科里化``以及让这个之前没有接触过函数式编程的我感到神奇的``compose``等。因为如此，它的``异步操作``更是与``Vuex``有着很大的差异，各种各样基于``redux``的异步处理工具如雨后春笋般冒出来，例如``redux-saga``、``redux-observable``以及官方的一个只有12行代码的``redux-thunk``。

因为没有时间可以给我实战中去理解它，所以我选择通过阅读其源代码，来加深对``redux``的理解

## 主要的api

``redux``暴露出来的api并不多

- createStore：接受``reducers``生成状态树
- combineReducers：把多个``reducer``合成一个``reducers``
- bindActionCreators: 封装多一层函数嵌套，规避使用``dispatch``
- applyMiddleware：初始化中间件
- compose：非常函数式的一个方法，从右到左把接收到的函数合成后的最终函数

在``src/indexjs``里面能看到，``redux``就只有暴露这五个api，其中经常用到的也就 3-4 个。

## createStore

先看一下``createStore``的参数
```javascript
// src/createStore.js
/**
 * @param {Function} reducer
 * @param {any} [preloadedState] 初始化的state
 * @param {Function} [enhancer] 这里是一个中间件
 * @returns {Store} 返回的是状态树
**/
```

参数很简单，只有三个，一个是``reducer``，第二个是初始化state，在我们有定义``reducer``的初始化``state``的时候，这个参数是默认不传的，在这里，先把其忽略。第三个是``applyMiddleware(...middlewares)``，是中间件

看下整体的代码
其中``getState``很简单，就不单独拎出来说，``obervable``用于观察者模式，平时几乎用不到，也不讲
```JavaScript
// src/createStore.js
export default function createStore(reducer, preloadedState, enhancer) {
    // 前面的代码是在做参数的初始化
    if (typeof enhancer !== 'undefined') {
        // 这里是处理中间件的地方，先不看这里，后面看中间件的时候反过来看
        return enhancer(createStore)(reducer, preloadedState)
    }
    // ..    
    let currentReducer = reducer // reducer
    let currentState = preloadedState // state
    let currentListeners = [] // 初始化监听器
    let nextListeners = currentListeners
    let isDispatching = false


    function ensureCanMutateNextListeners() {
        // ..
    }
    function getState() {
        // 获取state的方法
        // getState极其简单，就不单独拎出来的，就返回了当前的state
        if (isDispatching) {
            // 抛出错误
        }
        return currentState
    }
    function subscribe(listener) {
        // 监听器，dispatch action的时候就会执行
        // ..
    }
    function dispatch(action) {
        // 分发action的唯一方法
        // ..
    }
    function replaceReducer(nextReducer) {
        // 替换 store 当前用来计算 state 的 reducer
        // ..
    }
    function observable() {
        // 这个方法在文档里也没有体现，貌似是给redux-obervable专门提供的吗还是？不懂
        // ..
    }
    // 这里dispatch一个init action进行初始化
    // 在 src/utils/actionTypes.js 里
    dispatch({ type: ActionTypes.INIT })
    return {
        dispatch,
        subscribe,
        getState,
        replaceReducer,
        [$$observable]: observable
    }
}
```
接下来一个个看它的方法

### subscribe

为什么要先讲``subscribe``呢，这涉及到``react-redux``的连接

其实我并不了解``react-redux``，只知道它调用了``subscribe``开启了个监听器，并传入了一个``listener``

这段代码在``react-redux``里的``src/utils/Subscription.js``里

```JavaScript
trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.onStateChange)
        : this.store.subscribe(this.onStateChange)
      this.listeners = createListenerCollection()
    }
}
```
在这里传入了一个``onStateChange``方法作为``listener``

然后再看回``subscribe``
```javascript
// src/createStore.js
function subscribe(listener) {
    
    // ...
    let isSubscribed = true

    // 判断当前的listener是否与下一个listener一样
    ensureCanMutateNextListeners()
    // push到nextListeners
    nextListeners.push(listener)
    console.log(listener)
    // 返回一个方法，作为取消订阅
    return function unsubscribe() {
      // ...

        isSubscribed = false
        ensureCanMutateNextListeners()
        const index = nextListeners.indexOf(listener)
        nextListeners.splice(index, 1)
    }
  }
```

### dispatch

``dispatch``做的事情很简单，只是把``reducers``传入``state``和``action``去执行然后返回来的新的state赋值到``currentState``，接着再执行通过``subscribe``的监听函数就完成了。

```JavaScript
// src/createStore.js
function dispatch(action) {
if (!isPlainObject(action)) {
    // 如果不是春函数，抛错
}

if (typeof action.type === 'undefined') {
    // 没有 type 抛错
}

if (isDispatching) {
    // 正在分发其他action，抛错
}

try {
    // 进行分发
    isDispatching = true
    // currentReducer 就是我们传进来的reducers
    currentState = currentReducer(currentState, action)
} finally {
    isDispatching = false
}
// 执行监听函数
const listeners = (currentListeners = nextListeners)
for (let i = 0; i < listeners.length; i++) {
    const listener = listeners[i]
    listener()
}
// 返回action
return action
}
```
对于如何执行的``reducers``，我们到后面``combineReducers``再看

### replaceReducer

在文档里，``replaceReducer``是这样描述的

>   替换 store 当前用来计算 state 的 reducer。这是一个高级 API。只有在你需要实现代码分隔，而且需要立即加载一些 reducer 的时候才可能会用到它。在实现 Redux 热加载机制的时候也可能会用到。

在这里我也没用过，不知道具体用途，可是代码简单，就贴出来看看
```JavaScript
// src/createStore.js
function replaceReducer(nextReducer) {
    // 更替reducers再dispatch REPLACE type
    currentReducer = nextReducer
    dispatch({ type: ActionTypes.REPLACE })
  }
```
## combineReducers
对于``reducers``，如果只有一个``reducer``那好说，``action``进去然后``switch case``执行完就完事了。可是大部分情况我们需要``combineReducers``组合我们的多个``reducer``，接下来看下``combineReducers``的代码实现
```JavaScript
// src/combineReducers.js
export default function combineReducers(reducers) {
  // 获取reducers的key
  const reducerKeys = Object.keys(reducers)
  // 最终生成的reducers
  const finalReducers = {}
  // 遍历生成reducers，剔除不是function的
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    // ...
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)
  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  let shapeAssertionError
  try {
    // 判断是否有初始化state
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }
  // 最后返回了一个combinenation
  // 在 dispatch 的时候，就执行这个函数
  // currentState = currentReducer(currentState, action)
  return function combination(state = {}, action) {
      // ...
  }
}

```
``combineReducers``做的事情也很简单，通过高阶函数，利用闭包剔除不合格的``reducer``返回一个``combination``，接下来，来看看``combination``做了些什么

## combination

先回顾下``dispatch``里，是怎么调用``combination``的

```JavaScript
// src/createStore.js
currentState = currentReducer(currentState, action)
```

传入了``state``还有``action``

```JavaScript
// src/combineReducers.js
function combination(state = {}, action) {
    // ...

    let hasChanged = false
    // 全新的数据
    const nextState = {}
    // 遍历reducers keys
    // 每一个reducer都会去执行一遍
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      // 拿到变更前的state
      const previousStateForKey = state[key]
      // 变更后的state
      const nextStateForKey = reducer(previousStateForKey, action)
      // ...
      // 插入到nextState
      nextState[key] = nextStateForKey
      // 判断是否有变化
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
```
``combination``也很简单，就是遍历``reduces``并执行，再返回state。

到这里，整个dispatch的流程就可以清晰列出来了

``dispatch => combination => iterator reducers => return new state``


## 中间件的实现

在``redux``里，可以通过``applyMiddleware``添加中间件，``redux``的代码非常简洁，需要复杂的功能，就需要通过中间件去加强。

###　applyMiddleware

中间件的作用是，在``dispatch``做分发的时候，会按照在``applyMiddleware``时传入的中间件顺序，依次执行，到最后再做``dispatch``

先看看中间件的写法

```JavaScript
// 回调给的getState， dispatch方法
// 一般dispatch不会使用
function someMiddleware ({ getState, dispatch}) {
  // next是下一个中间件
  // action就是dispatch的action
  return next => action => {
    // before dispatch
    // 执行下一个中间件
    let returnValue = next(action)
    // after dispatch
    // 一般会是 action 本身，除非后面的 middleware 修改了它。
    return returnValue
  }
}
```

在需要中间件去生成``store``的方式有两种

```JavaScript
import { createStore, combineReducers, applyMiddleware } from 'redux'
// 这是第一种
const store = createStore(
  reducers,
  applyMiddleware(
    thunkMiddleware,
    logger,
    loggerMiddleware
  )
)

// 这是第二种
const store = applyMiddleware(
    thunkMiddleware,
    logger,
    loggerMiddleware
  )(createStore)(reducers)

```
两种方式并没有区别，还记不记得之前在``createStore``里，说先不看的那句代码

```JavaScript
return enhancer(createStore)(reducer, preloadedState)
```

这里的``enhancer``就是``applyMiddleware(...middlewares)``，看下``applyMiddleware``的代码
```JavaScript
// src/applyMiddleware.js
function applyMiddleware(...middlewares) {
  // 返回一个参数为createStore的匿名函数
  // args就是需要传入的reduces还有init state
  return createStore => (...args) => {
    // 重新走一遍createStore，生成store
    const store = createStore(...args)
    let dispatch = function () {
      // 这里提供dispatch，可是只用来在中间件中dispatch报错
    }
    let chain = [] // 定义中间件的chain
    // 在中间件需要的两个方法
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => {
        console.log('DISPATCH')
        return dispatch(...args)
      }
    }
    // 把需要的getState以及dispatch方法传入中间件
    chain = middlewares.map(middleware => {return middleware(middlewareAPI)})
    // 这里通过compose生成一个dispatch函数
    // 这里还需要传入最初的dispatch，就是为了在中间件执行完毕只有，最后能成功dispatch action
    dispatch = compose(...chain)(store.dispatch)
    // 最后返回store的方法以及新的dispatch
    // 原来的dispatch被覆盖
    return {
      ...store,
      dispatch
    }
  }
}

```

### compose

compose是干什么的呢？先看看一个在``Ramda.js``下的``compose``

```JavaScript
import R from 'ramda'
let compse = R.compose(
    R.multiply(2),
    R.add(7),
    R.divide(3)
)
compose(9)
// 输出20
```
简单的说，``compose``是一个柯里化函数，将多个函数合并成一个函数，从右到左执行

在例子中，``compose``接受三个方法，从右到左就是 ``除以3`` ``加7`` 以及``乘以2``。添加一个参数``9``，依次执行，最后输出的是20

我们先跳到``applyMiddleware``看他的执行

```JavaScript
// src/applyMiddleware.js
chain = middlewares.map(middleware => {return middleware(middlewareAPI)})
dispatch = compose(...chain)(store.dispatch)
```
看看``redux``中的``compose``
```JavaScript
// src/compose.js
export default function compose(...funcs) {
 // 判断funcs的数量
  if (funcs.length === 0) {
    return arg => arg
  }
  
  if (funcs.length === 1) {
    return funcs[0]
  }
  // 这里通过reduce，非常优雅地实现了compose
  return funcs.reduce((a, b) => {
    // 这里是compose(...chain)
    return (...args) => {
      // 这里传入store.dispatch，最终形成一个执行链
      return a(b(...args))
    }
  })
}
```

### reduce

``reduce``方法是一个非常函数式的方法，在``MDN``上我们可以看到``reduce``的详情
> reduce() 方法对累加器和数组中的每个元素（从左到右）应用一个函数，将其减少为单个值。

看看``reduce``的参数，第一个是执行回调函数，直接贴上``MDN``的参数说明
>   callback
        执行数组中每个值的函数，包含四个参数：
        accumulator
        累加器累加回调的返回值; 它是上一次调用回调时返回的累积值，或initialValue（如下所示）。
        currentValue
        数组中正在处理的元素。
        currentIndex
        数组中正在处理的当前元素的索引。 如果提供了initialValue，则索引号为0，否则为索引为1。
        array
        调用reduce的数组

```javascript
arr.reduce(callback[, initialValue])

const total = [0, 1, 2, 3].reduce(function(sum, value) {
  return sum + value;
}, 0);
// log 6
```
到此，整个中间件就完成了，整理下中间件的流程，举例上面的例子

``dispatch(action) => thunkMiddleware => logger => loggerMiddleware => store.dispatch``

## 总结

从整体来说，``redux``是一个非常巧妙的库，它代码不多，随处散发着函数式编程的魅力。也是因为那个``compose``驱使着我去探索函数式编程。

作为一个浅入``react``开发的程序员，``redux``只是整个技术栈的一部分，希望接下来可以去探索一些异步流程库，例如``redux-saga`` ``redux-observable``等。
