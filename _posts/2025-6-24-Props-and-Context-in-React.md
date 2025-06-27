---
title: "Props and Context in React"
date: 2025-06-24
permalink: /posts/2025/Props and Context in React/
tags:
  - React
  - Props
  - Context
  - Frontend
---

React 组件间经常需要传递数据，本篇文章介绍如何使得数据的管理和传递更为高效。

- [1. Props](#1-props)
  - [什么是 Props](#什么是-props)
  - [如何传递 Props](#如何传递-props)
- [2. Context](#2-context)
  - [1. 引入库](#1-引入库)
  - [2. 基本步骤](#2-基本步骤)
- [3. Props 与 Context 的比较](#3-props-与-context-的比较)
- [4. 总结](#4-总结)



# 1. Props
## 什么是 Props
Props（属性）是 React 组件的主要数据传递机制。它们允许父组件向子组件传递数据，从而实现组件间的通信。Props 是只读的，子组件不能直接修改接收到的 Props，这种设计保证了组件的可预测性和重用性。
## 如何传递 Props
在没有使用 Context 的情况下，需要通过层层传递来实现数据的共享。例如，在一个简单的计数器应用中：
```jsx
const App = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <h1>计数器</h1>
      <CounterDisplay count={count} />
      <CounterButton setCount={setCount} />
    </div>
  );
};

const CounterDisplay = ({ count }) => {
  return <p>当前计数：{count}</p>;
};

const CounterButton = ({ setCount }) => {
  return (
    <button onClick={() => setCount((prev) => prev + 1)}>增加</button>
  );
};
```
尽管这种方式简单有效，但在组件树较深时会变得冗长且难以维护。而使用 React Context 可以解决这个问题，使得共享变量变得更加方便。

# 2. Context
## 1. 引入库

要使用 Context，我们需要从 React 中导入必要的库：
```js
import React, { createContext, useContext, useState } from 'react';
```
Context 设计是为了解决多组件间复杂状态同步的问题，可以通过 React 的上下文 API（Context API）来实现。这种方式允许你在组件树中共享数据，而不必通过每个组件的 props 进行传递。

## 2. 基本步骤
还是以计数器应用为例：
1. **创建上下文：** 使用 `createContext` 创建一个新的上下文来存储共享的变量。

    ```javascript  
	  const CounterContext = createContext();  
    ```

2. **在组件中提供上下文：** 在父组件中，用 `CounterContext.Provider` 包裹子组件，传入共享状态及更新函数。

    ```javascript  
    const App = () => {
      const [count, setCount] = useState(0);

      return (
        <CounterContext.Provider value={{ count, setCount }}>
          <CounterDisplay />
          <CounterButton />
        </CounterContext.Provider>
      );
    };
    ```

3. **在子组件中消费上下文：** 在子组件中，用 `useContext` Hook 访问上下文。

    ```js  
    const CounterDisplay = () => {
      const { count } = useContext(CounterContext);
      return <p>当前计数：{count}</p>;
    };

    const CounterButton = () => {
      const { setCount } = useContext(CounterContext);
      return (
        <button onClick={() => setCount((prev) => prev + 1)}>增加</button>
      );
    };
    ```

# 3. Props 与 Context 的比较

Props 适用于父组件与子组件之间的简单数据传递，尤其是当组件之间的关系非常明确时。而 Context 数据可以在整个组件树中进行共享，避免了因为层层传递 props 而导致的 “props drilling” 问题。Context 更适合在多层嵌套的组件中共享状态，例如主题、用户认证信息等。在一些大项目中需要跨越多层组件传递数据时，Context 提供了更简洁的解决方案。


# 4. 总结
通过合理选择这两种方式，我们可以有效地管理和传递数据，从而提高应用的维护性和可扩展性。希望本篇文章能帮助您更好地理解和应用 React 中的 Props 和 Context。如果您有任何问题，欢迎留言讨论。
