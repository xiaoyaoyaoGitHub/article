# `Vue`中`runtime-core`执行流程之组件实例的产生

> 在上一篇文中我们知道，用户创建完虚拟节点后，就会去调用`render`进行挂载，然后我们给`render`传递参数，在虚拟节点上，我们其实给用户定义了类型`shapeFlag`，组件类型不同也需要执行不同的方法，今天我们主要看下组件实例的第一次创建流程

#### `render`函数

```js
/**
 * 判断是否挂载还是更新
 * @param n1         旧虚拟节点
 * @param n2         新虚拟节点
 * @param container  挂载跟节点
 */
function patch(n1, n2, container) {
	// 判断新虚拟节点类型
	const { shapeFlag } = n2;
	if (shapeFlag & ShapeFlags.ELEMENT) {
		//节点类型
	} else if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
		//组件类型
		processComponent(n1, n2, container);
	}
}

/**
 * render函数
 * @param vnode     虚拟节点
 * @param container 挂载跟节点
 */
const render = (vnode, container) => {
	// 对比新旧节点的不同
	patch(null, vnode, container);
};
```

> 在`processComponent`中，我们需要根据旧的虚拟节点增加判断组件是初次创建还是更新

```js
/**
 * 组件处理
 * @param n1
 * @param n2
 * @param container
 */
function processComponent(n1, n2, container) {
	if (!n1) {
		// 如果没有旧vode 则是挂载
		mountComponent(n2, container);
	} else {
		//否则是更新
		updateComponent(n1, n2, container);
	}
}
```

#### 创建实例

> 在确定组件是第一次创建实例后，我们将当前虚拟节点通过方法`createComponentInstance`创建实例，具体实现如下：

```js
let uid = 0;
/**
 * 创建对象实例
 * @param vnode 虚拟节点
 */
function createComponentInstance(vnode) {
	const instance = {
		uid: uid++,
		vnode: vnode, // 实例上的vnode就是我们处理过的vnode
		type: vnode.type, // 用户写的组件的内容
		props: {}, // props就是组件里用户声明过的
		attrs: {}, // 用户没用到的props 就会放到attrs中
		slots: {}, // 组件的是插槽
		setupState: {}, // setup的返回值
		proxy: null,
		emit: null, // 组件通信
		ctx: {}, // 上下文
		isMounted: false, // 组件是否挂载
		subTree: null, // 组件u敌营的渲染内容
		render: null,
	};
	instance.ctx = { _: instance };
	return instance;
}
```

#### 初始化实例参数

> 创建完成后，我们需要对实例上的参数进行初始化，例如`props`，`setup`等，我们通过创建方法`setupComponent`进行处理，具体实现如下

```js
/**
 * 处理实例参数
 * @param instance 实例
 */
export function setupComponent(instance) {
	const { props, children } = instance.vnode;
	// TODO  props是响应式
	instance.props = props;
	instance.slots = children;
  // 对整个instance.ctx做劫持，因为render函数需要用当前实例对象作为参数传入，所以对其做劫持，方便取值
  instance.proxy = new Proxy(instance.ctx, componentPublicInstance)
	// 处理setup 函数
	setupStatefulComponent(instance);
}
```

#### 根据`setup`返回结果不同进行不同处理


> `setup`函数需要有两个入入参，第一个为实例的`props`，第二个参数我们通过方法`createSetupContext`获得，具体实现如下

```js
/**
 * 处理setup 第二个参数
 * @param instance
 * @returns
 */
export function createSetupContext(instance) {
	return {
		slots: instance.slots,
		emit: instance.emit,
		attrs: instance.attrs,
		expose: () => {}, // 可以通过该方法暴露参数和方法,对外使用ref被调用
	};
}
```

> 我们需要对`setup`函数单独处理，如果`setup`返回结果是对象，我们将结果放在实例的`setupState`参数上，如果返回的是函数，则直接作为`render`函数而忽略其他`render`函数

```js
/**
 * 处理setup参数并执行setup方法
 * @param instance 实例
 */
export function setupStatefulComponent(instance) {
	const { type: Component } = instance;
	const { setup } = Component;
	console.log(setup);
	if (setup) {
		// 处理setup需要传入的两个参数
		const setupContext = createSetupContext(instance);
		const setupResult = setup(instance.props, setupContext);
		// 处理setup返回结果
		handleSetupResult(instance, setupResult);
	} else { //如果用户没有传入setup函数处理
		finishComponentSetup(instance);
	}
}
```

> 处理`setup`返回结果

```js
/**
 * 处理setup执行返回结果 处理setup返回结果,如果返回的是对象,则赋值为实例的state,如果是函数,则作为render函数触发
 * @param instance    当前组件实例对象
 * @param setupResult setup执行结果
 */
function handleSetupResult(instance, setupResult) {
	if (isObject(setupResult)) {
        instance.setupState = setupResult // 将返回结果保存到setupState中
	} else if (isFunction(setupResult)) {
		instance.render = setupResult;
	}
	finishComponentSetup(instance);
}
```

> 如果用户没有传入`setup`或者`setup`没有返回值处理，则会去整个组件中去寻找`render`函数

```js
/**
 * 处理如果没有setup函数或者无返回结果render处理方式
 * @param instance
 */
function finishComponentSetup(instance) {
	const Component = instance.type;
	if (Component && Component.render) {
		instance.render = Component.render;
	}
}
```

#### `mountComponent`方法的实现

```js
/**
 * 挂载组件
 * @param n2         新虚拟节点
 * @param container
 */
function mountComponent(n2, container) {
	// 创建组件实例, 并将实例挂载到自己本身上
	let instance = (n2.instance = createComponentInstance(n2));
	// 处理实例参数
	setupComponent(instance);
	// 执行render 方法 劫持组件
	setupRenderEffect(instance, container);
}
```

> 我们使用`effect`对整个组件做劫持

```js
/**
 * 挂载
 * @param instance
 * @param container
 */
function setupRenderEffect(instance, container) {
	effect(function componentEffect() {
		if (!instance.isMounted) { //根据isMounted判断是更新还是第一次挂载
			console.log("第一次挂载");
			// 在vue中的render函数中,有个参数,是对当前实例的拦截的proxy
			instance.render.call(instance.proxy, instance.proxy);
		} else {
			instance.isMounted = true;
			console.log("组件更新");
		}
	});
}
```

> 在这里面我们还没有对属性做处理，简单学习，后续补充

