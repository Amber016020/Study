# **React的狀態管理(State Management)**

在React 的狀態流為單一方向, 也就是常聽到的 `Single Source of Truth(SSOT)`

```jsx
function Counter() {
  // State: a counter value
  const [counter, setCounter] = useState(0)

  // Action: code that causes an update to the state when something happens
  const increment = () => {
    setCounter(prevCounter => prevCounter + 1)
  }
  // View: the UI definition
  return (
    <div>
      Value: {counter} <button onClick={increment}>Increment</button>
    </div>
  )
}
```

React 的資料流主要有三個角色: `State`, `View`, `Action`

**1. State：**當前元件的資料來源.

**2. View：**呈現當前元件的狀態的介面

**3. Action：**畫面上觸發的任何事件, 如: 使用者輸入資料, 背景程式發送的事件而變更畫面上的狀態

大致上的流程為

1. 元件初始化, 宣告元件初始狀態(state)
2. 元件的視圖(view)讀取狀態, 並呈現在頁面上
3. 當使用者有任何操作行為(action), 則會發出事件
4. 元件收到事件後, 針對事件執行相關邏輯, 並修改狀態
5. 元件的視圖(view) 讀取新狀態, 並呈現在頁面上(重複步驟2)

![0_FnM3rJ4dbFa5ye6k.webp](attachment:5b990ac8-553f-4b7e-8b04-511ffa478117:0_FnM3rJ4dbFa5ye6k.webp)

當這個狀態只有當前元件才會使用到的情況下, 這樣的資料流是最單純的, 如同上方的範例一樣, 很方便管理程式碼, 程式碼也很容易追蹤.

不過當網頁的規模越來越大, 一定會遇到`狀態共享`的狀況.

如果只是在規模比較小的區域共用相同狀態, 或許可以透過 [“lifting state up”](https://react.dev/learn/sharing-state-between-components)的方式來解決

<aside>
💡

ps: 「**lifting state up（提升狀態）**」是 React 裡一個很重要的觀念，用來解決**多個元件需要共享同一份 state** 的問題。👉 **把 state 從子元件「搬到」共同的父元件，讓大家共用**

</aside>

但也會遇到在網頁內不同區塊, 不同頁面也需要共用一樣的狀態. 那即使是用lifting state up, 邏輯也很容易分散, 不容易管理

這時候就會需要一個`獨立` , `集中式` 的狀態管理中心, 讓不管是在哪個頁面的元件, 都可以取得共用的狀態, 或是發送事件來更新狀態

# **JavaScript的資料不變性(Immutability)**

Immutability, 翻譯是 `永不改變` , 用在JavaScript 內 即是希望`資料是可以永恆不變`

通常會是在JavaScript 的 `Object`/`Array` 套用這個概念

因為JavaScript中, Object 類型的資料是`call-by-address`的方式傳遞值

簡單的例子

```jsx
let a = { name: 'Hello' }
let b = a // b: { name: 'Hello' }

b.name = 'World' // a, b 都是 { name: 'World' }
```

該例子就`不是`一個Immutability 的例子

因為例子中, 只有對b 進行操作, 但連帶的a 的值也改變

要改成Immutability的例子, 就需要`複製` 原本的物件

```jsx
let a = { name: 'Hello' }
let b = { ...a } // shallow copy

b.name = 'World' // a: { name: 'Hello' }, b: { name: 'World' }
```

<aside>
💡

「**shallow copy（淺拷貝）**」是指：

👉 **只複製最外層資料，但內部的參考（reference）還是共用同一份，但如果是巢狀架構就會被改到，所以 copy 的時候要注意**

</aside>

Redux 也為了避免JavaScript 物件的call-by-address 所帶來的side effect. 所以在Redux內所有的狀態都必須用`Immutability` 方式更新

# **Redux專有名詞(Terminology)**

在Redux 內常見的專有名詞

- Actions
- Action Creators
- Reducers
- Store
- Dispatch
- Selectors

### **Action**

- 通常是一個物件, 包含`type`屬性. 通常type 指的是`事件發生的名稱`
- 常見的事件命名方式: `type: “事件類別(或功能)/行為”` 如: `type: "todos/todoAdded”`
- 當事件需要傳遞其他資料時, 可以在物件內多一個屬性: `payload` , 如以下範例

**payload = 你要傳送的「資料內容」**

```json
{
		type:'ADD_TODO',
		payload: {
			text:'Learn Redux'
	  }
}
```

👉 這裡：

- `type`：做什麼事（動作名稱）
- `payload`：**這個動作帶的資料**

### **Action Creators**

- 一個會回傳action物件的 function.
- 通常在Redux 內不會直接在reducer 內寫action, 而是是用action creator, 避免重複不斷的action

範例

```jsx
const addTodo = text => {
  return {
    type: 'todos/todoAdded',
    payload: text
  }
}
```

- 宣告一個函式 `addTodo`
- 接收一個參數 `text`（待辦事項內容）
- 回傳一個「物件」（這就是 action）
- `type`：描述這個動作要做什麼
- `payload`：實際資料

### **Reducer**

- 一個function, 會接收`目前狀態`與`一個action 物件`
- reducer function 會根據接收到的`action 類型`與`目前的狀態`來計算`新的狀態`並回傳
- 如果reducer function 發現收到的action類型找不到配對的邏輯可以計算, 則會回傳目前的狀態

> *💡 Reducer命名來源: 主要是reducer function 的行為與`Array.reduce()` 的callback 行為類似, 才會將它命名為reducer*
> 

Reducer function 會遵循幾個規則:

- 只會根據`action`與`現在的狀態`計算出新的狀態並回傳
- `禁止`直接更新目前狀態的資料, 應該要複製一份目前的狀態, 並只能更動這個複製出來的狀態
- 不允許在reducer function內放入任何處理`非同步事件` 的邏輯或是做出任何會出現`side effect`的程式
    - 同步 = 一件事做完才能做下一件事
    - 非同步 = 可以先去做其他事，結果晚一點才回來
    - **side effect（副作用） = 除了回傳結果之外，還會影響外部世界的程式行為**

[官方範例](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#reducers)

```jsx
const initialState = { value: 0 }

function counterReducer(state = initialState, action) {
  // Check to see if the reducer cares about this action
  if (action.type === 'counter/increment') {
    // If so, make a copy of `state`
    return {
      ...state,
      // and update the copy with the new value
      value: state.value + 1
    }
  }
  //Otherwise return the existing state unchanged
  return state
}
```

### **Store**

- 一個龐大的物件, 儲存了整個應用程式共用的狀態
- 建立的方式需要傳遞一個root reducer function

範例

```jsx
import { createStore } from 'redux'

let store = createStore(counterReducer);
console.log(store.getState()) // {value: 0}
```

### **Dispatch**

- Redux 唯一一個可以傳遞事件的`方法`(method), 如: `store.dispatch({ type: 'counter/increment' })`
- 當store 收到一個dispatch 時, 讓reducer 去執行狀態更新(若配對不到action類型, 則回傳store內的當前狀態)

[官方範例](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#dispatch)

```jsx
store.dispatch({ type: 'counter/increment' })

console.log(store.getState()) // {value: 1}
```

- 也可以搭配`action creator` 使用dispatch

[官方範例](https://redux.js.org/tutorials/essentials/part-1-overview-concepts#dispatch)

```jsx
const increment = () => {
  return {
    type: 'counter/increment'
  }
}

store.dispatch(increment())
console.log(store.getState() // {value: 2}
```

### **Selectors**

- 一個function, 用來取得store內, 某部分指定的狀態.
- 建立selector function 的好處就是當應用程式的規模越來越大, 需要取得相同資料的機會也會變高, selector可以`避免建立重複的邏輯`.

# **Redux 資料流**

Redux 的資料流與上方的元件資料流有點類似, 但不同的是, 元件內的狀態改放到一個獨立的狀態管理中心. 以至於`狀態`, `事件發送的方式` 就不一樣

官網有提供一張清楚的的流程圖

![1_yYkitaR24SuFNXYyTxL1xA.gif](attachment:486a12c0-de91-4173-81d0-49926d8f2276:1_yYkitaR24SuFNXYyTxL1xA.gif)

主要的流程如下:

1. 初始化
2. 在應用程式內建立一個狀態管理中心(store), 並初始化root reducer
3. 初始化root reducer的同時, 也會初始化內部的狀態(state)
4. 當渲染畫面時, 會向狀態管理中心取得狀態, 並顯示在畫面上, 同時也向狀態管理中心`訂閱`該狀態, 當資料更新時, 可以同步更新畫面上的資料
5. 資料更新
6. 當使用者在畫面觸發事件(如: 在畫面上點擊按鈕)
7. 應用程式會將使用者觸發的行為(action)發送給狀態管理中心 (如: `dispatch({ type: ‘counter/increment’ })`)
8. 狀態管理中心收到事件之後會開始執行`reducer`, 這個reducer 會包含前一個狀態(state), 以及收到的行為(action)
9. reducer 更新狀態並回傳新的狀態(會符合immutability)
10. 狀態管理中心會通知所有有訂閱狀態的元件, 告知他們目前狀態管理中心的狀態有更新
11. 元件到狀態管理中心檢查自己所需的狀態是否有更新.
12. 當元件發現自己所需的元件有更新時, 會強制重新渲染畫面, 並顯示最新的資料

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Redux Counter Demo</title>
  <style>
    body { font-family: sans-serif; padding: 20px; }
    button { margin: 5px; padding: 10px 20px; }
    #counter { font-size: 2em; margin-top: 10px; }
  </style>
</head>
<body>

<h1>Redux Counter Demo</h1>
<div>
  <button id="increment">Increment</button>
  <button id="decrement">Decrement</button>
</div>
<div id="counter">0</div>

<!-- 正確 UMD 版本 Redux -->
<script src="https://unpkg.com/redux@4.2.1/dist/redux.min.js"></script>
<script>
  // 1️⃣ 初始狀態
  const initialState = { value: 0 };

  // 2️⃣ 定義 reducer
  function counterReducer(state = initialState, action) {
    // 8️⃣ 根據 action 的 type 來更新 state
    switch(action.type) {
      case 'counter/increment':
        // 9️⃣ 回傳新的 state（不可變性）
        return { ...state, value: state.value + 1 };
      case 'counter/decrement':
        return { ...state, value: state.value - 1 };
      default:
        return state;
    }
  }

  // 3️⃣ 建立 store (直接用 Redux.createStore)
  // 🔟 狀態管理中心會通知所有有訂閱狀態的元件, 告知他們目前狀態管理中心的狀態有更新
  // 1️⃣1️⃣ 元件到狀態管理中心檢查自己所需的狀態是否有更新.
  // 1️⃣2️⃣ 當元件發現自己所需的元件有更新時, 會強制重新渲染畫面, 並顯示最新的資料
  const store = Redux.createStore(counterReducer);

  // 4️⃣ 訂閱 state 更新
  // store.subscribe 是 Redux 提供的方法，用來 訂閱（監聽）store 的狀態變化。
  store.subscribe(() => {
    // 更新 UI
    document.getElementById('counter').innerText = store.getState().value;
  });

  // 5️⃣ 綁定按鈕事件             6️⃣ 在畫面上點擊按鈕
  document.getElementById('increment').addEventListener('click', () => {
    // 7️⃣ 發送 action
    store.dispatch({ type: 'counter/increment' });
  });

  document.getElementById('decrement').addEventListener('click', () => {
    store.dispatch({ type: 'counter/decrement' });
  });

</script>
</body>
</html>
```

參考網頁：

[https://ceall8650.medium.com/redux-redux-學習筆記-基本概念-6bd58c0eff5c](https://ceall8650.medium.com/redux-redux-%E5%AD%B8%E7%BF%92%E7%AD%86%E8%A8%98-%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5-6bd58c0eff5c)

# Dispatch 的簡化

讓能夠讓我們不需要再Dispatch中打入這麼多東西，可以另外建立一個js檔案，把 dispatch 動作封裝成獨立的 JS 函式，這樣在主程式就不需要每次都手動寫 `{ type: '...' }`。

```jsx
// Action Creator：回傳 action 物件
function increment() {
  return { type: 'counter/increment' };
}

function decrement() {
  return { type: 'counter/decrement' };
}

// 如果你想要讓別人 import，可以加 export
export { increment, decrement };
```

從原本的

```jsx
dispatch( {type: "INCREASE",變數命名:變數} );
```

To

```jsx
dispatch( increase(變數) );
```

是不是變得很簡單了呢?

# Store 多個Dispatch進行匯集

一個專案中可能會有多個Dispatch所以我們會需要一個東西來進行匯集，而這個東西就是Store，創建Store的指令如下

`const store = configureStore({ reducer: allReducers });`

但如果我們有很多的Reducer呢?譬如我有一個加減計算的Reducer和一個代辦清單的Reducer的話要怎麼匯入呢?

這個時候我們就需要用到combineReducers
Reducers匯入進一個檔案後呢，並使用combineReducers這個函式把他們整合成一個Reducer。

假設有兩個 slice reducer：

```jsx
constcounterReducer= (state=0,action) => {
switch(action.type) {
case'increment':returnstate+1;
default:returnstate;
  }
};

consttodosReducer= (state= [],action) => {
switch(action.type) {
case'addTodo':return [...state,action.payload];
default:returnstate;
  }
};

// 傳給 configureStore
conststore=ReduxToolkit.configureStore({
  reducer: { counter:counterReducer, todos:todosReducer }
});
```

- 這樣 `store.getState()` 會是：

```jsx
{
	counter:0,
	todos: []
}
```

---

💡 **重點**：

- `configureStore` = **RTK 簡化版 createStore**
- 推薦新專案都用 RTK，因為更簡潔、安全、支援 DevTools 與 middleware
- 傳統 `createStore` 還是可以用，但需要自己手動設定很多東西

## 如何**取出State**

那我們如何在元件中取得值呢?

我們到專案給予我們的App.js中。

將能夠取得State的函式匯入:

```jsx
import { useSelector } from "react-redux";
```

使用函式將變數取出:

```jsx
const counter = useSelector(state=>state.counter);
```

最後將畫面渲染:

```jsx
<div className="App">
    Counter from Store is :{counter};
</div>
```

## 更改State

我們將能夠更改State會使用到的函式匯入，並將之命名，以便我們能夠在onClick事件中使用:

```jsx
import { useDispatch } from "react-redux";
const dispatch = useDispatch();
```

並且在匯入我們存放在actions資料夾下的index.js中的幾個函式來運用。

```jsx
import { increment, decrement, increase} from "./actions";
```

```jsx
export const increment = () =>{
    return{
        type: 'INCREMENT',
    };
}

export const decrement = () =>{
    return{
        type: 'DECREMENT',
    };
}

export const increase = (value) =>{
    return{
        type: 'INCREASE',
        value:value,
    };
}
```

並且新增兩個Button讓我們能夠操作。

並在其中新增onClick事件能夠使用Dispatch來使用相對應的action，撰寫如下:

```jsx
  return (
    <div className="App">
      Counter from Store is :{counter};
      <button onClick={()=>dispatch(increment())}>增加</button>
      <button onClick={()=>dispatch(decrement())}>減少</button>
    </div>
  );
```

之後在網頁中就能夠透過點擊來增加或減少State中的Counter

參考網頁：https://hackmd.io/@Agry/BJWmCg6vq

參考網頁：https://ithelp.ithome.com.tw/m/articles/10287139