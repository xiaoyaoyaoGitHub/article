# `Vue`中`runtime-core`之`h`函数从创建到挂载的简单实现

> 在`vue`中的源码中可以看到，`h`函数的参数有很多种，以下来自[vue源码](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/h.ts)中`h`函数的使用方式

```js
/*
// type only
h('div')
// type + props
h('div', {})
// type + omit props + children
// Omit props does NOT support named slots
h('div', []) // array
h('div', 'foo') // text
h('div', h('br')) // vnode
h(Component, () => {}) // default slot
// type + props + children
h('div', {}, []) // array
h('div', {}, 'foo') // text
h('div', {}, h('br')) // vnode
h(Component, {}, () => {}) // default slot
h(Component, {}, {}) // named slots
// named slots without props requires explicit `null` to avoid ambiguity
h(Component, null, {})
```

> 我们在[官网](https://vue3js.cn/docs/zh/api/global-api.html#类型声明)中可以查看到，`h`函数包含3个参数

- `type`：标识标签名或者组件类型；
- `props`：为组件或者标签的属性，但是这个参数为可选值，用户可以不传入，这就意味着第二个参数有可能是描述子元素的；
- `children`：标识子元素，有可能是一段纯文本，或者描述子元素的vnode。

> 我们今天的实现不考虑组件的解析，我们通过自己实现`h`方法解析下面一段代码

```js
h('div', ['fist-page',h('h1', { style: {color: 'yellow' }}, 'wangly')])
```

#### `h`函数的实现

> `h`函数的关键作用是创建虚拟节点，这里我们需要利用到之前说到的创建虚拟节点方法

```js
export function h(type, propsOrChildren, children) {
	const argLength = arguments.length;
	if (argLength == 2) {
		if (isObject(propsOrChildren) && !isArray(propsOrChildren)) {
			if (isVnode(propsOrChildren)) {
				//表示没有属性值
				return createVNode(type, null, [propsOrChildren]);
			} else {
				return createVNode(type, propsOrChildren); // 表示没有子元素
			}
		} else {
			return createVNode(type, null, propsOrChildren);
		}
	} else if (argLength === 3) {
		if (isVnode(children)) {
			children = [children];
		}
		return createVNode(type, propsOrChildren, children);
	} else if (argLength > 3) {
		return createVNode(
			type,
			propsOrChildren,
			Array.from(arguments).slice(2)
		);
	}
}
```

> 创建完成后，我们可以看到完整的vnode结构

```json
{
  _v_isVnode: true,
  type: 'div',
  shapeFlag: 17,
  el: 'div',
  children:[
    "first page",
    {
      _v_isVnode: true,
      type: 'h1',
      shapeFlag: 17,
      el: 'h1',
      props:{
        style:{
          color:'yellow'
        }
      },
      children:['wangly']
    }
  ]
}
```

#### 遍历虚拟节点

> 我们执行的流程是，先判断当前虚拟节点的类型，参考之前虚拟节点的创建一文中`shapeFlag`，如果是文本则直接插入，如果是元素类型，则根据`type`字段创建节点，循环`children`字段依次插入到元素中

- 判断类型（这里先不考虑更新）

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
    processElement(n1, n2, container);
  } else { // 如果是文本内容则直接插入到节点中
    container.textContent = n2
  }
}
```

- 创建节点类型

```js
	/**
	 * 创建节点
	 * @param n1
	 * @param n2
	 * @param container
	 */
	function processElement(n1, n2, container) {
		if (n1 === null) {
			mountElement(n2, container);
		} else {
			// 更新 diff 算法
		}
	}
```

```js
	/**
	 * 挂载节点
	 * @param vnode
	 * @param container
	 */
	function mountElement(vnode, container) {
		const { type, props, children, shapeFlag } = vnode || {};
		let el = (vnode.el = hostCreateElement(type));
		if (props) {
			for (let key in props) { // 循环设置属性，
				hostPatchProp(el, key, null, props[key]);
			}
		}
		if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
			// 如果子元素是数组
			mountChildren(children, el);
		} else {
			hostSetElementText(el, children);
		}
		hostInsert(el, container);
	}
```

- 循环子节点

```js
	function mountChildren(children, container) {
		for (let i = 0; i < children.length; i++) {
			patch(null, children[i], container); // 递归创建
		}
	}
```

> 注：里面涉及到元素添加或者属性设置参照[runtime-dom]()一文

简单学习，后续补充

