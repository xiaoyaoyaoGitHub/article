# `Vue3`中`ref/toRef/toRefs`的简单实现

> 在`Vue3`中`ref`的实现是依靠类实现的，然后在编译过程中转化成了`defineProperty`，详情请看[RefImple类的编译结果](https://babel.docschina.org/repl/#?browsers=defaults%2C%20not%20ie%2011%2C%20not%20ie_mob%2011&build=&builtIns=false&spec=false&loose=false&code_lz=MYGwhgzhAEBKCmAzAkgWwA4mgbwFAEhgB7AOwgBcAnAV2HKMoApKwB3ANTBGvgEocC-cgAsAlhAB0AfQBuXHtAC80Fh3nwA3AQC-uAgHN45aHO7xG_PPnyUj1SiWgjx00zy35dBCEZPrGJPBqZpa40OER4c6SsupK0IHB7jq4ukA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=env%2Ces2015%2Creact%2Cstage-2%2Cenv&prettier=false&targets=&version=7.15.2&externalPlugins=)

#### `ref`的简单实现

```js
class RefImpl {
	public _value;
	constructor(public rawValue) {
		this._value = reactive(rawValue);
	}

	get value() {
		track(this, "get", "value");
		return this._value;
	}

	set value(newValue) {
     this._value = newValue;
		 trigger(this, "set", "value", this.rawValue, newValue);
	}
}

export function ref(value){
  return new RefImpl(value)
}
```

这里我们利用了之前说过的`track`和`trigger`收集和触发依赖

#### `toRef`的简单实现

> 其实是利用代理的方式去取`Proxy`实例上的值

```js
// 通过代理的方式去Proxy中取值
class ObjectRefImpl {
	public _v_isRef = true;
	constructor(public target, public key) {}

	get value() {
    //其实是取proxy中的值，所以会触发track
		return this.target[this.key];
	}

	set value(newValue) {
		this.target[this.key] = newValue;
	}
}
// 其中target 是被 reactive 包装过的proxy
export function toRef(target, key) {
	return new ObjectRefImpl(target, key);
}
```

> 使用场景

```js
const school = {name:'wangly', age:'19'};
const nameByRef = toRef(school, 'name');
effect(() => {
  //此时我们取值的方式就需要使用.value
  console.log(nameByRef.value)
})
setTimeout(() => {
  nameByRef.value = 'change wangly'
},1000)
```

在间隔1s后更改`nameByRef`的值，会触发`effect`的重新执行

#### `toRefs`的简单实现

> 就是批量将`target`上的属性的值进行`toRef`

```js
export function toRefs(target) {
	const res = isArray(target)
		? new Array(target.length)
		: Object.create(null);
	// 循环将每一项都设置toRef
	for (let key in target) {
		res[key] = toRef(target, key);
	}
	return res;
}
```

简单学习，后续补充