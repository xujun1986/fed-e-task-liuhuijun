## 「译」React Fiber 那些事: 深入解析新的协调算法
+ 翻译自：[Inside Fiber: in-depth overview of the new reconciliation algorithm in React](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04700ef6e)

+ React 是一个用于构建用户交互界面的 JavaScript 库，其核心 机制 就是跟踪组件的状态变化，并将更新的状态映射到到新的界面。在 React 中，我们将此过程称之为协调。我们调用 setState 方法来改变状态，而框架本身会去检查 state 或 props 是否已经更改来决定是否重新渲染组件。

+ React 的官方文档对 协调机制 进行了良好的抽象描述： React 的元素、生命周期、 render 方法，以及应用于组件子元素的 diffing 算法综合起到的作用，就是协调。从 render 方法返回的不可变的 React 元素通常称为「虚拟 DOM」。这个术语有助于早期向人们解释 React，但它也引起了混乱，并且不再用于 React 文档。在本文中，我将坚持称它为 React 元素的树。

+ 除了 React 元素的树之外，框架总是在内部维护一个实例来持有状态（如组件、 DOM 节点等）。从版本 16 开始， React 推出了内部实例树的新的实现方法，以及被称之为 Fiber 的算法。如果想要了解 Fiber 架构带来的优势，可以看下 [React 在 Fiber 中使用链表的方式和原因](https://medium.com/react-in-depth/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7) 

+ 这是本系列的第一篇文章，这一系列的目的就是向你描绘出 React 的内部架构。在本文中，我希望能够提供一些与算法相关的重要概念和数据结构，并对其进行深入阐述。一旦我们有足够的背景，我们将探索用于遍历和处理 Fiber 树的算法和主要功能。本系列的下一篇文章将演示 React 如何使用该算法执行初始渲染和处理 state 以及 props 的更新。到那时，我们将继续讨论调度程序的详细信息，子协调过程以及构建 effect 列表的机制。

+ 我将给你带来一些非常高阶的知识 。我鼓励你阅读来了解 Concurrent React 的内部工作的魔力。如果您计划开始为 React 贡献代码，本系列也将为您提供很好的指导。我是 [逆向工程的坚定信徒](https://indepth.dev/level-up-your-reverse-engineering-skills/)，因此本文会有很多最新版本 16.6.0 中的源代码的链接。

+ 需要消化的内容绝对是很多的，所以如果你当下还不能很理解的话，不用感到压力。花些时间是值得的。**请注意，只是使用 React 的话，您不需要知道任何文中的内容。本文是关于 React 在内部是如何工作的。**

### 背景介绍
+ 如下是我将在整个系列中使用的一个简单的应用程序。我们有一个按钮，点击它将会使屏幕上渲染的数字加 1：
  + ![](./images/react示例1.gif)
+ 而它的实现如下：
```js
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
+ 你可以在 [这里](https://stackblitz.com/edit/react-t4rdmh) 把玩一下。如您所见，它就是一个可从 render 方法返回两个子元素 -- button 和 span 的简单组件。只要你单击该按钮，组件的状态将在处理程序内被更新，而状态的更新就会导致 span 元素内的文本更新。

+ 在协调阶段内，React 进行了各种各样的活动。例如，在我们的简单应用程序中，从第一次渲染到状态更新后的期间内，React 执行了如下高阶操作：
  + 更新了 ClickCounter 组件的内部状态的 count 属性
  + 获取和比较了 ClickCounter 组件的子组件及其 props
  + 更新 span 元素的 props
  + 更新 span 元素的 textContent 属性

+ 除了上述活动，React 在协调期间还执行了一些其他活动，如调用 生命周期方法 或更新 refs。所有这些活动在 Fiber 架构中统称为「工作」。工作类型通常取决于 React 元素的类型。例如，对于类定义的组件，React 需要创建实例，但是函数定义的组件就不必执行此操作。正如我们所了解的，React 中有许多元素类型，例如：类和函数组件，宿主组件（DOM 节点）portal 等。React 元素的类型由 [createElement](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/react/src/ReactElement.js#L171) 函数的第一个参数定义，此函数通常在 render 方法中调用以创建元素。

+ 在我们开始探索活动细节和 Fiber 算法的主要内容之前，我们首先来熟悉下 React 在内部使用的一些数据结构。

### 从 React 元素到 Fiber 节点
+ React 中的每个组件都有一个 UI 表示，我们可以称之为从 render 方法返回的一个视图或模板。这是 ClickCounter 组件的模板：
```js
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```

### React 元素
+ 如果模板经过 JSX 编译器处理，你就会得到一堆 React 元素。这是从 React 组件的 render 方法返回的，但并不是 HTML 。由于我们并没有被强制要求使用 JSX，因此我们的 ClickCounter 组件的 render 方法可以像这样重写：
```js
class ClickCounter {
    ...
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
+ render 方法中调用的 React.createElement 会产生两个如下的数据结构：
```js
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

+ 可以看到，React 为这些对象添加了 $$typeof 属性，从而将它们唯一地标识为 React 元素。此外我们还有属性 type、key 和 props 来描述元素。这些值取自你传递给 React.createElement 函数的参数。请注意React 如何将文本内容表示为 span 和 button 节点的子项，以及 click 钩子如何成为 button 元素 props 的一部分。 React 元素上还有其他字段，如 ref 字段，而这超出了本文的范围。

+ 而 ClickCounter 的 React 元素就没有什么 props 或 key 属性:
```js
{
    $$typeof: Symbol(react.element),
    key: null,
    props: {},
    ref: null,
    type: ClickCounter
}
```

### Fiber 节点
+ 在协调期间，从 render 方法返回的每个 React 元素的数据都会被合并到 Fiber 节点树中。每个 React 元素都有一个相应的 Fiber 节点。与 React 元素不同，不会在每次渲染时重新创建这些 Fiber 。这些是持有组件状态和 DOM 的可变数据结构。

+ 我们之前讨论过，根据不同 React 元素的类型，框架需要执行不同的活动。在我们的示例应用程序中，对于类组件 ClickCounter ，它调用生命周期方法和 render 方法，而对于 span 宿主组件（DOM 节点），它进行得是 DOM 修改。因此，每个 React 元素都会转换为 [相应类型](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js) 的 Fiber 节点，用于描述需要完成的工作。

+ **您可以将 Fiber 视为表示某些要做的工作的数据结构，或者说，是一个工作单位。Fiber 的架构还提供了一种跟踪、规划、暂停和销毁工作的便捷方式。**

+ 当 React 元素第一次转换为 Fiber 节点时，React 在 [createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) 函数中使用元素中的数据来创建 Fiber。在随后的更新中，React 会再次利用 Fiber 节点，并使用来自相应 React 元素的数据更新必要的属性。如果不再从 render 方法返回相应的 React 元素，React 可能还需要根据 key 属性来移动或删除层级结构中的节点。

+ 查看 [ChildReconciler](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L239) 函数以查看 React 为现有 Fiber 节点执行的所有活动和相应函数的列表。

+ 因为React为每个 React 元素创建一个 Fiber 节点，并且因为我们有一个这些元素组成的树，所以我们可以得到一个 Fiber 节点树。对于我们的示例应用程序，它看起来像这样：
  + ![](./images/fiber节点树.png)

+ 所有 Fiber 节点都通过链表连接，具体是使用Fiber节点上的 child、sibling 和 return 属性。至于它为什么以这种方式工作，如果您还没有阅读过我的文章，更多详细信息请查看 [React 在 Fiber 中使用链表的方法和原因](https://medium.com/react-in-depth/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

#### current 树及 workInProgress 树
+ 在第一次渲染之后，React 最终得到一个 Fiber 树，它反映了用于渲染 UI 的应用程序的状态。这棵树通常被称为 **current 树（当前树）**。当 React 开始处理更新时，它会构建一个所谓的 **workInProgress 树（工作过程树）**，它反映了要刷新到屏幕的未来状态。

+ 所有工作都在 workInProgress 树的 Fiber 节点上执行。当 React 遍历 current 树时，对于每个现有 Fiber 节点，React 会创建一个构成 workInProgress 树的备用节点，这一节点会使用 render 方法返回的 React 元素中的数据来创建。处理完更新并完成所有相关工作后，React 将准备好一个备用树以刷新到屏幕。一旦这个 workInProgress 树在屏幕上呈现，它就会变成 current 树。

+ React 的核心原则之一是一致性。 React 总是一次性更新 DOM - 它不会显示部分中间结果。workInProgress 树充当用户不可见的「草稿」，这样 React 可以先处理所有组件，然后将其更改刷新到屏幕。

+ 在源代码中，您将看到很多函数从 current 和 workInProgress 树中获取 Fiber 节点。这是一个这类函数的签名：
```js
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```

+ 每个Fiber节点持有备用域在另一个树的对应部分的引用。来自 current 树中的节点会指向 workInProgress 树中的节点，反之亦然。

#### 副作用
+ 我们可以将 React 中的一个组件视为一个使用 state 和 props 来计算 UI 表示的函数。其他所有活动，如改变 DOM 或调用生命周期方法，都应该被视为副作用，或者简单地说是一种效果。[文档中](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook) 是这样描述的：

+ 您之前可能已经在 React 组件中执行数据提取，订阅或手动更改 DOM。我们将这些操作称为“副作用”（或简称为“效果”），因为它们会影响其他组件，并且在渲染过程中无法完成。

+ 您可以看到大多 state 和 props 更新都会导致副作用。既然使用副作用是工作（活动）的一种类型，Fiber 节点是一种方便的机制来跟踪除了更新以外的效果。每个 Fiber 节点都可以具有与之相关的副作用，它们可在 effectTag 字段中编码。

+ 因此，Fiber 中的副作用基本上定义了处理更新后需要为实例完成的 [工作](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)。对于宿主组件（DOM 元素），所谓的工作包括添加，更新或删除元素。对于类组件，React可能需要更新 refs 并调用 componentDidMount 和 componentDidUpdate 生命周期方法。对于其他类型的 Fiber ，还有相对应的其他副作用。

#### 副作用列表
+ React 处理更新元素非常迅速，为了达到这种水平的性能，它采用了一些有趣的技术。**其中之一是构建具有副作用的 Fiber 节点的线性列表，从而能够快速遍历。**遍历线性列表比树快得多，并且没有必要在没有副作用的节点上花费时间。

+ 此列表的目标是标记具有 DOM 更新或其他相关副作用的节点。此列表是 finishedWork 树的子集，并使用 nextEffect 属性而不是 current 和 workInProgress 树中使用的 child 属性进行链接。

+ Dan Abramov 为副作用列表提供了一个类比。他喜欢将它想象成一棵圣诞树，「圣诞灯」将所有有效节点捆绑在一起。为了使这个可视化，让我们想象如下的 Fiber 节点树，其中标亮的节点有一些要做的工作。例如，我们的更新导致 c2 被插入到 DOM 中，d2 和 c1 被用于更改属性，而 b2 被用于触发生命周期方法。副作用列表会将它们链接在一起，以便 React 稍后可以跳过其他节点：
+ ![](./images/节点操作.png)
+ 可以看到具有副作用的节点是如何链接在一起的。当遍历节点时，React 使用 firstEffect 指针来确定列表的开始位置。所以上面的图表可以表示为这样的线性列表：
+ ![](./images/副作用线性列表.png)
+ 如您所见，React 按照从子到父的顺序应用副作用。

#### Fiber 树的根节点
+ 每个 React 应用程序都有一个或多个充当容器的 DOM 元素。在我们的例子中，它是带有 ID 为 container 的 div 元素。React 为每个容器创建一个 [Fiber 根](https://github.com/facebook/react/blob/0dc0ddc1ef5f90fe48b58f1a1ba753757961fc74/packages/react-reconciler/src/ReactFiberRoot.js#L31) 对象。您可以使用对 DOM 元素的引用来访问它：
```js
const fiberRoot = query('#container')._reactRootContainer._internalRoot
```
+ 这个 Fiber 根是React保存对 Fiber 树的引用的地方，它存储在 Fiber 根对象的 current 属性中：
```js
const hostRootFiberNode = fiberRoot.current
```
+ Fiber 树以 [一个特殊类型](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/shared/ReactWorkTags.js#L34) 的 Fiber 节点 HostRoot 开始。它在内部创建的，并充当最顶层组件的父级。HostRoot 节点可通过 stateNode 属性返回到 FiberRoot：
```js
fiberRoot.current.stateNode === fiberRoot; // true
```
+ 你可以通过 Fiber 根访问最顶层的 HostRoot 节点来探索 Fiber 树，或者可以从组件实例中获取单独的 Fiber 节点，如下所示：
```js
compInstance._reactInternalFiber
```

#### Fiber 节点结构
+ 现在让我们看一下为 ClickCounter 组件创建的 Fiber 节点的结构
```js
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
+ 以及 span DOM 元素：
```js
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
+ Fiber 节点上有很多字段。我在前面的部分中描述了字段 alternate、 effectTag 和 nextEffect 的用途。现在让我们看看为什么需要其他字段。

#### stateNode
+ 保存组件的类实例、DOM 节点或与 Fiber 节点关联的其他 React 元素类型的引用。总的来说，我们可以认为该属性用于保持与一个 Fiber 节点相关联的局部状态。

#### type
+ 定义此 Fiber 节点的函数或类。对于类组件，它指向构造函数，对于 DOM 元素，它指定 HTML 标记。我经常使用这个字段来理解 Fiber 节点与哪个元素相关。










