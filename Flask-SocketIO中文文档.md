官网在[这里](https://flask-socketio.readthedocs.org/en/latest/)，英语好的直接去看官网文档吧，我是英语渣只能翻译个大概;另外注意本文翻译时间，可能你看到的时候官网已经更新了。

* * *

flask-socketio赋予了flask程序支持服务端和客户端间双向低延迟通讯的能力，客户端可以使用 [SocketIO](http://socket.io/) 库或任何支持与服务端建立长链接的兼容库。

## 安装

可以直接使用pip安装：
<pre class="lang:python decode:true ">pip install flask-socketio</pre>

## 依赖

自从1.0版开始，这个扩展完全兼容了python2.7和python3.3+版本。异步服务的支持基于下面3个选择中的一个：

*   [eventlet](http://eventlet.net/) 是3个选项中性能最高的，同时支持长轮循(long-polling)和WebSocket。
*   [gevent](http://www.gevent.org/) 是在以前版本中使用的框架，支持长轮循，如果想支持WebSocket的话需要同时安装[gevent-websocket](https://pypi.python.org/pypi/gevent-websocket/) 库。使用gevent和gevent-websocket结合性能也不错，但略低于eventlet。
*   flask 基于Werkzeug的开发服务也能用，不过性能上不如上面2个选项，所以它应该只用于开发时使用。这个选项只支持长轮循。
本扩展将自动检测哪些异步框架被安装，默认首选eventlet，其次是gevent，最后是flask自带的开发服务。

对于客户端来说，可以使用官方的Socket.Io来建立于服务端的链接，也有使用swift和c++写成的客户端。非官方的客户端也能工作，只要它实现了[Socket.IO](https://github.com/socketio/socket.io-protocol) 协议。

## 目前的局限

目前flask-socketio只能同时运行在单个进程中（这里应该指的是一个进程中仅能存在一个实例的意思，而非只能开启一个进程吧...），解决这个限制的工作正在进行中。

（关于升级变化、以及从老版本迁移到新版的注意事项我就不翻译了，因为没用过老版本）

## 初始化

下面的代码展示了如何添加flask-socketio到flask程序中：
<pre class="lang:python decode:true ">from flask import Flask, render_template
from flask_socketio import SocketIO

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret!'
socketio = SocketIO(app)

if __name__ == '__main__':
    socketio.run(app)</pre>
init_app()这种初始化方式也支持，注意web服务器的启动方式。socketio.run()封装并替换了app.run()这种flask的标准启动方式。在debug模式中，Werkzeug服务依然被使用并在socketio.run()中进行了正确的配置。在生产环境中将优先使用eventlet或者gevent，如果这2个都没安装的话werkzeug将被使用。

程序必须为客户端提供一个页面使其能够加载Socket.io库并建立链接：
<pre class="lang:js decode:true ">&lt;script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/socket.io/1.3.6/socket.io.min.js"&gt;&lt;/script&gt;
&lt;script type="text/javascript" charset="utf-8"&gt;
    var socket = io.connect('http://' + document.domain + ':' + location.port);
    socket.on('connect', function() {
        socket.emit('my event', {data: 'I\'m connected!'});
    });
&lt;/script&gt;
</pre>

## 接收消息

当使用Socketio时，消息以事件的形式被双方接收。客户端使用Javascript的回调函数，服务端需要为事件注册处理函数，就像在视图中注册路由处理函数那样。

下面的代码展示了如何创建服务端未命名事件处理函数：
<pre class="lang:python decode:true ">@socketio.on('message')
def handle_message(message):
    print('received message: ' + message)</pre>
上面的例子使用了字符串消息，下面的例子展示如何使用json格式的消息：
<pre class="lang:python decode:true ">@socketio.on('json')
def handle_json(json):
    print('received json: ' + str(json))</pre>
更灵活的方式是使用自定义的事件名称：
<pre class="lang:python decode:true ">@socketio.on('my event')
def handle_my_custom_event(json):
    print('received json: ' + str(json))</pre>
命名事件是非常灵活的，这种方式不需要添加额外的数据来描述消息类型（不过截至到翻译这里，没看出自定义名称的好处...待使用后看看吧）

Flask-SocketIO也支持SocketIO的命名空间，准许客户端在同一个socket上建立多个独立的链接：
<pre class="lang:python decode:true ">@socketio.on('my event', namespace='/test')
def handle_my_custom_namespace_event(json):
    print('received json: ' + str(json))</pre>
如果没有指定命名空间，默认使用全局默认命名空间“/”。

客户端也许需要一个回调来确认收到了消息，任何从服务端处理函数返回的值将作为客户端回调函数的参数被传递回客户端：
<pre class="lang:python decode:true ">@socketio.on('my event')
def handle_my_custom_event(json):
    print('received json: ' + str(json))
    return 'one', 2</pre>
上面的例子中，客户端的回调函数将收到2个参数，'one'和2。如果服务端没返回任何值，客户端的回调函数就不会收到任何参数。

## 发送消息

正如上面说的，SocketIO事件处理函数可以通过调用send()和emit()函数来向链接的客户端发送回应消息。下面的例子展示了服务端接受客户端的消息并返回给客户端：
<pre class="lang:python decode:true ">from flask_socketio import send, emit

@socketio.on('message')
def handle_message(message):
    send(message)

@socketio.on('json')
def handle_json(json):
    send(json, json=True)

@socketio.on('my event')
def handle_my_custom_event(json):
    emit('my response', json)</pre>
注意send()用于非自定义的事件而emit()用于自定义的事件中。

当使用命名空间时，send()和emit()默认使用传进来的消息的命名空间，我们也可以使用namespace参数来指定使用哪个命名空间：
<pre class="lang:python decode:true ">@socketio.on('message')
def handle_message(message):
    send(message, namespace='/chat')

@socketio.on('my event')
def handle_my_custom_event(json):
    emit('my response', json, namespace='/chat')</pre>
SocketIO支持确认回调来确认消息被客户端成功接收：
<pre class="lang:python decode:true ">def ack():
    print 'message was received!'

@socketio.on('my event')
def handle_my_custom_event(json):
    emit('my response', json, callback=ack)</pre>
当使用回调的时候Javascript客户端接受一个回调函数来调用接收到的消息，在客户端回调函数执行完成后服务端相应的回调函数将被执行。如果客户端的回调函数返回了任何值，它们将被作为参数传递给服务端的回调函数。

应用程序的客户端也可以向服务端为一个事件请求确认回调，如果服务端想为这个回调提供参数，仅需要在事件处理函数中返回它们就可以了：
<pre class="lang:python decode:true ">@socketio.on('my event')
def handle_my_custom_event(json):
    # ... handle the event

    return 'foo', 'bar', 123  # client callback will receive these 3 arguments
</pre>
（关于上面客户端回调、服务端回调以及相关确认回调神码的有些绕，需要从代码层面去理解一下真正的含义）

## 广播

SocketIO中另一个特别有用的特性就是消息广播。Flask-SocketIO通过在send()和emit()函数中设定参数broadcast=True来支持这一特性：
<pre class="lang:python decode:true ">@socketio.on('my event')
def handle_my_custom_event(data):
    emit('my response', data, broadcast=True)</pre>
当一条消息以广播方式发送时，同命名空间下的所有建立了链接的客户端都将收到这条消息，包括发送者自己。如果没有使用命名空间，所以链接到全局命名空间的客户端都将收到这条消息。注意，回调函数不会被广播消息触发。

至此，上述所有的示例都在说明服务端如何响应客户端发起的事件请求，但对于某些情况，服务端需要作为消息的发起者。比如服务器需要在后端向所有客户端发送通知的情景中，socketio.send()和socketio.emit()可以用来向所有用户发送广播：
<pre class="lang:python decode:true ">def some_function():
    socketio.emit('some event', {'data': 42})</pre>
这里注意socketio.send()、socketio.emit()和send()、emit()函数并不相同。

（原文是：Note that socketio.send() and socketio.emit() are not the same functions as the context-aware send() and emit()，但context-aware翻译成上下文敏感感觉有些怪异）

同时注意上述例子中并没有客户端卵事，所以默认broadcast=True，不需要额外指定了。

## 房间

在很多程序需要对用户进行分组，并且每个组之间的成员可以相互发送消息。典型的例子就是聊天室程序，用户接受他们进入的房间的消息，而不会收到其它房间发送的消息。Flask-SocketIO通过join_room() 和leave_room()函数来支持这种特性：
<pre class="lang:python decode:true ">from flask_socketio import join_room, leave_room

@socketio.on('join')
def on_join(data):
    username = data['username']
    room = data['room']
    join_room(room)
    send(username + ' has entered the room.', room=room)

@socketio.on('leave')
def on_leave(data):
    username = data['username']
    room = data['room']
    leave_room(room)
    send(username + ' has left the room.', room=room)</pre>
send()和emit()函数通过指定room参数来确保消息发送给所有在同房间的客户端。

所有的客户端在建立链接时就被分配了一个房间，并根据链接的session id来命名。这个值可以从request.sid中获得。客户端可以加入任何房间并可以被赋予任何名字。当客户端断开链接时，它将被从所有加入的房间中被移除。与上下文无关的(context-free)socketio.send()、socketio.emit()函数也接受room参数向房间中的所有客户端发送广播。

每个客户端都被指定了个人房间后，向个人发送消息可以使用客户端的session id作为room的参数。（个人理解就是私聊功能）

## 链接事件

Flask-SocketIO也分离了建立链接和断开链接事件，下面的示例展示了如何为这2个事件注册处理函数：
<pre class="lang:python decode:true ">@socketio.on('connect', namespace='/chat')
def test_connect():
    emit('my response', {'data': 'Connected'})

@socketio.on('disconnect', namespace='/chat')
def test_disconnect():
    print('Client disconnected')</pre>
connect事件可以选择return False来拒绝链接，这样可以对客户端进行身份认证。

注意connect和disconnect事件在每个命名空间中被单独发送。

## 错误控制

Flask-SocketIO也可以处理异常：
<pre class="lang:python decode:true ">@socketio.on_error()        # Handles the default namespace
def error_handler(e):
    pass

@socketio.on_error('/chat') # handles the '/chat' namespace
def error_handler_chat(e):
    pass

@socketio.on_error_default  # handles all namespaces without an explicit error handler
def default_error_handler(e):
    pass</pre>
错误处理程序接受exception object作为参数。

当前请求的消息和数据可以通过request.event 来获取，这在记录错误日志和调试程序时候是非常有用的：
<pre class="lang:python decode:true ">from flask import request

@socketio.on("my error event")
def on_my_event(data):
    raise RuntimeError()

@socketio.on_error_default
def default_error_handler(e):
    print request.event["message"] # "my error event"
    print request.event["args"]    # (data,)</pre>

## 访问Flask全局上下文

SocketIO事件处理函数和路由处理函数是不同的，这些不同使人们在SocketIO事件处理函数中什么可以做、什么不可以做感到很困惑。最主要的区别在于：所有为客户端SocketIO而编写的事件处理函数都发生在单一的长时间请求上下文中。

尽管有差异，Flask-SocketIO尝试通过使环境类似一个普通的HTTP请求来简化SocketIO的事件处理函数。下面的列表说明了什么可以做和什么不可以做：

1.  应用程序上下文(application context)在事件处理函数被调用前载入，所以current_app 和 g可以在处理函数中使用
2.  请求上下文(request context)在事件处理函数被调用前载入，所以request和session可以在处理函数中使用。但注意，在WebSocket事件中并不和单个请求关联，所以请求上下文在链接建立时被所有事件都载入了并在整个请求的生命周期中都存在。
3.  全局请求上下文( request context global)通过增加sid来支持为链接设置独一无二的session id。这个值在第一个用户进入房间是被使用。
4.  全局请求上下文( request context global)通过增加 namespace和event来记录当前事件处理函数的命名空间和事件参数。event是一个用message和args当作key的字典。
5.  全局会话上下文(session context global)的行为和常规的不同，在SocketIO链接建立时创建的用户session副本可以在事件处理函数中被使用。如果SocketIO事件处理函数中修改了这个session，修改后的值可以在以后的SocketIO事件处理函数中被使用，但常规的http路由处理函数并不知道这些改变。事实上，当一个SocketIO事件处理函数修改了session，一个当前session的分支将会被专门创建出来。作出这种限制的原因在于当我们需要存储会话时cookie将被发送到客户端，而这需要HTTP请求与回应，这里并不存在SocketIO链接。当使用类似Flask-Session或Flask-KVSession这种服务端会话扩展时，在HTTP路由处理函数中修改session在SocketIO事件处理函数中是可见的，只要不再SocketIO事件处理函数中修改。（上面这一大段翻译的比较绕，核心观点就是如果要对session进行操作，请在http路由处理函数中而别在SocketIO处理函数中。路由函数中处理后事件函数中可见，反之则不行）
6.  before_request 和 after_request钩子不会在事件处理函数中调用
7.  SocketIO事件处理函数可以被自定义的装饰器所装饰，但大多数Flask装饰器不适用于事件处理函数，因为在事件处理函数中没有了Response object这个概念。

## 认证

一个常见的需求就是验证用户身份，传统的验证方式基于HTTP请求和web from不能在SocketIO链接中使用，因为没有地方发送HTTP请求和响应。如果必要的话，应用程序可以实现一个定制的登录功能，当用户按下登录按钮时通过SocketIO向服务端发送一个凭证。

然而更方便的方法是在建立SocketIO链接之前进行传统的认证，用户的认证信息可以被存储在session或者cookie中，晚些时候在SocketIO链接建立时候，这些认证信息将可以在事件处理函数中使用。

## 结合使用Flask-Login与Flask-SocketIO

Flask_socketIO能够使用由Flask-Login维护的用户信息。当常规的Flask-Login身份验证执行后， login_user()被调用在session中记录用户信息，任何事件处理函数都可以使用current_user变量：
<pre class="lang:python decode:true ">@socketio.on('connect')
def connect_handler():
    if current_user.is_authenticated:
        emit('my response',
             {'message': '{0} has joined'.format(current_user.name)},
             broadcast=True)
    else:
        return False  # not allowed here</pre>
注意login_required装饰器不能在SocketIO事件处理函数上使用，但下面所示的自定义装饰器可以断开未认证用户的链接：
<pre class="lang:python decode:true ">import functools
from flask import request
from flask_login import current_user
from flask_socketio import disconnect

def authenticated_only(f):
    @functools.wraps(f)
    def wrapped(*args, **kwargs):
        if not current_user.is_authenticated:
            disconnect()
        else:
            return f(*args, **kwargs)
    return wrapped

@socketio.on('my event')
@authenticated_only
def handle_my_custom_event(data):
    emit('my response', {'message': '{0} has joined'.format(current_user.name)},
         broadcast=True)</pre>

## 部署

最简单的部署方式就是安装eventlet或者gevent，并且按照上面说的调用socketio.run(app)，这将在eventlet或者gevent上运行程序。

另一种替代方式是使用[gunicorn](http://gunicorn.org/) 作为网络服务器，并使用eventlet或者gevent workers:
<pre class="lang:sh decode:true ">gunicorn --worker-class eventlet module:app</pre>
<pre class="lang:sh decode:true ">gunicorn --worker-class gevent module:app</pre>
上面的命令中，module指定义了应用程序实例的python的包或模块名，app就是应用程序实例本身。

eventlet本身就支持了WebSocket标准，但gevent想使用WebSocket需要安装gevent-websocket，如果这个没安装则仅能使用长轮循的机制。

当在gunicorn中使用gevent worker和WebSocket时，启动命令必须指明使用支持WebSocket的gevent worker:
<pre class="lang:sh decode:true ">gunicorn --worker-class geventwebsocket.gunicorn.workers.GeventWebSocketWorker module:app</pre>

## 使用Nginx作为WebSocket的反向代理

使用nginx作为应用程序的前端反向代理是可能的，然而有一点需要特别注意的是只有1.4及以后的版本才支持WebSocket协议，下面是一个常规的配置：
<pre class="lang:sh decode:true ">server {
    listen 80;
    server_name localhost;
    access_log /var/log/nginx/example.log;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_redirect off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /socket.io {
        proxy_pass http://127.0.0.1:5000/socket.io;
        proxy_redirect off;
        proxy_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}</pre>
至于API部分就不翻译了，有需求的自己去官网看吧。
