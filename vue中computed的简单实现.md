# `VUE`中`computed`的简单实现

> `computed`的实现其实也是依赖于`effect`，在我们使用`computed`时，里面引用的属性值会将其放到自己的依赖栈中，当属性值被更新时，会用`trigger`，具体实现如下

```js
/**
 * @param objectOptions
 *  可能是一个方法或者是一个对象里面设置了set/get方法
 */
export function computed(objectOptions) {
	let getter, setter;
	if (isObject(objectOptions)) {
		getter = objectOptions.get;
		setter = objectOptions.set;
	}
	getter = objectOptions;
	setter = () => {
		console.log("no setter");
	};

	return new ComputedRefImpl(getter, setter);
}
```

> 我们首先将`computed`中传入的参数做下处理，然后创建`ComputedRefImpl`实例，`ComputedRefImpl`的实现如下

```js
class ComputedRefImpl {
	public effect;
	public _value;
	constructor(public getter, public setter) {
		this.effect = effect(getter, {
			lazy: true, //开始不执行
			// 如果设置了schedular,则更新的时候就不触发effect,而是触发schedular
			schedular: (effect) => { //依赖有更新
			},
		});
	}
	get value() {
		this._value = this.effect();
		return this._value;
	}

	set value(newValue) {
		this.setter(newValue);
	}
}
```

> 将`computed`传入的方法作为`effect`的方法传入，这样在`computed`中取值的响应式数据的属性就会收集到当前这个`effect`作为依赖，当我们修改响应式数据的属性值，就会触发`computed`中的`effect`执行，我们在次基础上增加属性`schedular`，并修改`effect`中的`trigger`方法，当含有属性`schedular`时，我们就直接触发`schedular`而不触发依赖，相当于做了一个定制化，`effect`中的`trigger`修改如下：

```js
export function trigger(target, type, key, oldValue?, newValue?) {
	const depsMap = depMaps.get(target);
	if (!depsMap) return;
	const willEffectsSet = new Set();
	const add = (effectsSet) => {
		if (effectsSet) {
			effectsSet.forEach((effect: any) => willEffectsSet.add(effect));
		}
	};
	if (key === "length" && isArray(target)) {
		...
	} else {
		...
	}
	willEffectsSet.forEach((effect: any) => {
+		if (effect.options.schedular) {
+			effect.options.schedular(effect);
		} else {
			effect();
		}
	});
}

```

#### 案例一

```js
  const {computed,reactive,effect} = VueReactivity
  const proxy = reactive({baseName: 'wangly'})
  const newName = computed(() => {
    return proxy.baseName + 'change'
  })
  // 第一次取值, 会将computed里面的effect收集到proxy.baseName的依赖树中
  newName.value  // wanglychange
	// 修改proxy.baseName
	proxy.baseName = 'outside wangly'
	// 依赖被修改取值
	newName.value // outside wanglychange
```

> 如果我们在响应式属性值不改变的情况下，多次取值`newName`，会不停的在`get`中去触发`this.effect`，这李我们可以做个优化，依赖值没有变化时，多次取值不会触发`this.effect`的执行，修改如下：

```js
class ComputedRefImpl {
	...
+	public dirty = true; //防止多次取值
	constructor(public getter, public setter) {
		this.effect = effect(getter, {
			lazy: true, //开始不执行
			// 如果设置了schedular,则更新的时候就不触发effect,而是触发schedular
			schedular: (effect) => { //依赖有更新
+				if(!this.dirty){ 
+         this.dirty = true // 将标识设置成需要计算最新值
        }
			},
		});
	}
	get value() {
+		if (this.dirty) {  //如果依赖有更新,则重新取值,如果没有则直接返回缓存
+     console.log('依赖有更新,重新取值');
+     this._value = this.effect();
+     this.dirty = false //将标识设置成已计算最新值，不需要重新计算
		}
		return this._value;
	}

	set value(newValue) {
		this.setter(newValue);
	}
}

```

#### 案例二

```js
	const {computed,reactive,effect} = VueReactivity
  const proxy = reactive({baseName: 'wangly'})
  const newName = computed(() => {
    console.log('重新取值')
    return proxy.baseName + 'change'
  })
  // 多次取值 只会打印一次  重新取值
  newName.value  // wanglychange
  newName.value  // wanglychange
  newName.value  // wanglychange

	// 改变 proxy.baseName 的值
	proxy.baseName = 'outside wangly'
	// 再次取值，会重新计算
  console.log(newName.value); // 重新取值  outside wanglychange
```

> 我们还需要考虑下，如果计算属性被其他`effect`引用时，该怎么触发依赖

#### 案例三

```js
const {computed,reactive,effect} = VueReactivity
const proxy = reactive({baseName: 'wangly'})
const newName = computed(() => {
  return proxy.baseName + 'change'
})
effect(() => { // 在当前effect引用了计算属性的值
  console.log(newName.value);
})
proxy.baseName = 'outside wangly' // 修改计算属性引用的响应书数据的值
```

> 我们就需要再在`computed`取值的时候收集依赖，取值的时候重新触发自己本身的`effect`，修改如下

```js
class ComputedRefImpl {
	public effect;
	public _value;
	public dirty = true; //防止多次取值
	constructor(public getter, public setter) {
		this.effect = effect(getter, {
			lazy: true, //开始不执行
			// 如果设置了schedular,则更新的时候就不触发effect,而是触发schedular
			schedular: (effect) => { //依赖有更新
				if(!this.dirty){
          this.dirty = true
          trigger(this, 'set', 'value') //触发依赖更新
        }
			},
		});
	}
	get value() {
		if (this.dirty) {  //如果依赖有更新,则重新取值,如果没有则直接返回缓存
      console.log('依赖有更新,重新取值');
			this._value = this.effect();
      this.dirty = false
		}
    track(this, 'get', 'value') // 当计算属性被其他effect引用时,也要收集依赖
		return this._value;
	}

	set value(newValue) {
		this.setter(newValue);
	}
}

```

完成！

简单学习，后续补充