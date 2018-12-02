# JavaScript
# 数据结构与算法

# Redux-Saga
本篇文章是建立在读者已经了解redux以及redux-saga的基本使用方法的前提下进行书写的。如果对于redux和redux中间件的机制以及redux-saga的基本使用方法还不太清除的同学，建议先去了解一下这仨玩意儿再回来看~本文是基于我自己对阅读redux-saga源码的一些粗浅理解，如果哪里有没说明白或者说错的话，请一定告知~谢谢~

本篇简单解析一下redux-saga这个中间件中的take和put的原理~
我将会使用下面这个最简单的计数器作为例子
rootSaga.js:
<pre>
  <code>
    import { take, put } from 'redux-saga/effects'
    import * as types from './store/action-types'

    export default function* counterWatcher() {
      let action = yield take(types.ASYNC_ADD)
      yield put({type: types.ADD})
    }

    export default function* () {
      yield counterWatcher()
    }

    reducer.js:
    import * as types from './action-types'

    let initState = { number: 0 }
    export default function(state = initState, action) {
      switch(action.type) {
        case types.ADD:
          return { number: state.number + (action.payload || 1)}
        default:
          return state
      }
    }
  </code>
</pre>

一般有副作用的action不写在reducer中，比如处理异步请求，或者读取缓存之类的这样的action都是带有副作用的action，无法交给reducer处理，可以交给saga中间件去干。让saga监听这样的有effect的action

store/index.js: (redux的入口)
<pre>
  <code>
    import { createStore, applyMiddleware } from 'redux'
    import reducer from './reducer'
    import createSagaMiddleware from 'redux-saga'
    import rootSaga from '../saga.source'

    // 此中间件可以监听动作 执行对应逻辑
    let sagaMiddleware = createSagaMiddleware()

    // 第二个参数是初始值 初始的state
    let store = createStore(reducer, applyMiddleware(sagaMiddleware))

    // saga 是一个generarot函数 run是为了启动saga 启动generator就是执行 .next()
    sagaMiddleware.run(rootSaga) // 调用一次next()

    export default store
  </code>
</pre>

首先执行createSagaMiddleware其实就是执行sagaMiddlewareFactory 这个是整个中间件的入口
sagaMiddlewareFactory是一个redux中间件的写法：
<pre>
  <code>
    return function sagaMiddleware({getState, dispatch}) {
      // ......
      return function (next) { // 该next是下一个中间件或者是真正的最原始的dispatch
        // ......
        return function (action) { // action 是dispatch时候的{type: //xxx}
          // ......
        }
      }
    }
  </code>
</pre>
然后sagaMiddlewareFactory会给sagaMiddleware挂上一个run方法，也就是之后在redux入口文件要执行的sagaMiddleware.run(rootSaga)。该run方法接收一个generator函数，这个generator作为所有saga中要用到的生成器函数的根生成器，也就是rootSaga。

<pre>
  <code>
    export default function sagaMiddlewareFactory({ context = {}, ...options } = {}) {
      // 省略一些代码...
      function sagaMiddleware({ getState, dispatch }) {

        boundRunSaga = runSaga.bind(null, {// 省略一些代码...})

        return next => action => {
          // 省略一些代码...
        }
      }
      sagaMiddleware.run = (...args) => {
        return boundRunSaga(...args)
      }

      return sagaMiddleware
    }
  </code>
</pre>


整个saga中间件的运行，就是从这个run函数开始运行的
run是个简单的函数，其中只执行了一个叫 boundRunSaga的函数，而这个boundRunSaga函数，实际又是一个叫runSaga的函数，这个runSaga才是最后真正执行的所谓的run方法。

我们自己定义的rootSaga作为参数被传到这个runSaga中，而runSaga干的第一件事儿就是执行了一下这个generator
<pre>
  <code>
    export function runSaga(options, saga, ...args) {
      const iterator = saga(...args) // 执行了一些 rootSaga

      const {
        channel = stdChannel(), // 这个比较重要, 后面会用来保存next函数
        dispatch
        // 省略一些代码...
      } = options

      const env = {
        // 这个比较重要, 后面会用来把next函数(也就是执行完上一个yield之后
        // 下面的那个yield要执行时调用的next)作为栈保存进一个数组
        stdChannel: channel,
        dispatch
        // 省略一些代码...
      }
      // 省略一些代码...
      return immediately(() => {
        const task = proc(env, iterator, context, effectId, getMetaInfo(saga), /* isRoot */ true, noop)
        if (sagaMonitor) {
          sagaMonitor.effectResolved(effectId, task)
        }
        return task
      })
    }
  </code>
</pre>


之后一个比较重要的步骤就是会执行一个immediately函数，这个函数主要就是用来立即执行作为参数的函数。可以看到执行了一个叫做proc的函数。对于这个proc函数，我是这么理解的:
proc函数就相当于开启一条线程(伪线程，不是真的开一条线程，要是js也能开线程那就无敌了)。
而saga中间件，碰上一个generator函数的时候，都有可能会调用一次这个proc函数，相当于开
启了一条新的伪线程，当然不一定是所有的generator都会开一条新的伪线程，只要在需要不阻塞
后面代码执行的时候才会这么做，比如takeEvery函数
执行这个proc函数时，把当前执行完的这个generator和env对象{stdChannel, // 省略几个属性...}作为参数传进去

<pre>
  <code>
    export default function proc(env, iterator, parentContext, parentEffectId, meta, isRoot, cont) {
      // 得到的 finalRunEffect 基本上就是下面那个runEffect
      const finalRunEffect = env.finalizeRunEffect(runEffect)

      // 省略一堆代码...

      next()

      return task

      function next(arg, isErr) {
        // 暂时省略一堆代码...
      }

      // effect 就是yield 后面的value
      function runEffect(effect, effectId, currCb) {
        if (is.iterator(effect)) {
          resolveIterator(effect, effectId, meta, currCb)
        } else if (effect && effect[IO]) {
          // 如果不是执行rootSaga或者用了all就走这里
          const { type, payload } = effect
          if (type === effectTypes.TAKE) runTakeEffect(payload, currCb)
          else if (type === effectTypes.PUT) runPutEffect(payload, currCb)
          else if (type === effectTypes.FORK) runForkEffect(payload, effectId, currCb)
          // 省略一堆代码...
      // 这里其实应该还有好多effects提供的名为 runXXXeffect 的函数处理，暂时先不讨论
          else currCb(effect)
        } else {
          currCb(effect)
        }
      }

      // cb 就是next
      function digestEffect(effect, parentEffectId, label = '', cb) {
        // 暂时省略一些代码...
        function currCb(res, isErr) {
          // 暂时省略一些代码...
        }
        finalRunEffect(effect, effectId, currCb)
      }

      function resolveIterator(iterator, effectId, meta, cb) {
        // 这里跟 runSaga中的方法一样
        proc(env, iterator, taskContext, effectId, meta, /* isRoot */ false, cb)
      }

      // pattern 就是action
      function runTakeEffect({ channel = env.stdChannel, pattern, maybe }, cb) {
        const takeCb = input => {
          // 省略一些代码...
          cb(input)
        }
        try {
          // 这里的channel是env.stdChannel 而这个stdChannel是runSaga.js中引入的 ./channel.js 文件中的函数
          // 然后这里的channel.take实际上就是往一个叫做 takes的数组中推入了 下一个next函数
          channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null)
        } catch (err) {
          cb(err, true)
          return
        }
        cb.cancel = takeCb.cancel
      }

    function runEffect() {
        // 省略一堆代码...
      }

      function runPutEffect({ channel, action, resolve }, cb) {
        // 省略一堆代码...
      }

      function runForkEffect({ context, fn, args, detached }, effectId, cb) {
        // 省略一堆代码...
      }

    }
  </code>
</pre>

然后proc函数上来先做了一些处理，比如给每条伪线程和generator分配id号之类的。
然后主要执行了一个名为 next()的函数

<pre>
  <code>
    function next(arg, isErr) {
      try {
        // 省略一些代码...
        let result = iterator.next(arg)
        if (!result.done) {
          digestEffect(result.value, parentEffectId, '', next)
        }
      } catch (error) {
        // 省略一些代码...
      }
    }
  </code>
</pre>


可以看到next函数中，先是执行了一些generator函数自己本身的next方法，也就是走到generator函数中的第一个 yield那里。可以回到上面看看例子中的rootSaga函数中的第一个yield后面是什么，是滴，是yield counterWatcher()。也就是说，这个result的value就是这个counterWatcher执行返回的结果，也就是一个iterator迭代器~~之后把返回的迭代器value和分配的伪线程id以及这个next函数本身(重要!)作为参数传到了一个叫digestEffect函数中(参数effect此时就是value就是counterWatcher返回的iterator迭代器)

<pre>
  <code>
    // cb 就是next
    function digestEffect(effect, parentEffectId, label = '', cb) {
      // 省略一些代码...
      function currCb(res, isErr) {
        // 省略一些代码...
        cb(res, isErr)
      }
      // 省略一些代码...
      // effect 就是counterWatcher返回的iterator
      finalRunEffect(effect, effectId, currCb)
    }
  </code>
</pre>


这个digestEffect中，省略的部分是给回调函数cb添加了一些额外的功能以及执行了一些monitor监听器的功能函数，暂时不太重要，暂且不谈~，比较重要的就是又把cb作为参数，执行了一个叫finalRunEffect的函数(名字叫final，终于到最后一个了，先前的一些流程，很多都是给saga添加额外的功能和纠错的)。这个finalRunEffect暂时可以理解成就是这个proc函数中定义的一个叫 runEffect的函数。
<pre>
  <code>
    // effect 就是yield 后面的iterator
    function runEffect(effect, effectId, currCb) {
    if (is.iterator(effect)) {
        // 如果rootSaga没用all就走这里
        resolveIterator(effect, effectId, meta, currCb)
      } else if (effect && effect[IO]) {
        // 如果不是执行rootSaga或者用了all就走这里
        const { type, payload } = effect
        if (type === effectTypes.TAKE) runTakeEffect(payload, currCb)
        else if (type === effectTypes.PUT) runPutEffect(payload, currCb)
        else if (type === effectTypes.FORK) runForkEffect(payload, effectId, currCb)
        // 这里还有好多if else 比如什么 runCancelEffect之类的effects提供的方法
        else currCb(effect)
      } else {
        currCb(effect)
      }
    }
  </code>
</pre>


由于最上面的例子，所以导致现在的effect是iterator，所以会进入is.iterator这个分支，该分支中执行一个叫做resolveIterator的函数并把该iterator和currCb作为参数传进去，这里的currCb就是上面的next函数。resolveIterator函数其实就一句话

<pre>
  <code>
  function resolveIterator(iterator, effectId, meta, cb) {
    // 这个里头真的一点代码都没省略~
    // 这里跟 runSaga.js中的方法一样 同时也就是最外层的这个proc函数
    // 不同的是这次作为参数的iterator已经不是rootSaga了，而是我们写在rootSaga函数
    // 中的counterWatcher函数返回的iterator迭代器
    proc(env, iterator, taskContext, effectId, meta, /* isRoot */ false, cb)
  }
  </code>
</pre>

之后会走几乎一样的流程:
proc(iterator) → next() → result = iterator.next(arg) → digestEffect(result.value, id, '', next)
→ finalRunEffect(effect/*此时这个effect已经变成了counterWatcher中的第一个yield后面的返回
值*/, id, next) → runEffect(effect, id, next) 到了这里的runEffect再往下就不太一样了。重新来看
一下runEffect函数
<pre>
  <code>
    function runEffect(effect, effectId, currCb) {
      if (is.iterator(effect)) {
          resolveIterator(effect, effectId, meta, currCb)
        } else if (effect && effect[IO]) {
          const { type, payload } = effect
          if (type === effectTypes.TAKE) runTakeEffect(payload, currCb)
          else if (type === effectTypes.PUT) runPutEffect(payload, currCb)
          else if (type === effectTypes.FORK) runForkEffect(payload, effectId, currCb)
          // 这里还有好多if else 比如什么 runCancelEffect之类的effects提供的方法
          else currCb(effect)
      }
  </code>
</pre>

这个时候由于例子中的counterWatcher中第一个yield后面是 take(types.ASYNC_ADD)，像这种redux-saga/effects自己提供的函数，大都会走到 effect && effect[IO] 这个分支中，为啥呢~我们来看一些take函数的返回值，take函数在 interna/io.js 文件中

<pre>
  <code>
    export function take(patternOrChannel = '*', multicastPattern) {
      if (is.pattern(patternOrChannel)) {
        return makeEffect(effectTypes.TAKE, { pattern: patternOrChannel })
      }
      if (is.multicast(patternOrChannel) && is.notUndef(multicastPattern) && is.pattern(multicastPattern)) {
        return makeEffect(effectTypes.TAKE, { channel: patternOrChannel, pattern: multicastPattern })
      }
      if (is.channel(patternOrChannel)) {
        return makeEffect(effectTypes.TAKE, { channel: patternOrChannel })
      }
    }
  </code>
</pre>


可以看到不管走哪个分支，都会执行一个叫makeEffect的函数。(这里的patternOrChannel可以简单理解成自己传过来的需要监听的action动作)。makeEffect也就一句话
<pre>
  <code>
    const makeEffect = (type, payload) => ({ [IO]: true, type, payload })
  </code>
</pre>

所以最后返回的其实就是类似 { [IO]: true, type: 'take', payload: { pattern: action }} 这样的对象。这种对象也是saga中间件中最常见的类型，上面那个runEffect函数中的effect实际上就是这种对象。然后通过 type === effectTypes.XXX 进入不同的分支。本例子中执行的是take，所以进入第一个runTakeEffect(payload, next) 分支中

<pre>
  <code>
    // pattern 就是action
    function runTakeEffect({ channel = env.stdChannel, pattern, maybe }, cb) {
      const takeCb = input => {
        // 省略一些容错代码...
        cb(input)
      }
      try {
        // 这里的channel是env.stdChannel 而这个stdChannel是runSaga.js中引入的 ./channel.js 文件中的函数
        // 然后这里的channel.take实际上就是往一个叫做 takes的数组中推入了 下一个next函数
        channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null)
      } catch (err) {
        cb(err, true)
        return
      }
      cb.cancel = takeCb.cancel
    }
  </code>
</pre>

这个方法很明显，如果channel.take都正常执行，那就皆大欢喜，如果不正常才直接执行回调。那channel.take是个啥玩意儿呢~看上面的形参解构那里，channel = env.stdChannel。env是最外层的proc直接传进来的参数，而这个stdChannel就是上面在说 runSaga函数时，注释里提到很重要的属性。现在可以回到runSaga方法中，runSaga方法中很明显的写到，stdChannel属性来自于channel
<pre>
  <code>
    const env = {
      stdChannel: channel,
      dispatch: wrapSagaDispatch(dispatch),
      getState,
      sagaMonitor,
      logError,
      onError,
      finalizeRunEffect,
    }
  </code>
</pre>

而channel则来自于
<pre>
  <code>
  const {
    channel = stdChannel(),
    // 省略一些属性...
  } = options
  </code>
</pre>

这个stdChannel方法，则定义在 ./channel.js 文件中

<pre>
  <code>
    export function channel(buffer = buffers.expanding()) {
      // 省略一些代码...

      let takers = []
      function put(input) {
        // 暂时省略一些代码...
      }

      function take(cb) {
        // 暂时省略一些代码...
      }

      // 省略俩函数...

      return {
        take,
        put,
        // 省略俩方法...
      }
    }
  </code>
</pre>

这个channel返回一个对象，对象中有两个目前比较重要的函数，一个take，一个put。
其中take就是上面runTakeEffect中执行的那个channel.take方法。
<pre>
  <code>
    function take(cb) {
      // 省略一些容错处理...
      takers.push(cb)
      cb.cancel = () => {
        remove(takers, cb)
      }
    }
  </code>
</pre>

可以发现，其实这个take函数的作用，就是一个发布订阅模式的过程。将传进来的cb(也就是next，这个next指proc中定义的那个next)，推入一个名为tasks的数组当中，并且给这个回调一个取消订阅的方法。
到这里，基本上这个saga做的第一部分的事情就干完了。也就是说，saga弄了这么一大堆，实际上只是为了让counterWatcher函数能够一直卡在第一个执行完take函数的第一个yield后面，并且把拥有执行下一个yield能力的next函数放入一个数组中。

到这里，就相当于saga中间件已经开始监听我们自己放在take方法中的type: ASYNC_ADD命令了~~

那么如果触发这个命令呢~

我们都应该知道，redux的中间运行机制就是将redux原始提供的dispatch做一层高阶函数的处理然后返回一个新的dispatch，同理saga中间也是返回一个新的dispatch函数。

<pre>
  <code>
    export default function sagaMiddlewareFactory({ context = {}, ...options } = {}) {
      // 省略一些代码...
      function sagaMiddleware({ getState, dispatch }) {
        boundRunSaga = runSaga.bind(null, {// 省略一些代码...})
        return next => action => {
          // 省略一些代码...
        }
      }
      sagaMiddleware.run = (...args) => {
        return boundRunSaga(...args)
      }
      return sagaMiddleware
    }
  </code>
</pre>

sagaMiddleware中返回的
<pre>
  <code>
    return next => action => {
      // 这个就是用了中间件之后，真正触发的dispatch
    }
  </code>
</pre>

那个next可以是下一个中间件(如果有的话)，如果没有的话那这个next就是真正的redux提供的原生dispatch，里头可以放{type: action}的那种~~

所以真正触发了我们用saga的take监听的action回调的方法就是这个中间件新返回的dispatch
<pre>
  <code>
    return next => action => {
      // 省略一个侦听的函数执行 不影响啥

      const result = next(action) // hit reducers
      channel.put(action)
      return result
    }
  </code>
</pre>

该dispatch内部调用了next，在这个例子中，这个next就是真正的redux原生dispatch。但是，一般这种类似 ‘ASYNC_ADD’ 这样的action，很明显就是带有副作用的异步action命令，reducer是不能处理这种带有副作用的命令，只能处理同步代码。所以reducer中就不会写关于ASYNC_ADD的逻辑，相当于这个next执行的没有任何意义也不会发生任何事情(因为redux内部会自己对比reducer执行完的新生成的state和上一次旧的state。这里很明显state不会发生任何变化，所以redux不会做任何的处理)。
所以也就是说主要触发了我们监听的action的是下面这个channel.put(action)。
可以回到上面那个channel函数中看下put的定义
<pre>
  <code>
    function put(input) {
      // 一些容错处理...
      const cb = takers.shift()
      cb(input)
    }
  </code>
</pre>

put中从takers中，把第一位的回调函数拿了出来，这个回调正好就是上面放进去的proc中的next函数，然后将input(也就是action) 作为参数执行该next。
<pre>
  <code>
    // 这里的arg就是传进来的input也就是action 'ASYNC_ADD'
    function next(arg, isErr) {
      try {
        // 省略一些代码...
        let result = iterator.next(arg)
        if (!result.done) {
          digestEffect(result.value, parentEffectId, '', next)
        }
      } catch (error) {
        // 省略一些代码...
      }
    }
  </code>
</pre>

现在这个next中执行了iterator.next(action)，就是执行了例子中的counterWatcher的第二个yield后面的函数put
<pre>
  <code>
    export default function* counterWatcher() {
      let action = yield take(types.ASYNC_ADD)
      yield put({type: types.ADD}) // 由于上一次调用next一直卡在了上面那行，所以这一次调用next就会走到这行执行put
    }
  </code>
</pre>

所以这次得到的result其实就是put函数执行完的返回值，put的返回值基本和take的一样，大概是这种格式 { [IO]: true, type: 'put', payload }。之后next函数继续执行同样的逻辑 digestEffect → finalRunEffect → runEffect → runPutEffect(这次会走到这个分支，因为返回的type是put)
<pre>
  <code>
    function runPutEffect({ channel, action, resolve }, cb) {
      // 省略一些代码...
      let result = (channel ? channel.put : env.dispatch)(action)
    }
  </code>
</pre>

runPutEffect中，实际就是执行了env.dispatch(action) 这个env.dispatch是一开始就通过proc传进来的env的属性，这个dispatch是redux的applyMiddleware中间件函数重新封装的dispatch，基本上和原生redux的dispatch没啥太大区别，可以约等于redux原生的dispatch。
所以这里就基本上相当于执行了原生的dispatch(action)，这里的action通过我们的例子可知
<pre>
  <code>
    yield put({type: types.ADD})
  </code>
</pre>

action是同步的 ’ADD‘， 同步处理的操作是可以放在reducer中的，所以从这里开始，就是redux自己处理reducer的逻辑了。

基本上redux-saga中的take和put主要的原理就是这样~下次有空再一起了解一下比较复杂的takeEvery的原理吧~

