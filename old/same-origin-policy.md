# 同源策略那些事儿

> **Tips**
在本文中，`域`和`源`指代的都是 *origin*，换着讲纯粹是出于通用的习惯。另外，出于学习的目的且为了避免浏览器的差异导致的麻烦，请在 Chrome 下运行以下所有客户端的代码。

## 重现

我们先来写个用 ajax 提交表单的小小小的 demo，这毕竟太常见了。

**/Test/index.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>index</title>
  </head>
  <body>
    <script>
    const xhr = new XMLHttpRequest();

    xhr.open('post', 'http://127.0.0.1/Test/index.php', true);

    xhr.onreadystatechange = () => {
      if(xhr.readyState === 4 && xhr.status === 200) {
        document.write(xhr.responseText);
      }
    };

    xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    xhr.send('username=yang');
    </script>
  </body>
</html>
```

**/Test/index.php**

```php
<?php
echo $_POST['username'];
?>
```
> **Tips**
请将上述代码放在本地服务器中运行。

不出意外，你会得到以下大礼包：

```bash
XMLHttpRequest cannot load http://127.0.0.1/Test/index.php. No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost' is therefore not allowed access.
```

此刻，有的人微微一笑，有的人一脸懵逼。

## 怎么回事？

之所以会出现这个问题，是因为浏览器有同源策略 (*Same-origin policy*) 的限制，一个域 (*origin*) 的脚本，在未经允许的情况下，不得通过 [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model) **读取**另一个域的文档 (*document*) 的内容或属性。

同源策略在 Web 应用安全中扮演着重要的角色，它能保护一个网站的敏感信息，防止恶意脚本的窃取。同源策略中的`同源`，指的是`协议`、`host`、`端口`相同。同源下的文档内容及属性可以共享，不同源下的文档内容及属性在未经允许时不可以直接获取。

> **Tips**
同源中的`host`可以为主机名 (*hostname*) 或 IP 地址。

如果有一个地址为：`http://example.com`，则：

```bash
http://example.com/test/a.html # 同源
https://example.com # 不同源，协议不同
http://www.example.com # 不同源，host 不同
http://example.com:8080 # 不同源，端口不同
```

## 规避同源策略的方法

然而我们有些时候是需要在不同源的地址间进行通信的，有以下的方法可以用来规避同源策略。

### **document.domain**

* 适用情况：不同的窗口 (*window*) 或 内敛框架元素 \<iframe\> 之间互相访问文档内容，包括 cookie
* 实现前提：
    - 协议、端口相同
    - 当前域名 (*domain name*) 的父域 (*superdomain*) 相同
    - 其中一个窗口或 iframe 能得到另一个窗口或 iframe 的引用

> **Tips**
如果两个地址为`one.example.com`和`two.example.com`，则它们的父域为`example.com`。注意上面所述父域不能为顶级域名 (*top-level domain*)，或者说不能为一级域名 (*first-level domain*)。关于域名分级，详见[**domain name space**](https://en.wikipedia.org/wiki/Domain_name#Domain_name_space)。

无码言*。我们来试试。

**/Test/main.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Main window</title>
  </head>
  <body>
    <p>This is the main window</p>
    <iframe src="http://w3.w1.localhost/Test/iframe.html" width="300px" height="250px" id="child-iframe" name="my"></iframe>
    <script>
      document.domain = 'w1.localhost';
    </script>
  </body>
</html>
```

**/Test/iframe.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Iframe window</title>
  </head>
  <body>
    <p>This is the iframe window</p>
    <button id="btn">Close this iframe</button>
    <script>
    document.domain = 'w1.localhost';
    const btn = document.querySelector('#btn');
    const parent = window.parent.document;
    const frame = parent.querySelector('#child-iframe');

    btn.onclick = () => {
      parent.body.removeChild(frame);
    };
    </script>
  </body>
</html>
```

> **Tips**
跨域通信中，有些窗口属性是允许跨域访问的，详见[这里](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Window)。

> **Tips**
注意`localhost`是预留的顶级域名，不可以把它设置为`document.domain`。

访问`http://w2.w1.localhost/Test/main.html`，点击按钮，iframe 窗口消失了，再注释掉任何一个 html 文件的 `document.domain = 'w1.localhost'`看看，emmmm..

另外，我们可以通过在服务器中如下设置`Set-Cookie`头来实现在多个拥有相同父域的子域名间共享 cookie。

```php
<?php
# 假设发起请求的域与此域是同域

# 指定了域名的话就相当于包含了所有子域名，所有子域名和此父域名都可以共享 cookie
setcookie('username', 'Sam Yang', time() + 24 * 3600, '/', "w1.localhost");
# setcookie('user', 'Sam Yang', time() + 24 * 3600, '/'); // 不指定则默认当前域名，cookie 不可被子域名共享
?>
```

然后你在客户端的所有子域名下的页面都可以通过`document.cookie`获得共享的 cookie。

### 跨文档消息传递 (*cross-document messaging*)

* 适用情况：不同的窗口 (*window*) 或 内敛框架元素 \<iframe\> 之间跨域传递数据
* 实现前提：浏览器支持 HTML5 的`window.postMessage`

通过 HTML5 提供的`window.postMessage`方法，可以让一个页面的脚本，传递数据给另一个页面的脚本，而无需理会脚本所在的页面是否跨域。

这个方法的主要语法是这样的：

**otherWindow.postMessage(message, targetOrigin)**

* **otherWindow** - 另一个窗口的引用，换言之，是接收数据的窗口的引用
* **message** - 发送给其他窗口的数据，数据在发送前会被自动序列化，而且数据类型不受限制
* **targetOrigin** - `otherWindow`的源 (*origin*)，可以是字符串`*`(可以发送给任何源)或一个 URI，注意只有当这里指定的`targetOrigin`的值和想要接收数据的窗口的源 (*origin*) 完全匹配，`window.postMessage`触发的事件才会被发送

接收数据的窗口可以监听`message`事件，这个事件接收到的数据参数包含三个重要属性：

* **data** - 发送数据的窗口传来的数据对象
* **origin** - 发送数据的窗口的源 (*origin*)
* **source** - 发送数据的窗口的引用，可以用这个属性建立两个非同源窗口间的双向通信

我们来用一下：

**/Test/main.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Main window</title>
  </head>
  <body>
    <p>This is the main window</p>
    <iframe src="http://w3.w1.localhost/Test/iframe.html" width="300px" height="250px" id="child-iframe" name="my"></iframe>
    <script>
      const iframe = document.querySelector('#child-iframe');
      const iframeWin = iframe.contentWindow; // 获得 iframe 元素的窗口

      iframe.onload = () => { // 等待 iframe 窗口完全加载完再发送消息
        iframeWin.postMessage('hello, my friend', 'http://w3.w1.localhost');
      };
    </script>
  </body>
</html>
```

**/Test/iframe.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Iframe window</title>
  </head>
  <body>
    <p>This is the iframe window</p>
    <script>
    window.onmessage = (event) => {
      if(event.origin !== 'http://w2.w1.localhost') return;

      document.write(event.data);
    };
    </script>
  </body>
</html>
```

访问`http://w2.w1.localhost/Test/main.html`，可以看到子窗口已经接收到了父窗口传来的数据，你还可以尝试传输其他更复杂的数据形式。

> **Tips**
这个方法虽然很棒，但是需要注意以下几点安全问题，否则你的站点可能被爆得体无完肤。
> 
> * 不希望从其他站点接收数据时，不要设置任何`message`事件的监听器
> * 希望从其他站点接收数据时，接收方务必使用`origin`属性 (有必要的话再加上`source`属性) 验证发送者的身份，避免恶意代码的攻击，可保平安
> * 发送方应总是指定精确的接收方源，即`targetOrigin`属性，而不是`*`，因为后者可以让猥琐的站点恶意改变你发送的数据的相关属性进而拦截你的数据

### JSONP (*JSON with padding*)

* 适用情况：客户端与服务器端的跨域通信
* 实现前提：无

> **Tips**
由于内在的风险，JSONP 正逐渐被 CORS (见下文) 取代，此处仅出于了解的目的讲解此技巧，你可以选择跳过这个部分。

直译这个东东，就是“填充的 JSON”，这是跟 [Ajax](https://en.wikipedia.org/wiki/Ajax_(programming)) 一样的老爷爷了，不同的是，前者要退休了。

这项技术之所以出现，是因为`<script src="..."></script>`标签不受同源策略的限制。利用`<script>`元素，页面可以向服务器端请求 JSON 数据，服务器端收到请求后，将 JSON 数据传传入一个指定名字的回调函数里再传回给客户端，这种使用模式就是所谓的 JSONP。

**/Test/index.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>index</title>
  </head>
  <body>
    <script>
    function addScriptTag(src) {
      const script = document.createElement('script');
      script.src = src;
      document.body.appendChild(script);
    }

    window.onload = () => {
      addScriptTag('http://127.0.0.1/Test/index.php?callback=sayHello'); // 这里的 `callback` 换成其他名字也可以
    };
    
    function sayHello(data) { // 浏览器会自动解析得到的 JSON 数据，无需手动解析
      document.body.append(`Hello, ${data.username}`);
    }
    </script>
  </body>
</html>
```

**/Test/index.php**

```php
<?php
$callback = $_GET['callback'];
$data = array(
  'username' => 'samyang'
);
$result = json_encode($data);

echo "{$callback}({$result})";
?>
```

访问`http://localhost/Test/index.html`。

当然啦，JSONP 是很容易遭到[跨站请求伪造](https://en.wikipedia.org/wiki/Cross-site_request_forgery)攻击的，所以你懂的。

### WebSocket

* 适用情况：实时通信 (无论是否跨域)
* 实现前提：浏览器和服务器支持 HTML5 的 WebSocket

WebSocket 是一种基于 `ws(非加密) / wss(加密)` 协议的技术，使用这种技术可以建立客户端和服务器端双向且持续的通信连接，并且这不受同源策略限制。但是，一旦你使用了 WebSocket 的 URI，请求头中就会加入`Origin字段，指明请求连接的脚本所在的源 (*origin*)，服务器用这个字段来检验跨站请求是否安全。

这项技术仍然不稳定，但有一些成熟的相应实现，如 [Socket.IO](https://socket.io/)，有兴趣可以参见我写的 Socket.IO 的[教程](https://yangkean.com/blog/2017/8/socket-io.html)。

这项技术主要用于实时通信，此处不做进一步详述，想进一步探索，见 [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)。

### 跨域资源共享 (*Cross-Origin Resource Sharing, CORS*)

好了，请出我们的大 boss。与 JSONP 的只支持 GET 请求相比，CORS 支持所有类型的 HTTP 请求。

* 适用情况：客户端与服务器端的跨域通信
* 实现前提：客户端和服务器端同时支持 CORS

ajax 受到同源策略的限制，使用 ajax 技术时，在未经允许的情况下，如果跨域请求发出给了服务器端并返回了数据 (视浏览器情况而定，有的在发出时即拦截)，则客户端无法读取服务器端返回的数据。CORS 允许服务器端进行跨域访问控制，从而使跨域数据传输得以安全进行。

我们首先来了解下为了支持 CORS，增加了哪些 HTTP 首部字段。

#### HTTP 请求首部字段

> **Tips**
这些请求字段无需手动设置，当你使用 XMLHttpRequest 发起跨域请求时，浏览器已自动帮你设置好了它们。

##### Origin

```bash
Origin: <origin>
```

表明发起请求的源 (*origin*)，这是一个 URI，不包含任何路径信息，只是服务器的名字。

##### Access-Control-Request-Method

```bash
Access-Control-Request-Method: <method>
```

这个字段用于预检请求 (下文会讲)，其作用是将实际请求所使用的 HTTP 方法告诉服务器。

##### Access-Control-Request-Headers

```bash
Access-Control-Request-Headers: <field-name>[, <field-name>]*
```

这个字段用于预检请求，其作用是将实际请求所携带的自定义首部字段告诉服务器。

#### HTTP 响应首部字段

##### Access-Control-Allow-Origin

```bash
Access-Control-Allow-Origin: <origin> | *
```

指定可以访问当前资源的外源 (*origin*)，可以为不含路径信息的一个 URI，也可以是通配符`*`。后者表示当前资源可以被任何外源访问，即允许来自所有域的请求。**注意，当请求携带有身份凭证时 (下文讲)，服务器端不可以指定该值为通配符**。

##### Access-Control-Expose-Headers

用 XMLHttpRequest 对象的 getResponseHeader() 方法可以获取一些基本的响应头，当想要获取一些额外的响应头时，可以用这个字段指定。

##### Access-Control-Max-Age

```bash
Access-Control-Max-Age: <delta-seconds>
```

指定预检请求的结果能被缓存多少秒。在这个缓存时间内，浏览器无须为同一请求再次发起预检请求。

##### Access-Control-Allow-Methods

```bash
Access-Control-Allow-Methods: <method>[, <method>]*
```

这个字段用于预检请求响应，其指明了实际请求时所允许使用的 HTTP 方法。

##### Access-Control-Allow-Headers

```bash
Access-Control-Allow-Headers: <field-name>[, <field-name>]*
```

这个字段用于预检请求响应，其指明了实际请求时允许携带的自定义首部字段。

##### Access-Control-Allow-Credentials

讲这个字段前，我们先讲下 XMLHttpRequest 对象的 [withCredentials](https://xhr.spec.whatwg.org/#the-withcredentials-attribute) 属性来用，withCredentials 设置为`true`时，[身份凭证](https://xhr.spec.whatwg.org/#user-credentials) (cookies、HTTP 认证信息等) 就会被浏览器包含在跨域请求中，设置为`false`则排除在跨域请求之外并且浏览器忽略响应中设置 cookie 的字段，这个属性默认为`false`，也就是说一般而言，在跨域请求中，浏览器不会发送身份凭证信息。**注意，在同源请求中，设置这个属性没有任何影响**。

Access-Control-Allow-Credentials 字段指定了当客户端设置了 withCredentials 为`true`时是否允许浏览器读取响应体 (*response*) 的内容，为`true`时表示允许。当此字段作为预检请求中的响应头的一部分时，它指示的是实际请求是否可以使用身份凭证 (*credentials*)。

**当允许携带身份凭证时，请求头中包含了可能含有隐私信息的 cookie 数据，所以服务器端不得设置 Access-Control-Allow-Origin 的值为`*`**，只能设置为准确的域 (*origin*)。

> **Tips**
虽然 CORS 允许跨域请求，但是 cookie 仍然受限于浏览器的同源策略，这意味着除了使用前文所讲的`document.domain`方法外，只有来自同一个源的页面可以读写这个源的 cookie，你无法通过 JavaScript 读写跨域的 cookie。设置`withCredentials`为`true`只能让你把跨域请求的那个服务器端设置在客户端的 cookie 发送回给服务器端，不能让你把客户端设置的 cookie 发送给服务器端。详情见[这里](https://stackoverflow.com/questions/14462423/cross-domain-post-request-is-not-sending-cookie-ajax-jquery)。

>**Tips**
当服务器端设置`Set-Cookie`字段时，如果设置的域名 (*domain*) 不包含服务器的地址，那么设置的这个 cookie 就会被用户代理拒绝保存。比如说你服务器端的地址为`http://w1.localhost`，下面设置的 cookie 就会被用户代理拒绝保存：
> ```php
> setcookie('user', 'Sam Yang', time() + 24 * 3600, '/', "w2.localhost");
> ```
> 详情见 [Invalid domains](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#Invalid_domains)。

#### 简单请求 (*simple request*)

不会触发 CORS 预检请求 (下文会提到) 的请求即为简单请求，具体来说，就是同时满足下列条件的请求：

* 使用下列请求方法之一：
    * GET
    * HEAD
    * POST
* 除了用户代理自动设置的头部字段，只允许手动设置以下集合中的首部字段：
    * Accept
    * Accept-Language
    * Content-Language
    * Content-Type (值为下面三种之一)
        * text/plain
        * multipart/form-data
        * application/x-www-form-urlencoded
    * DPR
    * Downlink
    * Save-Data
    * Viewport-Width
    * Width
* 请求时没有在任何 [XMLHttpRequestUpload](https://xhr.spec.whatwg.org/#the-upload-attribute) 对象上注册事件监听器
* 没有在请求中使用 [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) 对象

**/Test/index.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>index</title>
  </head>
  <body>
    <script>
    const xhr = new XMLHttpRequest();
    xhr.open('get', 'http://w1.localhost/Test/index.php', true);
    xhr.send();
    </script>
  </body>
</html>
```

**/Test/index.php**

```php
<?php
header('Access-Control-Allow-Origin: *');
?>
```

访问`http://w2.localhost/Test/main.html`，下面分别是请求头和响应头：

```bash
# 请求头
GET /Test/index.php HTTP/1.1
Host: w1.localhost
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Origin: http://w2.localhost # 注意这个字段
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36
Accept: */*
DNT: 1
Referer: http://w2.localhost/Test/main.html
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,ja;q=0.4,zh-TW;q=0.2
HT-Ver: 1.1.2
HT-Sid: KtctXse5-mUErYXtS-aUEwI5XF-NOSNsGw+-vH+sk2L4-8852iahF-tMQulEKm-0Hvpoi9J

# 响应头
HTTP/1.1 200 OK
Date: ****** # 已打码
Server: Apache/2.4.25 (Unix) PHP/5.6.30
X-Powered-By: PHP/5.6.30
Access-Control-Allow-Origin: * # 注意这个字段
Content-Length: 0
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

根据上述条件，这是一个简单请求，我们使用 `Origin` 和 `Access-Control-Allow-Origin` 就能完成最简单的访问控制。

#### 预检请求 (*preflight request*) 和 实际请求 (*actual request*)

为了避免那些可能对服务器数据产生副作用的 HTTP 请求方法，浏览器必须首先先使用 OPTIONS 方法发起一个预检请求，从而获知服务器端是否允许该跨域请求，获得允许之后才发起实际请求。满足下述任一条件时，即应首先发送预检请求：

* 使用下列请求方法之一
    * PUT
    * DELETE
    * CONNECT
    * OPTIONS
    * TRACE
    * PATCH
* 除了用户代理自动设置的头部字段，手动设置了以下集合 **之外** 的首部字段：
    * Accept
    * Accept-Language
    * Content-Language
    * Content-Type (值为下面三种之一)
        * text/plain
        * multipart/form-data
        * application/x-www-form-urlencoded
    * DPR
    * Downlink
    * Save-Data
    * Viewport-Width
    * Width
* 请求时在一个 [XMLHttpRequestUpload](https://xhr.spec.whatwg.org/#the-upload-attribute) 对象上注册了一个至多个事件监听器
* 在请求中使用 [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) 对象

**/Test/index.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>index</title>
  </head>
  <body>
    <script>
    const xhr = new XMLHttpRequest();
    const body = '<?xml version="1.0"?><person><name>Arun</name></person>';

    xhr.open('post', 'http://w1.localhost/Test/index.php', true);
    xhr.setRequestHeader('X-PINGOTHER', 'pingpong');
    xhr.setRequestHeader('Content-Type', 'application/xml');

    xhr.send(body);
    </script>
  </body>
</html>
```

**/Test/index.php**

```php
<?php
header('Access-Control-Allow-Origin: http://w2.localhost');
header('Access-Control-Allow-Headers: X-PINGOTHER, Content-Type');
header('Access-Control-Allow-Methods: POST, GET, OPTIONS');
header('Access-Control-Max-Age: 86400');
?>
```

访问`http://w2.localhost/Test/main.html`，下面分别是预检请求头和响应头：

```bash
# 请求头
OPTIONS /Test/index.php HTTP/1.1
Host: w1.localhost
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Access-Control-Request-Method: POST # 关注它
Origin: http://w2.localhost # 关注它
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36
Access-Control-Request-Headers: content-type,x-pingother # 关注它
Accept: */*
DNT: 1
Referer: http://w2.localhost/Test/main.html
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,ja;q=0.4,zh-TW;q=0.2
HT-Ver: 1.1.2
HT-Sid: KtctXse5-mUErYXtS-aUEwI5XF-NOSNsGw+-vH+sk2L4-8852iahF-tMQulEKm-0Hvpoi9J

# 响应头
HTTP/1.1 200 OK
Date: ****** # 已打码
Server: Apache/2.4.25 (Unix) PHP/5.6.30
X-Powered-By: PHP/5.6.30
Access-Control-Allow-Origin: http://w2.localhost # 关注它
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type # 关注它
Access-Control-Allow-Methods: POST, GET, OPTIONS # 关注它
Access-Control-Max-Age: 86400 # 关注它
Content-Length: 0
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

预检请求之后的实际请求头和响应头：

```bash
# 请求头
POST /Test/index.php HTTP/1.1
Host: w1.localhost
Connection: keep-alive
Content-Length: 55
Pragma: no-cache
Cache-Control: no-cache
X-PINGOTHER: pingpong # 关注它
Origin: http://w2.localhost
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36
Content-Type: application/xml # 关注它
Accept: */*
DNT: 1
Referer: http://w2.localhost/Test/main.html
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,ja;q=0.4,zh-TW;q=0.2
HT-Ver: 1.1.2
HT-Sid: KtctXse5-mUErYXtS-aUEwI5XF-NOSNsGw+-vH+sk2L4-8852iahF-tMQulEKm-0Hvpoi9J

# 响应头
HTTP/1.1 200 OK
Date: ****** # 已打码
Server: Apache/2.4.25 (Unix) PHP/5.6.30
X-Powered-By: PHP/5.6.30
Access-Control-Allow-Origin: http://w2.localhost
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Max-Age: 86400
Content-Length: 0
Keep-Alive: timeout=5, max=99
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

### 一些小伎俩

之所以叫小伎俩，是因为这里所讲的方法都只是在小数据量的跨域通信中比较方便，大数据量则不宜使用。

* 适用情况：小数据量的跨域通信

> **Tips**
如果对 URL 的结构不甚清晰，可以参见 [URL](https://en.wikipedia.org/wiki/URL#Syntax)。

#### 片段标识符 (*fragment identifier*)

* 实现前提：无。

片段标识符是网址 URL 中`#`符后面的部分。我们可以这样来使用之。

**/Test/main.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Main window</title>
  </head>
  <body>
    <p>This is the main window</p>
    <iframe src="http://w3.w1.localhost/Test/iframe.html" width="300px" height="250px" id="child-iframe" name="my"></iframe>
  </body>
</html>
```

**/Test/iframe.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Iframe window</title>
  </head>
  <body>
    <p>This is the iframe window</p>
    <script>
    window.onhashchange = () => {
      document.write(`hello, ${window.location.hash.substr(1)}`);
    };
    </script>
  </body>
</html>
```

访问`http://w2.w1.localhost/Test/main.html`，打开控制台，输入：

```js
document.querySelector('#child-iframe').src += '#samyang'
```

emmmm... 小窗口内容变了！

但是呢，这种方法是比较鸡肋的，虽然`#`有其他的用途 (见 [这里](http://www.ruanyifeng.com/blog/2011/03/url_hash.html))，但它一般是用于定位页面中某个部分的，如果页面中有一些标签含有相同的片段标识符，便会产生一些意料之外的行为。再者，这种方法传输的数据量太少了。

#### 查询字符串 (*search string*, aka, *query string*)

* 实现前提：无。

这是另一种用于小数据量跨域通信的方法，也是我所常用的方法，也是相比于前者更推荐的做法。查询字符串是 URL 中`?`及其后面但不包括`#`及片段标识符的部分。

**/Test/main.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Main window</title>
  </head>
  <body>
    <p>This is the main window</p>
    <button>Click me</button>
    <script>
      document.querySelector('button').onclick = () => {
        location.href = `http://w3.w1.localhost/Test/iframe.html?username=samyang`;
      };
    </script>
  </body>
</html>
```

**/Test/iframe.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Iframe window</title>
  </head>
  <body>
    <p>This is the iframe window</p>
    <button id="btn">Close this iframe</button>
    <script>
    window.onload = () => {
      document.write(location.search.substr(1))
    };
    </script>
  </body>
</html>
```

访问`http://w2.w1.localhost/Test/main.html`，点击按钮。

#### **window.name**

* 实现前提：跨域页面在同一个窗口中。

这个属性是属于窗口的，只要窗口不变，即使页面变为不同域，这个属性的值也不变。

**/Test/main.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Main window</title>
  </head>
  <body>
    <p>This is the main window</p>
    <script>
      window.name = 'samyang';
      location.href = 'https://www.sogou.com/';
    </script>
  </body>
</html>
```

访问`http://w2.w1.localhost/Test/main.html`，窗口跳到新页面后打开控制台输入`window.name`并回车可以看到原页面设置的值。

## 结尾

好了，大概是这么多了(其实还有很多)，如有疏漏请指出。

## Reference

* [Same-origin policy wikipedia](https://en.wikipedia.org/wiki/Same-origin_policy)
* [Same-origin policy MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
* [Window.postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
* [JSONP wikipedia](https://en.wikipedia.org/wiki/JSONP)
* [Uniform Resource Identifier](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)
* [Cross-Origin Resource Sharing W3C](https://www.w3.org/TR/cors/)
* [HTTP access control (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)
* [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)
* [浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
* [「JavaScript」JS四种跨域方式详解](https://segmentfault.com/a/1190000003642057)
