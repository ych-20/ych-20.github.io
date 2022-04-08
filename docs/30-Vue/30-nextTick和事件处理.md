# Vue 中nextTick 和 事件的处理
---


## nextTick
```
单看nextTick 其实很简单，在vue2.6之后就改为全部为微任务执行，做了一些兼容性判断，如果有原生Promise
则用Promise
注意点: 
1. nextTick的上一次回调还没执行完成（pending = true）则回调函数 会先放入callbacks缓存起来。这样多
个使用nextTick一样会在同一个事件循环内完成.
2. 如果平台不支持微任务 就会用setTimeout 宏任务来兜底，但是这个兜底在vue3+就去掉了。也就是放弃支持
IE低版本
```
### next-tick.js
```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'
export let isUsingMicroTask = false
const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
## 事件处理 
```
vue中的事件可以分为三类
1. 自定义事件
2. 组件事件
3. DOM事件

```
### 自定义事件
```
最简单的自定义事件 通过 $on 注册，通过 $emit 去触发
$on 注册的事件 会保存在 实例 _event 字段  
```
`贴部分代码`
##### $emit
`这里可以看出 event的name 是不区分大小写的`
```js
// $emit
Vue.prototype.$emit = function(event) {  
    var vm = this;    
    var _events= event.toLowerCase();    
    var cbs = vm._events[_events]; 
    if (cbs) {
        cbs = cbs.length > 1 ? toArray(cbs) : cbs;        
        var args = toArray(arguments, 1);        
        for (var i = 0, l = cbs.length; i < l; i++) {
          cbs[i].apply(vm, args);
        }

    }    
    return vm
};
// $on 
Vue.prototype.$on = function(event, fn) {
    var vm = this;
    if (Array.isArray(event)) {
        for (var i = 0,l = event.length; i < l; i++) {
            this.$on(event[i], fn);
        }
    }
    else {
        (vm._events[event] || (vm._events[event] = [])).push(fn);
    }
    // 为了链式调用
    return vm
};
```
### 组件上的事件
`
子组件实例化的时候 会把事件注册在子组件实例的_event中 所以在子组件this.$emit 时 可以触发事件
`
### DOM事件
```
tips:最早走的微任务，后来有连续事件(例如冒泡)导致的渲染上的bug改为使用宏任务解决，在2.6之后再改回微任务
target$1.addEventListener(
      name,
      handler,
      {capture: capture, passive: passive}
    );
passive: 告诉浏览器 这个事件不会调用 event.preventDefault()，主要用于性能优化，例如滚动屏幕时 会触发touch事件，但是浏览器并不知道
touch事件是否会阻止默认行为从而不知道是否滚动。浏览器对touch事件的监听和执行会导致滚动有延时。（passive为true，就是告诉浏览器不会调用preventDefault，浏览器直接执行滚动就行，不用考虑回调函数了。）
```
