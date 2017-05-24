http会话的解析
====

这个指南的目的是传授一个对node.js中http处理过程更准确的理解.我们将假设你大致了解http请求的工作原理，无论是什么语言或者编程环境下。我们也将估计你会熟悉一点关于node.js的<code>EventEmitters</code>和<code>Streams</code>.如果你还没很熟悉，那你需要去通过api文档快速浏览一下他们.

创建服务器
-----

任何node的web服务器应用都在某些时候必须创建一个web服务对象.这通常是使用<code>createServer</code>完成.

```javascript
var http = require('http');

var server = http.createServer(function(request, response) {
  // magic happens here!
});.
```

<code>createServer</code>每被调用一次时传进去的方法函数是针对该服务器的每个http请求的处理方法,因此成为request handler.事实上,通过<code>createServer</code>返回的服务器对象是一个<code>EventEmitter</code>,我们先在这里简单的把这些行为成为制造一个`server`对象并随之添加一个监听者.

```javascript
var server = http.createServer();
server.on('request', function(request, response) {
  // the same kind of magic happens here!
});
```
当一个http请求hit该服务器,node调用请求处理方法通过一些封装好的对象去处理这个会话，`request`和`response`.我们很快会说到那里的。
为了真正的服务这些请求,`listen`方法需要被`server`对象上调用.在大多数情况下,你只需要传递给`listen`方法的参数是你需要服务器去监听的端口.这里也有些其他的选项,请参考[API reference](https://nodejs.org/api/http.html).

请求方法,url和请求头部
-----

当处理一个请求,第一件事你可能去做就是观察请求方法和url,以便做出适当的处理.node通过在`request`对象上添加这些的属性使其获取时相对简便.

```javascript
var method = request.method;
var url = request.url;
```
    **注意:这`request`对象是`IncomingMessage`的实例**</br></br>
这个`method`在这里总是一个普通的HTTP方法/动词.这个`url`没有了服务器名字,协议或者端口的完整url,对于通常的url来说,意思是包括第三个正斜杠以及之后的所有内容.

请求头也是。他们在`request`对象上有自己对象，叫做header.
```javascript
var headers = request.headers;
var userAgent = headers['user-agent'];
```
记住一点,所有的header都仅是小写形式显示，无论客户端怎么实际上怎么发送它们。这里简化了为任何目的而解析header任务.

如果一些header重复了,那么他们的值将会重写或者连接起来形成一个有分隔符的字符串,依赖于这个header上.在某些情况下,这个会是一个隐患,所以<code>rawHeaders</code>也是可用的.

请求体
-----

当收到一个`POST`或者`PUT`请求时,这个请求体可能对你的程序来说非常重要的。因此收集请求体的数据相对比获取请求头来说更为频繁一些。这个在handler的参数`request`对象是由`ReadableStream`接口实现的.这可读流可以像其他流一样被监听或者被piped到其他地方.我们可以通过监听可写流的`'data'`和`'event'`事件来抓取可读流中数据。

每一个`data`事件在响应时所提供的块(chunk)都是`Buffer`.如果你知道将会是字符串数据,最好的处理方法就是把数据收集在数组中,然后在`end`事件中,把字符串连接起来.

```javascript
var body = [];
request.on('data', function(chunk) {
  body.push(chunk);
}).on('end', function() {
  body = Buffer.concat(body).toString();
  // at this point, `body` has the entire request body stored in it as a string
});
```
    **注意 这看上去可能很乏味,而且在大多情况下,也真的是.幸运的是,这里在`npm`上有一些模块例如`concat-stream` 和
    `body`可以帮助你隐藏一些逻辑.
    在往下走之前,要了解what's going on?这是非常重要的,也就是为什么你会在这里.**
</br>
关于错误的一些小东西
----
因为这个`request`对象是一个可写流,因此当报错时，它也会有`eventEmitter`的处理错误的行为.

在`request`流中的错误是通过在这个流中的`error`事件的时间响应而显示的.如果你没有一个监听这个事件的监听者,那么这个错误将会抛出,当抛出时会使你的node.js程序崩溃.因此你应该添加一个`error`监听者在你的请求流中,即使你只是打印它并且继续做你的.(虽然最好还是发送某种类型的http错误回应,但稍后再说)。

```javascript
request.on('error', function(err) {
  // This prints the error message and stack trace to `stderr`.
  console.error(err.stack);
});
```
这里有其他方法去[处理这些错误](https://nodejs.org/api/errors.html)例如其他抽象概念或者工具,但是总是要意识到错误确实是会发生,而你要不得不处理他们.


----
