# 05-Node 的事件模块

## 一 使用发布订阅模式

异步编程中使用最广的设计模式是发布订阅模式，也常称呼为事件监听机制。Node 的 events 模块是其简单实现，大多 Node 的其他模块都继承自它。

```js
const events = require('events')

let emitter = new events()

// 订阅事件 event1
emitter.on('event1', () => {
  console.log('1--')
})

emitter.on('event1', (msg) => {
  console.log('2--', msg)
})

// 发布
emitter.emit('event1', 'hello world')
```

输出结果为：

```txt
1--
2-- hello world
```

事实上，其本质是操作了一个这样的对象：

```json
{
  "event1": [fn1, fn2]
}
```

事件的发布订阅模式可以实现一个事件和多个回调函数的关联，这些回调函数可以称为事件侦听器，使用 emit() 发布事件后，消息会立即传递给当前事件的所有侦听器执行。

此外该对象的方法还有：once（执行一次）、off（取消订阅）。

Node 还有一个内部定义的特殊事件：`newListener`：

```js
// 每次用户调用了一次 on，则执行该事件的方法
emitter.on('newListener', (type) => {})
```

## 二 理解发布订阅模式

发布订阅只是一种设计模式，无关异步、同步，但是在 Node 中，emit() 调用多半伴随了事件循环而异步触发，所以说事件发布/订阅可以广泛用于异步编程。

同样的，发布订阅也解耦了应用程序：事件的发布者无序关注订阅者（即侦听器）如何实现业务，消息只要能在订阅者和发布者之间流转即可。

当然，事件的侦听器模式也可以理解为一种钩子机制（hooks），利用钩子导出内部数据、状态给外部的调用者，比如在 Node 中：

```js
var options = {
  host: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
}

var req = http.request(options, function (res) {
  console.log('STATUS: ' + res.statusCode)
  console.log('HEADERS: ' + JSON.stringify(res.headers))

  res.setEncoding('utf8')

  res.on('data', function (chunk) {
    console.log('BODY: ' + chunk)
  })

  res.on('end', function () {
    // TODO
  })
})

req.on('error', function (e) {
  console.log('problem with request: ' + e.message)
})

// write data to request body
req.write('data\n')
req.write('data\n')
req.end()
```

在上述代码中，开发者只需要把视线放在 error、data、end 这些业务中即可，内部流程的运转，无需过多关注。

Node 对发布订阅机制增加了健壮性处理：

- 事件侦听器超过 10 个，将会触发警告，以避免内存泄露等现象产生
- 在运行期间如果触发了 error 事件，EventEmitter 对象会检查是否有对 error 事件添加过侦听器，如果添加了则 erro 事件由侦听器处理，否则将抛出异常

实现一个继承自 EventEmitter 类的方式：

```js
var events = require('events')

function Stream() {
  events.EventEmitter.call(this)
}
util.inherits(Stream, events.EventEmitter)
```

## 三 事件队列解决雪崩问题

缓存是软件系统中基于内存的数据存储系统，访问速度极快，常用于一些对性能要求较高的场景。但是在高并发场景中，缓存容易失效，大量的请求穿过缓存同时涌入数据库，会引起数据库服务宕机。

事件的发布订阅模式中，once() 方法可以解决雪崩问题。

普通的数据库查询：

```js
function select(callback) {
  db.select('SQL', function (results) {
    callback(results)
  })
}
```

第一种改进方案是添加一个类似锁的变量：

```js
var status = 'ready'
function select(callback) {
  if (status === 'ready') {
    status = 'pending'
    db.select('SQL', function (results) {
      status = 'ready'
      callback(results)
    })
  }
}
```

上述场景中，连续多次调用 select() 时，只有第一次调用是生效的，后续的 select() 是没有数据服务的，这时候可以引入事件队列：

```js
var proxy = new events.EventEmitter()
var status = 'ready'
function select(callback) {
  proxy.once('selected', callback)

  if (status === 'ready') {
    status = 'pending'
    db.select('SQL', function (results) {
      proxy.emit('selected', results)
      status = 'ready'
    })
  }
}
```

上述的 once 方法会将所有请求压入事件队列中，利用其执行一次就移除监视器的特点，保证每一个回调函数只会被执行一次。此处可能引发侦听器过多警告，可以通过调用`setMaxListeners(0)`移除警告或者设置更大值。

## 四 手写 EventEmitter

```js
function Event() {
  this._events = {}
}

Event.prototype.on = function (eName, callback) {
  // 支持继承类调用
  if (!this._events) {
    this._events[eName] = Object.create(null)
  }

  if (this._events[eName]) {
    this._events[eName].push(callback)
  } else {
    this._events[eName] = [callback]
  }
}

Event.prototype.emit = function (eName, ...args) {
  if (this._events[eName]) {
    this._events[eName].forEach((fn) => fn(...args))
  }
}

Event.prototype.off = function (eName, callback) {
  if (!this._events) {
    return
  }
  this._events[eName] = this._events[eName].filter((fn) => {
    fn !== callback && fn.l !== callback
  })
}

Event.prototype.once = function (eName, callback) {
  const once = (...args) => {
    callback(...args)
    this.off(eName, once)
  }
  // 标识这个 once 是谁的
  once.l = callback
  this.on(eName, once)
}

module.exports = Event
```
