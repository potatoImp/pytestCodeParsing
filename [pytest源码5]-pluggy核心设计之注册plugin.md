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


# `PluginManager.register()`是怎么实现的？
**Demo中的`pm.register(HookImpl1())`是怎么实现的？**

</br>

### pm.register`的作用是注册一个pluggy的实现并将其与对应的hook关联起来，我们来看主要代码
```python
# register matching hook implementations of the plugin
self._plugin2hookcallers[plugin] = hookcallers = []
for name in dir(plugin):
    hookimpl_opts = self.parse_hookimpl_opts(plugin, name)    #获取pluggy的属性或方法中的特殊attribute project_name + _impl
    if hookimpl_opts is not None:
        normalize_hookimpl_opts(hookimpl_opts)
        method = getattr(plugin, name)    #特殊attribute存在时获取到plugin的对应方法
        hookimpl = HookImpl(plugin, plugin_name, method, hookimpl_opts)
        hook = getattr(self.hook, name, None)
        if hook is None:
            hook = _HookCaller(name, self._hookexec)    #为hook添加一个_HookCaller对象
            setattr(self.hook, name, hook)
        elif hook.has_spec():
            self._verify_hook(hook, hookimpl)
            hook._maybe_apply_history(hookimpl)
        hook._add_hookimpl(hookimpl)    #将hookimpl添加到hook中
        hookcallers.append(hook)    #将遍历找到的每一个plugin hook添加到hookcallers,以待调用
```
  - **遍历pluggy对象的所有属性或方法（method），并获取该pluggy method的特殊`attribute` `project_name + _impl`**
  - **将带有project_name + _impl的method封装成一个HookImpl中**
  - **再把一个`_HookCaller`的对象添加到`hook`中，并为`self.hook`新增一个value为`hook`，name为`method`的属性（比如前面的demo的`calculate`）**
  - **最后将遍历找到的每一个`_HookCaller`添加到hookcallers,以待调用**
  