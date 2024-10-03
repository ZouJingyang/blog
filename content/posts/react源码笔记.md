+++
title = "React 源码学习笔记"
date = "2024-10-01"

[taxonomies]
tags=["前端","React"]
+++

## 启动过程

react入口文件：

```jsx
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### createRoot 函数

主要任务：

- 调用 `createContainer` 方法，传入容器和配置选项，创建一个新的 Fiber 根节点。
- 调用 `markContainerAsRoot` 方法，标记 Dom 对象, 把 Dom 和 Fiber 对象关联起来。
- 调用 `listenToAllSupportedEvents` 方法，监听所有支持的事件。
- 返回一个新的 `ReactDOMRoot` 实例，该实例包含了内部的 Fiber 根节点。

注意事件：

-  `createContainer` 中创建了 **fiberRoot** 对象与 **HostRootFiber** 对象，并且将它们关联
-  `markContainerAsRoot` 是将 **HostRootFiber** 与 容器元素进行了关联

### render 方法

 `ReactDOMRoot` 原型上挂载的方法，调用来自 `reconciler` 中的 `updateContainer` 函数对元素进行渲染

之后会调用 `reconciler` 中的 `scheduleUpdateOnFiber` 函数 -> `ensureRootIsScheduled` 函数 进入任务调度

## 任务调度

调用 `scheduler` 中的 `unstable_scheduleCallback` 函数，将外部传入的一个回调函数`performConcurrentWorkOnRoot` 注册为 task，并添加到 `taskQueue` 中，之后调用 `requestHostCallback` 通过 `MessageChannel` 去执行回调函数（`flushWork`， 宏任务，异步调用）；

`flushWork` 中会调用 `workLoop` 创建任务调度循环，去清空 `taskQueue` ，之前被添加进来的任务会被取出进行执行

## render阶段

`performConcurrentWorkOnRoot` 中 会调用 `renderRootConcurrent` 函数 -> `workLoopConcurrent` 函数， 创建一个构建fiber树的循环，会持续执行下面两个阶段，直到fiber树被构建成功

### beginWork 阶段

会从**HostRootFiber**节点开始，进行深度优先遍历，这里存在两个指针：

- `workInProgress` ：指向当前正在构造的`fiber`节点
- `current` : 指向当前页面正在使用的`fiber`节点. 初次构造时, 页面还未渲染, 此时`current = null`.

两个指针之间的关系是 `current` = `workInProgress.alternate`

#### 具体工作

1. **类型判断**：
   - 根据当前 Fiber 节点的类型（如函数组件、类组件、原生 DOM 元素等），调用相应的更新函数。
2. **子节点的协调**：
   - 通过 `renderWithHooks` 拿到 `nextChildren`， 调用 `reconcileChildren` 函数，将新生成的子元素与当前 Fiber 节点的子元素进行比较。
   - 根据比较结果，创建、更新或删除子 Fiber 节点。
3. **准备工作**：
   - 计算并更新当前 Fiber 节点的 `props`、`state` 等信息。
   - 为子节点创建必要的 Fiber 节点，并将它们连接到当前 Fiber 节点上。

持续上面的流程，直到 `workInProgress` 指针指向 null

### completeWork 阶段

#### 具体工作

1. **DOM 创建与更新**：
   - 如果当前 Fiber 节点对应一个原生 DOM 元素，`completeWork` 会负责创建或更新实际的 DOM 节点。
   - 将创建的 DOM 节点与 Fiber 树中的节点关联起来（stateNode）。
2. **副作用的收集**：
   - 收集当前节点的副作用（如插入、更新、删除等操作）形成一个队列，随着遍历一步一步将其上移到 HostRootFiber上，以便在 commit 阶段应用到实际的 DOM 树上。
   - 副作用队列的顺序是层级越深子节点越靠前.
3. **返回父节点**：
   - 处理完当前节点后，将控制权返回给父节点，继续处理父节点的其他子节点。

## Commit 阶段

Commit Phase 是 React 渲染过程中的一个关键阶段，它负责将已经计算好的 DOM 操作应用到实际的 DOM 树中
主要在于以下三个函数：`commitBeforeMutationEffects`、`commitMutationEffects`、`commitLayoutEffects`

### Before Mutation 阶段

- 遍历 Fiber 树的每个节点，检查并处理节点的 Before Mutation Effects。
  - 如果启用了 `enableCreateEventHandleAPI` 并且需要处理焦点相关逻辑，则执行相关操作。
  - 对于有 `Snapshot` 标志的 Fiber 节点，调用 `getSnapshotBeforeUpdate` 方法获取快照信息。
  - 对于 `ClassComponent` 类型的 Fiber 节点，调用 `getSnapshotBeforeUpdate` 方法并保存返回的快照信息。

### Mutation 阶段

下面是Mutation 阶段的主要函数的缩略，涉及到了两个重要的函数： `recursivelyTraverseMutationEffects`、`commitReconciliationEffects`

```js
function commitMutationEffectsOnFiber(finishedWork, root) {
  switch (finishedWork.tag) {
    case HostRoot:
    case HostComponent:
    case HostText: {
      // 递归处理 Fiber 树
      recursivelyTraverseMutationEffects(finishedWork);
      // 处理自身节点的副作用
      commitReconciliationEffects(finishedWork);
      break;
    }
  }
}
```

 `recursivelyTraverseMutationEffects`

1. **处理删除效果**：
   - 检查当前 Fiber 节点是否有删除效果 (`deletions`)。
   - 如果有，遍历 `deletions` 数组，对每个需要删除的子节点调用 `commitDeletionEffects` 函数。
     - 执行子树所有组件的`unmount`卸载逻辑。
     - 执行子树某些类组件的`componentWillUnmount`方法。
     - 执行子树某些函数组件的`useEffect`，`useLayoutEffect`等hooks的`destory`销毁方法。
     - 执行子树所有`ref`属性的卸载操作。
2. **递归处理子节点**：
   - 检查当前 Fiber 节点的子树标志 (`subtreeFlags`) 是否包含 Mutation Effects (`MutationMask`)。
   - 如果包含，遍历当前 Fiber 的子节点，并递归调用 `commitMutationEffectsOnFiber` 函数处理每个子节点的变更效果。
   - 在处理每个子节点时，记录当前调试 Fiber 节点，以便在开发环境中进行调试。

`commitReconciliationEffects`

**处理插入效果**：

- 检查当前 Fiber 节点的标志 (`flags`) 是否包含插入效果 (`Placement`)。
- 如果包含，调用 `commitPlacement` 函数执行插入操作。
- 在调用 `commitPlacement` 时，捕获并处理可能发生的错误。
- 清除当前 Fiber 节点的插入标志 (`Placement`)，以确保在后续生命周期方法调用之前该节点已经被插入。

除了这两个函数之外，对不同的组件类型也会进行不同的处理，例如 HostComponent，会进行重置`ref`对象和重置`dom`内容，最后更新dom的属性和样式。

### Layout 阶段

在真实DOM加载完成后：

- 执行函数组件的`useLayoutEffect hook`的回调。
- 执行类组件的生命周期钩子及`setState`相关回调。
- 执行`ref`对象的绑定。

注意 rootcurrent = finishedWork; 的设置，被放在了 Mutation 之后，Layout 之前。