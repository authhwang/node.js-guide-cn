一个http会话的解剖
====

这个指南的目的是传授一个对node.js中http处理过程更准确的理解.我们将假设你大致了解http请求的工作原理，无论是什么语言或者编程环境下。我们也将估计你会熟悉一点关于node.js的<code>EventEmitters</code>和<code>Streams</code>.如果你还没很熟悉，那你需要去通过api文档快速浏览一下他们.

创建服务器
-----
