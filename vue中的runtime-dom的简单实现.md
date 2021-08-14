# `Vue`中的`runtime-dom`的简单实现

> `runtime-dom`的主要功能针对不同平台统一dom操作，比如浏览器的对`dom`操作的原生`api`的整合，通过修改这个包，可以实现对不同平台的接入，例如`weex`，可以将自己平台的`dom`操作整合到`runtime-dom`中，之后我们将这些整合后的`api`传入到`runtime-core`中使用

#### 节点的操作

`nodeOps.js`

```js
// 增 删 改 查 元素中插入文本  文本的创建 文本元素内容的设置 获取父节点 获取相邻节点
export const nodeOps = {
  // 创建节点
	createElement: (tag) => document.createElement(tag), 
  // 移除节点
	remove: (child) => child.parentNode && child.parentNode.removeChild(child),
  // 插入节点，如果anchor为null,则是appendChild
	insert: (child, parent, anchor = null) => parent.insertBefore(child, anchor),
  // 获取节点
	querySelector: (selector) => document.querySelector(selector),
  // 设置节点内容
	setElementText: (el, text) => (el.textContent = text),
  // 创建文本节点
	createText: (text) => document.createTextNode(text),
  // 设置节点值
	setText: (node, text) => (node.nodeValue = text),
  // 获取父节点
	getParent: (node) => node.parentNode,
  // 获取相邻节点
	getNextSibling: (node) => node.nextElementSibling,
};

```

#### 节点中属性的操作

`patchProp.js`

* 总入口

```js
/**
 *
 * @param el   节点
 * @param key  对比的属性
 * @param prev 旧值
 * @param next 新值
 */
export const patchProp = (el, key, prev, next) => {
	switch (key) {
		case "class":
			patchClass(el, next);
			break;
		case "style":
			patchStyle(el, prev, next);
			break;
		default:
			if (/^on[^a-z]/.test(key)) {
				// 事件
				patchEvents(el, key, next);
			} else {
				// 其他属性
        patchAttrs(el, key, next)
			}
	}
};

```

* 针对`class`属性

```js
/**
 * 设置class属性值
 * @param el   节点
 * @param next 新属性值
 */
function patchClass(el, next) {
	if (next == null) {
		next = "";
	}
	el.className = next;  //不需要比对，直接设置新值
}
```

* 针对`style`属性

```js
/**
 * 设置style属性值
 * @param el    节点
 * @param prev  旧值
 * @param next  新值
 */
function patchStyle(el, prev, next) {
	if (next == null) {
		//如果新属性值没有,则移除style属性
		el.removeAttribute("style");
	} else {
		if (prev) {
			for (let key in prev) {
				if (!next[key]) {
					//如果新属性值中没有当前属性值,则直接设置为空
					el.style[key] = "";
				}
			}
			// 循环新属性值 依次设置
			for (let key in next) {
				el.style[key] = next[key];
			}
		}
	}
}
```

* 针对`Event`的绑定

```js
/**
 * 设置事件
 * @param el   节点
 * @param key  事件类型
 * @param next 触发事件
 */
function patchEvents(el, key, next) {
	// 判断当前节点有没有绑定事件
	const invokers = el._vei || (el.vei = {});
	const exists = invokers[key];
	if (exists && next) {
		exists.value = next;
	} else {
		// 处理事件类型 onClick => click
		const eventName = key.toLowerCase().slice(2);
		if (!next) {
			//如果绑定事件木有了则移除监听
			el.removeEventListener(eventName);
		} else {
			// 如果绑定或者删除事件监听,对性能损耗较大,然后可以将需要更改监听统一使用invoker,在invoker中的value属性中保存需要监听的方法,之后改变就改变value的值
			const invoker = createInvoker(next);
			el.addEventListener(eventName, invoker);
		}
	}
}
```

> 在这里，我们对事件监听函数统一传入由`createInvoker`创建的函数，在这个函数上我们使用属性`value`保存当前用户传入的方法，这样如果用户更改传入，我们就可以直接去替换`value`对应的值，不比再去操作`removeEventListener`，节省性能，以下是`createInvoker`的实现

```js
/**
 * 创建监听函数
 * @param fn addEventListener 需要监听的事件
 */
function createInvoker(fn) {
	const invoker = (e) => invoker.value(e);
	invoker.value = fn;
	return invoker;
}
```

* 其他属性

```js
/**
 * 对比其他属性值
 * @param el   节点
 * @param key  属性名
 * @param next 新值
 */
function patchAttrs(el, key, next) {
	if (next) {
		el.setAttribute(key, next);
	} else {
		el.removeAttribute(key);
	}
}
```

#### 入口文件

> `index.js`提供方法`createApp`，在该函数中调用由`runtime-core`提供的方法`createRenderer`方法，将我们之前封装的`dom`操作传入给`runtime-core`，后续我们重写`runtime-core`提供的`mount`方法，在挂载之前先清空挂载节点的内容

```js

import { extend } from "@wangly/share";
import { createRenderer } from "@wangly/runtime-core";
import { nodeOps } from "./nodeOps";
import { patchProp } from "./patchProp";

const renderOptions = extend(nodeOps, { patchProp });

/**
 *  创建节点
 * @param rootComponent  组件
 * @param rootProp       组件传入的属性值
 */
export function createApp(rootComponent, rootProp = null) {
	const app = createRenderer(renderOptions).createApp(
		rootComponent,
		rootProp
	);
	const { mount } = app || {};
	// 重写mount
	app.mount = function (container) { // 函数劫持
		container = document.querySelector(container);
		container.innerHTML = ""; //挂载之前先清空
		mount(container);
	};
	return app;
}
```

完成

简单学习， 后续补充