# `Vue diff`算法之乱序比对

如下为需要比对的新旧节点：

![演示文稿1](/Users/wangly/Documents/study/文章/演示文稿1-9807223.png)

> 经过我们之前的从开始位置比对和从末尾比对后，我们得到节点差异如下虚框中所示

![1](/Users/wangly/Documents/study/文章/1.png)

#### 以新节点中的差异节点生成映射表

> 我们使用节点中的`key`值为映射，将当前虚拟节点的索引做保存

```js
let s2 = i; // 从开始位置循环到节点不同的索引值
const keyForNewNode = new Map(); // 没有使用weakMap 是因为weakMap 只能使用引用类型做key
for(let i = s2; i<= e2; i++){
  keyForNewNode.set(c2[i].key, i) // c2为新节点
}
```

得到的结果如下所示，这是我们表示新虚拟节点中的各个节点和对应的索引

```js
Map(4) {'C' => 1, 'F' => 2, 'B' => 3, 'E' => 4}
```

> 之后我们需要对旧虚拟节点中的差异部分进行循环，在映射表中查找是否有`key`相同的节点，如果有则进行比对，如果映射表中不存在则需要进行删除

```js
let s1 = i;
for(let i = s1; i <= e2; i++){
  const oldVnode = c1[i];
  const newVnodeIndex = keyForNewNode.get(oldVnode.key); //根据key取在新虚拟dom中的索引值
  if(newVnodeIndex){ //有值 开始比对差异
    	patch(oldVnode, c2[newVnodeIndex], container)
  }else{ // 映射表中不存在
     unmount(oldVnode)
  }
}
```

> 到这一步为止，我们可以删除掉`c2`中不存在的节点，也可以更新`c1`中的节点的属性值，结果如下所示

![2](/Users/wangly/Documents/study/文章/2.png)

可以看到在新节点中不存在的节点`G`被删掉了，节点`C`的颜色也变成了红了。但目前我们还有两个问题没有解决

- 节点位置不对
- 新增节点`F`没有挂载成功

#### 创建数组保存旧虚拟节点可`patch`节点的真实位置

> 为了解决上面这两个问题，我们可以考虑创建数组用来保存差异节点的节点对应旧节点中的索引，在循环旧节点的时候，将需要可以`patch`的旧节点的索引保存到数组中

```js
+ const toBePatch = e2 - s2 + 1;
+ const newIndexToOldIndexMap = new Array(toBePatch).fill(0) //创建一个内容都为0的数组
  for(let i = s1; i <= e2; i++){
    const oldVnode = c1[i];
    const newVnodeIndex = keyForNewNode.get(oldVnode.key); //根据key取在新虚拟dom中的索引值
    if(newVnodeIndex){ //有值 开始比对差异
+     newIndexToOldIndexMap[newVnodeIndex-s2] = i + 1 //保存当前节点在旧节点中的位置
      patch(oldVnode, c2[newVnodeIndex], container)
    }else{ // 映射表中不存在
      unmount(oldVnode)
    }
  }
```

#### 从尾部移动节点位置和创建新节点

> 我们得到了节点在旧节点中的位置，打印如下

```js
newIndexToOldIndexMap (4) [5, 0, 2, 3]
```

- 5：为节点`C`在旧节点中的第5位
- 0：节点`F`新虚拟节点中有，在旧节点中不存在
- 2：节点`B`在旧节点中的第2位
- 3：节点`E`在旧节点中的第3位

可以看出当数组值为`0`时是需要我们新创建的节点，然后我们就从差异节点尾部循环插入或者创建

```js
for(let i = toBePatch - 1; i>=0; i--){
  const newIndex = i + s2; //c2中差异节点的真实位置
  // 找到auchor,如果当前节点后面还有节点就取出
  const auchor = newIndex + 1 < c2.length ? c2[newIndex + 1].el : null;
  if(newIndexToOldIndexMap[i] === 0){ // 新节点挂载
    	patch(null, c2[newIndex], container, author)
  }else{
     hostInsert(c2[newIndex].el, container, auchor)
  }
}
```

查看效果：

![3](/Users/wangly/Documents/study/文章/3-9809555.png)

因为我们现在的实现是移动了所有的差异节点，其实是有性能问题，我们下节更新`vue`中的最长递增子序列是如何做优化，简单实现，后续补充

