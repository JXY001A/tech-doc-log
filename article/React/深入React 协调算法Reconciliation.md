<!--
 * @description: 
 * @author: JXY
 * @Date: 2019-11-01 23:39:52
 * @Email: JXY001a@aliyun.com
 * @LastEditTime: 2019-11-01 23:41:29
 -->
# （译）深入React 协调算法 Reconciliation

> [原文引用](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)


![reactFiber.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1571065095513-09bf84c7-6d15-4763-891c-96214c0997a4.png#align=left&display=inline&height=300&name=reactFiber.png&originHeight=300&originWidth=691&search=&size=18196&status=done&width=691)<br />

<a name="tNaNH"></a>
### 前言

`React`  是一个用于构建用户界面的 `JavaScript`  库。它的核心原理是追踪组件中的状态的改变，然后将这些被更新的状态自动刷新到屏幕上。在 `React` 中有一个过程被称之为 `协调 reconciliation` 。也就是当我们调用  `setState`  方法，或是框架检测到 `state` or `props`变化后，便开始重新计算比对并渲染组件 UI。<br />
<br />React 官方文档提供了对其原理的[高阶概述](https://reactjs.org/docs/reconciliation.html)：`React Element` ，生命周期方法，`render`方法，组件孩子节点的 `diff`算法的应用等 在 `React `中所扮演的角色。其中 render 方法所返回的 `React elements Tree `就是我们常常提到的  虚拟DOM `virtual DOM` 。这个术语有助于前期向人们解释 `React` ，但是它也会造成一些困惑，因为任何 `React` 的官方文档中都没出现过它。所以在这篇文章中，我将坚定称它为  `React elements` 树。<br />
<br />除以外，React 还有另外一棵内部实例树（组件实例，或 DOM 节点等），它被用来保存 state 状态。从 React16 开始 React 推出了新的内部实例树的实现 以及它的管理算法 统称为 `Fiber`。 <br />

<a name="cfqJD"></a>
### 背景知识

下面是一个非常简单的小应用，接下来的整篇文章都将使用到它。一个按钮，点击数字加一，并渲染到屏幕上。

![example.gif](https://cdn.nlark.com/yuque/0/2019/gif/250267/1571068384215-4317217f-c3de-453b-bb58-6e89e60ba26f.gif#align=left&display=inline&height=68&name=example.gif&originHeight=68&originWidth=210&search=&size=83003&status=done&width=210)<br />代码：<br />

```jsx
class ClickCounter extends React.Component {
    constructor(props) {
        super(props);
        this.state = {count: 0};
        this.handleClick = this.handleClick.bind(this);
    }

    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }


    render() {
        return [
            <button key="1" onClick={this.handleClick}>Update counter</button>,
            <span key="2">{this.state.count}</span>
        ]
    }
}
```

可以看到，这个简单的组件从 `render` 方法中返回了两个元素   `button` 和 `span` 。当你快速点击按钮的时候，组件的状态会在内部更新。这时 `span` 的文本内容也随之更新。

在 React **协调**过程中有各种各样的工作需要被执行。以上面代码为例，在 React 第一次渲染和随后的 state 更新期间的做了一些事情，大致如下：

- 更新  `ClickCounter`  组件 `state`  的  `count` 属性。
- 检索  `ClickCounter`  组件的孩子节点并对比他们的 `props`。
- 更新 `span`元素。

还有一些在**协调**阶段执行的工作，如：[生命周期方法](https://reactjs.org/docs/react-component.html#updating)调用或 [refs ](https://reactjs.org/docs/refs-and-the-dom.html) 更新等。所有这些活动在 `Fiber` 架构中被总称为 `‘work’`。`‘work’` 的类型通常基于 `React Element` 的 `type `。 例如，对类组件来说需要 React 来创建实例，相对于函数组件来说就没有必要了。众所周知，在 React 中有很的组件类型，类组件，函数组件（无状态组件），宿主组件(DOM)，`portals `等。`React Element` 的 `type `由 [createElement](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/react/src/ReactElement.js#L171) 函数的第一个参数定义。这个函数通常在 `render`方法中被使用，用于创建上面提到的 `React elements`。

在我们探索 `‘work’` 和  `Fiber`  架构算法之前，我们先熟悉下一在 React 内部使用的数据结构。

<a name="Y1Jnj"></a>
### 从 React Element 到 Fiber Node
在 `React` 中每一个组件都有其对应的 UI 呈现，我们称之为视图，或者也可以说是 `render` 方法返回的模板。例子 `ClickCounter`  组件的模板如下：

```jsx
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```

<a name="9nM8V"></a>
#### React Element 
当一个模板被传入到 `JSX` 编译器，最终会生成 `React Element`。它其实就是 `React` 组件的 `render` 方法返回的实际内容，并不是 HTML..，当然我们也可以不使用 `JSX` 语法，也可以直接像下面代码示例中展示那样写组件 ，能接受的话，哈哈哈……：

```javascript
class ClickCounter {
    //...
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2'
                },
                this.state.count
            )
        ]
    }
}
```

这里调用的  `React.createElement`  方法返回了如下所示的两个数据结构：

```json
[
    {
        $$typeof: Symbol(react.element),
        type: 'button',
        key: "1",
        props: {
            children: 'Update counter',
            onClick: () => { ... }
        }
    },
    {
        $$typeof: Symbol(react.element),
        type: 'span',
        key: "2",
        props: {
            children: 0
        }
    }
]
```

在上面代码中你可以看到，React 为两个返回的数据对象都添加了 `[$$typeof](https://overreacted.io/why-do-react-elements-have-typeof-property/)` 属性，它在后会用到，主要是用来甄别是否为合法有效的 `Element`。同时我们可看到了`type`, `key` and `props`三个属性。这里我们需要要注意一下 `React` 是如何表示 `span `和 `button` 两个节点的文本内容的，还有 `button` 节点的 `onClick` 部分。

对于示例组件 `**ClickCounter**` 来说，它自身的 `React Element` 并没有添加任何属性 或者 `key`。

```jsx
{
    $$typeof: Symbol(react.element),
    key: null,
    props: {},
    ref: null,
    type: ClickCounter
}
```

<a name="1q71G"></a>
#### Fiber nodes
协调实际上是将从组件 `render` 方法返回的每个 `React Element` 合并进入`fiber node` 树的过程。每一个 `React Element` 都有一个对应的 `fiber` 节点。不同于  `React Element` ，`fiber` 并不会在每次渲染时候都重新创建。它们是可变的数据结构，保存着组件的状态，以及对应实体 `DOM` 节点的引用，这个下面会讲到。 

我们之前提到过，`React` 会基于 `React Element` 的 `type` 属性执行不同工作。在示例程序中， `ClickCounter`  组件将调用生命周期函数 和 `render` 方法，而宿主组件  `span`  将会改变 `DOM` 内容。每一个 `React Element`  都会转换为它所对应的 `Fiber Node`，用于描述接下来要被执行的工作。

也就是说，可以认为 `fiber` 节点就是描述随后要执行工作的一种数据结构，可以类比于调用栈中的帧 ，换而言之，就是一个工作单元。`fiber` 架构同时还提供了便捷的方式去**追踪**，**调度**，**暂停**，**中断**协调进程。

当一个 ` React Element` 第一次被转换为 `fiber` 节点的时候，`React` 将会从 `React Element` 种提取数据子并在在 [`createFiberFromTypeAndProps`](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) 函数中创建一个新的 `fiber`。随后的更新过程中 `React`会复用这个创建的 `fiber` 节点，并将对应 `React Element`  被改变数据更新到这个 `fiber` 节点上。React 也会移除一些 `fiber`节点，例如：当同层级上对应 `key `属性改变时，或 `render` 方法返回的  `React Element` 没有该 `fiber` 对应的 `Element` 对象时，该 `fiber` 就会被删除。

因为 `React` 会为每个得到的 `React Element` 创建 `fiber`，这些 `fiber` 节点被连接起来组成 `fiber tree`。在我们的例子中它是这样的：<br /> <br />![fiberTree.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1571238664626-49292e0c-489f-443c-97ad-1e6a2ec6a855.png#align=left&display=inline&height=242&name=fiberTree.png&originHeight=242&originWidth=555&search=&size=17151&status=done&width=555)

所有的 `fiber` 节点都通过属性 `child`, `sibling` and `return` 连接起来。<br />

<a name="N7HS9"></a>
### current 和 workInProgress 变量
在第一次渲染完成后，`React` 最终生成 `fiber tree`，它可以理解为渲染出的 UI 界面的在内存中的映射。`fiber tree` 的引用变量通常是 `current`。当 `React` 开始**更新**操作时，它会又会构建一棵被称为 `workInProgress` 的新树。`workInProgress` 接下来将会替换 `current`变量所引用的旧的 `fiber tree`，然后随之会被刷新到屏幕上。

所有执行相关更新，删除等操作的 `fibers` 都来自于  `workInProgress` 树。当 React 遍历  `current` 树的时候，会为每个 `fiber node` 创建一个备用节点，这些备用节点最终组成整个  `workInProgress` 树。这些被新创建的 fibers 的数据来自于 `render` 方法返回的 `React Element` 。一旦更新操作的工作完成，`React` 将会拥有一颗随时便可刷新到屏幕的备用树。当  `workInProgress` 树被刷新到屏幕后，那么它就会变成  `current` 树。也就是说 `current` 变量会保存这颗新树的指针。

`React`  的一个核心原则就是一致性。所以 `React` 会一次性遍更新所有需要处理的 `DOM`，不会只显示部分结果。 `workInProgress tree` 对用户而言类似于一个不可见的草稿。所以 React 会先处理所有的组件内部状态的更新，然后再将它们改变了的部分重新刷新到屏幕上。

在源码中你会看到很多的函数从  `current` and `workInProgress` 树 中获取 `fiber` 节点。每一个 `fiber` 节点在  `alternate` 域中保存它在另一棵树中对应节点的引用。也就是说一个来自  `current` 树的节点会指向  `workInProgress` 树中相对应的节点，反之亦然。

<a name="qCGkI"></a>
### 副作用

我们可以认为一个 `React` 组件，它其实就是一个使用 `state` 和 `props` 来计算 UI 表现形式的函数。像 `DOM` 或 调用生命周期方法这样的活动都被认为是一个副作用。关于副作用的详情可以查看[官方文档](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook)。

日常开发中，其实很多时候 state  和 props 的更新都会导致副作用的产生。每种副作用的应用实际上是一种指定类型的工作，而 `fiber` 节点是一种方便的机制去追踪副作用，从它被添加直到被应用。每一个 `fiber node` 都有会有与之相关的副作用。它们都被编码在`fiber` 节点的 `effectTag`域上。

所以，`Fiber`中的副作用基本上可以理解为，定义了在更新进程完成后需要对实例执行的具体操作。对宿主组件而言，也就是 `DOM` ，的这些工作包括 `element` 的添加，更新，移除。对的类组件来说可能需要更新 `refs` 以及调用生命周期方法等。

<a name="pIrnt"></a>
### 副作用列表
`React` 的 `DOM` 更新速度非常快，为了实现这样的性能，它的实现上采用了一些有趣的技术，其中之一就是构建了一个可以高效迭代遍历的线性 `fiber node` 列表，该列表中所有 `fiber node` 都是有副作用需要被执行的，对于没有副作用的 `fiber node`  则不必浪费宝贵的时间了。

这个列表的目的是标记那些需要执行各种副作用的节点，包括 `DOM` 更新，删除，生命周期函数调用等。它同时也是  `finishedWork` 树的子集，列表中的节点之间通过  `nextEffect` 属性连接。

[Dan Abramov](https://medium.com/u/a3a8af6addc1?source=post_page-----e1c04700ef6e----------------------) 对副作用列表做了一个有趣的比喻，他认为副作用列表像一颗圣诞树，所有的副作用节点被导线绑在一起。如下图：<br />![effectslist.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1571410236544-341c8c06-2646-4643-8203-648f49764f8c.png#align=left&display=inline&height=357&name=effectslist.png&originHeight=357&originWidth=475&search=&size=20030&status=done&width=475)<br />
<br />可以看到，有副作用的节点被都连接在一起。当遍历这些节点的时候，`React` 通过 `firstEffect` 指针得到列表的起点。所以上面的图也可以理解为这样：<br />
<br />![effectsline.png](https://cdn.nlark.com/yuque/0/2019/png/250267/1571410448837-e8bcc1e7-beb3-48dd-9caf-c98feb290a1f.png#align=left&display=inline&height=103&name=effectsline.png&originHeight=103&originWidth=712&search=&size=4145&status=done&width=712)<br />
<br />通过观察，可以发现 `React` 应用副作用的顺序是从子节点到父节点。<br />

<a name="yXKJ1"></a>
### fiber tree 的根节点
每一个 `React` 应用都有一个或多个 `DOM` 元素作为容器，通常开发中那个 `id` 为 `root` 的元素。示例程序中的容器是 `id` 为 `container` 的 `div`。

```javascript
const domContainer = document.querySelector('#container');
ReactDOM.render(React.createElement(ClickCounter), domContainer);
```
 <br />React 会为每一个容器创建一个 `fiber root` 对象。 进入这个[地址](https://github.com/facebook/react/blob/0dc0ddc1ef5f90fe48b58f1a1ba753757961fc74/packages/react-reconciler/src/ReactFiberRoot.js#L31)可以看到它的具体实现。

```javascript
const fiberRoot = query('#container')._reactRootContainer._internalRoot
```

`fiber root` 是 `React` 保存  `fiber tree` 引用的地方。也就是 `fiber root` 节点上的属性  `current` 。

```javascript
const hostRootFiberNode = fiberRoot.current
```

`fiber tree` 起点是一个被称之为 `HostRoot`  的特殊类型的 `fiber node`。它在内部被创建，扮演着所有节点的祖先节点的角色。`HostRoot` 的属性 `stateNode` 反过来又指向 `FiberRoot` 。

```javascript
fiberRoot.current.stateNode === fiberRoot; // true
```

你可以探索你自己应用的中的 `fiber tree` 通过进入顶层的  `fiberRoot` ，进而得到  `HostRoot` 。或者你也可以通过组件实例得到单个 `fiber` 节点。

```javascript
compInstance._reactInternalFiber
```

<a name="suGgH"></a>
### Fiber node 结构

现在让我们瞥一眼创建的 `ClickCounter` 组件的 fiber node。

```javascript
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1,
    effectTag: 0,
    nextEffect: null
}
```

还有 `span` DOM 元素

```javascript
{
    stateNode: new HTMLSpanElement,
    type: "span",
    alternate: null,
    key: "2",
    updateQueue: null,
    memoizedState: null,
    pendingProps: {children: 0},
    memoizedProps: {children: 0},
    tag: 5,
    effectTag: 0,
    nextEffect: null
}
```

 `fiber`节点上有很多的域，`alternate`, `effectTag` 和 `nextEffect`  这几个域的用处在之前的部分已经讲解过了，现在让我们开始研究剩下的这些。

<a name="nPjDD"></a>
#### stateNode
用于保存类组件的实例，宿主组件的 DOM 实例等。通常我们也可以说这个属性是用来保存与该 `fiber` 相对应的的本地状态 。

<a name="uOQsK"></a>
#### type
定义了与该 `fiber node` 相对应的是一个函数组件还是一个类组件。如果是一个类组件该属性指向这个类的构造函数。如果是一个 DOM 元素，该属性则是与之相对应的 `HTML` 标签。使用这个域很容易就能理解与该 `fiber` 节点相关联的元素是什么。

<a name="764dS"></a>
#### tag
定义了当前 `fiber` 的类型。协调算法用它来判断具体需要做什么工作。像之前说的这个工作的类型还是基于 `React Element` 的类型。[`createFiberFromTypeAndProps`](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) 函数映射一个 React Element 到与之相对应的 fiber node 类型。在我们的示例中， `ClickCounter` 组件的属性 `tag` 是 1，表示  `ClassComponent` ， `span` 元素的 tag 是5 ，表示  `HostComponent`。

<a name="J3bhK"></a>
#### updateQueue
一个状态更新队列，包括回调 和 `DOM` 更新。

<a name="boEjN"></a>
#### memoizedState
已经被使用渲染的过的 `fiber` 状态。也就是当前屏幕上 UI  状态的映射。

<a name="zMYcy"></a>
#### memoizedProps
已经使用渲染过的 `fiber` 属性，也是构成当前屏幕 UI  状态映射的一部分。

<a name="PQqHh"></a>
#### pendingProps
保存着最近一次从 `render` 方法返回的 `React Element` 中拿到的数据，等待随后被应用到子组件或是 `DOM` 元素上。

<a name="ht6fB"></a>
#### key
相同层级孩子节点唯一标记，可以优化提升 React 对子节点更新，添加，删处的判断效率。与它具体功能相关的官方文档可以看这里。[https://reactjs.org/docs/lists-and-keys.html#keys](https://reactjs.org/docs/lists-and-keys.html#keys)

<a name="eon5W"></a>
### 通用算法
React 工作执行大致可以分为两个阶段：`render `和` commit`**。**<br />**<br />在 `render `阶段，React 通过调度  `setState` 或者 `React.render` 来实现组件更新，同时也会计算出需要被更新的那部分 UI 的状态。如果是是初始化渲染，`React` 会为 `render `方法返回的每一个 React Element 创建一个 fiber 对象。在随后的更新中，如果当时创建 `fiber `时对应的 `Reatc Element` 还存在，那么该 `fiber `还会被复用，或者更新。这个阶段最后的成果就是在这颗 `fiber`  树上对标记了出了那些有副作用的 `fiebr` 节点。在接下来的 `commit` 阶段就是处理这些被标记节点副作用的节点 ，最后呈现为可视化的 UI。

有一件很重要的事情需要理解，那就是 `render` 阶段的执行可以是异步的。React 会处理一个或多个 `fiber` 节点 ，基于可利用的有效时间，如果有效时间用完它就会停下来，亦或者让位于优先级更高的事件，例如：用户点击操作等。随后再次找到它暂停的位置处继续它未完成的工作。但有时，则需要放弃已经执行过的工作，然后从头开始。这些暂停之所以成为可能是因为它们执行的工作并不会造成用户可见的改变，像 DOM 更新之类的。相反，接下来的  `commit`  阶段则都是同步执行的。因为这个阶段所有执行的工作是**副作用**的**应用**，最主要的DOM 更新，会导致用户可见的改变。这就是为什么 `React` 需要一次性执行完所有副作用的原因。

调用生命周期方法也是副作用的一部分。一些方法在 `render` 阶段被调用，剩下的当然在 commit 阶段被调用啦。下面列举了在 `render` 阶段调用的生命周期方法：

- [UNSAFE_]componentWillMount (deprecated)
- [UNSAFE_]componentWillReceiveProps (deprecated)
- getDerivedStateFromProps
- shouldComponentUpdate
- [UNSAFE_]componentWillUpdate (deprecated)
- render

从列表中你可以看到，很多在 `render` 阶段被调用的旧的生命周期方法在 `version 16.3`  及后面的版本中都被标记上了 `UNSAFE`。现在在这篇文档中它们被统称为**遗留**生命周期方法。它们很有可能会在未来的版本中被移除。

想知道原因吗？

好吧，其实这其中的原因与我们上面学到的知识息息相关，首先我们知道 `render` 阶段的更新不会有任何影响用户视觉上可见的的副作用产生，例如: `DOM` 更新之类的，甚至 `React` 还会异步的更新组件。然而这些被使用  `UNSAFE` 标记的生命周期方法经常会被开发者错误的使用，甚至滥用。开发者倾向于将包含副作用的代码放置到这些方法中，这样就可能导致新的异步渲染被触发，例如：在 `componentWillUpdate `函数中调用 `setState` 方法就会导致错误产生。甚至程序无限循环，直至奔溃。`UNSAFE` **标记也是提醒开发者慎重使用**。

接下来我们讲讲 `commit `阶段被调用的生命周期方法：

- getSnapshotBeforeUpdate
- componentDidMount
- componentDidUpdate
- componentWillUnmount

因为这些方法的执行的执行都是同步的，所以它们可以包含很多的副作用以及 `DOM` 操作。

ok，目前我们已经掌握了足够的背景知识，接下来就可以潜入探究一番， `React` 用到的遍历和执行相关操作的算法。

<a name="TxDtt"></a>
### Render phase
**协调**算法的执行总是从顶层的 `HostRoot`节点开始执行工作，通过调用  [`renderRoot`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132) 方法开始。然而 `React` 会跳过那些被处理过的 `fiber` 节点，直到找到还未被处理的节点。例如，如果你在组件树的某一处调用了 `setState` ，`React` 会从顶部开始遍历，但是会快速的跳过它的祖先节点，直到找到触发 `setState` 的组件为止。

<a name="utBMS"></a>
#### 循环工作的主要步骤
所有的 `fiebr `节点都在在循环遍历中被处理。我们先看看循环算法实现的同步部分：

```javascript
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```

代码中的 `nextUnitOfWork` 变量保存着从 `workInProgress` 树中取出需要被处理的 `fiebr` 节点的引用。当 React 遍历这个 `Fiber `树的时候，它可以使用这个变量的引用来不断检索遍历到剩下还未被处理的 `fiber `节点。在当前的 `fiber` 节点被处理后，这个变量随后会指向下一个要被处理的 `fiber` 节点的引用，存在的话。否则为 `null`。

在遍历树和初始化的过程中重要用的的 4 个方法：

- [performUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1056)
- [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)
- [completeUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L879)
- [completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)

为了说明他们是如何被使用的，请看下面的 `fiber` 树遍历的示意动画：

![workloop.gif](https://cdn.nlark.com/yuque/0/2019/gif/250267/1571668691048-4487d2dc-402f-4d94-9ffc-10fb9de00e73.gif#align=left&display=inline&height=420&name=workloop.gif&originHeight=420&originWidth=780&search=&size=541575&status=done&width=780)

> 注意：垂直方向上节点之间通过 siblings 属性连接，折线处表示连接孩子节点，通过 children 属性。


让我们首先从 `performUnitOfWork` 和 `beginWork`这两个方法开始。

```javascript
function performUnitOfWork(workInProgress) {
    let next = beginWork(workInProgress);
    if (next === null) {
        next = completeUnitOfWork(workInProgress);
    }
    return next;
}

function beginWork(workInProgress) {
    console.log('work performed for ' + workInProgress.name);
    return workInProgress.child;
}
```

 `performUnitOfWork` 方法接受一个来自 `workInProgress`  树的 `fiber` 节点作为参数，同时调用 `beginWork` 方法。一个 `fiber `节点的所有需要执行的处理都开始于这个方法。为了证明这一点，我们简单的在日志中记录那些已经完成功能处理的 fiber 节点的名字。 `beginWork` 方法总是会返回一个指向下一个将要被处理的孩子节点的指针，或者 `null`。

如果存在下一个孩子节点，那么它将在循环中被赋值给  `nextUnitOfWork` 变量。然而要是返回了 `null`，这时 `React`  知道已经到达了分支的末端，所以一旦当前的节点处理完成，接下来就需要处理它的兄弟节点，或者返回到父节点。这些都在  `completeUnitOfWork` 函数中执行：

```javascript
function completeUnitOfWork(workInProgress) {
    while (true) {
        let returnFiber = workInProgress.return;
        let siblingFiber = workInProgress.sibling;

        nextUnitOfWork = completeWork(workInProgress);

        if (siblingFiber !== null) {
            // If there is a sibling, return it
            // to perform work for this sibling
            return siblingFiber;
        } else if (returnFiber !== null) {
            // If there's no more work in this returnFiber,
            // continue the loop to complete the parent.
            workInProgress = returnFiber;
            continue;
        } else {
            // We've reached the root.
            return null;
        }
    }
}

function completeWork(workInProgress) {
    console.log('work completed for ' + workInProgress.name);
    return null;
}
```

该函数的主体是一个大的 `while`循环。当 `workInProgress` 没有孩子节点的时候 `React` 就会进入这个函数。当完成当前遍历到的孩子节点的工作后，React 就检查是否有兄弟节点，如果有 `React` 会退出这个函数，并返回该节点兄弟节点的指针。它同样会被赋值给 `nextUnitOfWork` 变量，节点来 `React` 会重复上面的过程，直至整棵子树被遍历处理完成。当子节点以及子节点的孩子节点都被处理完成后，回溯至父节点，再重复循环父节点的兄弟节点，直至整棵树被遍历完成，最终返回到根节点  `fiberRoot` 。

<a name="WMGwn"></a>
### Commit phase
这个阶段开始于  [`completeRoot`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306)  函数。在这个里 `React `会更新 `DOM` ，调用相关生命周期方法。

当 React 进入这个阶段，它有两棵树和副作用列表。第一棵树就是是当前已经刷新到屏幕上 UI 对应的状态。另一颗备用树就是在 `render` 阶段构建的，在源码中它通常称之为`finishedWork` 或  `workInProgress` ，在接下来的 `commit` 阶段会替换之前的旧树，将新的状态刷新到屏幕上。

`finishedWork` 树的上通过 `nextEffect` 指针连接的 `fiber `节点构成副作用列表。_副作用列表可以看做是 `render `阶段运行产生的成果。_渲染的意义就是去决定节点的插入，更新，删除，或是组件生命周期函数的调用。这些就是副作用列表将要告诉我们的，也是接下来**提交阶段**需要遍历的节点集合。

在提交阶段运行的主函数是 [`commitRoot`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523)，基本上它做了如下工作：

- 在标记 `Snapshot`  副作用的节点上调用 `getSnapshotBeforeUpdate` 生命周期方法。
- 在标记了 `Deletion`  副作用的节点上调用 `componentWillUnmount` 生命周期方法。
- 执行所有的 `DOM` 插入，更新，删除操作。
- 让 `current `指针指向  `finishedWork` 树。
- 在标记了 `Placement` 副作用的组件节点上调用 `componentDidMount` 生命周期方法。
- 在标记了 `Update` 副作用的组件节点上调用 `componentDidUpdate` 生命周期方法。

在 `getSnapshotBeforeUpdate`  调用后，`React` 会提交整棵树的所有副作用。整个过程分为两步。第一步执行 `DOM` 插入，更新，删除，`ref` 的卸载。接下来 `React` 将`finishedWork` 赋值给 `FiberRoot` ，并标记 `workInProgress` 树为  `current` 树。这样做的原因是，第一步相当于是 `componentWillUnmount` 阶段，`current `指向之前的树，而接下里的第二步则相当于是 `componentDidMount/Update` 阶段，`current`要指向新树。

上面描述的主要执行函数：

```javascript
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```

每一个子函数的实现都是遍历整个副作用列表，检查副作用的类型。当它找到需要它执行的副作用时就会执行应用。

<a name="kxW39"></a>
#### 生命周期方法
下面是一个例子，这部分代码迭代了整个副作用列表，并检查循环到的节点是否有 `Snapshot`  副作用：

```javascript
function commitBeforeMutationLifecycles() {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;
        if (effectTag & Snapshot) {
            const current = nextEffect.alternate;
            commitBeforeMutationLifeCycles(current, nextEffect);
        }
        nextEffect = nextEffect.nextEffect;
    }
}
```

对于一个类组件来说，这个副作用意味着调用 `getSnapshotBeforeUpdate`  生命周期方法。

<a name="O1cV2"></a>
#### DOM 更新
React 执行DOM 更新使的是  [`commitAllHostEffects`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376)  函数。

```javascript
function commitAllHostEffects() {
    switch (primaryEffectTag) {
        case Placement: {
            commitPlacement(nextEffect);
            ...
        }
        case PlacementAndUpdate: {
            commitPlacement(nextEffect);
            commitWork(current, nextEffect);
            ...
        }
        case Update: {
            commitWork(current, nextEffect);
            ...
        }
        case Deletion: {
            commitDeletion(nextEffect);
            ...
        }
    }
}
```

非常有趣，在 `commitDeletion`  函数中 React 调用 `componentWillUnmount`  是方法作为删除处理的一部分。

<a name="QPIDx"></a>
#### 剩余生命周期方法
[`commitAllLifecycles`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L465) 函数中 React 调用了所有剩下的生命周期方法，`componentDidUpdate` ， `componentDidMount`
