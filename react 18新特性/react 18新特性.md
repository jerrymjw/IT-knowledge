# react 18新特性

## 1. userId

- `useId`是一个新的钩子，用于在客户端和服务器上生成唯一 ID，同时避免水合不匹配。它主要用于与需要唯一 ID 的可访问性 API 集成的组件库。这解决了 React 17 及更低版本中已经存在的问题，但在 React 18 中更为重要，因为新的流式服务器渲染器如何无序交付 HTML。

## 2. 批处理(batching)
 
### Hooks:

**React 17(使用ReactDOM.render):**

注：不会自动批处理

```js
import { useState, useLayoutEffect } from "react";
import * as ReactDOM from "react-dom";

function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    console.log("=== click ===");
    fetchSomething().then(() => {
      // React 17 and earlier does NOT batch these:
      setCount((c) => c + 1); // Causes a re-render
      setFlag((f) => !f); // Causes a re-render
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
      <LogEvents />
    </div>
  );
}

function LogEvents(props) {
  useLayoutEffect(() => {
    console.log("Commit");
  });
  console.log("Render");
  return null;
}

function fetchSomething() {
  return new Promise((resolve) => setTimeout(resolve, 100));
}

const rootElement = document.getElementById("root");
ReactDOM.render(<App />, rootElement);
```

**React 18(使用ReactDOM.creatRoot):**

注：自动批处理

```js
import { useState, useLayoutEffect } from "react";
import * as ReactDOM from "react-dom";

function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    console.log("=== click ===");
    fetchSomething().then(() => {
      // React 18 with createRoot batches these:
      setCount((c) => c + 1); // Does not re-render yet
      setFlag((f) => !f); // Does not re-render yet
      // React will only re-render once at the end (that's batching!)
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
      <LogEvents />
    </div>
  );
}

function LogEvents(props) {
  useLayoutEffect(() => {
    console.log("Commit");
  });
  console.log("Render");
  return null;
}

function fetchSomething() {
  return new Promise((resolve) => setTimeout(resolve, 100));
}

const rootElement = document.getElementById("root");
// This opts into the new behavior!
ReactDOM.createRoot(rootElement).render(<App />);
```

React18中自动批处理，如果不想自动批处理，那就需要使用flushSync函数

```js
import { flushSync } from 'react-dom'; // Note: react-dom, not react

function handleClick() {
  flushSync(() => {
    setCounter(c => c + 1);
  });
  // React has updated the DOM by now
  flushSync(() => {
    setFlag(f => !f);
  });
  // React has updated the DOM by now
}
```

### Class:

**React 17**:

类组件有一个实现怪癖，可以在事件内部同步读取状态更新。这意味着您将能够在调用`this.state`之间读取`setState`：

```js
handleClick = () => {
  setTimeout(() => {
    this.setState(({ count }) => ({ count: count + 1 }));

    // { count: 1, flag: false }
    console.log(this.state);

    this.setState(({ flag }) => ({ flag: !flag }));
  });
};
```

**React 18**:

注：由于所有更新`setTimeout`都是批处理的，所以 React 不会`setState`同步渲染第一次的结果——渲染发生在下一个浏览器滴答声中。所以渲染还没有发生：

```js
handleClick = () => {
  setTimeout(() => {
    this.setState(({ count }) => ({ count: count + 1 }));

    // { count: 0, flag: false }
    console.log(this.state);

    this.setState(({ flag }) => ({ flag: !flag }));
  });
};
```

ç. Start Transition

来自于`useTransition`hook的`startTransition`允许您将应用程序中的某些更新标记为**非紧急更新**，因此它们会暂停，同时优先考虑更紧急的更新。这让你的应用感觉更快，并且可以减少在你的应用中呈现非绝对必要的项目的负担。因此，无论您在渲染什么，您的应用程序仍在响应用户的输入。

```js
import {useState, useEffect, startTransition} from 'react';

startTransition(() => {

    setSearchResult(list);

});
```

## 3. React DOM 客户端

这些新 API 现在从以下位置导出`react-dom/client`：

- `createRoot` `render`: 为or创建根的新方法`unmount`。使用它代替`ReactDOM.render`. 没有它，React 18 中的新功能就无法工作。
- `hydrateRoot`: 水合服务器渲染应用程序的新方法。使用它而不是`ReactDOM.hydrate`与新的 React DOM 服务器 API 结合使用。没有它，React 18 中的新功能就无法工作。

两者都`createRoot`接受`hydrateRoot`一个新选项`onRecoverableError`，以防你想在 React 从渲染期间的错误中恢复或日志记录时收到通知。默认情况下，React 将使用[`reportError`](https://developer.mozilla.org/en-US/docs/Web/API/reportError), 或`console.error`在较旧的浏览器中。

## 4.  React DOM 服务器

这些新的 API 现在从`react-dom/server`服务器上导出并完全支持流式传输 Suspense：

- `renderToPipeableStream`: 用于 Node 环境中的流式传输。
- `renderToReadableStream`：适用于现代边缘运行时环境，例如 Deno 和 Cloudflare worker。

现有`renderToString`方法继续有效，但不鼓励使用。

## 5. ß弃用

- `react-dom`:`ReactDOM.render`已弃用。使用它会警告并在 React 17 模式下运行您的应用程序。
- `react-dom`:`ReactDOM.hydrate`已弃用。使用它会警告并在 React 17 模式下运行您的应用程序。
- `react-dom`:`ReactDOM.unmountComponentAtNode`已弃用。
- `react-dom`:`ReactDOM.renderSubtreeIntoContainer`已弃用。
- `react-dom/server`:`ReactDOMServer.renderToNodeStream`已弃用。