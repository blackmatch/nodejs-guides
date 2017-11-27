# 剖析一个HTTP事务

这篇指南的目的是让我们深入理解Node.js处理HTTP的过程。我们假设你懂的通常情况下HTTP是如何运作的，这与语言和编程环境无关。我们也假设你对Node.js的`EventEmitters`和`Streams`有一定的了解。如果你不是很了解这些内容，很值得快速浏览一遍这些内容的API文档。

## 创建服务

任意node网页服务程序在某种意义上都必须创建一个网页服务对象。可以通过`createServer`来实现。

```js
const http = require('http');

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

`createServer`传入的是一个方法，当任何HTTP请求相应的服务时，这个方法就会被调用，我们称之为请求方法。实际上`createServer`返回的服务对象是一个`EventEmitter`，我们得到的只是一个创建服务的快捷方式，我们会在稍后添加一个监听器。

```js
const server = http.createServer();
server.on('request', (request, response) => {
  // the same kind of magic happens here!
});
```

当一个HTTP请求传达到服务器端，node会调用请求处理函数，使用一些方便的对象来处理事务，即发起请求和响应请求。我们很快就会接触到这些内容。

为了自然而然地响应请求，`listen`方法需要在服务器实例中被调用。大多数情况下，你只需要把你想监听的端口传递给`listen`方法就可以了。当然也有其他一些可选的参数，你可以查阅[相关的API文档](https://nodejs.org/api/http.html)。

## 请求方法，路径和请求头

当发起一个请求的时候，你最关心的第一件事就是看请求的方法和路径，以便做出合适的相应。Node通过把这些属性添加到`request`对象中，使得这一切变得相对简单。

> 注意：`request`是`IncomingMessage`类的一个实例。

这里的请求方法指的是一个正规的HTTP方法（或者说动词）。路径指的是一个完整的URL，但是不包含服务名、协议和端口号。一个典型的URL，是指第三个正斜杠（包含）往后的所有内容。

接下来该说说请求头了。请求头作为一个对象被存放在`request`的`headers`属性中。

```js
const { headers } = request;
const userAgent = headers['user-agent'];
```

这里有一点非常重要，就是不管客户端以怎样的形式发送请求头，所有的请求头都会被转换成全小写字母的形式。这样做简化了在任何情况下对请求头的解析。

如果一些请求头被重复发送了，这些请求头的值会被覆盖或者会议逗号分隔的形式串联成字符串作为请求头的值。在某些情况下，这样会带来麻烦，所以`rawHeaders`也是可以使用的。

## 请求体

当接收到一个`POST`或者一个`put`请求时，`body`对你的程序来说就显得比较重要了。获取`body`的数据比获取请求头稍微复杂一点。通过传递给一个处理程序，`request`对象实现了可读流的接口功能。这种流可以像其他任何流一样被监听或者传送。我们可以通过监听流的`data`和`end`事件来获取流的数据。

每一次`data`事件触发的时候，数据块会以`Buffer`对象的形式被传递。如果你知道传输的是一个字符串类型的数据，最好的方式就是把每一次的块数据收集到一个数组中，在触发`end`事件的时候，把这个数组连接起来，然后转成字符串类型。

```js
let body = [];
request.on('data', (chunk) => {
  body.push(chunk);
}).on('end', () => {
  body = Buffer.concat(body).toString();
  // at this point, `body` has the entire request body stored in it as a string
});
```

> 注意：这样做可能有点啰嗦繁琐，但是大多数情况都是这么做的。幸运的是，在npm中，有很多像`concat-stream`和`body`这样的模块可以把这些繁琐的逻辑简化。再继续往下之前，好好理解这些原理是非常重要的，这也是你为什么读这篇指南的原因。

## 快速理解错误

由于`request`对象是一个可读流实例，也是一个`EventEmitter`实例，所以产生错误的行为也一样。

`request`流中的错误通过提交`error`时间来呈现。**如果你没有监听这个时间，错误就会被抛出，导致程序崩溃。**所以你应该在你的`request`流中监听错误事件，即使你只是在这个事件里打印点东西然后继续执行。（不过最后的方式是返回一些HTTP错误响应。稍后我们会讨论更多。）

```js
request.on('error', (err) => {
  // This prints the error message and stack trace to `stderr`.
  console.error(err.stack);
});
```

处理[这些错误](https://nodejs.org/api/errors.html)还有诸如抽象化和工具之类的方法，但是必须注意的是，错误是存在并且确实会发生的，所以你必须要处理好这些错误。

## 小结一下

目前，我们已经讨论了如何创建一个服务端，并从请求中获取请求方法，URL，请求头以及`body`。当我们这所有这些知识点都组合起来，就得到了下面的代码：

```js
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // At this point, we have the headers, method, url and body, and can now
    // do whatever we need to in order to respond to this request.
  });
}).listen(8080); // Activates this server, listening on port 8080.
```

如果我们运行这个例子的代码，我们可以发送请求，但是得不到响应。事实上，如果你在浏览器上跑这个例子，会出现请求超时，因为没有任何东西返回给客户端。

目前为止我们没有设计到任何关于`response`对象的内容，`response`对象是一个`ServerResponse`实例，是一个可写流。这个对象包含了很多给客户端发送数据的方法。我们稍后会谈到这些。

HTTP状态码

如果你不进行相关的设置，HTTP在响应的时候总是`200`。当然，并不是每一个HTTP响应都希望返回这个状态码，在一些特定的情况下，你可能需要设置一些不同的状态码。你可以通过设置`statusCode`属性来实现。

```js
response.statusCode = 404; // Tell the client that the resource wasn't found.
```

也有一些其他的快捷方式来实现，我们稍后会介绍。

## 设置响应头

通过`setHeader`方法，可以很方便的设置响应头。

```js
response.setHeader('Content-Type', 'application/json');
response.setHeader('X-Powered-By', 'bacon');
```

在设置响应头的时候，名称是大小写敏感的。如果你重复设置了相应头，最后一个会把前面设置的值覆盖掉。

## 显式发送头数据

上述谈到的设置头部消息状态码的方法我们称之为『隐式头部』。也就意味着，在发送主体数据之前，你是
依赖node在正确的时间发送头部消息。

只要你想，你可以明确地把头部信息写在响应流中。为了实现这点，有一个叫`writeHead`的方法，它可以
把状态码和头部消息写到流中。

```js
response.writeHead(200, {
  'Content-Type': 'application/json',
  'X-Powered-By': 'bacon'
});
```

一旦你设置了头部消息（不管是显式还是隐式），就已经做好了发送响应数据的准备了。

## 发送响应主体

由于`response`对象是可写流，写入响应数据发送到客户端只是一件调用通常的流方法的事而已。

```js
response.write('<html>');
response.write('<body>');
response.write('<h1>Hello, World!</h1>');
response.write('</body>');
response.write('</html>');
response.end();
```

`end`方法在流中也可以接受一些数据（可选）作为最后一部分数据进行发送，所以上面的例子可以简化为：

```js
response.end('<html><body><h1>Hello, World!</h1></body></html>');
```

> 注意：在开始写数据块到主体之前，设置好状态码和头部消息是非常重要的。这是有理由的，因为头部消息
会在发送主题数据之前先发送。

## 再来快速过一下错误

`response`流也可以提交`error`事件，而且在某些情况下，你必须处理好错误。所有关于`request`流
的建议在这里都适用。

## 整合起来

现在我们已经学习了如何响应一个HTTP请求，现在我们把这些都整合起来。基于前面的例子，我们将创建一个
服务端来实现，把用户发送的数据全部返回。我们会使用`JSON.stringify`把数据格式化成JSON格式。

```js
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // BEGINNING OF NEW STUFF

    response.on('error', (err) => {
      console.error(err);
    });

    response.statusCode = 200;
    response.setHeader('Content-Type', 'application/json');
    // Note: the 2 lines above could be replaced with this next one:
    // response.writeHead(200, {'Content-Type': 'application/json'})

    const responseBody = { headers, method, url, body };

    response.write(JSON.stringify(responseBody));
    response.end();
    // Note: the 2 lines above could be replaced with this next one:
    // response.end(JSON.stringify(responseBody))

    // END OF NEW STUFF
  });
}).listen(8080);
```

## 一个回声服务端的例子

我们把前面的例子简化一下，实现一个响应服务端，这个服务端仅仅将从请求中接收到的数据（不管是啥数据）
直接返回。我们只需要做的就是，获取请求流中的数据，然后把这些数据写入到相应流中，类似于我们之前的
操作。

```js
const http = require('http');

http.createServer((request, response) => {
  let body = [];
  request.on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    response.end(body);
  });
}).listen(8080);
```

我们来修改一下上面的例子。我们只想发送一个回声消息，基于一下条件：

* 使用POST请求
* 请求路径是`/echo`

其他任何情况下，我们让它简单返回一个404好了。

```js
const http = require('http');

http.createServer((request, response) => {
  if (request.method === 'POST' && request.url === '/echo') {
    let body = [];
    request.on('data', (chunk) => {
      body.push(chunk);
    }).on('end', () => {
      body = Buffer.concat(body).toString();
      response.end(body);
    });
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

> 注意：使用这种方式来判别路径，我们称之为『路由』。其他实现路由的方式可以是`switch`语句，或者
像`express`这样的复杂框架。如果你只是想实现路由功能，你可以试试`router`这个模块。