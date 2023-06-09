<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Chapter 3](#chapter-3)
  - [Receiving Form Data](#receiving-form-data)
  - [Improving FIeld Validation](#improving-field-validation)
  - [Generating Links](#generating-links)
- [chapter 4:Database](#chapter-4database)
  - [Databases in Flask](#databases-in-flask)
  - [Database Migrations](#database-migrations)
  - [Flask-SQLAlchemy Configuration](#flask-sqlalchemy-configuration)
  - [Database Models](#database-models)
  - [Creating The Migration Repository](#creating-the-migration-repository)
  - [The First Database Migration](#the-first-database-migration)
  - [Database Upgrade and Downgrade Workflow](#database-upgrade-and-downgrade-workflow)
  - [Database Relationships](#database-relationships)
  - [Playing with the Database](#playing-with-the-database)
  - [Shell Context](#shell-context)
- [Chapter 5:User Logins](#chapter-5user-logins)
  - [Password Hashing](#password-hashing)
  - [Introduction to Flask-Login](#introduction-to-flask-login)
  - [Preparing The User Model for Flask-Login](#preparing-the-user-model-for-flask-login)
  - [User Loader Function](#user-loader-function)
  - [Logging Users In](#logging-users-in)
  - [Logging Users Out](#logging-users-out)
  - [Requiring Users To Login](#requiring-users-to-login)
  - [Showing The Logged In User in Templates](#showing-the-logged-in-user-in-templates)
  - [User Registration](#user-registration)
- [Chapter6:Profile Page and Avatars](#chapter6profile-page-and-avatars)
  - [User Profile Page](#user-profile-page)
  - [Avatars](#avatars)
  - [Using Jinja2 SUb-Templates](#using-jinja2-sub-templates)
  - [More Interesting Profiles](#more-interesting-profiles)
  - [Recording THe Last Visit Time For a User](#recording-the-last-visit-time-for-a-user)
  - [Profile Editor](#profile-editor)
- [Chapter7:Error Handling](#chapter7error-handling)
  - [Error Handling in Flask](#error-handling-in-flask)
- [Debug Mode](#debug-mode)
  - [Custom Error Pages](#custom-error-pages)
  - [Sending Error by Email](#sending-error-by-email)
  - [Logging to a File](#logging-to-a-file)
  - [Fixing the Duplicate Username Bug](#fixing-the-duplicate-username-bug)
- [Chapter8:Followers](#chapter8followers)
  - [Database Relationships Revisited](#database-relationships-revisited)
    - [One-to-Many](#one-to-many)
    - [Many-to-Many](#many-to-many)
    - [Many-to-One and One-to-One](#many-to-one-and-one-to-one)
  - [Representing Followers](#representing-followers)
  - [Database Model Representation](#database-model-representation)
  - [Adding and Removing "follows"](#adding-and-removing-follows)
  - [Obtaining the Posts from Followed Users](#obtaining-the-posts-from-followed-users)
    - [Joins](#joins)
    - [Filters](#filters)
    - [Sorting](#sorting)
  - [Combining Own and Followed Posts](#combining-own-and-followed-posts)
  - [Unit Testing the User Model](#unit-testing-the-user-model)
  - [Integrating Followers with the Application](#integrating-followers-with-the-application)
- [Chapter9: Pagination](#chapter9-pagination)
  - [Submission of Blog Posts](#submission-of-blog-posts)
  - [Displaying Bolg Posts](#displaying-bolg-posts)
  - [Making It Easier to Find Users to Follow](#making-it-easier-to-find-users-to-follow)
  - [Pagination of Blog Posts](#pagination-of-blog-posts)
  - [Page Navigation](#page-navigation)
  - [Pagination in the User Profile Page](#pagination-in-the-user-profile-page)
- [Chapter10: Email Support](#chapter10-email-support)
  - [Introduction to Flask-Mail](#introduction-to-flask-mail)
  - [Flask-Mail Usage](#flask-mail-usage)
  - [A Simple Email Framework](#a-simple-email-framework)
  - [Requesting a Password Reset](#requesting-a-password-reset)
  - [Password Reset Tokens](#password-reset-tokens)
  - [Sending a Password Reset Email](#sending-a-password-reset-email)
  - [Resetting a User Password](#resetting-a-user-password)
  - [Asynchronous Emails](#asynchronous-emails)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->



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

# chapter 4:Database

## Databases in Flask

我相信你已经听说了，Flask不支持原生的数据库。这是Flask有意不涉及的众多领域之一，这很好，因为你可以自由选择最适合你的应用程序的数据库，而不是被迫适应一个数据库。

Python中的数据库有很多选择，其中很多都有Flask扩展，可以与应用程序更好地整合。这些数据库可以分为两大类，一类是遵循关系模型的，另一类是不遵循关系模型的。后一组通常被称为NoSQL，表明它们没有实现流行的关系查询语言SQL。虽然这两组中都有很好的数据库产品，但我的看法是，关系型数据库更适合于有结构化数据的应用，如用户列表、博客文章等，而NoSQL数据库往往更适合于结构不太明确的数据。这个应用和其他大多数应用一样，可以使用两种类型的数据库来实现，但由于上述原因，我打算使用关系型数据库。

在第三章中，我向你展示了第一个Flask扩展。在这一章中，我将会使用另外两个。第一个是Flask-SQLAlchemy，这个扩展为流行的SQLAlchemy包提供了一个Flask友好的包装，它是一个对象关系映射器或ORM。ORM允许应用程序使用高级实体来管理数据库，如类、对象和方法，而不是表和SQL。ORM的工作是将高层操作转换为数据库命令。

SQLAlchemy的好处是，它不是针对一个，而是针对许多关系型数据库的ORM。SQLAlchemy支持一长串的数据库引擎，包括流行的MySQL、PostgreSQL和SQLite。这是非常强大的，因为你可以使用一个不需要服务器的简单SQLite数据库进行开发，然后当需要在生产服务器上部署应用程序时，你可以选择一个更强大的MySQL或PostgreSQL服务器，而不必改变你的应用程序。

要在你的虚拟环境中安装Flask-SQLAlchemy，首先确保你已经激活它，然后运行：

```Shell
(venv) $ pip install flask-sqlalchemy
```

## Database Migrations

我所看到的大多数数据库教程都涉及数据库的创建和使用，但没有充分解决随着应用需求的变化或增长而对现有数据库进行更新的问题。这很难，因为关系型数据库是以结构化数据为中心的，所以当结构发生变化时，数据库中已有的数据需要迁移到修改后的结构中。

本章要介绍的第二个扩展是Flask-Migrate，这实际上是一个由你自己创造的扩展。这个扩展是Alembic的Flask包装器，Alembic是SQLAlchemy的数据库迁移框架。使用数据库迁移会增加一些工作来启动数据库，但这对于在未来对数据库进行修改的强大方式来说是一个很小的代价。

Flask-Migrate的安装过程与你见过的其他扩展类似:

```Shell
(venv) $ pip install flask-migrate
```

## Flask-SQLAlchemy Configuration

在开发过程中，我将使用一个SQLite数据库。SQLite数据库是开发小型应用程序最方便的选择，有时甚至是不小的应用程序，因为每个数据库都存储在磁盘上的一个文件中，不需要像MySQL和PostgreSQL那样运行数据库服务器。

我们有两个新的配置项要添加到配置文件中：

config.py: Flask-SQLAlchemy configuration
```python
import os
basedir = os.path.abspath(os.path.dirname(__file__))

class Config(object):
    # ...
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'sqlite:///' + os.path.join(basedir, 'app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

Flask-SQLAlchemy扩展从`SQLALCHEMY_DATABASE_URI`配置变量中获取应用程序的数据库位置。正如你在第3章中所回忆的，一般来说，从环境变量中设置配置是一个很好的做法，当环境没有定义该变量时，提供一个后备值(fallback)。在这种情况下，我从`DATABASE_URL`环境变量中获取数据库URL，如果这个变量没有被定义，我将配置一个名为 _app.db_ 的数据库，它位于应用程序的主目录中，该目录被存储在`basedir`变量中。

`SQLALCHEMY_TRACK_MODIFICATIONS`配置选项被设置为`False`，以禁用Flask-SQLAlchemy的一个我不需要的功能，即每次数据库即将发生变化时向应用程序发送一个信号。

在应用程序中，数据库将由数据库实例来表示。数据库迁移引擎也将有一个实例。这些都是需要在应用程序之后，在 _app/\_\_init\_\_.py_ 文件中创建的对象：

app/\_\_init\_\_.py: Flask-SQLAlchemy and Flask-Migrate initialization

```python
from flask import Flask
from config import Config
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

app = Flask(__name__)
app.config.from_object(Config)
db = SQLAlchemy(app)
migrate = Migrate(app, db)

from app import routes, models
```

我对init脚本做了三个改动。首先，我添加了一个代表数据库的`db`对象。然后我又添加了一个代表迁移引擎的对象。希望你能在如何使用Flask扩展方面看到一个模式。大多数扩展的初始化都是这两个。最后，我在底部导入了一个名为`models`的新模块。这个模块将定义数据库的结构。

## Database Models

让我们将被存储在数据库中的数据将由一个类的集合来表示，通常称为数据库模型。SQLAlchemy中的ORM层将进行必要的翻译，将从这些类中创建的对象映射到适当的数据库表中的行。

让我们首先创建一个代表用户的模型。使用[WWW SQL Designer](https://ondras.zarovi.cz/sql/demo/)工具，我做了下面的图来表示我们想在用户表中使用的数据：

![data](/assets/img/data.png "data")

`id`字段通常在所有模型中都有，并被用作主键。数据库中的每个用户将被分配一个唯一的id值，存储在这个字段中。在大多数情况下，主键是由数据库自动分配的，所以我只需要提供标记为主键的`id`字段。

`username`、`email`和`password_hash`字段被定义为字符串（或数据库术语中的`VARCHAR`），它们的最大长度被指定，以便数据库能够优化空间使用。虽然`username`和`email`字段是不言自明的，但`password_hash`字段值得注意。我想确保我正在构建的应用程序采用了安全的最佳做法，为此我不会在数据库中存储用户密码。存储密码的问题是，如果数据库一旦被破坏，攻击者就可以获得密码，这对用户来说可能是毁灭性的。我不打算直接写密码，而是写密码哈希值，这样可以大大改善安全性。这将是另一章的主题，所以现在不要太担心。

所以现在我知道了我对用户表的要求，我可以在新的 _app/models.py_ 模块中将其转化为代码：

app/models.py: User database model

```python
from app import db

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), index=True, unique=True)
    email = db.Column(db.String(120), index=True, unique=True)
    password_hash = db.Column(db.String(128))

    def __repr__(self):
        return '<User {}>'.format(self.username)
```

上面创建的`User`类继承自`db.Model`，这是Flask-SQLAlchemy中所有模型的基类。这个类定义了几个字段作为类变量。字段被创建为`db.Column`类的实例，该类将字段类型作为一个参数，加上其他可选参数，例如，允许我指出哪些字段是唯一的和有索引的，这对数据库搜索的效率很重要。

\_\_repr\_\_ 方法告诉 Python 如何打印这个类的对象，这对调试是很有用的。你可以在下面的 Python 解释器会话中看到 \_\_repr\_\_() 方法的作用：

```Shell
>>> from app.models import User
>>> u = User(username='susan', email='susan@example.com')
>>> u
<User susan>
```

## Creating The Migration Repository

上一节中创建的模型类定义了这个应用程序的初始数据库结构（或模式(schema)）。但随着应用程序的不断发展，我很可能需要对该结构进行修改，比如添加新的东西，有时还要修改或删除项目。Alembic（Flask-Migrate使用的迁移框架）将以一种不需要在每次需要改变时从头开始重新创建数据库的方式来进行这些模式的改变。

为了完成这项看似困难的任务，Alembic维护了一个迁移库，这是一个存储迁移脚本的目录。每次对数据库模式进行更改时，都会在存储库中添加一个迁移脚本，其中包括更改的细节。为了将迁移应用到数据库中，这些迁移脚本按照创建的顺序被执行。

Flask-Migrate通过`flask`命令公开其命令。你已经看到了`flask run`，它是Flask原生的一个子命令。`flask db`子命令是由Flask-Migrate添加的，用来管理与数据库迁移有关的一切。所以，让我们通过运行`flask db init`来创建微博的迁移库：

```Shell
(venv) $ flask db init
  Creating directory /home/miguel/microblog/migrations ... done
  Creating directory /home/miguel/microblog/migrations/versions ... done
  Generating /home/miguel/microblog/migrations/alembic.ini ... done
  Generating /home/miguel/microblog/migrations/env.py ... done
  Generating /home/miguel/microblog/migrations/README ... done
  Generating /home/miguel/microblog/migrations/script.py.mako ... done
  Please edit configuration/connection/logging settings in
  '/home/miguel/microblog/migrations/alembic.ini' before proceeding.
```

记住，`flask`命令依靠`FLASK_APP`环境变量来知道Flask应用程序的位置。对于这个应用程序，你要把`FLASK_APP`设置为`microblog.py`这个值，正如第1章中所讨论的。

运行这个命令后，你会发现一个新的migrations目录，里面有一些文件和一个版本子目录。从现在开始，所有这些文件都应该被视为你的项目的一部分，特别是应该和你的应用程序代码一起被添加到源代码控制中。

## The First Database Migration

有了迁移库，现在是时候创建第一个数据库迁移了，它将包括映射到用户数据库模型的`user`表。有两种方法来创建数据库迁移：手动或自动。要自动生成迁移，Alembic将数据库模型所定义的数据库模式与当前数据库中使用的实际数据库模式进行比较。然后，它在迁移脚本中加入必要的修改，以使数据库模式与应用模式相匹配。在这种情况下，由于没有以前的数据库，自动迁移会将整个`user`模型添加到迁移脚本中。`flask db migrate`子命令产生了这些自动迁移：

```Shell
(venv) $ flask db migrate -m "users table"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'user'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_user_email' on '['email']'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_user_username' on '['username']'
  Generating /home/miguel/microblog/migrations/versions/e517276bb1c2_users_table.py ... done
```

该命令的输出让你了解Alembic在迁移中包含的内容。前两行是信息性的，通常可以被忽略。然后它说它找到了一个用户表和两个索引。然后它告诉你它在哪里写了迁移脚本。`e517276bb1c2`代码是自动生成的用于迁移的唯一代码（对你来说会有所不同）。`-m`选项给出的注释是可选的，它为迁移添加了一个简短的描述性文本。

生成的迁移脚本现在是你项目的一部分，需要并入源代码控制。如果你想看看这个脚本的样子，欢迎你来检查。你会发现它有两个函数，叫做 `upgrade() `和` downgrade()`。`upgrade()`函数应用了迁移，而`downgrade()`函数则将其删除。这使得Alembic可以通过使用降级路径将数据库迁移到历史上的任何一点，甚至迁移到更早的版本。

`flask db migrate `命令不会对数据库进行任何更改，它只是生成了迁移脚本。要将这些变化应用到数据库中，必须使用`flask db upgrade`命令。

```Shell
(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> e517276bb1c2, users table
```

因为这个应用程序使用的是SQLite，`upgrade`命令会检测到数据库不存在，并会创建它（你会注意到在这个命令完成后会添加一个名为 _app.db_ 的文件，这就是SQLite数据库）。当使用MySQL和PostgreSQL等数据库服务器时，你必须在运行升级之前在数据库服务器中创建数据库。

请注意，Flask-SQLAlchemy默认对数据库表使用 "snake case "命名惯例。对于上面的`User`模型，数据库中相应的表将被命名为`user`。对于`AddressAndPhone`模型类，该表将被命名为`address_and_phone`。如果你喜欢选择你自己的表名，你可以在模型类中添加一个名为\_\_tablename\_\_的属性，设置为所需的字符串名称。

## Database Upgrade and Downgrade Workflow

在这一点上，应用程序还处于起步阶段，但讨论一下未来的数据库迁移策略并没有什么坏处。想象一下，你在你的开发机器上有你的应用程序，也有一个副本被部署到一个在线并正在使用的生产服务器。

比方说，在你的应用程序的下一个版本中，你必须对你的模型进行修改，例如需要添加一个新的表。如果没有迁移，你就需要想办法改变数据库的模式，既要在开发机上改变，又要在服务器上改变，这可能是一件很麻烦的事情。

但是有了数据库迁移支持，在你修改应用程序中的模型后，你会生成一个新的迁移脚本（`flask db migrate`），你可能会审查它，以确保自动生成的东西是正确的，然后将这些变化应用到你的开发数据库（`flask db upgrade`）。你将把迁移脚本添加到源码控制中，并提交它。

当你准备将新版本的应用程序发布到生产服务器上时，你需要做的就是抓取应用程序的更新版本，其中将包括新的迁移脚本，并运行`flask db upgrade`。Alembic会检测到生产数据库没有更新到最新版本的模式，并运行所有在上一次发布后创建的新迁移脚本。

正如我之前提到的，你还有一个`flask db downgrade`命令，可以撤销上一次的迁移。虽然你在生产系统中不太可能需要这个选项，但你可能会发现它在开发过程中非常有用。你可能已经生成了一个迁移脚本并应用了它，但却发现你所做的改变并不完全是你所需要的。在这种情况下，你可以将数据库降级，删除迁移脚本，然后再生成一个新的脚本来替代它。

## Database Relationships

关系型数据库擅长于存储数据项之间的关系。考虑到一个用户写了一篇博客文章的情况。该用户在`users`表中有一条记录，而该帖子在`posts`表中有一条记录。记录谁写了某篇文章的最有效的方法是将这两条相关的记录联系起来。

一旦一个用户和一个帖子之间建立了联系，数据库就可以回答关于这个联系的查询。最小的一个例子是当你有一篇博客文章，需要知道是哪个用户写的。一个更复杂的查询是与此相反的。如果你有一个用户，你可能想知道这个用户写的所有帖子。Flask-SQLAlchemy将有助于这两种类型的查询。

让我们扩展数据库来存储博客文章，看看关系的作用。下面是一个新的帖子表的模式：

![ch04-users-posts](/assets/img/ch04-users-posts.png "ch04-users-posts")


`posts`表将有必要的`id`、帖子的`body`和`timestamp`。但除了这些预期的字段外，我还要添加一个`user_id`字段，它将帖子与作者联系起来。你已经看到，所有的用户都有一个id主键，它是唯一的。将一篇博文与撰写博文的用户联系起来的方法是添加一个对用户`id`的引用，而这正是`user_id`字段的作用。这个`user_id`字段被称为外键(_foreign key_)。上面的数据库图显示外键是该字段和它所指向的表的`id`字段之间的联系。这种关系被称为一对多(_one-to-many_)，因为 "一个 "用户写了 "许多 "帖子。

修改后的app/models.py如下所示：

app/models.py: Posts database table and relationship

```python
from datetime import datetime
from app import db

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), index=True, unique=True)
    email = db.Column(db.String(120), index=True, unique=True)
    password_hash = db.Column(db.String(128))
    posts = db.relationship('Post', backref='author', lazy='dynamic')

    def __repr__(self):
        return '<User {}>'.format(self.username)

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    body = db.Column(db.String(140))
    timestamp = db.Column(db.DateTime, index=True, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))

    def __repr__(self):
        return '<Post {}>'.format(self.body)
```

新的`Post`类将代表用户写的博客文章。`timestamp`字段将被索引，如果你想按时间顺序检索帖子，这很有用。我还添加了一个`default`，并传递了`datetime.utcnow`函数。当你传递一个函数作为默认参数时，SQLAlchemy将把字段设置为调用该函数的值（注意，我没有在`utcnow`后面加上`（）`，所以我传递的是函数本身，而不是调用函数的结果）。一般来说，你会希望在服务器应用程序中使用UTC日期和时间。这可以确保你使用统一的时间戳，而不管用户位于何处。这些时间戳在显示时将被转换为用户的当地时间。

`user_id`字段被初始化为`user.id`的外键，这意味着它引用了用户表中的一个`id`值。在这个引用中，`user`部分是模型所代表的数据库表的名称。不幸的是，在某些情况下，比如在`db.relationship()`调用中，模型是通过模型类来引用的，通常以大写字母开头，而在其他情况下，比如这个`db.ForeignKey()`声明中，模型是通过其数据库表名来指定的，对于这些名称，SQLAlchemy自动使用小写字符，并对多词模型名称使用蛇形命名法。

`User`类有一个新的`posts`字段，该字段使用`db.relationship`进行初始化。这不是一个实际的数据库字段，而是用户和帖子之间关系的高层视图，因此它不在数据库图表中。对于一对多关系，`db.relationship`字段通常在“一”的一侧定义，并被用作访问“多”的一种方便方式。例如，如果我有一个存储在`u`中的用户，表达式`u.posts`将运行一个数据库查询，返回该用户写的所有帖子。`db.relationship`的第一个参数是表示关系“多”一侧的模型类。如果模型在模块中稍后定义，则可以将此参数提供为类名的字符串。`backref`参数定义了一个字段的名称，该字段将添加到“多”类的对象中，指向“一”对象。这将添加一个`post.author`表达式，该表达式将在给定帖子的情况下返回用户。`lazy`参数定义了如何发出关系的数据库查询，这是我稍后将讨论的内容。如果这些细节现在不太明白，不要担心，我将在本文末尾给你示例。

由于我对应用模型有更新，所以需要生成一个新的数据库迁移：

```Shell
(venv) $ flask db migrate -m "posts table"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'post'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_post_timestamp' on
'['timestamp']'
  Generating /home/miguel/microblog/migrations/versions/780739b227a7_posts_table.py ... done
```

同时迁移需要应用于数据库：

```Shell
(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade e517276bb1c2 -> 780739b227a7, posts table
```

如果你在源码控制中存储你的项目，也要记得将新的迁移脚本添加到其中。

## Playing with the Database

我让您经历了一个漫长的过程来定义数据库，但我还没有向您展示一切是如何工作的。由于应用程序尚未具有任何数据库逻辑，因此让我们在Python解释器中玩一下数据库，以熟悉它。通过在终端上运行`python`启动Python。在启动解释器之前，请确保您的虚拟环境已被激活。

一旦进入Python提示符，让我们导入应用程序、数据库实例和模型:

```python
>>> from app import app, db
>>> from app.models import User, Post
```
下一步有点奇怪。为了让 Flask 及其扩展程序能够访问 Flask 应用程序，而无需将 `app` 作为参数传递到每个函数中，必须创建并推送应用程序上下文。应用程序上下文将在本教程的后面更详细地介绍，因此现在，请在您的 Python shell 会话中键入以下代码：
```python
>>> app.app_context().push()
```
这将推送应用程序上下文，以便访问应用程序和数据库实例。
接下来，创建一个新用户：
```python
>>> u = User(username='john', email='john@example.com')
>>> db.session.add(u)
>>> db.session.commit()
```
这将在数据库中创建一个新用户，并将其添加到会话中。最后，会话被提交以将更改保存到数据库。

对数据库的更改是在数据库会话的上下文中完成的，可以通过 `db.session` 访问。多个更改可以在一个会话中累积，一旦所有更改都已注册，就可以发出单个 `db.session.commit()`，将所有更改原子地写入。如果在会话期间的任何时候出现错误，则调用 `db.session.rollback()` 将中止会话并删除其中存储的任何更改。重要的是要记住，只有在使用 `db.session.commit()` 发出提交时，更改才会写入数据库。会话保证数据库永远不会处于不一致状态。

你是否想知道所有这些数据库操作如何知道要使用哪个数据库？上面推送的应用程序上下文允许 Flask-SQLAlchemy 访问 Flask 应用程序实例` app`，而无需将其作为参数接收。该扩展程序在 `app.config` 字典中查找 `SQLALCHEMY_DATABASE_URI` 条目，其中包含数据库的 URL。

让我们添加另一个用户：
```python
>>> u = User(username='susan', email='susan@example.com')
>>> db.session.add(u)
>>> db.session.commit()
```
这将创建一个名为 susan 的新用户，并将其添加到数据库中。

数据库可以回答一个查询，返回所有用户:
```python
>>> users = User.query.all()
>>> users
[<User john>, <User susan>]
>>> for u in users:
...     print(u.id, u.username)
...
1 john
2 susan
```
这将查询所有用户并将它们存储在一个名为 users 的列表中，最后将其打印出来

所有模型都有一个`query`属性，它是运行数据库查询的入口点。最基本的查询是返回该类的所有元素的查询，适当地命名为 `all()`。请注意，当添加这些用户时，id 字段被自动设置为 1 和 2。

这里有另一种查询方式。如果你知道一个用户的 id，你可以按如下方式检索该用户：

```python
user = User.query.get(1)
print(user.username)
```

这将检索具有 id 为 1 的用户，并将其打印出来。

现在让我们添加一篇博客文章：
```python
>>> u = User.query.get(1)
>>> p = Post(body='my first post!', author=u)
>>> db.session.add(p)
>>> db.session.commit()
```

我不需要为`timestamp`字段设置值，因为该字段有一个默认值，你可以在模型定义中看到。那么`user_id`字段呢？记住我在`User`类中创建的`db.relationship`会为用户添加一个`posts`属性，并为帖子添加一个`author`属性。我使用虚拟字段`author`来为帖子分配作者，而不必处理用户ID。 SQLAlchemy在这方面非常出色，因为它提供了关系和外键的高级抽象。

为了完成本节，让我们再看一些数据库查询：
```python
>>> # get all posts written by a user
>>> u = User.query.get(1)
>>> u
<User john>
>>> posts = u.posts.all()
>>> posts
[<Post my first post!>]

>>> # same, but with a user that has no posts
>>> u = User.query.get(2)
>>> u
<User susan>
>>> u.posts.all()
[]

>>> # print post author and body for all posts
>>> posts = Post.query.all()
>>> for p in posts:
...     print(p.id, p.author.username, p.body)
...
1 john my first post!

# get all users in reverse alphabetical order
>>> User.query.order_by(User.username.desc()).all()
[<User susan>, <User john>]
```

[Flask-SQLAlchemy](http://packages.python.org/Flask-SQLAlchemy/index.html) 的文档是学习数据库查询的众多选项的最佳场所。

为了完成本节，让我们删除上面创建的测试用户和帖子，以便数据库干净并准备好进入下一章节：
```python
>>> users = User.query.all()
>>> for u in users:
...     db.session.delete(u)
...
>>> posts = Post.query.all()
>>> for p in posts:
...     db.session.delete(p)
...
>>> db.session.commit()
```

## Shell Context

上一节开始时你在Python解释器中运行了一些导入语句，记得吗？
```python
>>> from app import app, db
>>> from app.models import User, Post
>>> app.app_context().push()
```

在开发应用程序时，您经常需要在Python shell中测试各种功能，因此每次重复以上语句会变得很繁琐，所以现在是解决这个问题的好时机。

`flask shell`命令是`Flask`命令中另一个非常有用的工具。`shell`命令是Flask中实现的第二个“核心”命令，在`run`命令之后。该命令的目的是在应用程序上下文中启动一个Python解释器。这是什么意思？看下面的例子：

```Shell
(venv) $ python
>>> app
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'app' is not defined
>>>

(venv) $ flask shell
>>> app
<Flask 'app'>
```

使用普通的解释器会话，`app` 符号不会自动导入，除非显式导入它。但使用 `flask shell` 命令时，它会预先导入应用程序实例。 `flask shell` 的好处不仅在于它预先导入了 `app`，还可以配置一个 "shell context"，即预导入的其他符号的列表。

以下函数在 microblog.py 中创建了一个 shell context，将数据库实例和模型添加到 shell 会话中:

```python
from app import app, db
from app.models import User, Post

@app.shell_context_processor
def make_shell_context():
    return {'db': db, 'User': User, 'Post': Post}
```
`app.shell_context_processor`装饰器将该函数注册为shell context函数。当`flask shell`命令运行时，它将调用这个函数并在shell会话中注册由它返回的项目。该函数返回的是一个字典而不是一个列表，原因是对于每个项目，你还必须提供一个名字，它将在 shell 中被引用，这个名字由字典的键值给出。

在你添加了 shell 上下文处理器函数后，你可以使用数据库实体，而不必导入它们：
```Shell
(venv) $ flask shell
>>> db
<SQLAlchemy engine=sqlite:////Users/migu7781/Documents/dev/flask/microblog2/app.db>
>>> User
<class 'app.models.User'>
>>> Post
<class 'app.models.Post'>
```

如果你尝试上述方法，并在试图访问`db`、`User`和`Post`时得到`NameError`异常，那么`make_shell_context()`函数就没有被Flask注册。最有可能的原因是，你没有在环境中设置F`LASK_APP=microblog.py`。在这种情况下，请回到第1章，回顾一下如何设置`FLASK_APP`环境变量。如果你在打开新的终端窗口时经常忘记设置这个变量，你可以考虑在你的项目中添加一个 _.flaskenv_ 文件，如该章末尾所述。

# Chapter 5:User Logins

## Password Hashing

在第4章中，用户模型添加了一个`password_hash`字段，到目前为止尚未使用。此字段的目的是保存用户密码的哈希值，将用于验证用户在登录过程中输入的密码。密码哈希是一个复杂的主题，应该留给安全专家，但有几个易于使用的库实现了所有这些逻辑，可以从应用程序中简单地调用。

其中一个实现密码哈希的包是Werkzeug，你可能已经在安装Flask时看到它的输出中引用了它，因为它是Flask的核心依赖项之一。由于它是一个依赖项，[Werkzeug](http://werkzeug.pocoo.org/)已经安装在您的虚拟环境中。下面的Python shell会话演示了如何哈希密码:

```Shell
>>> from werkzeug.security import generate_password_hash
>>> hash = generate_password_hash('foobar')
>>> hash
'pbkdf2:sha256:50000$vT9fkZM8$04dfa35c6476acf7e788a1b5b3c35e217c78dc04539d295f011f01f18cd2'
```

在这个例子中，密码"`foobar`"被通过一系列的加密操作转化成一个长字符串，这些操作没有已知的反向操作，这意味着获得散列密码的人将无法使用它来获取原始密码。为了进一步保护，如果你多次哈希相同的密码，你会得到不同的结果，这使得通过查看哈希值来确定两个用户是否具有相同的密码变得不可能。

验证过程使用 Werkzeug 的第二个函数，如下所示：

```Shell
>>> from werkzeug.security import check_password_hash
>>> check_password_hash(hash, 'foobar')
True
>>> check_password_hash(hash, 'barfoo')
False
```

验证函数接受之前生成的密码哈希和用户在登录时输入的密码。如果用户提供的密码与哈希相匹配，则该函数返回`True`，否则返回`False`。

整个密码哈希逻辑可以在用户模型中实现为两个新方法：

app/models.py: Password hashing and verification

```python
from werkzeug.security import generate_password_hash, check_password_hash

# ...

class User(db.Model):
    # ...

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)
```

有了这两个方法，用户对象现在可以进行安全的密码验证，而无需存储原始密码。以下是这些新方法的示例用法：

```Shell
>>> u = User(username='susan', email='susan@example.com')
>>> u.set_password('mypassword')
>>> u.check_password('anotherpassword')
False
>>> u.check_password('mypassword')
True
```

## Introduction to Flask-Login

在本章中，我将向您介绍一个非常受欢迎的Flask扩展，称为[Flask-Login](https://flask-login.readthedocs.io/)。该扩展管理用户的登录状态，例如，用户可以登录到应用程序，然后在应用程序记住用户已登录的情况下导航到不同的页面。它还提供了“记住我”功能，允许用户在关闭浏览器窗口后仍然保持登录状态。为了准备开始本章，您可以在虚拟环境中安装Flask-Login：

```Shell
(venv) $ pip install flask-login
```
和其他扩展一样，Flask-Login需要在 _app/\_\_init\_\_.py_ 中的应用实例之后创建和初始化。以下是初始化该扩展的代码示例：

app/\_\_init\_\_.py: Flask-Login initialization

```python
# ...
from flask_login import LoginManager

app = Flask(__name__)
# ...
login = LoginManager(app)

# ...
```

## Preparing The User Model for Flask-Login

Flask-Login扩展与应用程序的用户模型一起工作，并期望在其中实现某些属性和方法。这种方法很好，因为只要将这些所需的项添加到模型中，Flask-Login就没有任何其他要求，因此，例如，它可以使用基于任何数据库系统的用户模型。

下面列出了四个必需的项：

* `is_authenticated`：如果用户具有有效凭据，则为`True`，否则为`False`的属性。
* `is_active`：如果用户的帐户处于活动状态，则为`True`，否则为`False`的属性。
* `is_anonymous`：常规用户为`False`，而特殊的匿名用户为`True`的属性。
* `get_id()`：返回用户的唯一标识符作为字符串（如果使用Python 2，则为unicode）的方法。

我可以很容易地实现这四个方法，但由于这些实现是相当通用的，Flask-Login提供了一个称为`UserMixin`的mixin类，其中包括适用于大多数用户模型类的通用实现。以下是将mixin类添加到模型的方式：

app/models.py: Flask-Login user mixin class
```python

# ...
from flask_login import UserMixin

class User(UserMixin, db.Model):
    # ...
```

## User Loader Function

Flask-Login通过将用户的唯一标识符存储在Flask的用户会话(user session)中来跟踪已登录的用户，这是为每个连接到应用程序的用户分配的存储空间。每当已登录的用户导航到新页面时，Flask-Login会从会话中检索用户的ID，然后将该用户加载到内存中。

由于Flask-Login对数据库一无所知，它需要应用程序的帮助来加载用户。因此，该扩展期望应用程序将配置一个用户加载函数，该函数可以根据ID调用以加载用户。可以将此函数添加到 _app/models.py_ 模块中：

_app/models.py_: Flask-Login user loader function

```python
from app import login
# ...

@login.user_loader
def load_user(id):
    return User.query.get(int(id)
```

用户加载器使用`@login.user_loader`装饰器在Flask-Login中进行注册。Flask-Login传递给函数的`id`将作为参数成为字符串，因此使用数字ID的数据库需要像上面一样将字符串转换为整数。

## Logging Users In

让我们回顾一下登录视图函数，你可能还记得，它实现了一个虚假的登录功能，只是发出一个 `flash()` 消息。现在应用程序可以访问用户数据库并知道如何生成和验证密码哈希值，因此可以完成此视图函数的编写。

_app/routes.py_: Login view function logic

```python
# ...
from flask_login import current_user, login_user
from app.models import User

# ...

@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        if user is None or not user.check_password(form.password.data):
            flash('Invalid username or password')
            return redirect(url_for('login'))
        login_user(user, remember=form.remember_me.data)
        return redirect(url_for('index'))
    return render_template('login.html', title='Sign In', form=form)
```

`login()` 函数中的前两行处理了一种奇怪的情况。想象一下，如果用户已经登录并导航到应用程序的 _/login_ URL，那么这显然是个错误，我不想允许这种情况。`current_user` 变量来自 Flask-Login，可以在处理期间随时使用它获取代表请求客户端的用户对象。此变量的值可以是来自数据库的用户对象（Flask-Login 通过我提供的用户加载器回调函数来读取它），也可以是特殊的匿名用户对象（如果用户尚未登录）。还记得 Flask-Login 在用户对象中所需的属性吗？其中之一是 `is_authenticated`，这在检查用户是否已登录时非常有用。当用户已经登录时，我只需要重定向到主页。

我可以用之前使用的 `flash()` 调用来代替，现在我可以真正让用户登录。第一步是从数据库中加载用户。用户名是通过表单提交的，所以我可以使用该用户名查询数据库以查找用户。为此，我使用 SQLAlchemy 查询对象的 `filter_by()` 方法。`filter_by()` 的结果是一个仅包括具有匹配用户名的对象的查询。由于我知道只会有一个或零个结果，所以我通过调用 `first()` 完成查询，如果该用户对象存在，则返回该用户对象，否则返回 `None`。在第4章中，你已经看到当在查询中调用 `all()` 方法时，查询将执行并返回与该查询匹配的所有结果列表。`first()` 方法是另一种常用的执行查询的方式，当你只需要一个结果时就可以使用它。

如果用户名匹配成功，我可以接下来检查表单中提供的密码是否有效。这是通过调用我上面定义的`check_password()`方法来完成的。这将使用与用户存储的密码哈希值来确定表单中输入的密码是否与哈希值匹配。因此，现在我有两种可能的错误情况：用户名无效或密码不正确。在任何一种情况下，我都会显示一条消息，并重定向回登录提示，以便用户可以再次尝试。

如果用户名和密码都正确，那么我调用Flask-Login提供的`login_user()`函数。这个函数将把用户注册为已登录，这意味着用户导航到的任何未来页面都将使用`current_user`变量设置为该用户。

为了完成登录过程，我只需将新登录的用户重定向到索引页面。

## Logging Users Out

我知道我还需要给用户提供退出应用程序的选项。这可以使用Flask-Login的logout_user()函数完成。下面是退出视图函数的代码：

_app/routes.py_: Logout view function

```python
# ...
from flask_login import logout_user

# ...

@app.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('index'))
```

为了向用户公开此链接，我可以在用户登录后使导航栏中的登录链接自动切换为注销链接。这可以在base.html模板中使用条件语句完成：

_app/templates/base.html_: Conditional login and logout links

```html
<div>
    Microblog:
    <a href="{{ url_for('index') }}">Home</a>
    {% if current_user.is_anonymous %}
    <a href="{{ url_for('login') }}">Login</a>
    {% else %}
    <a href="{{ url_for('logout') }}">Logout</a>
    {% endif %}
</div>
```

`is_anonymous`属性是Flask-Login通过`UserMixin`类添加到用户对象中的属性之一。`current_user.is_anonymous`表达式只有在用户未登录时才为`True`。

## Requiring Users To Login

Flask-Login提供了一个非常有用的功能，可以在用户查看应用程序的某些页面之前强制他们先进行登录。如果未登录的用户尝试查看受保护的页面，Flask-Login将自动将用户重定向到登录表单，并在完成登录过程后才将其重定向回用户想要查看的页面。

为了实现这个功能，Flask-Login需要知道处理登录的视图函数是什么。可以在 _app/\_\_init\_\_.py_ 中添加此功能：

```python
# ...
login = LoginManager(app)
login.login_view = 'login'
```

上面的 `'login'` 值是登录视图的函数名（或端点名）。换句话说，它是您在 `url_for()` 调用中使用的名称以获取 URL。

Flask-Login使用名为`@login_required`的装饰器来保护视图函数免受匿名用户的访问。当你在一个使用了Flask的`@app.route`装饰器的视图函数下方添加了这个装饰器后，这个函数就变成了受保护的，不允许未经过身份验证的用户访问。以下是如何将装饰器应用于应用程序的index视图函数：

_app/routes.py_: @login\_required decorator

```python
from flask_login import login_required

@app.route('/')
@app.route('/index')
@login_required
def index():
    # ...
```

剩下的就是实现从成功登录到用户想要访问的页面的重定向。当一个没有登录的用户访问一个用`@login_required`装饰器保护的视图函数时，装饰器将重定向到登录页面，但它将在这个重定向中包含一些额外的信息，以便应用程序能够返回到第一个页面。例如，如果用户导航到 _/index_ ，`@login_required`装饰器将拦截该请求并响应重定向到 _/login_ ，但它将在这个URL上添加一个查询字符串参数，使完整的重定向URL _/login?next=/index_。`next`查询字符串参数被设置为原始的URL，所以应用程序可以在登录后使用它来重定向回来。

下面是一个代码片段，显示了如何读取和处理`next`查询字符串参数：

_app/routes.py_: Redirect to "next" page

```python
from flask import request
from werkzeug.urls import url_parse

@app.route('/login', methods=['GET', 'POST'])
def login():
    # ...
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        if user is None or not user.check_password(form.password.data):
            flash('Invalid username or password')
            return redirect(url_for('login'))
        login_user(user, remember=form.remember_me.data)
        next_page = request.args.get('next')
        if not next_page or url_parse(next_page).netloc != '':
            next_page = url_for('index')
        return redirect(next_page)
    # ...
```

就在用户通过调用Flask-Login的`login_user()`函数登录后，获得了`next`查询字符串参数的值。Flask提供了一个`request`变量，包含了客户端随请求发送的所有信息。特别是，`request.args`属性以友好的字典格式展示了查询字符串的内容。实际上，有三种可能的情况需要考虑，以确定在成功登录后重定向到哪里：

* 如果登录URL没有`next`参数，那么用户将被重定向到索引页。
* 如果登录的URL包括一个被设置为相对路径的`next`参数（或者换句话说，一个没有域名部分的URL），那么用户将被重定向到该URL。
* 如果登录的URL包括一个`next`参数，该参数被设置为一个包括域名的完整URL，那么用户将被重定向到索引页。

第一和第二种情况是不言自明的。第三种情况是为了使应用程序更加安全。攻击者可以在下一个参数中插入一个恶意网站的URL，所以应用程序只在URL是相对的情况下重定向，这样可以确保重定向停留在与应用程序相同的网站内。为了确定URL是相对的还是绝对的，我用Werkzeug的`url_parse()`函数解析(parse)它，然后检查`netloc`组件是否被设置。

## Showing The Logged In User in Templates

你还记得在第二章中，在用户子系统建立之前，我创建了一个假用户来帮助我设计应用程序的主页吗？好了，现在应用程序有了真正的用户，所以我现在可以删除假用户，开始使用真正的用户。我可以在模板中使用Flask-Login的`current_user`，而不是假用户：

_app/templates/index.html_: Pass current user to template

```html
{% extends "base.html" %}

{% block content %}
    <h1>Hi, {{ current_user.username }}!</h1>
    {% for post in posts %}
    <div><p>{{ post.author.username }} says: <b>{{ post.body }}</b></p></div>
    {% endfor %}
{% endblock %}
```
而且我可以在视图函数中删除用户模板参数：
_app/routes.py_: Do not pass user to template anymore

```python
@app.route('/')
@app.route('/index')
@login_required
def index():
    # ...
    return render_template("index.html", title='Home Page', posts=posts)
```

这是一个测试登录和注销功能如何工作的好时机。由于仍然没有用户注册，向数据库添加用户的唯一方法是通过Python shell来完成，所以运行`flask shell`并输入以下命令来注册一个用户：

```Shell
>>> u = User(username='susan', email='susan@example.com')
>>> u.set_password('cat')
>>> db.session.add(u)
>>> db.session.commit()
```

如果你启动应用程序并进入应用程序的/或/index URL，你将立即被重定向到登录页面，在你使用你添加到数据库的用户的凭证登录后，你将返回到原始页面，在其中你将看到一个个性化的问候语。

## User Registration

本章中我要建立的最后一个功能是一个注册表单，这样用户就可以通过一个web表单进行自我注册。让我们首先在 _app/forms.py_ 中创建Web表单类：

_app/forms.py_: User registration form

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import ValidationError, DataRequired, Email, EqualTo
from app.models import User

# ...

class RegistrationForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired()])
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired()])
    password2 = PasswordField(
        'Repeat Password', validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Register')

    def validate_username(self, username):
        user = User.query.filter_by(username=username.data).first()
        if user is not None:
            raise ValidationError('Please use a different username.')

    def validate_email(self, email):
        user = User.query.filter_by(email=email.data).first()
        if user is not None:
            raise ValidationError('Please use a different email address.')
```

在这个新的表单中，有几件有趣的事情与验证有关。首先，对于`email`字段，我在`DataRequired`之后添加了第二个验证器，叫做`Email`。这是另一个WTForms自带的验证器，它将确保用户在这个字段中输入的内容符合电子邮件地址的结构。

WTForms的`Email()`验证器需要安装一个外部依赖：

```Shell
(venv) $ pip install email-validator
```

由于这是一个注册表格，习惯上要求用户输入两次密码以减少打错的风险。出于这个原因，我设置了`password`和`password2`字段。第二个密码字段使用另一个名为EqualTo的验证器，它将确保其值与第一个密码字段的值相同。

当你添加任何与模式`validate_<field_name>`相匹配的方法时，WTForms将这些方法作为自定义验证器，并在原有验证器的基础上调用它们。我在这个类中为`username`和`email`字段添加了两个这样的方法。在这种情况下，我想确保用户输入的用户名和电子邮件地址不在数据库中，所以这两个方法发出数据库查询，期望没有结果。在有结果的情况下，通过引发一个`ValidationError`类型的异常来触发验证错误。异常中作为参数的信息将是显示在字段旁边供用户查看的信息。

为了在网页上显示这个表单，我需要有一个HTML模板，我将把它存放在 _app/templates/register.html_ 文件中。这个模板的构造与登录表单的模板类似：

_app/templates/register.html_: Registration template

```html
{% extends "base.html" %}

{% block content %}
    <h1>Register</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.username.label }}<br>
            {{ form.username(size=32) }}<br>
            {% for error in form.username.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.email.label }}<br>
            {{ form.email(size=64) }}<br>
            {% for error in form.email.errors %}
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
        <p>
            {{ form.password2.label }}<br>
            {{ form.password2(size=32) }}<br>
            {% for error in form.password2.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

登录表格模板需要一个链接，将新用户发送到注册表格，就在表格的下面：

_app/templates/login.html_: Link to registration page

```html
    <p>New User? <a href="{{ url_for('register') }}">Click to Register!</a></p>
```

最后，我需要在 _app/routes.py_ 中编写处理用户注册的视图函数：

_app/routes.py_: User registration view function

```python
from app import db
from app.forms import RegistrationForm

# ...

@app.route('/register', methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    form = RegistrationForm()
    if form.validate_on_submit():
        user = User(username=form.username.data, email=form.email.data)
        user.set_password(form.password.data)
        db.session.add(user)
        db.session.commit()
        flash('Congratulations, you are now a registered user!')
        return redirect(url_for('login'))
    return render_template('register.html', title='Register', form=form)
```

而这个视图功能也应该大部分是不言自明的。我首先要确保调用这个路径的用户没有登录。这个表单的处理方式与登录时的处理方式相同。在`if validate_on_submit()`条件中完成的逻辑是用提供的用户名、电子邮件和密码创建一个新的用户，将其写入数据库，然后重定向到登录提示，这样用户就可以登录了。

![ch05-register-form](/assets/img/ch05-register-form.png "ch05-register-form")

有了这些变化，用户应该能够在这个应用程序上创建账户，并登录和退出。请确保你尝试我在注册表单中添加的所有验证功能，以更好地了解它们的工作原理。我将在未来的一章中重新审视用户验证子系统，以增加额外的功能，如允许用户在忘记密码时重置密码。但现在，这已经足够了，可以继续构建应用程序的其他区域。

# Chapter6:Profile Page and Avatars

这一章将专门讨论在应用程序中添加用户资料页。用户简介页是一个展示用户信息的页面，通常是由用户自己输入的信息。我将向你展示如何为所有用户动态地生成简介页，然后我将添加一个小型的简介编辑器，用户可以用它来输入自己的信息。

## User Profile Page

为了创建一个用户配置文件页面，让我们在应用程序中添加一个 _/user/<username>_ 路由。

_app/routes.py_: User profile view function

```python
@app.route('/user/<username>')
@login_required
def user(username):
    user = User.query.filter_by(username=username).first_or_404()
    posts = [
        {'author': user, 'body': 'Test post #1'},
        {'author': user, 'body': 'Test post #2'}
    ]
    return render_template('user.html', user=user, posts=posts
```

我用来声明这个视图函数的`@app.route`装饰器看起来与之前的有点不同。在这种情况下，我在其中有一个动态组件，它被表示为`<username>`URL组件，被`<`和`>`所包围。当路由有一个动态组件时，Flask会接受URL中那一部分的任何文本，并以实际文本为参数调用视图函数。例如，如果客户端浏览器请求URL `/user/susan`，视图函数将被调用，参数`username`设置为`'susan'`。这个视图函数只能被登录的用户访问，所以我加入了Flask-Login的`@login_required`装饰器。

这个视图函数的实现是相当简单的。我首先尝试通过用户名的查询从数据库中加载用户。你以前已经看到，如果你想得到所有的结果，可以通过调用`all()`来执行数据库查询；如果你想得到第一个结果，可以调用`first()`；如果没有结果，可以调用`None`。在这个视图函数中，我使用了`first()`的一个变体，叫做`first_or_404()`，当有结果时，它的工作方式与`first()`完全一样，但在没有结果的情况下，会自动向客户端发送一个[404错误](https://en.wikipedia.org/wiki/HTTP_404)。以这种方式执行查询，我省去了检查查询是否返回用户的麻烦，因为当用户名在数据库中不存在时，该函数不会返回，而会产生一个404异常。

如果数据库查询没有引发404错误，那就意味着找到了一个具有给定用户名的用户。接下来我为这个用户初始化一个假的帖子列表，最后渲染一个新的user.html模板，我把用户对象和帖子列表传给它。

user.html模板如下所示：

_app/templates/user.html_: User profile template

```html
{% extends "base.html" %}

{% block content %}
    <h1>User: {{ user.username }}</h1>
    <hr>
    {% for post in posts %}
    <p>
    {{ post.author.username }} says: <b>{{ post.body }}</b>
    </p>
    {% endfor %}
{% endblock %}
```

档案页现在已经完成了，但在网站的任何地方都没有它的链接。为了使用户更容易查看自己的资料，我打算在顶部的导航栏中添加一个链接：

_app/templates/base.html_: User profile template

```html
    <div>
      Microblog:
      <a href="{{ url_for('index') }}">Home</a>
      {% if current_user.is_anonymous %}
      <a href="{{ url_for('login') }}">Login</a>
      {% else %}
      <a href="{{ url_for('user', username=current_user.username) }}">Profile</a>
      <a href="{{ url_for('logout') }}">Logout</a>
      {% endif %}
    </div>
```

这里唯一有趣的变化是`url_for()`的调用，它被用来生成到个人资料页面的链接。由于用户资料查看函数需要一个动态参数，所以`url_for()`函数接收一个作为关键字参数的值。由于这是一个指向登录的用户配置文件的链接，我可以使用Flask-Login的`current_user`来生成正确的URL。

![ch06-user-profile](/assets/img/ch06-user-profile.png "ch06-user-profile")


现在就试一试这个应用程序。点击顶部的个人资料链接，你就可以进入你自己的用户页面。在这一点上，没有链接可以进入其他用户的资料页面，但如果你想访问这些页面，你可以在浏览器的地址栏中手动输入URL。例如，如果你有一个名为 "john "的用户在你的应用程序上注册，你可以通过在地址栏中输入 _http://localhost:5000/user/john_，查看相应的用户资料。

## Avatars

我相信你也同意，我刚刚建立的个人资料页面是非常无聊的。为了使它们更有趣，我打算增加用户的头像，但不是在服务器中处理可能有大量的上传图片，而是使用[Gravatar](http://gravatar.com/)服务来为所有用户提供图片。

Gravatar服务使用起来非常简单。要为一个给定的用户请求图像，需要一个格式为     _https://www.gravatar.com/avatar/<hash>_ 的URL，其中`<hash>`是用户的电子邮件地址的MD5哈希值。下面你可以看到如何获得一个用户的Gravatar URL，电子邮件为`john@example.com`：

```Shell
>>> from hashlib import md5
>>> 'https://www.gravatar.com/avatar/' + md5(b'john@example.com').hexdigest()
'https://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6'
```
如果你想看一个实际的例子，我自己的Gravatar网址是：

```Shell
https://www.gravatar.com/avatar/729e26a2a2c7ff24a71958d4aa4e5f35
```
以下是Gravatar对这个URL的返回结果：

![ch06-gravatar](/assets/img/ch06-gravatar.jpg "ch06-gravatar")

默认情况下，返回的图像尺寸是80x80像素，但可以通过在URL的查询字符串中添加一个`s`参数来请求不同的尺寸。例如，要获得我自己的头像为128x128像素的图片，URL为： _\linebreak https://www.gravatar.com/avatar/729e26a2a2c7ff24a71958d4aa4e5f35?s=128_。

另一个可以作为查询字符串参数传递给Gravatar的有趣参数是`d`，它决定了Gravatar为那些没有在该服务中注册头像的用户提供什么样的图像。我最喜欢的是 "identicon"，它返回一个漂亮的几何设计，每封邮件都不同。比如说：

![ch06-gravatar-identicon](/assets/img/ch06-gravatar-identicon.png "ch06-gravatar-identicon")

请注意，一些网络浏览器扩展程序，如Ghostery，会阻止Gravatar图像，因为他们认为Automattic（Gravatar服务的所有者）可以根据他们收到的对你的头像的请求，确定你访问的网站。如果你在浏览器中看不到头像，请考虑问题可能是由于你在浏览器中安装了一个扩展。

由于头像是与用户相关联的，因此将生成头像URL的逻辑添加到用户模型中是有意义的。

_app/models.py_: User avatar URLs

```python
from hashlib import md5
# ...

class User(UserMixin, db.Model):
    # ...
    def avatar(self, size):
        digest = md5(self.email.lower().encode('utf-8')).hexdigest()
        return 'https://www.gravatar.com/avatar/{}?d=identicon&s={}'.format(
            digest, size)
```

用户类的新`avatar()`方法返回`User`头像的URL，缩放到所要求的像素大小。对于没有注册头像的用户，将生成一个 "identicon "图像。为了生成MD5哈希值，我首先将电子邮件转换为小写，因为这是Gravatar服务的要求。然后，由于Python中的MD5支持在字节上工作，而不是在字符串上，所以我在将字符串传递给哈希函数之前将其编码为字节。

如果你有兴趣了解Gravatar服务提供的其他选项，请访问他们的[文档网站](https://gravatar.com/site/implement/images)。

下一步是在用户资料模板中插入头像图片：

_app/templates/user.html_: User avatar in template

```html
{% extends "base.html" %}

{% block content %}
    <table>
        <tr valign="top">
            <td><img src="{{ user.avatar(128) }}"></td>
            <td><h1>User: {{ user.username }}</h1></td>
        </tr>
    </table>
    <hr>
    {% for post in posts %}
    <p>
    {{ post.author.username }} says: <b>{{ post.body }}</b>
    </p>
    {% endfor %}
{% endblock %}
```

让`User`类负责返回头像的好处是，如果有一天我决定Gravatar头像不是我想要的，我就可以重写`avatar()`方法来返回不同的URL，所有的模板将开始自动显示新的头像。

我在用户资料页面的顶部有一个漂亮的大头像，但实际上没有理由停在那里。我在底部有一些用户的帖子，每个帖子也可以有一个小头像。对于用户资料页来说，当然所有的帖子都会有同样的头像，但是我可以在主页上实现同样的功能，然后每篇帖子都会用作者的头像来装饰，这样看起来真的很不错。

为了显示各个帖子的头像，我只需要在模板中再做一个小改动：

_app/templates/user.html_: User avatars in posts

```html
{% extends "base.html" %}

{% block content %}
    <table>
        <tr valign="top">
            <td><img src="{{ user.avatar(128) }}"></td>
            <td><h1>User: {{ user.username }}</h1></td>
        </tr>
    </table>
    <hr>
    {% for post in posts %}
    <table>
        <tr valign="top">
            <td><img src="{{ post.author.avatar(36) }}"></td>
            <td>{{ post.author.username }} says:<br>{{ post.body }}</td>
        </tr>
    </table>
    {% endfor %}
{% endblock %}
```

![ch06-avatars.png](/assets/img/ch06-avatars.png "ch06-avatars.png")

## Using Jinja2 SUb-Templates

我设计了用户资料页面，使其显示用户所写的帖子，以及他们的头像。现在我想让索引页也显示具有类似布局的帖子。我可以直接复制/粘贴模板中处理帖子的部分，但这并不理想，因为以后如果我决定对这个布局进行修改，我将不得不记住更新两个模板。

相反，我要做一个子模板，只渲染一个帖子，然后我将从 _user.html_ 和 _index.html_ 模板中引用它。首先，我可以创建一个子模板，其中只有一个帖子的HTML标记。我将把这个模板命名为 _app/templates/\_post.html_。`_`前缀只是一个命名惯例，帮助我识别哪些模板文件是子模板。

_app/templates/\_post.html_: Post sub-template

```html
    <table>
        <tr valign="top">
            <td><img src="{{ post.author.avatar(36) }}"></td>
            <td>{{ post.author.username }} says:<br>{{ post.body }}</td>
        </tr>
    </table>
```

为了从user.html模板中调用这个子模板，我使用Jinja2的`include`语句：

_app/templates/user.html_: User avatars in posts

```html
{% extends "base.html" %}

{% block content %}
    <table>
        <tr valign="top">
            <td><img src="{{ user.avatar(128) }}"></td>
            <td><h1>User: {{ user.username }}</h1></td>
        </tr>
    </table>
    <hr>
    {% for post in posts %}
        {% include '_post.html' %}
    {% endfor %}
{% endblock %}
```

应用程序的索引页还没有真正充实起来，所以我还不打算在那里添加这个功能。

## More Interesting Profiles

新的用户资料页面有一个问题，就是他们在上面并没有真正展示什么。用户喜欢在这些页面上讲述他们的情况，所以我打算让他们在这里写一些关于他们自己的东西来展示。我还将跟踪每个用户最后一次访问网站的时间，并在他们的个人资料页面上显示出来。

为了支持所有这些额外的信息，我首先要做的是用两个新的字段扩展数据库中的用户表：

_app/models.py_: New fields in user model

```python
class User(UserMixin, db.Model):
    # ...
    about_me = db.Column(db.String(140))
    last_seen = db.Column(db.DateTime, default=datetime.utcnow)
```

每次数据库被修改，都有必要生成一个数据库迁移。在第4章中，我向你展示了如何设置应用程序以通过迁移脚本跟踪数据库的变化。现在我有两个新的字段要添加到数据库中，所以第一步是生成迁移脚本：

```Shell
(venv) $ flask db migrate -m "new fields in user model"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added column 'user.about_me'
INFO  [alembic.autogenerate.compare] Detected added column 'user.last_seen'
  Generating migrations/versions/37f06a334dbf_new_fields_in_user_model.py ... done
```
`migrate`命令的输出结果看起来不错，因为它显示在`User`类中的两个新字段被检测到。现在我可以把这个变化应用到数据库中：

```Shell
(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 780739b227a7 -> 37f06a334dbf, new fields in user model
```
我希望你能意识到与迁移框架合作是多么有用。任何在数据库中的用户都还在，迁移框架以外科手术的方式应用迁移脚本中的变化，而不破坏任何数据。

下一步，我将把这两个新字段添加到用户配置文件模板中：

_app/templates/user.html_: Show user information in user profile template

```html
{% extends "base.html" %}

{% block content %}
    <table>
        <tr valign="top">
            <td><img src="{{ user.avatar(128) }}"></td>
            <td>
                <h1>User: {{ user.username }}</h1>
                {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
                {% if user.last_seen %}<p>Last seen on: {{ user.last_seen }}</p>{% endif %}
            </td>
        </tr>
    </table>
    ...
{% endblock %}
```
请注意，我把这两个字段包装在Jinja2的条件中，因为我只想让它们在被设置后才可见。在这一点上，这两个新字段对所有用户来说都是空的，所以如果你现在运行这个应用程序，你不会看到这些字段。

## Recording THe Last Visit Time For a User

让我们从`last_seen`字段开始，这是两个字段中最简单的。我想做的是，只要某个用户向服务器发送请求，就在这个字段上写入当前的时间。

在每一个可能被浏览器请求的视图函数上添加登录来设置这个字段显然是不切实际的，但是在请求被派发到视图函数之前执行一点通用逻辑是Web应用程序中非常常见的任务，所以Flask将其作为一个本地功能提供。看看这个解决方案吧：

_app/routes.py_: Record time of last visit

```python
from datetime import datetime

@app.before_request
def before_request():
    if current_user.is_authenticated:
        current_user.last_seen = datetime.utcnow()
        db.session.commit()
```

Flask的`@before_request`装饰器注册了被装饰的函数，使其在视图函数之前被执行。这非常有用，因为现在我可以在应用程序中的任何视图函数之前插入我想执行的代码，而且我可以把它放在一个地方。这个实现只是检查`current_user`是否登录，在这种情况下，将`last_seen`字段设置为当前时间。我之前提到过，一个服务器应用程序需要在一致的时间单位下工作，标准做法是使用UTC时区。使用系统的本地时间并不是一个好主意，因为这样的话，数据库中的内容就取决于你的位置。最后一步是提交数据库会话，这样上面所做的改变就被写入了数据库。如果你想知道为什么在提交之前没有`db.session.add()`，考虑到当你引用`current_user`时，Flask-Login会调用user loader的回调函数，它将运行一个数据库查询，将目标用户放入数据库会话。所以你可以在这个函数中再次添加用户，但这是不必要的，因为它已经在那里了。

如果你在做了这个改动后查看你的个人资料页面，你会看到 "最后一次见到 "一行的时间与当前时间非常接近。如果你离开个人资料页面，然后再返回，你会看到时间是不断更新的。

事实上，我将这些时间戳存储在UTC时区，使得个人资料页面上显示的时间也是UTC时区的。除此之外，时间的格式也不是你所期望的那样，因为它实际上是Python datetime对象的内部表示。现在，我不打算担心这两个问题，因为我将在后面的章节中讨论在网络应用中处理日期和时间的话题。

![ch06-last-seen.png](/assets/img/ch06-last-seen.png "ch06-last-seen.png")

## Profile Editor

我还需要给用户一个表单，让他们可以输入一些关于自己的信息。这个表单将让用户改变他们的用户名，并写一些关于他们自己的信息，这些信息将被保存在新的`about_me`域中。让我们开始为它编写一个表单类：

_app/forms.py_: Profile editor form

```python
from wtforms import StringField, TextAreaField, SubmitField
from wtforms.validators import DataRequired, Length

# ...

class EditProfileForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired()])
    about_me = TextAreaField('About me', validators=[Length(min=0, max=140)])
    submit = SubmitField('Submit')
```

我在这个表单中使用了一个新的字段类型和一个新的验证器。对于` "About" `字段，我使用了`TextAreaField`，它是一个多行框，用户可以在其中输入文本。为了验证这个字段，我使用了`Length`，它将确保输入的文本在0到140个字符之间，这是我在数据库中为相应字段分配的空间。

渲染这个表单的模板如下所示：

_app/templates/edit_profile.html_: Profile editor form

```html
{% extends "base.html" %}

{% block content %}
    <h1>Edit Profile</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.username.label }}<br>
            {{ form.username(size=32) }}<br>
            {% for error in form.username.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.about_me.label }}<br>
            {{ form.about_me(cols=50, rows=4) }}<br>
            {% for error in form.about_me.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```
最后，这里是将一切联系起来的视图功能：

_app/routes.py_: Edit profile view function

```python
from app.forms import EditProfileForm

@app.route('/edit_profile', methods=['GET', 'POST'])
@login_required
def edit_profile():
    form = EditProfileForm()
    if form.validate_on_submit():
        current_user.username = form.username.data
        current_user.about_me = form.about_me.data
        db.session.commit()
        flash('Your changes have been saved.')
        return redirect(url_for('edit_profile'))
    elif request.method == 'GET':
        form.username.data = current_user.username
        form.about_me.data = current_user.about_me
    return render_template('edit_profile.html', title='Edit Profile',form=form)
```
这个视图函数以一种稍微不同的方式处理表单。如果`validate_on_submit()`返回`True`，我就把表单中的数据复制到用户对象中，然后把该对象写入数据库中。但是当`validate_on_submit()`返回`False`时，可能是由于两个不同的原因。首先，这可能是因为浏览器刚刚发送了一个`GET`请求，我需要通过提供一个初始版本的表单模板来回应。也有可能是浏览器发送了一个带有表单数据的`POST`请求，但数据中有些东西是无效的。对于这个表单，我需要分别处理这两种情况。当表单第一次被`GET`请求时，我想用存储在数据库中的数据预先填充字段，所以我需要做与提交情况相反的事情，将存储在用户字段中的数据移到表单中，因为这将确保这些表单字段有为用户存储的当前数据。但是在验证错误的情况下，我不想向表单字段写任何东西，因为那些字段已经被WTForms填充了。为了区分这两种情况，我检查了`request.method`，对于最初的请求，它将是`GET`，而对于验证失败的提交，它将是`POST`。

![ch06-user-profile](/assets/img/ch06-user-profile.png "ch06-user-profile")

为了方便用户访问个人资料编辑页面，我可以在他们的个人资料页面添加一个链接：

_app/templates/user.html_: Edit profile link

```html
                {% if user == current_user %}
                <p><a href="{{ url_for('edit_profile') }}">Edit your profile</a></p>
                {% endif %}
```

请注意我所使用的巧妙条件，以确保在你查看自己的资料时出现编辑链接，而在你查看别人的资料时则不出现。

![ch06-user-profile-link](/assets/img/ch06-user-profile-link.png "ch06-user-profile-link")

# Chapter7:Error Handling

在本章中，我将暂停在我的微博应用程序中编写新功能，而是讨论一些处理错误的策略，这些错误总是会出现在每个软件项目中。 为了帮助说明这个主题，我故意在第 6 章中添加的代码中遗漏了一个错误。在继续阅读之前，看看您是否能找到它！

## Error Handling in Flask

当 Flask 应用程序发生错误时会发生什么？ 找出答案的最好方法是亲身体验。 继续并启动应用程序，并确保您至少注册了两个用户。 以其中一个用户身份登录，打开个人资料页面并单击“编辑”链接。 在配置文件编辑器中，尝试将用户名更改为另一个已注册用户的用户名，然后砰！ 这将带来一个看起来很可怕的“内部服务器错误”页面：

![ch07-500-error](/assets/img/ch07-500-error.png "ch07-500-error")

如果查看应用程序运行的终端会话，您将看到错误的堆栈跟踪。 堆栈跟踪在调试错误时非常有用，因为它们显示了该堆栈中的调用顺序，一直到产生错误的行：

```Shell
(venv) $ flask run
 * Serving Flask app "microblog"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
[2021-06-14 22:40:02,027] ERROR in app: Exception on /edit_profile [POST]
Traceback (most recent call last):
  File "venv/lib/python3.6/site-packages/sqlalchemy/engine/base.py", in _execute_context
    context)
  File "venv/lib/python3.6/site-packages/sqlalchemy/engine/default.py", in do_execute
    cursor.execute(statement, parameters)
sqlite3.IntegrityError: UNIQUE constraint failed: user.username
```
堆栈跟踪指示了错误所在。该应用程序允许用户更改用户名，但不验证新选择的用户名是否与系统中已有的其他用户冲突。错误来自SQLAlchemy，它尝试将新用户名写入数据库，但是由于`username`列定义为`unique=True`，因此数据库拒绝了它。

需要注意的是，向用户呈现的错误页面并没有提供有关错误的太多信息，这是好事。我绝对不希望用户知道崩溃是由数据库错误引起的，或者我正在使用哪个数据库，或者我的数据库中有哪些表和字段名称。所有这些信息都应该保持内部。

有一些问题远非理想。我有一个非常丑陋且与应用程序布局不匹配的错误页面。我还需要在终端上倾泻重要的应用程序堆栈跟踪以确保不会错过任何错误。当然，我也有一个bug需要修复。 我将解决所有这些问题，但首先让我们谈谈Flask _调试模式_。

# Debug Mode

你上面看到的错误处理方式非常适合在生产服务器上运行的系统。如果出现错误，用户会收到一个模糊的错误页面（尽管我将使这个错误页面更好），而重要的错误细节则在服务器进程输出或日志文件中。

但是，在开发应用程序时，您可以启用调试模式，这是 Flask 直接在浏览器上输出真正漂亮的调试器的一种模式。要激活调试模式，请停止应用程序，然后设置以下环境变量：

```Shell 
(venv) $ export FLASK_ENV=development
```

如果您使用的是 Microsoft Windows，请记得使用 `set` 而不是 `export`。

在设置 `FLASK_ENV` 后，重新启动服务器。您终端上的输出将与您习惯看到的略有不同：

```Shell
(venv) microblog2 $ flask run
 * Serving Flask app 'microblog.py' (lazy loading)
 * Environment: development
 * Debug mode: on
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 118-204-854
 ```

现在让应用程序再次崩溃，以查看浏览器中的交互式调试器：
![ch07-debugger](/assets/img/ch07-debugger.png "ch07-debugger")

调试器允许您展开每个堆栈帧并查看相应的源代码。 您还可以在任何框架上打开Python提示符并执行任何有效的Python表达式，例如检查变量的值。

非常重要的是，您永远不要在生产服务器上以调试模式运行Flask应用程序。 调试器允许用户在服务器上远程执行代码，因此它可能成为想要渗透您的应用程序或服务器的恶意用户意外获得的礼物。 作为额外的安全措施，在浏览器中运行的调试器会启动锁定，并且第一次使用时将请求PIN号码，该号码可以在`flask run`命令输出中看到。

既然我正在谈论调试模式，我应该提到启用了调试模式后激活第二个重要功能——重新加载程序。 这是一个非常有用的开发功能，当修改源文件时自动重新启动应用程序。 如果您在调试模式下运行`flask run`，则可以随时对应用程序进行操作，并且每次保存文件时都会重新启动以获取新代码。

## Custom Error Pages

Flask 提供了一种机制，使应用程序能够安装自己的错误页面，这样您的用户就不必看到普通和无聊的默认页面。例如，让我们为 HTTP 错误 404 和 500 定义自定义错误页面，这是最常见的两个错误。为其他错误定义页面的方式相同。

要声明自定义错误处理程序，请使用 `@errorhandler` 装饰器。我将把我的错误处理程序放在一个新的 _app/errors.py_ 模块中。

_app/errors.py_: Custom error handlers

```python
from flask import render_template
from app import app, db

@app.errorhandler(404)
def not_found_error(error):
    return render_template('404.html'), 404

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()
    return render_template('500.html'), 500
```

错误函数的工作方式与视图函数非常相似。对于这两个错误，我返回它们各自模板的内容。请注意，这两个函数在模板之后都返回第二个值，即错误代码编号。到目前为止，我创建的所有视图函数都不需要添加第二个返回值，因为默认情况下200（成功响应的状态码）是我想要的。但在这种情况下是错误页面，所以我希望响应状态码反映出来。

500 错误处理程序可能会在数据库错误之后被调用，在上面重复用户名时实际上就是如此。为了确保任何失败的数据库会话不会干扰由模板触发的任何数据库访问，请执行会话回滚操作。这将使会话恢复到一个干净状态。

以下是404错误页面模板：

_app/templates/404.html_: Not found error template

```html
{% extends "base.html" %}

{% block content %}
    <h1>File Not Found</h1>
    <p><a href="{{ url_for('index') }}">Back</a></p>
{% endblock %}
```
这是针对500错误的页面模板：
_app/templates/500.html_: Internal server error template

```html
{% extends "base.html" %}

{% block content %}
    <h1>An unexpected error has occurred</h1>
    <p>The administrator has been notified. Sorry for the inconvenience!</p>
    <p><a href="{{ url_for('index') }}">Back</a></p>
{% endblock %}
```
两个模板都继承自`base.html`模板，因此错误页面与应用程序的普通页面具有相同的外观和感觉。

要将这些错误处理程序注册到Flask中，我需要在创建应用程序实例后导入新的 _app/errors.py_ 模块：

_app/\_\_init\_\_.py_: Import error handlers

```python
# ...

from app import routes, models, errors
```

如果您在终端会话中设置了`FLASK_ENV=production`，然后再次触发重复的用户名错误，您将看到一个稍微友好一些的错误页面。
![ch07-500-custom](/assets/img/ch07-500-custom.png "ch07-500-custom")

## Sending Error by Email

Flask提供的默认错误处理存在另一个问题，即没有通知，错误的堆栈跟踪会打印到终端上，这意味着需要监视服务器进程的输出以发现错误。在开发过程中运行应用程序时，这是完全可以接受的，但一旦将应用程序部署到生产服务器上，就不会有人查看输出了，因此需要采取更强大的解决方案。

我认为采取积极主动的方法来处理错误非常重要。如果在应用程序生产版本中出现错误，则希望立即得知。因此我的第一个解决方案是配置Flask，在出现错误后立即向我发送电子邮件，并将堆栈跟踪放入电子邮件正文中。

第一步是将电子邮件服务器详细信息添加到配置文件中：

_config.py_: Email configuration

```python
class Config(object):
    # ...
    MAIL_SERVER = os.environ.get('MAIL_SERVER')
    MAIL_PORT = int(os.environ.get('MAIL_PORT') or 25)
    MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS') is not None
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    ADMINS = ['your-email@example.com']
```

电子邮件的配置变量包括服务器和端口、一个布尔标志以启用加密连接，以及可选的用户名和密码。这五个配置变量是从它们的环境变量对应项中获取的。如果在环境中没有设置电子邮件服务器，则我将使用此作为禁用发送错误电子邮件的信号。电子邮件服务器端口也可以在环境变量中给出，但如果未设置，则使用标准端口25。默认情况下不使用电子邮件服务器凭据，但如有需要可以提供。`ADMINS`配置变量是接收错误报告的电子邮件地址列表，因此您自己的电子邮件地址应该在该列表中。

Flask使用Python`logging`包来编写其日志，并且该包已经具有通过电子邮件发送日志的功能。要使错误时发送电子邮件，我只需向Flask记录器对象`app.logger`添加[SMTPHandler](https://docs.python.org/3.6/library/logging.handlers.html#smtphandler)实例即可：

_app/\_\_init\_\_.py_: Log errors by email

```python
import logging
from logging.handlers import SMTPHandler
# ...
if not app.debug:
    if app.config['MAIL_SERVER']:
        auth = None
        if app.config['MAIL_USERNAME'] or app.config['MAIL_PASSWORD']:
            auth = (app.config['MAIL_USERNAME'], app.config['MAIL_PASSWORD'])
        secure = None
        if app.config['MAIL_USE_TLS']:
            secure = ()
        mail_handler = SMTPHandler(
            mailhost=(app.config['MAIL_SERVER'], app.config['MAIL_PORT']),
            fromaddr='no-reply@' + app.config['MAIL_SERVER'],
            toaddrs=app.config['ADMINS'], subject='Microblog Failure',
            credentials=auth, secure=secure)
        mail_handler.setLevel(logging.ERROR)
        app.logger.addHandler(mail_handler)
```

正如您所看到的，我只会在应用程序没有调试模式运行时启用电子邮件记录器，这可以通过`app.debug`为`True`以及配置中存在电子邮件服务器来指示。

由于必须处理许多电子邮件服务器中存在的可选安全选项，因此设置电子邮件记录器有些繁琐。但本质上，上面的代码创建了一个`SMTPHandler`实例，并将其级别设置为仅报告错误而不是警告、信息或调试消息，并最终将其附加到Flask的`app.logger`对象上。

测试此功能有两种方法。最简单的方法是使用Python提供的SMTP调试服务器。这是一个虚假的电子邮件服务器，它接受电子邮件，但不发送它们，而是将它们打印到控制台。要运行此服务器，请打开第二个终端会话并在其中运行以下命令：

```Shell
(venv) $ python -m smtpd -n -c DebuggingServer localhost:8025
```

让调试SMTP服务器保持运行状态，回到第一个终端并在环境中设置`export MAIL_SERVER=localhost`和`MAIL_PORT=8025`（如果您使用Microsoft Windows，请使用`set`而不是`export`）。确保`FLASK_ENV`变量设置为`production`或未设置，因为应用程序将不会在调试模式下发送电子邮件。运行应用程序并再次触发SQLAlchemy错误，以查看运行虚假电子邮件服务器的终端会话如何显示带有完整堆栈跟踪错误的电子邮件。

这个功能的第二种测试方法是配置真实的电子邮件服务器。以下是使用Gmail帐户的电子邮件服务器进行配置：

```
export MAIL_SERVER=smtp.googlemail.com
export MAIL_PORT=587
export MAIL_USE_TLS=1
export MAIL_USERNAME=<your-gmail-username>
export MAIL_PASSWORD=<your-gmail-password>
```

如果您使用的是Microsoft Windows，请记得在上述每个语句中使用`set`而不是`export`。

您Gmail帐户中的安全功能可能会阻止应用程序通过它发送电子邮件，除非您明确允许“较不安全的应用程序”访问您的Gmail帐户。 您可以在[此处](https://support.google.com/accounts/answer/6010255?hl=en)阅读有关此内容的信息，如果您担心帐户的安全性，可以创建一个仅用于测试电子邮件配置的二次账户，或者只暂时启用较不安全的应用程序来运行此测试，然后恢复默认设置。

另一种选择是使用专门的电子邮件服务（例如[SendGrid](https://sendgrid.com/)），该服务允许您在免费账户上每天发送最多100封电子邮件。 SendGrid博客详细介绍了[如何在Flask应用程序中使用该服务](https://sendgrid.com/blog/sending-emails-from-python-flask-applications-with-twilio-sendgrid/)。

## Logging to a File

通过电子邮件接收错误信息很好，但有时这还不够。有些故障条件并没有以Python异常的形式结束，并且它们不是一个主要问题，但仍然可能足够有趣以便于调试目的保存下来。因此，我还将为应用程序维护一个日志文件。

为了启用基于文件的日志记录，需要将另一个处理程序（这次是`RotatingFileHandler`类型）附加到应用程序记录器上，类似于电子邮件处理程序。

_app/\_\_init\_\_.py_: Logging to a file

```python
# ...
from logging.handlers import RotatingFileHandler
import os

# ...

if not app.debug:
    # ...

    if not os.path.exists('logs'):
        os.mkdir('logs')
    file_handler = RotatingFileHandler('logs/microblog.log', maxBytes=10240,
                                       backupCount=10)
    file_handler.setFormatter(logging.Formatter(
        '%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'))
    file_handler.setLevel(logging.INFO)
    app.logger.addHandler(file_handler)

    app.logger.setLevel(logging.INFO)
    app.logger.info('Microblog startup')
```
我正在创建一个名为`microblog.log`的日志文件，并将其写入logs目录中，如果该目录不存在，则会创建它。

`RotatingFileHandler`类非常好用，因为它可以轮换日志，确保应用程序长时间运行时不会使日志文件变得过大。在这种情况下，我将日志文件大小限制为10KB，并保留最后十个备份日志文件。

`logging.Formatter`类提供了自定义格式化的日志消息。由于这些消息要写入到文件中，所以我希望它们包含尽可能多的信息。因此，我使用了一种格式，其中包括时间戳、记录级别、消息以及生成该日志条目的源代码文件和行号。

为了使记录更有用，在应用程序记录器和文件记录器处理程序中也将记录级别降低到`INFO`类别。如果您不熟悉记录类别，则按严重性递增顺序分为`DEBUG`、`INFO`、`WARNING`、`ERROR`和`CRITICAL`。

作为对日志文件第一个有趣的使用方式，在服务器每次启动时都会向日志中写入一行内容。当此应用程序在生产服务器上运行时，这些日志条目将告诉您服务器何时重新启动。

## Fixing the Duplicate Username Bug

我已经利用了用户名重复漏洞太久了。现在我已经向您展示了如何准备应用程序来处理此类错误，我可以继续修复它。

如果您还记得，`RegistrationForm` 已经实现了用户名验证，但编辑表单的要求略有不同。在注册期间，我需要确保在表单中输入的用户名不存在于数据库中。在编辑个人资料表单上，我必须进行相同的检查，但有一个例外。如果用户未更改原始用户名，则验证应允许该名称通过，因为该用户名已分配给该用户。下面是我如何为此表单实现用户名验证：

_app/forms.py_: Validate username in edit profile form.

```python
class EditProfileForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired()])
    about_me = TextAreaField('About me', validators=[Length(min=0, max=140)])
    submit = SubmitField('Submit')

    def __init__(self, original_username, *args, **kwargs):
        super(EditProfileForm, self).__init__(*args, **kwargs)
        self.original_username = original_username

    def validate_username(self, username):
        if username.data != self.original_username:
            user = User.query.filter_by(username=self.username.data).first()
            if user is not None:
                raise ValidationError('Please use a different username.')
```

实现在自定义验证方法中，但是有一个重载的构造函数接受原始用户名作为参数。该用户名保存为实例变量，并在`validate_username()`方法中进行检查。如果表单中输入的用户名与原始用户名相同，则无需检查数据库是否存在重复。

要使用这个新的验证方法，在创建表单对象的视图函数中需要添加原始用户名参数：

_app/routes.py_: Validate username in edit profile form.

```python
@app.route('/edit_profile', methods=['GET', 'POST'])
@login_required
def edit_profile():
    form = EditProfileForm(current_user.username)
    # ...
```

现在该错误已经修复，大多数情况下将防止编辑个人资料表单中的重复。这不是一个完美的解决方案，因为当两个或更多进程同时访问数据库时可能无法工作。在那种情况下，竞争条件可能会导致验证通过，但稍后尝试重命名时，数据库已被另一个进程更改，并且无法重命名用户。除非是具有大量服务器进程的非常繁忙的应用程序，否则这种情况不太可能发生，所以我现在不会担心它。

此时您可以再次尝试重现错误以查看新表单验证方法如何防止它。

# Chapter8:Followers

在这一章中，我将对应用程序的数据库进行更多的工作。我希望应用程序的用户能够轻松地选择他们想要关注的其他用户。因此，我将扩展数据库，使其能够跟踪谁在关注谁，这比你想象的要棘手。

## Database Relationships Revisited

我在上面说，我想为每个用户维护一个 "被关注 "和 "关注者 "的列表。不幸的是，关系型数据库没有我可以用于这些列表的列表类型，有的只是带有记录和这些记录之间关系的表。

数据库有一个代表用户的表，所以剩下的就是想出适当的关系类型来模拟关注者/被关注者的链接。这是一个回顾基本数据库关系类型的好时机：

### One-to-Many

我在第四章已经使用了一对多的关系。下面是这种关系的示意图：
    
    
![ch04-users-posts](/assets/img/ch04-users-posts.png "ch04-users-posts")

    
由这种关系连接的两个实体是用户和帖子。我说，一个用户有很多帖子，而一个帖子有一个用户（或作者）。这种关系在数据库中通过在 "许多 "一侧使用外键(_forign key_)来表示。在上面的关系中，外键是添加到`posts`表中的`user_id`字段。这个字段将每个帖子与用户表中的作者记录联系在一起。

很明显，`user_id字`段提供了对某一特定帖子的作者的直接访问，但反过来说呢？为了使这种关系变得有用，我应该能够得到一个给定用户所写的帖子的列表。`posts`表中的`user_id`字段也足以回答这个问题，因为数据库有索引，可以进行有效的查询，例如 "检索所有user_id为X的帖子"。

### Many-to-Many

多对多的关系就比较复杂了。作为一个例子，考虑一个有学生和教师的数据库。我可以说一个学生有很多老师，而一个老师有很多学生。这就像两端重叠的一对多关系。

对于这种类型的关系，我应该能够查询数据库并获得教某个学生的教师列表，以及教师班上的学生列表。这在关系型数据库中实际上是不难表现的，因为它不能通过向现有的表添加外键来完成。

表示多对多的关系需要使用一个称为关联表的辅助表。下面是学生和教师例子中的数据库的样子：
    
    
![ch08-students-teachers](/assets/img/ch08-students-teachers.png "ch08-students-teachers")

    
虽然一开始看起来并不明显，但带有两个外键的关联表能够有效地回答关于关系的所有查询。

### Many-to-One and One-to-One
多对一类似于一对多的关系。不同的是，这种关系是从 "多 "的方面来看的。

一对一关系是一对多的一个特例。其表现形式类似，但在数据库中加入了一个约束条件，以防止 "多 "方有一个以上的链接。虽然在某些情况下，这种关系是有用的，但它不像其他类型那样常见。

## Representing Followers

看看所有关系类型的总结，很容易确定跟踪追随者的正确数据模型是多对多的关系，因为一个用户关注 _许多_ 用户，而一个用户有 _许多_ 追随者。但是有一个转折。在学生和教师的例子中，我有两个实体是通过多对多的关系联系起来的。但是在关注者的例子中，我有用户关注其他用户，所以只有用户。那么，多对多关系的第二个实体是什么？

该关系的第二个实体也是用户。一个类的实例与同一类的其他实例相联系的关系被称为自指关系(_self-referential_)，而这正是我在这里的情况。

下面是跟踪追随者的自我参照的多对多关系的图示：
    
![ch08-followers-schema](/assets/img/ch08-followers-schema.png "ch08-followers-schema")
    
跟随者表是该关系的关联表。这个表中的外键都是指向用户表中的条目，因为它是将用户与用户联系起来的。这个表中的每条记录都代表一个追随者用户和被追随者用户之间的一个联系。就像学生和教师的例子一样，这样的设置允许数据库回答我所需要的关于被关注用户和关注者的所有问题。相当整洁。

## Database Model Representation
让我们先把`followers`添加到数据库中。这里是`followers`关联表：

_app/models.py_: Followers association table

```python
followers = db.Table('followers',
    db.Column('follower_id', db.Integer, db.ForeignKey('user.id')),
    db.Column('followed_id', db.Integer, db.ForeignKey('user.id'))
)
```

这是我上图中关联表的直接翻译。请注意，我没有把这个表作为一个模型来声明，就像我为用户和帖子表所做的那样。因为这是一个辅助表，除了外键之外没有其他数据，所以我创建它时没有关联的模型类。

现在我可以在用户表中声明多对多的关系：
_app/models.py_: Many-to-many followers relationship

```python
class User(UserMixin, db.Model):
    # ...
    followed = db.relationship(
        'User', secondary=followers,
        primaryjoin=(followers.c.follower_id == id),
        secondaryjoin=(followers.c.followed_id == id),
        backref=db.backref('followers', lazy='dynamic'), lazy='dynamic')
```
这个关系的设置并不复杂。就像我对`posts`的一对多关系所做的那样，我使用`db.relationship`函数来定义模型类中的关系。这种关系将`User`实例链接到其他`User`实例上，所以作为惯例，我们说对于由这种关系链接的一对用户，左边的用户是跟随右边的用户。我定义的关系是从左边的用户看到的，名字是`followed`，因为当我从左边查询这个关系时，我将得到被关注的用户列表（即右边的用户）。让我们逐一检查 `db.relationship()` 调用的所有参数：

* `User`是这个关系的右侧实体（左侧实体是父类）。由于这是一个自指关系，我必须在两边使用相同的类。
* `secondary`配置用于这种关系的关联表，我在这个类上面定义了它。
* `primaryjoin`表示连接左侧实体（跟随者用户）和关联表的条件。关系左侧的连接条件是与关联表的`follower_id`字段匹配的用户ID。这个参数的值是`followers.c.follower_id`，它引用了关联表的`follower_id`列。
* `secondaryjoin`表示将右边的实体（被关注的用户）与关联表联系起来的条件。这个条件类似于`primaryjoin`的条件，唯一的区别是现在我使用`followed_id`，它是关联表中的另一个外键。
* `backref`定义了如何从右边的实体访问这种关系。从左边来看，这个关系被命名为`followed`，所以从右边来看，我将使用`followers`这个名字来代表所有与右边的目标用户有联系的左边用户。额外的`lazy`参数表示这个查询的执行模式。一个`dynamic`的模式设置了查询，直到特别要求才运行，这也是我设置帖子一对多关系的方式。
* `lazy`与`backref`中的同名参数类似，但这个参数适用于左边的查询，而不是右边的。

如果这很难理解，请不要担心。我一会儿会告诉你如何使用这些查询，然后一切都会变得更加清晰。

对数据库的改变需要记录在一个新的数据库迁移中：
```Shell
(venv) $ flask db migrate -m "followers"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'followers'
  Generating /home/miguel/microblog/migrations/versions/ae346256b650_followers.py ... done

(venv) $ flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 37f06a334dbf -> ae346256b650, followers
```

## Adding and Removing "follows"

感谢SQLAlchemy ORM，一个用户跟随另一个用户，可以在数据库中记录下被跟随的关系，就像它是一个列表一样。例如，如果我有两个用户存储在`user1`和`user2`变量中，我可以用这个简单的语句使第一个用户跟随第二个用户：

```user1.followed.append(user2)```

要取关用户，那么我可以做：

```user1.followed.remove(user2)```

尽管添加和删除关注者是相当容易的，但我想在我的代码中促进可重用性，所以我不打算在代码中撒上`appends`和`removes`。相反，我将把`follow`和`unfollow`功能作为`User`模型的方法来实现。最好是将应用逻辑从视图函数中移出，移到模型或其他辅助类或模块中，因为正如你在本章后面所看到的，这使得单元测试更加容易。

下面是用户模型中增加和删除关系的变化：
_app/models.py_: Add and remove followers

```python
class User(UserMixin, db.Model):
    #...

    def follow(self, user):
        if not self.is_following(user):
            self.followed.append(user)

    def unfollow(self, user):
        if self.is_following(user):
            self.followed.remove(user)

    def is_following(self, user):
        return self.followed.filter(
            followers.c.followed_id == user.id).count() > 0
```

`follow()`和`unfollow()`方法使用关系对象的`append()`和`remove()`方法，正如我在上面显示的那样，但是在它们接触关系之前，它们使用`is_following()`支持方法来确保请求的操作是合理的。例如，如果我要求`user1`跟随`user2`，但结果发现这个跟随关系已经存在于数据库中，我不想增加一个重复的关系。同样的逻辑也可以应用于取消关注的情况。

`is_following()`方法对`followed`关系发出查询，以检查两个用户之间的链接是否已经存在。你以前看到过我使用SQLAlchemy查询对象的`filter_by()`方法，例如找到一个用户的用户名。我在这里使用的`filter()`方法是类似的，但层次较低，因为它可以包括任意的过滤条件，不像`filter_by()`只能检查是否与一个常量值相等。我在`is_following()`中使用的条件是在关联表中寻找那些左边外键设置为`self` user，右边设置为`user`参数的项目。该查询以`count()`方法结束，该方法返回结果的数量。这个查询的结果将是`0`或`1`，所以检查计数是否为1或大于0实际上是等同的。你在过去看到我使用的其他查询终止器是`all()`和`first()`。

## Obtaining the Posts from Followed Users

对数据库中追随者的支持几乎已经完成，但实际上我还缺少一个重要的功能。在应用程序的索引页中，我要显示所有被登录用户关注的人写的博客文章，所以我需要想出一个数据库查询来返回这些文章。

最明显的解决方案是运行一个返回被关注用户列表的查询，正如你已经知道的，它应该是`user.followed.all()`。然后对于每个返回的用户，我可以运行一个查询来获取帖子。一旦我有了所有的帖子，我就可以把它们合并到一个列表中，并按日期排序。听起来不错吧？嗯，不是真的。

这种方法有几个问题。如果一个用户关注了一千个人，会发生什么？我将需要执行一千个数据库查询，只是为了收集所有的帖子。然后我还需要在内存中对这一千个列表进行合并和排序。作为一个次要的问题，考虑到应用程序的主页最终将实现分页(_pagination_)，所以它不会显示所有可用的帖子，而只是显示前几个，如果需要的话，还可以通过链接获得更多的帖子。如果我打算显示按日期排序的帖子，我怎么能知道哪些帖子是所有被关注的用户中最新的，除非我先得到所有的帖子并对它们进行排序？这实际上是一个糟糕的解决方案，不能很好地扩展。

真的没有办法避免这种博客文章的合并和排序，但在应用程序中这样做会导致一个非常低效的过程。这种工作是关系型数据库所擅长的。数据库有索引，允许它以更有效的方式进行查询和排序，这是我在自己这边可以做到的。因此，我真正想要的是提出一个单一的数据库查询，定义我想得到的信息，然后让数据库找出如何以最有效的方式提取这些信息。

下面你可以看到这个查询：
_app/models.py_: Followed posts query

```pythonclass User(UserMixin, db.Model):
    #...
    def followed_posts(self):
        return Post.query.join(
            followers, (followers.c.followed_id == Post.user_id)).filter(
                followers.c.follower_id == self.id).order_by(
                    Post.timestamp.desc())
```

这是迄今为止我在这个应用中使用的最复杂的查询。我将试着一步一步地解读这个查询。如果你看一下这个查询的结构，你会发现有三个主要部分是由SQLAlchemy查询对象的join()、filter()和order_by()方法设计的：

```Post.query.join(...).filter(...).order_by(...)```

### Joins
为了理解连接操作的作用，让我们看一个例子。让我们假设我有一个用户表，其内容如下：
| id |	username |
|----|:---------:|
|1	 |john       |
|2	 |susan      |
|3	 |mary       |
|4	 |david      |

为了简单起见，我不显示用户模型中的所有字段，只显示对这个查询重要的字段。

假设`followers`关联表显示，用户`john`正在关注用户`susan`和`david`，用户`susan`正在关注`mary`，用户`mary`正在关注`david`。表示上述情况的数据是这样的：


|follower_id |followed_id|
|------------|-----------|
|1           |      	2|
|1           |      	4|
|2           |      	3|
|3           |      	4|

最后，帖子表包含每个用户的一个帖子：

|id	|text	    |user_id |
|---|-----------|--------|
|1	|post from susan|	2|
|2	|post from mary |	3|
|3	|post from david|	4|
|4	|post from john |	1|

这个表还省略了一些不属于本讨论范围的字段。

下面是我为这个查询再次定义的`join()`调用：

```
Post.query.join(followers, (followers.c.followed_id == Post.user_id))
```

我正在调用`posts`表的连接操作。第一个参数是关注者关联表，第二个参数是连接条件。通过这个调用我想说的是，我想让数据库创建一个临时表，将帖子和关注者表中的数据合并起来。这些数据将根据我作为参数传递的条件进行合并。

我使用的条件是，关注者表的`followed_id`字段必须等于帖子表的`user_id`。为了执行这个合并，数据库将从post表中获取每条记录（连接的左边），并从`followers`表中追加任何符合条件的记录（连接的右边）。如果`followers`中的多条记录符合条件，那么帖子条目将对每条记录进行重复。如果对于一个给定的帖子，在关注者表中没有匹配的记录，那么这个帖子记录就不是连接的一部分。

用我上面定义的例子数据，连接操作的结果是：


| id | text             | user_id | follower_id | followed_id |
|----|-----------------|---------|-------------|-------------|
| 1  | post from susan  | 2       | 1           | 2           |
| 2  | post from mary   | 3       | 2           | 3           |
| 3  | post from david  | 4       | 1           | 4           |
| 3  | post from david  | 4       | 3           | 4           |

请注意，所有情况下的 `user_id` 和 `followed_id` 列都相等，因为这是连接条件。来自用户 `john` 的帖子未出现在连接表中，因为在 `followers` 中没有以 `john` 作为跟随用户的条目，或者换句话说，没有人在关注 `john`。而 `david` 的帖子出现了两次，因为该用户被两个不同的用户关注。

可能并不立即清楚通过创建这个连接，我得到了什么，但请继续阅读，因为这只是更大查询的一部分。

### Filters

连接操作提供了所有被某个用户关注的帖子的列表，这是我真正想要的更多数据。但是，我只对这个列表的一个子集感兴趣，即被单个用户关注的帖子，因此我需要修剪掉所有不需要的条目，这可以通过`filter()`调用来完成。

这是查询的filter部分：

```filter(followers.c.follower_id == self.id)```

由于此查询在 `User` 类的方法中，因此 `self.id` 表达式指的是我感兴趣的用户的用户 ID。`filter()` 调用选择具有 `follower_id` 列设置为此用户的连接表中的项目，这意味着我只保留具有此用户作为关注者的条目。

假设我感兴趣的用户是 `john`，它的 `id` 字段设置为 1。在过滤后，连接表如下所示：
| id | text            | user_id | follower_id | followed_id |
|----|----------------|---------|-------------|-------------|
| 1  | post from susan | 2       | 1           | 2           |
| 3  | post from david | 4       | 1           | 4           |

而这些恰好就是我想要的帖子！

请记住，查询是在`Post`类上执行的，因此尽管我最终得到了一个由数据库作为此查询的一部分创建的临时表，结果将是包含在此临时表中的帖子，而不是联接操作添加的额外列。

### Sorting

这个过程的最后一步是对结果进行排序。实现排序的部分查询语句如下：

```order_by(Post.timestamp.desc())```

这里我指定了根据每篇博客文章的时间戳字段进行降序排列的排序方式。按照这种排序方式，第一个结果将是最近的博客文章。

## Combining Own and Followed Posts

我在followed_posts()函数中使用的查询非常有用，但是有一个限制。人们希望在他们关注的用户的时间轴中看到自己的帖子，而现有的查询并没有这个能力。

有两种扩展此查询以包括用户自己的帖子的可能方法。最直接的方法是保持查询不变，但确保所有用户都在关注自己。如果您是自己的关注者，则如上所示的查询将找到您自己的帖子以及您关注的所有人的帖子。这种方法的缺点是它会影响有关关注者的统计数据。所有关注者计数都会增加一个，因此必须在显示之前进行调整。第二种方法是创建第二个查询，返回用户自己的帖子，然后使用"union"运算符将两个查询组合成一个单一的查询。

在考虑了两个选项之后，我决定选择第二个选项。下面可以看到扩展后的followed_posts()函数，其中通过union包含了用户的帖子：

_app/models.py_: Followed posts query with user's own posts.

```python
    def followed_posts(self):
        followed = Post.query.join(
            followers, (followers.c.followed_id == Post.user_id)).filter(
                followers.c.follower_id == self.id)
        own = Post.query.filter_by(user_id=self.id)
        return followed.union(own).order_by(Post.timestamp.desc())
```

注意在排序之前，关注的和自己的查询被合并成了一个查询。

## Unit Testing the User Model

尽管我认为我构建的关注者实现是一个“复杂”的功能，但我也认为它并不是简单的。当我编写非常规代码时，我的关注点是确保这些代码将在未来继续工作，因为我在应用程序的不同部分进行修改。确保您已经编写的代码将在未来继续工作的最好方法是创建一套自动化测试，您可以在每次进行更改时重新运行这些测试。

Python 包括一个非常有用的 `unittest` 包，使编写和执行单元测试变得容易。让我们在 tests.py 模块中为 `User` 类中的现有方法编写一些单元测试：

_tests.py_: User model unit tests.

```python
import os
os.environ['DATABASE_URL'] = 'sqlite://'

from datetime import datetime, timedelta
import unittest
from app import app, db
from app.models import User, Post

class UserModelCase(unittest.TestCase):
    def setUp(self):
        self.app_context = app.app_context()
        self.app_context.push()
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()

    def test_password_hashing(self):
        u = User(username='susan')
        u.set_password('cat')
        self.assertFalse(u.check_password('dog'))
        self.assertTrue(u.check_password('cat'))

    def test_avatar(self):
        u = User(username='john', email='john@example.com')
        self.assertEqual(u.avatar(128), ('https://www.gravatar.com/avatar/'
                                         'd4c74594d841139328695756648b6bd6'
                                         '?d=identicon&s=128'))

    def test_follow(self):
        u1 = User(username='john', email='john@example.com')
        u2 = User(username='susan', email='susan@example.com')
        db.session.add(u1)
        db.session.add(u2)
        db.session.commit()
        self.assertEqual(u1.followed.all(), [])
        self.assertEqual(u1.followers.all(), [])

        u1.follow(u2)
        db.session.commit()
        self.assertTrue(u1.is_following(u2))
        self.assertEqual(u1.followed.count(), 1)
        self.assertEqual(u1.followed.first().username, 'susan')
        self.assertEqual(u2.followers.count(), 1)
        self.assertEqual(u2.followers.first().username, 'john')

        u1.unfollow(u2)
        db.session.commit()
        self.assertFalse(u1.is_following(u2))
        self.assertEqual(u1.followed.count(), 0)
        self.assertEqual(u2.followers.count(), 0)

    def test_follow_posts(self):
        # create four users
        u1 = User(username='john', email='john@example.com')
        u2 = User(username='susan', email='susan@example.com')
        u3 = User(username='mary', email='mary@example.com')
        u4 = User(username='david', email='david@example.com')
        db.session.add_all([u1, u2, u3, u4])

        # create four posts
        now = datetime.utcnow()
        p1 = Post(body="post from john", author=u1,
                  timestamp=now + timedelta(seconds=1))
        p2 = Post(body="post from susan", author=u2,
                  timestamp=now + timedelta(seconds=4))
        p3 = Post(body="post from mary", author=u3,
                  timestamp=now + timedelta(seconds=3))
        p4 = Post(body="post from david", author=u4,
                  timestamp=now + timedelta(seconds=2))
        db.session.add_all([p1, p2, p3, p4])
        db.session.commit()

        # setup the followers
        u1.follow(u2)  # john follows susan
        u1.follow(u4)  # john follows david
        u2.follow(u3)  # susan follows mary
        u3.follow(u4)  # mary follows david
        db.session.commit()

        # check the followed posts of each user
        f1 = u1.followed_posts().all()
        f2 = u2.followed_posts().all()
        f3 = u3.followed_posts().all()
        f4 = u4.followed_posts().all()
        self.assertEqual(f1, [p2, p4, p1])
        self.assertEqual(f2, [p2, p3])
        self.assertEqual(f3, [p3, p4])
        self.assertEqual(f4, [p4])

if __name__ == '__main__':
    unittest.main(verbosity=2)
```

我已经添加了四个测试来测试用户模型中的密码哈希、用户头像和关注者功能。`setUp()`和`tearDown()`方法是特殊的方法，单元测试框架在每个测试之前和之后执行它们。

我实现了一个小技巧，以防止单元测试使用我用于开发的常规数据库。通过将`DATABASE_URL`环境变量设置为`sqlite://`，我改变了应用程序配置，以便在测试期间将SQLAlchemy定向到使用内存中的SQLite数据库。

然后，`setUp()`方法创建一个应用程序上下文并将其推送。这确保了Flask应用程序实例以及其配置数据对于Flask扩展是可访问的。如果您现在不太明白，也不用担心，因为稍后会更详细地介绍。

`db.create_all()`调用创建所有数据库表。这是一种从头开始创建数据库的快速方法，对于测试非常有用。对于开发和生产使用，我已经向您展示了如何通过数据库迁移创建数据库表。

您可以使用以下命令运行整个测试套件：

```Shell
(venv) $ python tests.py
test_avatar (__main__.UserModelCase) ... ok
test_follow (__main__.UserModelCase) ... ok
test_follow_posts (__main__.UserModelCase) ... ok
test_password_hashing (__main__.UserModelCase) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.494s

OK
```

从现在开始，每当对应用程序进行更改时，您都可以重新运行测试，以确保正在测试的功能未受到影响。此外，每次添加另一个功能到应用程序时，都应该为它编写单元测试.

## Integrating Followers with the Application

支持数据库和模型中的关注者功能现在已经完成，但我还没有将这些功能整合到应用程序中，所以我现在要添加它们。

因为关注和取消关注操作会引入应用程序中的更改，所以我将实现它们作为`POST`请求，这些请求是由Web浏览器通过提交Web表单触发的。虽然将这些路由实现为`GET`请求会更容易，但是它们可能会被利用进行[CSRF](http://en.wikipedia.org/wiki/Cross-site_request_forgery)攻击。因为`GET`请求很难防止CSRF攻击，在不引入状态更改的情况下才能使用。通过表单提交来实现这些操作更好，因为可以向表单添加CSRF令牌。

但是如果用户只需要点击“关注”或“取消关注”，而无需提交任何数据，则如何从Web表单触发跟随或取消跟随操作呢？为了使其正常工作，该表单将为空。该表单中唯一存在的元素将是CSRF令牌（作为隐藏字段自动添加），以及一个提交按钮（用户需要点击此按钮来触发操作）。由于两个操作几乎相同，我打算使用同一个表格来处理两个操作，并称之为空白表格（`EmptyForm`）。

_app/forms.py_: Empty form for following and unfollowing.

```python
class EmptyForm(FlaskForm):
    submit = SubmitField('Submit')
```

让我们在应用程序中添加两个新的路由来关注和取消关注用户：

_app/routes.py_: Follow and unfollow routes.

```python
from app.forms import EmptyForm

# ...

@app.route('/follow/<username>', methods=['POST'])
@login_required
def follow(username):
    form = EmptyForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=username).first()
        if user is None:
            flash('User {} not found.'.format(username))
            return redirect(url_for('index'))
        if user == current_user:
            flash('You cannot follow yourself!')
            return redirect(url_for('user', username=username))
        current_user.follow(user)
        db.session.commit()
        flash('You are following {}!'.format(username))
        return redirect(url_for('user', username=username))
    else:
        return redirect(url_for('index'))

@app.route('/unfollow/<username>', methods=['POST'])
@login_required
def unfollow(username):
    form = EmptyForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=username).first()
        if user is None:
            flash('User {} not found.'.format(username))
            return redirect(url_for('index'))
        if user == current_user:
            flash('You cannot unfollow yourself!')
            return redirect(url_for('user', username=username))
        current_user.unfollow(user)
        db.session.commit()
        flash('You are not following {}.'.format(username))
        return redirect(url_for('user', username=username))
    else:
        return redirect(url_for('index'))
```

这些路由中的表单处理更简单，因为我们只需要实现提交部分。与其他表单（如登录和编辑个人资料表单）不同，这两个表单没有自己的页面，它们将由`user()`路由呈现，并显示在用户的个人资料页面上。如果`validate_on_submit()`调用失败，则唯一可能原因是CSRF令牌丢失或无效，在这种情况下，我会将应用程序重定向回主页。

如果表单验证通过，则在执行关注或取消关注操作之前进行一些错误检查。这是为了防止意外问题，并尝试在出现问题时向用户提供有用的消息。

要呈现关注或取消关注按钮，我需要实例化一个`EmptyForm`对象并将其传递给user.html模板。因为这两个操作是互斥的，所以我可以将此通用表单的一个实例传递给模板：

_app/routes.py_: Follow and unfollow routes.

```python
@app.route('/user/<username>')
@login_required
def user(username):
    # ...
    form = EmptyForm()
    return render_template('user.html', user=user, posts=posts, form=form)
```

我现在可以在每个用户的个人资料页面中添加关注或取消关注的表单：

_app/templates/user.html_: Follow and unfollow links in user profile page.

```html
        ...
        <h1>User: {{ user.username }}</h1>
        {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
        {% if user.last_seen %}<p>Last seen on: {{ user.last_seen }}</p>{% endif %}
        <p>{{ user.followers.count() }} followers, {{ user.followed.count() }} following.</p>
        {% if user == current_user %}
        <p><a href="{{ url_for('edit_profile') }}">Edit your profile</a></p>
        {% elif not current_user.is_following(user) %}
        <p>
            <form action="{{ url_for('follow', username=user.username) }}" method="post">
                {{ form.hidden_tag() }}
                {{ form.submit(value='Follow') }}
            </form>
        </p>
        {% else %}
        <p>
            <form action="{{ url_for('unfollow', username=user.username) }}" method="post">
                {{ form.hidden_tag() }}
                {{ form.submit(value='Unfollow') }}
            </form>
        </p>
        {% endif %}
        ...
```

用户个人资料模板的更改在最后一次查看时间戳下面添加了一行，显示此用户有多少个关注者和被关注用户。当您查看自己的个人资料时，“编辑”链接所在的行现在可以有三个可能的链接：

* 如果用户查看自己的个人资料，则“编辑”链接与以前一样显示。
* 如果用户查看当前未关注的用户，则显示“关注”表单。
* 如果用户正在查看当前已关注的用户，则显示“取消关注”表单。

为了重用`EmptyForm（）`实例用于跟随和取消关注表单，我在呈现提交按钮时传递了一个`value`参数。在提交按钮中，`value`属性定义标签，因此通过这个技巧，我可以根据需要向用户呈现的操作更改提交按钮中的文本。

此时，您可以运行应用程序，创建一些用户并尝试关注和取消关注用户。您需要记住的唯一一件事是输入您想要关注或取消关注的用户的个人资料页面URL，因为目前没有查看用户列表的方法。例如，如果您想关注用户名为`susan`的用户，则需要在浏览器的地址栏中输入 _http://localhost:5000/user/susan_ 来访问该用户的个人资料页面。确保您检查当您发出关注或取消关注时，关注者和被关注者计数如何更改。

我应该在应用程序的首页中显示所关注的帖子列表，但我还没有完全准备好执行此操作，因为用户还不能写博客文章。因此，我将推迟此更改，直到该功能就绪为止。

# Chapter9: Pagination

在第8章中，我对数据库做了一些必要的修改，以支持社交网络中流行的 "追随者 "范式。有了这些功能，我就准备去掉我在一开始就设置好的最后一块脚手架，即假的帖子。在这一章中，应用程序将开始接受来自用户的博客文章，并在主页和个人资料页面中以分页列表的形式提供这些文章。

## Submission of Blog Posts

让我们从简单的事情开始。主页需要有一个表单，用户可以在其中输入新的帖子。首先，我创建一个表单类：

_app/forms.py_: Blog submission form.
```python
class PostForm(FlaskForm):
    post = TextAreaField('Say something', validators=[
        DataRequired(), Length(min=1, max=140)])
    submit = SubmitField('Submit')
```

接下来，我可以把这个表单添加到应用程序的主页面的模板中：

_app/templates/index.html_: Post submission form in index template
```html
{% extends "base.html" %}

{% block content %}
    <h1>Hi, {{ current_user.username }}!</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.post.label }}<br>
            {{ form.post(cols=32, rows=4) }}<br>
            {% for error in form.post.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
    {% for post in posts %}
    <p>
    {{ post.author.username }} says: <b>{{ post.body }}</b>
    </p>
    {% endfor %}
{% endblock %}
```

这个模板中的变化与以前的表单处理方式类似。最后一部分是在视图函数中添加表单的创建和处理：

_app/routes.py_: Post submission form in index view function.
```python
from app.forms import PostForm
from app.models import Post

@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    form = PostForm()
    if form.validate_on_submit():
        post = Post(body=form.post.data, author=current_user)
        db.session.add(post)
        db.session.commit()
        flash('Your post is now live!')
        return redirect(url_for('index'))
    posts = [
        {
            'author': {'username': 'John'},
            'body': 'Beautiful day in Portland!'
        },
        {
            'author': {'username': 'Susan'},
            'body': 'The Avengers movie was so cool!'
        }
    ]
    return render_template("index.html", title='Home Page', form=form,
                           posts=post
```

让我们逐一回顾一下这个视图函数中的变化：

* 我现在导入了`Post`和`PostForm`类
* 除了`GET`请求外，我还在与`index`视图函数相关的两个路由中接受`POST`请求，因为这个视图函数现在将接收表单数据。
* 表单处理逻辑在数据库中插入一个新的`Post`记录。
* 模板接收`form`对象作为一个额外的参数，这样它就可以渲染文本字段。

在我继续之前，我想提一下与处理网络表单有关的一些重要事情。请注意在我处理完表单数据后，我是如何通过发出一个重定向到主页来结束请求的。我本可以很容易地跳过重定向，让函数继续向下进入模板渲染部分，因为这已经是索引视图函数了。

那么，为什么要重定向呢？用重定向来响应由网络表单提交产生的`POST`请求是一种标准做法。这有助于减轻刷新命令在网络浏览器中的实现方式的烦恼。当你按下刷新键时，网络浏览器所做的就是重新发出最后的请求。如果一个带有表单提交的`POST`请求返回一个普通的响应，那么刷新将重新提交表单。因为这是出乎意料的，浏览器会要求用户确认重复提交，但大多数用户不会理解浏览器在问他们什么。但是如果一个`POST`请求得到了重定向的回答，浏览器现在被指示发送一个`GET`请求来抓取重定向中指示的页面，所以现在最后一个请求不再是`POST`请求了，刷新命令以一种更可预测的方式工作。

这个简单的技巧被称为[帖子/重定向/获取模式](https://en.wikipedia.org/wiki/Post/Redirect/Get)。它可以避免在用户提交网络表单后无意中刷新页面时插入重复的帖子。

## Displaying Bolg Posts

如果你还记得，我创建了几个假的博客文章，在主页上显示了很长时间。这些假对象是在索引视图函数中明确创建的，是一个简单的Python列表：

```
    posts = [
        { 
            'author': {'username': 'John'}, 
            'body': 'Beautiful day in Portland!' 
        },
        { 
            'author': {'username': 'Susan'}, 
            'body': 'The Avengers movie was so cool!' 
        }
    ]
```

但现在我在用户模型中有 followed_posts() 方法，该方法返回一个特定用户想看的帖子的查询。所以现在我可以用真实的帖子来代替假的帖子：

_app/routes.py_: Display real posts in home page.

```python
@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    # ...
    posts = current_user.followed_posts().all()
    return render_template("index.html", title='Home Page', form=form,
                           posts=posts)
```

`User`类的 `followed_posts` 方法返回一个 SQLAlchemy 查询对象，该对象被配置为从数据库中抓取用户感兴趣的帖子。对这个查询调用`all()`会触发其执行，返回值是一个包含所有结果的列表。因此，我最终得到的结构非常类似于到目前为止我一直在使用的带有假帖子的结构。它是如此接近，以至于模板甚至不需要改变。

## Making It Easier to Find Users to Follow

我相信你已经注意到了，目前的应用程序在让用户找到其他用户进行关注方面做得不是很好。事实上，实际上根本就没有办法看到其他用户在那里。我将通过一些简单的改变来解决这个问题。

我将创建一个新的页面，我将称之为 `"Explore"` 页面。这个页面将像主页一样工作，但不是只显示被关注用户的帖子，而是显示所有用户的全球帖子流。这里是新的探索视图功能：

_app/routes.py_: Explore view function.

```python
@app.route('/explore')
@login_required
def explore():
    posts = Post.query.order_by(Post.timestamp.desc()).all()
    return render_template('index.html', title='Explore', posts=posts)
```

你是否注意到这个视图函数中的一些奇怪之处？`render_template()`调用引用了index.html模板，我在应用程序的主页面中使用了它。由于这个页面将与主页面非常相似，我决定重新使用这个模板。但与主页面不同的是，在探索页面中，我不希望有一个写博文的表单，所以在这个视图函数中，我没有在模板调用中包含`form`参数。

为了防止index.html模板在试图呈现一个不存在的Web表单时崩溃，我将添加一个条件，只有在表单被定义时才会呈现：

_app/templates/index.html_: Make the blog post submission form optional.

```html
{% extends "base.html" %}

{% block content %}
    <h1>Hi, {{ current_user.username }}!</h1>
    {% if form %}
    <form action="" method="post">
        ...
    </form>
    {% endif %}
    ...
{% endblock %}
```

我还将在导航栏中添加一个指向这个新页面的链接：

_app/templates/base.html_: Link to explore page in navigation bar.

```        <a href="{{ url_for('explore') }}">Explore</a>```

还记得我在第6章中介绍的_post.html子模板吗，它可以在用户资料页中呈现博客文章。这是一个从用户资料页模板中包含的小模板，而且是独立的，这样它也可以从其他模板中使用。我现在要对它做一个小小的改进，那就是把博文作者的用户名作为一个链接显示出来：

_app/templates/_post.html_: Show link to author in blog posts.

```html
    <table>
        <tr valign="top">
            <td><img src="{{ post.author.avatar(36) }}"></td>
            <td>
                <a href="{{ url_for('user', username=post.author.username) }}">
                    {{ post.author.username }}
                </a>
                says:<br>{{ post.body }}
            </td>
        </tr>
    </table>
```

现在我可以使用这个子模板在主页和探索页中呈现博客文章：

_app/templates/index.html_: Use blog post sub-template.

```html
    ...
    {% for post in posts %}
        {% include '_post.html' %}
    {% endfor %}
    ...
```

子模板希望有一个名为`post`的变量存在，而索引模板中的循环变量也是这样命名的，所以工作起来非常完美。

通过这些小改动，应用程序的可用性得到了很大的改善。现在，用户可以访问探索页面，阅读未知用户的博文，并根据这些博文寻找新的用户进行关注，这可以通过简单地点击一个用户名来访问个人资料页面来完成。很神奇，对吗？

在这一点上，我建议你再试一次这个应用程序，以便你体验这些最后的用户界面改进。

## Pagination of Blog Posts

该应用程序看起来比以往任何时候都好，但在主页上显示所有被关注的帖子，迟早会成为一个问题。如果一个用户有一千个被关注的帖子会怎样？或者一百万个？你可以想象，管理这么大的帖子列表将是非常缓慢和低效的。

为了解决这个问题，我将对帖子列表进行分页。这意味着，最初我打算一次只显示有限数量的帖子，并包括链接来浏览整个帖子列表。Flask-SQLAlchemy通过`paginate()`查询方法原生支持分页。例如，如果我想获得用户的前20个被关注的帖子，我可以把终止查询的all()调用改为：

```>>> user.followed_posts().paginate(page=1, per_page=20, error_out=False).items```

`paginate`方法可以在Flask-SQLAlchemy的任何查询对象上调用。它需要三个参数：

* 页数，从1开始
* 每页的项目数
* 一个错误标志。如果是`True`，当一个超出范围的页面被请求时，将自动返回给客户端一个404错误。如果是假的，超出范围的页面将返回一个空列表。

`paginate`的返回值是一个`Pagination`对象。这个对象的 `items` 属性包含了请求页面中的项目列表。在`Pagination`对象中还有其他有用的东西，我将在后面讨论。

现在让我们思考一下我如何在`index()`视图函数中实现分页。我可以先在应用程序中添加一个配置项，确定每页将显示多少个项目。

_config.py_: Posts per page configuration.

```
class Config(object):
    # ...
    POSTS_PER_PAGE = 3
```

在配置文件中设置这些可以改变行为的应用范围的 "旋钮 "是个好主意，因为这样我就可以到一个地方去做调整。在最终的应用中，我当然会使用比每页三个项目更大的数字，但对于测试来说，用小数字工作是很有用的。

接下来，我需要决定如何将页码纳入应用程序的URL。一个相当常见的方法是使用一个查询字符串参数来指定一个可选的页码，如果没有给出，则默认为第1页。下面是一些URL的例子，说明我将如何实现这一点：

* 第1页，隐式：http://localhost:5000/index
* 第1页，显式：http://localhost:5000/index?page=1
* 第3页：http://localhost:5000/index?page=3

为了访问查询字符串中的参数，我可以使用Flask的`request.args`对象。你已经在第5章中看到了这一点，在那里我从Flask-Login中实现了用户登录URL，可以包括`next`查询字符串参数。

下面你可以看到我是如何给主页和探索视图功能添加分页的：

_app/routes.py_: Followers association table

```python
@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    # ...
    page = request.args.get('page', 1, type=int)
    posts = current_user.followed_posts().paginate(
        page=page, per_page=app.config['POSTS_PER_PAGE'], error_out=False)
    return render_template('index.html', title='Home', form=form,
                           posts=posts.items)

@app.route('/explore')
@login_required
def explore():
    page = request.args.get('page', 1, type=int)
    posts = Post.query.order_by(Post.timestamp.desc()).paginate(
        page=page, per_page=app.config['POSTS_PER_PAGE'], error_out=False)
    return render_template("index.html", title='Explore', posts=posts.items)
```

有了这些变化，这两条路由就会根据`page`查询字符串参数或默认的1来确定要显示的页码，然后使用`paginate()`方法只检索所需的结果页面。决定页面大小的`POSTS_PER_PAGE`配置项是通过`app.config`对象访问的。

请注意，这些变化是多么容易，而且每次变化时受影响的代码很少。我试图在不对其他部分的工作方式做任何假设的情况下编写应用程序的每一部分，这使我能够编写模块化和健壮的应用程序，这些应用程序更容易扩展和测试，而且不太可能失败或出现错误。

来吧，试试分页支持。首先要确保你有三篇以上的博客文章。这在探索页中比较容易看到，它显示所有用户的帖子。你现在要看到的只是最近的三篇文章。如果你想看后面三篇，请在浏览器的地址栏中输入http://localhost:5000/explore?page=2。

## Page Navigation

下一个变化是在博文列表的底部添加链接，让用户可以浏览到下一页和/或上一页。还记得我提到`paginate()`调用的返回值是Flask-SQLAlchemy的Pagination类的一个对象吗？到目前为止，我已经使用了这个对象的 items 属性，它包含了为所选页面检索的项目列表。但是这个对象还有一些其他的属性，在建立分页链接时很有用：

* `has_next`： 如果在当前页面之后至少还有一个页面，则为真
* `has_prev`：如果在当前页面之前至少还有一个页面，则为真。
* `next_num`: 下一页的页数
* `prev_num`: 前一页的页码

有了这四个元素，我就可以生成下一页和上一页的链接，并把它们传递给模板进行渲染：

_app/routes.py_: Next and previous page links.

```python
@app.route('/', methods=['GET', 'POST'])
@app.route('/index', methods=['GET', 'POST'])
@login_required
def index():
    # ...
    page = request.args.get('page', 1, type=int)
    posts = current_user.followed_posts().paginate(
        page=page, per_page=app.config['POSTS_PER_PAGE'], error_out=False)
    next_url = url_for('index', page=posts.next_num) \
        if posts.has_next else None
    prev_url = url_for('index', page=posts.prev_num) \
        if posts.has_prev else None
    return render_template('index.html', title='Home', form=form,
                           posts=posts.items, next_url=next_url,
                           prev_url=prev_url)

 @app.route('/explore')
 @login_required
 def explore():
    page = request.args.get('page', 1, type=int)
    posts = Post.query.order_by(Post.timestamp.desc()).paginate(
        page=page, per_page=app.config['POSTS_PER_PAGE'], error_out=False)
    next_url = url_for('explore', page=posts.next_num) \
        if posts.has_next else None
    prev_url = url_for('explore', page=posts.prev_num) \
        if posts.has_prev else None
    return render_template("index.html", title='Explore', posts=posts.items,
                          next_url=next_url, prev_url=prev_url)
```

这两个视图函数中的`next_url`和`prev_url`将被设置为由`url_for()`返回的URL，只有在该方向有一个页面的情况下。如果当前页面位于帖子集合的某一端，那么分页对象的`has_next`或`has_prev`属性将为假，在这种情况下，该方向的链接将被设置为无。

`url_for()`函数有一个有趣的地方，我以前没有讨论过，就是你可以向它添加任何关键字参数，如果这些参数的名字没有在URL中直接引用，那么Flask会把它们作为查询参数包含在URL中。

分页链接被设置为index.html模板，所以现在让我们在页面上渲染它们，就在帖子列表的下面：

_app/templates/index.html_: Render pagination links on the template.
```html
    ...
    {% for post in posts %}
        {% include '_post.html' %}
    {% endfor %}
    {% if prev_url %}
    <a href="{{ prev_url }}">Newer posts</a>
    {% endif %}
    {% if next_url %}
    <a href="{{ next_url }}">Older posts</a>
    {% endif %}
    ...
```

这一改动在索引和探索页面的帖子列表下面增加了两个链接。第一个链接被标记为 "较新的帖子"，它指向前一页（请记住，我显示的帖子是按最新的先排序的，所以第一页是有最新内容的那一页）。第二个链接被标记为 "较旧的帖子"，指向下一页的帖子。如果这两个链接中的任何一个是 "无"，那么它将通过一个条件从页面中省略。

![ch09-pagination.png](/assets/img/ch09-pagination.png "ch09-pagination.png")

## Pagination in the User Profile Page

目前，对索引页的修改已经足够了。然而，在用户资料页中也有一个帖子列表，它只显示资料所有者的帖子。为了保持一致，应该改变用户资料页，使其与索引页的分页风格一致。

我首先更新了用户档案查看功能，该功能中仍然有一个假的帖子对象列表。

_app/routes.py_: Pagination in the user profile view function.

```python
@app.route('/user/<username>')
@login_required
def user(username):
    user = User.query.filter_by(username=username).first_or_404()
    page = request.args.get('page', 1, type=int)
    posts = user.posts.order_by(Post.timestamp.desc()).paginate(
        page=page, per_page=app.config['POSTS_PER_PAGE'], error_out=False)
    next_url = url_for('user', username=user.username, page=posts.next_num) \
        if posts.has_next else None
    prev_url = url_for('user', username=user.username, page=posts.prev_num) \
        if posts.has_prev else None
    form = EmptyForm()
    return render_template('user.html', user=user, posts=posts.items,
                           next_url=next_url, prev_url=prev_url, form=for
```
                          
为了获得用户的帖子列表，我利用了user.post关系是SQLAlchemy已经设置好的查询，这是用户模型中db.relationship()定义的结果。我利用这个查询并添加一个order_by()子句，这样我就可以先得到最新的帖子，然后像我对索引和探索页面中的帖子那样进行分页处理。请注意，由url_for()函数生成的分页链接需要额外的用户名参数，因为它们是指向用户配置文件页面的，该页面的用户名是URL的一个动态组成部分。

最后，对user.html模板的修改与我在索引页上的修改相同：

_app/templates/user.html_: Pagination links in the user profile template.
```html
    ...
    {% for post in posts %}
        {% include '_post.html' %}
    {% endfor %}
    {% if prev_url %}
    <a href="{{ prev_url }}">Newer posts</a>
    {% endif %}
    {% if next_url %}
    <a href="{{ next_url }}">Older posts</a>
    {% endif %}
```

在你完成了分页功能的实验后，你可以将POSTS_PER_PAGE配置项设置为一个更合理的值：

_config.py_: Posts per page configuration.

```python
class Config(object):
    # ...
    POSTS_PER_PAGE = 25
```


# Chapter10: Email Support

## Introduction to Flask-Mail

至于实际的电子邮件发送，Flask有一个流行的扩展，叫做Flask-Mail，可以使这个任务非常容易。一如既往，这个扩展是用pip安装的：

```(venv) $ pip install flask-mail```

密码重置链接中会有一个安全令牌。为了生成这些令牌，我将使用JSON Web令牌，它也有一个流行的Python包：

```(venv) $ pip install pyjwt```

Flask-Mail扩展是通过app.config对象配置的。还记得在第7章中我添加了电子邮件配置，以便在生产中出现错误时给自己发送电子邮件吗？当时我没有告诉你，但我对配置变量的选择是以Flask-Mail的要求为模型的，所以其实不需要任何额外的工作，配置变量已经在应用程序中了。

像大多数Flask扩展一样，你需要在Flask应用程序创建后立即创建一个实例。在这种情况下，这是一个邮件类的对象：

_app/__init__.py_: Flask-Mail instance.

```python
# ...
from flask_mail import Mail

app = Flask(__name__)
# ...
mail = Mail(app)
```

如果你打算测试电子邮件的发送，你有我在第7章中提到的同样的选择。如果你想使用一个模拟的电子邮件服务器，Python提供了一个非常方便的服务器，你可以在第二个终端用下面的命令启动：

```(venv) $ python -m smtpd -n -c DebuggingServer localhost:8025```

要为这个服务器进行配置，你需要设置两个环境变量：

```
(venv) $ export MAIL_SERVER=localhost
(venv) $ export MAIL_PORT=8025
```

如果你喜欢真实地发送电子邮件，你需要使用一个真正的电子邮件服务器。如果你有一个，那么你只需要为它设置`MAIL_SERVER`、`MAIL_PORT`、`MAIL_USE_TLS`、`MAIL_USERNAME`和`MAIL_PASSWORD`环境变量。如果你想要一个快速的解决方案，你可以使用Gmail账户来发送电子邮件，设置如下：

```
(venv) $ export MAIL_SERVER=smtp.googlemail.com
(venv) $ export MAIL_PORT=587
(venv) $ export MAIL_USE_TLS=1
(venv) $ export MAIL_USERNAME=<your-gmail-username>
(venv) $ export MAIL_PASSWORD=<your-gmail-password>
```

如果你使用的是微软的Windows系统，你需要在上面的每一个`export`语句中用`set`代替`export`。

记住，你的Gmail账户的安全功能可能会阻止应用程序通过它发送电子邮件，除非你明确允许 "不太安全的应用程序 "访问你的Gmail账户。你可以在[这里](https://support.google.com/accounts/answer/6010255?hl=en)阅读这方面的内容，如果你担心你的账户的安全性，你可以创建一个辅助账户，只为测试电子邮件而配置，或者你可以只暂时启用安全性较低的应用程序来运行你的测试，然后再恢复到更安全的默认值。

如果你想使用一个真正的电子邮件服务器，但又不想让自己与Gmail的配置复杂化，SendGrid是一个不错的选择，它可以让你使用免费账户每天发送100封电子邮件。

## Flask-Mail Usage

为了学习Flask-Mail的工作原理，我将向你展示如何从Python shell中发送电子邮件。所以用flask shell启动Python，然后运行以下命令：

```Shell
>>> from flask_mail import Message
>>> from app import mail
>>> msg = Message('test subject', sender=app.config['ADMINS'][0],
... recipients=['your-email@example.com'])
>>> msg.body = 'text body'
>>> msg.html = '<h1>HTML body</h1>'
>>> mail.send(msg)
```

上面的代码片段将向你放在收件人参数中的电子邮件地址列表发送一封邮件。我把发件人设为第一个配置的管理员（我在第七章添加了ADMINS配置变量）。这封邮件将有纯文本和HTML版本，所以取决于你的电子邮件客户端是如何配置的，你可能会看到一个或另一个。

所以正如你所看到的，这是很简单的。现在让我们把电子邮件集成到应用程序中。

## A Simple Email Framework

我将首先编写一个发送电子邮件的辅助函数，这基本上是上一节中shell练习的一个通用版本。我将把这个函数放在一个名为`app/email.py`的新模块中：

_app/email.py_: Email sending wrapper function.
```python
from flask_mail import Message
from app import mail

def send_email(subject, sender, recipients, text_body, html_body):
    msg = Message(subject, sender=sender, recipients=recipients)
    msg.body = text_body
    msg.html = html_body
    mail.send(msg)
```

Flask-Mail支持一些我在这里没有利用的功能，比如抄送和密送列表。如果你对这些选项感兴趣，请务必查看Flask-Mail文档。

## Requesting a Password Reset

正如我上面提到的，我希望用户可以选择要求重置他们的密码。为此，我将在登录页面中添加一个链接：

_app/templates/login.html_: Password reset link in login form.

```html
    <p>
        Forgot Your Password?
        <a href="{{ url_for('reset_password_request') }}">Click to Reset It</a>
    </p>
```

当用户点击该链接时，将出现一个新的网络表单，要求用户提供电子邮件地址，以此来启动密码重置过程。这里是表单类：

_app/forms.py_: Reset password request form.

```python
class ResetPasswordRequestForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    submit = SubmitField('Request Password Reset')
```

而这里是相应的HTML模板：

_app/templates/reset_password_request.html_: Reset password request template.
```html
{% extends "base.html" %}

{% block content %}
    <h1>Reset Password</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.email.label }}<br>
            {{ form.email(size=64) }}<br>
            {% for error in form.email.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

我还需要一个视图函数来处理这个表单：

_app/routes.py_: Reset password request view function.
```python
from app.forms import ResetPasswordRequestForm
from app.email import send_password_reset_email

@app.route('/reset_password_request', methods=['GET', 'POST'])
def reset_password_request():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    form = ResetPasswordRequestForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user:
            send_password_reset_email(user)
        flash('Check your email for the instructions to reset your password')
        return redirect(url_for('login'))
    return render_template('reset_password_request.html',
                           title='Reset Password', form=form)
```

这个视图功能与其他处理表单的功能相当类似。我首先要确定用户没有登录。如果用户已经登录，那么使用密码重置功能就没有意义了，所以我重定向到索引页。

当表单被提交并且有效时，我通过用户在表单中提供的电子邮件来查找用户。如果我找到了这个用户，我就发送一封密码重置邮件。`send_password_reset_email()`辅助函数完成了这个任务。我将在下面向你展示这个函数。

邮件发送后，我闪现一条信息，引导用户寻找邮件以获得进一步的指示，然后重定向到登录页面。你可能会注意到，即使用户提供的电子邮件是未知的，也会显示闪现的信息。这是为了让客户无法使用这个表格来确定某个用户是否是会员。

## Password Reset Tokens

在我实现`send_password_reset_email()`函数之前，我需要有一种方法来生成一个密码请求链接。这将是通过电子邮件发送给用户的链接。当该链接被点击时，一个可以设置新密码的页面就会呈现给用户。这个计划的棘手之处在于，要确保只有有效的重置链接可以用来重置账户密码。

这些链接将被配置一个令牌，这个令牌将在允许更改密码之前被验证，以证明要求发送电子邮件的用户能够访问账户上的电子邮件地址。对于这种类型的过程，一个非常流行的令牌标准是JSON Web令牌，或JWT。JWT的好处是，它们是自包含的。你可以在电子邮件中向用户发送一个令牌，当用户点击将令牌送回应用程序的链接时，它可以自行验证。

JWTs是如何工作的？没有什么比一个快速的Python shell会话来了解它们更好了：

```Shell
>>> import jwt
>>> token = jwt.encode({'a': 'b'}, 'my-secret', algorithm='HS256')
>>> token
'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhIjoiYiJ9.dvOo58OBDHiuSHD4uW88nfJik_sfUHq1mDi4G0'
>>> jwt.decode(token, 'my-secret', algorithms=['HS256'])
{'a': 'b'}

```

`{'a': 'b'}`字典是一个将被写入令牌的有效载荷的例子。为了使令牌安全，需要提供一个秘密密钥，用于创建一个加密签名。在这个例子中，我使用了字符串`'my-secret'`，但是在应用中我将使用配置中的`SECRET_KEY`。算法参数指定了令牌的生成方式。`HS256`算法是最广泛使用的算法。

正如你所看到的，生成的令牌是一长串的字符。但不要以为这是一个加密的令牌。令牌的内容，包括有效载荷，可以被任何人轻易地解码（不相信我？复制上述令牌，然后在JWT调试器中输入它，就可以看到它的内容）。使得令牌安全的是，有效载荷被签名。如果有人试图伪造或篡改令牌中的有效载荷，那么签名就会失效，而要生成一个新的签名，则需要秘密密钥。当一个令牌被验证时，有效载荷的内容被解码并返回给调用者。如果令牌的签名被验证了，那么有效载荷就可以被信任为真实的。

我将用于密码重置令牌的有效载荷的格式是`{'reset_password': user_id, 'exp': token_expiration}`。`exp`字段是JWTs的标准字段，如果存在，它表示令牌的到期时间。如果一个令牌有一个有效的签名，但它已经超过了它的过期时间戳，那么它也将被视为无效的。对于密码重置功能，我打算给这些令牌10分钟的寿命。

由于这些令牌属于用户，我将把令牌生成和验证功能写成用户模型中的方法：

_app/models.py_: Reset password token methods.
```python
from time import time
import jwt
from app import app

class User(UserMixin, db.Model):
    # ...

    def get_reset_password_token(self, expires_in=600):
        return jwt.encode(
            {'reset_password': self.id, 'exp': time() + expires_in},
            app.config['SECRET_KEY'], algorithm='HS256')

    @staticmethod
    def verify_reset_password_token(token):
        try:
            id = jwt.decode(token, app.config['SECRET_KEY'],
                            algorithms=['HS256'])['reset_password']
        except:
            return
        return User.query.get(id)
```

`get_reset_password_token()`函数返回一个JWT令牌的字符串，它是由`jwt.encode()`函数直接生成的。

`verify_reset_password_token()`是一个静态方法，这意味着它可以直接从类中被调用。静态方法与类方法类似，唯一的区别是静态方法不接收类作为第一个参数。这个方法接收一个标记，并试图通过调用PyJWT的`jwt.decode()`函数对其进行解码。如果令牌不能被验证或过期，就会产生一个异常，在这种情况下，我会捕捉它以防止错误发生，然后向调用者返回`None`。如果令牌是有效的，那么来自令牌有效载荷的`reset_password`键的值就是用户的ID，所以我可以加载用户并返回它。

## Sending a Password Reset Email

`send_password_reset_email()`函数依赖于我上面写的`send_email()`函数来生成密码重置邮件。

_app/email.py_: Send password reset email function.
```python
from flask import render_template
from app import app

# ...

def send_password_reset_email(user):
    token = user.get_reset_password_token()
    send_email('[Microblog] Reset Your Password',
               sender=app.config['ADMINS'][0],
               recipients=[user.email],
               text_body=render_template('email/reset_password.txt',
                                         user=user, token=token),
               html_body=render_template('email/reset_password.html',
                                         user=user, token=token))
```

这个函数中有趣的部分是，电子邮件的文本和HTML内容是使用熟悉的`render_template()`函数从模板生成的。模板接收用户和令牌作为参数，这样就可以生成一个个性化的电子邮件信息。下面是重置密码邮件的文本模板：

_app/templates/email/reset_password.txt_: Text for password reset email.

```
Dear {{ user.username }},

To reset your password click on the following link:

{{ url_for('reset_password', token=token, _external=True) }}

If you have not requested a password reset simply ignore this message.

Sincerely,

The Microblog Team
```

这里是同一封电子邮件的较好的HTML版本：

_app/templates/email/reset_password.html_: HTML for password reset email.

```html
<p>Dear {{ user.username }},</p>
<p>
    To reset your password
    <a href="{{ url_for('reset_password', token=token, _external=True) }}">
        click here
    </a>.
</p>
<p>Alternatively, you can paste the following link in your browser's address bar:</p>
<p>{{ url_for('reset_password', token=token, _external=True) }}</p>
<p>If you have not requested a password reset simply ignore this message.</p>
<p>Sincerely,</p>
<p>The Microblog Team</p>
```

在这两个电子邮件模板的`url_for()`调用中引用的`reset_password`路由还不存在，这将在下一节添加。我在两个模板的`url_for()`调用中包含的`_external=True`参数也是新的。默认情况下，由`url_for()`生成的URL是相对URL，只包括URL的路径部分。这对于在网页中生成的链接来说通常是足够的，因为网络浏览器通过从地址栏中的URL中获取缺失的部分来完成URL的生成。然而，当通过电子邮件发送URL时，这种情况并不存在，所以需要使用完全合格的URLs。当`_external=True`作为一个参数被传递时，完整的URL会被生成，所以前面的例子会返回 _http://localhost:5000/user/susan_，或者当应用程序被部署在一个域名上时，会返回适当的URL。

## Resetting a User Password

当用户点击电子邮件链接时，与此功能相关的第二条路由被触发。这里是密码请求视图功能：

_app/routes.py_: Password reset view function.

```python
from app.forms import ResetPasswordForm

@app.route('/reset_password/<token>', methods=['GET', 'POST'])
def reset_password(token):
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    user = User.verify_reset_password_token(token)
    if not user:
        return redirect(url_for('index'))
    form = ResetPasswordForm()
    if form.validate_on_submit():
        user.set_password(form.password.data)
        db.session.commit()
        flash('Your password has been reset.')
        return redirect(url_for('login'))
    return render_template('reset_password.html', form=form)
```

在这个视图函数中，我首先确保用户没有登录，然后我通过调用`User`类中的令牌验证方法来确定用户是谁。如果令牌有效，该方法将返回用户，如果无效，则返回`None`。如果令牌无效，我会重定向到主页。

如果令牌是有效的，那么我将向用户展示第二个表单，其中要求提供新密码。这个表单的处理方式与之前的表单类似，作为一个有效表单提交的结果，我调用User的`set_password()`方法来改变密码，然后重定向到登录页面，用户现在可以登录。

下面是ResetPasswordForm类：

_app/forms.py_: Password reset form.

```python
class ResetPasswordForm(FlaskForm):
    password = PasswordField('Password', validators=[DataRequired()])
    password2 = PasswordField(
        'Repeat Password', validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Request Password Reset')
```

而这里是相应的HTML模板：

_app/templates/reset_password.html_: Password reset form template.

```html
{% extends "base.html" %}

{% block content %}
    <h1>Reset Your Password</h1>
    <form action="" method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.password.label }}<br>
            {{ form.password(size=32) }}<br>
            {% for error in form.password.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>
            {{ form.password2.label }}<br>
            {{ form.password2(size=32) }}<br>
            {% for error in form.password2.errors %}
            <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
{% endblock %}
```

密码重置功能现在已经完成，所以请确保你尝试。

## Asynchronous Emails

如果你正在使用Python提供的模拟电子邮件服务器，你可能没有注意到这一点，但是发送电子邮件会大大降低应用程序的速度。发送邮件时需要发生的所有交互使任务变得很慢，通常需要几秒钟才能发出一封邮件，如果收件人的邮件服务器很慢，或者有多个收件人，可能会更多。

我真正想要的是让`send_email()`函数成为异步的。那是什么意思？它意味着当这个函数被调用时，发送邮件的任务被安排在后台发生，释放`send_email()`立即返回，这样应用程序就可以在发送邮件的同时继续运行。

Python支持运行异步任务，实际上不止一种方式。线程和多进程模块都可以做到这一点。为正在发送的邮件启动一个后台线程比启动一个全新的进程要节省资源，所以我打算采用这种方法：

_app/email.py_: Send emails asynchronously.

```python
from threading import Thread
# ...

def send_async_email(app, msg):
    with app.app_context():
        mail.send(msg)


def send_email(subject, sender, recipients, text_body, html_body):
    msg = Message(subject, sender=sender, recipients=recipients)
    msg.body = text_body
    msg.html = html_body
    Thread(target=send_async_email, args=(app, msg)).start()
```

`send_async_email`函数现在在一个后台线程中运行，在`send_email()`的最后一行通过`Thread`类调用。有了这个变化，电子邮件的发送将在线程中运行，而当这个过程完成后，线程将结束并清理自己。如果你已经配置了一个真正的电子邮件服务器，当你按下密码重置请求表的提交按钮时，你一定会注意到速度的提高。

你可能以为只有`msg`参数会被发送到线程中，但正如你在代码中看到的，我也在发送应用程序实例。在使用线程时，Flask有一个重要的设计方面需要牢记。Flask使用上下文(_contexts_)来避免跨函数传递参数。我不打算在这方面做很多详细的介绍，但要知道有两种类型的上下文，应用上下文(_application context_)和请求上下文(_request context_)。在大多数情况下，这些上下文是由框架自动管理的，但当应用程序启动自定义线程时，这些线程的上下文可能需要手动创建。

有很多扩展需要有应用上下文才能工作，因为这可以让它们找到Flask应用实例，而不需要把它作为一个参数传递。许多扩展之所以需要知道应用实例，是因为它们的配置存储在`app.config`对象中。这正是Flask-Mail的情况。`mail.send()`方法需要访问电子邮件服务器的配置值，而这只能通过知道应用程序是什么来实现。用`app.app_context()`调用创建的应用程序上下文使应用程序实例可以通过Flask的`current_app`变量访问。