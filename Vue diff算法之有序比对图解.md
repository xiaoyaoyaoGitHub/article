# `Vue diff`算法之有序比对图解

#### 中间新增

- 从开始位置比对

  ![幻灯片1](/Users/wangly/Documents/study/文章/幻灯片1-9789913.png)

- 从尾部位置开始比对

  ![幻灯片2](/Users/wangly/Documents/study/文章/幻灯片2-9789945.png)

对比结果:

```js
i = 2;
e1 = 1;
e2 = 3
```

#### 尾部新增

![演示文稿1](/Users/wangly/Documents/study/文章/演示文稿1-9790212.png)

对比结果:

```js
i = 4；
e1 = 3;
e2 = 5
```

#### 头部新增

![幻灯片4](/Users/wangly/Documents/study/文章/幻灯片4.png)

对比结果：

```js
i = 0;
e1 = -1;
e2 = 1
```

综合以上情况我们可以得出：

- 当` i > e1 && i <= e2`时，说明我们新节点要比旧节点多，反之如果`i > e2 && i<= e1`则为旧节点要比新节点多；
- 当新节点比旧节点多时，就可以利用`e2+1`查找如果到从尾部循环最后相同的一个节点值
  - 如果当前节点值存在，则将新增节点插入到当前节点之前；
  - 如果当前节点不存在，则是在尾部添加；
- 当新节点少于旧节点时，则直接删除；

```js
// 有序比对
if (i > e1) {
  if ( i <= e2) {
    const nextPos = e2 + 1;
    const anchor = nextPos < c2.length - 1 ? c2[nextPos].el : null;
    // 如果anchor 不为null,则是在当前元素添加
    while (i <= e2) {
      patch(null, c2[i++], container, anchor);
    }
  }
} else if (i > e2) {
  // 老的多,新的少
  while (i <= e1) {
    unmount(c1[i++]);
  }
} 
```

