# pluggy代码结构

#### 按照前面demo中的代码顺序，在分析pluggy的核心逻辑之前，我们先来了解`HookspecMarker`、`HookspecMarker`的用处是什么？

<br/>

### 1.`HookspecMarker`的实现逻辑是什么?
#### 我们来先来看它的代码注释

```python
class HookspecMarker(object):
      """ Decorator helper class for marking functions as hook specifications.

      You can instantiate it with a project_name to get a decorator.
      Calling PluginManager.add_hookspecs later will discover all marked functions
      if the PluginManager uses the same project_name.
      """

      def __init__(self, project_name):
          self.project_name = project_name
```
* **我们可以传入`project_name`实例化`HookspecMarker`以获得装饰器，当我们调用`PluginManager.add_hookspec`将会寻找所有与当前`PluginManager`同`project_name`的标记函数，这也是前面要求整个项目project name一致的原因之一。**
```python
def __call__(
        self, function=None, firstresult=False, historic=False, warn_on_impl=None
    ):
        """ if passed a function, directly sets attributes on the function
        which will make it discoverable to add_hookspecs().  If passed no
        function, returns a decorator which can be applied to a function
        later using the attributes supplied.

        If firstresult is True the 1:N hook call (N being the number of registered
        hook implementation functions) will stop at I<=N when the I'th function
        returns a non-None result.

        If historic is True calls to a hook will be memorized and replayed
        on later registered plugins.

        """

        def setattr_hookspec_opts(func):
            if historic and firstresult:
                raise ValueError("cannot have a historic firstresult hook")
            setattr(
                func,
                self.project_name + "_spec",
                dict(
                    firstresult=firstresult,
                    historic=historic,
                    warn_on_impl=warn_on_impl,
                ),
            )
            return func

        if function is not None:
            return setattr_hookspec_opts(function)
        else:
            return setattr_hookspec_opts
```
**通过分析`__call__`的逻辑代码可以发现，主要功能是调用了一个`setattr(object, name, value)`，给被装饰的函数新增一个属性`project_nam + _spec`，并且该属性的value为装饰器参数取值。**

<br/>

### 2.`HookspecMarker`的实现逻辑是什么?
#### `HookimplMarker`的实现逻辑类似，区别在于被装饰的函数新增的属性为`project_name + _impl`，下面只显示了部分代码
```python
        def setattr_hookimpl_opts(func):
            setattr(
                func,
                self.project_name + "_impl",
                dict(
                    hookwrapper=hookwrapper,
                    optionalhook=optionalhook,
                    tryfirst=tryfirst,
                    trylast=trylast,
                ),
            )
            return func
            
        if function is None:
            return setattr_hookimpl_opts
        else:
            return setattr_hookimpl_opts(function)
```

<br/><br/>

## pluggy核心设计
#### plugy的核心逻辑就是几行代码
```python
pm = PluginManager("myPluggyDemo")
pm.add_hookspecs(HookSpec)
pm.register(HookImpl1())
pm.hook.calculate(a=2, b=3)
```
* **创建一个`PluginManager`对象，用于管理plugin**
* **调用`add_hookspecs`, 增加一个新的hook module object（标准对象）**
* **调用`register`，注册一个新的plugin object**
* **通过`pm.hook`实现对与`calculate`同名的所有`plugin`的调用**
#### 按照上面的代码逻辑来走，我们来分析三行代码的实现，以帮助我们更好的理解
1. #### `pm.add_hookspecs(HookSpec)`是怎么实现的？
2. #### `pm.register(HookImpl1())`是怎么实现的？
3. #### `pm.hook.calculate(a=2, b=3)`是怎么实现的？
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
 
 <br/>
 
2. #### pm.register`的作用是注册一个pluggy的实现并将其与对应的hook关联起来，我们来看主要代码
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
#### pm.hook是什么？实现调用pluggy的逻辑是什么？
##### 这里就涉及到了上一步的`_HookCaller`了，`pm.hook.calculate`其实是相当于获取了对应`_HookCaller`，调用的是他的`__call__`方法，来看下代码
```python
    def __call__(self, *args, **kwargs):
        if args:      #只能传入键值对形式的参数
            raise TypeError("hook calling supports only keyword arguments")
        assert not self.is_historic()
        if self.spec and self.spec.argnames:
            notincall = (
                set(self.spec.argnames) - set(["__multicall__"]) - set(kwargs.keys())
            )
            if notincall:
                warnings.warn(
                    "Argument(s) {} which are declared in the hookspec "
                    "can not be found in this hook call".format(tuple(notincall)),
                    stacklevel=2,
                )
        return self._hookexec(self, self.get_hookimpls(), kwargs)
```
#### 核心代码在最后一行，我们再来看看self._hhokexec是什么，发现它是在构造_HookCaller时传入的一个参数，再找到它的定义
```python
    def _hookexec(self, hook, methods, kwargs):
        # called from all hookcaller instances.
        # enable_tracing will set its own wrapping function at self._inner_hookexec
        return self._inner_hookexec(hook, methods, kwargs)
```
#### 顺着走到最后，发现核心其实是hook.multicall

```python
self._inner_hookexec = lambda hook, methods, kwargs: hook.multicall(
            methods,
            kwargs,
            firstresult=hook.spec.opts.get("firstresult") if hook.spec else False,
        )
```
#### 这是一个PluggyManager构建时的封装函数`_multicall`，代码实现如下，详细逻辑留到后面再讲。
```python
def _multicall(hook_impls, caller_kwargs, firstresult=False):
    """Execute a call into multiple python functions/methods and return the
    result(s).

    ``caller_kwargs`` comes from _HookCaller.__call__().
    """
    __tracebackhide__ = True
    results = []
    excinfo = None
    try:  # run impl and wrapper setup functions in a loop
        teardowns = []
        try:
            for hook_impl in reversed(hook_impls):
                try:
                    args = [caller_kwargs[argname] for argname in hook_impl.argnames]
                except KeyError:
                    for argname in hook_impl.argnames:
                        if argname not in caller_kwargs:
                            raise HookCallError(
                                "hook call must provide argument %r" % (argname,)
                            )

                if hook_impl.hookwrapper:
                    try:
                        gen = hook_impl.function(*args)
                        next(gen)  # first yield
                        teardowns.append(gen)
                    except StopIteration:
                        _raise_wrapfail(gen, "did not yield")
                else:
                    res = hook_impl.function(*args)
                    if res is not None:
                        results.append(res)
                        if firstresult:  # halt further impl calls
                            break
        except BaseException:
            excinfo = sys.exc_info()
    finally:
        if firstresult:  # first result hooks return a single value
            outcome = _Result(results[0] if results else None, excinfo)
        else:
            outcome = _Result(results, excinfo)

        # run all wrapper post-yield blocks
        for gen in reversed(teardowns):
            try:
                gen.send(outcome)
                _raise_wrapfail(gen, "has second yield")
            except StopIteration:
                pass

        return outcome.get_result()
```

