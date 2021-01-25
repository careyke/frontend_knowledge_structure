# 保存无副作用状态：useCallback、useMemo和useRef

在日常开发`React`组件的时候，我们常常会碰到一些特殊的值，有以下特点：

1. 需要长久保存
2. 值可能会随着state的变化而变化
3. 它的修改不需要触发组件更新

我把这类值称为**无副作用的状态**。

> 所谓无副作用，就是它的更改不会导致组件的更新

这类无副作用的状态并不适合保存在`state`中的，不仅会污染`state`，而且可能会导致组件多次更新。

举两个例子：

ClassComponent：

```react
class App extends React.Component {
  state = {
    didMount: false
  };

  componentDidMount() {
    this.setState({ didMount: true });
  }

  render() {
    return <span>1</span>;
  }
}
```

这里`didMount`放在`state`中会导致组件多更新一次



FunctionComponent：(legacy模式下)

```react
function Test() {
  const [count, setCount] = useState(0);
  const [bool, setBool] = useState(false);
  const buttonRef = useRef();

  useEffect(() => {
    buttonRef.current.addEventListener("click", () => {
      setCount(1);
      setBool(true);
    });
  }, []);

  return (
    <div>
      <button ref={buttonRef}>button</button>
    </div>
  );
}
```

在legacy模式下，点击按钮会渲染两次。

上面代码中的`didMount`和`bool`都会无副作用的状态，不应该保存在state中。

**在`ClassComponent`中，通常是直接将无副作用状态保存在组件实例`this`上**。但是`FunctionComponent`中并没有实例，为了保存无副作用的状态，React提供了三种`Hook`来保存。

1. useCallback
2. useMemo
3. useRef

