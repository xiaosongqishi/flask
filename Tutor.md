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

