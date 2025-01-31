#HTTP/1.1

HTTP/1.1是应用相当广泛的协议，从1997年发布至今，在众多的互联网服务中仍能频繁看到它的身影。

HTTP/1.1 同时也是一个非常简单的文本协议，使用CRLF+空白行进行描述，响应报文基本上由协议版本，状态码（表示请求成功与失败的数字代码），用以解释状态码的原因短语，可选的响应首部字段以及实体主体构成， 也正式因为HTTP相对简单的结构造成HTTP非常被容易攻击，如CRLF注入等等漏洞。

<div  align="center">
	<p>图：HTTP返回结构</p>
	<img src="/assets/chapter2/http-response.png" width = "550"  align=center />
</div>

### HTTP状态码

服务器返回的 响应报文 中第一行为状态行，包含了状态码以及原因短语，用来告知客户端请求的结果。对于以下的状态码我们通常只要知道 4xx为客户端出错、5xx为服务端出错即能排查出绝大部分的问题。

|状态码	| 类别| 原因|
|:---|:---|:---|
|1xx|Informational（信息性状态码）	|接收的请求正在处理|
|2xx|Success（成功状态码）	|请求正常处理完毕|
|3xx|Redirection（重定向状态码）|需要进行附加操作以完成请求|
|4xx|Client Error（客户端错误状态码）|服务器无法处理请求|
|5xx|Server Error（服务器错误状态码）|服务器处理请求出错|

### HTTP/1.1 的性能优化

相较之前的版本HTTP/1.1最大的改进是复用连接，在一个TCP连接上可以串行多次HTTP请求，这个机制减少频繁建立连接释放连接的系统开销，极大改善了HTTP连接性能。


<div  align="center">
	<p>图：HTTP的连接复用机制</p>
	<img src="/assets/chapter2/http-ka.png" width = "280"  align=center />
</div>


#### 场景优化

现在Web类的服务发展已有非常明显的趋势：

* 消息变大 ：从几 KB 大小的消息，到几 MB 大小的消息
* 页面资源变多 ：从每个页面不到 10 个的资源，到每页超 100 多个资源
* 内容形式变多样 ：从单纯到文本内容，到图片、视频、音频等内容;
* 实时性要求变高 ：对页面的实时性要求的应用越来越多

HTTP/1.1虽然复用了TCP连接，减小了握手环节的消耗，但在应对以上的需求发展，还是明显存在很多不足：
队头阻塞 (Head-Of-Line Blocking)问题导致高延迟、浏览器针对域名的并发限制、ASCII形式的文本报文（导致无法分割），重复请求得HTTP Header头信息等等

<div  align="center">
	<p>图：HTTP 队头阻塞导致无法充分利用带宽</p>
	<img src="/assets/chapter2/http-rtt.jpeg" width = "550"  align=center />
</div>


为了解决HTTP性能和延迟等问题，针对Web类的场景应用，也产生了各类的优化方法（总体的思想还是减小请求，避免队头阻塞）

**无论何种基于HTTP/1.1的优化，都是修修补补，无法解决本质问题。想要根本性的提升服务质量，还是建议升级HTTP协议版本， 在您无法进行升级的时候，也可以了解以下的几种修补改善方式**


**图片合并 Spriting**

该策略指的是将多个小图片合并到一张大的图片里，这样多个小的请求就被合并成了一个大的图片请求，然后再利用js或者css文件来取出其中的小张图片使用。好处显而易见，请求数减少，延迟自然低。坏处是文件的粒度变大了，让图片合成处理等变得更复杂。 

**内容内嵌 Inlining**

Inlining的思考角度和spriting类似，是将额外的数据请求通过base64编码之后内嵌到一个总的文件当中。比如一个网页有一张背景图，我们可以通过如下代码嵌入：

background: url(data:image/png;base64,)

data部分是base64编码之后的字节码，这样也避免了一次多余的http请求。

**文件合并 Concatenation**

Concatenation主要是针对js、css这类文件，现在前端开发交互越来越多，零散的js、css文件也在变多。
将多个静态文件合并到一个大的文件里在做一些压缩处理也可以减小延迟和传输的数据量


**域名分片 Domain Sharding**

浏览器或者客户端是根据domain（域名）来建立连接的。比如针对www.iq.com只允许同时建立2个连接，但static.iq.com被认为是另一个域名，可以再建立两个新的连接。依次类推，如果我再多建立几个子域名，那么同时可以建立的http请求就会更多，这就是Domain Sharding了。

连接数变多之后，受限制的请求就不需要等待前面的请求完成才能发出了。这个技巧被大量的使用，一个颇具规模的网页请求数可以超过100，使用domain sharding之后同时建立的连接数可以多到50个甚至更多。

domain sharding还有一大好处，对于资源文件来说一般是不需要cookie的，将这些不同的静态资源文件分散在不同的域名服务器上，可以减小请求的size。