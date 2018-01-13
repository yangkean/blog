# Socket.IO 学习总结

## Socket.IO 是什么？

**Socket.IO** 是一个面向实时 web 应用的 JavaScript 库。它使得服务器和客户端之间实时双向的通信成为可能。他有两个部分：在浏览器中运行的客户端库，和一个面向 Node.js 的服务端库。两者有着几乎一样的 API。像 Node.js 一样，它也是[事件驱动](https://zh.wikipedia.org/w/index.php?title=%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1&action=edit&redlink=1)的。**一句话，这是用来写实时应用的库。**

下面我从官方文档中截取部分客户端和服务器端 API，你只需简单浏览了解就可以写个简单的聊天室了。

## Server API

### Server

#### new Server(httpServer[, options])

* httpServer `http.Server` - 要绑定的 http 服务器
* options `object` - 见 [options](https://socket.io/docs/server-api/#new-server-httpserver-options)

新建 *socket.io* 实例有两种方法：

```js
const io = require('socket.io')();
// or
const Server = require('socket.io');
const io = new Server();
```

#### server.of(nsp)

* nsp `String`
* Returns `Namespace`

初始化并获得给定路径名 `nsp` 的命名空间 (*namespace*)。

```js
const adminNamespace = io.of('/admin');
```

### Namespace

一个特定作用域 (*scope*) 下连接的 socket 的池 (*pool*)。

#### namespace.emit(eventName[, ...args])

* eventName `String`
* args

发送一个事件给该命名空间下所有连接的客户端。

```js
const io = require('socket.io')();
io.emit('an event sent to all connected clients'); // 默认的主命名空间

const chat = io.of('/chat');
chat.emit('an event sent to all connected clients in chat namespace'); // 'chat' 命名空间
```

#### 常见事件：

##### 'connect' / 'connection'

接收参数：`socket` - {Socket} 和客户端的 *socket* 连接

当客户端发起一个连接时触发此事件。

```js
io.on('connection', (socket) => {
  // ...
});
```

### Socket

一个`Socket`是与浏览器客户端交互的基础类 (*class*)，属于一个特定的命名空间 (*namespace*) (默认是 `/`)，使用底层的客户端来交流。

#### socket.emit(eventName[, ...args][, ack])

* eventName `String`
* args
* ack `Function`
* Returns `Socket`

只发送事件给发送者的客户端。`ack`参数作为一个函数传给客户端然后由客户端调用但在服务器端执行。

```js
io.on('connection', (socket) => {
  socket.emit('ferret', 'tobi', (data) => {
    console.log(data); // data will be 'hello'
  });

  // the client code
  // client.on('ferret', (name, fn) => {
  //   console.log(name); // 客户端打印 'tobi'
  //   fn('hello'); // 服务器端打印 'hello'
  // });
});
```

#### socket.on(eventName, callback)

* eventName `String`
* callback `Function`
* Returns `Socket`

监听事件并处理之。

```js
socket.on('news', (data) => {
  console.log(data);
});
```

#### 常用标志：

##### 'broadcast'

设置了此修饰符的事件将发送给相应命名空间下的所有客户端，除了发送此事件的人。

```js
io.on('connection', (socket) => {
  socket.broadcast.emit('an event', { some: 'data' }); // 除了发送者每个人都接收这一事件
});
```

#### 常见事件：

##### 'disconnect' / 'disconnecting'

断开连接时触发此事件。

接收参数：`reason` - {String} 断开连接的原因

```js
io.on('connection', (socket) => {
  socket.on('disconnect', (reason) => {
    // ...
  });
});
```

##### 'error'

错误发生时触发此事件。

接收参数：`error` - {Object} error 对象

```js
io.on('connection', (socket) => {
  socket.on('error', (error) => {
    // ...
  });
});
```

## Client API

### IO
 
作为 `io` 命名空间暴露出来。

#### io([url][, options])

* url `String` (默认是 `window.location`)
* options `Object` - 见 [options](https://socket.io/docs/client-api/#new-manager-url-options)
* Returns `Socket`

新建 socket.io 客户端管理器 (*manager*) 可以这样做：

```html
<script src="/socket.io/socket.io.js"></script>
<script>
  const socket = io('http://localhost');
</script>
````

### Socket

#### socket.open() / socket.connect()

* Returns `Socket`

手动打开 socket。

```js
socket.on('disconnect', () => {
  socket.open();
});
```

#### socket.emit(eventName[, ...args][, ack])

同 server API `socket.emit(eventName[, ...args][, ack])`

> **Tips**\
注意 `namespace.emit(eventName[, ...args])` 是发送事件给该命名空间下的所有连接的客户端；`socket.emit(eventName[, ...args][, ack])` 只发送事件给发送者的客户端；`socket.broadcast.emit(eventName[, ...args][, ack])` 是除发送事件的人外发送给相应命名空间下的所有客户端。

#### socket.on(eventName, callback)

同 server API `socket.on(eventName, callback)`

#### 常见事件

##### 'connect'

连接到服务器端时触发，包括重连 (*reconnection*)。

```js
socket.on('connect', () => {
  // ...
});

// 注意：与服务器端不同的是，你应该在连接事件之外注册事件处理
// 器，避免了在重连时重复注册
socket.on('myevent', () => {
  // ...
});
```

##### 'error'

接收参数：`error` - {Object} error 对象

错误发生时触发此事件。

```js
socket.on('error', (error) => {
    // ...
});
```

##### 'disconnect'

* `reason` - {String} 断开连接的原因

断开连接时触发事件。

```js
socket.on('disconnect', (reason) => {
  // ...
});
```

## 写个简单的聊天室

我照着官方的撸了个简易的，demo 见[这里](https://github.com/yangkean/socket.io-demo)。

## Reference

* [Socket.IO - Docs](https://socket.io/docs/)
* [Get Started: Chat application](https://socket.io/get-started/chat/)
* [Socket.IO - 维基百科](https://zh.wikipedia.org/wiki/Socket.IO)
