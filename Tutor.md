# Chapter 3

## Receiving Form Data

如果你试图按下提交按钮，浏览器会显示一个 "Method Not Allowed"的错误。这是因为上一节中的登录视图函数(view function)到目前为止只完成了一半的工作。它可以在网页上显示表单，但它还没有处理用户提交的数据的逻辑。这是Flask-WTF让工作变得非常简单的另一个领域。下面是视图函数的更新版本，它接受并验证了用户提交的数据：

app/routes.py: Receiving login credentials
```python
from flask import render_template, flash, redirect

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        flash('Login requested for user {}, remember_me={}'.format(
            form.username.data, form.remember_me.data))
        return redirect('/index')
    return render_template('login.html', title='Sign In', form=form)'
```
这个版本的第一个更新是route decorator中的方法参数。这告诉Flask这个视图函数接受`GET`和`POST`请求，复写了默认值，即只接受`GET`请求。HTTP协议规定，`GET`请求是那些返回信息给客户端的请求（这里是指Web浏览器）。到目前为止，应用程序中的所有请求都属于这种类型。`POST`请求通常在浏览器向服务器提交表单数据时使用（实际上`GET`请求也可用于此目的，但不推荐这种做法）。之前浏览器向你显示的 "Method Not Allowed "的错误，是因为浏览器试图发送一个`POST`请求，而应用程序没有被配置为接受该请求。通过提供`methods`参数，你是在告诉Flask应该接受哪些请求方法。

`form.validate_on_submit()`方法完成所有的表单处理工作。当浏览器发送`GET`请求接收带有表单的网页时，这个方法将返回`False`，所以在这种情况下，函数跳过if语句，直接在函数的最后一行渲染模板。

当用户按下提交按钮，浏览器发送`POST`请求时，`form.validate_on_submit()`将收集所有的数据，运行所有附加在字段上的验证器(validators)，如果一切正常，它将返回`True`，表示数据有效，可以被应用程序处理。但是如果至少有一个字段没有通过验证，那么该函数将返回`False`，这将导致表单被渲染回给用户，就像`GET`请求的情况一样。稍后我将在验证失败时添加一条错误信息。

当`form.validate_on_submit()`返回`True`时，登录视图函数会调用两个从Flask导入的新函数。`flash()`函数是一种向用户显示信息的有用方法。很多应用程序都使用这种技术来让用户知道某些操作是否成功。在这种情况下，我打算把这种机制作为一个临时的解决方案，因为我们还没有掌握所有必要的基础部件来真正登录用户。我们现在能做的最好的事情就是显示一条确认应用程序收到凭证的消息。

登录视图函数中使用的第二个新函数是`redirect()`。这个函数指示客户端网络浏览器自动导航到一个不同的页面，作为一个参数给出。这个视图函数用它来重定向用户到应用程序的索引页。

当你调用`flash()`函数时，Flask会存储消息，但闪现的消息不会神奇地出现在网页上。应用程序的模板需要以适合网站布局的方式呈现这些闪烁的消息。我打算把这些信息添加到基础模板中，这样所有的模板都会继承这个功能。这就是更新后的基础模板：

app/templates/base.html: Flashed messages in base template
```html
<html>
    <head>
        {% if title %}
        <title>{{ title }} - microblog</title>
        {% else %}
        <title>microblog</title>
        {% endif %}
    </head>
    <body>
        <div>
            Microblog:
            <a href="/index">Home</a>
            <a href="/login">Login</a>
        </div>
        <hr>
        {% with messages = get_flashed_messages() %}
        {% if messages %}
        <ul>
            {% for message in messages %}
            <li>{{ message }}</li>
            {% endfor %}
        </ul>
        {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </body>
</html>
```
在这里，我使用 `with` 构造将调用 `get_flashed_messages()`的结果分配给`message`变量，所有这些都在模板的上下文中。`get_flashed_messages()`函数来自 Flask，并返回之前使用`flash()`注册的所有消息的列表。后面的条件检查`message`是否具有某些内容，在这种情况下，`<ul>`将元素与每条消息一起呈现为`<li>`列表项。这种呈现风格看起来不是很好，但稍后将介绍 Web 应用程序样式的主题。

这些闪烁的消息的一个有趣属性是，一旦通过 `get_flashed_messages` 函数请求一次，它们就会从消息列表中删除，因此它们在调用 `flash()`函数后只出现一次。

这是再次尝试应用程序并测试表单工作原理的好时机。请确保尝试在用户名或密码字段为空的情况下提交表单，以查看`DataRequired`验证程序如何停止提交过程。

## Improving FIeld Validation

附加到表单字段的验证程序可防止无效数据被接受到应用程序中。应用程序处理无效表单输入的方式是重新显示表单，让用户进行必要的更正。

如果您尝试提交无效数据，我相信您注意到，虽然验证机制运行良好，但没有向用户指示表单有问题，用户只是取回表单。下一个任务是通过在每个未通过验证的字段旁边添加有意义的错误消息来改善用户体验。

事实上，表单验证器(form validators)已经生成了这些描述性错误消息，因此缺少的只是模板中的一些附加逻辑来呈现它们。

以下是在用户名和密码字段中添加了字段验证消息的登录模板：

app/templates/login.html: Validation errors in login form template

```html
{% extends "base.html" %}

{% block content %}
    <h1>Sign In</h1>
    <form action="" method="post" novalidate>
        {{ form.hidden_tag() }}
        <p>
            {{ form.username.label }}<br>
            {{ form.username(size=32) }}<br>
            {% for error in form.username.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.password.label }}<br>
            {{ form.password(size=32) }}<br>
            {% for error in form.password.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.remember_me() }} {{ form.remember_me.label }}</p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

我所做的唯一更改是在用户名和密码字段之后添加 for 循环，这些字段以红色呈现验证器添加的错误消息。作为一般规则，任何附加了验证程序的字段都会有在`from`下添加验证时产生的任何错误消息。`<field_name>.errors`。这将是一个列表，因为字段可以附加多个验证器，并且多个验证器可能会提供错误消息以显示给用户。

如果您尝试使用空的用户名或密码提交表单，您现在会收到一条红色的错误消息。

## Generating Links

登录表单现在相当完整，但在结束本章之前，我想讨论在模板和重定向中包含链接的正确方法。到目前为止，您已经看到了一些定义链接的实例。例如，这是基本模板中的当前导航栏：

```html
    <div>
        Microblog:
        <a href="/index">Home</a>
        <a href="/login">Login</a>
    </div>
```

登录视图函数还定义了一个传递给`redirect()`函数的链接：

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        # ...
        return redirect('/index')
    # ...
```

直接在模板和源文件中编写链接的一个问题是，如果有一天您决定重新组织链接，那么您将不得不在整个应用程序中搜索和替换这些链接。

为了更好地控制这些链接，Flask 提供了一个名为`url_for()`的函数，该函数使用其内部映射来生成URLs以查看函数。例如，表达式`url_for（'login'）`返回`/login`，`url_for（'index'）`返回`/index`。`url_for()` 的参数是 _endpoint_ 名称，即视图函数的名称。

您可能会问为什么使用函数名称而不是 URLs 更好。事实是，URLs 比完全是整体的视图函数名称更有可能更改。第二个原因是，正如您稍后将了解到的那样，某些 URLs 中包含动态组件，因此手动生成这些 URLs 需要连接多个元素，这既乏味又容易出错。`url_for()` 也能够生成这些复杂的 URLs。

所以从现在开始，每次需要生成应用程序 URL 时，我都会使用 `url_for()`。然后，基本模板中的导航栏将变为：

app/templates/base.html: Use url\_for() function for links

```html
    <div>
        Microblog:
        <a href="{{ url_for('index') }}">Home</a>
        <a href="{{ url_for('login') }}">Login</a>
    </div>
```

app/routes.py: Use url\_for() function for links

```python
from flask import render_template, flash, redirect, url_for

# ...

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        # ...
        return redirect(url_for('index'))
    # ...
```
注意事项：
routes.py不能完全复制，还需要
```python
from app import app
from app.forms import LoginForm

@app.route('/')
```

问题：

1，改成url_for方式之后报错