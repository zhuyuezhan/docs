https://overreacted.io/before-you-memo/
https://kentcdodds.com/blog/optimize-react-re-renders

### 使用Memo前首先考虑的优化方式（React如何处理props传递的组件？）

一种避免不依赖父组件数据的子组件重新渲染的方式：**通过props（不一定是children）传递子组件**。

from

```javascript
export default function App() {
  let [color, setColor] = useState('red');
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      <p>Hello, world!</p>
      <ExpensiveTree />
    </div>
  );
}
```
to
```javascript
export default function App() {
  return (
    <ColorPicker>
      <p>Hello, world!</p>
      <ExpensiveTree />
    </ColorPicker>
  );
}
 
function ColorPicker({ children }) {
  let [color, setColor] = useState("red");
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      {children}
    </div>
  );
}
```


当使用props传递组件时，它不会随父组件重新渲染（如果它本身是静态的）。是什么导致了这个现象？

>@kentcdodds: If you give React the same element you gave it on the last render, it wont bother re-rendering that element.

React 可以自动为我们提供此优化，并且不会费心重新渲染元素，因为它不需要重新渲染。这基本上就像 React.memo，只是 React 是检查 props 对象整体，而不是单独检查每个 props。