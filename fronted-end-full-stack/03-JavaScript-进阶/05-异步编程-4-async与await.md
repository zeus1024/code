# 05-异步编程 -4-async 与 await

## 一 ES7 的 async/await

### 1.0 用法示例

Promise 虽然给异步编程带来了便利，但是在大型项目上用起来仍然较为繁琐，ES6 的生成器过渡方案并不简洁。ES7 引入了 async/await，被大众接受，并在 ES8 正式发布。

async/await 语法：

```js
async function(){

    // await 是等待的意思，等待异步操作返回的结果，会阻塞代码
    let res1 = await 异步操作1(Promise/generator/async);

    // 这时候异步操作 2 需要等待 res1 的结果获取后才能执行
    let res2 = await 异步操作2(Promise/generator/async);
}
```

基本示例：

```js
async function show(params) {
  console.log('阶段一')

  await new Promise(function (resolve, rejec) {
    setTimeout(function () {
      resolve()
    }, 3000)
  })

  console.log('阶段二')

  await new Promise(function (resolve, rejec) {
    setTimeout(function () {
      resolve()
    }, 2000)
  })

  console.log('阶段三')
}

show() // 阶段一  阶段二  阶段三
```

上面的写法已经符合正常人的顺序思维了，不过要理解的是：async/await 不过是语法糖，并未真正将异步代码同步化！

### 1.1 async 关键字

async 关键字用于声明异步函数，可以用在函数声明、函数表达式、箭头函数：

```js
async function foo() {}

let bar = async function () {}

let baz = async () => {}

class Qux {
  async qux() {}
}
```

使用 async 声明的函数具有异步的特征，不过其求值仍然是同步求值的，不过如果函数内部使用 return 返回，则返回的值则会被 `Promise.resolve()` 包装成一个期约对象：

```js
// 第一种情况
async function foo() {
  return 3 // 与 return Promise.resolve(3) 效果一致
}

// 返回值由 then 函数解包
foo().then((data) => {
  console.log('后打印：', data) // 3
})

console.log('先打印') // 1

// 第二种情况：返回一个实现了 thenable 接口的非期约对象
async function baz() {
  const thenable = {
    then(callback) {
      callback('baz')
    },
  }
  return thenable
}

// 由 then() 解包
baz().then(console.log) // baz
```

### 1.2 await

使用 await 关键字可以暂停异步函数代码的执行，等待期约解决：

```js
// 纯 Promise 解决方案：
let p = new Promise((resolve, reject) => setTimeout(resolve, 1000, 3))
p.then((x) => console.log(x)) // 3

// 使用 async/await 可以写成这样：
async function foo() {
  let p = new Promise((resolve, reject) => setTimeout(resolve, 1000, 3))
  console.log(await p)
}
foo() // 3
```

await 关键字会暂停执行异步函数后面的代码，让出 JavaScript 运行时的执行线程。这个行为与生成器函数中的 yield 关键字是一样的。await 关键字同样是尝试“解包”对象的值，然后将这个值传给表达式，再异步恢复异步函数的执行。

await 的用法与医院操作一样，可以单独使用，也可以在表达式中使用：

```js
// 异步打印"foo"
async function foo() {
  console.log(await Promise.resolve('foo'))
}
foo() // foo

// 异步打印"bar"
async function bar() {
  return await Promise.resolve('bar')
}
bar().then(console.log) // bar

// 1000 毫秒后异步打印"baz"
async function baz() {
  await new Promise((resolve, reject) => setTimeout(resolve, 1000))
  console.log('baz')
}
baz() // baz（1000 毫秒后）
```

await 关键字期待（但实际上并不要求）一个实现 thenable 接口的对象，但常规的值也可以。如果是实现 thenable 接口的对象，则这个对象可以由 await 来“解包”。如果不是，则这个值就被当作已经解决的期约。

### 1.3 await 的限制

await 关键字必须在异步函数中使用，不能在顶级上下文如`<script></script>`标签或模块中使用：

```js
async function foo() {
  console.log(await Promise.resolve(3))
}
foo() // 3

// 立即调用的异步函数表达式
;(async function () {
  console.log(await Promise.resolve(3))
})() // 3
```

await 错误示例：

```js
// 不允许：await 出现在了箭头函数中
function foo() {
  const syncFn = () => {
    return await Promise.resolve('foo')
  }
  console.log(syncFn())
}

// 不允许：await 出现在了同步函数声明中
function bar() {
  function syncFn() {
    return await Promise.resolve('bar')
  }
  console.log(syncFn())
}

// 不允许：await 出现在了同步函数表达式中
function baz() {
  const syncFn = function () {
    return await Promise.resolve('baz')
  }
  console.log(syncFn())
}

// 不允许：IIFE 使用同步函数表达式或箭头函数
function qux() {
  ;(function () {
    console.log(await Promise.resolve('qux'))
  })()
  ;(() => console.log(await Promise.resolve('qux')))()
}
```

## 二 错误处理

拒绝期约的错误不会被异步函数捕获：

```js
async function foo() {
  console.log(1)
  Promise.reject(3)
}
// Attach a rejected handler to the returned promise
foo().catch(console.log)
console.log(2)
// 1
// 2
// Uncaught (in promise): 3
```

等待会抛出错误的同步操作，会返回拒绝的期约：

```js
async function foo() {
  console.log(1)
  await (() => {
    throw 3
  })()
}

// 给返回的期约添加一个拒绝处理程序，输出 1 2 3
foo().catch(console.log)
console.log(2)
```

对拒绝的期约使用 await 则会释放（unwrap）错误值（将拒绝期约返回）：

```js
async function foo() {
  console.log(1)
  await Promise.reject(3)
  console.log(4) // 这行代码不会执行
}
// 给返回的期约添加一个拒绝处理程序
foo().catch(console.log)
console.log(2)
```

## 三 停止和恢复执行

异步函数如果不包含 await 关键字，其执行基本上跟普通函数没有什么区别，不过加上 await 上之后：

```js
async function foo() {
  console.log(await Promise.resolve('foo'))
}
async function bar() {
  console.log(await 'bar')
}
async function baz() {
  console.log('baz')
}
foo()
bar()
baz()
// baz
// bar
// foo
```

JavaScript 运行时在碰到 await 关键字时，会记录在哪里暂停执行。等到 await 右边的值可用了，JavaScript 运行时会向消息队列中推送一个任务，这个任务会恢复异步函数的执行。

即使 await 后面跟着一个立即可用的值，函数的其余部分也会被异步求值：

```js
async function foo() {
  console.log(2)
  await null
  console.log(4)
}
console.log(1)
foo()
console.log(3)

// 1
// 2
// 3
// 4
```

运行过程：

```txt
(1) 打印 1；
(2) 调用异步函数 foo()；
(3)（在 foo() 中）打印 2；
(4)（在 foo() 中）await 关键字暂停执行，为立即可用的值 null 向消息队列中添加一个任务；
(5) foo() 退出；
(6) 打印 3；
(7) 同步线程的代码执行完毕；
(8) JavaScript 运行时从消息队列中取出任务，恢复异步函数执行；
(9)（在 foo() 中）恢复执行，await 取得 null 值（这里并没有使用）；
(10)（在 foo() 中）打印 4；
(11) foo() 返回。
```

如果 await 后面是一个期约，为了执行异步函数，实际上会有两个任务被添加到消息队列并被异步求值：

```js
async function foo() {
  console.log(2)
  console.log(await Promise.resolve(8))
  console.log(9)
}
async function bar() {
  console.log(4)
  console.log(await 6)
  console.log(7)
}
console.log(1)
foo()
console.log(3)
bar()
console.log(5)
// 1
// 2
// 3
// 4
// 5
// 6
// 7
// 8
// 9
```

执行步骤：

```txt
(1) 打印 1；
(2) 调用异步函数 foo()；
(3)（在 foo() 中）打印 2；
(4)（在 foo() 中）await 关键字暂停执行，向消息队列中添加一个期约在落定之后执行的任务；
(5) 期约立即落定，把给 await 提供值的任务添加到消息队列；
(6) foo() 退出；
(7) 打印 3；
(8) 调用异步函数 bar()；
(9)（在 bar() 中）打印 4；
(10)（在 bar() 中）await 关键字暂停执行，为立即可用的值 6 向消息队列中添加一个任务；
(11) bar() 退出；
(12) 打印 5；
(13) 顶级线程执行完毕；
(14) JavaScript 运行时从消息队列中取出解决 await 期约的处理程序，并将解决的值 8 提供给它；
(15) JavaScript 运行时向消息队列中添加一个恢复执行 foo() 函数的任务；
(16) JavaScript 运行时从消息队列中取出恢复执行 bar() 的任务及值 6；
(17)（在 bar() 中）恢复执行，await 取得值 6；
(18)（在 bar() 中）打印 6；
(19)（在 bar() 中）打印 7；
(20) bar() 返回；
(21) 异步任务完成，JavaScript 从消息队列中取出恢复执行 foo() 的任务及值 8；
(22)（在 foo() 中）打印 8；
(23)（在 foo() 中）打印 9；
(24) foo() 返回
```

## 四 异步实践

### 4.1 实现 sleep()

利用异步函数实现类似 Java 中的 `Thread.sleep()`：

```js
async function sleep(delay) {
  return new Promise((resolve) => setTimeout(resolve, delay))
}
async function foo() {
  const t0 = Date.now()
  await sleep(1500) // 暂停约 1500 毫秒
  console.log(Date.now() - t0)
}
foo()
// 1502
```

### 4.2 利用平行执行

示例中，顺序等待了 5 个随机的超时：

```js
async function randomDelay(id) {
  // 延迟 0~1000 毫秒
  const delay = Math.random() * 1000
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log(`${id} finished`)
      resolve()
    }, delay)
  )
}
async function foo() {
  const t0 = Date.now()
  await randomDelay(0)
  await randomDelay(1)
  await randomDelay(2)
  await randomDelay(3)
  await randomDelay(4)
  console.log(`${Date.now() - t0}ms elapsed`)
}
foo()
// 0 finished
// 1 finished
// 2 finished
// 3 finished
// 4 finished
// 877ms elapsed
```

用 for 重写：

```js
async function randomDelay(id) {
  // 延迟 0~1000 毫秒
  const delay = Math.random() * 1000
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log(`${id} finished`)
      resolve()
    }, delay)
  )
}
async function foo() {
  const t0 = Date.now()
  for (let i = 0; i < 5; ++i) {
    await randomDelay(i)
  }
  console.log(`${Date.now() - t0}ms elapsed`)
}
foo()
```

就算这些期约之间没有依赖，异步函数也会依次暂停，等待每个超时完成。这样可以保证执行顺序，但总执行时间会变长。如果顺序不是必需保证的，那么可以先一次性初始化所有期约，然后再分别等待它们的结果：

```js
async function randomDelay(id) {
  // 延迟 0~1000 毫秒
  const delay = Math.random() * 1000
  return new Promise((resolve) =>
    setTimeout(() => {
      setTimeout(console.log, 0, `${id} finished`)
      resolve()
    }, delay)
  )
}
async function foo() {
  const t0 = Date.now()
  const p0 = randomDelay(0)
  const p1 = randomDelay(1)
  const p2 = randomDelay(2)
  const p3 = randomDelay(3)
  const p4 = randomDelay(4)
  await p0
  await p1
  await p2
  await p3
  await p4
  setTimeout(console.log, 0, `${Date.now() - t0}ms elapsed`)
}
foo()
```

用数组和 for 循环再包装一下就是：

```js
async function randomDelay(id) {
  // 延迟 0~1000 毫秒
  const delay = Math.random() * 1000
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log(`${id} finished`)
      resolve()
    }, delay)
  )
}
async function foo() {
  const t0 = Date.now()
  const promises = Array(5)
    .fill(null)
    .map((_, i) => randomDelay(i))
  for (const p of promises) {
    await p
  }
  console.log(`${Date.now() - t0}ms elapsed`)
}
foo()
```

虽然期约没有按照顺序执行，但 await 按顺序收到了每个期约的值：

```js
async function randomDelay(id) {
  // 延迟 0~1000 毫秒
  const delay = Math.random() * 1000
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log(`${id} finished`)
      resolve(id)
    }, delay)
  )
}
async function foo() {
  const t0 = Date.now()
  const promises = Array(5)
    .fill(null)
    .map((_, i) => randomDelay(i))
  for (const p of promises) {
    console.log(`awaited ${await p}`)
  }
  console.log(`${Date.now() - t0}ms elapsed`)
}
foo()
```

### 4.3 串行执行期约

使用 async/await 实现期约连锁：

```js
function addTwo(x) {
  return x + 2
}
function addThree(x) {
  return x + 3
}
function addFive(x) {
  return x + 5
}
async function addTen(x) {
  for (const fn of [addTwo, addThree, addFive]) {
    x = await fn(x)
  }
  return x
}
addTen(9).then(console.log) // 19
```

await 直接传递了每个函数的返回值，结果通过迭代产生。当然，这个例子并没有使用期约，如果要使用期约，则可以把所有函数都改成异步函数。这样它们就都返回期约了：

```js
async function addTwo(x) {
  return x + 2
}
async function addThree(x) {
  return x + 3
}
async function addFive(x) {
  return x + 5
}
async function addTen(x) {
  for (const fn of [addTwo, addThree, addFive]) {
    x = await fn(x)
  }
  return x
}
addTen(9).then(console.log) // 19
```

### 4.4 栈追踪与内存管理

期约与异步函数的功能有相当程度的重叠，但它们在内存中的表示则差别很大：

```js
// 拒绝期约栈追踪
function fooPromiseExecutor(resolve, reject) {
  setTimeout(reject, 1000, 'bar')
}
function foo() {
  new Promise(fooPromiseExecutor)
}

foo()
// Uncaught (in promise) bar
// setTimeout
// setTimeout (async)
// fooPromiseExecutor
// foo
```

在超时处理程序执行时和拒绝期约时，我们看到的错误信息包含嵌套函数的标识符，那是被调用以创建最初期约实例的函数。可是，我们知道这些函数已经返回了，因此栈追踪信息中不应该看到它们。这是因为 JavaScript 引擎会在创建期约时尽可能保留完整的调用栈。在抛出错误时，调用栈可以由运行时的错误处理逻辑获取，因而就会出现在栈追踪信息中。当然，这意味着栈追踪信息会占用内存，从而带来一些计算和存储成本。

```js
function fooPromiseExecutor(resolve, reject) {
  setTimeout(reject, 1000, 'bar')
}
async function foo() {
  await new Promise(fooPromiseExecutor)
}
foo()
// Uncaught (in promise) bar
// foo
// async function (async)
// foo
```

这时栈追踪信息就准确地反映了当前的调用栈。fooPromiseExecutor() 已经返回，所以它不在错误信息中。但 foo() 此时被挂起了，并没有退出。JavaScript 运行时可以简单地在嵌套函数中存储指向包含函数的指针，就跟对待同步函数调用栈一样。这个指针实际上存储在内存中，可用于在出错时生成栈追踪信息。这样就不会像之前的例子那样带来额外的消耗，因此在重视性能的应用中是可以优先考虑的。
