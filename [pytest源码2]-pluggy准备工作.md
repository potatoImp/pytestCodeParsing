# 前言
###### 简单了解了pluggy之后，我们还需要再了解些知识，为解读代码逻辑做准备
##### 个人拙见，有错请各位指出。
###### _如果的我的文章对您有帮助，不符动动您的金手指给个Star，予人玫瑰，手有余香，不胜感激。_
</br>
</br>
</br>

# hook和plugin的关系
#### hook和plugin是`1:N`的对应关系，假设同时注册了多个实现了同一hook的plugin，则会对应的返回多个结果。
### Demo如下
```
# -*- coding:utf-8 -*-

from pluggy import PluginManager, HookspecMarker, HookimplMarker

hookspec = HookspecMarker("myPluggyDemo_2")
hookimpl = HookimplMarker("myPluggyDemo_2")


class HookSpec:
    @hookspec
    def calculate(self, a, b):
        pass


class HookImpl1:
    @hookimpl
    def calculate(self, a, b):
        return a + b


class HookImpl2:
    @hookimpl
    def calculate(self, a, b):
        return a * b
    

pm = PluginManager("myPluggyDemo_2")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.register(HookImpl2())
print(pm.hook.calculate(a=2, b=3))
```
### Output
```
[6, 5]
```
### 解析：
* #### 在Demo2中，我们注册了两个`plugin`，`HookImpl1`和`HookImpl2`，分别实现了加法和乘法两个逻辑。
* #### 每次调用hook都会返回两个`plugin`执行的结果，先执行后注册的`HookImpl2`，再执行先注册的`HookImpl1`,即越晚注册的plugin越先执行。(后续会讲原因)

<br/>
<br/>
</br>

# Plugin的调用顺序与参数
***
<br/>
</br>

## HookspecMarker装饰器参数
#### HookspckMarker装饰器支持传入一些特定的参数，常用的有
* ##### firstresult - 如果firstresult值为True时，获取第一个plugin执行结果后就停止（中断）继续执行。
* ##### historic - 如果值为True时，表示这个hook是需要保存调用记录（call history）的，并将该调用记录回放在未来新注册的plugins上。
#### 当装饰器传入了firstresult=True时，plugin的执行会在后注册的HookImpl2执行完毕后停止，不再往下执行。
### Demo如下
```
# -*- coding:utf-8 -*-

from pluggy import PluginManager, HookspecMarker, HookimplMarker

hookspec = HookspecMarker("myPluggyDemo_3")
hookimpl = HookimplMarker("myPluggyDemo_3")


class HookSpec:
    @hookspec(firstresult=True)
    def calculate(self, a, b): 
        pass


class HookImpl1:
    @hookimpl
    def calculate(self, a, b):
        return a + b            #本例子中不会执行加法逻辑


class HookImpl2:
    @hookimpl
    def calculate(self, a, b):
        return a * b


pm = PluginManager("myPluggyDemo_3")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.register(HookImpl2())
print(pm.hook.calculate(a=2, b=3))
```
### Output
```
6
```

<br/>
</br>

## HookimplMarker装饰器参数
#### HookimplMarker装饰器支持传入一些特定的参数，常用的有
* #### tryfirst - 如果tryfirst值为True，则此plugin会尽可能早的在1:N的实现链路执行
* #### trylast - 如果trylast值为True，则此plugin会相应地尽可能晚的在1:N的实现链中执行
* #### hookwrapper - 如果该参数为True，需要在plugin内实现一个yield，plugin执行时先执行wrapper plugin前面部分的逻辑，然后转去执行其他plugin，最后再回来执行wrapper plugin后面部分的逻辑。
* #### optionalhook - 如果该参数为True，在此plugin缺少相匹配的hook时，不会报error（spec is found）。

</br>
</br>

## tryfirst的Demo
##### 我们修改一下demo3，把HookImpl1加上tryfirst=True参数，即可达到先执行先注册的HookImpl1。
```
# -*- coding:utf-8 -*-

from pluggy import PluginManager, HookspecMarker, HookimplMarker

hookspec = HookspecMarker("myPluggyDemo_4")
hookimpl = HookimplMarker("myPluggyDemo_4")


class HookSpec:
    @hookspec()
    def calculate(self, a, b):
        pass


class HookImpl1:
    @hookimpl(tryfirst=True)
    def calculate(self, a, b):
        return a + b


class HookImpl2:
    @hookimpl
    def calculate(self, a, b):
        return a * b


pm = PluginManager("myPluggyDemo_4")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.register(HookImpl2())
print(pm.hook.calculate(a=2, b=3))
```
### Output

``` 
[5, 6]
```
#### trylast以此类推，携带者变为后执行

</br>

## hookwrapper
#### 我们实现一个特殊的plugin`WrapperPlugin`
```
# -*- coding:utf-8 -*-

from pluggy import PluginManager, HookspecMarker, HookimplMarker

hookspec = HookspecMarker("myPluggyDemo_5")
hookimpl = HookimplMarker("myPluggyDemo_5")


class HookSpec:
    @hookspec
    def calculate(self, a, b):
        pass


class HookImpl1:
    @hookimpl
    def calculate(self, a, b):
        print('HookImpl1 execute!')
        return a + b


class HookImpl2:
    @hookimpl(tryfirst=True)
    def calculate(self, a, b):
        print('HookImpl2 execute!')
        return a * b


class WrapperPluggy:
    @hookimpl(hookwrapper=True)
    def calculate(self, a, b):
        print('WrapperPluggy execute!')
        print("Before yield")
        result = yield            #此处返回的值为其他两个pluggy执行的结果
        print(f"After yield,result is {result.get_result()}")
        return a * b + (a + b)


pm = PluginManager("myPluggyDemo_5")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.register(HookImpl2())
pm.register(WrapperPluggy())
print(pm.hook.calculate(a=2, b=3))
```
### Output
```
WrapperPluggy execute!
Before yield
HookImpl2 execute!
HookImpl1 execute!
After yield,result is [6, 5]
[6, 5]
```
### 解析：
* ##### `ImplWrapper`中的`pluggy`的代码逻辑，以`result = yield` 为分割线，分成两个部分。第一部分执行完毕后，中断继续执行，转去执行其他`plugin`，待其他`plugin`都执行完时，回来继续执行剩下的部分。
* ##### `result = yield`result通过yield来获取到其他plugin执行的结果，即非`wrapper plugin`的执行结果（`HookImpl2`和`HookImpl1`）
* ##### 从Output中可以看出，我们`WrapperPluggy`的返回结果没有被打印出来，这是因为`wrapper plugin`的返回值会被`Ignore`，原因后续会提到。