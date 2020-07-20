---
title: Vue的数据驱动之收集依赖
date: 2019-05-23 23:17:55
tags: Vue
---

 在Vue中，通过initState对props、data等数据进行了初始化，将其转化为了响应式对象，但是需要在数据发生变化时触发指定DOM或数据的更新，还需要知道哪些地方依赖这些数据。接下来就来学习一下，Vue是如何收集这些依赖的。

### defineReactive 中的 getter

（文件位置：src/core/instance/observe/index.js defineReactive()）

```js
const dep = new Dep()

const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
    return
}

// cater for pre-defined getter/setters
const getter = property && property.get
const setter = property && property.set
if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
}

let childOb = !shallow && observe(val)  
Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
		...
    }
  })

```

依赖的收集在getter中完成，主要是通过一个Dep类型的对象完成的。可以看到，每个访问式对象都有dep（如果是在Observe构造方法中调用的defineReactive，那么在Observe对象中还有个dep属性，这里其实不太明白，为什么两边都要有dep属性，只在Observe中声明不就够了吗？）。经过`Object.defineProperty`方法修改之后的对象的值，在获取这个属性的值时就会触发get方法，这里调用了`dep.depend()`，并且判断这个属性是否有子属性，如果有那么也对其进行依赖的收集。这样的话，即便某对象的子属性中的值发生了变化，也能够被获知。

接下来，来看下Dep这个类是如何保存依赖的。

### Dep

（文件位置： /src/core/observer/dep.js）

```js
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

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
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

这个类其实很简单，用subs来保存watcher，用addSub和removeSub方法对subs进行修改，通过depend方法进行依赖的收集，用notify方法来派发更新。需要注意的是，Dep有个静态属性target，表示当前正在进行依赖收集的watcher，因为是静态属性，所以同一时间下只能有一个全局的watcher。

Dep实际上就是对watcher进行管理的类，抛开Watcher谈论Dep是没有意义的，下面来看下Watcher的定义：

### Watcher

(文件位置：/src/core/observer/watcher.js)

```js
/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 */
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }
  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  } 
}
```

watcher的定义比Dep复杂些，在这里只列出了和依赖收集相关的属性和方法。

从Watcher的构造方法中，主要是一些属性的赋值和初始化，注意到，其中有这几个属性`deps`, `newDeps`, `depIds`, `newDepIds`，`deps` 和`newDeps`表示这个Watcher持有的dep实例，而`depIds` 和`newDepIds`则表示这个实例持有的dep实例的id的集合，每个watcher保存了两套实例是为了方便清空依赖，这个在后面在详细分析。

除了一些属性，还有addDep、cleanupDeps、depend这几个和收集依赖相关的方法。





### 过程分析

在之前的[Vue的数据驱动之初始化](https://zacherywu.github.io/2019/04/04/Vue的数据驱动之初始化/) 中，曾经见到过对watcher的使用，也就是在那里完成第一次的依赖收集，在mountComponent中，调用new Watcher实现对数据依赖收集。

(文件位置：/src/core/instance/lifecycle.js)

```js
 updateComponent = () => {
     vm._update(vm._render(), hydrating)
 }
   // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```

首先Watcher的构造方法，主要是一些赋值和初始化属性操作，赋值结束之后，会调用get方法，get方法中，首先调用了pushTarget

```js
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}
```

这是在Dep.js模块中的方法，用来修改Dep.target对应的Watcher，并且把这个Watcher压入targetStack栈中，这也是为了之后的恢复使用，暂存起来。

回到watcher中，调用pushTarget之后，会直接调用getter方法：

```js
value = this.getter.call(vm, vm)
```

这个getter方法，就是在声明Watcher时传入的第二个参数——updateComponent方法，就在这里把数据渲染成了vnode，就在这个过程中，会对vm上的数据进行访问，从而触发访问式数据对象的getter。每个数据对象都有个dep，并且在getter中都会触发dep.depend()

```js
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
```

这时由于之前调用pushTarget，所以Dep.target指向了之前的那个Watcher，所以调用deeDep，

```js
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

```

把这个watcher的依赖放到了newDeps中，这时newDepIds属性就派上了用场，用来判断这个依赖是否已经被添加过，保证一个数据不会被添加多次。然后还会执行dep.addSub(this)，也就是把当前watcher添加到依赖的dep的subs中，这样可以在之后数据发生变化时能够通知到对应的watcher。

所以在vm._render()的过程中，会触发所有数据的getter，也就完成了所有的依赖收集。依赖的收集到这里就结束了，然而在后面还有一些操作。

回到watcher的get方法中，在finally中：

```js
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
```

首先调用traverse，这是要递归的去访问value，从而触发value子属性的依赖收集，然后会调用popTarget()

```js
export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

这个方法也声明在dep.js模块中，和pushTarget对应，因为对这个watcher的依赖收集已经完成，所以通过targetStatck.pop()把这个watcher弹出，并且恢复target的指向。

再接下来，会调用cleanupDeps，进行依赖清空。

```js
  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
```

由于之前添加watcher对应的依赖时，只是修改了newDepIds和newDeps，所以只有newDeps中存在的才是现在的Watcher的依赖，deps和depIds中存在的是上次的依赖Dep。所以首先把已经不存在于newDeps中的subs清空，然后交换depIds和newDepIds，交换deps和newDeps。最后清空newDepIds和newDeps。

这么做的原因，是考虑到以下场景的存在：

我们的模板中会使用`v-if`来渲染不同的子模板a和b，如果第一次渲染的是子模版a，那么收集的将是a模板所依赖的数据，这时如果`v-if`条件发生了变化，需要渲染子模版b，收集的则是b模板所依赖的数据。如果不把之前收集的a模板的依赖清空掉，那么在那些数据发生变化时依旧会通知a模板更新，这就造成了不必要的性能浪费。

因此Vue在每次添加新的订阅后会移除旧的订阅，这样就保证了在我们刚才的场景中，如果渲染 b 模板的时候去修改 a 模板的数据，a 数据订阅回调已经被移除了，所以不会有任何浪费。这就是Vue在对于细节的处理，能够在一定的程度上提高性能。

## 总结

经过这次的学习，已经了解Vue收集依赖的过程，以及一些处理依赖的细节。收集依赖主要目的是为了知道在数据发生变化时通知哪些部分更新，也就是所谓的派发更新。前面的生成响应式对象，收集依赖都是为了派发更新做准备，接下来将会分析Vue的派发更新的过程。