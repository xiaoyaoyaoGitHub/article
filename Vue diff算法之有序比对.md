# `Vue diff`算法之有序比对

> 上一篇文章[Vue中runtime-core之h函数从创建到挂载的简单实现]()，我们了解了如果通过`h`函数创建虚拟节点并挂载到元素节点上。事实上，第一次挂载完成后，后续的节点变更我们都需要去比较变化，我们按照创建的新旧虚拟节点去比对，在此，我们复习下虚拟节点上挂载的主要属性

```js
vnode {
  	children, // 包含的子节点
    el,       // 挂载节点
    key,      // props中的key 用来判断节点是否一致  
    props,    // 属性
    shapeFlag, // 当前虚拟节点类型
    type       // 原始类型
}
```

> `shapeFlag`可以参考之前[ue中runtime-core的执行流程之创建虚拟节点]()一文，我们在此主要用到下面几种类型定义

```js
TEXT_CHILDREN = 1 << 3, // 这个组件的孩子是文本
ARRAY_CHILDREN = 1 << 4, // 孩子是数组
```

> 在我们实现比对功能之前，可以从3个维度考虑

- 新旧虚拟节点类型不同，`type`不同，则直接用新节点替换旧节点即可；
- 新旧虚拟节点类型相同，但是新虚拟节点子节点为文本，则直接替换旧节点即可；
- 新旧虚拟节点类型相同，旧子节点为文本，新子节点为数组，则直接替换旧节点即可；
- 新旧虚拟节点类型相同，新旧子节点都为数组，则我们需要进行遍历比较

#### 新旧节点类型不同

```js
// 旧节点
h('div', {}, '123')

// 新节点
h('ul', {}, ['345', '4555'])
```

> 在`patch`方法中判断

```js
	/**
	 * 判断是否挂载还是更新
	 * @param n1         旧虚拟节点
	 * @param n2         新虚拟节点
	 * @param container  挂载跟节点
	 */
	function patch(n1, n2, container, auchor = null) {
		// 判断节点是否相同,如果不同则直接删除旧节点
+	if (n1 && !isSameVnode(n1, n2)) {
+			container.innerHTML = "";
+			n1 = null;
+		}
		// 判断新虚拟节点类型
		const { shapeFlag } = n2;
		if (shapeFlag & ShapeFlags.ELEMENT) {
			//节点类型
			processElement(n1, n2, container, auchor);
		} else if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
			//组件类型
			processComponent(n1, n2, container);
		}
	}
```

#### 新旧节点类型不同，新子节点为文本

```js
	/**
	 * 判断是否挂载还是更新
	 * @param n1         旧虚拟节点
	 * @param n2         新虚拟节点
	 * @param container  挂载跟节点
	 */
	function patch(n1, n2, container, auchor = null) {
		...
		if (shapeFlag & ShapeFlags.ELEMENT) {
			//节点类型
			processElement(n1, n2, container, auchor);
		} else if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
			//组件类型
			processComponent(n1, n2, container);
		} else {
+ 		container.textContent = container.textContent + n2;
    }
	}
```

#### 新旧虚拟节点类型相同，旧子节点为文本，新子节点为数组

```js
// 旧节点
h('div', {}, '123')

// 新节点
h('div', {}, [h('span', {}, '345'), h('span', {}, '456')])
```



> 在对根节点类型相同后，我们需要对比子节点，因为我们提供的例子都是使用的元素类型，故在`processElement`中改造

```js

	/**
	 * 创建节点
	 * @param n1
	 * @param n2
	 * @param container
	 */
	function processElement(n1, n2, container, auchor) {
		if (n1 === null) {
			mountElement(n2, container, auchor);
		} else {
			// 更新 diff 算法
			patchElement(n1, n2, container, auchor);
		}
	}
```

```js
	/**
	 * 对比新旧节点的属性/子节点
	 * @param n1
	 * @param n2
	 * @param container
	 */
	function patchElement(n1, n2, container, auchor) {
		let el = (n2.el = n1.el);
		const oldProps = n1.props || {};
		const newProps = n2.props || {};
		patchProps(el, oldProps, newProps); //对比属性
		// 对比子节点
		patchChildren(n1, n2, el, auchor);
	}
```

> 属性对比实现

```js
/**
	 * 属性对比
	 * @param el
	 * @param oldProps
	 * @param newProps
	 */
	function patchProps(el, oldProps, newProps) {
		if (oldProps === newProps) return;
		for (let key in newProps) { //先循环新节点属性
			const prev = oldProps[key];
			const next = newProps[key];
			if (prev !== next) {
				hostPatchProp(el, key, prev, next);
			}
		}
		for (let key in oldProps) {
			if (!hasOwn(newProps, key)) { // 删除新属性中不存在的旧属性的key
				hostPatchProp(el, key, oldProps[key], null);
			}
		}
	}
```

> 子节点比对

```js
/**
	 * 对比子节点
	 * @param n1   旧节点
	 * @param n2   新节点
	 * @param container
	 */
	function patchChildren(n1, n2, container, auchor) {
		const c1 = n1.children;
		const c2 = n2.children;

		const prevShageFlag = n1.shapeFlag;
		const shapeFlag = n2.shapeFlag;
		// 1. 当前子节点是文本,则直接替换
		if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
			hostSetElementText(container, c2);
		} else {
			// 当前子节点是数组
			if (prevShageFlag & ShapeFlags.ARRAY_CHILDREN) {
				//之前的子节点也是数组
        patchKeyedChildren(c1, c2, container);
			} else {
				// 之前子节点是文本
				hostSetElementText(container, ""); // 清空之前节点
				mountChildren(c2, container, auchor); //挂载当前子节点
			}
		}
	}
```

#### 根节点属性相同，且新旧节点都为数组

``` js
	/**
	 * 数组子节点比较
	 * @param c1
	 * @param c2
	 * @param container
	 */
	function patchKeyedChildren(c1, c2, container) {
		let i = 0;
		let e1 = c1.length - 1;
		let e2 = c2.length - 1;
		// 从前往后比较
		while (i <= e1 && i <= e2) {
			if (isSameVnode(c1[i], c2[i])) {
				// 如果是相同类型元素, 则比较属性和子节点
				patch(c1[i], c2[i], container);
			} else {
				break;
			}
			i++;
		}
		// 从后往前比较
		while (i <= e1 && i <= e2) {
			if (isSameVnode(c1[e1], c2[e2])) {
				patch(c1[e1], c2[e2], container);
			} else {
				break;
			}
			e1--;
			e2--;
		}
		// 有序比对
		if (i > e1) {
			//新的多,旧的少
			if (i <= e2) {
				const nextPos = e2 + 1;
				const anchor = nextPos < c2.length - 1 ? c2[nextPos].el : null;
				// 如果anchor 不为null,则是在当前元素添加
				while (i <= e2) {
					patch(null, c2[i++], container, anchor);
				}
			}
		} else if (i > e2) {
			// 老的多,新的少
		} else {
			// 乱序比对
		}
	}
```

> 简单学习，后续补充乱序比对

