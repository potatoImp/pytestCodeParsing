# 前言
###### 最近可能遇到了人生的低谷了，事情都不太顺心。打算好好研究下pytest的源码。之前一直想做，但迟迟没有开始，这次下定决心完成它吧。
##### 个人拙见，有错请各位指出。
#### 源码这个东西怎么入手还是挺讲究的，我打算从pytest的核心框架Python Pluggy出发，首先介绍下Pluggy。
#### 解读过程主要按代码逻辑走，不会按照源码分布去解读，望理解。
###### _如果的我的文章对您有帮助，不符动动您的金手指给个Star，予人玫瑰，手有余香，不胜感激。_
<br/>

## Pluggy
#### [官方](https://github.com/pytest-dev/pluggy "点击访问官方网址")是这么介绍Pluggy的 
**"This is the core framework used by the `pytest`, `tox`, and `devpi` projects."**

**“这是`pytest`、`tox`和`devpi`项目使用的核心框架。”**
#### 很显然，这就是我们读pytest源码要从它开始的原因。  
<br/>

## 第一个Demo
#### Pluggy一开始是作为Pytest源码的一部分存在的，在后期被分离出来了，作为一个外部的依赖来使用。
### 我们由一个Demo来认识它，代码如下：

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
### Output
```bash
[6]
```
### 解析：
* #### `Pluggy`的核心为三个类`PluginManager`、`HookspecMarker`和`HookimplMarker`
    - ##### `PluginManager`用于注册hook与管理hook的实现
    - ##### `HookspecMarker`用于生成一个hook抽象方法装饰器
    - ##### `HookimplMarker`用于生成一个hook具体实现方法装饰器
* #### 整个项目中需要保证一个全局唯一的Project Name（具体原因放到后面再讲），此Demo为`myPluggyDemo_1`
* #### `Hookspec`是一个声明hook method的类，每一个hook method需要用`hookspec`装饰器装饰
* #### `HookImpl1`是一个plugin的实现，需要完整实现对应的hook方法，并通过`hookimpl`装饰器装饰实现方法
* #### 代码的核心逻辑是先创建一个插件管理对象PluginManager，并在该对象上注册hook对象HookSpec和与之对应的plugin对象HookImpl1，然后通过PluginManager自带的hook属性来调用对应的hook方法，传入相关参数。
  ##### 注意：调用hook方法时参数需以关键字的形式传递。
  
