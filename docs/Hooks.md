<h1 align="center">Fastify</h1>

## 钩子方法

钩子 (hooks) 让你能够监听应用或请求/响应生命周期之上的特定事件。使用 `fastify.addHook` 可以注册钩子。你必须在事件被触发之前注册相应的钩子，否则，事件将得不到处理。

## 请求/响应钩子

通过钩子方法，你可以在 Fastify 的生命周期内直接进行交互。有四个可用的钩子 *(按执行顺序排序)*：
- `'onRequest'`
- `'preHandler'`
- `'onSend'`
- `'onResponse'`

示例：
```js
fastify.addHook('onRequest', (req, res, next) => {
  // 其他代码
  next()
})

fastify.addHook('preHandler', (request, reply, next) => {
  // 其他代码
  next()
})

fastify.addHook('onSend', (request, reply, payload, next) => {
  // 其他代码
  next()
})

fastify.addHook('onResponse', (res, next) => {
  // 其他代码
  next()
})
```
或使用 `async/await`
```js
fastify.addHook('onRequest', async (req, res) => {
  // 其他代码
  await asyncMethod()
  // 错误处理
  if (err) {
    throw new Error('some errors occurred.')
  }
  return
})

fastify.addHook('preHandler', async (request, reply) => {
  // 其他代码
  await asyncMethod()
  // 错误处理
  if (err) {
    throw new Error('some errors occurred.')
  }
  return
})

fastify.addHook('onSend', async (request, reply, payload) => {
  // 其他代码
  await asyncMethod()
  // 错误处理
  if (err) {
    throw new Error('some errors occurred.')
  }
  return
})

fastify.addHook('onResponse', async (res) => {
  // 其他代码
  await asyncMethod()
  // 错误处理
  if (err) {
    throw new Error('some errors occurred.')
  }
  return
})
```

**注意：** 使用 `async`/`await` 或返回一个 `Promise` 时，`next` 回调不可用。在这种情况下，仍然使用 `next` 可能会导致难以预料的行为，例如，处理器的重复调用。

|  参数  |  说明  |
|-------------|-------------|
| req | Node.js 的 [IncomingMessage](https://nodejs.org/api/http.html#http_class_http_incomingmessage) 对象 |
| res | Node.js 的 [ServerResponse](https://nodejs.org/api/http.html#http_class_http_serverresponse) 对象 |
| request | Fastify 的 [Request](https://github.com/fastify/docs-chinese/blob/master/docs/Request.md) 接口 |
| reply | Fastify 的 [Reply](https://github.com/fastify/docs-chinese/blob/master/docs/Reply.md) 接口 |
| payload | 序列化的 payload |
| next | 调用该函数执行[生命周期](https://github.com/fastify/docs-chinese/blob/master/docs/Lifecycle.md)的下一阶段 |

[生命周期](https://github.com/fastify/docs-chinese/blob/master/docs/Lifecycle.md)一文清晰地展示了各个钩子执行的位置。<br>
钩子可被封装，因此可以运用在特定的路由上。更多信息请看[作用域](#scope)一节。

在钩子的执行过程中如果发生了错误，只需将错误传递给 `next()`，Fastify 就会自动关闭请求，并发送一个相应的错误码给用户。

```js
fastify.addHook('onRequest', (req, res, next) => {
  next(new Error('some error'))
})
```

如果你想自定义发送给用户的错误码，使用 `reply.code()` 即可：
```js
fastify.addHook('preHandler', (request, reply, next) => {
  reply.code(500)
  next(new Error('some error'))
})
```

*错误最终会在 [`Reply`](https://github.com/fastify/docs-chinese/blob/master/docs/Reply.md#errors) 中得到处理*

请注意，`'preHandler'` 和 `'onSend'` 钩子中的请求与回复对象与 `'onRequest'` 中的对象不同。这是因为前两者中的对象是 Fastify 的 [`request`](https://github.com/fastify/docs-chinese/blob/master/docs/Request.md) 与 [`reply`](https://github.com/fastify/docs-chinese/blob/master/docs/Reply.md)，而后者的则是 Node 的 `req` 与 `res` 对象。

#### `onSend` 钩子

使用 `onSend` 钩子可以改变 payload。例如：

```js
fastify.addHook('onSend', (request, reply, payload, next) => {
  var err = null;
  var newPayload = payload.replace('some-text', 'some-new-text')
  next(err, newPayload)
})

// 或者使用 async
fastify.addHook('onSend', async (request, reply, payload) => {
  var newPayload = payload.replace('some-text', 'some-new-text')
  return newPayload
})
```

你也可以通过将 payload 置为 `null`，发送一个空消息主体的响应：

```js
fastify.addHook('onSend', (request, reply, payload, next) => {
  reply.code(304)
  const newPayload = null
  next(null, newPayload)
})
```

> 将 payload 设为空字符串 `''` 也可以发送空的消息主体。但要小心的是，这么做会造成 `Content-Length` header 的值为 `0`。而 payload 为 `null` 则不会设置 `Content-Length` header。

注：你只能将 payload 修改为 `string`、`Buffer`、`stream` 或 `null`。

### 在钩子中响应请求
需要的话，你可以在路由控制器执行前响应一个请求。一个例子便是身份验证的钩子。如果你在 `onRequest` 或中间件中发出响应，请使用 `res.end`。如果是在 `preHandler` 钩子中，使用 `reply.send`。

```js
fastify.addHook('onRequest', (req, res, next) => {
  res.end('early response')
})

// 也可使用 async 函数
fastify.addHook('preHandler', async (request, reply) => {
  reply.send({ hello: 'world' })
})
```

如果你想要使用流 (stream) 来响应请求，你应该避免使用 `async` 函数。必须使用 `async` 函数的话，请参考 [test/hooks-async.js](https://github.com/fastify/docs-chinese/blob/94ea67ef2d8dce8a955d510cd9081aabd036fa85/test/hooks-async.js#L269-L275) 中的示例来编写代码。

```js
const pump = require('pump')

fastify.addHook('onRequest', (req, res, next) => {
  const stream = fs.createReadStream('some-file', 'utf8')
  pump(stream, res, err => req.log.error(err))
})
```

## 应用钩子

你也可以在应用的生命周期里使用钩子方法。要格外注意的是，这些钩子并未被完全封装。钩子中的 `this` 得到了封装，但处理器可以响应封装界线外的事件。

- `'onClose'`
- `'onRoute'`

<a name="on-close"></a>
**'onClose'**<br>
使用 `fastify.close()` 停止服务器时被触发。当[插件](https://github.com/fastify/docs-chinese/blob/master/docs/Plugins.md)需要一个 "shutdown" 事件时有用，例如一个用于连接数据库的插件。<br>
该钩子的第一个参数是 Fastify 实例，第二个为 `done` 回调函数。
```js
fastify.addHook('onClose', (instance, done) => {
  // 其他代码
  done()
})
```
<a name="on-route"></a>
**'onRoute'**<br>

当注册一个新的路由时被触发。它的监听函数拥有一个唯一的参数：`routeOptions` 对象。该函数是同步的，其本身并不接受回调作为参数。
```js
fastify.addHook('onRoute', (routeOptions) => {
  // 其他代码
  routeOptions.method
  routeOptions.schema
  routeOptions.url
  routeOptions.bodyLimit
  routeOptions.logLevel
  routeOptions.prefix
})
```
<a name="scope"></a>
### 作用域
除了[应用钩子](#application-hooks)，所有的钩子都是封装好的。这意味着你可以通过 `register` 来决定在何处运行它们，正如[插件指南](https://github.com/fastify/docs-chinese/blob/master/docs/Plugins-Guide.md)所述。如果你传递一个函数，那么该函数会获得 Fastify 的上下文，如此你便能使用 Fastify 的 API 了。

```js
fastify.addHook('onRequest', function (req, res, next) {
  const self = this // Fastify 上下文
  next()
})
```
注：使用箭头函数会破坏 Fastify 实例对 this 的绑定。

<a name="before-handler"></a>
### beforeHandler
别被 `beforeHandler` 的名字迷惑了。不像 `preHandler` 是一个标准的钩子，`beforeHandler` 只是一个在路由选项中注册的函数，仅在该路由内执行。当你想在路由层面而非钩子层面 (例如 `preHandler`) 处理身份认证时，它能派上用场。`beforeHandler` 还可以是一个函数数组。<br>
**`beforeHandler` 总是在 `preHandler` 之后执行。**

```js
fastify.addHook('preHandler', (request, reply, done) => {
  // 你的代码
  done()
})

fastify.route({
  method: 'GET',
  url: '/',
  schema: { ... },
  beforeHandler: function (request, reply, done) {
    // 你的代码
    done()
  },
  handler: function (request, reply) {
    reply.send({ hello: 'world' })
  }
})

fastify.route({
  method: 'GET',
  url: '/',
  schema: { ... },
  beforeHandler: [
    function first (request, reply, done) {
      // 你的代码
      done()
    },
    function second (request, reply, done) {
      // 你的代码
      done()
    }
  ],
  handler: function (request, reply) {
    reply.send({ hello: 'world' })
  }
})
```
