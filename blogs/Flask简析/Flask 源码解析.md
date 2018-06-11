# Flask 源码解析

## 基本概念
### WSGI：服务端与程序端的协议
实际生产中，`python`程序是放在服务器的 `http server`（比如 `apache`， `nginx` 等）上的。现在的问题是 服务器程序怎么把接受到的请求传递给` python `呢，怎么在网络的数据流和 `python` 的结构体之间转换呢？这就是` wsgi` 做的事情：一套关于程序端和服务器端的规范，或者说统一的接口。
@import "wsgi概念图1.jpg"
`wsgi`需要一个符合规范的`application`实例来处理所有的请求，并将`http server`的请求转换为符合`pyhton`标准的请求传递给这个实例。
`python `标准官方库中，提供了一个简单的`wsgi`实现，位于`wsgiref`包中：

```python
from wsgiref.simple_server import make_server, demo_app

httpd = make_server('', 8000, demo_app)
print "Serving HTTP on port 8000..."

# Respond to requests until process is killed
httpd.serve_forever()

# Alternative: serve one request, then exit
httpd.handle_request()
```
其中`make_server`返回了一个继承自`HTTPServer`的`WSGIServer`实例。而`WSGIServer`实例是通过继承`SocketServer`包中的`TCPServer`来实现的。`WSGIServer`实现了上文中叙述的`application`的绑定，并实现了处理请求的逻辑。

其中`demo_app`即为上文描述的对`wsgi`提供的

##### werkzeug：Flask框架提供的默认wsgi协议实现

`werkzeug`是一个开源的`wsgi`协议的实现，与`flask`框架的作者是同一帮人。
与官方标准包不同的是，`werkzeug`不仅提供了多线程，多进程的能力，还封装了请求，响应，cookie等常用的web数据结构，以供使用。比如：

```python
from werkzeug.wrappers import Request, Response

def applicatio n(environ, start_response):
    request = Request(environ)
    text = 'Hello %s!' % request.args.get('name', 'World')
    response = Response(text, mimetype='text/plain')
    return response(environ, start_response)
```
`werkzeug`可以看做是一个web框架底层的工具集，非常灵活。下面是一个最简单的使用例子：

```python
from werkzeug.wrappers import Request, Response

@Request.application
def application(request):
    return Response('Hello, World!')

if __name__ == '__main__':
    from werkzeug.serving import run_simple
    run_simple('localhost', 4000, application)
```

看起来与`python`标准包中提供的`simple_server`非常相似，实际上二者的实现机理也是类似的。在`werkzeug`底层，通过`ThreadedWSGIServer`与`ForkingWSGIServer`这两个类实现多线程或者多进程处理请求服务器，这两个类继承自`BaseWSGIServer`，而`BaseWSGIServer`的父类也是上文中介绍过的`HTTPServer`。
值得一提的是，`werkzeug`在`BaseWSGIServer`中提供了`ssl`的支持：

```python
if ssl_context is not None:
    if isinstance(ssl_context, tuple):
        ssl_context = load_ssl_context(*ssl_context)
    if ssl_context == 'adhoc':
        ssl_context = generate_adhoc_ssl_context()
    # If we are on Python 2 the return value from socket.fromfd
    # is an internal socket object but what we need for ssl wrap
    # is the wrapper around it :(
    sock = self.socket
    if PY2 and not isinstance(sock, socket.socket):
        sock = socket.socket(sock.family, sock.type, sock.proto, sock)
    self.socket = ssl_context.wrap_socket(sock, server_side=True)
    self.ssl_context = ssl_context
else:
    self.ssl_context = None
```
#####uWSGI与uwsgi：高性能服务器与uWSGI服务器独占协议

上文中提到的，无论是标准库中的默认实现，还是`werkzeug`，均为调试开发的服务器。`flask`文档中明确建议，不应该在生产环境中使用：
> The development server is not intended to be used on production systems. It was designed especially for development purposes and performs poorly under high load.

由于`werkzeug`及标准库中的默认实现，均不是为生产环境所设计，它们在高负载的情况下，表现会非常差。无论是`apache`还是`nginx`，在性能方面都做了许多工作，这并不是一个简单的多线程或者多进程就可以相媲美的（比如在安全性，内存优化等方面都有所差距）。所以我们还需要高性能的生产环境的web服务器。`uwsgi`就是这么一个东西。

而`uwsgi`，简单的来说是一个`uWSGI`服务器自有的协议，它用于定义传输信息的类型（type of information），每一个`uwsgi packet`前**4byte**为传输信息类型描述。至于为什么需要这个东西，网上找到一些[讨论](https://www.zhihu.com/question/46945479)：

>WSGI是一种编程接口，而uwsgi是一种传输协议，从作用上来讲，的确跟fastcgi是最接近的。跟fastcgi的区别在于它是面向多并发的。我们知道fastcgi是CGI的替代品，它的工作方式仍然跟CGI是类似的，当一个请求进入的时候，通过socket发送请求的环境变量，然后发送POST数据（相当于CGI的stdin），然后等待程序输出（相当于CGI的stdout），等输出结束后，再发送下一个请求。这就导致fastcgi最大的并发量被限制为fastcgi后端的数量，显然这样的服务器模式对于并发量很大、单个请求耗时比较长的服务是不合适的，因此很多时候我们不愿意使用fastcgi部属而是使用反向代理的方式配置。但是跟反向代理比起来，fastcgi显然也是有好处的，最重要的好处在于解析HTTP协议的部分被offload到了前端服务器一级，后端服务器不再解析HTTP协议，这样就减轻了后端的压力，由于前端是nginx这样用C/C++高性能实现的服务器，比起在后端的Python当中使用脚本语言解析HTTP协议，效率要高不少。uwsgi想要继承fastcgi的这种好处，它通过将消息分片的方式，可以在一个socket上并发传输多个请求，这样就解决了一个连接上一次只能传输一个请求的问题。熟悉HTTP2.0的话会发现这个分片机制跟HTTP2.0很像

## Flask启动流程

`flask`的hello word代码：

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run()
```

`app.run()`内部：

```python
def run(self, host=None, port=None, debug=None, **options):
    """Runs the application on a local development server."""
    from werkzeug.serving import run_simple

    # 如果host 和 port 没有指定，设置 host 和 port 的默认值 127.0.0.1 和 5000
    if host is None:
        host = '127.0.0.1'
    if port is None:
        server_name = self.config['SERVER_NAME']
        if server_name and ':' in server_name:
            port = int(server_name.rsplit(':', 1)[1])
        else:
            port = 5000

    # 调用 werkzeug.serving 模块的 run_simple 函数，传入收到的参数
    # 注意第三个参数传进去的是 self，也就是要执行的 web application
    try:
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False
```

可以看出，此处不仅做了端口及地址的配置，而且调用了上文中提到的`werkzeug`的`run_simple`函数。从而完成了服务器的启动。同时还将`self`即`flask`实例作为`application`传递了进去。

根据`wsgi`协议，这个`application`实例应该是可以被调用的，并且每次请求过来，都会调用这个实例。找到app.py中的`__call__`函数：

```python
def __call__(self, environ, start_response):
    """The WSGI server calls the Flask application object as the
    WSGI application. This calls :meth:`wsgi_app` which can be
    wrapped to applying middleware."""
    return self.wsgi_app(environ, start_response)
    
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

每次请求过来时，`flask`都会调用`wsgi_app`函数，在内部，通过外部传入的`env`参数获得`ctx`变量，执行压入操作。通过`full_dispatch`分发请求且获得响应。最后执行`ctx`的弹出操作。

放过为什么要进行压入弹出操作，先看分发请求的代码`full_dispatch_request`：

```python
def full_dispatch_request(self):
    self.try_trigger_before_first_request_functions()
    try:
        request_started.send(self)
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    return self.finalize_request(rv)
    
def dispatch_request(self):
    req = _request_ctx_stack.top.request
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
    rule = req.url_rule
    # if we provide automatic options for this URL and the
    # request came with the OPTIONS method, reply automatically
    if getattr(rule, 'provide_automatic_options', False) \
       and req.method == 'OPTIONS':
        return self.make_default_options_response()
    # otherwise dispatch to the handler for that endpoint
    return self.view_functions[rule.endpoint](**req.view_args)    
```

忽略一些细节及错误处理的代码，可以看到最终通过`dispatch_request`这个方法中获取`url`对应的视图函数并调用获得最终的响应。

## Flask路由实现

官方文档中提供的示例程序`hello world`是通过`python`中的装饰器来实现的，其本质是调用了`add_url_rule`这个方法：

```python
    def add_url_rule(self, rule, endpoint=None, view_func=None,
                     provide_automatic_options=None, **options):
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)
        options['endpoint'] = endpoint
        methods = options.pop('methods', None)
		 
		 # ... 删除了一些不相关的逻辑代码
		 
        rule = self.url_rule_class(rule, methods=methods, **options)
        
        self.url_map.add(rule)
        if view_func is not None:
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError('View function mapping is overwriting an '
                                     'existing endpoint function: %s' % endpoint)
            self.view_functions[endpoint] = view_func
```

可以看到有几个比较关键的逻辑：

1. 获取`endpoint`
2. 获取`rule`并绑定到`self.url_map`上
3. 获取`view_func`并绑定的`self.view_functions`上

根据文档中的说明:

```python
Basically this example::

    @app.route('/')
    def index():
        pass

Is equivalent to the following::

    def index():
        pass
    app.add_url_rule('/', 'index', index)
```

可以得知，`endpoint`为规则映射函数的函数名，`view_func`就是映射函数。于是上部分代码中的`_endpoint_from_view_func(view_func)`应当就是获取`view_func`的函数名：

```python
def _endpoint_from_view_func(view_func):
    assert view_func is not None, 'expected view func if endpoint ' \
                                  'is not provided.'
    return view_func.__name__
```

`self.url_map`是继承自`flask`实例中的`map`实例（继承自`werkzeug:Rule`类）。可见`flask`中的路由功能是`werkzeug`来实现的。放一段`werkzeug`文档中关于路由匹配的代码:

```python

>>>>  m = Map([
...     Rule('/', endpoint='index'),
...     Rule('/downloads/', endpoint='downloads/index'),
...     Rule('/downloads/<int:id>', endpoint='downloads/show')
... ])
>>> urls = m.bind("example.com", "/")
>>> urls.match("/", "GET")
('index', {})
>>> urls.match("/downloads/42")
('downloads/show', {'id': 42})

>>> urls.match("/downloads")
Traceback (most recent call last):
  ...
RequestRedirect: http://example.com/downloads/
>>> urls.match("/missing")
Traceback (most recent call last):
  ...
NotFound: 404 Not Found
```
可见`flask`利用`werkzeug`的路由功能，现实了获取请求URL中部分信息及绑定到`endpoint`的功能。
同时，`flask`将`endpoint`与`view_func`保存在实例中的一个字典内。

到这里，我们大概能清楚路由的实现逻辑。回到`dispatch_request`：

```python
def dispatch_request(self):
    req = _request_ctx_stack.top.request
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
    rule = req.url_rule
    # if we provide automatic options for this URL and the
    # request came with the OPTIONS method, reply automatically
    if getattr(rule, 'provide_automatic_options', False) \
       and req.method == 'OPTIONS':
        return self.make_default_options_response()
    # otherwise dispatch to the handler for that endpoint
    return self.view_functions[rule.endpoint](**req.view_args)    
```

根据上文，可以得以解释为什么可以从`view_fucntions`中通过`endpoint`获取视图函数。唯一的疑问就是为什么可以从`req`中获取`rule`，且`rule`中的`point`是什么时候绑定上去的？

因为已经知道`werkzeug`中实现了通过URL获取`endpoint`的功能，我们可以猜想此处的`req`很可能是对`werkzeug`相关功能的封装。

抛开为什么要从`_request_ctx_stack.top`获取`req`，着重于此处的`req`。

之前提到过，`WSGI`协议规定，`application`实例需要实现`__call__`方法，下面这个`wsgi_app`方法就是`flask`实例`__call__`方法内部调用的一个方法。

```python
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

注意：

1. `ctx = self.request_context(environ)`
2. `ctx.push()`

第一个就是构造`req`所在。查找相关的方法可以得知，此处构造了一个`class RequestContext(object):`实例。
在此实例的对象方法中：

```python
def __init__(self, app, environ, request=None):
    self.app = app
    if request is None:
        request = app.request_class(environ)
    self.request = request
    
    #此处通过self.request将请求信息传递给url_adapter
    self.url_adapter = app.create_url_adapter(self.request)
    
    self.flashes = None
    self.session = None
    self._implicit_app_ctx_stack = []
    self.preserved = False
    self._preserved_exc = None
    self._after_request_functions = []

  	 #注意此处，开始匹配url
    self.match_request()
    
def match_request(self):
    """Can be overridden by a subclass to hook into the matching
    of the request.
    """
    try:
        url_rule, self.request.view_args = \
            self.url_adapter.match(return_rule=True)
        self.request.url_rule = url_rule
    except HTTPException as e:
        self.request.routing_exception = e
```

可见在构造`req`实例的时候，先将实际的请求传递给了`slef.url_adapter`，之后调用了`match_reqeust`这个方法。在内部通过`url_adapter`进行了匹配，获得了此前注册好的`endpoint`。

此处的`url_adapter`是通过`flask`实例中`url_map`(`werkzeug:Map`)的`bind`方法创建出来的，是`werkzeug:MapAdapter`实例。

至此，抛开为什么要进行请求的压入弹出操作，我们已经了解了`flask`的路由过程：

1. 通过`werkzeug`提供的URL匹配功能，将`endpoint`和视图函数绑定起来。
2. 请求到来时，通过`werkzeug`的匹配功能，取出URL对应的`endpoint`。之后通过此`endpoint`取出对应的视图函数。

## Flask实例上下文及请求上下文

`flask`中有两种上下文，分别是请求上下文及`app`上下文，两种上下文提供了线程隔离的访问。具体实现在gloabs.py文件中：

```python
# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```

