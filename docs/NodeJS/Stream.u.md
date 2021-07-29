# 为什么要有流?
我们把任何东西都整体拿过来不就行了吗, 为什么要有流?
流 === 数据分包, 那么为什么数据要分包呢?
  数据量特别大 或者 数据不是一次性产出的

数据不是一次性产生很好理解, 比如直播流, 无法作为整体拿过来

当数据量特别大的情况, 那么如果我们直接一次性拿整个数据, 比如一个 400MB 的文件
那么内存的占用量一下就会飙升到 400MB, 非常占用资源, 比如:
```js
const fs = require('fs');

// 不使用流
const server = require('http').createServer();
server.on('request', (req, res) => {
    fs.readFile('./big.file', (err, data) => {
        if(err) throw err;
        res.end(data);
    })
});
server.listen(8000);

// 使用了流
const server = require('http').createServer();
server.on('request', (req, res) => {
  const src = fs.createReadStream('./big.file');
  src.pipe(res);
});
server.listen(8000);
```
重点在于 src.pipe, 这是一个神奇公式:
```js
readableSrc.pipe(writableDest)
```
每次流入一个数据块给 response (writable) 对象, 意味着再也不用在内存中缓存整个对象了


# 关于 Nodejs Stream 最好的文章
https://www.freecodecamp.org/news/node-js-streams-everything-you-need-to-know-c9141306be93/