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

说了这么多
----
在这个点上,我们已经知道了创建一个服务器,获取请求方法,请求方法,请求头和请求体.当我们在这些都放在一起,它可能会长这样.
```javascript
var http = require('http');

http.createServer(function(request, response) {
  var headers = request.headers;
  var method = request.method;
  var url = request.url;
  var body = [];
  request.on('error', function(err) {
    console.error(err);
  }).on('data', function(chunk) {
    body.push(chunk);
  }).on('end', function() {
    body = Buffer.concat(body).toString();
    // At this point, we have the headers, method, url and body, and can now
    // do whatever we need to in order to respond to this request.
  });
}).listen(8080); // Activates this server, listening on port 8080.
```

如果我们运行这代码,我们将有能力去接收请求,但无法响应他们.事实上,如果你用浏览器上8080端口,你的请求将会time out,因为没有东西被发送给客户端.

http 状态码
-----

如果你不去设置它,这响应的http状态码将永远是200.当然,不是每一个http响应都保证这一点,并且在某一情况你绝对会想发送一个不同的状态码.为了做到这个目的,你可以设置`statusCode`属性

```javascript
response.statusCode = 404; // Tell the client that the resource wasn't found.
```
这里还有一些别的捷径,稍后再说.

设置响应头
----

响应头都是通过一个简便的方法叫做`setHeader`设置

```javascript
response.setHeader('Content-Type', 'application/json');
response.setHeader('X-Powered-By', 'bacon');
```

当在response上设置请求头,要注意的情况是他们的名字.如果你重复设置同一个响应头,最后那个值是最终值.



发送响应体
-----

因为`response`对象是一个可写流,将响应体写入客户端只是一个平常的流方法使用.

```javascript
response.write('<html>');
response.write('<body>');
response.write('<h1>Hello, World!</h1>');
response.write('</body>');
response.write('</html>');
response.end();
```

可写流的`end`方法也可以采用一些可选数据作为最后的流数据去发送,因此我们可以简化上述示例如下.

```javascript
response.end('<html><body><h1>Hello, World!</h1></body></html>');
```
      **注意: 在你开始写数据到响应体前先设置响应头与状态码是非常重要的，因为在http响应里是响应头比响应体先出现的,所以很重要.**
 </br>
 另一个关于错误的quick thing
 ----
 `response`可写流也可以响应`error`事件,而且在某个时候你也将会需要处理它。所有对于`request`流的错误的建议也都适用于这里.
 
总结一起
----

现在我们学习了关于http响应,我们将他它在一起.在之前的例子上构建,将创建一个服务器可以发送回用户发送给我们的数据.我们用JSON.stringify将格式化那数据成为JSON.

```javascript
var http = require('http');

http.createServer(function(request, response) {
  var headers = request.headers;
  var method = request.method;
  var url = request.url;
  var body = [];
  request.on('error', function(err) {
    console.error(err);
  }).on('data', function(chunk) {
    body.push(chunk);
  }).on('end', function() {
    body = Buffer.concat(body).toString();
    // BEGINNING OF NEW STUFF

    response.on('error', function(err) {
      console.error(err);
    });

    response.statusCode = 200;
    response.setHeader('Content-Type', 'application/json');
    // Note: the 2 lines above could be replaced with this next one:
    // response.writeHead(200, {'Content-Type': 'application/json'})

    var responseBody = {
      headers: headers,
      method: method,
      url: url,
      body: body
    };

    response.write(JSON.stringify(responseBody));
    response.end();
    // Note: the 2 lines above could be replaced with this next one:
    // response.end(JSON.stringify(responseBody))

    // END OF NEW STUFF
  });
}).listen(8080);
```

回声服务器例子
----

来简化之前的例子来建造一个简单的回声服务器,它是可以发送任何从请求中接受的数据到响应中。我们只需要从请求可读流中抓取数据并写入数据到响应可写流,类似我们之前做的那种.

```javascript
var http = require('http');

http.createServer(function(request, response) {
  var body = [];
  request.on('data', function(chunk) {
    body.push(chunk);
  }).on('end', function() {
    body = Buffer.concat(body).toString();
    response.end(body);
  });
}).listen(8080);
```

现在我们来调整一下,我们只想在以下情况下发送回声:

* 请求方法是GET
* url是/echo

在任何别的情况下,我们想要简单回应404.

```javascript
var http = require('http');

http.createServer(function(request, response) {
  if (request.method === 'GET' && request.url === '/echo') {
    var body = [];
    request.on('data', function(chunk) {
      body.push(chunk);
    }).on('end', function() {
      body = Buffer.concat(body).toString();
      response.end(body);
    })
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

    **注意: 用这种方式去检查URL，我们其实在做路由选择的一种格式路由选择.其他格式的路由选择可以简单的像`switch`或者复杂的像一整个框架例如｀express｀** 
    **如果你在寻找只可以做路由选择的东西,可以尝试router.**
</br>

很好！现在让我们简化一下.记住,`request`对象是一个`可写流`然后`response`对象是一个`可写流`.这意外着我们可以使用`pipe`去导向数据从一个到另一个.等于完全就是我们想要的回声服务器！

```javascript
var http = require('http');

http.createServer(function(request, response) {
  if (request.method === 'GET' && request.url === '/echo') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```
耶！Stream

我们




