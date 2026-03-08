# keep-alive

> `<keep-alive> `包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。`<keep-alive>` 是一个抽象组件：它自身不会渲染一个 DOM 元素，也不会出现在父组件链中。

> 当组件在 `<keep-alive> `内被切换，它的 `activated` 和 `deactivated` 这两个生命周期钩子函数将会被对应执行。

在官方文档种可以看到这样的描述，我们来一步一步解释一下作者是怎么实现`keep-alive`的。

1. `keep-alive`是如何缓存组件实例，不销毁的
2. `keep-alive`是如何做到不渲染自身的
3. `keep-alive`是怎么去执行钩子函数的

### `keep-alive`是如何缓存组件实例，不销毁的
首先，在created中，创建了一个cache和keys，cathe缓存VNode,即虚拟DOM，keys缓存VNode对应的键。

  	created () {
		this.cache = Object.create(null)
		this.keys = []
  	}

然后我们看一下render函数

	render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
	// 获取keep-alive包裹的第一个组件

    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) { // 组件参数存在
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if ( // 条件判断
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) { // 如果cathe中存在组件实例
        vnode.componentInstance = cache[key].componentInstance // 获取实例
        // make current key freshest
        remove(keys, key)
        keys.push(key) // 更新key
      } else {
        cache[key] = vnode
        keys.push(key) 
		// 将VNode和key缓存
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) { // 如果设置了max并且超过限制则销毁组件
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
	  // 防止组件执行生命钩子，即不再渲染组件，销毁组件
    }
    return vnode || (slot && slot[0])
  	}

### `keep-alive`是如何做到不渲染自身的
	
	export default {
  		name: 'keep-alive',
  		abstract: true,
		···
	}
在vue设置`abstract`为`true`即不渲染自身

### `keep-alive`是怎么去执行钩子函数的
设置了`keepAlive`为`true`后，即不执行原有的钩子函数
在patch的阶段，最后会执行invokeInsertHook函数，而这个函数就是去调用组件实例（VNode）自身的`insert`钩子：

	// src/core/vdom/patch.js
  	function invokeInsertHook (vnode, queue, initial) {
    	if (isTrue(initial) && isDef(vnode.parent)) {
      		vnode.parent.data.pendingInsert = queue
    	} else {
      		for (let i = 0; i < queue.length; ++i) {
        		queue[i].data.hook.insert(queue[i])  // 调用VNode自身的insert钩子函数
      		}
    	}
  	}
复制代码再看`insert`钩子：

	// src/core/vdom/create-component.js
	const componentVNodeHooks = {
  		// init()
  		insert (vnode: MountedComponentVNode) {
    	const { context, componentInstance } = vnode
    	if (!componentInstance._isMounted) {
      		componentInstance._isMounted = true
      		callHook(componentInstance, 'mounted')
    	}
   	 	if (vnode.data.keepAlive) {
      		if (context._isMounted) {
        		queueActivatedComponent(componentInstance)
      		} else {
        		activateChildComponent(componentInstance, true /* direct */)
      		}
    	}
  		// ...
	}
复制代码在这个钩子里面，调用了`activateChildComponent`函数递归地去执行所有子组件的activated钩子函数：

	// src/core/instance/lifecycle.js
	export function activateChildComponent (vm: Component, direct?: boolean) {
  		if (direct) {
    		vm._directInactive = false
    		if (isInInactiveTree(vm)) {
      			return
    		}
  		} else if (vm._directInactive) {
    	return
  		}
  		if (vm._inactive || vm._inactive === null) {
    		vm._inactive = false
    		for (let i = 0; i < vm.$children.length; i++) {
      			activateChildComponent(vm.$children[i])
    		}
    		callHook(vm, 'activated')
  		}
	}

复制代码相反地，`deactivated`钩子函数也是一样的原理，在组件实例（VNode）的`destroy`钩子函数中调用`deactivateChildComponent`函数。
