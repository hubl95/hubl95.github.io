---
title: JavaScript中的Event Loop（事件循环）机制
date: "2024-06-20"
description: "详细讲解了事件循环"
---
#### 1. JavaScript是单线程，非阻塞的
单线程：
JavaScript的主要用途是与用户互动，以及操作DOM。如果它是多线程的会有很多复杂的问题要处理，比如有两个线程同时操作DOM，一个线程删除了当前的DOM节点，一个线程是要操作当前的DOM阶段，最后以哪个线程的操作为准？为了避免这种，所以JS是单线程的。即使H5提出了web worker标准，它有很多限制，受主线程控制，是主线程的子线程。

非阻塞：通过 event loop 实现。

#### 2. 浏览器的事件循环
##### 执行栈和事件队列
![EventLoop](./eventloop.png)

执行栈: 同步代码的执行，按照顺序添加到执行栈中
```js
function a() {
    b();
    console.log('a');
}
function b() {
    console.log('b')
}
a();
```
1. 执行函数 a()先入栈
1. a()中先执行函数 b() 函数b() 入栈
1. 执行函数b(), console.log('b') 入栈
1. 输出 b， console.log('b')出栈
1. 函数b() 执行完成，出栈
1. console.log('a') 入栈，执行，输出 a, 出栈
1. 函数a 执行完成，出栈。

**事件队列**: 异步代码的执行，遇到异步事件不会等待它返回结果，而是将这个事件挂起，继续执行执行栈中的其他任务。当异步事件返回结果，将它放到事件队列中，被放入事件队列不会立刻执行起回调，而是等待当前执行栈中所有任务都执行完毕，主线程空闲状态，主线程会去查找事件队列中是否有任务，如果有，则取出排在第一位的事件，并把这个事件对应的回调放到执行栈中，然后执行其中的同步代码。

我们再上面代码的基础上添加异步事件：
```js
function a() {
    b();
    console.log('a');
}
function b() {
    console.log('b')
    setTimeout(function() {
        console.log('c');
    }, 2000)
}
a();
```
![EventLoop2](./eventloop2.png)

##### 宏任务和微任务
为什么要引入微任务，只有一种类型的任务不行么？
页面渲染事件，各种IO的完成事件等随时被添加到任务队列中，一直会保持先进先出的原则执行，我们不能准确地控制这些事件被添加到任务队列中的位置。但是这个时候突然有高优先级的任务需要尽快执行，那么一种类型的任务就不合适了，所以引入了微任务队列。
不同的异步任务被分为：宏任务和微任务

宏任务：
- script(整体代码)
- setTimeout()
- setInterval()
- postMessage
- I/O
- UI交互事件

微任务:
- new Promise().then(回调)
- MutationObserver(html5 新特性)

##### 运行机制
异步任务的返回结果会被放到一个任务队列中，根据异步事件的类型，这个事件实际上会被放到对应的宏任务和微任务队列中去。
在当前执行栈为空时，主线程会查看微任务队列是否有事件存在
- 存在，依次执行队列中的事件对应的回调，直到微任务队列为空，然后去宏任务队列中取出最前面的事件，把当前的回调加到当前指向栈。
- 如果不存在，那么再去宏任务队列中取出一个事件并把对应的回调加入当前执行栈；

当前执行栈执行完毕后时会立刻处理所有微任务队列中的事件，然后再去宏任务队列中取出一个事件。同一次事件循环中，微任务永远在宏任务之前执行。

在事件循环中，每进行一次循环操作称为 tick，每一次 tick 的任务处理模型是比较复杂的，但关键步骤如下：
- 执行一个宏任务（栈中没有就从事件队列中获取）
- 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中
- 宏任务执行完毕后，立即执行当前微任务队列中的所有微任务（依次执行）
- 当前宏任务执行完毕，开始检查渲染，然后GUI线程接管渲染
- 渲染完毕后，JS线程继续接管，开始下一个宏任务（从事件队列中获取）

简单总结一下执行的顺序：
执行宏任务，然后执行该宏任务产生的微任务，若微任务在执行过程中产生了新的微任务，则继续执行微任务，微任务执行完毕后，再回到宏任务中进行下一轮循环。
![EventLoop3](./eventloop3.png)
```js
console.log('start')

setTimeout(function() {
  console.log('setTimeout')
}, 0)

Promise.resolve().then(function() {
  console.log('promise1')
}).then(function() {
  console.log('promise2')
})

console.log('end')
```

![EventLoop](./eventloop.gif)

1. 全局代码压入执行栈执行，输出 start
1. setTimeout压入 macrotask队列，promise.then 回调放入 microtask队列，最后执行 console.log('end')，输出 end
1. 调用栈中的代码执行完成（全局代码属于宏任务），接下来开始执行微任务队列中的代码，执行promise回调，输出 promise1, promise回调函数默认返回 undefined, promise状态变成 fulfilled ，触发接下来的 then回调，继续压入 microtask队列，此时产生了新的微任务，会接着把当前的微任务队列执行完，此时执行第二个 promise.then回调，输出 promise2
1. 此时，microtask队列 已清空，接下来会会执行 UI渲染工作（如果有的话），然后开始下一轮 event loop, 执行 setTimeout的回调，输出 setTimeout

最后的执行结果如下
- start
- end
- promise1
- promise2
- setTimeout

#### 3.node环境下的事件循环
##### 和浏览器环境有何不同
表现出的状态与浏览器大致相同。不同的是 node 中有一套自己的模型。node 中事件循环的实现依赖 libuv 引擎。Node的事件循环存在几个阶段。

如果是node10及其之前版本，microtask会在事件循环的各个阶段之间执行，也就是一个阶段执行完毕，就会去执行 microtask队列中的任务。

node版本更新到11之后，Event Loop运行原理发生了变化，一旦执行一个阶段里的一个宏任务(setTimeout,setInterval和setImmediate)就立刻执行微任务队列，跟浏览器趋于一致。下面例子中的代码是按照最新的去进行分析的。

#### 4.案例分析
##### 一. 下面代码输出什么
```js
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}
async function async2() {
  console.log('async2');
}
console.log('script start');
setTimeout(function() {
  console.log('setTimeout');
}, 0)
async1();
new Promise(function(resolve) {
  console.log('promise1');
  resolve();
}).then(function() {
  console.log('promise2');
});
console.log('script end');
```
先执行宏任务（当前代码块也算是宏任务），然后执行当前宏任务产生的微任务，然后接着执行宏任务
1. 从上往下执行代码，先执行同步代码，输出 script start
2. 遇到setTimeout，现把 setTimeout 的代码放到宏任务队列中
3. 执行 async1()，输出 async1 start, 然后执行 async2(), 输出 async2，把 async2() 后面的代码 console.log('async1 end')放到微任务队列中
4. 接着往下执行，输出 promise1，把 .then()放到微任务队列中；注意Promise本身是同步的立即执行函数，.then是异步执行函数
5. 接着往下执行， 输出 script end。同步代码（同时也是宏任务）执行完成，接下来开始执行刚才放到微任务中的代码
6. 依次执行微任务中的代码，依次输出 async1 end、 promise2, 微任务中的代码执行完成后，开始执行宏任务中的代码，输出 setTimeout

最后的执行结果如下
- script start
- async1 start
- async2
- promise1
- script end
- async1 end
- promise2
- setTimeout

##### 二. 下面代码输出什么
```js
console.log('start');
setTimeout(() => {
  console.log('children2');
  Promise.resolve().then(() => {
    console.log('children3');
  })
}, 0);

new Promise(function(resolve, reject) {
  console.log('children4');
  setTimeout(function() {
    console.log('children5');
    resolve('children6')
  }, 0)
}).then((res) => {
  console.log('children7');
  setTimeout(() => {
    console.log(res);
  }, 0)
})
```
这道题跟上面题目不同之处在于，执行代码会产生很多个宏任务，每个宏任务中又会产生微任务
1. 从上往下执行代码，先执行同步代码，输出 start
2. 遇到setTimeout，先把 setTimeout 的代码放到宏任务队列①中
3. 接着往下执行，输出 children4, 遇到setTimeout，先把 setTimeout 的代码放到宏任务队列②中，此时.then并不会被放到微任务队列中，因为 resolve是放到 setTimeout中执行的
4. 代码执行完成之后，会查找微任务队列中的事件，发现并没有，于是开始执行宏任务①，即第一个 setTimeout， 输出 children2，此时，会把 Promise.resolve().then放到微任务队列中。
5. 宏任务①中的代码执行完成后，会查找微任务队列，于是输出 children3；然后开始执行宏任务②，即第二个 setTimeout，输出 children5，此时将.then放到微任务队列中。
6. 宏任务②中的代码执行完成后，会查找微任务队列，于是输出 children7，遇到 setTimeout，放到宏任务队列中。此时微任务执行完成，开始执行宏任务，输出 children6;

最后的执行结果如下
- start
- children4
- children2
- children3
- children5
- children7
- children6

##### 三. 下面代码输出什么
```js
const p = function() {
  return new Promise((resolve, reject) => {
    const p1 = new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve(1)
      }, 0)
      resolve(2)
    })
    p1.then((res) => {
      console.log(res);
    })
    console.log(3);
    resolve(4);
  })
}


p().then((res) => {
  console.log(res);
})
console.log('end');
```
1. 执行代码，Promise本身是同步的立即执行函数，.then是异步执行函数。遇到setTimeout，先把其放入宏任务队列中，遇到p1.then会先放到微任务队列中，接着往下执行，输出 3
2. 遇到 p().then 会先放到微任务队列中，接着往下执行，输出 end
3. 同步代码块执行完成后，开始执行微任务队列中的任务，首先执行 p1.then，输出 2, 接着执行p().then, 输出 4
4. 微任务执行完成后，开始执行宏任务，setTimeout, resolve(1)，但是此时 p1.then已经执行完成，此时 1不会输出。

最后的执行结果如下
- 3
- end
- 2
- 4

转载：https://segmentfault.com/a/1190000022805523
MDN：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/EventLoop#%E8%BF%90%E8%A1%8C%E6%97%B6%E6%A6%82%E5%BF%B5
loupe：http://latentflip.com/loupe