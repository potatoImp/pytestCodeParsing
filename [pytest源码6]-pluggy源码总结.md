# 前言
###### 简单了解了pluggy之后，我们还需要再了解些知识，为解读代码逻辑做准备
##### 个人拙见，有错请各位指出。
###### _如果的我的文章对您有帮助，不符动动您的金手指给个Star，予人玫瑰，手有余香，不胜感激。_
</br>
</br>
</br>


# 总结
#### 对前面分析的源码进行一次总结，这里就不会详细的去讲了。
#### 阅读了一遍源码之后我们再对着pluggy的demo进行总结

<br/><br/>

### pluggy Demo

```python
# -*- coding:utf-8 -*-

from pluggy import PluginManager, HookspecMarker, HookimplMarker

hookspec = HookspecMarker("myPluggyDemo_1") #一个声明hook method的类，每个hook method都需要用@hookspec来装饰
hookimpl = HookimplMarker("myPluggyDemo_1") #一个plugin的实现，需要完整实现对应的hook方法，并通过@hookimpl来装饰


class HookSpec:
    @hookspec
    def calculate(self, a, b):
        pass


class HookImpl1:
    @hookimpl
    def calculate(self, a, b):
        return a + b


pm = PluginManager("myPluggyDemo_1") #创建PluginManager对象
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
results = pm.hook.calculate(a=1, b=5)
print(results)
```

<br/><br/>

1.首先保证项目唯一的project name是因为我们添加hook与注册pluggy时都需要利用封装类，装饰器将hook方法、plugin实现的引用以project name +  spec+ method 为属性名增加到封装类中。保证唯一才能正确的找到对应的hook、plugin。

2.实例化HookspecMarker, HookimplMarker时会返回一个装饰器，我们可以通过这个方式自定义（重新定义）装饰器引用名称

3.在通过装饰器声明hook方法、plugin方法时，传入不同的参数可以达到如：改变执行顺序、定义wrapper plugin（yield）等效果

4.通过`add_hookspecs`将传入的`hook instance`中的特殊方法名`hook name`与对应的hook调用的封装实例`_HookCaller`以`name:hc`的形式作为属性添加加到`PluginManager`实例中的`self.hook`属性上，如果该特殊方法`hook method`存在的话。

- _倘若特殊方法`hook method`不存在的话，将`module_or_class`与该“`hook name`”通过`HookSpec()`执行一遍2.的逻辑，并遍历当前`PluginManager`实例中的`_nonwrappers`与`_wrappers`检查hook是否正确添加。_


5.通过`register`将传入的`pluggy instance`中特殊方法`pluggy method`进行数据封装得到`HookImpl instance`，并获取到当前`PluginManager`实例中的`self.hook`的同名属性，关键点就在这，此前我们在第4步添加hook时，所添加的同名hook方法就是在这个时候以这个同名属性关联起来了。然后将`plugin`的实现方法的调用方法以一定顺序（通过装饰器传入的的参数）添加到同名属性中，并保存到列表hookcallers中，即最后存储在`PluginManager`实例的`self._plugin2hookcallers[plugin]`

- 倘若根本没有对应的同名hook呢？此时就会通过执行与第4步中相同的操作，封装`_HookCaller`以`name:hook`的形式作为属性添加到`PluginManager`实例中的`self.hook`属性上。然后再将`plugin`的调用方法添加到同名属性中，并保存到`PluginManager`实例的`self._plugin2hookcallers[plugin]`。
- 前置：会预先检查当前pluggy是否已经注册过了，抛出ValueError

6.每个pluggy实现都会被封装成`_HookCaller instance`保存在`PluginManager`实例属性pm.hook中，每次对同名pluggy方法进行调用时，大致的调用逻辑是这样的：获得存储着对应的plugin实现的列表，将其反转（所以后register的plugin先执行），并检查plugin方法的参数是否有遵守对应hook方法的约定。分成hookwrapper和nonwrapper两种形式去执行`plugin`的实现代码，执行结果用_Result类进行封装，最后检查plugin call过程有无异常，没有异常就返回执行结果。

