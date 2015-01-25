.. _first:


在 Flask 中使用 Celery
=======================

.. image:: images/flask-celery.png

后台运行任务的话题是有些复杂，因为围绕这个话题会让人产生困惑。为了简单起见，在以前我所有的例子中，我都是在线程中执行后台任务，但是我一直注意到更具有扩展性以及具备生产解决方案的任务队列像 `Celery <http://www.celeryproject.org/>`_  应该可以替代线程中执行后台任务。

不断有读者问我关于 Celery 问题，以及怎样在 Flask 应用中使用它，因此今天我将会向你们展示两个例子，我希望能够覆盖大部分的应用需求。


什么是 Celery?
--------------

Celery 是一个异步任务队列。你可以使用它在你的应用上下文之外执行任务。总的想法就是你的应用程序可能需要执行任何消耗资源的任务都可以交给任务队列，让你的应用程序自由和快速地响应客户端请求。

使用 Celery 运行后台任务并不像在线程中这样做那么简单。但是好处多多，Celery 具有分布式架构，使你的应用易于扩展。一个 Celery 安装有三个核心组件：

1. Celery 客户端: 用于发布后台作业。当与 Flask 一起工作的时候，客户端与 Flask 应用一起运行。
2. Celery workers: 这些是运行后台作业的进程。Celery 支持本地和远程的 workers，因此你就可以在 Flask 服务器上启动一个单独的 worker，随后随着你的应用需求的增加而新增更多的 workers。
3. 消息代理: 客户端通过消息队列和 workers 进行通信，Celery 支持多种方式来实现这些队列。最常用的代理就是 `RabbitMQ <http://www.rabbitmq.com/>`_ 和 `Redis <http://redis.io/>`_。


致缺少耐心的读者
--------------

如果你是行动派，本文开头的截图勾起你的好奇心的话，那么可以直接到 `Github repository <https://github.com/miguelgrinberg/flask-celery-example>`_ 获取本文用到的代码。 README 文件将会给你快速和直接的方式去运行示例应用。

接着可以回到本文来了解工作机制！


Flask 和 Celery 一起工作
-------------------------

Flask 与 Celery 整合是十分简单，不需要任何插件。一个 Flask 应用需要使用 Celery 的话只需要初始化 Celery 客户端像这样::

	from flask import Flask
	from celery import Celery

	app = Flask(__name__)
	app.config['CELERY_BROKER_URL'] = 'redis://localhost:6379/0'
	app.config['CELERY_RESULT_BACKEND'] = 'redis://localhost:6379/0'

	celery = Celery(app.name, broker=app.config['CELERY_BROKER_URL'])
	celery.conf.update(app.config)

正如你所见，Celery 通过创建一个 Celery 类对象来初始化，传入应用名称以及消息代理的连接 URL，这个 URL 我把它放在 app.config 中的 CELERY_BROKER_URL 的键值。URL 告诉 Celery 代理服务在哪里运行。如果你运行的不是 Redis，或者代理服务运行在一个不同的机器上，相应地你需要改变 URL。

Celery 其它任何配置可以直接用 celery.conf.update() 通过 Flask 的配置直接传递。CELERY_RESULT_BACKEND 选项只有在你必须要 Celery 任务的存储状态和运行结果的时候才是必须的。展示的第一个示例是不需要这个功能的，但是第一个是需要的，因此最好从一开始就配置好。

任何你需要作为后台任务的函数需要用 celery.task 装饰器装饰。例如::

	@celery.task
	def my_background_task(arg1, arg2):
	    # some long running task here
	    return result

接着 Flask 应用能够请求这个后台任务的执行，像这样::

	task = my_background_task.delay(10, 20)

delay() 方法是强大的 apply_async() 调用的快捷方式。这样相当于使用 apply_async()::

    task = my_background_task.apply_async(args=[10, 20])

当使用 apply_async()，你可以给 Celery 后台任务如何执行的更详细的说明。一个有用的选项就是要求任务在未来的某一时刻执行。例如，这个调用将安排任务运行在大约一分钟后::

    task = my_background_task.apply_async(args=[10, 20], countdown=60)

delay() 和 apply_async() 的返回值是一个表示任务的对象，这个对象可以用于获取任务状态。我将会在本文的后面展示如何获取任务状态等信息，但现在让我们保持简单些，不用担心任务的执行结果。

更多可用的选项请参阅 `Celery 文档 <http://docs.celeryproject.org/en/latest/index.html>`_ 。


简单例子：异步发送邮件
-------------------------

我要举的第一个示例是应用程序非常普通的需求：能够发送邮件但是不阻塞主应用。

在这个例子中我会用到 `Flask-Mail <https://pythonhosted.org/Flask-Mail/>`_ 扩展，我会假设你们熟悉这个扩展。

我用来说明的示例应用是一个只有一个输入文本字段的简单表单。要求用户在此字段中输入一个电子邮件地址，并在提交，服务器会发送一个测试电子邮件到这个邮件地址。表单中包含两个提交按钮，一个立即发送邮件，一个是一分钟后发送邮件。表单的截图在文章开始。

这里就是支持这个示例的 HTML 模板:

.. sourcecode:: html

	<html>
	  <head>
	    <title>Flask + Celery Examples</title>
	  </head>
	  <body>
	    <h1>Flask + Celery Examples</h1>
	    <h2>Example 1: Send Asynchronous Email</h2>
	    {% for message in get_flashed_messages() %}
	    <p style="color: red;">{{ message }}</p>
	    {% endfor %}
	    <form method="POST">
	      <p>Send test email to: <input type="text" name="email" value="{{ email }}"></p>
	      <input type="submit" name="submit" value="Send">
	      <input type="submit" name="submit" value="Send in 1 minute">
	    </form>
	  </body>
	</html>

这里没有什么让特别的东西。只是一个普通的 HTML 表单，再加上 Flask 闪现消息。

Flask-Mail 扩展需要一些配置，尤其是电子邮件服务器发送邮件的时候会用到一些细节。为了简单我使用我的 Gmail 账号作为邮件服务器::

	# Flask-Mail configuration
	app.config['MAIL_SERVER'] = 'smtp.googlemail.com'
	app.config['MAIL_PORT'] = 587
	app.config['MAIL_USE_TLS'] = True
	app.config['MAIL_USERNAME'] = os.environ.get('MAIL_USERNAME')
	app.config['MAIL_PASSWORD'] = os.environ.get('MAIL_PASSWORD')
	app.config['MAIL_DEFAULT_SENDER'] = 'flask@example.com'

注意为了避免我的账号丢失的风险，我将其设置在系统的环境变量，这是我从应用中导入的。

有一个单一的路由来支持这个示例::

	@app.route('/', methods=['GET', 'POST'])
	def index():
	    if request.method == 'GET':
	        return render_template('index.html', email=session.get('email', ''))
	    email = request.form['email']
	    session['email'] = email

	    # send the email
	    msg = Message('Hello from Flask',
	                  recipients=[request.form['email']])
	    msg.body = 'This is a test email sent from a background Celery task.'
	    if request.form['submit'] == 'Send':
	        # send right away
	        send_async_email.delay(msg)
	        flash('Sending email to {0}'.format(email))
	    else:
	        # send in one minute
	        send_async_email.apply_async(args=[msg], countdown=60)
	        flash('An email will be sent to {0} in one minute'.format(email))

	    return redirect(url_for('index'))

再次说明，这是一个很标准的 Flask 应用。由于这是一个非常简单的表单，我决定在没有扩展的帮助下处理它，因此我用 request.method 和 request.form 来完成所有的管理。我保存用户在文本字段中输入的值在 session 中，这样在页面重新加载后就能记住它。

在这个函数中让人有兴趣的是发送邮件的时候是通过调用一个叫做 send_async_email 的 Celery 任务，该任务调用 delay() 或者 apply_async() 方法。

这个应用的最后一部分就是能够完成作业的异步任务::

	@celery.task
	def send_async_email(msg):
	    """Background task to send an email with Flask-Mail."""
	    with app.app_context():
	        mail.send(msg)

这个任务使用 celery.task 装饰使得成为一个后台作业。这个函数唯一值得注意的就是 Flask-Mail 需要在应用的上下文中运行，因此需要在调用 send() 之前创建一个应用上下文。

重点注意在这个示例中从异步调用返回值并不保留，因此应用不能知道调用成功或者失败。当你运行这个示例的时候，需要检查 Celery worker 的输出来排查发送邮件的问题。


复杂例子：显示状态更新和结果
-------------------------



状态更新的后台任务
^^^^^^^^^^^^^^^^^


从 Flask 应用中访问任务状态
^^^^^^^^^^^^^^^^^^^^^^^^^^^


客户端的  Javascript 
^^^^^^^^^^^^^^^^^^^^^^^^^^^



运行示例
----------