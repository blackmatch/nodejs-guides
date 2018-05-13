# 安装完`Node.js`后该如何开始呢？

如果你已经安装了`Node`，我们开始构建第一个服务端程序吧。创建一个名为"app.js"的文件，然后粘贴下面的代码：

```js
const http = require('http');

const hostname = '127.0.0.1';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

然后，使用`node app.js`命令启动程序，访问[http://localhost:3000/](http://localhost:3000/)，
你会看到一条'Hello World'消息。
