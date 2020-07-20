---
title: Vue的数据驱动之派发更新
date: 2019-05-26 21:58:39
tags: Vue
---

 之前的生成响应式对象和收集依赖，为的就是准确的派发更新，下面从set开始，看下派发更新的过程：

### 更新过程

```js
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
	...
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

set中除去前面的判断，主要逻辑有三步：

1. 赋值，把新值通过setter方法赋值或者直接赋值原属性。
2. 调用observe(newVal)，将新值改造成响应式对象。
3. 执行dep.notify()，通知所有的订阅者进行更新，下面来看下notify

```js
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```

notify其实也很简单，就是把订阅该依赖的watcher（放在subs中）遍历一遍，都去执行update方法。

```js
  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

这里先忽视掉watcher为lazy（计算属性）和同步的watcher的情况，直接来看queueWatcher方法：

（文件位置：/src/core/observer/scheduler.js）

```js
const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

这里定义了一个队列queue，用来维护需要在这一个周期中需要更新的watcher，用flushing表示是否正在执行队列中的任务，从而对新加入的watcher进行不同的处理，用waiting来控制nextTick(flushSchedulerQueue)的执行只会有一次。

回到queueWatcher方法本身，首先通过has[id] 来保证同一个watcher只被添加以此。然后判断flushing，如果不是正在执行队列中的watcher，那么可以直接把这个watcher加到队列中，如果正在执行，那么把这个watcher按照它的id加入到队列中，这就意味着，如果它之前的id都被执行了，它将会被立即执行。接下来会判断是否需要waiting，当队列被flushing结束后waiting才会为false，这时才可以再次执行nextTick(flushSchedulerQueue)，这可以保证同一时间内，nextTick(flushSchedulerQueue)的执行只有会一次。

nextTick和我们平时在Vue中使用的$nextTick是同一段代码，它的作用是让一些方法异步的在下一个时间周期中执行，他会根据不同的环境使用不同的底层方法，比如setImmediate和MessageChannel，setTimeout、Promise等等。

回到queueWatcher方法中，nextTick(flushSchedulerQueue)会在下个周期执行flushSchedulerQueue:

```js
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

在这个方法中，首先对queue进行了排序，保证序号较小的watcher排在更前面，这么做是为了：

1. 组件在更新时是由父到子的（因为父节点的watcher的id总会小于子节点的id）
2. 用户创建的watcher比渲染watcher更早被更新（因为用户watcher是被更早创建的）
3. 如果一个组件在他的父组件更新时(即执行run()时)被销毁了，那么它对应的watcher都应该被跳过，所以父组件的watcher应该被先执行。

排完序后就是对队列的遍历，这里并没有缓存队列的length，这时因为有可能在run的时候有新的watcher加入到队列中(其实这里不太明白为什么会在run的时候加入新的watcher，run难道不都是同步执行的吗？)。下面来看下run：

```js
  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
```

在run中，首先执行this.get()，这是为触发更新，在渲染watcher中，调用this.get()就会执行到这段代码：

```js
 updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
```

从而触发页面对应部分的重新渲染。对于我们自定义的用户watcher，如果新旧值不相同或者这个值是Object类型或者是deep模式，那么就修改对应的属性值，并且执行watcher的回调，会把value和oldValue都作为参数传入，这也是我们在watcher中能在回调函数的参数中拿到新旧值的原因。

执行完run之后，回到flushSchedulerQueue中，会执行resetSchedulerState来恢复状态：

```js
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

会把queue清空，并且waiting和flushing都置为false，等待下一次的flush。并且会调用activated和updated钩子函数。

## 总结

派发更新的过程，实际上就是触发响应式对象的setter之后，把依赖中所有的观察者(watcher)，全部触发一遍update的过程，在vue中对这个过程利用了队列进行了优化，会把一些watcher的变化统一执行，这样可以减少不必要的更新。这也是我们更新的数据是异步的原因，比如我们的某段代码依赖前面对数据修改后的DOM变化，那么我们就需要在下一个周期里去获取这个数据变化，比如使用this.$nextTick()方法。

到这里，我们已经把Vue的数据驱动的整个过程过了一遍，从生成响应式对象，到收集依赖，最后再到派发更新。但是对于数据变化后组件的实际更新过程还没有了解，也就是patch的过程，这部分内容将在之后的学习中分析。

