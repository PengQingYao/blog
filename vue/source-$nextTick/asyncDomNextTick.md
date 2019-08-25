## 天啦，vue出bug了，DOM又不刷新了？
&nbsp;&nbsp;工作中，用vue开发，经常会碰到用数据驱动dom，然后操作dom却没有效果的情况。如果有用到tab切换加上echarts展示，肯定是气的想砸桌子。下面来谈谈vue中dom的刷新。
###什么是DOM异步更新？
&nbsp;&nbsp; 所谓异步更新，就是vue中用数据去驱动dom，数据变化了，DOM却不会立即的更新，而是在下一个Tick中更新dom。当然，vue中手动操作DOM，DOM是立即刷新的。
&nbsp;&nbsp;什么是数据驱动DOM？其实就是声名一个响应式的数据，当数据改变时，不用手动的操作dom，vue会自动的更新视图。
先来简单的看下代码
![异步dom源码](/blogImage/vue/asyncDomNextTic/asyncDom.jpg)

在浏览器中看vue的dom异步刷新
![异步dom源码](/blogImage/vue/asyncDomNextTic/asyncDom.gif).
&nbsp;&nbsp; 从图中可以看出，当showDom发生变化的时候，‘有梦想的api搬运工’ 本应该隐藏的，而却没有隐藏,他会等到下一个队列中去刷新,这就是所谓的dom异步刷新。同样的，如果工作中通过数据去驱动dom，然后立即的去操作这个dom，有很大概率很报错哦。
&nbsp;&nbsp;看看手动的操作dom是什么结果呢。
![同步dom源码](/blogImage/vue/asyncDomNextTic/synchronous.gif)
从上图可以看出，当手动操作dom时候，当修改dom后，颜色是立刻变化的。这样就不会碰到使用异步dom出现的问题了。

&nbsp;&nbsp;如果在vue中使用了数据去驱动dom，碰到了问题该怎么办呢？当然是今天的主角<font color="#ec7ab4" face="微软雅黑">$nextTick</font>去解决了。什么是nextTick?套用官网的话来说
<font color="#ec7ab4">将回调延迟到下次 DOM 更新循环之后执行。在修改数据之后立即使用它，然后等待 DOM 更新。它跟全局方法 Vue.nextTick 一样，不同的是回调的 this 自动绑定到调用它的实例上。</font>解释的果然比我正规。
![我很菜](/blogImage/vue/asyncDomNextTic/jishu.jpg).

如果对浏览器eventloop 和microtask,macrotask还不熟悉的，请左拐，等会儿再来这儿看。[从浏览器多进程到JS单线程，JS运行机制最全面的一次梳理](https://juejin.im/post/5a6547d0f265da3e283a1df7#heading-17).
别忘记回来哦，后面的更精彩。下面的vue源码需要用到此部分的知识。

## 使用异步刷新有什么优点？
&nbsp;&nbsp;谈到vue的异步更新，不得不谈到vdom(虚拟dom)。什么是vdom？<font color="#ec7ab4">虚拟dom就是一个javascript对象，这个对象内部有许多dom的属性，以此来模拟一个真正的dom对象。而vue中所操作的dom，就是这个vdom，等到所有dom完成，把vdom挂载到真实的dom下，减少对真实dom的操作</font>。这里就不展开多讲了。

&nbsp;&nbsp;到底异步刷新有什么优点呢？看下面两端代码。
![我很菜](/blogImage/vue/asyncDomNextTic/numberUpdate.jpg).

如果通过vue提供的数据驱动的方式异步刷新dom，add()方法，numbershu改变后，dom并不会立即的刷新，等到for循环结束后，得到最终值后，更新一次dom。dom的重绘与回流只发生一次。
而通过computedNum()方法，每一次的for循环，都会触发一次视图的更新，引发多次的dom的重绘与回流。网站的性能十分的差。
而如果把computedNum()方法改成这样
```
 computedNum() {
     let m = 0;
       for(let i = 0; i < 100 ;i ++) {
           m = i;
       }
        this.$refs.computedNum = m;
    }

```
如果改写成这样，dom只会刷新一次。反而比vue使用vdom进行diff算法进行计算，然后更新dom，性能更加的好。
关于vue的vdom设计理念，其实就是在用户在不考虑性能优化的情况下，替用户进行看得过去的优化，并不能保证vue中的dom操作都是最优选择，只是让用户开发的更爽。

##vue异步更新dom得策略，以及nextTick
说到vue异步更新dom得策略，得先看一下nextTick的实现原理。
 ### nextTick实现原理
 ```
let callbacks = [];
let pending = false;
let timerFunc



if(typeof Promise !== 'undefined) { //判断当前浏览器是否支持promise，如支持，用promise实现异步刷新dom
    const p = Promise.resolve();
    timerFunc = () => {
        p.then(flushCallbacks)
    }
} else if(typeof MutationObserver !== 'undefined') {
    let counter = 1;
    const observe = new MutationObserver(flushCallbacks);
    const textNode = document.createTextNode(String(counter));
    observe.observe(textNode,{
        characterData: true
    })
    timerFunc = () => {
        conter = (conter + 1) % 2;
        textNode.data = String(counter)
    }
} else {
    timerFunc = () => {
        setTimeout(flushCallbacks,0)
    }
}

function flushCallbacks () {
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}

function nextTick(cb,ctx) { //ctx是vue实例
    let _resolve;
    callbacks.push(() => {
        if(cb) {
            cb.call(ctx)
        } else if (_resolve) {
            _resolve(ctx);
        }
    })
    if(!pending) {
        pending = true;
        timerFunc();
    }
    if(!cb && type Promise !== 'undefined) {
        return new Promise(resolve => {
            _resolve = resolve
        })
    }
}

 ```
 这里把实现nextTick最重要的三部分扣了出来，并且简化了一下，如果有感兴趣的同学，可以去源码里面看完整版的。
&nbsp;&nbsp;1.首先 nextTick需要传入一个回调函数，在当前执行栈内，当第一次调用nextTick方法的时候，callbacks里push回调函数，此时，pending的值是默认的false，然后改变pending，执行timerFun();p.then异步执行flushCallbacks函数，其实就是执行callbacks数组里的函数。<font color=ec7ab4>在当前执行栈内，如果有第二次调用nextTick函数，继续向callbacks里推入回调函数。但是pending已经是true了，不会在重复调用timerFunc，由于flushCallbacks是异步执行的，callbacks内的回调还未执行，又向callbacks推入一个新的回调函数，此时callbacks数组里有两个回调，以此类推。</font>等到当前栈任务执行完，开始执行flushCallbacks，改变pending的状态，为下一个队列做铺垫。此时callbacks数组内已经有n个回调，然后执行这些函数。
&nbsp;&nbsp;2. 到底采用哪个方法去异步执行？根据上面的判断，如果有promise，就使用promise，如果没有，就使用MutationObserver，如果还没有，就使用setTimeout。(IE:你们都看我干啥？我长得漂亮？)。简单说一下MutationObserve，h5新出来的api，当dom变化时，会触发回调。和promise一样，都是microtask任务。[点我看MutationObserver的相关api](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)。
&nbsp;&nbsp;3.根据源码我知道了，nextTick还能当作一个promise使用。this.$nextTick().then();当然，$nexTick()不能传函数哟。

### DOM的异步更新
谈到vue，肯定张口就来，通过defineProperty重写get与set方法，实现数据的双向绑定。毕竟面试必备的一句话。 下面来谈一谈dom是如何异步更新的。
当vue中的的响应式数据发生变化，通过set方法，会调用Watch类的update()方法,而update方法会调用queueWatcher方法来更新视图，看一下queueWatcher的定义
```
let waiting = false;
let index = 0;
let has = {};
...
...
export function queueWatcher (watcher) { // watch的实例
  const id = watcher.id //每个响应式数据中都有一个独立watch.id
  if (has[id] == null) { // 如果一个响应式数据多次改变，只会向queue数组中推入一次，视图刷新只更新最终的值。(还记得上面的add()方法吗，numbershu这个变量只更新一次)
    has[id] = true
    queue.push(watcher)
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) { // 如果是设置的不是异步更新，就执行更新dom。(好像从来没有用到过)
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
}
```
上面的代码也都是简化的代码，便于理解，如果对原版感兴趣的，可以去看vue源码。

&nbsp;&nbsp;1.当第一次响应式数据变化后，queueWatcher方法会向queue队列中添加一个更新视图的回调函数，如果queue中已经有了这个实例，就不添加。而flushSchedulerQueue这个方法就是遍历执行queue这个数组内的函数，queueWatcher这个方法最后调用了nextTick(flushSchedulerQueue),上面讲到过，nextTick会把回调放入到callbacks的数组内，这里是异步执行。所以calbacks里会有一个flushSchedulerQueue的函数。而flushSchedulerQueue内又有一个queue队列，当前执行栈内，如果有第二个响应式数据发生变化，又会向还未遍历的queue队列中添加watch实例，以此类推。当前执行栈任务结束后，调用任务队列中的callbacks内的回调函数，调用到flushSchedulerQueue函数时，此函数又会遍历调用queue的回调函数，最终调用watch.run()方法去更新视图。

说白了，vue的异步更新，最终的calbacks数组结构就类似[fn,fn,flushSchedulerQueue,fn]。

flushSchedulerQueue函数内部又有一个queue数组等待去遍历，其结构类似[fn,fn,fn],当然纯属猜测。


❤️ 如果各位看官看的还进行，请给一个赞，顺手点个关注，就是对我的最大支持









