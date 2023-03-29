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

![data](/assets/img/data.jpg "data")

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