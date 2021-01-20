# React Hooks的设计理念

本节开始，笔者会开始分析`React Hooks`的工作原理和实现，在分析具体代码实现之前，我们有必要来了解一下`Hooks`的设计哲学。



## 1. Hooks的设计理念

首先我们来看看一下`React`的`logo`。

![reactLogo](./images/reactLogo.jpg)

`React`的`logo`是一张**原子结构图**，物理学中有介绍过，自然界中的一起物体都是由原子构成的。React借助这张图来表达了自己的核心思想——**所有的页面都可以由一个个组件来构成**。组件可以类比为自然界中的原子，组件之间相互独立。

原子在希腊语中是不可分割的意思，但是后来科学家发现原子中还存在更小的粒子——电子。让人没有想到的是，React的发展过程也十分巧合的经历了这么一个过程。

`Hooks`的提出让组件这个原子中拥有了更小的独立单位，我们可以**把组件（component）类比为原子，把`Hook`类比为电子**。这种结构刚好和`logo`完全吻合（不得不感叹，冥冥之中自有天意~）

回到`React`本身来说，**Hooks的出现，进一步细化了`状态复用`的基本单位**。

- Hooks出现之前，状态复用只能以组件为单位来复用
- Hooks出现之后，可以以`Hook`为单位来复用状态



## 2. 状态复用

上面我们讲到，Hooks的出现细化了状态复用的基本单位，那么Hooks是如何实现细化的呢？

在`ClassComponent`中，如果要复用一个组件的某些状态，需要使用**高阶组件或者`renderProps`**的方案来实现复用。这类方案需要将公共的状态封装成一个组件，然后去包裹原来的组件，会重新组织你的组件结构，慢慢组件结构会越来越复杂，代码也越来越难理解。

Hooks的出现可以**将组件复杂的状态或副作用解构成一个一个简单的状态或副作用，而且不需要额外封装组件，而是以钩子的形式挂载在`FunctionComponent`组件中，以插拔的形式来实现状态的复用**。



Hooks出现之前，`FunctionComponent`表示的是没有内部状态和副作用的简单组件，**`Hooks`的出现赋予了`FunctionComponent`内部状态和副作用**，也让React使用者有了更多的选择。

下面我们来比较一下`ClassComponent`和`FunctionComponent`



## 3. ClassComponent VS FunctionComponent

### 3.1 ClassComponent的劣势

1. **状态复用成本较大**：组件之间的状态复用只能使用高阶组件或者`renderProps`这类方案，会修改组件结构，长此以往会导致代码难以维护
2. **关注点分离**：复杂的组件中，常常需要在`componentDidMount`和`componentDidUpdate`生命周期中获取数据，在`componentDidMount`中设置事件监听然后在`componentWillUnmount`中取消监听。导致原本相关联的逻辑分散在各个生命周期函数中，容易产生`bug`
3. **class有理解成本**：使用`class`必须要去了解`javascript`中`class`的实现原理，还要弄清楚`this`的指向问题，有一定成本。



### 3.2 FunctionComponent的优势

在`FunctionComponent`中，`Hooks`的加入解决了状态复用成本大和关注点分离的问题，然后`function`本身也比`class`更容易理解和使用。

所以推荐大家使用`FunctionComponent`（笔者只从Hooks出来之后，基本就没有使用过`ClassComponent`）



## 4. Hooks的分类

在后面的文章中，比较将会结合源码来分析`Hooks`的实现。

这里首先来将`React`定义的常用的`Hooks`进行分类：

1. **StateHooks**：状态相关的Hooks，源码中叫做`Hooks`
   - useState
   - useReducer
   - useMemo
   - useCallback
   - useRef
2. **EffectHooks**：副作用相关的Hooks，源码中叫做`Effect`
   - useEffect
   - useLayoutEffect
3. **ContextHooks**：上下文相关的Hooks
   - useContext

