

## React理念

官方说明：

> 我们认为，React 是用 JavaScript 构建**`快速响应`**的大型 Web 应用程序的首选方式。它在 Facebook 和 Instagram 上表现优秀。

关键是实现`快速响应`

制约快速响应的因素为：

+ CPU瓶颈
+ IO瓶颈

### CPU瓶颈

JS可以操作DOM，`GUI渲染线程`与`JS线程`是互斥的。所以**JS脚本执行**和**浏览器布局、绘制**不能同时执行。

在每16.6ms时间内，需要完成如下工作：**JS脚本执行 -----  样式布局 ----- 样式绘制**

当JS执行时间过长，超出了16.6ms，这次刷新就没有时间执行**样式布局**和**样式绘制**了。

**React的解决办法：**

> 在浏览器每一帧的时间中，预留一些时间给JS线程，`React`利用这部分时间更新组件（预留的初始时间为5ms）

当预留的时间不够用时，`React`将线程控制权交还给浏览器使其有时间渲染UI，`React`则等待下一帧时间到来继续被中断的工作。

> 这种将长任务分拆到每一帧中，像蚂蚁搬家一样一次执行一小段任务的操作，被称为`时间切片`（time slice）

所以，解决`CPU瓶颈`的关键是实现`时间切片`，而`时间切片`的关键是：将**同步的更新**变为**可中断的异步更新**。

### IO瓶颈

如何在`网络延迟`客观存在的情况下，减少用户对`网络延迟`的感知？

将人机交互研究的结果整合到真实的UI中。为此，`React`实现了**Suspense**功能及配套的`hook`-**useDeferredValue**



## React15架构

React15架构可以分为两层：

- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

### Reconciler（协调器）

在`React`中可以通过`this.setState`、`this.forceUpdate`、`ReactDOM.render`等API触发更新。

每当有更新发生时，**Reconciler**会做如下工作：

- 调用函数组件、或class组件的`render`方法，将返回的JSX转化为虚拟DOM
- 将虚拟DOM和上次更新时的虚拟DOM对比
- 通过对比找出本次更新中变化的虚拟DOM
- 通知**Renderer**将变化的虚拟DOM渲染到页面上

### Renderer（渲染器）

由于`React`支持跨平台，所以不同平台有不同的**Renderer**。前端最熟悉的是负责在浏览器环境渲染的**Renderer** —— ReactDOM。

除此之外，还有：

- **ReactNative**渲染器，渲染App原生组件
- **ReactTest**渲染器，渲染出纯Js对象用于测试
- **ReactArt **渲染器，渲染到Canvas, SVG 或 VML (IE8)

在每次更新发生时，**Renderer**接到**Reconciler**通知，将变化的组件渲染在当前宿主环境。

---

### 架构缺点

React15会通过递归的方式更新子组件，由于递归执行，所以更新一旦开始，中途就无法中断。当层级很深时，递归更新时间超过了16ms，用户交互就会变得卡顿。



## 新的架构（16、17）

React16架构可以分为三层：

- Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入**Reconciler**
- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

### Scheduler（调度器）

既然我们以浏览器是否有剩余时间作为任务中断的标准，那么我们需要一种机制，当浏览器有剩余时间时通知我们。

其实部分浏览器已经实现了这个API，这就是requestIdleCallback。但是由于以下因素，`React`放弃使用：

- 浏览器兼容性
- 触发频率不稳定，受很多因素影响。比如当我们的浏览器切换tab后，之前tab注册的`requestIdleCallback`触发的频率会变得很低

基于以上原因，`React`实现了功能更完备的`requestIdleCallback`polyfill，这就是**Scheduler**。除了在空闲时触发回调的功能外，**Scheduler**还提供了多种调度优先级供任务设置。

### Reconciler（协调器）

更新工作从递归变成了可以中断的循环过程。每次循环都会调用`shouldYield`判断当前是否有剩余时间。

```js
/** @noinline */
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

在React16中，**Reconciler**与**Renderer**不再是交替工作。当**Scheduler**将任务交给**Reconciler**后，**Reconciler**会为变化的虚拟DOM打上代表增/删/更新的标记，类似这样：

```js
export const Placement = /*             */ 0b0000000000010;
export const Update = /*                */ 0b0000000000100;
export const PlacementAndUpdate = /*    */ 0b0000000000110;
export const Deletion = /*              */ 0b0000000001000;
```

整个**Scheduler**与**Reconciler**的工作都在内存中进行。只有当所有组件都完成**Reconciler**的工作，才会统一交给**Renderer**。

### Renderer（渲染器）

**Renderer**根据**Reconciler**为虚拟DOM打的标记，同步执行对应的DOM操作。



## 代数效应

`代数效应`是`函数式编程`中的一个概念，用于将`副作用`从`函数`调用中分离。

React中的代数效应体现为`hooks`



## Fiber

`Fiber`并不是计算机术语中的新名词，他的中文翻译叫做`纤程`，与进程（Process）、线程（Thread）、协程（Coroutine）同为程序执行过程。

`React Fiber`可以理解为：

`React`内部实现的一套状态更新机制。支持任务不同`优先级`，可中断与恢复，并且恢复后可以复用之前的`中间状态`。

其中每个任务更新单元为`React Element`对应的`Fiber节点`。

### Fiber结构

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  // Fiber对应组件的类型 Function/Class/Host...
  this.tag = tag;
  // key属性
  this.key = key;
  // 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
  this.elementType = null;
  // 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName
  this.type = null;
  // Fiber对应的真实DOM节点
  this.stateNode = null;

  // 用于连接其他Fiber节点形成Fiber树
  // 指向父级Fiber节点
	this.return = null;
	// 指向子Fiber节点
	this.child = null;
	// 指向右边第一个兄弟Fiber节点
	this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态的工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;
	// 保存本次更新会造成的DOM操作
  this.effectTag = NoEffect;
  this.nextEffect = null;

  this.firstEffect = null;
  this.lastEffect = null;

  // 调度优先级相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 指向该fiber在另一次更新时对应的fiber
  this.alternate = null;
}
```

### 双缓存

> **在内存中构建并直接替换**的技术

`React`使用“双缓存”来完成`Fiber树`的构建与替换——对应着`DOM树`的创建与更新。

#### 双缓存fiber树

在`React`中最多会同时存在两棵`Fiber树`。当前屏幕上显示内容对应的`Fiber树`称为`current Fiber树`，正在内存中构建的`Fiber树`称为`workInProgress Fiber树`。

`current Fiber树`中的`Fiber节点`被称为`current fiber`，`workInProgress Fiber树`中的`Fiber节点`被称为`workInProgress fiber`，他们通过`alternate`属性连接。

```js
currentFiber.alternate === workInProgressFiber;
workInProgressFiber.alternate === currentFiber;
```

`React`应用的根节点通过使`current`指针在不同`Fiber树`的`rootFiber`间切换来完成`current Fiber`树指向的切换。

即当`workInProgress Fiber树`构建完成交给`Renderer`渲染在页面上后，应用根节点的`current`指针指向`workInProgress Fiber树`，此时`workInProgress Fiber树`就变为`current Fiber树`。

每次状态更新都会产生新的`workInProgress Fiber树`，通过`current`与`workInProgress`的替换，完成`DOM`更新。

> FiberRootNode -> rootFiber -> ...



## render阶段

### 递-阶段

首先从rootFiber开始向下深度优先遍历。为遍历到的每个Fiber节点调用beginWork方法。

该方法会根据传入的Fiber节点创建子Fiber节点，并将这两个Fiber节点连接起来。

当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段。

### 归-阶段

在“归”阶段会调用completeWork处理Fiber节点。

当某个Fiber节点执行完completeWork，如果其存在兄弟Fiber节点（即fiber.sibling !== null），会进入其兄弟Fiber的“递”阶段。

如果不存在兄弟Fiber，会进入父级Fiber的“归”阶段。

“递”和“归”阶段会交错执行直到“归”到rootFiber。至此，render阶段的工作就结束了。

### beginWork

```jsx
ReactDOM.render(
  <Demo />,
  document.getElementById('root')
);

export default class Demo extends React.Component<any, any> {
    public render(): React.ReactNode {
        return (
            <div id="APP">
                <span>nihao</span>
                <p>
                    zhongguo
                </p>
            		wangyu
            </div>
        )
    }
}
```

beginWork执行（深度优先遍历）

> rootFiber(child) <-> (return)-Demo-(child) <-> (return)-div-(child) <-> (return)-span-(child) <-> nihao(return)

completeWork执行

> nihao(return) <-> span-(sibling) <-> p(child) <-> zhongguo(return) -> p -> div -> Demo ...... 

- 当执行到没有子节点的节点时就会调用completeWork，之后依次往上寻找父节点的兄弟节点等操作
- 如果存在兄弟节点，那就是beginWork和completeWork交替执行的过程

- 当节点只有唯一文本节点时，react提供的优化不会为其生成自己的fiber节点

通过current === null ?来区分组件是处于mount还是update。

beginWork的工作可以分为两部分：

- update时：如果current存在，在满足一定条件时可以复用current节点，这样就能克隆current.child作为workInProgress.child，而不需要新建workInProgress.child。
- mount时：除fiberRootNode以外，current === null。会根据fiber.tag不同，创建不同类型的子Fiber节点

```jsx
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // update时：如果current存在可能存在优化路径，可以复用current（即上一次更新的Fiber节点）
  if (current !== null) {
    // ...省略

    // 复用current
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderLanes,
    );
  } else {
    didReceiveUpdate = false;
  }

  // mount时：根据tag不同，创建不同的子Fiber节点
  switch (workInProgress.tag) {
    case IndeterminateComponent: 
      // ...省略
    case LazyComponent: 
      // ...省略
    case FunctionComponent: 
      // ...省略
    case ClassComponent: 
      // ...省略
    case HostRoot:
      // ...省略
    case HostComponent:
      // ...省略
    case HostText:
      // ...省略
    // ...省略其他类型
  }
}
```

- current：当前组件对应的Fiber节点在上一次更新时的Fiber节点，即workInProgress.alternate
- workInProgress：当前组件对应的Fiber节点

- renderLanes：优先级相关

#### reconcileChildren

- 对于mount的组件，他会创建新的子Fiber节点
- 对于update的组件，他会将当前组件与该组件在上次更新时对应的Fiber节点比较（也就是俗称的Diff算法），将比较的结果生成新Fiber节点

```jsx
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    // 对于mount的组件
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // 对于update的组件
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

> mountChildFibers与reconcileChildFibers这两个方法的逻辑基本一致。唯一的区别是：reconcileChildFibers会为生成的Fiber节点带上effectTag属性，而mountChildFibers不会。



![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632874736160-235ed54e-c263-4a58-844e-ebe2dc72f82f.png)



#### mount

```js
// mount时：根据tag不同，创建不同的Fiber节点
switch (workInProgress.tag) {
  case IndeterminateComponent: 
    // ...省略
  case LazyComponent: 
    // ...省略
  case FunctionComponent: 
    // ...省略
  case ClassComponent: 
    // ...省略
  case HostRoot:
    // ...省略
  case HostComponent:
    // ...省略
  case HostText:
    // ...省略
  // ...省略其他类型
}
```

#### update

满足如下情况时`didReceiveUpdate === false`（即可以直接复用前一次更新的`子Fiber`，不需要新建`子Fiber`）

1. `oldProps === newProps && workInProgress.type === current.type`，即`props`与`fiber.type`不变
2. `!includesSomeLane(renderLanes, updateLanes)`，即当前`Fiber节点`优先级不够

```js
if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      switch (workInProgress.tag) {
        // 省略处理
      }
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderLanes,
      );
    } else {
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }
```

#### effectTag

`render阶段`的工作是在内存中进行，当工作结束后会通知`Renderer`需要执行的`DOM`操作。要执行`DOM`操作的具体类型就保存在`fiber.effectTag`中。

```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
...
```

> 通过二进制表示`effectTag`，可以方便的使用位操作为`fiber.effectTag`赋值多个`effect`。

那么，如果要通知`Renderer`将`Fiber节点`对应的`DOM节点`插入页面中，需要满足两个条件：

1. `fiber.stateNode`存在，即`Fiber节点`中保存了对应的`DOM节点`
2. `(fiber.effectTag & Placement) !== 0`，即`Fiber节点`存在`Placement effectTag`



### completeWork

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      // ...省略
      return null;
    }
    case HostRoot: {
      // ...省略
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      // ...省略
      return null;
    }
  // ...省略
```

和`beginWork`一样，我们根据`current === null ?`判断是`mount`还是`update`。

同时针对`HostComponent`，判断`update`时我们还需要考虑`workInProgress.stateNode != null ?`（即该`Fiber节点`是否存在对应的`DOM节点`）

```js
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;

  if (current !== null && workInProgress.stateNode != null) {
    // update的情况
    // ...省略
  } else {
    // mount的情况
    // ...省略
  }
  return null;
}
```

#### mount阶段

- 为`Fiber节点`生成对应的`DOM节点`
- 将子孙`DOM节点`插入刚生成的`DOM节点`中
- 与`update`逻辑中的`updateHostComponent`类似的处理`props`的过程

```js
// mount的情况

// ...省略服务端渲染相关逻辑

const currentHostContext = getHostContext();
// 为fiber创建对应DOM节点
const instance = createInstance(
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
    workInProgress,
  );
// 将子孙DOM节点插入刚生成的DOM节点中
appendAllChildren(instance, workInProgress, false, false);
// DOM节点赋值给fiber.stateNode
workInProgress.stateNode = instance;

// 与update逻辑中的updateHostComponent类似的处理props的过程
if (
  finalizeInitialChildren(
    instance,
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
  )
) {
  markUpdate(workInProgress);
}
```

`mount`时只会在`rootFiber`存在`Placement effectTag`。那么`commit阶段`是如何通过一次插入`DOM`操作（对应一个`Placement effectTag`）将整棵`DOM树`插入页面的呢？

原因就在于`completeWork`中的`appendAllChildren`方法。

由于`completeWork`属于“归”阶段调用的函数，每次调用`appendAllChildren`时都会将已生成的子孙`DOM节点`插入当前生成的`DOM节点`下。那么当“归”到`rootFiber`时，我们已经有一个构建好的离屏`DOM树`。

（在`reconcileChildren`的时候，rootFiber是存在alternate的，即rootFiber存在对应的current Fiber，所以rootFiber会走`reconcileChildFibers`的逻辑，所以`shouldTrackSideEffects`等于true会追踪副作用，最后为rootFiber打上`Placement`的`effectTag`，然后将dom一次性插入，提高性能。）

#### update阶段

当`update`时，`Fiber节点`已经存在对应`DOM节点`，所以不需要生成`DOM节点`。需要做的主要是处理`props`，比如：

- `onClick`、`onChange`等回调函数的注册
- 处理`style prop`
- 处理`DANGEROUSLY_SET_INNER_HTML prop`
- 处理`children prop`

最主要的逻辑是调用`updateHostComponent`方法。

```js
if (current !== null && workInProgress.stateNode != null) {
  // update的情况
  updateHostComponent(
    current,
    workInProgress,
    type,
    newProps,
    rootContainerInstance,
  );
}
```

在`updateHostComponent`内部，被处理完的`props`会被赋值给`workInProgress.updateQueue`，并最终会在`commit阶段`被渲染在页面上。

```ts
workInProgress.updateQueue = (updatePayload: any);
```

其中`updatePayload`为数组形式，他的偶数索引的值为变化的`prop key`，奇数索引的值为变化的`prop value`。

#### effectList

作为`DOM`操作的依据，`commit阶段`需要找到所有有`effectTag`的`Fiber节点`并依次执行`effectTag`对应操作。难道需要在`commit阶段`再遍历一次`Fiber树`寻找`effectTag !== null`的`Fiber节点`么？？？

> 为了解决这个问题，在`completeWork`的上层函数`completeUnitOfWork`中，每个执行完`completeWork`且存在`effectTag`的`Fiber节点`会被保存在一条被称为`effectList`的单向链表中。
>
> `effectList`中第一个`Fiber节点`保存在`fiber.firstEffect`，最后一个元素保存在`fiber.lastEffect`。
>
> 类似`appendAllChildren`，在“归”阶段，所有有`effectTag`的`Fiber节点`都会被追加在`effectList`中，最终形成一条以`rootFiber.firstEffect`为起点的单向链表。
>
> 这样，在`commit阶段`只需要遍历`effectList`就能执行所有`effect`了。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633739447486-482e22dd-6673-4aad-b421-0bf1f9587e22.png)

## commit阶段

`commitRoot`方法是`commit阶段`工作的起点。`fiberRootNode`会作为传参。

在`rootFiber.firstEffect`上保存了一条需要执行`副作用`的`Fiber节点`的单向链表`effectList`，这些`Fiber节点`的`updateQueue`中保存了变化的`props`。

这些`副作用`对应的`DOM操作`在`commit`阶段执行。

除此之外，一些生命周期钩子（比如`componentDidXXX`）、`hook`（比如`useEffect`）需要在`commit`阶段执行。

`Renderer`工作的阶段被称为`commit`阶段。`commit`阶段可以分为三个子阶段：

- before mutation阶段（执行`DOM`操作前）
- mutation阶段（执行`DOM`操作）
- layout阶段（执行`DOM`操作后）

```js
function commitRoot(root) {
  var renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority$1(ImmediatePriority$1, commitRootImpl.bind(null, root, renderPriorityLevel));
  return null;
}
```

`commitRoot`函数真正执行的是`commitRootImpl`方法

### 整体流程

#### before mutation之前

`commitRootImpl`方法中直到第一句`if (firstEffect !== null)`之前属于`before mutation`之前。

```js
do {
    // 触发useEffect回调与其他同步任务。由于这些任务可能触发新的渲染，所以这里要一直遍历执行直到没有任务
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);

  // root指 fiberRootNode
  // root.finishedWork指当前应用的rootFiber
  const finishedWork = root.finishedWork;

  // 凡是变量名带lane的都是优先级相关
  const lanes = root.finishedLanes;
  if (finishedWork === null) {
    return null;
  }
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  // 重置Scheduler绑定的回调函数
  root.callbackNode = null;
  root.callbackId = NoLanes;

  let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
  // 重置优先级相关变量
  markRootFinished(root, remainingLanes);

  // 清除已完成的discrete updates，例如：用户鼠标点击触发的更新。
  if (rootsWithPendingDiscreteUpdates !== null) {
    if (
      !hasDiscreteLanes(remainingLanes) &&
      rootsWithPendingDiscreteUpdates.has(root)
    ) {
      rootsWithPendingDiscreteUpdates.delete(root);
    }
  }

  // 重置全局变量
  if (root === workInProgressRoot) {
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  } else {
  }

  // 将effectList赋值给firstEffect
  // 由于每个fiber的effectList只包含他的子孙节点
  // 所以根节点如果有effectTag则不会被包含进来
  // 所以这里将有effectTag的根节点插入到effectList尾部
  // 这样才能保证有effect的fiber都在effectList中
  let firstEffect;
  if (finishedWork.effectTag > PerformedWork) {
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // 根节点没有effectTag
    firstEffect = finishedWork.firstEffect;
  }
```

可以看到，`before mutation`之前主要做一些变量赋值，状态重置的工作。

#### layout之后

```js
const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

// useEffect相关
if (rootDoesHavePassiveEffects) {
  rootDoesHavePassiveEffects = false;
  rootWithPendingPassiveEffects = root;
  pendingPassiveEffectsLanes = lanes;
  pendingPassiveEffectsRenderPriority = renderPriorityLevel;
} else {}

// 性能优化相关
if (remainingLanes !== NoLanes) {
  if (enableSchedulerTracing) {
    // ...
  }
} else {
  // ...
}

// 性能优化相关
if (enableSchedulerTracing) {
  if (!rootDidHavePassiveEffects) {
    // ...
  }
}

// ...检测无限循环的同步任务
if (remainingLanes === SyncLane) {
  // ...
} 

// 在离开commitRoot函数前调用，触发一次新的调度，确保任何附加的任务被调度
ensureRootIsScheduled(root, now());

// ...处理未捕获错误及老版本遗留的边界问题


// 执行同步任务，这样同步任务不需要等到下次事件循环再执行
// 比如在 componentDidMount 中执行 setState 创建的更新会在这里被同步执行
// 或useLayoutEffect
flushSyncCallbackQueue();

return null;
```

主要包括三点内容：

1. `useEffect`相关的处理。
2. 性能追踪相关。
3. 在`commit`阶段会触发一些生命周期钩子（如 `componentDidXXX`）和`hook`（如`useLayoutEffect`、`useEffect`）。

### before mutation阶段

```js
// 保存之前的优先级，以同步优先级执行，执行完毕后恢复之前优先级
const previousLanePriority = getCurrentUpdateLanePriority();
setCurrentUpdateLanePriority(SyncLanePriority);

// 将当前上下文标记为CommitContext，作为commit阶段的标志
const prevExecutionContext = executionContext;
executionContext |= CommitContext;

// 处理focus状态
focusedInstanceHandle = prepareForCommit(root.containerInfo);
shouldFireAfterActiveInstanceBlur = false;

// beforeMutation阶段的主函数
commitBeforeMutationEffects(finishedWork);

focusedInstanceHandle = null;
```

#### commitBeforeMutationEffects

```js
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;

    if (!shouldFireAfterActiveInstanceBlur && focusedInstanceHandle !== null) {
      // ...focus blur相关
    }

    const effectTag = nextEffect.effectTag;

    // 调用getSnapshotBeforeUpdate
    if ((effectTag & Snapshot) !== NoEffect) {
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }

    // 调度useEffect
    if ((effectTag & Passive) !== NoEffect) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

整体可以分为三部分：

1. 处理`DOM节点`渲染/删除后的 `autoFocus`、`blur` 逻辑。
2. 调用`getSnapshotBeforeUpdate`生命周期钩子。
3. 调度`useEffect`。

#### 调用getSnapshotBeforeUpdate

`commitBeforeMutationEffectOnFiber`是`commitBeforeMutationLifeCycles`的别名。

在该方法内会调用`getSnapshotBeforeUpdate`。

```js
function commitBeforeMutationLifeCycles(current, finishedWork) {
	switch (finishedWork.tag) {
      // ...
    case ClassComponent:
        {
          if (finishedWork.flags & Snapshot) {
            if (current !== null) {
              var prevProps = current.memoizedProps;
              var prevState = current.memoizedState;
              var instance = finishedWork.stateNode;
              // ...
              var snapshot = instance.getSnapshotBeforeUpdate(finishedWork.elementType === finishedWork.type ? prevProps : resolveDefaultProps(finishedWork.type, prevProps), prevState);
              // ...
              instance.__reactInternalSnapshotBeforeUpdate = snapshot;
            }
          }
          return;
        }
  }
  // ...
}
```

从`React`v16开始，`componentWillXXX`钩子前增加了`UNSAFE_`前缀。

究其原因，是因为`Stack Reconciler`重构为`Fiber Reconciler`后，`render阶段`的任务可能中断/重新开始，对应的组件在`render阶段`的生命周期钩子（即`componentWillXXX`）可能触发多次。

这种行为和`React`v15不一致，所以标记为`UNSAFE_`。

为此，`React`提供了替代的生命周期钩子`getSnapshotBeforeUpdate`。

`getSnapshotBeforeUpdate`是在`commit阶段`内的`before mutation阶段`调用的，由于`commit阶段`是同步的，所以不会遇到多次调用的问题。

#### 调度`useEffect`

```js
// 调度useEffect
if ((effectTag & Passive) !== NoEffect) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    scheduleCallback(NormalSchedulerPriority, () => {
      // 触发useEffect
      flushPassiveEffects();
      return null;
    });
  }
}
```

**如何异步调度**

在`flushPassiveEffects`方法内部会从全局变量`rootWithPendingPassiveEffects`获取`effectList`。

effect副作用包括:

- 插入`DOM节点`（Placement）
- 更新`DOM节点`（Update）
- 删除`DOM节点`（Deletion）

除此外，当一个`FunctionComponent`含有`useEffect`或`useLayoutEffect`，他对应的`Fiber节点`也会被赋值`effectTag`。

在`flushPassiveEffects`方法内部会遍历`rootWithPendingPassiveEffects`（即`effectList`）执行`effect`回调函数。

整个`useEffect`异步调用分为三步：

1. `before mutation阶段`在`scheduleCallback`中调度`flushPassiveEffects`
2. `layout阶段`之后将`effectList`赋值给`rootWithPendingPassiveEffects`
3. `scheduleCallback`触发`flushPassiveEffects`，`flushPassiveEffects`内部遍历`rootWithPendingPassiveEffects`

**为什么需要异步调用**

> 与 componentDidMount、componentDidUpdate 不同的是，在浏览器完成布局与绘制之后，传给 useEffect 的函数会延迟调用。这使得它适用于许多常见的副作用场景，比如设置订阅和事件处理等情况，因此不应在函数中执行阻塞浏览器更新屏幕的操作。

`useEffect`异步执行的原因主要是防止同步执行时阻塞浏览器渲染。



### mutation阶段

类似`before mutation阶段`，`mutation阶段`也是遍历`effectList`，执行函数。这里执行的是`commitMutationEffects`。

```js
nextEffect = firstEffect;
do {
  try {
      commitMutationEffects(root, renderPriorityLevel);
    } catch (error) {
      invariant(nextEffect !== null, 'Should be working on an effect.');
      captureCommitPhaseError(nextEffect, error);
      nextEffect = nextEffect.nextEffect;
    }
} while (nextEffect !== null);
```

#### commitMutationEffects

```js
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  // 遍历effectList
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;

    // 根据 ContentReset effectTag重置文字节点
    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }

    // 更新ref
    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // 根据 effectTag 分别处理
    const primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    switch (primaryEffectTag) {
      // 插入DOM
      case Placement: {
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        break;
      }
      // 插入DOM 并 更新DOM
      case PlacementAndUpdate: {
        // 插入
        commitPlacement(nextEffect);

        nextEffect.effectTag &= ~Placement;

        // 更新
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // SSR
      case Hydrating: {
        nextEffect.effectTag &= ~Hydrating;
        break;
      }
      // SSR
      case HydratingAndUpdate: {
        nextEffect.effectTag &= ~Hydrating;

        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 更新DOM
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 删除DOM
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

`commitMutationEffects`会遍历`effectList`，对每个`Fiber节点`执行如下三个操作：

1. 根据`ContentReset effectTag`重置文字节点
2. 更新`ref`
3. 根据`effectTag`分别处理，其中`effectTag`包括(`Placement` | `Update` | `Deletion` | `Hydrating`)

#### Placement effect

当`Fiber节点`含有`Placement effectTag`，意味着该`Fiber节点`对应的`DOM节点`需要插入到页面中。

调用的方法为`commitPlacement`。

该方法所做的工作分为三步：

1. 获取父级`DOM节点`。其中`finishedWork`为传入的`Fiber节点`。

```js
const parentFiber = getHostParentFiber(finishedWork);
// 父级DOM节点
const parentStateNode = parentFiber.stateNode;
```

1. 获取`Fiber节点`的`DOM`兄弟节点

```js
const before = getHostSibling(finishedWork);
```

1. 根据`DOM`兄弟节点是否存在决定调用`parentNode.insertBefore`或`parentNode.appendChild`执行`DOM`插入操作。

```js
// parentStateNode是否是rootFiber
if (isContainer) {
  insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
} else {
  insertOrAppendPlacementNode(finishedWork, before, parent);
}
```

值得注意的是，`getHostSibling`（获取兄弟`DOM节点`）的执行很耗时，当在同一个父`Fiber节点`下依次执行多个插入操作，`getHostSibling`算法的复杂度为指数级。

这是由于`Fiber节点`不只包括`HostComponent`，所以`Fiber树`和渲染的`DOM树`节点并不是一一对应的。要从`Fiber节点`找到`DOM节点`很可能跨层级遍历。

#### Update effect

当`Fiber节点`含有`Update effectTag`，意味着该`Fiber节点`需要更新。调用的方法为`commitWork`，他会根据`Fiber.tag`分别处理。

**FunctionComponent mutation**

当`fiber.tag`为`FunctionComponent`，会调用`commitHookEffectListUnmount`。该方法会遍历`effectList`，执行所有`useLayoutEffect hook`的销毁函数。

```js
var firstEffect = lastEffect.next;
var effect = firstEffect;

do {
  if ((effect.tag & tag) === tag) {
    // Unmount
    var destroy = effect.destroy;
    effect.destroy = undefined;

    if (destroy !== undefined) {
      destroy();
    }
  }

  effect = effect.next;
} while (effect !== firstEffect);
```



**HostComponent mutation**

当`fiber.tag`为`HostComponent`，会调用`commitUpdate`。

最终会在`updateDOMProperties`中将`render阶段 completeWork`中为`Fiber节点`赋值的`updateQueue`对应的内容渲染在页面上。

#### Deletion effect

当`Fiber节点`含有`Deletion effectTag`，意味着该`Fiber节点`对应的`DOM节点`需要从页面中删除。调用的方法为`commitDeletion`。

该方法会执行如下操作：

1. 递归调用`Fiber节点`及其子孙`Fiber节点`中`fiber.tag`为`ClassComponent`的`componentWillUnmount`生命周期钩子，从页面移除`Fiber节点`对应`DOM节点`
2. 解绑`ref`
3. 调度`useEffect`的销毁函数



### layout阶段

```js
root.current = finishedWork;
```

`workInProgress Fiber树`在`commit阶段`完成渲染后会变为`current Fiber树`。这行代码的作用就是切换`fiberRootNode`指向的`current Fiber树`。

那么这行代码为什么在这里呢？（在`mutation阶段`结束后，`layout阶段`开始前。）

> 因为`componentWillUnmount`发生在`commitMutationEffects`函数中，这时还可以获取之前的Update，而`componentDidMount`和`componentDidUpdate`会在`commitLayoutEffects`中执行，这时已经可以获取更新后的真实dom了

我们知道`componentWillUnmount`会在`mutation阶段`执行。此时`current Fiber树`还指向前一次更新的`Fiber树`，在生命周期钩子内获取的`DOM`还是更新前的。

`componentDidMount`和`componentDidUpdate`会在`layout阶段`执行。此时`current Fiber树`已经指向更新后的`Fiber树`，在生命周期钩子内获取的`DOM`就是更新后的。

----

> 该阶段之所以称为`layout`，因为该阶段的代码都是在`DOM`渲染完成（`mutation阶段`完成）后执行的。
>
> 该阶段触发的生命周期钩子和`hook`可以直接访问到已经改变后的`DOM`，即该阶段是可以参与`DOM layout`的阶段。

#### 执行commitLayoutEffects函数

```js
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;

    // 调用生命周期钩子和hook
    if (effectTag & (Update | Callback)) {
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }

    // 赋值ref
    if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

`commitLayoutEffects`一共做了两件事：

1. commitLayoutEffectOnFiber（**commitLifeCycles**）（调用`生命周期钩子`和`hook`相关操作）
2. commitAttachRef（赋值 ref）

#### commitLifeCycles

`commitLifeCycles`方法会根据`fiber.tag`对不同类型的节点分别处理。

- 对于`ClassComponent`，他会通过`current === null?`区分是`mount`还是`update`，调用`componentDidMount`或`componentDidUpdate`。触发`状态更新`的`this.setState`如果赋值了第二个参数`回调函数`，也会在此时调用。
- 对于`FunctionComponent`及相关类型，他会调用`useLayoutEffect hook`的`回调函数`，调度`useEffect`的`销毁`与`回调`函数

```js
switch (finishedWork.tag) {
    // 以下都是FunctionComponent及相关类型
  case FunctionComponent:
  case ForwardRef:
  case SimpleMemoComponent:
  case Block: {
    // 执行useLayoutEffect的回调函数
    commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
    // 调度useEffect的销毁函数与回调函数
    schedulePassiveEffects(finishedWork);
    return;
 }
```

`useLayoutEffect hook`从上一次更新的`销毁函数`调用到本次更新的`回调函数`调用是同步执行的。而`useEffect`则需要先调度，在`Layout阶段`完成后再异步执行。这就是`useLayoutEffect`与`useEffect`的区别。

- 对于`HostRoot`，即`rootFiber`，如果赋值了第三个参数`回调函数`，也会在此时调用。

```js
ReactDOM.render(<App />, document.querySelector("#root"), function() {
  console.log("i am mount~");
});
```

#### commitAttachRef

```js
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;

    // 获取DOM实例
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }

    if (typeof ref === "function") {
      // 如果ref是函数形式，调用回调函数
      ref(instanceToUse);
    } else {
      // 如果ref是ref实例形式，赋值ref.current
      ref.current = instanceToUse;
    }
  }
}
```



| useEffect                                   | useLayoutEffect |
| ------------------------------------------- | --------------- |
| beforeMutation阶段：调度flushPassiveEffects | 无 |
| mutation阶段：无 | 执行destroy |
| layout阶段：注册destroy | 执行create |
| commit阶段完成后：执行flushPassiveEffects，内部执行注册的回调 | 无 |





## diff算法

### 单节点diff

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632358405887-61698573-2c2c-4471-9372-e85d87b95cfa.png)

- key和type相同表示可以复用节点
- key不同直接标记删除节点，然后新建节点

- key相同type不同，标记删除该节点和兄弟节点，然后新创建节点



```js
function reconcileSingleElement(
  returnFiber: Fiber, // diff节点的父节点
  currentFirstChild: Fiber | null, // current fiber
  element: ReactElement // 最新的jsx对象
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  
  // 首先判断是否存在对应DOM节点
  while (child !== null) {
    // 上一次更新存在DOM节点，接下来判断是否可复用

    // 首先比较key是否相同
    if (child.key === key) {

      // key相同，接下来比较type是否相同

      switch (child.tag) {
        // ...省略case
        
        default: {
          if (child.elementType === element.type) {
            var _existing3 = useFiber(child, element.props);
            // ...
            // type相同则表示可以复用
            // 返回复用的fiber
            return _existing3;
          }
          
          // type不同则跳出switch
          break;
        }
      }
      // 代码执行到这里代表：key相同但是type不同
      // 将该fiber及其兄弟fiber标记为删除
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // key不同，将该fiber标记为删除
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  // 创建新Fiber，并返回 ...省略
}
```

### 多节点diff

`多Diff`的整体逻辑会经历两轮遍历：

第一轮遍历：处理`更新`的节点。

第二轮遍历：处理剩下的不属于`更新`的节点。

#### 第一轮遍历

第一轮遍历步骤如下：

1. `let i = 0`，遍历`newChildren`，将`newChildren[i]`与`oldFiber`比较，判断`DOM节点`是否可复用。
2. 如果可复用，`i++`，继续比较`newChildren[i]`与`oldFiber.sibling`，可以复用则继续遍历。
3. 如果不可复用，分两种情况：
   1. `key`不同导致不可复用，立即跳出整个遍历，**第一轮遍历结束。**
   2.   `key`相同`type`不同导致不可复用，会将`oldFiber`标记为`DELETION`，并继续遍历

4. 如果`newChildren`遍历完（即`i === newChildren.length - 1`）或者`oldFiber`遍历完（即`oldFiber.sibling === null`），跳出遍历，**第一轮遍历结束。**

当遍历结束后，会有两种结果：

**步骤3跳出的遍历**

此时`newChildren`没有遍历完，`oldFiber`也没有遍历完。

```jsx
// 之前
<li key="0">0</li>
<li key="1">1</li>
<li key="2">2</li>
            
// 之后
<li key="0">0</li>
<li key="2">1</li>
<li key="1">2</li>
```

第一个节点可复用，遍历到`key === 2`的节点发现`key`改变，不可复用，跳出遍历，等待第二轮遍历处理。

此时`oldFiber`剩下`key === 1`、`key === 2`未遍历，`newChildren`剩下`key === 2`、`key === 1`未遍历。

**步骤4跳出的遍历**

可能`newChildren`遍历完，或`oldFiber`遍历完，或他们同时遍历完。

```jsx
// 之前
<li key="0" className="a">0</li>
<li key="1" className="b">1</li>
            
// 之后 情况1 —— newChildren与oldFiber都遍历完
<li key="0" className="aa">0</li>
<li key="1" className="bb">1</li>
            
// 之后 情况2 —— newChildren没遍历完，oldFiber遍历完
// newChildren剩下 key==="2" 未遍历
<li key="0" className="aa">0</li>
<li key="1" className="bb">1</li>
<li key="2" className="cc">2</li>
            
// 之后 情况3 —— newChildren遍历完，oldFiber没遍历完
// oldFiber剩下 key==="1" 未遍历
<li key="0" className="aa">0</li>
```



#### 第二轮遍历

对于第一轮遍历的结果，我们分别讨论：

`newChildren`与`oldFiber`同时遍历完

那就是最理想的情况：只需在第一轮遍历进行组件更新。此时`Diff`结束。

---

`newChildren`没遍历完，`oldFiber`遍历完

已有的`DOM节点`都复用了，这时还有新加入的节点，意味着本次更新有新节点插入，我们只需要遍历剩下的`newChildren`为生成的`workInProgress fiber`依次标记`Placement`。

---

`newChildren`遍历完，`oldFiber`没遍历完

意味着本次更新比之前的节点数量少，有节点被删除了。所以需要遍历剩下的`oldFiber`，依次标记`Deletion`。

---

`newChildren`与`oldFiber`都没遍历完

这意味着有节点在这次更新中改变了位置。

由于有节点改变了位置，所以不能再用位置索引`i`对比前后的节点，我们需要使用`key`将同一个节点在两次更新中对应。

为了快速的找到`key`对应的`oldFiber`，将所有还未处理的`oldFiber`存入以`key`为key，`oldFiber`为value的`Map`中。

```javascript
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
```

接下来遍历剩余的`newChildren`，通过`newChildren[i].key`就能在`existingChildren`中找到`key`相同的`oldFiber`。

**标记节点是否移动**

既然我们的目标是寻找移动的节点，那么我们需要明确：节点是否移动是以什么为参照物？

参照物是：最后一个可复用的节点在`oldFiber`中的位置索引（用变量`lastPlacedIndex`表示）。

由于本次更新中节点是按`newChildren`的顺序排列。在遍历`newChildren`过程中，每个`遍历到的可复用节点`一定是当前遍历到的`所有可复用节点`中**最靠右的那个**，即一定在`lastPlacedIndex`对应的`可复用的节点`在本次更新中位置的后面。

那么只需要比较`遍历到的可复用节点`在上次更新时是否也在`lastPlacedIndex`对应的`oldFiber`后面，就能知道两次更新中这两个节点的相对位置改变没有。

用变量`oldIndex`表示`遍历到的可复用节点`在`oldFiber`中的位置索引。如果`oldIndex < lastPlacedIndex`，代表本次更新该节点需要向右移动。

`lastPlacedIndex`初始为`0`，每遍历一个可复用的节点，如果`oldIndex >= lastPlacedIndex`，则`lastPlacedIndex = oldIndex`。



## 状态更新



```
触发状态更新（根据场景调用不同方法）

    |
    |
    v

创建Update对象（`updateContainer`）

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```



### 创建Update对象

每次`状态更新`都会创建一个保存**更新状态相关内容**的对象，我们叫他`Update`。在`render阶段`的`beginWork`中会根据`Update`计算新的`state`。

在`React`中，有如下方法可以触发状态更新：

- ReactDOM.render —— HostRoot
- this.setState —— ClassComponent
- this.forceUpdate —— ClassComponent
- useState —— FunctionComponent
- useReducer —— FunctionComponent

一共三种组件（`HostRoot` | `ClassComponent` | `FunctionComponent`）可以触发更新。

由于不同类型组件工作方式不同，所以存在两种不同结构的`Update`，其中`ClassComponent`与`HostRoot`共用一套`Update`结构，`FunctionComponent`单独使用一种`Update`结构。

虽然他们的结构不同，但是他们工作机制与工作流程大体相同。

#### Update结构

```js
function createUpdate(eventTime, lane) {
  var update = {
    eventTime: eventTime,
    lane: lane,
    tag: UpdateState,
    payload: null,
    callback: null,
    next: null
  };
  return update;
}
```

- eventTime：任务时间，通过`performance.now()`获取的毫秒数。
- lane：优先级相关字段。当前还不需要掌握他，只需要知道不同`Update`优先级可能是不同的。

> 可以将`lane`类比`心智模型`中`需求的紧急程度`。

- suspenseConfig：`Suspense`相关，暂不关注。
- tag：更新的类型，包括`UpdateState` | `ReplaceState` | `ForceUpdate` | `CaptureUpdate`。
- payload：更新挂载的数据，不同类型组件挂载的数据不同。对于`ClassComponent`，`payload`为`this.setState`的第一个传参。对于`HostRoot`，`payload`为`ReactDOM.render`的第一个传参。
- callback：更新的回调函数。
- next：与其他`Update`连接形成链表。



#### Update与Fiber的联系

`Update`存在一个连接其他`Update`形成链表的字段`next`

类似`Fiber节点`组成`Fiber树`，`Fiber节点`上的多个`Update`会组成链表并被包含在`fiber.updateQueue`中。

`Fiber节点`最多同时存在两个`updateQueue`：

- `current fiber`保存的`updateQueue`即`current updateQueue`
- `workInProgress fiber`保存的`updateQueue`即`workInProgress updateQueue`

在`commit阶段`完成页面渲染后，`workInProgress Fiber树`变为`current Fiber树`，`workInProgress Fiber树`内`Fiber节点`的`updateQueue`就变成`current updateQueue`。

#### updateQueue

`ClassComponent`与`HostRoot`使用的`UpdateQueue`结构

```js
function initializeUpdateQueue(fiber) {
  var queue = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null
    },
    effects: null
  };
  fiber.updateQueue = queue;
}
```

- baseState：本次更新前该`Fiber节点`的`state`，`Update`基于该`state`计算更新后的`state`。

- `firstBaseUpdate`与`lastBaseUpdate`：本次更新前该`Fiber节点`已保存的`Update`。以链表形式存在，链表头为`firstBaseUpdate`，链表尾为`lastBaseUpdate`。之所以在更新产生前该`Fiber节点`内就存在`Update`，是由于某些`Update`优先级较低所以在上次`render阶段`由`Update`计算`state`时被跳过。

- `shared.pending`：触发更新时，产生的`Update`会保存在`shared.pending`中形成单向环状链表。当由`Update`计算`state`时这个环会被剪开并连接在`lastBaseUpdate`后面。

- effects：数组。保存`update.callback !== null`的`Update`。



### 从fiber到root

调用`markUpdateLaneFromFiberToRoot`方法。

该方法做的工作可以概括为：从`触发状态更新的fiber`一直向上遍历到`rootFiber`，并返回`rootFiber`。

### 调度更新

调用的方法是`ensureRootIsScheduled` 通知`Scheduler`根据**更新**的优先级，决定以**同步**还是**异步**的方式调度本次更新。

```js
if (newCallbackPriority === SyncLanePriority) {
  // 任务已经过期，需要同步执行render阶段
  newCallbackNode = scheduleSyncCallback(
    performSyncWorkOnRoot.bind(null, root)
  );
} else {
  // 根据任务优先级异步执行render阶段
  var schedulerPriorityLevel = lanePriorityToSchedulerPriority(
    newCallbackPriority
  );
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root)
  );
}
```





### 深入理解优先级

```js
const NoPriority = 0; // 初始化时的无优先级
const ImmediatePriority = 1; // 立刻执行的优先级 同步优先级
const UserBlockingPriority = 2; // 用户触发更新时的优先级
const NormalPriority = 3; // 一般优先级 最常见
const LowPriority = 4; // 低优先级
const IdlePriority = 5; // 空闲的优先级
```

`React`根据`人机交互研究的结果`中用户对`交互`的预期顺序为`交互`产生的`状态更新`赋予不同优先级。

- 生命周期方法：同步执行。
- 受控的用户输入：比如输入框内输入文字，同步执行。
- 交互事件：比如动画，高优先级执行。
- 其他：比如数据请求，低优先级执行。

每当需要调度任务时，`React`会调用`Scheduler`提供的方法`runWithPriority`。

该方法接收一个`优先级`常量与一个`回调函数`作为参数。`回调函数`会以`优先级`高低为顺序排列在一个`定时器`中并在合适的时间触发。

```js
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}
```

对于更新来讲，传递的`回调函数`一般为`render阶段的入口函数`。

```js
function initializeUpdateQueue(fiber) {
  var queue = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null
    },
    effects: null
  };
  fiber.updateQueue = queue;
}
```

高优先级的任务出现时会立即替换掉地低优先级的任务并执行高优先级的任务

在`processUpdateQueue`方法中，`shared.pending`环状链表会被剪开并拼接在`baseUpdate`后面。



**如何保证状态正确？**

- `render阶段`可能被中断。如何保证`updateQueue`中保存的`Update`不丢失？

  - 在`render阶段`，`shared.pending`的环被剪开并连接在`updateQueue.lastBaseUpdate`后面。

    实际上`shared.pending`会被同时连接在`workInProgress updateQueue.lastBaseUpdate`与`current updateQueue.lastBaseUpdate`后面。

    当`render阶段`被中断后重新开始时，会基于`current updateQueue`克隆出`workInProgress updateQueue`。由于`current updateQueue.lastBaseUpdate`已经保存了上一次的`Update`，所以不会丢失。

    当`commit阶段`完成渲染，由于`workInProgress updateQueue.lastBaseUpdate`中保存了上一次的`Update`，所以 `workInProgress Fiber树`变成`current Fiber树`后也不会造成`Update`丢失。

- 有时候当前`状态`需要依赖前一个`状态`。如何在支持跳过`低优先级状态`的同时保证**状态依赖的连续性**？

  - 当某个`Update`由于优先级低而被跳过时，保存在`baseUpdate`中的不仅是该`Update`，还包括链表中该`Update`之后的所有`Update`。
  - 当有`Update`被跳过时，`下次更新的baseState !== 上次更新的memoizedState`



### 同步更新的React

将`更新机制`类比`代码版本控制`。

在没有代码版本控制前，我们在代码中逐步叠加功能。一切看起来井然有序，直到我们遇到了一个紧急线上bug（红色节点）。

![img](https://react.iamkasong.com/img/git1.png)

为了修复这个bug，我们需要首先将之前的代码提交。

在React中，所有通过ReactDOM.render创建的应用都是通过类似的方式更新状态。

即没有优先级概念，高优更新（红色节点）需要排在其他更新后面执行。

### 并发更新的React

当有了代码版本控制，有紧急线上bug需要修复时，我们暂存当前分支的修改，在master分支修复bug并紧急上线。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1634655634031-27e0b49f-dd77-4140-9b8e-2a45a429f168.png)

bug修复上线后通过git rebase命令和开发分支连接上。开发分支基于修复bug的版本继续开发。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1634655634049-720a76db-18be-4358-ab9a-1e00ddd01022.png)

在React中，通过ReactDOM.createBlockingRoot和ReactDOM.createRoot创建的应用会采用并发的方式更新状态。

高优更新（红色节点）中断正在进行中的低优更新（蓝色节点），先完成render - commit流程。

待高优更新完成后，低优更新基于高优更新的结果重新更新。



### ReactDOM.render

```js
function render(element, container, callback) {
  ...
  return legacyRenderSubtreeIntoContainer(null, element, container, false, callback);
}
function legacyRenderSubtreeIntoContainer(parentComponent, children, container, forceHydrate, callback) {
		...
  	root = container._reactRootContainer = legacyCreateRootFromDOMContainer(container, forceHydrate);
    fiberRoot = root._internalRoot;
    ...
    unbatchedUpdates(function () {
      updateContainer(children, fiberRoot, parentComponent, callback); // 创建update对象
    });
}
function legacyCreateRootFromDOMContainer(container, forceHydrate) {
		while (rootSibling = container.lastChild) {
        container.removeChild(rootSibling);
    }
    return createLegacyRoot(container, shouldHydrate ? {
    hydrate: true
  	} : undefined);
}
// createLegacyRoot -> ReactDOMBlockingRoot -> createRootImpl -> createContainer -> createFiberRoot
  
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  // 创建fiberRootNode
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  
  // 创建rootFiber
  const uninitializedFiber = createHostRootFiber(tag);

  // 连接rootFiber与fiberRootNode
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  // 初始化updateQueue
  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

`legacyCreateRootFromDOMContainer`方法内部会调用`createFiberRoot`方法完成`fiberRootNode`和`rootFiber`的创建以及关联。并初始化`updateQueue`。



#### ReactDOM.createRoot

```js
function createRoot(container, options) {
  ...
  return new ReactDOMRoot(container, options);
}
function ReactDOMRoot(container, options) {
  this._internalRoot = createRootImpl(container, ConcurrentRoot, options);
}
```



#### 创建update

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  // ...省略与逻辑不相关代码

  // 创建update
  const update = createUpdate(eventTime, lane, suspenseConfig);
  
  // update.payload为需要挂载在根节点的组件
  update.payload = {element};

  // callback为ReactDOM.render的第三个参数 —— 回调函数
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }

  // 将生成的update加入updateQueue
  enqueueUpdate(current, update);
  // 调度更新
  scheduleUpdateOnFiber(current, lane, eventTime);

}
```



### 整体流程

```
创建fiberRootNode、rootFiber、updateQueue（`legacyCreateRootFromDOMContainer`）

    |
    |
    v

创建Update对象（`updateContainer`）

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```



当前`React`共有三种模式：

- `legacy`，这是当前`React`使用的方式。当前没有计划删除本模式，但是这个模式可能不支持一些新功能。
- `blocking`，开启部分`concurrent`模式特性的中间模式。目前正在实验中。作为迁移到`concurrent`模式的第一个步骤。
- `concurrent`，面向未来的开发模式。我们之前讲的`任务中断/任务优先级`都是针对`concurrent`模式。

调用方式:

- `legacy` -- `ReactDOM.render(<App />, rootNode)`
- `blocking` -- `ReactDOM.createBlockingRoot(rootNode).render(<App />)`
- `concurrent` -- `ReactDOM.createRoot(rootNode).render(<App />)`

**三种调用方式只是针对createRootImpl函数传参的不同**

```js
var LegacyRoot = 0;
var BlockingRoot = 1;
var ConcurrentRoot = 2;

function ReactDOMRoot(container, options) {
  this._internalRoot = createRootImpl(container, ConcurrentRoot, options);
}

function ReactDOMBlockingRoot(container, tag, options) {
  this._internalRoot = createRootImpl(container, tag, options);
```



### tihs.setState流程

```js
Component.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```



```js
enqueueSetState(inst, payload, callback) {
  // 通过组件实例获取对应fiber
  const fiber = getInstance(inst);

  const eventTime = requestEventTime();
  const suspenseConfig = requestCurrentSuspenseConfig();

  // 获取优先级
  const lane = requestUpdateLane(fiber, suspenseConfig);

  // 创建update
  const update = createUpdate(eventTime, lane, suspenseConfig);

  update.payload = payload;

  // 赋值回调函数
  if (callback !== undefined && callback !== null) {
    update.callback = callback;
  }

  // 将update插入updateQueue
  enqueueUpdate(fiber, update);
  // 调度update
  scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```

对于`ClassComponent`，`update.payload`为`this.setState`的第一个传参（即要改变的`state`）



#### this.forceUpdate

````js
enqueueForceUpdate(inst, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const lane = requestUpdateLane(fiber, suspenseConfig);

    const update = createUpdate(eventTime, lane, suspenseConfig);

    // 赋值tag为ForceUpdate
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
};
````

**`update.tag = ForceUpdate` 的作用？**

在判断`ClassComponent`是否需要更新时有两个条件需要满足：

```js
const shouldUpdate =
  checkHasForceUpdateAfterProcessing() ||
  checkShouldComponentUpdate(
    workInProgress,
    ctor,
    oldProps,
    newProps,
    oldState,
    newState,
    nextContext,
  );
```

- checkHasForceUpdateAfterProcessing：内部会判断本次更新的`Update`是否为`ForceUpdate`。即如果本次更新的`Update`中存在`tag`为`ForceUpdate`，则返回`true`。
- checkShouldComponentUpdate：内部会调用`shouldComponentUpdate`方法。以及当该`ClassComponent`为`PureComponent`时会浅比较`state`与`props`。

所以，当某次更新含有`tag`为`ForceUpdate`的`Update`，那么当前`ClassComponent`不会受其他`性能优化手段`（`shouldComponentUpdate`|`PureComponent`）影响，一定会更新。



## Hook

### 简单实现useState

```js
let isMount = true;
let workInProcessHook = null;

const filber = {
  memoizedState: null, // hook
  stateNode: App, // 节点实例
}

function run() {
  workInProcessHook = filber.memoizedState;
  const app = filber.stateNode;
  isMount = false;
  return app();
}

function dispatchAction(queue, action) {
  const update = {
    action,
    next: null,
  };
  if (queue.pending === null) {
    update.next = update;
  } else {
    update.next = queue.pending.next;
    queue.pending.next = update;
  }
  queue.pending = update;
  run();
}

function useState(initialState) {
  let hook;
  if (isMount) {
    hook = {
      queue: {
        pending: null
      },
      memoizedState: initialState, // hook state
      next: null,
    }
    if (!filber.memoizedState) {
      filber.memoizedState = hook;
    } else {
      workInProcessHook.next = hook;
    }
    workInProcessHook = hook;
  } else {
    hook = workInProcessHook;
    workInProcessHook = workInProcessHook.next;
  }

  let baseState = hook.memoizedState;
  if (hook.queue.pending) {
    let firstUpdate = hook.queue.pending.next;
    do {
      const action = firstUpdate.action;
      baseState = action(baseState);
      firstUpdate = firstUpdate.next;
    } while (firstUpdate !== hook.queue.pending.next);

    hook.queue.pending = null;
  }
  hook.memoizedState = baseState;
  return [baseState, dispatchAction.bind(null, hook.queue)];
}

function App() {
  const [num, updateNum] = useState(0);

  console.log('isMount=', isMount);
  console.log('num=', num);

  return {
    onClick() {
      updateNum(num => num + 1);
      updateNum(num => num + 1);
      updateNum(num => num + 1);
    }
  }
}
window.app = App();
```



### useState与useReduer

```js
// mount时的Dispatcher
const HooksDispatcherOnMount: Dispatcher = {
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  // ...
};

// update时的Dispatcher
const HooksDispatcherOnUpdate: Dispatcher = {
  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  // ...
};
```

`mount`时调用的`hook`和`update`时调用的`hook`其实是两个不同的函数。

在`FunctionComponent` `render`前，会根据`FunctionComponent`对应`fiber`的以下条件区分`mount`与`update`。

```js
current === null || current.memoizedState === null
```

并将不同情况对应的`dispatcher`赋值给全局变量`ReactCurrentDispatcher`的`current`属性。

```js
ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
```

在`FunctionComponent` `render`时，会从`ReactCurrentDispatcher.current`（即当前`dispatcher`）中寻找需要的`hook`。

不同的调用栈上下文为`ReactCurrentDispatcher.current`赋值不同的`dispatcher`，则`FunctionComponent` `render`时调用的`hook`也是不同的函数。

#### memoizedState

>`hook`与`FunctionComponent fiber`都存在`memoizedState`属性，不要混淆他们的概念。
>
>+ `fiber.memoizedState`：`FunctionComponent`对应`fiber`保存的`Hooks`链表。
>+  `hook.memoizedState`：`Hooks`链表中保存的单一`hook`对应的数据。

不同类型`hook`的`memoizedState`保存不同类型数据，具体如下：

- useState：对于`const [state, updateState] = useState(initialState)`，`memoizedState`保存`state`的值
- useReducer：对于`const [state, dispatch] = useReducer(reducer, {});`，`memoizedState`保存`state`的值
- useEffect：`memoizedState`保存包含`useEffect回调函数`、`依赖项`等的链表数据结构`effect`，你可以在[这里 (opens new window)](https://github.com/acdlite/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberHooks.new.js#L1181)看到`effect`的创建过程。`effect`链表同时会保存在`fiber.updateQueue`中
- useRef：对于`useRef(1)`，`memoizedState`保存`{current: 1}`
- useMemo：对于`useMemo(callback, [depA])`，`memoizedState`保存`[callback(), depA]`
- useCallback：对于`useCallback(callback, [depA])`，`memoizedState`保存`[callback, depA]`。与`useMemo`的区别是，`useCallback`保存的是`callback`函数本身，而`useMemo`保存的是`callback`函数的执行结果

有些`hook`是没有`memoizedState`的，比如：

- useContext



#### 声明阶段

当`FunctionComponent`进入`render阶段`的`beginWork`时，会调用`renderWithHooks`方法。

该方法内部会执行`FunctionComponent`对应函数（即`fiber.type`）

```js
function useState(initialState) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
function useReducer(reducer, initialArg, init) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useReducer(reducer, initialArg, init);
}
```

**mount时**

`mount`时，`useReducer`会调用`mountReducer`，`useState`会调用`mountState`

```tsx
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 创建并返回当前的hook
  const hook = mountWorkInProgressHook();

  // ...赋值初始state

  // 创建queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });

  // ...创建dispatch
  return [hook.memoizedState, dispatch];
}

function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 创建并返回当前的hook
  const hook = mountWorkInProgressHook();

  // ...赋值初始state

  // 创建queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });

  // ...创建dispatch
  return [hook.memoizedState, dispatch];
}
```

`mount`时这两个`Hook`的唯一区别为`queue`参数的`lastRenderedReducer`字段。

```js
const queue = (hook.queue = {
  // 保存update对象
  pending: null,
  // 保存dispatchAction.bind()的值
  dispatch: null,
  // 上一次render时使用的reducer
  lastRenderedReducer: reducer,
  // 上一次render时的state
  lastRenderedState: (initialState: any),
});
```

其中，`useReducer`的`lastRenderedReducer`为传入的`reducer`参数。`useState`的`lastRenderedReducer`为`basicStateReducer`。

`basicStateReducer`方法如下：

```js
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

可见，`useState`即`reducer`参数为`basicStateReducer`的`useReducer`。

**update时**

`update`时，`useReducer`与`useState`调用的则是同一个函数`updateReducer `。

```js
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 获取当前hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  queue.lastRenderedReducer = reducer;

  // ...同update与updateQueue类似的更新逻辑

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

整个流程可以概括为一句话：

> 找到对应的`hook`，根据`update`计算该`hook`的新`state`并返回。

`mount`时获取当前`hook`使用的是`mountWorkInProgressHook`，而`update`时使用的是`updateWorkInProgressHook`，这里的原因是：

- `mount`时可以确定是调用`ReactDOM.render`或相关初始化`API`产生的`更新`，只会执行一次。
- `update`可能是在事件回调或副作用中触发的`更新`或者是`render阶段`触发的`更新`，为了避免组件无限循环`更新`，后者需要区别对待。



#### 调用阶段

调用阶段会执行`dispatchAction`，此时该`FunctionComponent`对应的`fiber`以及`hook.queue`已经通过调用`bind`方法预先作为参数传入。

```js
function dispatchAction(fiber, queue, action) {

  // ...创建update
  var update = {
    eventTime: eventTime,
    lane: lane,
    suspenseConfig: suspenseConfig,
    action: action,
    eagerReducer: null,
    eagerState: null,
    next: null
  }; 

  // ...将update加入queue.pending
  
  var alternate = fiber.alternate;

  if (fiber === currentlyRenderingFiber$1 || alternate !== null && alternate === currentlyRenderingFiber$1) {
    // render阶段触发的更新
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
      // ...fiber的updateQueue为空，优化路径
    }

    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}
```

整个过程可以概括为：

> 创建`update`，将`update`加入`queue.pending`中，并开启调度。

这里值得注意的是`if...else...`逻辑，其中：

```js
if (fiber === currentlyRenderingFiber$1 || alternate !== null && alternate === currentlyRenderingFiber$1)
```

`currentlyRenderingFiber`即`workInProgress`，`workInProgress`存在代表当前处于`render阶段`。

触发`更新`时通过`bind`预先保存的`fiber`与`workInProgress`全等，代表本次`更新`发生于`FunctionComponent`对应`fiber`的`render阶段`。

所以这是一个`render阶段`触发的`更新`，需要标记变量`didScheduleRenderPhaseUpdate`，后续单独处理。

再来关注：

```js
if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes))
```

`fiber.lanes`保存`fiber`上存在的`update`的`优先级`。

`fiber.lanes === NoLanes`意味着`fiber`上不存在`update`。

通过`update`计算`state`发生在`声明阶段`，这是因为该`hook`上可能存在多个不同`优先级`的`update`，最终`state`的值由多个`update`共同决定。

但是当`fiber`上不存在`update`，则`调用阶段`创建的`update`为该`hook`上第一个`update`，在`声明阶段`计算`state`时也只依赖于该`update`，完全不需要进入`声明阶段`再计算`state`。

这样做的好处是：如果计算出的`state`与该`hook`之前保存的`state`一致，那么完全不需要开启一次调度。即使计算出的`state`与该`hook`之前保存的`state`不一致，在`声明阶段`也可以直接使用`调用阶段`已经计算出的`state`。



### useEffect和useLayoutEffect

```js
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
    return mountEffectImpl(
      PassiveEffect | PassiveStaticEffect,
      HookPassive,
      create,
      deps,
    );
}

function updateEffect(create, deps) {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(UpdateEffect, HookLayout, create, deps);
}

function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}


function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  var hook = mountWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber$1.flags |= fiberFlags;
  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, undefined, nextDeps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var destroy = undefined;

  if (currentHook !== null) {
    var prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy; // 前一个useEffect已经执行完成，可能存在销毁函数，需要在此次更新前执行上一次的销毁函数

    if (nextDeps !== null) {
      var prevDeps = prevEffect.deps;

       // 确认上次和本次的依赖项是否不同  不过不管是否不同都会追加链表，这是为了保证链表的顺序性
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber$1.flags |= fiberFlags;
  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, destroy, nextDeps);
}

function pushEffect(tag, create, destroy, deps) {
  var effect = {
    tag: tag,
    create: create, // useEffect 传入的回调函数
    destroy: destroy, // useEffect 回调函数的返回值-当前更新的销毁函数
    deps: deps, // useEffect 得第二个参数  回调函数的依赖项
    // Circular
    next: null
  };
  var componentUpdateQueue = currentlyRenderingFiber$1.updateQueue;

  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber$1.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    var lastEffect = componentUpdateQueue.lastEffect;

    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      var firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }

  return effect;
}
```

#### flushPassiveEffects

`flushPassiveEffects`内部会设置`优先级`，并执行`flushPassiveEffectsImpl`。

`flushPassiveEffectsImpl`主要做三件事：

- 调用该`useEffect`在上一次`render`时的销毁函数
- 调用该`useEffect`在本次`render`时的回调函数
- 如果存在同步任务，不需要等待下次`事件循环`的`宏任务`，提前执行他



**阶段一：销毁函数的执行**

`useEffect`的执行需要保证所有组件`useEffect`的`销毁函数`必须都执行完后才能执行任意一个组件的`useEffect`的`回调函数`。

这是因为多个`组件`间可能共用同一个`ref`。

如果不是按照“全部销毁”再“全部执行”的顺序，那么在某个组件`useEffect`的`销毁函数`中修改的`ref.current`可能影响另一个组件`useEffect`的`回调函数`中的同一个`ref`的`current`属性。

在`useLayoutEffect`中也有同样的问题，所以他们都遵循“全部销毁”再“全部执行”的顺序。

在阶段一，会遍历并执行所有`useEffect`的`销毁函数`。

```js
// pendingPassiveHookEffectsUnmount中保存了所有需要执行销毁的useEffect
const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = [];
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    const fiber = ((unmountEffects[i + 1]: any): Fiber);
    const destroy = effect.destroy;
    effect.destroy = undefined;

    if (typeof destroy === 'function') {
      // 销毁函数存在则执行
      try {
        destroy();
      } catch (error) {
        captureCommitPhaseError(fiber, error);
      }
    }
  }
```

其中`pendingPassiveHookEffectsUnmount`数组的索引`i`保存需要销毁的`effect`，`i+1`保存该`effect`对应的`fiber`。

向`pendingPassiveHookEffectsUnmount`数组内`push`数据的操作发生在`layout`阶段`commitLayoutEffectOnFiber`方法内部的`schedulePassiveEffects`方法中。

```js
function schedulePassiveEffects(finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      const {next, tag} = effect;
      if (
        (tag & HookPassive) !== NoHookEffect &&
        (tag & HookHasEffect) !== NoHookEffect
      ) {
        // 向`pendingPassiveHookEffectsUnmount`数组内`push`要销毁的effect
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        // 向`pendingPassiveHookEffectsMount`数组内`push`要执行回调的effect
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
      }
      effect = next;
    } while (effect !== firstEffect);
  }
}
```

**阶段二：回调函数的执行**

与阶段一类似，同样遍历数组，执行对应`effect`的`回调函数`。

其中向`pendingPassiveHookEffectsMount`中`push`数据的操作同样发生在`schedulePassiveEffects`中。

```js
// pendingPassiveHookEffectsMount中保存了所有需要执行回调的useEffect
const mountEffects = pendingPassiveHookEffectsMount;
pendingPassiveHookEffectsMount = [];
for (let i = 0; i < mountEffects.length; i += 2) {
  const effect = ((mountEffects[i]: any): HookEffect);
  const fiber = ((mountEffects[i + 1]: any): Fiber);
  
  try {
    const create = effect.create;
   effect.destroy = create();
  } catch (error) {
    captureCommitPhaseError(fiber, error);
  }
}
```



### useRef

```js
function mountRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = mountWorkInProgressHook();
  // 创建ref
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = updateWorkInProgressHook();
  // 返回保存的数据
  return hook.memoizedState;
}
                                         
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  return refObject;
}
```

`useRef`仅仅是返回一个包含`current`属性的对象。



`ref`的工作流程分为两部分：

- `render阶段`为含有`ref`属性的`fiber`添加`Ref effectTag`
- `commit阶段`为包含`Ref effectTag`的`fiber`执行对应操作

#### render阶段

```js
// beginWork的markRef
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  if (
    (current === null && ref !== null) ||
    (current !== null && current.ref !== ref)
  ) {
    // Schedule a Ref effect
    workInProgress.effectTag |= Ref;
  }
}
// completeWork的markRef
function markRef(workInProgress: Fiber) {
  workInProgress.effectTag |= Ref;
}
```

在`beginWork`中，如下两处调用了`markRef`：

- `updateClassComponent`内的`finishClassComponent` ，对应`ClassComponent`

- `updateHostComponent` ，对应`HostComponent`

在`completeWork`中，如下两处调用了`markRef`：

- `completeWork`中的HostComponent 类型
- `completeWork`中的ScopeComponent类型

总结下`组件`对应`fiber`被赋值`Ref effectTag`需要满足的条件：

- `fiber`类型为`HostComponent`、`ClassComponent`、`ScopeComponent`
- 对于`mount`，`workInProgress.ref !== null`，即存在`ref`属性
- 对于`update`，`current.ref !== workInProgress.ref`，即`ref`属性改变



#### commit阶段


在`commit阶段`的`mutation阶段`中，对于`ref`属性改变的情况，需要先移除之前的`ref`。

```js
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;
    // ...

    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        // 移除之前的ref
        commitDetachRef(current);
      }
    }
    // ...
  }
  // ...
  
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') {
      // function类型ref，调用他，传参为null
      currentRef(null);
    } else {
      // 对象类型ref，current赋值为null
      currentRef.current = null;
    }
  }
}
```



接下来，在`mutation阶段`，对于`Deletion effectTag`的`fiber`（对应需要删除的`DOM节点`），需要递归他的子树，对子孙`fiber`的`ref`执行类似`commitDetachRef`的操作。

在`commitDeletion`——`unmountHostComponents`——`commitUnmount`——`ClassComponent | HostComponent`类型`case`中调用的`safelyDetachRef`方法负责执行类似`commitDetachRef`的操作。

```js
function safelyDetachRef(current: Fiber) {
  const ref = current.ref;
  if (ref !== null) {
    if (typeof ref === 'function') {
      try {
        ref(null);
      } catch (refError) {
        captureCommitPhaseError(current, refError);
      }
    } else {
      ref.current = null;
    }
  }
}
```

接下来进入`ref`的赋值阶段。

> `commitLayoutEffect`会执行`commitAttachRef`（赋值`ref`）

```js
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    // 获取ref属性对应的Component实例
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }

    // 赋值ref
    if (typeof ref === 'function') {
      ref(instanceToUse);
    } else {
      ref.current = instanceToUse;
    }
  }
}
```

至此，`ref`的工作流程完毕。



### useMemo

把“创建”函数和依赖项数组作为参数传入 `useMemo`，它仅会在某个依赖项改变时才重新计算 memoized 值。这种优化有助于避免在每次渲染时都进行高开销的计算。

记住，传入 `useMemo` 的函数会在渲染期间执行。请不要在这个函数内部执行与渲染无关的操作，诸如副作用这类的操作属于 `useEffect` 的适用范畴，而不是 `useMemo`。

如果没有提供依赖项数组，`useMemo` 在每次渲染时都会计算新的值。

**你可以把 `useMemo` 作为性能优化的手段，但不要把它当成语义上的保证。**将来，React 可能会选择“遗忘”以前的一些 memoized 值，并在下次渲染时重新计算它们，比如为离屏组件释放内存。先编写在没有 `useMemo` 的情况下也可以执行的代码 —— 之后再在你的代码中添加 `useMemo`，以达到优化性能的目的。



- `mountMemo`会将`回调函数`(nextCreate)的执行结果作为`value`保存

```js
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 计算value
  const nextValue = nextCreate();
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }
  // 变化，重新计算value
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```



### useCallback

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

返回一个回调函数。

把内联回调函数及依赖项数组作为参数传入 `useCallback`，它将返回该回调函数的 memoized 版本，该回调函数仅在某个依赖项改变时才会更新。当你把回调函数传递给经过优化的并使用引用相等性去避免非必要渲染（例如 `shouldComponentUpdate`）的子组件时，它将非常有用。

```js
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }

  // 变化，将新的callback作为value
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```



## Concurrent Mode

> Concurrent 模式是一组 React 的新功能，可帮助应用保持响应，并根据用户的设备性能和网速进行适当的调整。



### Scheduler

#### 时间切片

`时间切片`的本质是模拟实现requestIdleCallback。

```js
一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback
```

`requestIdleCallback`是在“浏览器重排/重绘”后如果当前帧还有空余时间时被调用的。

浏览器并没有提供其他`API`能够在同样的时机（浏览器重排/重绘后）调用以模拟其实现。

唯一能精准控制调用时机的`API`是`requestAnimationFrame`，能让我们在“浏览器重排/重绘”之前执行`JS`。

这也是为什么通常用这个`API`实现`JS`动画 —— 这是浏览器渲染前的最后时机，所以动画能快速被渲染。

所以，退而求其次，`Scheduler`的`时间切片`功能是通过`task`（宏任务）实现的

最常见的`task`当属`setTimeout`了。但是有个`task`比`setTimeout`执行时机更靠前，那就是`MessageChannel`。

所以`Scheduler`将需要被执行的回调函数作为`MessageChannel`的回调执行。如果当前宿主环境不支持`MessageChannel`，则使用`setTimeout`。

在`React`的`render`阶段，开启`Concurrent Mode`时，每次遍历前，都会通过`Scheduler`提供的`shouldYield`方法判断是否需要中断遍历，使浏览器有时间渲染：

```js
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

是否中断的依据，最重要的一点便是每个任务的剩余时间是否用完。

在`Schdeduler`中，为任务分配的初始剩余时间为`5ms`。

随着应用运行，会通过`fps`动态调整分配给任务的可执行时间。

```js
deadline = currentTime - yieldInterval;

function forceFrameRate(fps) {
  if (fps < 0 || fps > 125) {
    // Using console['error'] to evade Babel and ESLint
    console['error'](
      'forceFrameRate takes a positive int between 0 and 125, ' +
        'forcing frame rates higher than 125 fps is not supported',
    );
    return;
  }
  if (fps > 0) {
    yieldInterval = Math.floor(1000 / fps);
  } else {
    // reset the framerate
    yieldInterval = 5;
  }
}

shouldYield = shouldYieldToHost = () => {
	return getCurrentTime() >= deadline;
}
```





#### 优先级调度

`Scheduler`是独立于`React`的包，所以他的`优先级`也是独立于`React`的`优先级`的。

`Scheduler`对外暴露了一个方法unstable_runWithPriority。

这个方法接受一个`优先级`与一个`回调函数`，在`回调函数`内部调用获取`优先级`的方法都会取得第一个参数对应的`优先级`：

```js
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}
```

可以看到，`Scheduler`内部存在5种优先级。

在`React`内部凡是涉及到`优先级`调度的地方，都会使用`unstable_runWithPriority`。

比如，`commit`阶段是同步执行的。`commit`阶段的起点`commitRoot`方法的优先级为`ImmediateSchedulerPriority`。

`ImmediateSchedulerPriority`即`ImmediatePriority`的别名，为最高优先级，会立即执行。

```js
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority(
    ImmediateSchedulerPriority,
    commitRootImpl.bind(null, root, renderPriorityLevel),
  );
  return null;
}
```



#### 优先级的意义

`Scheduler`对外暴露最重要的方法便是unstable_scheduleCallback。该方法用于以某个`优先级`注册回调函数。

比如在`React`中，之前讲过在`commit`阶段的`beforeMutation`阶段会调度`useEffect`的回调：

```js
if (!rootDoesHavePassiveEffects) {
  rootDoesHavePassiveEffects = true;
  scheduleCallback(NormalSchedulerPriority, () => {
    flushPassiveEffects();
    return null;
  });
}
```

这里的回调便是通过`scheduleCallback`调度的，优先级为`NormalSchedulerPriority`，即`NormalPriority`。

不同`优先级`意味着什么？不同`优先级`意味着不同时长的任务过期时间：

```js
var timeout;
switch (priorityLevel) {
  case ImmediatePriority:
    timeout = IMMEDIATE_PRIORITY_TIMEOUT;
    break;
  case UserBlockingPriority:
    timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
    break;
  case IdlePriority:
    timeout = IDLE_PRIORITY_TIMEOUT;
    break;
  case LowPriority:
    timeout = LOW_PRIORITY_TIMEOUT;
    break;
  case NormalPriority:
  default:
    timeout = NORMAL_PRIORITY_TIMEOUT;
    break;
}

var expirationTime = startTime + timeout;
```

其中：

```js
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
```

可以看到，如果一个任务的`优先级`是`ImmediatePriority`，对应`IMMEDIATE_PRIORITY_TIMEOUT`为`-1`，那么

```js
var expirationTime = startTime - 1;
```

则该任务的过期时间比当前时间还短，表示他已经过期了，需要立即被执行。



####  不同优先级任务的排序

我们已经知道`优先级`意味着任务的过期时间。设想一个大型`React`项目，在某一刻，存在很多不同`优先级`的`任务`，对应不同的过期时间。

同时，又因为任务可以被延迟，所以我们可以将这些任务按是否被延迟分为：

- 已就绪任务
- 未就绪任务

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      // 任务被延迟
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    // 判断是否存在过期任务以及当前任务是否是最早过期的未过期任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

所以，`Scheduler`存在两个队列：

- timerQueue：保存未就绪任务
- taskQueue：保存已就绪任务

每当有新的未就绪的任务被注册，我们将其插入`timerQueue`并根据开始时间重新排列`timerQueue`中任务的顺序。

当`timerQueue`中有任务就绪，即`startTime <= currentTime`，我们将其取出并加入`taskQueue`。

取出`taskQueue`中最早过期的任务并执行他。

为了能在O(1)复杂度找到两个队列中时间最早的那个任务，`Scheduler`使用**小顶堆**实现了`优先级队列`。



> 那么当shouldYield为true，以至于performUnitOfWork被中断后是如何重新启动的呢？

在“取出`taskQueue`中最早过期的任务并执行他”这一步中有如下关键步骤：

```js
const continuationCallback = callback(didUserCallbackTimeout);
currentTime = getCurrentTime();
if (typeof continuationCallback === 'function') {
  // continuationCallback是函数
  currentTask.callback = continuationCallback;
  markTaskYield(currentTask, currentTime);
} else {
  if (enableProfiling) {
    markTaskCompleted(currentTask, currentTime);
    currentTask.isQueued = false;
  }
  if (currentTask === peek(taskQueue)) {
    // 将当前任务清除
    pop(taskQueue);
  }
}
advanceTimers(currentTime);
```

当注册的回调函数执行后的返回值`continuationCallback`为`function`，会将`continuationCallback`作为当前任务的回调函数。

如果返回值不是`function`，则将当前被执行的任务清除出`taskQueue`。

`render`阶段被调度的函数为`performConcurrentWorkOnRoot`，在该函数末尾有这样一段代码：

```js
if (root.callbackNode === originalCallbackNode) {
  // The task node scheduled for this root is the same one that's
  // currently executed. Need to return a continuation.
  return performConcurrentWorkOnRoot.bind(null, root);
}
```

可以看到，在满足一定条件时，该函数会将自己作为返回值。



### lane模型

满足以下条件

- 可以表示`优先级`的不同
- 可能同时存在几个同`优先级`的`更新`，所以还得能表示`批`的概念
- 方便进行`优先级`相关计算



使用31位的二进制表示优先级，位数越小的优先级最高，相邻的位数拥有相同的优先级。其中同步优先级为最高优先级。

从`SyncLane`往下一直到`SelectiveHydrationLane`，`优先级`逐步降低。

```js
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;

const InputContinuousHydrationLane: Lane = /*           */ 0b0000000000000000000000000100000;
const InputContinuousLanes: Lanes = /*                  */ 0b0000000000000000000000011000000;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000100000000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000001000000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000011110000000000000000000000;

export const SomeRetryLane: Lanes = /*                  */ 0b0000010000000000000000000000000;

export const SelectiveHydrationLane: Lane = /*          */ 0b0000100000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0000111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0001000000000000000000000000000;
const IdleLanes: Lanes = /*                             */ 0b0110000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```



#### 表示“批”的概念

```js
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;
```

这就是`批`的概念，被称作`lanes`（区别于`优先级`的`lane`）。

其中`InputDiscreteLanes`是“用户交互”触发更新会拥有的`优先级`范围。

`DefaultLanes`是“请求数据返回后触发更新”拥有的`优先级`范围。

`TransitionLanes`是`Suspense`、`useTransition`、`useDeferredValue`拥有的`优先级`范围。

越低`优先级`的`lanes`占用的位越多。比如`InputDiscreteLanes`占了2个位，`TransitionLanes`占了9个位。

原因在于：越低`优先级`的`更新`越容易被打断，导致积压下来，所以需要更多的位。相反，最高优的同步更新的`SyncLane`不需要多余的`lanes`。



#### 方便进行优先级相关计算

`优先级`相关计算其实就是位运算。

比如：

计算`a`、`b`两个`lane`是否存在交集，只需要判断`a`与`b`按位与的结果是否为`0`：

```js
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}
```

计算`b`这个`lanes`是否是`a`对应的`lanes`的子集，只需要判断`a`与`b`按位与的结果是否为`b`：

```js
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}
```

将两个`lane`或`lanes`的位合并只需要执行按位或操作：

```js
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}
```

从`set`对应`lanes`中移除`subset`对应`lane`（或`lanes`），只需要对`subset`的`lane`（或`lanes`）执行按位非，结果再对`set`执行按位与。

```js
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
```



#### Lane模型中task是怎么获取优先级的

```js
export function findUpdateLane(
  lanePriority: LanePriority,
  wipLanes: Lanes,
): Lane {
  switch (lanePriority) {
    //...
    case DefaultLanePriority: {
      let lane = pickArbitraryLane(DefaultLanes & ~wipLanes);//找到下一个优先级最高的lane
      if (lane === NoLane) {//上一个level的lane都占满了下降到TransitionLanes继续寻找可用的其他优先级中的位置
        lane = pickArbitraryLane(TransitionLanes & ~wipLanes);
        if (lane === NoLane) {//TransitionLanes也满了
          lane = pickArbitraryLane(DefaultLanes);//从DefaultLanes开始找
        }
      }
      return lane;
    }
  }
}
```



#### Lane模型中高优先级是怎么插队的

 在Lane模型中如果一个低优先级的任务执行，并且还在调度的时候触发了一个高优先级的任务，则高优先级的任务打断低优先级任务，此时应该先取消低优先级的任务，因为此时低优先级的任务可能已经进行了一段时间，Fiber树已经构建了一部分，所以需要将Fiber树还原，这个过程发生在函数prepareFreshStack中，在这个函数中会初始化已经构建的Fiber树

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;//之前已经调用过的setState的回调
  //...
	if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    //新的setState的回调和之前setState的回调优先级相等 则进入batchedUpdate的逻辑
    if (existingCallbackPriority === newCallbackPriority) {
      return;
    }
    //两个回调优先级不一致，则被高优先级任务打断，先取消当前低优先级的任务
    cancelCallback(existingCallbackNode);
  }
	//调度render阶段的起点
	newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
	//...
}

function unstable_cancelCallback(task) {
  if (enableProfiling) {
    if (task.isQueued) {
      const currentTime = getCurrentTime();
      markTaskCanceled(task, currentTime);
      task.isQueued = false;
    }
  }

  task.callback = null;
}
```



```js
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
	//...
  //workInProgressRoot等变量重新赋值和初始化
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
	//...
}
```



#### Lane模型中怎么解决饥饿问题（最低优先级的任务同样需要完成）

 在调度优先级的过程中，会调用markStarvedLanesAsExpired遍历pendingLanes（未执行的任务包含的lane），如果没过期时间就计算一个过期时间，如果过期了就加入root.expiredLanes中，然后在下次调用getNextLane函数的时候会优先返回expiredLanes

```js
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number,
): void {

  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {//遍历lanes
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {

      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        expirationTimes[index] = computeExpirationTime(lane, currentTime);//计算过期时间
      }
    } else if (expirationTime <= currentTime) {//过期了
      root.expiredLanes |= lane;//在expiredLanes加入当前遍历到的lane
    }

    lanes &= ~lane;
  }
}
```



```js
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
 	//...
  if (expiredLanes !== NoLanes) {
    nextLanes = expiredLanes;
    nextLanePriority = return_highestLanePriority = SyncLanePriority;//优先返回过期的lane
  } else {
  //...
    }
  return nextLanes;
}
```
