---
title: Vue的数据驱动之初始化
date: 2019-04-04 07:44:02
tags: Vue
---

 Vue的一个核心思想是数据驱动，数据驱动改变了我们写前端代码的方式。在此之前，大都是通过直接的修改DOM来控制页面变化的，而数据驱动则不会去直接修改DOM，我们只需要去专注于数据的维护，DOM的变化由框架来完成。我们所有的逻辑都是对数据的修改，不涉及DOM操作，这样的代码非常利于维护。

在Vue中，我们通常采用模板语法来进行开发，这种方式简单明了

```js
var app = new Vue({
    el: '#app',
    template: `
	<div>
		{{message}}
	</div>
	`,
    data: {
        message: 'Hello Vue!'
    }
})
```

Vue根据模板和数据，渲染出页面。这次就从源码的角度分析Vue是如何将数据和模板渲染成页面的，至于数据驱动的另一个重要部分——数据更新触发视图变化，将会在之后学习。

## new Vue

从入口new Vue开始。

Vue的定义是这样的：(文件位置：/src/core/instance/index.js)

```js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

当然了，在引入Vue的时候，已经对Vue做了处理，Vue的原型上有很多我们会用到的方法和属性，在后面用到的时候会介绍，除了原型上的属性，在Vue上还有很多的全局属性和方法。

上面的`this._init(options)`就是Vue原型的方法，用来进行初始化，这个方法是在initMixin()方法中定义的：

(文件位置：/src/core/instance/init.js)

```js
 Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

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
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
```

初始化过程中进行了：

- 合并配置
- 初始化生命周期
- 初始化事件中心
- 初始化渲染
- 初始化data、prop、method、computed等
- 挂载

而要将模板和数据渲染成真实的DOM，重点就在最后一步挂载上。

## Vue实例挂载

在init.js中，最后一步挂载：

```js
if(vm.$options.el) {
    vm.$mount(vm.$option.el)
}
```

根据上面的代码这里的$options就是我们传入的参数和vue本身携带的参数合并后的配置，可以看到我们的实例中是有传入el的，所以这里会通过vm.$mount(vm.$option.el)进行挂载。

$mount是在Vue的最外层入口函数中声明的（这次学习使用的是Runtime-Compiler的构建）

（文件位置/src/platforms/web/entry-runtime-with-compiler.js）

```js
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```

可以看到，在给原型上的$mount赋值之前，先把之前的$mount暂存到了变量mount中。

在vue中，所有的组件的渲染都要通过render 方法，无论我们是通过.vue文件开发，还是使用了template，最终都会被转化成render方法。

因此在这段代码中会判断参数中是否有render方法，如果没有那么就需要根据template生成一个render方法，使用的是compileToFunctions方法，在此，不展开学习，生成render 方法之后再调用原本的$mount方法。如果有，那么就直接调用原本原型上的$mount方法。

原本的$mount方法是这样声明的：（文件位置：/src/platform/web/runtime/index.js）

```js
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

$mount方法支持传入两个参数，第一个参数el可以是字符串也可以是DOM对象，如果是字符串并且是浏览器环境那么调用query方法转化为对应的DOM对象。第二个参数表示是否采用服务端渲染，在此不展开。

处理完el之后继续调用moutComponent，进行真正挂载：

（文件位置：/src/core/instanc/lifecycle.js）

```js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
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
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

在对一些异常情况进行处理之后，就是这段代码的核心，创建一个Watcher，这个watcher的回调函数中会调用前面声明的updateComponent，updateComponent中首先调用`vm._render()`生成一个虚拟节点vnode，然后调用`_update`将vnode渲染成为DOM节点更新页面。

vm.$vnode表示父节点的虚拟节点，如果这里为null，那么表明当前的节点是根节点。如果是根节点，那么将vm._isMounted设置为true，然后调用mounted钩子函数。

### 小结

挂载的流程最终落在了两个方法上，一个是`_render`一个是`_update`，接下来将主要学习这个两个函数。



## render

render是Vue在初始化的时候，通过renderMixin声明在Vue的原型上的方法。

(文件位置：/src/core/instance/render.js)

```js
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
      )
    }

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
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
        } catch (e) {
          handleError(e, vm, `renderError`)
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    } finally {
      currentRenderingInstance = null
    }
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
```

在这里首先对vm.$vnode进行了赋值为了父节点的Vnode，为的是在渲染方法中能够获取到当前节点在DOM中挂载的位置。

然后就是调用render方法，并传入参数vm.$createElement。这个render方法就是我们传入的渲染函数，如果使用模板的方式，就是通过compileToFunction生成的渲染函数。

之前的模板：

```html
<div id="app">
    {{message}}
</div>
```

如果用渲染函数来写的话是这样的：

```js
render: function(createElement) {
    return createElement('div', {
        attrs: {
            id: 'app'
        },
    }, this.message)
}
```

因此，渲染函数的核心就是利用createElement函数，结合传入的标签和标签的描述，生成vnode对象。那么我们来看一下这个传入的$createElement函数。在render.js这个文件中$createElement是这样定义的：

```js
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

这里只是对createElement做了一层封装，多传入了两个参数，真正的createElement在create-element.js中：

(文件位置： /src/core/vdom/craete-element.js)

```js
// wrapper function for providing a more flexible interface
// without getting yelled at by flow
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```

这里的createElement是对`_createElement`函数的封装，封装之后的createElement函数允许传入的参数更加灵活，下面来看位于同一个文件中的真正的核心`_createElement` 函数：

```js
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n' +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
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
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

`_createElement`中，做完对一些特殊情况的处理之后，主要做了两件事，一是对children的规范化，而是生成vnode对象。

### children的规范化

```js
 if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
```

children的规范化主要是这段代码，会针对不同的情况调用不同函数。对于模板编译成的render函数，理论上是不需要进行规范化的，因为compileToFunction函数中会把children处理成Vnode数组。然而又两种情况需要进行规范化处理，一种是函数式的组件，可能会返回一个数组而不是一个根节点，只需要调用simpleNormalizeChildren进行简单的规范化，对其中是数组的进行扁平化处理：

```js
// The template compiler attempts to minimize the need for normalization by
// statically analyzing the template at compile time.
//
// For plain HTML markup, normalization can be completely skipped because the
// generated render function is guaranteed to return Array<VNode>. There are
// two cases where extra normalization is needed:

// 1. When the children contains components - because a functional component
// may return an Array instead of a single root. In this case, just a simple
// normalization is needed - if any child is an Array, we flatten the whole
// thing with Array.prototype.concat. It is guaranteed to be only 1-level deep
// because functional components already normalize their own children.
export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}
```

另一种则是，render函数是用户手写的且children只有一个节点的时候，那么直接生成一个文本节点。或者是`<template>` `<slot>` `v-for` 以及提供了render函数的组件，会生成嵌套的数组，在这两种情况下，就需要调用normalizeChildren进行完整的规范化：

```js
// 2. When the children contains constructs that always generated nested Arrays,
// e.g. <template>, <slot>, v-for, or when the children is provided by user
// with hand-written render functions / JSX. In such cases a full normalization
// is needed to cater to all possible types of children values.
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) {
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

首先会对未定义的子节点或者布尔类型的直接跳过。而如果这个子节点还是数组类型，那么继续递归的调用`nomalizeArrayChildren`进行规范化的处理。而如果是原始类型，那么直接创建文本节点。如果都不是，则会判断是否是文本节点，如果是那么生成文本节点，并且对相邻的两个文本节点进行合并，而如果不是，那么就已经是vnode类型了。

children经过规范化处理，已经变成了vnode类型的数组。接下来就可以对这个节点创建vnode了。

### VNode的创建

经过规范化处理之后，就要进行VNode的创建了。主要是下面的这段代码：

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
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
```

如果tag不是字符串类型的，那么直接调用createComponent创建组件的VNode，我们的实例传入的tag是`'div'`，一个字符串，所以我不会通过createComponent创建，这个方法很重要，会在后面学习。

如果tag是字符串类型的，那么首先判断这个字符串是否是内置字段，如果是内置字段，则直接创建一个普通的VNode。否则如果是已经注册组件，那么直接调用craeteComponent创建一个VNode。再否则就直接创建一个未知标签的VNode，我们的实例创建的就是未知标签的VNode。

### 小结

createElement函数中，会根据参数的不同情况创建不同的VNode，但最终返回的都将是VNode类型元素。接下来就要把这个生成的VNode交给`_update`方法，创建出一个真正的DOM，并且渲染出来。

## update

在收到`_render`函数返回的VNode类型的对象之后，就要执行`_update ` 了：

(文件位置/src/core/instance/lifecycle.js)

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

可以看到，`_update`中主要就是执行`__patch__()` ， 在执行patch之前，先执行setActiveInstance

```js
export function setActiveInstance(vm: Component) {
  const prevActiveInstance = activeInstance
  activeInstance = vm
  return () => {
    activeInstance = prevActiveInstance
  }
}
```

通过setActiveInstance方法，修改一个存在于lifecycle中的全局变量activeInstance，这个全局变量用来保存正在渲染的实例。

然后就是`_update` 的核心`__patch__` ，patch在不同平台上的定义是不同的，比如在web平台：

(文件位置：/src/platforms/web/runtime/index.js)

```js
// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

而这个patch是在`/src/platforms/web/runtime/pathc.js` 中定义的：

```js
/* @flow */

import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// the directive module should be applied last, after all
// built-in modules have been applied.
const modules = platformModules.concat(baseModules)

export const patch: Function = createPatchFunction({ nodeOps, modules })
```

可以看到，patch其实是createPatchFunction返回的一个函数，在这里传入了nodeOpts和modules参数，nodeOps是封装的一系列DOM操作，而modules则是一些模块的钩子函数的实现。下面来看下createPatchFunction的实现：

(文件位置：/src/core/vdom/patch.js)

```js
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ...
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

这个函数的内部定义了一些辅助方法（省略号部分），最终返回了一个patch方法，这个patch就是`vm._update`里调用的那个`__patch__`。

```
在介绍 patch 的方法实现之前，我们可以思考一下为何 Vue.js 源码绕了这么一大圈，把相关代码分散到各个目录。因为前面介绍过，patch 是平台相关的，在 Web 和 Weex 环境，它们把虚拟 DOM 映射到 “平台 DOM” 的方法是不同的，并且对 “DOM” 包括的属性模块创建和更新也不尽相同。因此每个平台都有各自的 nodeOps 和 modules，它们的代码需要托管在 src/platforms 这个大目录下。

而不同平台的 patch 的主要逻辑部分是相同的，所以这部分公共的部分托管在 core 这个大目录下。差异化部分只需要通过参数来区别，这里用到了一个函数柯里化的技巧，通过 createPatchFunction 把差异化参数提前固化，这样不用每次调用 patch 的时候都传递 nodeOps 和 modules 了，这种编程技巧也非常值得学习。		
																	————Vue.js技术揭秘
```

传入的参数分别为：

- oldVnode ，表示旧的Vnode节点，也可以不存在或者是DOM节点。
- vnode，表示render函数生成的vnode
- hydrating，表示是否启用服务端渲染
- removeOnly，是给 `transition-group` 用的，之后会介绍。

结合我们的例子，传入的参数为：

- vm.$el， 表示template中id为app的对象
- vnode，表示render函数返回的Vnode
- hydrating，false，不启用服务端渲染
- removeOnly，false。

根据传入的参数来看，vm.$el为真实节点，所以isRealElement为true，所以通过emptyNodeAt(oldVnode)创建了一个空的节点，赋值给oldVnode。

接下来又调用createElm方法，这个方法也是在patch.js文件中声明的。

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
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // This vnode was used in a previous render!
      // now it's used as a new node, overwriting its elm would cause
      // potential patch errors down the road when it's used as an insertion
      // reference node. Instead, we clone the node on-demand before creating
      // associated DOM element for it.
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      if (process.env.NODE_ENV !== 'production') {
        if (data && data.pre) {
          creatingElmInVPre++
        }
        if (isUnknownElement(vnode, creatingElmInVPre)) {
          warn(
            'Unknown custom element: <' + tag + '> - did you ' +
            'register the component correctly? For recursive components, ' +
            'make sure to provide the "name" option.',
            vnode.context
          )
        }
      }

      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      /* istanbul ignore if */
      if (__WEEX__) {
        // in Weex, the default insertion order is parent-first.
        // List items can be optimized to use children-first insertion
        // with append="tree".
        const appendAsTree = isDef(data) && isTrue(data.appendAsTree)
        if (!appendAsTree) {
          if (isDef(data)) {
            invokeCreateHooks(vnode, insertedVnodeQueue)
          }
          insert(parentElm, vnode.elm, refElm)
        }
        createChildren(vnode, children, insertedVnodeQueue)
        if (appendAsTree) {
          if (isDef(data)) {
            invokeCreateHooks(vnode, insertedVnodeQueue)
          }
          insert(parentElm, vnode.elm, refElm)
        }
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

createElm的作用是将Vnode节点转化为DOM节点，并且插入到DOM上去。

在createElm中，首先会去通过createComponent创建组件，这个会在之后学习，在当前这种情况下，这个方法返回的是false。

接下来会对tag进行检测，在开发环境下检测tag的合法性。然后就是调用nodeOps上的createElement创建一个占位符元素。

```js
vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
```

createElementNS用来创建一个具有指定命名空间URI和限定名称的元素，如果要创建不指定命名空间的元素，则使用createElement。

接下来就是创建节点，并且插入到DOM上：

```js
createChildren(vnode, children, insertedVnodeQueue)
if (isDef(data)) {
    invokeCreateHooks(vnode, insertedVnodeQueue)
}
insert(parentElm, vnode.elm, refElm)
```

首先是创建子节点：

```js
  function createChildren (vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
      if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(children)
      }
      for (let i = 0; i < children.length; ++i) {
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
      }
    } else if (isPrimitive(vnode.text)) {
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
    }
  }
```

根据children的类型，进行不同的处理，如果children不是数组并且是原始类型，那么直接调用appendChild创建元素，加到之前创建的占位符元素中。而如果是数组，那么遍历整个数组，对数组中的每个元素递归调用createElm。

之后，会执行每个create钩子，并且把vnodepush到insertedVnodeQueue中：

```js
  function invokeCreateHooks (vnode, insertedVnodeQueue) {
    for (let i = 0; i < cbs.create.length; ++i) {
      cbs.create[i](emptyNode, vnode)
    }
    i = vnode.data.hook // Reuse variable
    if (isDef(i)) {
      if (isDef(i.create)) i.create(emptyNode, vnode)
      if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
    }
  }
```

最后调用insert方法，插入到DOM上：

```js
insert(parentElm, vnode.elm, refElm)
function insert (parent, elm, ref) {
    if (isDef(parent)) {
        if (isDef(ref)) {
            if (nodeOps.parentNode(ref) === parent) {
                nodeOps.insertBefore(parent, elm, ref)
            }
        } else {
            nodeOps.appendChild(parent, elm)
        }
    }
}
```

由于createElm是递归执行的，子元素会先调用insert方法，所以整个vnode树节点的插入顺序是先子后父。insert中，调用nodeOps.inserBefore和nodeOps.appendChild方法插入节点：

(文件位置：/src/platforms/web/runtime/node-Ops.js)

```js
export function appendChild (node: Node, child: Node) {
  node.appendChild(child)
}
export function insertBefore (parentNode: Node, newNode: Node, referenceNode: Node) {
  parentNode.insertBefore(newNode, referenceNode)
}
```

这些底层的插入还是调用的原生的DOM方法，最终成功的创建了一颗DOM树。因为最初进入createElm时，传入的parentElm就是oldVnode.elm的父元素，在这个例子也就是id为app的父元素，所以最终会插入到body上去。

在patch之后回到update中，调用setActiveInstance返回的restoreActiveInstance方法，将activeInstance修改回之前的实例。

然后回到mountComponent方法，执行mounted钩子：

```js
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
```

最后再返回一个Vue的实例，到这里挂载的过程就结束了。



## 总结

到这里就对Vue的初始化过程有了简单的了解，大致是下面的流程

1. init
2. $mount, compileToFunction
3. $mount,mountComponent,new Watcher
4. _render，createElement，new Vnode
5. _update,`__patch__`, createPatchFunction, createElm

当然了，这只是整个简单例子的初始化过程，我们在平时开发中会用到很多的组件，Vue的另一个核心思想就是组件化，将在之后学习中学习组件化的源码。