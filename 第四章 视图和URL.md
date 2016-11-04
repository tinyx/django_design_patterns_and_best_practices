#第四章 视图和URL   
    
这一章，我们将会讨论以下话题：

 - 基于类和基于函数的视图
 - Mixin
 - 修饰符
 - 通用的视图模式
 - 设计URL

##视图概览

在Django中，视图被定义成一个可以调用的对象，它接受一个请求对象并返回响应对象。它通常是一个函数，或者是一个带有特殊类方法(比如`as_view()`)的类。

在两种情况里，我们都会创建一个常规的Python函数，将`HTTPRequest`作为它的第一个参数，并返回`HTTPResponse`。`URLConf`也可以将附加的参数传给这个函数。这些参数的值可以从URL中获取，也可以事先给定默认值。

一个简单的视图大致长这样：

	# In views.py
	from django.http import HttpResponse

	def hello_fn(request, name="World"):
		return HttpResponse("Hello {}!".format(name))

这个只有两行的视图函数非常好理解。我们并没有根据`request`作出任何行为。我们可以检视一个request对象来更好的了解这个视图被调用的时候的上下文情况，比如查看`GET`/`POST`参数，URI路径，或者HTTP头字段，比如`REMOTE_ADDR`。

与之对应的`URLConf`如下：

	# In urls.py
		url(r'^hello-fn/(?P<name>\w+/$', views.hello_fn),
		url(r'^hello-fn/$', views.hello_fn),

我们使用相同的视图来响应两种URL模式。第一种模式将name作为参数。第二种模式不接受URL里的任何参数，视图会使用默认值`World`。

##基于类的视图

基于类的视图是在Django 1.4被引入的。以下是将之前的视图改写成功能一样的基于类的视图后的样子：

	from django.views.generic import View
	class HelloView(View):
		def get(self, request, name="World"):
			return HttpResponse("Hello {}!".format(name))

同样的，对应的`URLConf`也包含两行，如下例所示：

	# In urls.py
		url(r'^hello-cl/(?P<name>\w+)/$', views.HelloView.as_view()),
		url(r'^hello-cl/$', views.HelloViews.as_view()),

视图类和我们之前的视图函数有一些很有趣的不同。最明显的是我们现在需要定义一个类了。其次，我们显式地定义了我们只会处理`GET`请求。之前的视图函数会对`GET`，`POST`以及任何其它的HTTP行为做出相同的响应。我们可以在Django命令行里用测试客户端完成以下测试：

	>>> from django.test import Client
	>>> c = Client()

	>>> c.get("http://0.0.0.0:8000/hello-fn/").content
	b'Hello World!'

	>>> c.post("http://0.0.0.0:8000/hello-fn/").content
	b'Hello World!'

	>>> c.get("http://0.0.0.0:8000/hello-cl/").content
	b'Hello World!'

	>>> c.post("http://0.0.0.0:8000/hello-cl/").content
	b''

显式地定义行为从安全和维护的角度来看都更好一些。

当你需要自定义你的视图时，使用类的优点就会凸显出来了。假设你需要修改问候语的内容以及默认的称呼，你可以先定义一个可以发出任意问候的通用视图类，然后在你自己的类中修改问候的内容：

	class GreetView(View):
		greeting = "Hello {}!"
		default_name = "World"
		def get(self, request, **kwargs):
			name = kwargs.pop("name", self.default_name)
			return HttpResponse(self.greeting.format(name))

	class SuperVillainView(GreetView):
		greeting = "We are the future, {}. Not them. "
		default_name = "my friend"

现在，只需要修改`URLConf`将目标指向新的类：

	# In urls.py
		url(r'^hello-su/(?P<name>\w+)/$', views.SuperVillainView.as_view()),
		url(r'^hello-su/$', views.SuperVillainView.as_view()),

当然，用类似的方式来自定义视图函数并不是不可能，但你需要增加一些关键词参数并指定默认值。这很快就会变得难以维护。这正是当初通用视图从视图函数转型成基于类的视图的原因。

>被解放的姜戈
>