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
                  **注意:这`request`对象是`IncomingMessage`的实例**</br>
这个`method`在这里总是一个普通的HTTP方法/动词.这个`url`没有了服务器名字,协议或者端口的完整url,对于通常的url来说,意思是包括第三个正斜杠以及之后的所有内容.

请求头也是。他们在`request`对象上有自己对象，叫做header.
```javascript
var headers = request.headers;
var userAgent = headers['user-agent'];
```



