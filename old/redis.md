# 用 Koa2+Redis 写一个简单 Todo App

>**Tips**
> [*Redis*](http://www.redis.cn/) 是一个使用 `ANSI C` 编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库 (*key-value database*)。
> 
> 本文是参考 [Node & Express Todo App: Redis](https://javascriptplayground.com/blog/2012/06/node-express-todo-app-redis) 而写。如果你对 Redis 还不甚了解，可以先看看 [The Little Redis Book](http://openmymind.net/2012/1/23/The-Little-Redis-Book/)。

## 准备

我们用 Koa 写后端，用 Redis 做数据库存储，用 ejs 做模板引擎，简单地演示下如何结合 Node 和 Redis 写应用。适合 Redis 入门者。本文假设你已经掌握 [Koa](https://github.com/koajs/koa) 的基本用法。

koa2 需要 async/await 的支持，请确定你的 node 的版本不小于 7.6.0。

```bash
$ npm i --save koa koa-body koa-router koa-static koa-views ejs
```

编写基本的路由：

**index.js**

```js
const Koa = require('koa');
const Router = require('koa-router');
const views = require('koa-views');
const path = require('path');
const koaBody = require('koa-body');
const serve = require('koa-static');
const app = new Koa();
const router = new Router();

// 设置模板目录
app.use(views(path.join(__dirname, '/views'), {
  extension: 'ejs'
}));

// 设置静态文件目录
app.use(serve(path.join(__dirname, '/public')));

app.use(koaBody());

router.get('/', index)
      .post('/', toList);

app.use(router.routes());

async function index(ctx) {
 await ctx.render('index', {lists: ''});
}

async function toList(ctx) {
  // todo
}

app.listen(3000);
```

编写模板：

**views/index.ejs**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>TODO list</title>
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <h1>TODO list</h1>
    <form action="/" method="post">
      <input type="input" placeholder="new todo here and press Enter" class="todo-list" name="list">
      <input type="submit" value="Add" class="submit">
    </form>
    <% if(lists) { %>
    <ul>
       <% for(list in lists) { %>
        <li><%= lists[list] %></li>
       <% } %>
    </ul>
    <% } %>
  </body>
</html>
```

编写样式：

**public/style.css**

```css
.todo-list {
  width: 300px;
  line-height: 30px;
  margin-bottom: 20px;
  font-size: 1.1em;
}
.submit {
  margin-left: 20px;
}
```

现在执行`node index.js`，访问`http://localhost:3000/`就可以看到简陋的界面了，好，现在，我们连接 Redis 的数据库，并读取数据。如果你没有安装 Redis 的服务器，请参照[下载](http://www.redis.cn/download.html)安装并启动 Redis 服务器。

在 node 中要使用 Redis，我们要借助 node 的 Redis 客户端 **[node_redis](https://github.com/NodeRedis/node_redis)**。

```bash
$ npm i --save redis
```

在`index.js`头部引入依赖处加入如下两行：

```js
const redis = require('redis');
const client = redis.createClient(); // 返回一个 `RedisClient ` 对象
```

因为我们使用 async/await 来写函数的，但是 node_redis 是用回调来解决异步问题的，到现在仍然是不支持 promise，我们可以选择用 [bluebird](https://github.com/NodeRedis/node_redis#promises)包装 node_redis 的函数。好消息是，我们有一个更好的解决方法，node 从8.0.0 开始支持一个叫`util.promisify()`的函数，这个原生函数足以[实现](http://2ality.com/2017/05/util-promisify.html)我们的需求。

在`index.js`头部加入这一行：

```js
const promisify = require('util').promisify;
```

好，我们现在来完善我们的两个重要函数：

```js
async function index(ctx) {
  let lists = '';

  const hgetall = promisify(client.hgetall).bind(client);
  const obj = await hgetall('todo:samyang');

  if(obj) {
    lists = obj;
  }

  await ctx.render('index', {lists: lists});
}
```

`client.hgetall`是 node_redis 提供的函数，和 Redis 提供的哈希 (*hash*) 结构命令 [hgetall](http://redisdoc.com/hash/hgetall.html) 用法一样，我们用`util.promisify()`将其包装为能返回 promise 的函数，并绑定上下文为 `RedisClient ` 对象，即 `client`，然后往这个函数传入我们的 key，这个 key 我们接下来会用来做存储的 key，如果从 Redis 的数据库中 get 到了数据，会返回一个包含键值对数据的对象，否则返回 null。

接下来我们往 Redis 数据库里写东西。

```js
async function toList(ctx) {
  const list = ctx.request.body.list.trim();
  const field = list.replace(' ', '-');

  const hset = promisify(client.hset).bind(client);
  await hset('todo:samyang', field, list);

  ctx.redirect('/');
}
```

我们把前端提交的 todo 项用 `-` 连接作为 field，这个 field，其实就是上面用`hgetall`拿到的数据对象里的键名，然后我们用 hset 这个命令将 todo 项存入 Redis 数据库。

再次运行`node index.js`，访问`http://localhost:3000/`。尝试输入一些 todo 项，你会看到它们被立刻添加到页面上了，说明它被成功添加到 Redis 数据库中了，再添加几项试试。

我们来看看 Redis 命令行客户端查询的结果：

```bash
$  redis-cli
127.0.0.1:6379> hgetall todo:samyang
 1) "do-homework"
 2) "do homework"
 3) "wtite-node"
 4) "wtite node"
 5) "buy-meat"
 6) "buy meat"
127.0.0.1:6379>
```

大功告成！代码我放到[这里](https://github.com/yangkean/redis-demo)了。
