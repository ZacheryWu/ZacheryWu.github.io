---
title: Vue组件的初始化
date: 2019-05-11 15:28:39
tags: Vue
---

组件化是Vue的另一个另一个核心思想，通过组件化我们可以将重复的代码封装为组件，重复使用，方便开发和维护。我们在实际开发中也是这样的，编写一堆组件拼凑成页面，这次的学习就从源码的角度去了解组件的初始化、渲染、挂载等等。

将以下面这段简单的代码为例，来分析组件初始化的流程。

```js
var App = {
    name: 'app',
    template: '<span>HelloWorld</span>'
}
var app = new Vue({
    el: '#app',
    render: h=>h(App)
})
```

在通过createElement生成vnode之前，整个Vue的初始化是一样的，因为还没到渲染组件那一步，正是在createElement中，根据传入的参数不同，有了不同的处理方式。

## createComponent

在createElement中，有这样的一段代码：

```js
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
```

根据传入的参数tag，会进行不同的处理，在之前学习数据初始化时，这里传入的tag是一个字符串，因此通过new VNode的方式直接生成vnode；而对于组件，传入的tag是一个组件对象，会采用createComponent方法来生成vnode。

createComponent的定义是这样的：

(文件位置：/src/core/vdom/createComponent.js)

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```

这里传入的Ctor就是那个组件对象。可以看到，如果Ctor是对象类型，那么会进行一个赋值操作：

```js
  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }
```

这个baseCtor其实就是Vue的构造方法。这个$options._base是在initGlobalAPI方法中添加的：

（文件位置：/src/core/global-api/index.js）

```js
  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
```

并且还通过initExtend在Vue上添加了extend这个全局方法：

(文件位置：/src/core/global-api/extend.js)

```js
  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    cachedCtors[SuperId] = Sub
    return Sub
  }
}
```

在extend中，创建了一个VueComponent方法，其中会调用_init方法，并且通过原型模式实现了对Vue的继承，还会把传入的参数和Vue的原有参数合并，生成新的options挂在方法上，并且会在init中通过mergeOptions挂在Vue的实例vm.$options上。在Vue中，所有的组件都是通过VueComponent方法生成的，组件会在update的时候被实例，具体细节在之后学习。

回到createComponent，在通过extend方法生成组件构造方法后，会安装组件的钩子函数：

```js
installComponentHooks(data)
```

```js
function installComponentHooks (data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

function mergeHook (f1: any, f2: any): Function {
  const merged = (a, b) => {
    // flow complains about extra args which is why we use any
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}
// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        // vue-router#1212
        // During updates, a kept-alive component's child components may
        // change, so directly walking the tree here may call activated hooks
        // on incorrect children. Instead we push them into a queue which will
        // be processed after the whole patch process ended.
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },

  destroy (vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy()
      } else {
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
  }
}

const hooksToMerge = Object.keys(componentVNodeHooks)

```

installComponentHooks和mergeHook很简单，就是把componentVNodeHooks中的属性合并到data.hook中，如果已存在那么就同时调用原有的钩子函数和组件的钩子函数。这些钩子函数会在patch的时候被调用。

钩子函数挂载完毕之后就是生成vnode了，直接调用VNode构造方法生成新的vnode即可：

```js
  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )
```

和普通的vnode不同，组件的vnode是没有children的。

### 小结

组件通过createComponent创建vnode，主要有三个部分，一是根据传入的参数生成组件的构造方法，二是添加组件的钩子函数，三是生成vnode并返回。在返回vnode之后，会进入_update方法，然后执行patch，组件的patch又有些不同。

## patch

在patch中：

(文件位置：/scr/core/vdom/patch.js)

```js
 	if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
        ...
    }
```

如果是初始化，就直接调用createElm，在createElm中

（文件位置：/src/core/vdom/patch.js）

```js
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }
```

下面来看create Component：

（文件位置：/src/core/vdom/patch.js）

```js
  function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */)
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue)
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
```

在这里会去获取vnode.data.hook.init，这个是上节createComponent中通过InstalComponentHooks给data添加的钩子方法，在这里能够获取到。获取到之后直接调用这个init方法：

（文件位置：/src/core/vdom/create-component.js)

```js
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
```

我们先不看keepAlive为true的情况，可以看到在这里通过createComponentInstanceForVnode创建了组件的实例，然后通过实例上的$mount方法对组件进行了挂载。下面就来看下createComponentInstanceForVnode：

(文件位置：/src/core/vdom/create-component.js)

```js
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

在这里在options上添加一些参数，注意到_isComponent为true，这里在之后的初始化会用到，然后通过之前生成的组件的构造方法去实例化这个组件，这个实例化方法:

```js
function VueComponent (options) {
    this._init(options)
}
```

会调用`_init`，回到梦开始的地方，对组件进行初始化。组件的初始化还是有些不同的，我们再来看下`_init`

```js
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

	...

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
	...

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

在这里因为之前传的参数option中_isComponent为true，所以合并options的方法变了，改为通过initInternalComponent方法合并配置：

```js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

在这里把我们之前在createComponentInstanceForVnode中的参数放到了$options上，在合并完参数后，就到了挂载，因为是组件，vm.$options上是没有el的，所以这里不会进行挂载，而是在create-component.js中的componentVNodeHooks上的init钩子函数中的：

```js
child.$mount(hydrating ? vnode.elm : undefined, hydrating)
```

来对组件进行挂载，这个挂载也会进入到mountComponent，然后执行_render()方法

(文件位置：/src/core/instance/render.js)

```js
  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options
	...
    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      // There's no need to maintain a stack because all render fns are called
      // separately from one another. Nested component's render fns are called
      // when parent component is patched.
      currentRenderingInstance = vm
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
      ...
    } finally {
      currentRenderingInstance = null
    }
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
	...
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}

```

`_parentVnode`是我们之前合并配置时记录的当前组件的父组件的VNode，render函数生成的vnode就是当前组件的vnode，将vnode.parent指向了`_parentVnode`，表明两个组件的父子关系。

在render完之后，继续执行_update，去渲染vnode：

（文件位置：/src/core/instance/lifecycle.js）

```js
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```

这里又通过`vm._vnode`存储下了当前节点的vnode，而之前在render中通过vm.$vnode存储下了父节点的vnode，因此`vm._vnode`和`vm.$vnode`是父子关系。也就是`vm._vnode.parent === vm.$vnode`。而且在patch之前，通过setActiveInstance，存储下了当前正在渲染的节点：

```js
export let activeInstance: any = null

export function setActiveInstance(vm: Component) {
  const prevActiveInstance = activeInstance
  activeInstance = vm
  return () => {
    activeInstance = prevActiveInstance
  }
}
```

这个activeInstance的作用就是在全局范围内，保存当前上下文的Vue实例。在之前createComponentInstanceForVnode中，传递的参数就是用activeInstance来获取当前实例的，并且用parent字段保存放到options中传递下去：

```js
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent//这个parent就是activeInstance
  }
```

这里的options会在initInternalComponent中被合并，然后在initLifeCycle中：

```js
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

在vue的实例上用$parent保存了这个parent，这就保证了在实例中能够找到父子组件的关系。

回到`_update`中 ，通过setActiveInstance，修改了activeInstance，并且用preActiveInstance保存了前一个实例，并在patch结束后，将preActiveInstance赋值给activeInstance。由于vue的初始化实际上是一个深度优先遍历的过程，所以实际上preActiveInstance和activeInstance是父子关系，当一个vm实例完成了它的初始化，activeInstance就会回到它的父实例，这样就完美保证了在createComponentInstanceForVnode，能够完美的通过activeInstance传入当前子组件的父实例，并且在_init中通过initLifeCycle将这个父子关系保存下来。

下面进入到patch中，就又回到了本章开始的createElm：

```js
  function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    ...

    vnode.isRootInsert = !nested // for transition enter check
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      ...

      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      /* istanbul ignore if */
      if (__WEEX__) {
        ...
      } else {
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        insert(parentElm, vnode.elm, refElm)
      }

      if (process.env.NODE_ENV !== 'production' && data && data.pre) {
        creatingElmInVPre--
      }
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
```

根据我们的示例，之前生成的vnode并不是组件类型，所以没有vnode.data.hook，所以这里的createComponent返回为false。接下来的就和我们上一章学习的过程一样，首先生成一个父节点的占位符，然后遍历所有的子VNode，如果是组件，那么调用createElm，再次递归从本章开头再走一遍，如果不是组件，那么调用nodeOps里的DOM方法，生成DOM节点，然后调用insert完成DOM的插入。

## 总结

到这里我们又从组件的角度走了一遍初始化的过程，和非组件的初始化有不小的不同，不过整体流程是一样的。