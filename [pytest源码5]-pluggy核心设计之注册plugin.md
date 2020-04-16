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


# `pm.register(HookImpl1())`是怎么实现的？

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