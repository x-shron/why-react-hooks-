# why-react-hooks-cannot-be-included-in-(if/else)
为什么react的hooks不能出现在条件语句里
## Hook 使用规则
官方文档 [Hook 概览](https://react.docschina.org/docs/hooks-overview.html)中对于hooks规则叙述：

> - 只能在函数最外层调用 Hook。不要在循环、条件判断或者子函数中调用。
> - 只能在 React 的函数组件中调用 Hook。不要在其他 JavaScript 函数中调用。（还有一个地方可以调用 Hook —— 就是自定义的 Hook 中)

我看了很多解释，但是一直都模模糊糊，一知半解的, 结合源码与视频学习部分。

通手动实现一个 useState的基本实现思路，从而明白了，为什么要顺序调用 Hook。

## 手动实现一个 useState

### 实现单个 Hook

首先捋一下，useState 有啥用？

1. 从里边能够导出一个数组，useState[0] 是一个变量，useState[1] 是一个功能函数。
2. 能够传一个参数，作为它的初始值。
3. setState 之后，会把新的值赋值给value，然后，会重新渲染整个函数组件。这时候，组件取到的 value, 就是全新的值。

那我们大概实现一下代码 , 假设，我们在 一个 `<App />` 的组件里使用 useState.

```javascript
function useState(initialValue){
    let value = initialvalue;
    function setState(nextvalue)=>{
        value = nextValue;
        ReactDOM.render(<App />, document.getElementById('root'))
    }
    
    return [value,setState]
}
//.....
const [value,setState] = useState('initialValue');
```
我们捋一下流程： 第一次，使用 useState ,拿到初始值，正常 render， 取到 value 值。

当我们使用 setState 的时候，此时，按照流程，会重新触发渲染整个组件.

而重新渲染组件的时候，会重新执行一遍 useState, 那这个时候会发生什么呢？ 返回的 value 值是初始值。
很明显，按照现有的逻辑，每次都会重新渲染 useState, 我们把 value 放在 useState 里面不对，我们需要每次 setState  value 的值为新的后，返回的 [value,setState] 里面的值是最新的。

所以，我们需要把 value 的值放到 useState 的外面，每次 setState 后，重新返回的 value 值，取最外层最新值，这样 render 读取到的数值就是正确的。

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './App.css';

let value;

function useState(initialState) {
  if (typeof value === 'undefined') value = initialState;
  function setState(nextValue) {
    value = nextValue;
    ReactDOM.render(<App />, document.getElementById('root'))
  }
  return [value, setState];
}

function App() {

  const [name, setName] = useState('initialName');
  const handleInputChange = (e) => {
    setName(e.target.value)
  }

  return (
    <div className="App">
      <div>
        <h1>My name is:{name} </h1>
        <input onChange={handleInputChange}></input>
      </div>
    </div>
  );
}

export default App;

```
### 实现多个 Hook

按照上文的思路， 每次 setState 之后，都会改变 value 的值。那如果用了 两个 useState，就无法存储两个 value 对应的值了，value 只会存储最新的 setState 改变之后的值。

那怎么办呢？ 我们得让不同的 useState 对应不同的 value 的值。

为什么不能将 value 设置成一个数组呢？

第一声明的 useState 的值，放到 value[0], 第二次声明的 useState 的值，放到 value[1]。 这样它们就互不干涉，能够在一个组件函数中多次使用 useState 了。
为了将 useState 的声明和 value 数组对应起来，我们需要引入一个新的变量：`currentHook`

现在我们思路如下：

默认 currentHook = 0.声明第一个 useState 的时候，对 value[0] 的值进行操作。然后 将 currentHook++, 当再次声明 useState 的时候，就对 value[1] 进行操作，以此类推。

要注意的是，setState 以后，会重初始化 useState. 而如果不对 currentHook 重新赋值，它会无限增加，这样就会导致返回的结果不对了。所以，每次 useState 开始之前，都要对 currenHook 重新赋值为 0.

这样才能保证上一次的 新初始化的 useState 和现有的 value 数组值能够一一对应。

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './App.css';

let values = [];
let currentHook = 0;

function useState(initialState) {
  if (typeof values[currentHook] === 'undefined') values[currentHook] = initialState;
  let hookIndex = currentHook;
  function setState(nextValue) {
    values[hookIndex] = nextValue;
    ReactDOM.render(<App />, document.getElementById('root'))
  }
  return [values[currentHook++], setState];
}

function App() {

  currentHook = 0;

  const [name, setName] = useState('');
  const [age, setAge] = useState();

  console.log('values', values);

  return (
    <div className="App">
      <div>
        <h1>My name is:{name} </h1>
        <input onChange={(e) => setName(e.target.value)}></input>
        <br />
        <h1>My age is:{age} </h1>
        <input onChange={(e) => setAge(e.target.value)}></input>
      </div>
    </div>
  );
}

export default App;

```

> 以上为个人基于源码和视频学习的理解，实现的的简易hooks原理，实际的 Hook 实现复杂的多，用起来也更方便。但是它的基础实现原理是和文中的代码思路一致的



## 为什么不能放在条件语句里？

从上文的实现过程中，我们可以发现，每次渲染值的时候，useState 的初始值其实是和 value 数组中一一对应的。

当 setState 以后，重新渲染整个组件时：第一个 useState 取 value[0] 的值，第二个取 value[1] 的值，以此类推。

如果我们将 useState 放在条件语句里，如果这个条件在 N 次执行了，声明了 useState 语句，而在 N + 1 次没有执行，那么 useState 顺序取值的时候，取到的值就是上一个值，以此类推，接下来声明的 useState 全部会出现取值错误。

因此为保障react程序的正确性，严禁将hooks放入条件语句中使用



### 如果以上对您有帮助，记得给我点个赞噢~它会给予我继续更文的动力
