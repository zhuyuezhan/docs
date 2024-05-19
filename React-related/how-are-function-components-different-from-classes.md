[原文](https://overreacted.io/how-are-function-components-different-from-classes/)

# React函数组件与React类组件有何不同？

长久以来，标准答案一直是类组件提供了更多功能（例如状态管理）。但是有了Hooks，这种说法已经不再正确了。

也许你听说过性能上其中一种更好。到底是哪一种呢？许多这样的基准测试是有缺陷的，所以我会谨慎地从中得出结论。性能主要取决于代码的实际操作，而不是选择函数或类。在我们的观察中，性能差异可以忽略不计，但优化策略略有不同。

那么我们该怎么做呢？React函数组件和类组件之间是否存在根本区别？当然有——在**思维模型**上。在这篇文章中，我将探讨它们之间最大的区别。自2015年引入函数组件以来，它一直存在，但往往被忽视：

**函数组件会捕获渲染时的值**。

考虑这个函数组件：

```javascript
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

它对应的类组件实现：

```javascript
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

注意，区别与React Hooks本身无关。上面的例子甚至没有使用Hooks！
这完全是关于React中函数和类之间的区别。如果你打算在React应用中更频繁地使用函数组件，你可能需要了解这一点。

在上面的ProfilePage函数组件中，点击Dan的配置文件上的“关注”按钮然后导航到Sophie的配置文件，仍然会提示“Followed Dan”。
而在上面的ProfilePage类组件中，它会提示“Followed Sophie”。

why？仔细看看我们类组件中的showMessage方法：
```javascript
class ProfilePage extends React.Component {
  showMessage = () => {
    alert('Followed ' + this.props.user);
  };
```

这个类方法读取的是this.props.user。在React中，props是不可变的，所以它们永远不会改变。然而，this却是可变的。

事实上，这正是类中this的整个目的。React本身会随着时间的推移改变它，以便你可以在渲染和生命周期方法中读取最新的版本。

所以，如果我们的组件在请求进行中重新渲染，this.props会改变。showMessage方法从“太新的”props中读取用户信息。

这揭示了关于用户界面的一个有趣视角。如果我们说一个UI在概念上是当前应用状态的函数，那么事件处理器是渲染结果的一部分——就像视觉输出一样。我们的事件处理器“属于”特定的渲染，具有特定的props和状态。

然而，调度一个回调读取this.props的超时会破坏这种关联。我们的showMessage回调不再“绑定”到任何特定的渲染，因此它“丢失”了正确的props。从this读取断开了这种连接。

>This class method reads from this.props.user. Props are immutable in React so they can never change. However, this is, and has always been, mutable.
>Indeed, that’s the whole purpose of this in a class. React itself mutates it over time so that you can read the fresh version in the render and lifecycle methods.
>So if our component re-renders while the request is in flight, this.props will change. The showMessage method reads the user from the “too new” props.
>This exposes an interesting observation about the nature of user interfaces. If we say that a UI is conceptually a function of current application state, **the event handlers are a part of the render result — just like the visual output**. Our event handlers “belong” to a particular render with particular props and state.
>However, scheduling a timeout whose callback reads this.props breaks that association. Our showMessage callback is not “tied” to any particular render, and so it “loses” the correct props. Reading from this severed that connection.

如果我们完全依赖JavaScript闭包，问题就会消失。

由于闭包的值随时间变化，闭包常常被避免。但是在React中，props和state是不可变的！（或者至少是强烈推荐的不可变的。）这消除了闭包的一个主要陷阱。

这意味着，如果你从特定渲染中闭包props或state，你可以始终指望它们保持完全相同：

```javascript
class ProfilePage extends React.Component {
  render() {
    // 捕获props！
    const props = this.props;

    // 注意：我们在*渲染内部*。
    // 这些不是类方法。
    const showMessage = () => {
      alert('Followed ' + props.user);
    };

    const handleClick = () => {
      setTimeout(showMessage, 3000);
    };

    return <button onClick={handleClick}>Follow</button>;
  }
}
```

你已经在渲染时“捕获”了props，这样，内部的任何代码（包括showMessage）都保证会看到这个特定渲染的props。React不再“移动我们的奶酪”。

然后，我们可以在内部添加任意多的辅助函数，它们都将使用捕获的props和state。闭包来救场！

上面的例子是正确的，但看起来很奇怪。如果你在渲染内部定义函数而不是使用类方法，那么为什么要有一个类“壳”呢？

确实，我们可以通过移除类“外壳”来简化代码：

```javascript
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```
就像上面一样，props仍然被捕获——React将它们作为参数传递。与this不同，props对象本身从未被React更改。

如果你在函数定义中解构props，这一点会更加明显：

```javascript
function ProfilePage({ user }) {
  const showMessage = () => {
    alert('Followed ' + user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```
当父组件使用不同的props渲染ProfilePage时，React将再次调用ProfilePage函数。但是我们已经点击的事件处理器“属于”之前的渲染，具有自己的user值和读取它的showMessage回调。它们都保持不变。

所以我们知道React中的函数默认情况下会捕获props和state。但是，如果我们想读取不属于此特定渲染的最新props或state怎么办？如果我们想“从未来读取它们”怎么办？

在类组件中，你会通过读取this.props或this.state来做到这一点，因为this本身是可变的。React会更改它。在函数组件中，你也可以有一个由所有组件渲染共享的可变值。这被称为“ref”：
```javascript
function MyComponent() {
  const ref = useRef(null);
  // 你可以读取或写入`ref.current`。
  // ...
}
```

然而，你需要自己管理它。

ref的作用与实例字段相同。它是进入可变命令世界的逃生舱。你可能熟悉“DOM refs”，但这个概念要广泛得多。它只是一个你可以放入某些东西的盒子。

即使从视觉上看，this.something看起来像something.current的镜像。它们表示相同的概念。

默认情况下，React不会在函数组件中为最新的props或state创建refs。在许多情况下你不需要它们，分配它们会浪费工作。然而，如果你愿意，你可以手动跟踪该值：

```javascript
function MessageThread() {
  const [message, setMessage] = useState('');
  const latestMessage = useRef('');

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };

  const handleSendClick = () => {
    setTimeout(showMessage, 3000);
  };

  const handleMessageChange = (e) => {
    setMessage(e.target.value);
    latestMessage.current = e.target.value;
  };
}
```

如果我们在showMessage中读取message，我们将看到我们按下发送按钮时的message。但是当我们读取latestMessage.current时，我们会得到最新的值——即使我们在按下发送按钮后继续输入。

你可以比较这两个演示来亲自查看差异。ref是一种“选择退出”渲染一致性的方法，在某些情况下非常方便。

一般来说，你应该避免在渲染期间读取或设置refs，因为它们是可变的。我们希望保持渲染的可预测性。然而，如果我们想获取特定prop或state的最新值，手动更新ref可能很麻烦。我们可以通过使用effect来自动化它：

```javascript
function MessageThread() {
  const [message, setMessage] = useState('');

  // 跟踪最新值。
  const latestMessage = useRef('');
  useEffect(() => {
    latestMessage.current = message;
  });

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };
}
```

我们在effect内部进行赋值，因此ref值仅在DOM更新后更改。这确保我们的变更不会破坏依赖可中断渲染的功能，如时间切片和Suspense。

使用这种方式的ref并不是非常必要。捕获props或state通常是更好的默认选项。然而，当处理命令式API如间隔和订阅时，它可能很方便。记住，你可以像这样跟踪任何值——一个prop、一个状态变量、整个props对象，甚至一个函数。

这种模式对于优化也很有用——例如，当useCallback标识更改过于频繁时。然而，使用reducer通常是更好的解决方案。

在这篇文章中，我们探讨了类组件中常见的破损模式，以及闭包如何帮助我们解决它。然而，你可能已经注意到，当你尝试通过指定依赖项数组来优化Hooks时，你可能会遇到陈旧的闭包问题。这是否意味着闭包是问题所在？我不这么认为。

正如我们上面看到的，闭包实际上帮助我们解决了难以注意到的细微问题。同样，它们使得在并发模式下编写正确工作的代码变得更容易。这是可能的，因为组件内部的逻辑关闭了其渲染时的正确props和state。

到目前为止，我见过的所有情况下，“陈旧闭包”问题都是由于错误地假设“函数不会改变”或“props始终相同”而引起的。我希望这篇文章能够澄清这个问题。

函数闭包它们的props和state——因此它们的标识同样重要。这不是一个bug，而是函数组件的一个特性。例如，函数不应被排除在useEffect或useCallback的“依赖项数组”之外。（正确的解决方法通常是使用useReducer或上面的useRef解决方案——我们将很快记录如何在它们之间进行选择。）

当我们用函数编写大多数React代码时，我们需要调整我们关于优化代码和哪些值会随时间变化的直觉。

正如Fredrik所说：

我目前发现的关于hooks的最佳心理规则是“编写代码时假设任何值都可以随时改变”。

函数也不例外。这需要一些时间才能成为React学习材料中的常识。这需要从类组件的心态进行一些调整。但我希望这篇文章能帮助你以新的眼光看待它。

React函数总是捕获它们的值——现在我们知道为什么了。