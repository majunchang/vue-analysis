## Vue源码分析

### vue的源码目录设计

```js
src
├── compiler        # 编译相关 
├── core            # 核心代码 
├── platforms       # 不同平台的支持
├── server          # 服务端渲染
├── sfc             # .vue 文件解析
├── shared          # 共享代码
```



#### compiler 

compiler 目录包含 Vue.js 所有编译相关的代码。它包括把模板解析成 ast 语法树，ast 语法树优化，代码生成等功能。

编译的工作可以在构建时做（借助 webpack、vue-loader 等辅助插件）；也可以在运行时做，使用包含构建功能的 Vue.js。显然，编译是一项耗性能的工作，所以更推荐前者——离线编译。

#### core

core 目录包含了 Vue.js 的核心代码，包括内置组件、全局 API 封装，Vue 实例化、观察者、虚拟 DOM、工具函数等等。

#### platforms

Vue.js 是一个跨平台的 MVVM 框架，它可以跑在 web 上，也可以配合 weex 跑在 native 客户端上。platform 是 Vue.js 的入口，2 个目录代表 2 个主要入口，分别打包成运行在 web 上和 weex 上的 Vue.js。

#### sfc

通常我们开发 Vue.js 都会借助 webpack 构建， 然后通过 .vue 单文件来编写组件。

这个目录下的代码逻辑会把 .vue 文件内容解析成一个 JavaScript 的对象。

#### shared

Vue.js 会定义一些工具方法，这里定义的工具方法都是会被浏览器端的 Vue.js 和服务端的 Vue.js 所共享的。

### new Vue的背后 

> new 关键字在js语言中代表实例化一个对象，Vue实际上是一个类，类在js中是用Function来实现的，来看一下源码，在`src/core/instance/index.js` 中

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

Vue只能 通过new关键字进行初始化，然后调用this.init方法，src/core/instance/init.js

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

init中主要干了这几件事情：

1. 合并配置。
2. 初始化声明周期
3. 初始化事件中心，
4. 初始化渲染
5. 初始化data，props，computed，watcher等等。

### template的编译

`src/platform/web/entry-runtime-with-compiler.js` 文件中定义：mount方法

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
			 /*将template编译成render函数，这里会有render以及staticRenderFns两个返回，这是vue的编译时优化，static静态不需要在VNode更新时进行patch，优化性能*/
      const { render, staticRenderFns } = compileToFunctions(template, {
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



**解析**：

1. 缓存原型上的``$mount``方法,再重新定义``Vue.prototype.$mount``
2. 对el做了限制，vue不能挂载到body，html这样的根节点上。
3. options中没有定义render方法的时候，template存在的时候 去template，template不存在的时候 去el的outerHtml
4. vue的2.0版本中所有的渲染都会经过render函数



Template 会被编译为AST（抽象语法树），是源代码的抽象语法结构的树状表现形式。AST经过genereat函数得到render函数，render的返回值是vNode，Vnode是vue的虚拟dom节点



### Virtual Dom

> Virtual DOM 这个概念相信大部分人都不会陌生，它产生的前提是浏览器中的 DOM 是很“昂贵"的，为了更直观的感受，我们可以简单的把一个简单的 div 元素的属性都打印出来，如图所示：

![images](https://ustbhuangyi.github.io/vue-analysis/assets/dom.png)

Virtual Dom 本质来说就是用一原生的js对象去描述一个DOM节点，相比于创建一个Dom的代价要小很多。

Vue.js将DOM抽象成一个以JavaScript对象为节点的虚拟DOM树，以VNode节点模拟真实DOM，可以对这颗抽象树进行创建节点、删除节点以及修改节点等操作，在这过程中都不需要操作真实DOM，只需要操作JavaScript对象后只对差异修改，相对于整块的innerHTML的粗暴式修改，大大提升了性能。修改以后经过diff算法得出一些需要修改的最小单位，再将这些小单位的视图进行更新。这样做减少了很多不需要的DOM操作，大大提高了性能。

在 Vue.js 中，Virtual DOM 是用 `VNode` 这么一个 Class 去描述，它是定义在 `src/core/vdom/vnode.js` 中的。

```js
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  functionalContext: Component | void; // only for functional component root nodes
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions
  ) {
    /*当前节点的标签名*/
    this.tag = tag
    /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.data = data
    /*当前节点的子节点，是一个数组*/
    this.children = children
    /*当前节点的文本*/
    this.text = text
    /*当前虚拟节点对应的真实dom节点*/
    this.elm = elm
    /*当前节点的名字空间*/
    this.ns = undefined
    /*编译作用域*/
    this.context = context
    /*函数化组件作用域*/
    this.functionalContext = undefined
    /*节点的key属性，被当作节点的标志，用以优化*/
    this.key = data && data.key
    /*组件的option选项*/
    this.componentOptions = componentOptions
    /*当前节点对应的组件的实例*/
    this.componentInstance = undefined
    /*当前节点的父节点*/
    this.parent = undefined
    /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.raw = false
    /*静态节点标志*/
    this.isStatic = false
    /*是否作为根节点插入*/
    this.isRootInsert = true
    /*是否为注释节点*/
    this.isComment = false
    /*是否为克隆节点*/
    this.isCloned = false
    /*是否有v-once指令*/
    this.isOnce = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```

#### 生成一个Vnode的方法

##### 空Vnode节点

```js
/*创建一个空VNode节点*/
export const createEmptyVNode = () => {
  const node = new VNode()
  node.text = ''
  node.isComment = true
  return node
}
```

##### 文本节点

```js
/*创建一个文本节点*/
export function createTextVNode (val: string | number) {
  return new VNode(undefined, undefined, undefined, String(val))
}
```

##### 组件节点

```js
// plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  /*Github:https://github.com/answershuto*/
  /*如果在该阶段Ctor依然不是一个构造函数或者是一个异步组件工厂则直接返回*/
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  /*处理异步组件*/
  if (isUndef(Ctor.cid)) {
    Ctor = resolveAsyncComponent(Ctor, baseCtor, context)
    if (Ctor === undefined) {
      // return nothing if this is indeed an async component
      // wait for the callback to trigger parent update.
      /*如果这是一个异步组件则会不会返回任何东西（undifiened），直接return掉，等待回调函数去触发父组件更新。s*/
      return
    }
  }

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  data = data || {}

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
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners
    data = {}
  }

  // merge component management hooks onto the placeholder node
  mergeHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children }
  )
  return vnode
}
```



#### 举一个🌰

我现在有这么一个template

```vue
<div class="test">
    <span class="demo">hello,VNode</span>
</div>
```

经过转化之后变成了这样的一个VNode树

```javascript
{
    tag: 'div'
    data: {
        class: 'test'
    },
    children: [
        {
            tag: 'span',
            data: {
                class: 'demo'
            }
            text: 'hello,VNode'
        }
    ]
}

```



#### 总结

Vue中对于虚拟dom的定义还是略微复杂一些，里面包含了很多vue的特性，实际上vue借助了一个开源库[snabbdom](https://github.com/snabbdom/snabbdom)的实现。然后加入了一些vue特色的东西，Virtual DOM 除了它的数据结构的定义，映射到真实的 DOM 实际上要经历 VNode 的 create、diff、patch 等过程。

### Diff算法

> 首先介绍一下 update方法

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    /*如果已经该组件已经挂载过了则代表进入这个步骤是个更新的过程，触发beforeUpdate钩子*/
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    /*基于后端渲染Vue.prototype.__patch__被用来作为一个入口*/
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(
        vm.$el, vnode, hydrating, false /* removeOnly */,
        vm.$options._parentElm,
        vm.$options._refElm
      )
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // update __vue__ reference
    /*更新新的实例对象的__vue__*/
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

**Diff的过程实际上就是在内部将Vnode对象与之前旧的Vnode对象进行__patch__**

##### Patch

patch将新老VNode节点进行比对，然后将根据两者的比较结果进行最小单位地修改视图，而不是将整个视图根据新的VNode重绘。patch的核心在于diff算法，这套算法可以高效地比较virtual DOM的变更，得出变化以修改视图。

> Patch 的核心算法在于 通过同层树节点进行比较而非对树进行逐层搜索遍历的方式，时间复杂度是o(n),个人认为是一种广度优先的搜索遍历

[Vue原理解析之Virtual Dom](https://segmentfault.com/a/1190000008291645)

[你了解vue的diff算法吗](https://juejin.im/post/5ad6182df265da23906c8627)


##### react的diff算法的🌰

 假如我们有一个可排序的列表 

```html
<ul>
  <li>1</li>
  <li>2</li>
  <li>3</li>
</ul>
```



我们用下面的数组来表示ul标签的children

```js
[
  h('li', null, 1),
  h('li', null, 2),
  h('li', null, 3)
]
```



接着  由于数据的变化导致了列表的顺序发生了变化

```js
[
  h('li', null, 3),
  h('li', null, 1),
  h('li', null, 2)
]
```



```js
function patchChildren(
  prevChildFlags,
  nextChildFlags,
  prevChildren,
  nextChildren,
  container
) {
  switch (prevChildFlags) {
    // 省略...

    // 旧的 children 中有多个子节点
    default:
      switch (nextChildFlags) {
        case ChildrenFlags.SINGLE_VNODE:
          // 省略...
        case ChildrenFlags.NO_CHILDREN:
          // 省略...
        default:
          for (let i = 0; i < prevChildren.length; i++) {
            patch(prevChildren[i], nextChildren[i], container)
          }
          break
      }
      break
  }
}
```



- 这段代码的弊端在于当两者数组长度不一致的时候 就会出现计算偏差
- v-for的key  承载了一个什么样的作用，这段代码并没有表现出来
- 增加key为唯一标识的时候，我们需要进行两次遍历循环，那么如何去掉已经销毁的节点和新增新节点呢

**算法解析**

1. 判断需要移动的节点  寻找在旧children中所遇到的最大索引值，在此基础上存在索引值小于最大索引值的节点，意味着该节点需要被移动（实际上这是react中的算法）



```js
let lastIndex = 0
for (let i = 0; i < nextChildren.length; i++) {
  const nextVNode = nextChildren[i]
  let j = 0,
    find = false
  for (j; j < prevChildren.length; j++) {
    const prevVNode = prevChildren[j]
    if (nextVNode.key === prevVNode.key) {
      find = true
      patch(prevVNode, nextVNode, container)
      if (j < lastIndex) {
        // 需要移动
        const refNode = nextChildren[i - 1].el.nextSibling
        container.insertBefore(prevVNode.el, refNode)
        break
      } else {
        // 更新 lastIndex
        lastIndex = j
      }
    }
  }
  if (!find) {
    // 挂载新节点
    // 找到 refNode
    const refNode =
      i - 1 < 0
        ? prevChildren[0].el
        : nextChildren[i - 1].el.nextSibling
    mount(nextVNode, container, false, refNode)
  }
}

// 移除已经不存在的节点
// 遍历旧的节点
for (let i = 0; i < prevChildren.length; i++) {
  const prevVNode = prevChildren[i]
  // 拿着旧 VNode 去新 children 中寻找相同的节点
  const has = nextChildren.find(
    nextVNode => nextVNode.key === prevVNode.key
  )
  if (!has) {
    // 如果没有找到相同的节点，则移除
    container.removeChild(prevVNode.el)
  }
}
```

#### 算法优化 -- vue的双端算法 

参考双指针运算

```js
function patchChildren(
  prevChildFlags,
  nextChildFlags,
  prevChildren,
  nextChildren,
  container
) {
  switch (prevChildFlags) {
    // 旧的 children 是单个子节点，会执行该 case 语句块
    case ChildrenFlags.SINGLE_VNODE:
      switch (nextChildFlags) {
        case ChildrenFlags.SINGLE_VNODE:
          // 新的 children 也是单个子节点时，会执行该 case 语句块
          patch(prevChildren, nextChildren, container)
          break
        case ChildrenFlags.NO_CHILDREN:
          // 新的 children 中没有子节点时，会执行该 case 语句块
          container.removeChild(prevChildren.el)
          break
        default:
          // 但新的 children 中有多个子节点时，会执行该 case 语句块
          container.removeChild(prevChildren.el)
          for (let i = 0; i < nextChildren.length; i++) {
            mount(nextChildren[i], container)
          }
          break
      }
      break
    // 旧的 children 中没有子节点时，会执行该 case 语句块
    case ChildrenFlags.NO_CHILDREN:
      switch (nextChildFlags) {
        case ChildrenFlags.SINGLE_VNODE:
          // 新的 children 是单个子节点时，会执行该 case 语句块
          mount(nextChildren, container)
          break
        case ChildrenFlags.NO_CHILDREN:
          // 新的 children 中没有子节点时，会执行该 case 语句块
          break
        default:
          // 但新的 children 中有多个子节点时，会执行该 case 语句块
          for (let i = 0; i < nextChildren.length; i++) {
            mount(nextChildren[i], container)
          }
          break
      }
      break
    // 旧的 children 中有多个子节点时，会执行该 case 语句块
    default:
      switch (nextChildFlags) {
        case ChildrenFlags.SINGLE_VNODE:
          for (let i = 0; i < prevChildren.length; i++) {
            container.removeChild(prevChildren[i].el)
          }
          mount(nextChildren, container)
          break
        case ChildrenFlags.NO_CHILDREN:
          for (let i = 0; i < prevChildren.length; i++) {
            container.removeChild(prevChildren[i].el)
          }
          break
        default:
          // 当新的 children 中有多个子节点时，会执行该 case 语句块
          let oldStartIdx = 0
          let oldEndIdx = prevChildren.length - 1
          let newStartIdx = 0
          let newEndIdx = nextChildren.length - 1
          let oldStartVNode = prevChildren[oldStartIdx]
          let oldEndVNode = prevChildren[oldEndIdx]
          let newStartVNode = nextChildren[newStartIdx]
          let newEndVNode = nextChildren[newEndIdx]
          while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
            if (!oldStartVNode) {
              oldStartVNode = prevChildren[++oldStartIdx]
            } else if (!oldEndVNode) {
              oldEndVNode = prevChildren[--oldEndIdx]
            } else if (oldStartVNode.key === newStartVNode.key) {
              patch(oldStartVNode, newStartVNode, container)
              oldStartVNode = prevChildren[++oldStartIdx]
              newStartVNode = nextChildren[++newStartIdx]
            } else if (oldEndVNode.key === newEndVNode.key) {
              patch(oldEndVNode, newEndVNode, container)
              oldEndVNode = prevChildren[--oldEndIdx]
              newEndVNode = nextChildren[--newEndIdx]
            } else if (oldStartVNode.key === newEndVNode.key) {
              patch(oldStartVNode, newEndVNode, container)
              container.insertBefore(
                oldStartVNode.el,
                oldEndVNode.el.nextSibling
              )
              oldStartVNode = prevChildren[++oldStartIdx]
              newEndVNode = nextChildren[--newEndIdx]
            } else if (oldEndVNode.key === newStartVNode.key) {
              patch(oldEndVNode, newStartVNode, container)
              container.insertBefore(oldEndVNode.el, oldStartVNode.el)
              oldEndVNode = prevChildren[--oldEndIdx]
              newStartVNode = nextChildren[++newStartIdx]
            } else {
              const idxInOld = prevChildren.findIndex(
                node => node.key === newStartVNode.key
              )
              if (idxInOld >= 0) {
                const vnodeToMove = prevChildren[idxInOld]
                patch(vnodeToMove, newStartVNode, container)
                prevChildren[idxInOld] = undefined
                container.insertBefore(vnodeToMove.el, oldStartVNode.el)
              } else {
                // 新节点
                mount(newStartVNode, container, false, oldStartVNode.el)
              }
              newStartVNode = nextChildren[++newStartIdx]
            }
          }
          if (oldEndIdx < oldStartIdx) {
            // 添加新节点
            for (let i = newStartIdx; i <= newEndIdx; i++) {
              mount(nextChildren[i], container, false, oldStartVNode.el)
            }
          } else if (newEndIdx < newStartIdx) {
            // 移除操作
            for (let i = oldStartIdx; i <= oldEndIdx; i++) {
              container.removeChild(prevChildren[i].el)
            }
          }
          break
      }
      break
  }
}
```



### nextTick

>  我们知道Vue.js是默认异步执行DOM更新，当异步执行update的时候，会调用queueWatcher函数

```js
/*将一个观察者对象push进观察者队列，在队列中已经存在相同的id则该观察者对象将被跳过，除非它是在队列被刷新时推送*/
export function queueWatcher (watcher: Watcher) {
  /*获取watcher的id*/
  const id = watcher.id
  /*检验id是否存在，已经存在则直接跳过，不存在则标记哈希表has，用于下次检验*/
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      /*如果没有flush掉，直接push到队列中即可*/
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i >= 0 && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(Math.max(i, index) + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```

nextTick函数  

在 Vue 源码 2.5+ 后，`nextTick` 的实现单独有一个 JS 文件来维护它，它的源码并不多，总共也就 100 多行。接下来我们来看一下它的实现，在 `src/core/util/next-tick.js` 中：

```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

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

// Here we have async deferring wrappers using both microtasks and (macro) tasks.
// In < 2.4 we used microtasks everywhere, but there are some scenarios where
// microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using (macro) tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use microtask by default, but expose a way to force (macro) task when
// needed (e.g. in event handlers attached by v-on).
let microTimerFunc
let macroTimerFunc
let useMacroTask = false

// Determine (macro) task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// Determine microtask defer implementation.
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a (macro) task instead of a microtask.
 */
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
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
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```



`next-tick.js` 申明了 `microTimerFunc` 和 `macroTimerFunc` 2 个变量，它们分别对应的是 micro task 的函数和 macro task 的函数。对于 macro task 的实现，优先检测是否支持原生 `setImmediate`，这是一个高版本 IE 和 Edge 才支持的特性，不支持的话再去检测是否支持原生的 `MessageChannel`，如果也不支持的话就会降级为 `setTimeout 0`；而对于 micro task 的实现，则检测浏览器是否原生支持 Promise，不支持的话直接指向 macro task 的实现。

vue 2.5之前 几乎都是用微任务来模拟nodejs的 nexttick（）

vue2.5之后，默认使用微任务，在dom时间中，默认包裹一层函数来强制使用宏任务

- 先确定使用 macrotask 时用哪个API，优先级为：
  `setImmediate`-> `MessageChannel`->`setTimeout`

- 确定使用 microtask 时用哪个API，优先级为：`Promise`-> `macroTimerFunc`（和macrotask一致）

- 判断是否使用 macrotask ，是则调用`macroTimerFunc`，否则调用 `microTimerFunc`

- DOM事件默认会包裹一层函数来强制其使用 macrotask

------



##### 为什么默认使用微任务

根据[HTML Standard](https://link.jianshu.com/?t=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fhtml.spec.whatwg.org%2Fmultipage%2Fwebappapis.html%23event-loop-processing-model)，在每个 task 运行完以后，UI 都会重渲染（**知识点小结**那块有说到任务执行顺序以及什么时候渲染），那么在 microtask 中就完成数据更新，当前 task 结束就可以得到最新的 UI 了。反之如果新建一个 task 来做数据更新，那么渲染就会进行两次。







