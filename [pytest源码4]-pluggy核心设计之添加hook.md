# 前言
###### 简单了解了pluggy之后，我们还需要再了解些知识，为解读代码逻辑做准备
##### 个人拙见，有错请各位指出。
###### _如果的我的文章对您有帮助，不符动动您的金手指给个Star，予人玫瑰，手有余香，不胜感激。_
</br>
</br>
</br>

# pluggy核心逻辑
```python
pm = PluginManager("myPluggyDemo")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.hook.calculate(a=2, b=3)
```

<br/>
<br/>

## `PluginManager.add_hookspecs()`是怎么实现的？
### `pm.add_hookspecs(HookSpec)`是怎么实现的？

</br>

```python
def add_hookspecs(self, module_or_class):
    """ add new hook specifications defined in the given module_or_class.
    Functions are recognized if they have been decorated accordingly. """
    names = []
    for name in dir(module_or_class):          #1.遍历传入对象的所有属性方法列表
        spec_opts = self.parse_hookspec_opts(module_or_class, name)        #2.拿到我们前面在HookspecMarker为函数新增的那个属性project_name + _spec
```

  - **遍历传入对象的所有属性方法列表**
  - **拿到每个属性方法，若有特殊属性`project_name + _spec`，则返回它，否则返回`None`，下面是该方法的代码展示**
 ```python
def parse_hookspec_opts(self, module_or_class, name):        #2.1拿到该属性的方法实现
    method = getattr(module_or_class, name)        #此处获取到我们之前定义的hook方法
    return getattr(method, self.project_name + "_spec", None)        #此处获取到为该方法新增的属性project_name + _spec
 ```