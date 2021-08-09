# `Vue3`中的`effect`的实现

> 在`vue3`版本中的响应式实现是依赖`effect`，过程是我们在读取值`get`的时候出发`track`收集依赖，在设置值`set`的时候通过`trigger`触发依赖，实现更新

#### 使用`Proxy`实现数据劫持

```js
/**
 *
 * @param target 创建代理目标
 * @param isReadonly 是否为只读
 * @param baseHandler  针对不同的方式创建不同的代理对象
 */

const reactiveMap = new WeakMap(); //创建映射用作缓存,相对于map来说可以不用手动清除
const readonlyMap = new WeakMap(); //readonly映射缓存
function createReactiveObject(target, isReadonly = false, baseHandler) {
	if (!isObject(target)) {
		return target;
	}
	const proxyMap = isReadonly ? readonlyMap : reactiveMap;
	// 判断是否有缓存,如果有则直接返回缓存对象不再拦截
	if (proxyMap.get(target)) {
		return proxyMap.get(target);
	}
	const proxyTarget = new Proxy(target, baseHandler); //创建proxy
	proxyMap.set(target, proxyTarget);
	return proxyTarget;
}

```

> 在`baseHandler`里面，我们需要实现对数据的获取和设置的劫持，实现如下：

```json
{
  get:(target, key, receiver) {
		// 使用Reflect做映射取值
		const res = Reflect.get(target, key, receiver);
    // 需要过滤掉类型是symbol的属性不做收集
		if (typeof key !== 'symbol') { // 收集依赖
			track(target, "get", key);
		}
		// 如果res是对象,则对其进行再次代理
		if (isObject(res)) {
			// 懒递归,当我们去取值时,才去做递归,如果不取默认代理1层
			return reactive(res);
		}

		return res;
	},
	set:(target, key, value, receiver) {
		const oldValue = target[key];
		// 是否存在当前设置的key 区分下数组的新增和修改
		const hadKey =
			isArray(target) && isIntegerKey(key)
				? Number(key) < target.length
				: hasOwn(target, key);
		const res = Reflect.set(target, key, value, receiver);
		if (!hadKey) {//是新增
			trigger(target, "add", key, oldValue, value);
		} else if (hasChange(oldValue, value)) {
			trigger(target, "set", key, oldValue, value);
		}
		return res;
	};
}
```

> 可以看到，我们在`get`中调用`track`收集依赖，在`set`中调用`trigger`触发依赖

#### `effect`方法

```js
const effectStack = []; //effect 栈
let activeEffect; // 当前执行的effect
let id = 0;

function createReactiveEffect(fn, options) {
	const effect = function reactiveEffect() {
		try {
			effectStack.push(effect); // 放入当前执行栈
			activeEffect = effect;
			return fn(); //执行方法
		} finally {
			effectStack.pop(); // 执行完成后弹出
			activeEffect = effectStack[effectStack.length - 1]; // 修改当前执行effect
		}
	};
	effect.id = id++;
	effect.options = options;
	effect.deps = []; // 用来收集依赖
	return effect;
}
```

> 在这里，我们创建了一个`effectStack`执行栈，和变量`activeEffect`用来保存当前正在执行的`effect`。执行栈按照后入先出的原则执行。在此函数中我们在执行`fn()`的时候，会读取数据，之后会触发我们在`get`方法中的`track`函数

#### `track` -- 收集依赖

```js
// 创建weakMap, 收集依赖
const depMaps = new WeakMap();
export function track(target, type, key) {
	if (!activeEffect) {//如果只是单纯的取值,则不需要保存
		return;
	}
	let deps = depMaps.get(target);
	if (!deps) {
		//如果不存在则重新设置
		depMaps.set(target, (deps = new Map()));
	}
	let dep = deps.get(key);
	if (!dep) {
		deps.set(key, (dep = new Set())); //使用Set是为了防止重复
	}
	if (!dep.has(activeEffect)) {
		dep.add(activeEffect); //将当前执行栈放入属性的依赖结果中
	}
}
```

> 我们可以看下执行完后`depMaps`的格式

```markdown
WeakMap {
  0: {Array(5) => Map(9)}
      key: (5) [1, 2, 3, 45, 345]
      value: Map(9) {'valueOf' => Set(1), 'toString' => Set(1), 'join' => Set(1), 'length' => Set(1), '0' => Set(1)}
  1: {Object => Map(1)}
      key:{address: {num: '677'}, age: "18", name: "wangly",score: (5) [1, 2, 3, 45, 345]}
      value: Map(1) {'score' => Set(1)}
}
```

> 这样我们就知道了，如果属性值变化的话，需要通知到谁，可以按照这个结构取到我们需要触发的依赖

#### `trigger`函数 -- 依赖的触发

```js
// 触发更新
export function trigger(target, type, key, oldValue, newValue) {
	const depsMap = depMaps.get(target); //取到当前target中有没有属性收集依赖
	if (!depsMap) return;
	const willEffectsSet = new Set(); // 使用Set是可以去重，避免重复放置effect
	const add = (effectsSet) => {
		if (effectsSet) {
			effectsSet.forEach((effect: any) => willEffectsSet.add(effect));
		}
	};
	if (key === "length" && isArray(target)) { //如果数组直接修改length的长度
		depsMap.forEach((dep, setKey) => {
			//查看当前监听里面是否含有比当前数组大的索引值,如果有则更新
			if (setKey > newValue || setKey === "length") {
					add(dep);
				}
		});
	} else {
		add(depsMap.get(key));
		if (isArray(target) && isIntegerKey(key)) {
			switch (type) {
				case "add":// 如果数组通过push增加,修改了length,则触发不了监听,需要手动触发
					add(depsMap.get("length"));
					break;
			}
		}
	}
	willEffectsSet.forEach((effect: any) => effect());
}
```

> 这里面值得注意的是，`proxy`只能触发它代理过的属性，如果我们这样去修改

```js
const school = {score:[1,2,3,45,345]}
effect(() => {
  console.log(school.score[3])
})

setTimeout(() => {
  school.score.push(6666)
})
```

> 在这里，我们给数组新添加个值，新增索引6，但是在`Proxy`中其实我们并没有代理过，所以必须手动去触发`length`更新，在使用`push`方法时，会先增加索引，后续更改`length`属性（此处`length`已经被代理过）

简单学习，后续补充