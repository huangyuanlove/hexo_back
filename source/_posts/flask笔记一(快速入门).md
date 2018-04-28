---
title: flask笔记一 快速入门
date: 2017-07-15 09:55:52
tags: [Python,flask]
---
学习python和flask中看过的资料和自己的实践总结
[Flask英文](http://flask.pocoo.org/)
以下内容来自[flask中文](http://docs.jinkan.org/docs/flask)
<!--more-->
#### 环境安装
Flask依赖两个外部库：Werkzeug和Jinja2。Werkzeug是一个WSGI工具集、Jinja2负责渲染模板。首先需要python2.6或者更高的版本，python3.X安装方式可能有所不一致。
###### virtualenv
virtualenv 为每个不同项目提供一份 Python 安装。它并没有真正安装多个 Python 副本，但是它确实提供了一种巧妙的方式来让各项目环境保持独立。
`sudo apt-get install python-virtualenv`
virtualenv安装完成后，用IDE比如pycharm创建一个Flask的项目工程，并在其下创建一个venv文件夹
```shell
$ mkdir myproject
$ cd myproject
$ virtualenv venv
New python executable in venv/bin/python
Installing distribute............done.
```
然后激活相应的环境
`$ . venv/bin/activate`
然后激活virtualenv中的Flask
`$ pip install Flask`
#### 项目配置
在pycharm中，打开setting
![项目设置](/image/python/Flask/pycharm_project_setting1.png)
在Project Interpreter中选择当前工程文件下的virtualenv
打开工程的Configuration
![项目设置](/image/python/Flask/pycharm_project_setting2.png)
在python interpreter中选择当前工程文件下的virtualenv
#### 项目说明
###### 总述
刚刚新建的工程看起来会是这样的
``` python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run()
```
运行这段程序(文件名不要是 flask.py，这会和已有的文件冲突)，浏览器访问`127.0.0.1:5000`，然后就会看到熟悉的`hello world`.
那么，这段代码做了什么？

1. 首先，我们导入了 Flask 类。这个类的实例将会是我们的 WSGI 应用程序。
接下来，我们创建一个该类的实例，第一个参数是应用模块或者包的名称。 如果你使用单一的模块（如本例），你应该使用 __name__ ，因为模块的名称将会因其作为单独应用启动还是作为模块导入而有不同（ 也即是 '__main__' 或实际的导入名）。这是必须的，这样 Flask 才知道到哪去找模板、静态文件等等。详情见 Flask 的文档。
2. 然后，我们使用 route() 装饰器告诉 Flask 什么样的URL 能触发我们的函数。
这个函数的名字也在生成 URL 时被特定的函数采用，这个函数返回我们想要显示在用户浏览器中的信息。
3. 最后我们用 run() 函数来让应用运行在本地服务器上。 其中 if __name__ == '__main__': 确保服务器只会在该脚本被 Python 解释器直接执行的时候才会运行，而不是作为模块导入的时候。
  欲关闭服务器，按 Ctrl+C。
你会发现它只能从自己的计算机上访问，网络中其他任何地方都不能访问，我们可以修改run()方法使该服务公开可用
`app.run(host='0.0.0.0')`
这样会让操作系统监听所有的公网IP
同样可以开启debug模式
`app.debug = True;app.run()`
或者
`app.run(debug=True)` 
###### 路由
如上面的代码所示，route()装饰其吧一个函数绑定到对应的URL上
``` python
@app.route('/')
def index():
    return 'Index Page'

@app.route('/hello')
def hello():
    return 'Hello World'
```
不仅如此，还可以构造含有动态部分的URL，也可以在一个函数上附着多个规则
###### 变量规则
要给 URL 添加变量部分，你可以把这些特殊的字段标记为 <variable_name> ， 这个部分将会作为命名参数传递到你的函数。规则可以用 <converter:variable_name> 指定一个可选的转换器。如下所示

``` python
@app.route('/user/<username>')
def show_user_profile(username):
    # show the user profile for that user
    return 'User %s' % username

@app.route('/post/<int:post_id>')
def show_post(post_id):
    # show the post with the given id, the id is an integer
    return 'Post %d' % post_id
```
###### HTTP方法
HTTP有许多不同的访问URL方法。默认情况下，路由只回应GET请求，但是通过route()装饰器传递methods参数可以改变这个行为。

``` python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        do_the_login()
    else:
        show_the_login_form()
```
###### 模板渲染
像javaweb一样，不会直接用servlet直接输出html代码，Flask可以使用render_template()方法来渲染模板，Flask配备Jinja2模板引擎[Jinja2中文](http://docs.jinkan.org/docs/jinja2/)，[Jinja2英文](http://jinja.pocoo.org/)
```python
from flask import render_template

@app.route('/hello/')
@app.route('/hello/<name>')
def hello(name=None):
    return render_template('hello.html', name=name)
```
Flask 会在 templates 文件夹里寻找模板。所以，如果你的应用是个模块，这个文件夹应该与模块同级；如果它是一个包，那么这个文件夹作为包的子目录:
情况 1: 模块:
```
/application.py
/templates
    /hello.html
```
情况 2: 包:
```
/application
    /__init__.py
    /templates
        /hello.html
```
下面是`hello.html`文件内容
``` Jinja2
<!doctype html>
<title>Hello from Flask</title>
{% if name %}
  <h1>Hello {{ name }}!</h1>
{% else %}
  <h1>Hello World!</h1>
{% endif %}
```
###### 访问请求数据
可以通过`request.form`属性来访问表单数据。
``` python
@app.route("/signin", methods=["POST"])
def signin():
    print request.args.get("username")
    username = request.form["username"]
    return render_template('hello.html', name=username)
```
如果form表单中不存在所要获取的键，会抛出特殊的KeyError异常。
###### 文件上传
首先确保HTML表单中设置 `enctype="multipart/form-data" `属性，否则浏览器根本不会发送文件。
已上传的文件存储在内存或是文件系统中一个临时的位置。可以通过请求对象的`file`属性访问它。每个上传的文件都会存储在这个字典里。同时它还有一个`save()`方法，这个方法允许把文件保额哦村到服务器的文件系统上。如果像知道上传的文件在客户端是什么名字，可以访问`filename`属性
``` python
from flask import request

@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        f = request.files['the_file']
        f.save('/var/www/uploads/uploaded_file.txt')
```
###### Cookies
可以通过`cookies`属性来访问Cookies，用响应对象的`set_cookie`方法来设置Cookies。请求对象的`cookies`属性是一个内容为客户端提交的所有Cookies的字典.
读取cookies：
```python
from flask import request

@app.route('/')
def index():
    username = request.cookies.get('username')
    # use cookies.get(key) instead of cookies[key] to not get a
    # KeyError if the cookie is missing.
```
存储cookie：
``` python
from flask import make_response

@app.route('/')
def index():
    resp = make_response(render_template(...))
    resp.set_cookie('username', 'the username')
    return resp
```
需要注意的是，Cookies是设置在响应对象上的。由于通常视图函数只是返回字符串，之后Flask将字符串转换为响应对象。
###### 重定向和错误
可以使用`redirect()`函数把用户重定向到其他地方。用`abort()`函数放弃请求并返回错误代码，如下：
``` python
from flask import abort, redirect, url_for

@app.route('/')
def index():
    return redirect(url_for('login'))

@app.route('/login')
def login():
    abort(401)
    this_is_never_executed()
```
这是一个完全没有任何意义的代码，因为用户会从主页重定向到一个不能访问的页面。默认情况下，错误代码会显示一个黑白的错误页面，如果要定制错误页面，可以使用`errorhandler()`装饰器
```python
from flask import render_template

@app.errorhandler(404)
def page_not_found(error):
    return render_template('page_not_found.html'), 404
```
注意`render_template()调用之后的404，这告诉Flask，该页的错误代码是404。默认是200,也就是一切正常
###### 关于响应
视图函数的返回值会被自动转换为一个响应对象。如果返回值是一个字符串， 它被转换为该字符串为主体的、状态码为 200 OK``的 、 MIME 类型是 ``text/html 的响应对象。Flask 把返回值转换为响应对象的逻辑是这样：
1. 如果返回的是一个合法的响应对象，它会从视图直接返回。
2. 如果返回的是一个字符串，响应对象会用字符串数据和默认参数创建。
3. 如果返回的是一个元组，且元组中的元素可以提供额外的信息。这样的元组必须是 (response, status, headers) 的形式，且至少包含一个元素。 status 值会覆盖状态代码， headers 可以是一个列表或字典，作为额外的消息标头值。
4. 如果上述条件均不满足， Flask 会假设返回值是一个合法的 WSGI 应用程序，并转换为一个请求对象。
如果你想在视图里操纵上述步骤结果的响应对象，可以使用 make_response() 函数。

譬如你有这样一个视图:
```python
@app.errorhandler(404)
def not_found(error):
    return render_template('error.html'), 404
```
只需要把返回值表达式传递给`make_response()`，获取结果对象并修改，然后再返回它:
``` python 
@app.errorhandler(404)
def not_found(error):
    resp = make_response(render_template('error.html'), 404)
    resp.headers['X-Something'] = 'A value'
    return resp
```
###### 会话
除请求对象之后，还有一个`session`对象，它允许你在不同请求间存储特定用户的信息。他是在Cookies的基础上实现，并且对Cookies进行密钥签名。这意味着用户可以查看你Cookie的内容，但却不能修改它，除非用户知道签名的密钥。
要使用会话，需要设置一个密钥
``` python
from flask import Flask, session, redirect, url_for, escape, request

app = Flask(__name__)

@app.route('/')
def index():
    if 'username' in session:
        return 'Logged in as %s' % escape(session['username'])
    return 'You are not logged in'

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        session['username'] = request.form['username']
        return redirect(url_for('index'))
    return '''
        <form action="" method="post">
            <p><input type=text name=username>
            <p><input type=submit value=Login>
        </form>
    '''

@app.route('/logout')
def logout():
    # remove the username from the session if it's there
    session.pop('username', None)
    return redirect(url_for('index'))

# set the secret key.  keep this really secret:
app.secret_key = 'A0Zr98j/3yX R~XHH!jmN]LWX/,?RT'
```
这里提到的`escape()`可以在模板引擎外做转义

----
以上