# 剖析一个HTTP事务

这篇指南的目的是让我们深入理解Node.js处理HTTP的过程。我们假设你懂的通常情况下HTTP是如何运作的，这与语言和编程环境无关。我们也假设你对Node.js的`EventEmitters`和` Streams`有一定的了解。如果你不是很了解这些内容，很值得快速浏览一遍这些内容的API文档。

## 创建服务

任意node网页服务程序在某种意义上都必须创建一个网页服务对象。可以通过` createServer`来实现。

```js
const http = require('http');

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

`createServer`传入的是一个方法，当任何HTTP请求相应的服务时，这个方法就会被调用，我们称之为请求方法。实际上`createServer `返回的服务对象是一个`EventEmitter`，我们得到的只是一个创建服务的快捷方式，我们会在稍后添加一个监听器。

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





