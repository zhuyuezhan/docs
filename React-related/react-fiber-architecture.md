https://github.com/acdlite/react-fiber-architecture

### 介绍
React Fiber 是 React 核心算法的持续重实现。这是 React 团队经过两年多研究的成果。

React Fiber 的目标是提高其在动画、布局和手势等领域的适用性。其主要特点是增量渲染：将渲染工作分成多个块，并在多个帧之间分散。

其他关键功能包括在新更新到来时暂停、中止或重用工作；为不同类型的更新分配优先级；以及新的并发原语。

#### 关于本文档
Fiber 引入了几个仅通过查看代码难以理解的新概念。本文档最初是我在跟随 Fiber 实现过程中的笔记集合。随着内容的增加，我意识到它可能对其他人也有帮助。

我将尽量使用最通俗的语言，避免使用术语，并明确定义关键术语。我也会尽可能多地链接到外部资源。

请注意，我不是 React 团队的成员，也不具有权威性。这不是官方文档。我已请 React 团队的成员审核其准确性。

这也是一个正在进行的项目。Fiber 是一个持续的项目，在完成之前可能会进行重大重构。我的文档化尝试也是持续进行的。欢迎提出改进和建议。

我的目标是，通过阅读本文档，你将能够理解 Fiber 的原理，跟随其实现过程，并最终能够为 React 做出贡献。

#### 先决条件
在继续之前，我强烈建议你熟悉以下资源：

[React](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) 组件、元素和实例 - "组件"是一个常用的术语。掌握这些术语非常重要。
[Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - React Reconciliation 算法的高级描述。
[React 基本理论概念](https://github.com/reactjs/react-basic) - 无需考虑实现负担的 React 概念模型描述。有些内容在第一次阅读时可能不会理解。这没关系，随着时间推移会更明白。
[React 设计原则](https://facebook.github.io/react/contributing/design-principles.html) - 特别注意调度部分。它很好地解释了 React Fiber 的原因。
### 回顾
在我们深入研究新内容之前，让我们回顾一些概念。

#### 什么是 Reconciliation？
**reconciliation** 是 React 用于比较两棵树以确定需要更改哪些部分的算法。
**update** 是用于渲染 React 应用的数据更改。通常是 setState 的结果，最终导致重新渲染。
React API 的核心思想是将更新视为导致整个应用重新渲染。这允许开发者以声明方式思考，而不必担心如何高效地将应用从一个状态转换到另一个状态。

实际在每次更改时重新渲染整个应用只适用于最简单的应用；在现实应用中，这在性能方面的代价过高。React 进行了优化，使整个应用看起来像重新渲染，同时保持高性能。这些优化大部分是 Reconciliation 过程的一部分。

**Reconciliation** 是广为人知的 "虚拟 DOM" 背后的算法。大致描述如下：**当你渲染一个 React 应用时，会生成并保存在内存中的节点树。这棵树随后会被刷新到渲染环境中——例如，在浏览器应用中，它被转换为一组 DOM 操作。当应用更新时（通常通过 setState），会生成一棵新树。新树与之前的树进行差异计算，以确定更新渲染应用所需的操作。**

尽管 Fiber 是重新实现的 Reconciler，但 React 文档中描述的高级算法大致相同。关键点是：

**不同的组件类型被假定生成实质上不同的树。React 不会尝试比较它们，而是完全替换旧树。**
**列表的差异计算使用key。key应该是“稳定的、可预测的和唯一的”。**

#### Reconciliation 与渲染
DOM 只是 React 可以渲染的一个环境，其他主要目标是通过 React Native 实现的原生 iOS 和 Android 视图。（这就是为什么 "虚拟 DOM" 有点误导。）

它能够支持如此多目标的原因是 React 设计为 Reconciliation 和渲染是分开的阶段。Reconciler 负责计算树的哪些部分发生了变化；然后渲染器使用这些信息实际更新渲染的应用。

这种分离意味着 React DOM 和 React Native 可以使用自己的渲染器，同时共享由 React 核心提供的相同 Reconciler。

Fiber 重新实现了 Reconciler。它主要不关心渲染，尽管渲染器需要更改以支持（并利用）新架构。

#### 调度
**调度**是确定何时执行工作的过程。

工作是指必须执行的任何计算。工作通常是更新的结果（例如 setState）。

React 的设计原则文档在这个主题上非常出色，我将直接引用：

在当前实现中，React 递归遍历树，并在一个单一的 tick 中调用整个更新树的 render 函数。然而，将来它可能会开始延迟某些更新以避免丢帧。

这是 React 设计中的一个常见主题。一些流行的库实现了“推送”方法，即在新数据可用时执行计算。然而，React 坚持“拉取”方法，即可以延迟计算直到必要时再执行。

React 不是一个通用的数据处理库。它是一个用于构建用户界面的库。我们认为它在应用中处于独特的位置，能够知道哪些计算是当前相关的，哪些不是。

如果某些内容在屏幕外，我们可以延迟任何与其相关的逻辑。如果数据到达的速度超过了帧率，我们可以合并和批处理更新。我们可以优先处理来自用户交互的工作（例如按钮点击引起的动画）而不是不太重要的后台工作（例如渲染刚从网络加载的新内容）以避免丢帧。

关键点是：
在 UI 中，不需要每次更新都立即应用；事实上，这样做可能会浪费，导致丢帧和用户体验下降。
不同类型的更新有不同的优先级——动画更新需要比数据存储更新更快地完成。
推送方法需要应用（你，程序员）决定如何调度工作。拉取方法允许框架（React）智能地为你做这些决策。
React 目前没有显著利用调度；更新导致整个子树立即重新渲染。重新设计 React 核心算法以利用调度是 Fiber 背后的驱动思想。
现在我们准备深入研究 Fiber 的实现。下一节比我们迄今讨论的内容更技术化。请确保你对之前的材料感到舒适，然后再继续。

什么是 Fiber？
我们将讨论 React Fiber 架构的核心。Fiber 是比应用开发者通常考虑的抽象层次低得多的抽象。如果你在理解它时感到沮丧，不要气馁。继续尝试，它最终会明白。（当你最终明白时，请建议如何改进本节。）

我们开始吧！

我们已经确定 Fiber 的主要目标是使 React 能够利用调度。具体来说，我们需要能够：

暂停工作并在以后继续。
为不同类型的工作分配优先级。
重用先前完成的工作。
如果不再需要，可以中止工作。
为了做到这一点，我们首先需要一种方法将工作分解成单元。从某种意义上说，这就是 Fiber 的作用。Fiber 表示一个工作单元。

进一步讨论之前，让我们回到 React 组件作为数据函数的概念，通常表示为

**𝑣=𝑓(𝑑)**

这意味着渲染一个 React 应用类似于调用一个函数，该函数体包含对其他函数的调用，依此类推。当考虑 Fiber 时，这个类比非常有用。

计算机通常使用调用栈跟踪程序的执行。当一个函数执行时，一个新的栈帧被添加到栈中。这个栈帧表示该函数执行的工作。

在处理 UI 时，问题在于如果一次执行过多工作，会导致动画丢帧，看起来卡顿。更重要的是，如果这些工作被更近期的更新取代，其中一些工作可能是不必要的。这就是 UI 组件与一般函数的比较失效的地方，因为组件有比函数更具体的关注点。

较新的浏览器（和 React Native）实现了帮助解决这个问题的 API：**requestIdleCallback** 调度一个在空闲期间调用的低优先级函数，**requestAnimationFrame** 调度一个在下一帧调用的高优先级函数。问题是，为了使用这些 API，你需要一种方法将渲染工作分解成增量单元。如果你只依赖于调用栈，它会一直做工作，直到栈空。

如果我们能自定义调用栈的行为，以优化渲染 UI 的性能，那该多好啊？如果我们能随意中断调用栈并手动操作栈帧，那该多好啊？

这就是 React Fiber 的目的。Fiber 是重新实现的栈，专为 React 组件设计。你可以将单个 Fiber 视为虚拟栈帧。

重新实现栈的好处是，你可以将栈帧保存在内存中，并随时（以任何方式）执行它们。这对于实现调度目标至关重要。

除了调度，手动处理栈帧还解锁了并发和错误边界等功能的潜力。我们将在后续章节中讨论这些主题。

在下一节中，我们将更多地探讨 Fiber 的结构。

Fiber 的结构
注意：随着我们对实现细节讨论的深入，某些信息可能会发生变化的可能性增加。如果你发现任何错误或过时的信息，请提交 PR。

具体来说，Fiber 是包含有关组件、其输入和输出信息的 JavaScript 对象。

Fiber 对应于栈帧，但它也对应于组件的实例。

以下是一些重要的 Fiber 字段。（此列表并不详尽。）

**type** 和 **key**
Fiber 的 type 和 key 与 React 元素中的作用相同。（实际上，当从元素创建 Fiber 时，这两个字段会直接复制过来。）

Fiber 的 type 描述了它对应的组件。对于组合组件，type 是函数或类组件本身。对于宿主组件host（如 div、span 等），type 是字符串。

从概念上讲，type 是函数（如 v = f(d)）其执行由栈帧跟踪。

与 type 一起，key 在 Reconciliation 期间用于确定 Fiber 是否可以重用。

**child** 和 **sibling**
这些字段指向其他 Fiber，描述了 Fiber 的递归树结构。

子 Fiber 对应于组件 render 方法返回的值。例如：

```javascript
function Parent() {
  return <Child />
}
```
Parent 的子 Fiber 对应于 Child。

sibling 字段解释了 render 返回多个子项的情况（Fiber 中的新功能！）：

```javascript
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

子 Fiber 形成单链表，其头部是第一个子项。所以在这个例子中，Parent 的子项是 Child1，Child1 的兄弟是 Child2。

回到函数类比，你可以将子 Fiber 视为尾调用的函数。

**return**
返回 Fiber 是在处理当前 Fiber 后程序应返回的 Fiber。从概念上讲，它与栈帧的返回地址相同。也可以认为是父 Fiber。

如果一个 Fiber 有多个子 Fiber，每个子 Fiber 的返回 Fiber 是父 Fiber。所以在前面的例子中，Child1 和 Child2 的返回 Fiber 是 Parent。

**pendingProps** 和 **memoizedProps**
从概念上讲，props 是函数的参数。Fiber 的 pendingProps 在执行开始时设置，memoizedProps 在结束时设置。

当传入的 pendingProps 等于 memoizedProps 时，这表明可以重用 Fiber 的先前输出，防止不必要的工作。

**pendingWorkPriority**
表示 Fiber 所代表工作的优先级的数字。ReactPriorityLevel 模块列出了不同的优先级及其代表的内容。

除了 NoWork（为 0）外，较大数字表示较低优先级。例如，你可以使用以下函数检查 Fiber 的优先级是否至少与给定级别一样高：

```javascript
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority;
}
```

此函数仅用于说明；它实际上不属于 React Fiber 代码库。

调度程序使用优先级字段来搜索下一个要执行的工作单元。我们将在后续章节中讨论该算法。

**alternate**
flush 表示将 Fiber 渲染到屏幕上。
work-in-progress 是尚未完成的 Fiber；从概念上讲，是尚未返回的栈帧。
在任何时候，一个组件实例最多有两个对应的 Fiber：当前的、已刷新的 Fiber 和进行中的工作 Fiber。

当前 Fiber 的 alternate 是进行中的工作，进行中的工作 Fiber 的 alternate 是当前 Fiber。

alternate 是通过一个称为 cloneFiber 的函数懒创建的。cloneFiber 会尝试重用 Fiber 的 alternate（如果存在），而不是总是创建一个新对象，最小化分配。

你应该将 alternate 字段视为实现细节，但它在代码库中经常出现，因此在此讨论它是有价值的。

**output**
宿主组件是 React 应用的叶节点。它们是特定于渲染环境的（例如，在浏览器应用中，它们是 div、span 等）。在 JSX 中，它们用小写标签名表示。
从概念上讲，Fiber 的输出是函数的返回值。

每个 Fiber 最终都有输出，但输出仅在由宿主组件创建的叶节点处创建。然后将输出传递到树上。

输出是最终交给渲染器以便将更改刷新到渲染环境的内容。定义输出如何创建和更新是渲染器的责任。