## ReactNative JS 层渲染之 diff 算法{docsify-ignore}

React 在界面刷新（`setState`）时，并不会马上对所有的 DOM 节点进行操作，而是先通过 diff 算法计算后，再对有变化的 DOM 节点进行操作（native 是对原生 UI 层进行操作），刷新步骤如下：

```
1. state 变化，生成新的 Virtual Dom
2. 比较 Virtual Dom 与之前 Virtual Dom 的异同
3. 生成差异对象
4. 遍历差异对象并更新真实 DOM
```

### Virtual Dom 概述

对 DOM 的操作很耗时，使用 JS 对象来模拟 DOM Tree，在渲染更新时，先对 JS 对象进行操作，再按批将 JS 对象 Virtual Dom 渲染成 DOM Tree，减少对 DOM 的操作，提升性能。

整个 diff 算法，都是基于 Virtual Dom 的，那什么是 Virtual Dom 呢？

```
Virtual Dom：本质是用来模拟DOM的JS 对象。
一般含有标签名（tag）、属性（props）和子元素对象（children） 三个属性。
不同框架对属性的命名会有点差别，但表达的意思是一致的。
```

比如如下一段代码，怎么映射成对应的 Virtual Dom 呢？

```javascript
<A>
  Hello World
  <B>
    <C key="key-C" style={{ width: 100 }} />
    <D key="key-D" style={{ color: 'red' }} />
  </B>
</A>
```

由于 Virtual Dom 本质上就是由标签名（tag）、属性（props）和子元素对象（children） 三个属性组成的 js 对象，在该例中，`A`、`B`、`C`是标签（tag）,`key`、`style`是属性（props），节点`B`是节点`A`的子元素对象（children），节点`C`和`D`是节点`B`的子元素对象（children），转化后的 js 对象（Virtual Dom）如下：
![Virtual Dom结构图](https://upload-images.jianshu.io/upload_images/3995013-f9d8f09a2cf0b371.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/100)

### React Diff 算法的两个假设

基于以下两个假设，React 对 传统的 diff 算法进行优化，将复杂度从 O(n^3)降到 O(n)

```
1.两个相同组件将会生成相似的DOM结构，两个不同组件将会生成不同的DOM结构。
2.对于同一层次的一组子节点，它们可以通过唯一的id进行区分。
```

React Diff 算法的实现，几乎都是基于以上两个假设进行的。

### React Diff 算法的实现

React Diff 算法的实现，主要有相同类型节点的比较、不同节点类型的比较、列表节点的比较三种情况，这里也主要针对这三种情况进行分析。

#### 1. 相同类型节点的比较

- 修改某一个节点的属性（如 style）

![相同类型节点的比较](https://upload-images.jianshu.io/upload_images/3995013-3ea762aa423bc0a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

依据假设 `两个相同组件将会生成相似的 DOM 结构`，由于新旧节点类型相同，DOM 结构没有发生变化，React 仅对属性（如 style）进行重设从而实现节点的转换。

#### 2. 不同节点类型的比较

- 将节点 A 及其子节点改成节点 D 的子节点

![不同节点类型的比较-code](https://upload-images.jianshu.io/upload_images/3995013-767f6d8411462f23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

为了方便分析，首先抽象成 DOM tree 节点模型，如下
![不同节点类型的比较-tree](https://upload-images.jianshu.io/upload_images/3995013-5aebe779ce8fd693.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

依据假设`两个不同组件将会生成不同的 DOM 结构`，当发现节点已经不存在，则该节点及其子节点会被完全删除掉，不会用于进一步的比较，因此，React 会直接删除前面的节点，然后创建并插入新的节点。

#### 3. 列表节点的比较

在渲染列表节点时，它们一般都有相同的结构，只是内容有些不同而已。

依据假设`对于同一层次的一组子节点，它们可以通过唯一的 key 进行区分`，通过给每个节点添加唯一的 key，可以极大的简化 diff 算法，减少对 DOM 的操作。列表节点的比较主要有`添加节点`、`删除节点`、`排序`三种场景进行：

##### 3.1 添加节点

- 在节点 B 与 C 之间插入节点 F

![添加节点-题目](https://upload-images.jianshu.io/upload_images/3995013-8edd756eee6da9c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

每个节点是否添加唯一标识 key 的算法实现与对比

![列表节点的比较-添加节点](https://upload-images.jianshu.io/upload_images/3995013-4b488fc909020c7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

##### 3.2 删除节点

- 删除节点 B

![删除节点](https://upload-images.jianshu.io/upload_images/3995013-6628550854532c01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

##### 3.2 节点排序

- 交换节点 B 和 C 的位置

![排序-题目](https://upload-images.jianshu.io/upload_images/3995013-e620f7438440f311.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

每个节点是否添加唯一标识 key 的算法实现与对比

![列表节点的比较-排序](https://upload-images.jianshu.io/upload_images/3995013-9b884ceab2996cf6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

方案一：在没有添加 key 的情况下，无法定位到具体的节点，只能通过遍历，依次比较，再逐个更新。比如在该例中，要交换 shape5 和 shape6 的节点 B 和 C 的位置，执行操作如下：

```
  1.shape6 的节点 C 与 shape5 的节点 B 进行比较，节点 B 更新成节点 C

  2.shape6 的节点 B 与 shape5 的节点 C 进行比较，节点 C 更新成节点 B。
```

方案二：在有唯一稳定的 key 的情况下，可以直接定位到具体的节点，只需要对相应的节点进行排序即可。比如在该例中，要交换 B 和 C 的位置，执行操作如下：

```
  1.只需要交换节点 B、C 的位置即可。
```

综上，可以发现没有 key 的情况下，需要对 DOM 进行 2 次操作，有 key 的情况下，只需要对 DOM 进行 1 次操作即可。

```
注意事项：
1.如果key不稳定（如取index），则效果与不设置key是一样的
2.key如果不唯一（如不同节点的key取值相同），如果列表的所有组件类型相同，则效果与不设置key是一样的，如果列表的组件类型不同，则可能会出现重复渲染等异常情况。
```
