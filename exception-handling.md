# 异常的处理和 HTTP 状态码的分类
  
先来说一说 HTTP 的异常处理问题。  

当 urlopen 不能够处理一个 response 时，产生 urlError。不过通常的 Python APIs 异常如 ValueError，TypeError 等也会同时产生。HTTPError 是 urlError 的子类，通常在特定 HTTP URLs 中产生。
 
## URLError

通常，URLError 在没有网络连接(没有路由到特定服务器)，或者服务器不存在的情况下产生。这种情况下，异常同样会带有"reason"属性，它是一个 tuple（可以理解为不可变的数组），包含了一个错误号和一个错误信息。

我们建一个 urllib2_test06.py 来感受一下异常的处理：

```
import urllib2  
  
req = urllib2.Request('http://www.baibai.com')  
  
try: urllib2.urlopen(req)  
  
except urllib2.URLError, e:    
    print e.reason  
```

按下 F5，可以看到打印出来的内容是：

```
[Errno 11001] getaddrinfo failed
```

也就是说，错误号是 11001，内容是 getaddrinfo failed。

## HTTPError

服务器上每一个 HTTP 应答对象 response 包含一个数字"状态码"。有时状态码指出服务器无法完成请求。默认的处理器会为你处理一部分这种应答。

例如:response 是一个"重定向"，需要客户端从别的地址获取文档，urllib2 将为你处理。其他不能处理的，urlopen 会产生一个 HTTPError。典型的错误包含"404"(页面无法找到)，"403"(请求禁止)，和"401"(带验证请求)。HTTP 状态码表示 HTTP 协议所返回的响应的状态。比如客户端向服务器发送请求，如果成功地获得请求的资源，则返回的状态码为 200，表示响应成功。如果请求的资源不存在，则通常返回 404 错误。 HTTP 状态码通常分为5种类型，分别以 1～5 五个数字开头，由 3 位整数组成：

```
200：请求成功      处理方式：获得响应的内容，进行处理   
201：请求完成，结果是创建了新资源。新创建资源的 URI 可在响应的实体中得到    处理方式：爬虫中不会遇到   
202：请求被接受，但处理尚未完成    处理方式：阻塞等待   
204：服务器端已经实现了请求，但是没有返回新的信 息。如果客户是用户代理，则无须为此更新自身的文档视图。    处理方式：丢弃  
300：该状态码不被 HTTP/1.0 的应用程序直接使用， 只是作为 3XX 类型回应的默认解释。存在多个可用的被请求资源。    处理方式：若程序中能够处理，则进行进一步处理，如果程序中不能处理，则丢弃  
301：请求到的资源都会分配一个永久的 URL，这样就可以在将来通过该 URL 来访问此资源    处理方式：重定向到分配的 URL    
302：请求到的资源在一个不同的 URL 处临时保存     处理方式：重定向到临时的 URL   
304 请求的资源未更新     处理方式：丢弃   
400 非法请求     处理方式：丢弃   
401 未授权     处理方式：丢弃   
403 禁止     处理方式：丢弃   
404 没有找到     处理方式：丢弃   
5XX 回应代码以“5”开头的状态码表示服务器端发现自己出现错误，不能继续执行请求    处理方式：丢弃  
```

HTTPError 实例产生后会有一个整型'code'属性，是服务器发送的相关错误号。Error Codes 错误码
因为默认的处理器处理了重定向(300 以外号码)，并且 100-299 范围的号码指示成功，所以你只能看到 400-599 的错误号码。

BaseHTTPServer.BaseHTTPRequestHandler.response 是一个很有用的应答号码字典，显示了 HTTP 协议使用的所有的应答号。当一个错误号产生后，服务器返回一个 HTTP 错误号，和一个错误页面。你可以使用 HTTPError 实例作为页面返回的应答对象 response。这表示和错误属性一样，它同样包含了 read,geturl，和 info 方法。

我们建一个 urllib2_test07.py 来感受一下：

```
import urllib2  
req = urllib2.Request('http://bbs.csdn.net/callmewhy')  
  
try:  
    urllib2.urlopen(req)  
  
except urllib2.URLError, e:  
  
    print e.code  
    #print e.read()  
```

按下 F5 可以看见输出了 404 的错误码，也就说没有找到这个页面。


## Wrapping

所以如果你想为 HTTPError 或 URLError 做准备，将有两个基本的办法。推荐使用第二种。

我们建一个 urllib2_test08.py 来示范一下第一种异常处理的方案：

```
from urllib2 import Request, urlopen, URLError, HTTPError  
  
req = Request('http://bbs.csdn.net/callmewhy')  
  
try:  
  
    response = urlopen(req)  
  
except HTTPError, e:  
  
    print 'The server couldn\'t fulfill the request.'  
  
    print 'Error code: ', e.code  
  
except URLError, e:  
  
    print 'We failed to reach a server.'  
  
    print 'Reason: ', e.reason  
  
else:  
    print 'No exception was raised.'  
    # everything is fine  
```

和其他语言相似，try 之后捕获异常并且将其内容打印出来。

这里要注意的一点，except HTTPError 必须在第一个，否则 except URLError 将同样接受到 HTTPError。
因为 HTTPError 是 URLError 的子类，如果 URLError 在前面它会捕捉到所有的 URLError（包括HTTPError）。

我们建一个 urllib2_test09.py 来示范一下第二种异常处理的方案：

```
from urllib2 import Request, urlopen, URLError, HTTPError  
  
req = Request('http://bbs.csdn.net/callmewhy')  
    
try:    
    
    response = urlopen(req)    
    
except URLError, e:    
  
    if hasattr(e, 'code'):    
    
        print 'The server couldn\'t fulfill the request.'    
    
        print 'Error code: ', e.code    
  
    elif hasattr(e, 'reason'):    
    
        print 'We failed to reach a server.'    
    
        print 'Reason: ', e.reason    
    
    
else:    
    print 'No exception was raised.'    
    # everything is fine    
```
