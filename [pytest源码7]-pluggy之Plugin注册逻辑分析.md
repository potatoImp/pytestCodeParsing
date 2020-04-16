# 前言
###### 简单了解了pluggy之后，我们还需要再了解些知识，为解读代码逻辑做准备
##### 个人拙见，有错请各位指出。
###### _如果的我的文章对您有帮助，不符动动您的金手指给个Star，予人玫瑰，手有余香，不胜感激。_
</br>
</br>
</br>

# pluggy注册逻辑分析性

#### 我们来详细分析一下plugin的注册逻辑register方法
```python
  plugin_name = name or self.get_canonical_name(plugin)    #获取插件名

  if plugin_name in self._name2plugin or plugin in self._plugin2hookcallers:
      if self._name2plugin.get(plugin_name, -1) is None:
          return  # blocked plugin, return None to indicate no registration
      raise ValueError(
          "Plugin already registered: %s=%s\n%s"
          % (plugin_name, plugin, self._name2plugin)
      )
```
  * **根据传入plugin name或由plugin对象获取到插件名，将其赋值给plugin_name**
  * **self._name2plugin是以plugin_name为key的dict**
  * **self._pluginhookcallers是以plugin object为key的dict**
  * **通过上述两个dict来判断传入的plugin是否已经注册过了**
```python
  self._name2plugin[plugin_name] = plugin
```
  * **将这个pluggy以`plugin_name:plugin object`的形式保存到`self._name2plugin`中**
```python
  self._plugin2hookcallers[plugin] = hookcallers = []
```
  * **创建一个`list`对象`hookcallers`用来保存每个pluggy的实际调用对象`_HookCaller`，以`plugin object:hookcallers object`的形式保存在`self._plugin2hookcallers`**
```python
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
#### 我们来分析下上面代码提到的两个对象HookImpl与_HookCaller
#### 首先看下HookImpl的实现，其实就是一个数据封装类
```python
  class HookImpl(object):
      def __init__(self, plugin, plugin_name, function, hook_impl_opts):
          self.function = function
          self.argnames, self.kwargnames = varnames(self.function)
          self.plugin = plugin
          self.opts = hook_impl_opts
          self.plugin_name = plugin_name
          self.__dict__.update(hook_impl_opts)

      def __repr__(self):
          return "<HookImpl plugin_name=%r, plugin=%r>" % (self.plugin_name, self.plugin)
```
#### 最后看看核心_HookCaller的实现，它是整个plugin的核心类
```python
  class _HookCaller(object):
      def __init__(self, name, hook_execute, specmodule_or_class=None, spec_opts=None):
          self.name = name
          self._wrappers = []
          self._nonwrappers = []
          self._hookexec = hook_execute
          self.argnames = None
          self.kwargnames = None
          self.multicall = _multicall
          self.spec = None
          if specmodule_or_class is not None:
              assert spec_opts is not None
              self.set_specification(specmodule_or_class, spec_opts)
 ```
  * ##### 在register逻辑中，我们传入的参数是name与self._hookexec(hook_execute)，其中hook_execute表示的是一个函数对象，负责实际plugin的调用
```python
    def has_spec(self):
        return self.spec is not None

    def set_specification(self, specmodule_or_class, spec_opts):
        assert not self.has_spec()
        self.spec = HookSpec(specmodule_or_class, self.name, spec_opts)
        if spec_opts.get("historic"):
            self._call_history = []

    def is_historic(self):
        return hasattr(self, "_call_history")
```
  * **增加调用历史记录，返回调用历史记录**
```python
    def _add_hookimpl(self, hookimpl):
        """Add an implementation to the callback chain.
        """
        if hookimpl.hookwrapper:    #根据hookimpl.hookwrapper将plugin分为两类
            methods = self._wrappers
        else:
            methods = self._nonwrappers

        if hookimpl.trylast:    #根据参数为plugin调整对应的执行顺序
            methods.insert(0, hookimpl)
        elif hookimpl.tryfirst:
            methods.append(hookimpl)
        else:
            # find last non-tryfirst method
            i = len(methods) - 1
            while i >= 0 and methods[i].tryfirst:
                i -= 1
            methods.insert(i + 1, hookimpl)

        if "__multicall__" in hookimpl.argnames:    #版本更新做的处理，不再支持__multicall__的参数方式
            warnings.warn(
                "Support for __multicall__ is now deprecated and will be"
                "removed in an upcoming release.",
                DeprecationWarning,
            )
            self.multicall = _legacymulticall 
```
  * **在构造函数中我们可以看到`self._wrappers`和`self._nonwrappers`，通过`_add_hookimpl`我们将`plugin`分成两类**
  * **按照装饰器传入的参数对plugin的执行顺序进行排序**
```python
    def _remove_plugin(self, plugin):
        def remove(wrappers):
            for i, method in enumerate(wrappers):
                if method.plugin == plugin:
                    del wrappers[i]
                    return True

        if remove(self._wrappers) is None:
            if remove(self._nonwrappers) is None:
                raise ValueError("plugin %r not found" % (plugin,))
```
  * **remove plugin时需将`_wrappers`和`_nonwrappers`两类中的`plugin`都`remove`**
  * **通过遍历大类中的`method`的`plugin`属性来找到要删除的`plugin`，通过`del`来删除引用变量**
```python
    def call_extra(self, methods, kwargs):
        """ Call the hook with some additional temporarily participating
        methods using the specified kwargs as call parameters. """
        old = list(self._nonwrappers), list(self._wrappers)    #获取到所有原始插件list
        for method in methods:    #遍历plugin method
            opts = dict(hookwrapper=False, trylast=False, tryfirst=False)    #统一临时plugin参数
            hookimpl = HookImpl(None, "<temp>", method, opts)    #创建临时plugin实例
            self._add_hookimpl(hookimpl)    #默认排序plugin执行顺序
        try:
            return self(**kwargs)    #返回插件增加了临时plugin的插件引用
        finally:
            self._nonwrappers, self._wrappers = old    #执行完毕，恢复原始插件list
```
  * **有时我们会需要在某一次执行增加一些临时的`plugin`，是`Plugin`为我们提供一个方法`call_extra`**
  * **首先获取原本的`plugin`列表，在方法执行的最后需要恢复原来的plugin列表**
  * **对我们传入的临时`plugin method`都统一创建默认执行顺序的名为"<temp>"的临时`plugin object`**
  * **并将增加了临时`plugin`的`_HookCaller object`返回，以待调用**

```python
    def _maybe_apply_history(self, method):
        """Apply call history to a new hookimpl if it is marked as historic.
        """
        if self.is_historic():
            for kwargs, result_callback in self._call_history:
                res = self._hookexec(self, [method], kwargs)
                if res and result_callback is not None:
                    result_callback(res[0])
```
  * **判断hook是否有指定参数**
  * **遍历调用历史，得到插件执行的结果**
