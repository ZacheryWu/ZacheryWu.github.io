---
title: Vue的数据驱动之响应
date: 2019-05-19 15:40:14
tags: Vue
---

我们开发一个页面，主要有两部分，一是把数据渲染到页面，二是处理页面的交互，前面学习的就是把数据渲染到页面的过程，也就是初始化的过程。这次来学习页面的交互，这里的交互不止指的是用户的操作事件，还有数据因为其他原因变化后引发页面的页面变化。

如果不使用Vue来开发，那么我们处理交互通常会：

1. 监听这个事件
2. 修改数据
3. 操作DOM重新渲染

即便使用了Vue，实际上进行的也是这三个步骤，只是操作DOM重新渲染这一步由Vue来帮我们完成。在Vue中为了知道需要修改的DOM，采用了观察者模式，因此我们的数据响应主要分为以下三步：

1. 生成响应式对象
2. 收集依赖
3. 派发更新

## 生成响应式对象

Vue的响应式对象核心就是[Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)，这个方法会直接在一个对象上顶一个新的属性，或修改一个对象的现有属性并返回这个对象。

```js
Object.defineProperty(obj, prop, descriptor)
```

该方法的核心是descriptor（描述符），它有以下属性：

1. configurable，只有这个属性为true时，这个对象的这个属性描述符才能够被改变，默认为false。
2. emumerable，只有这个属性为true时，这个对象的这个属性才能够被枚举方法枚举，默认为false。
3. value，表示这个属性对应的值
4. writable，只有这个属性为true时，value才能够被修改。
5. get，这是个函数，当访问属性时会调用这个函数，并且该函数的返回值会作为属性的值。
6. set，这是个函数，当对这个属性进行写操作时会调用这个函数。

Vue就是通过给数据添加set和get描述符属性，来控制数据的读写，从而在数据改变时去修改DOM。我们简单的把拥有getter和setter的对象称为响应式对象。

### initState

Vue初始化阶段调用initState方法：

（文件位置：/src/core/instance/state.js）

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

这里主要是将参数中的props、methods、data、computed、watch分开，调用不同的方法进行初始化。接下来主要看下props和data的初始化。

#### initProps

```js
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
     ...
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

主要是遍历props属性，通过defineReactive方法将props上的所有属性变成响应式对象，并且把属性通过proxy方法代理到实例vm上，让开发者能够直接通过Vue实例获取到props属性。

#### initData

```js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      ...
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

初始化data的过程和初始化props差不多，一是对data属性遍历，通过proxy把每个data上的值代理到vm上。二是通过observe方法把整个data变成响应式对象。

可以看到initData和initProps的初始化都是把他们代理到vm上并且改造为响应式对象的过程，下面来看下把props和data中的属性代理到vm上的方法proxy。

### proxy

```js
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

proxy中，最终调用了Object.defineProperty，给Vue的实例添加了访问器属性。让用户可以通过vm直接获取或者修改data、prop中的属性。这也是我们可以直接通过this.msg访问到data或props中数据的原因。

接下来看下是如何通过observe把普通对象变成访问式对象的。

### observe

（文件位置: /src/core/instance/observer/index.js）

```js
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

observe方法就是给非VNode类型的数据添加一个observer，如果该对象已经有了`__ob__`属性，那么说明这个对象已经有observer了，直接返回即可。如果这个对象还没有`__ob__`这个属性，那么调用new Observer（）实例化一个新的并返回。

### Observer

接下来看一下Observer这个类：

（文件位置: /src/core/instance/observer/index.js）

```js
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

Observe是个类，主要作用是给对象的属性添加getter和setter方法，用于收集依赖和派发更新。

在构造函数中，首先通过def方法给传入的对象添加了`__ob__`属性，并且指向自己，这样就能很方便的获取到访问式对象的observer。然后判断对象的类型，如果是数组，那么通过observeArray方法对对象处理，如果是普通对象那么直接俄通过walk方法处理。

在observeArray中，会遍历每一项，然后递归的调用observe方法，将每一项都生成访问式对象。

在walk中，会对对象中的每个属性调用defineReactive方法，将对象中的每个属性都生成访问式对象。

最终都到了defineReactive方法上，接下来看下这个方法的实现：

### defineReactive

(文件位置：src/core/observer/index.js)

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

首先通过`Object.getOwnPropertyDescriptor`方法拿到对象上原本就有的访问器属性，然后通过递归的方式对子对象调用ovserve方法，这样可以确保无论对象的层次多深，都能够把每个属性变成访问式对象，在访问和修改每个对象的属性时都会触发getter和setter。最后就是根据原有的访问器属性，通过Object.defineProperty方法给属性添加getter和setter方法。getter和setter方法中还有不少的逻辑，这个在之后学习。

## 总结

data和props中的属性最终都通过defineProperty方法变成了访问式对象，这使得我们在访问或修改其中的属性时，会触发getter和setter，我们借此就能够在访问数据和修改数据时自动执行一些逻辑，也就是通过getter去收集依赖，通过setter去派发更新。接下来就继续学习Vue数据驱动中的很重要的收集依赖。