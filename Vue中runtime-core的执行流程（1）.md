# `Vue`中`runtime-core`的执行流程之创建虚拟节点

> 在上一篇`runtime-core`中我们知道在调用`createApp`方法实际上是调用的是`@wangly/runtime-core`中提供的方法`createRenderer`的执行结果中的`createApp`

#### `createRenderer`的实现

> 从我们的调用情况来看，其实改方法的执行结果会返回一个对象，对象里面提供了`createApp`方法

```js
/**
 * render函数
 * @param vnode     虚拟节点
 * @param container 挂载跟节点
 */
const render = (vnode, container) => {

};

export function createRenderer(renderOptions) {
	return {
		createApp: createAppApi(render),
		render,
	};
}
```

> 用户将创建的组件作为参数传入到`createRenderer().createAPP`方法中，在该方法里我们将组件转化为虚拟节点，并由`render`方法转化成真实节点挂载到容器中，且该方法提供了`mount`方法，具体实现如下：

```js
export function createAppApi(render) {
	return (rootComponent, rootProp) => {
		const app = {
			// 为了稍后组件挂载之前可以先校验组件是否有render函数或者模板
			_component: rootComponent,
			_props: rootProp,
			_container: null,
			mount(container) {
				// 根据用户传入的属性创建一个虚拟节点
				const vNode = createVNode(rootComponent, rootProp);
				console.log(vNode);
				// 更新节点的_container
				app._container = container;
				// 将虚拟节点转化成真实节点,插入到对应的容器中
				render( vNode, container);
			},
			render,
		};
		return app;
	};
}

```

#### `createVNode`的实现，虚拟节点的创建

> 首先我们需要定义下我们节点的类型总共有哪些，下面是`vue`提供给我们的类型

```js
export const enum ShapeFlags {
	ELEMENT = 1, // 标识是一个元素
	FUNCTIONAL_COMPONENT = 1 << 1, // 函数组件
	STATEFUL_COMPONENT = 1 << 2, // 带状态的组件
	TEXT_CHILDREN = 1 << 3, // 这个组件的孩子是文本
	ARRAY_CHILDREN = 1 << 4, // 孩子是数组
	SLOTS_CHILDREN = 1 << 5, // 插槽孩子
	TELEPORT = 1 << 6, // 传送门
	SUSPENSE = 1 << 7, // 实现异步组件等待
	COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8, // 是否需要keep-alive
	COMPONENT_KEPT_ALIVE = 1 << 9, // 组件的keep-alive
	COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT,
}
```

> `<<`是左移移位运算符，`|`表示按位或，`&`表示按位与。`1<<1`标识左移一位即为`10`,`1<<2`表示左移两位即为`100`，依次类推。计算出来的值为二进制

`createVNode`

```js
/**
 *
 * @param type     原始组件
 * @param props    属性
 * @param children 子元素
 */
export function createVNode(type, props, children = null) {
	// 虚拟节点属性描述内容 type props key children _v_isVnode
	const shapeFlag = isString(type) //直接是节点类型 如 div h1
		? ShapeFlags.ELEMENT
		: isObject(type) // 是对象类型例如 {setup:()=>{}}
		? ShapeFlags.STATEFUL_COMPONENT
		: 0;
	const vnode = {
		_v_isVnode: true,
		type, // 对组件而言就是一个对象
		props,
		children,
		el: null, // 对应真实节点
		component: null, // 组件实例
		shapeFlag, // 类型
	};
	// 加入子元素的类型判断
	normalizeChildren(vnode, children);

	return vnode;
}
/**
 * 子元素类型判断
 * @param vnode    虚拟节点
 * @param children 子元素
 */
function normalizeChildren(vnode, children) {
	let shapeFlag = 0;
	if (children) {
		shapeFlag = isArray(children)
			? ShapeFlags.ARRAY_CHILDREN //子元素为数组
			: ShapeFlags.TEXT_CHILDREN; // 子元素为文本
	}

	vnode.shapeFlag = shapeFlag;
}

```

> 当我们创建完虚拟节点后，作为传入给`render`函数，后续我们解析`render`函数的执行过程

简单学习，后续补充