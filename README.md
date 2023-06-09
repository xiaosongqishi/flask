# The Flask Mega-Tutorial

This is a self learning project
个人学习，转载翻译自 _https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world_

How to run:

command:
```Shell
>flask run
```
然后在浏览器打开:`127.0.0.1:5000`


查询数据库:

```
sqlite3 app.db
```


## 文件介绍：

```Shell
.
├── app --app目录
│   ├── forms.py --形式py
│   ├── __init__.py --初始化文件
│   ├── models.py --模型
│   ├── __pycache__
│   ├── routes.py --路由py
│   └── templates --模板文件夹，包含html
├── app.db --数据库
├── assets --说明文档的素材文件夹
│   └── img
├── config.py --配置py
├── microblog.py 
├── migrations
├── README.md
├── sourse_zip --对应资源文件
├── Tutor.md --web对应章节翻译
└── venv

```

## 项目介绍
[Flask](https://flask.palletsprojects.com/en/2.2.x/)是一个Python Web应用程序框架，基于Werkzeug工具箱和Jinja2模板引擎开发。它被设计为简单而灵活的框架，适用于从小型项目到大型应用程序的各种需求。由于其轻量级和灵活性，Flask常被用于快速原型开发，以及小型应用程序和API的开发。

Flask的核心是Werkzeug工具箱，它提供了一些实用的工具，如路由、调试器、请求和响应对象等。Jinja2模板引擎则提供了一种简单易用的方式来渲染HTML页面，并支持继承、过滤器和宏等功能。Flask还提供了大量的扩展和插件，可以方便地添加额外的功能，如数据库连接、身份验证和表单处理等。

Flask的优点在于它简单易学，提供了足够的灵活性来满足不同的需求。它不会强制性地规定应用程序的结构，而是允许开发者自由组织代码，更好地适应项目的需求。另外，Flask的扩展性也很好，开发者可以根据自己的需求自由添加或删除扩展，从而实现更加精简的应用程序。
