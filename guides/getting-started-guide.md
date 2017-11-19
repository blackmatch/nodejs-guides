# 安装完Node.js后我该如何开始使用它呢？

如果你已经安装了Node，让我们开始构建第一个web服务吧。创建一个文件，取名"app.js"，然后粘贴下面的代码到文件中：

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

然后，使用`node app.js`启动你的web服务，访问[http://localhost:3000/](http://localhost:3000/)，你会看到一条'Hello World'消息。



