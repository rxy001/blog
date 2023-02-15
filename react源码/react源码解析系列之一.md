### React 源码解析系列之一

***首先声明本系列所使用的源码为 18.2.0，legacy mode。[此仓库](https://github.com/rxy001/toy-react) 复制并简化了 react 中的源码，翻译源码中部分注释，并添加一些个人理解，建议搭配食用。其次本文只考虑函数式组件，有关类组件的源码都将忽略***

当查看 react repo 时会发现其中的 packages 非常多，在 web 端使用时，核心包有 react、react-dom、react-reconciler 及 scheduler。

#### react、react-dom、react-reconciler、scheduler

- *React* 负责对外暴露 API ，React Element（即 Virtual DOM）的创建。

- *ReactDOM* 作为 React 的 DOM 渲染的入口点，负责 DOM 的创建、更新等操作。

- *ReactReconciler* 是 React 核心所在，众所周知的 fiber 架构即其核心算法。

- *scheduler* 为 ReactReconciler 提供任务调度的实现。

以上4个包，主要研究对象是 **react-reconciler**。

---

```js
const element = <h1>Hello, world!</h1>;
```

在使用 React 时，通常都会搭配 JSX， JSX 语法类似于 HTML，是一个 JavaScript 的语法扩展。编译器会将其转换为浏览器可以理解的 React 函数调用，旧的 JSX 转化会把 JSX 转化为`React.createElement(...)` 调用 （17 版本以下）。

```js
const element = React.createElement('h1', null, 'Hello world');
```

新的 JSX 转化自动从 React 的 package 中引入新的入口函数并调用，因此源代码中不需要再主动引入 React 。

```js
// 由编译器引入（禁止自己引入！）
import {jsx as _jsx} from 'react/jsx-runtime';

const element = _jsx('h1', { children: 'Hello world' });

```

`_jsx` 函数返回值即是 React Element：

```js
{
    $$typeof: ReactElementSymbol,
    type: "h1",
    key: null,
    props: {
      children: 'Hello world'
    },
    ref: null,
}
```

---

![render](https://tva1.sinaimg.cn/large/008vxvgGly1h8rvsndahij30jr0dxgm0.jpg)

在入口文件中，都会先调用 `React.render(document.getElementById("root"))`，将 `root` 作为根节点。

> **为什么需要 root 节点 ？**
>
> 在 react-reconciler 中，所有的组件以及 DOM 元素都将转化为 fiber ，fiber 通过 return、child、sibling 三个指针表示了其中的关系，以 root 为起始节点形成链表，因此使得能够递归遍历 fiber tree。

`React.render()` 过程会先调用 `createFiberRoot ` 创建 `fiberRoot` 及 `rootFiber`，并为 `rootFiber` 初始化 `updateQueue、memoizedState` 。

```js
function createFiberRoot(container, tag, initialChildren = null) {
  const root = new FiberRootNode(container, tag);

  const uninitializedFiber = createHostRootFiber(tag);

  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  uninitializedFiber.memoizedState = {
    element: initialChildren,
  };

  initializeUpdateQueue(uninitializedFiber);

  return root;
}

function initializeUpdateQueue(fiber) {
  const queue = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
      interleaved: null,
    },
    effects: null,
  };
  fiber.updateQueue = queue;
}
```

> **fiberRoot 与 rootFiber 区别**
>
> 1. fiberRoot 会保存在 `container._reactRootContainer`，使得同一 container 多次调用 render 时只会创建一次 fiberRoot。 每次更新时，rootFiber 都会新建或复用之前的 fiber ，fiberRoot 不会。
> 2. fiberRoot 与 rootFiber 结构不同。fiberRoot 保存 fiber 构建过程中所依赖的全局状态，作为 reconciler 运行期间的全局上下文。 `fiberRoot.current === rootFiber,rootFiber.stateNode === fiberRoot `，rootFiber 是 fiber tree 的起点。
>
> **渲染阶段，为什么需要两个 update queue （firstBaseUpdate、lastBaseUpdate）**

`updateContainer` `element` 即 `React.render` 的第一个参数 `<App/>`。`enqueueUpdate` 将 `update` 添加到 `rootFiber` 的 `updateQueue.shared.interleaved`，通过 `update.next`，将所有 `update` 对象连接起来形成单向循环链表，`interleaved` 为链表末端即最新的 `update`。

```js
function updateContainer(element, container, parentComponent) {
  const current = container.current;
  
  const update = createUpdate();

  update.payload = { element };

  const root = enqueueUpdate(current, update);
  if (root !== null) {
    scheduleUpdateOnFiber(root, current);
  }
}
function createUpdate() {
  const update = {
    payload: null,
    callback: null,
    next: null,
  };
  return update;
}
// queue === fiber.updateQueue.shared
function enqueueConcurrentClassUpdate(fiber, queue, update) {
  const interleaved = queue.interleaved;
  if (interleaved === null) {
    update.next = update;
    pushConcurrentUpdateQueue(queue);
  } else {
    // interleaved.next === firstUpdate
    update.next = interleaved.next;
    // interleaved === lastUpdate
    interleaved.next = update;
  }
  queue.interleaved = update;
  return markUpdateLaneFromFiberToRoot(fiber);
}

// 包含本次渲染所接收的 update 的所有 update queues，当本次渲染退出时，无论是因为它完成了还是因为它被中断了，插入的更新都将被转移到队列的主要部分。
let concurrentQueues = null;
function pushConcurrentUpdateQueue(queue) {
  if (concurrentQueues === null) {
    concurrentQueues = [queue];
  } else {
    concurrentQueues.push(queue);
  }
}
```

![schedule.drawio](https://tva1.sinaimg.cn/large/008vxvgGly1h8s5um29l4j30700dnjrj.jpg)

之后调用 `scheduleUpdateOnFiber` 开始任务调度。每一个 `fiberRoot` 仅有一个任务，`ensureRootIsScheduled` 会注册调度任务 `performSyncWorkOnRoot` 。因为初始挂载 `updateContainer` 是在 `flushSync` 内部调用，因此注册完成后将直接调用 `performSyncWorkOnRoot`。

```js
function ensureRootIsScheduled(root, fiber) {
  if (root.tag === LegacyRoot) {
    scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else {
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  }
}

function legacyCreateRootFromDOMContainer(
  container,
  initialChildren,
  parentComponent
) {
  ...
  flushSync(() => {
    updateContainer(initialChildren, root, parentComponent);
  });
}
```

`performSyncWorkOnRoot` 分为两个阶段：`render（renderRootSync）`  和  `commit （commitRoot）`，`render` 阶段构造 `fiber tree`，commit 阶段渲染 `fiber tree`。

```js
function performSyncWorkOnRoot(root) {
  let exitStatus = renderRootSync(root);

  const finishedWork = root.current.alternate;
  root.finishedWork = finishedWork;

  commitRoot(root);

  return null;
}
```

在构建 `fiber tree` 过程中，内存中将会存在两个 `fiber tree` 即 `current ` 、 `workInProgress` 。`current` 表示当前屏幕显示内容对应的 `fiber tree` ，可通过 `fiberRoot.current` 访问，`workInProgress` 表示将要渲染到屏幕的内容对应的 `fiber tree` 。`current` 和 `workInProgress` 中的节点，通过 `alternate` 连接。

![fiber](https://tva1.sinaimg.cn/large/008vxvgGly1h8s6unleecj30h30dtgm2.jpg)

在构建 `workInProgress` 之前`（workLoopSync）`，先刷新栈帧 `prepareFreshStack`，`createWorkInProgress` 创建 `rootWorkInProgress` ，重置 `workInProgressRoot` 和 `workInProgress` 。此时 `fiberRoot.current(rootFiber)` 和 `workInProgress`  通过 `alternate` 相互引用。

```js
function renderRootSync(root) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;

  // 如果 fiberRoot 发生了变化，抛出现有堆栈并准备一个新的堆栈。否则我们将继续停止的地方。
  if (workInProgressRoot !== root) {
    prepareFreshStack(root);
  }
  
  workLoopSync();
  workInProgressRoot = null;
  executionContext = prevExecutionContext;
	...
}
function prepareFreshStack(root) {
  root.finishedWork = null;

  workInProgressRoot = root;
  const rootWorkInProgress = createWorkInProgress(root.current, null);
  workInProgress = rootWorkInProgress;

	// 将 iterleaved updates 转移到 pending queue。 每个队列都有一个 pending 字段和 interleaved 字段。
  // 当他们都不为 null 时， 他们的指针都指向环状链表的最后一个节点。我们需要将 interleaved 链表添加到
  // pending 链表的最后一个节点，形成一个环状链表。
  finishQueueingConcurrentUpdates();

  return rootWorkInProgress;
}
```

`workLoopSync` 将进入循环构建 `fiber`

```js
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

`performUnitOfWork` 采用深度优先算法递归生成 `fiber`。 `workInProgress` 是将要构建的 `fiber` 的父 `fiber` ，`current` 是 `workInProgress.alternate`  。

```js
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  // next === workInProgress.child
  let next = beginWork(current, unitOfWork);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

