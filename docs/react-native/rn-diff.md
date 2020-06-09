## ReactNative JS 层渲染之 diff 算法{docsify-ignore}

为什么讲 ReactNative JS 层渲染，重点讲 diff 算法呢？

> 使用 React 写过 Web 和 ReactNative 的，能很明显的感觉到，除了组件命名不一样之后，生命周期、刷新机制等几乎是完全一样的，这也就是 facebook 所说的`“learn once, write anywhere”`，只要会写 React，就能无压力同时开发 Web 和 ReactNative。而 React 框架相对于传统的纯 js 开发的优势，核心就是组件化和 diff 算法刷新机制，这两个极大的提升了开发效率和程序的渲染性能

React 通过`setState`界面刷新时，并不会马上对所有的真实的 DOM 节点进行操作，而是先通过 diff 算法计算后，再对有变化的 DOM 节点进行操作（native 是对原生 UI 层进行操作），具体刷新步骤如下：

```
1. state 变化，生成新的 Virtual Dom
2. 比较 Virtual Dom 与之前 Virtual Dom 的异同
3. 生成差异对象
4. 遍历差异对象并更新真实 DOM
```

### Virtual Dom 概述

对 DOM 的操作很耗时，使用 JS 对象来模拟 DOM Tree，在渲染更新时，先对 JS 对象进行操作，再按批将 JS 对象 Virtual Dom 渲染成 DOM Tree，减少对 DOM 的操作，提升性能。

整个 diff 算法，都是基于 Virtual Dom 的，那什么是 Virtual Dom 呢？

> Virtual Dom 本质是用来模拟 DOM 的 JS 对象。一般含有标签名（`tag`）、属性（`props`）和子元素对象（`children`） 三个属性，不同框架对属性的命名会有点差别，但表达的意思是一致的

例：对于以下一段代码，怎么映射成对应的 Virtual Dom 呢？

```javascript
<A>
  Hello World
  <B>
    <C key="key-C" style={{ width: 100 }} />
    <D key="key-D" style={{ color: 'red' }} />
  </B>
</A>
```

在该例中，按三个属性分析如下

- `A`、`B`、`C`是标签（tag）
- `key`、`style`是属性（props）
- 节点`B`是节点`A`的子元素对象（children）
- 节点`C`和`D`是节点`B`的子元素对象（children）
- 最后，映射出来的 js 对象（Virtual Dom）如下：

![Virtual Dom结构图](https://upload-images.jianshu.io/upload_images/3995013-f9d8f09a2cf0b371.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

### React Diff 算法的两个假设

为什么 React Diff 算法相对于传统的 diff 算法，复杂度从 O(n^3)降到 O(n)？

> 基于以下两个假设，减少不必要的计算

```
1.两个相同组件将会生成相似的DOM结构，两个不同组件将会生成不同的DOM结构。
2.对于同一层次的一组子节点，它们可以通过唯一的id进行区分。
```

> 对于假设 1: 两个相同组件，一般指的是相同的类，包含 React 官方定义的组件（View，Text）和程序员自定义的组件（这也是 React 组件化开发的一个原因，可以提升 diff 算法的效率）
>
> 对于假设 2: 一般指的是使用`map`遍历生成的列表视图或者使用`ListView/FlatList`等列表组件

React Diff 算法的实现，几乎都是基于以上两个假设进行优化的。

### React Diff 算法的实现

基于以上的两个假设，React Diff 算法的实现，主要可以分`相同类型节点的比较`、`不同节点类型的比较`、`列表节点的比较`三类情况，这里也主要针对这三类情况进行分析。

#### 1. 相同类型节点的比较

- 修改节点 A 的属性 style

![相同类型节点的比较](https://upload-images.jianshu.io/upload_images/3995013-3ea762aa423bc0a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

> 依据假设 1 的前半句 `两个相同组件将会生成相似的 DOM 结构`，由于新旧节点类型相同（tag 都是 A），DOM 结构没有发生变化，React 仅对属性（style）进行重设(将 styleBefore 改成将 styleAfter)从而实现节点的转换和界面的更新

#### 2. 不同节点类型的比较

- 将节点 A 及其子节点改成节点 D 的子节点

![不同节点类型的比较-code](https://upload-images.jianshu.io/upload_images/3995013-767f6d8411462f23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

为了方便分析，首先抽象成 DOM tree 节点模型，如下
![不同节点类型的比较-tree](https://upload-images.jianshu.io/upload_images/3995013-5aebe779ce8fd693.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

> 依据假设假设 1 的后半句`两个不同组件将会生成不同的 DOM 结构`，当发现节点已经不存在，则该节点及其子节点会被完全删除掉，不会用于进一步的比较，因此，React 会直接删除前面的节点，然后创建并插入新的节点。

在该例子中，会直接先移除根结点的左子树（即节点 A 及其子节点 B 和 C），然后再重新创建节点 A 及其子节点 B 和 C 作为根结点的右子树

#### 3. 列表节点的比较

在渲染列表节点时，它们一般都有相同的结构，只是内容有些不同而已，常见的，如使用`map`遍历生成的列表视图或者`ListView/FlatList`等列表组件，如果开发的时候没有写 key，编译器会给出警告提示，原因就是有没有添加 key，对应的 diff 算法差别很大，程序性能也会差别很大

> 依据假设 2`对于同一层次的一组子节点，它们可以通过唯一的 key 进行区分`，通过给每个节点添加唯一的 key，可以极大的简化 diff 算法，减少对 DOM 的操作。列表节点的比较主要有`添加节点`、`删除节点`、`节点排序`三种场景

##### 3.1 添加节点

- 在节点 B 与 C 之间插入节点 F

![添加节点-题目](https://upload-images.jianshu.io/upload_images/3995013-8edd756eee6da9c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

每个节点是否添加唯一标识 key 的算法实现与对比

![列表节点的比较-添加节点](https://upload-images.jianshu.io/upload_images/3995013-4b488fc909020c7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

1. 在方案一中，没有添加唯一的稳定的 key，无法定位到具体修改的位置，只能依次比较前后两个状态（即 `A、B、C、D、E` 和 `A、B、F、C、D、E`）,当比较到第三个时，发现 C 与 F 不相同，记录下来，往后依次比较，
   D 与 C，E 与 D，均不相同，也记录下来，最后加上新状态新增的一个 E 节点，一共需要对 DOM 进行 4 次操作

2. 在方案二中，如果添加了唯一的稳定的 key，则可以直接找到插入的位置，对 DOM 进行 1 次插入操作即可

3. 综上，可以发现，列表添加节点时，有唯一的稳定的 key，可以减少对 DOM 的操作，从而提升程序性能

##### 3.2 删除节点

- 删除节点 B

![删除节点](https://upload-images.jianshu.io/upload_images/3995013-6628550854532c01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

1. 在方案一中，没有添加唯一的稳定的 key，无法定位到具体修改的位置，只能依次比较前后两个状态（即 `A、B、C` 和 `A、C`）,当比较到第二个时，发现 B 与 C 不相同，记录下来，往后依次比较，
   发现新状态比旧状态少了节点 C，移除旧状态的节点 C，一共需要对 DOM 进行 2 次操作

2. 在方案二中，如果添加了唯一的稳定的 key，则可以直接找到删除的位置，对 DOM 进行 1 次删除操作即可

3. 综上，可以发现，列表删除节点时，有唯一的稳定的 key，可以减少对 DOM 的操作，从而提升程序性能

##### 3.2 排序节点

- 交换节点 B 和 C 的位置

![排序-题目](https://upload-images.jianshu.io/upload_images/3995013-e620f7438440f311.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

每个节点是否添加唯一标识 key 的算法实现与对比

![列表节点的比较-排序](https://upload-images.jianshu.io/upload_images/3995013-9b884ceab2996cf6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

1. 在方案一中，在没有添加 key 的情况下，无法定位到具体的节点，只能通过遍历，依次比较，再逐个更新。比如在该例中，要交换 shape5 和 shape6 的节点 B 和 C 的位置，执行操作如下：

```
  1.shape6 的节点 C 与 shape5 的节点 B 进行比较，节点 B 更新成节点 C

  2.shape6 的节点 B 与 shape5 的节点 C 进行比较，节点 C 更新成节点 B。
```

2. 在方案二中，在有唯一稳定的 key 的情况下，可以直接定位到具体的节点，只需要对相应的节点进行排序即可。比如在该例中，要交换 B 和 C 的位置，执行操作如下：

```
  1.只需要交换节点 B、C 的位置即可。
```

3. 综上，可以发现没有 key 的情况下，需要对 DOM 进行 2 次操作，有 key 的情况下，只需要对 DOM 进行 1 次操作即可。

> 总结：列表节点中，有唯一稳定的 key 的情况下，无论是添加节点、删除节点、排序节点，都能极大的减少对 DOM 的操作次数，提升程序的性能

最后，为什么说 key 必须是`唯一的`，且是`稳定的`呢？

```
注意事项：
1.如果key不稳定（如取index），则效果与不设置key的diff算法是一样的
2.key如果不唯一（如不同节点的key取值相同），如果列表的所有组件类型相同，则效果与不设置key的diff算法也是一样的，如果列表的组件类型不同，则可能会出现重复渲染等异常情况。
```
