##pytest-plugin深挖hook调用逻辑
###前面介绍了不少hook的调用逻辑，但是还有个hook_execute没接上，这里来完整的分析pm.hook.calculate(a=2, b=3)的执行过程
###每当我们调用pm.hook.xxx(**kwargs)的时候，实际上是调用了_HookCaller对象的__call__方法
* ```
    def __call__(self, *args, **kwargs):
        if args:
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
  * ####从`__call__`的代码可以看到核心逻辑是最后一行的`self._hookexec`，我们可以发现这是_HookCaller的一个属性
####我们顺着`self._hookexec`往回找到_HookCaller的构造函数
* ```
    def __init__(self, name, hook_execute, specmodule_or_class=None, spec_opts=None):
        self.name = name
        self._wrappers = []
        self._nonwrappers = []
        self._hookexec = hook_execute
  ```
  * ####可以发现，这个属性是构造`_HookCaller`对象时传入的的方法，我们再往回找，看看`hook_execute`是从哪里传进来的
####我们发现`hook_execute`是从PluginManager类的register方法中实例化_HookCaller时传递的
* ```
    if hook is None:
        hook = _HookCaller(name, self._hookexec)
  ```
####这是PluginManager类的一个方法，找到该方法，发现只是一个封装，继续往上找
* ```
    def _hookexec(self, hook, methods, kwargs):
        # called from all hookcaller instances.
        # enable_tracing will set its own wrapping function at self._inner_hookexec
        return self._inner_hookexec(hook, methods, kwargs)
  ```
####最后`hook_execute`其实是hook.multicall方法，也就是multicall函数的封装
* ```
  self._inner_hookexec = lambda hook, methods, kwargs: hook.multicall(
      methods,
      kwargs,
      firstresult=hook.spec.opts.get("firstresult") if hook.spec else False,
      )
  ```
####往回找，没想到回到了`_HookCaller`类的`self.multicall`
* ```
  class _HookCaller(object):
      def __init__(self, name, hook_execute, specmodule_or_class=None, spec_opts=None):
          self.name = name
          self._wrappers = []
          self._nonwrappers = []
          self._hookexec = hook_execute
          self.argnames = None
          self.kwargnames = None
          self.multicall = _multicall
  ```
####最后找到了`_HookCaller`类的`_multicall`方法，分段看一下
* ```
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
  ```
  * ####反转hook_impls,plugin执行从list末尾开始，这也是为什么后注册的plugin先执行的原因
  * ####检查参数，如果hook_impls使用的参数没有在hookspec中预先定义的话，抛出HookCallError
* ```
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
  ```
  * ####hookwrapper
    * #### 如果定义plugin时hookwrapper参数为True时,会先执行plugin中yield之前的代码，等其他plugin执行完才继续执行yield后面的的部分。
    * #### `gen = hook_impl.function(*args)`执行plugin function中yield前的部分，然后停下
    * #### `next(gen)`迭代到plugin function中yield后面的部分
    * #### 将gen得到的generator加到teardown中，用于后续的callback
  * ####nonwrapper
    * #### 直接调用plugin function
    * #### 将执行结果保存到result中
* ```
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
  * ####在一个hook的所有plugin实现都执行完后，所有的执行结果都要用_Result类封装
  * ####如果仍存在前面存入teardowns中的generator，遍历并执行这些“执行了一半的”hookwrapper
  * ####并将nowrapper的结果传递给hookwrapper的后半部分
  * ####这里不允许在一个hookwrapper使用两次yield，会导致在外部抛出异常终止逻辑 `_raise_wrapfail(gen, "has second yield")`->`RuntimeError`
    * ####这里详细讲下
    * ####当gen.send(outcome)将结果传递回hookwrapper，hookwrapper会接着往下执行，没有yield的时候会按执行完plugin的逻辑走
    * ####当再次遇到yield的时候，会再次跳出，跳回的位置就是这里，所以再往下执行`_raise_wrapfail(gen, "has second yield")`会抛出错误了。
  * ####最后将outcome的结果返回给调用方`hook_execute`->`self._hookexec`->`pm.hook.xxx(**kwargs)`
  
###额外再看看_Result的代码，先看构造函数，就是把执行结果与执行异常信息封装到类_Result
* ```
  class _Result(object):
      def __init__(self, result, excinfo):
          self._result = result
          self._excinfo = excinfo
  ```
###还有主要方法get_result()
* ```
  def get_result(self):
      """Get the result(s) for this hook call.

      If the hook was marked as a ``firstresult`` only a single value
      will be returned otherwise a list of results.
        """
      __tracebackhide__ = True
      if self._excinfo is None:
          return self._result
      else:
          ex = self._excinfo
          if _py3:
              raise ex[1].with_traceback(ex[2])
          _reraise(*ex)  # noqa
  ```
  * ####判断`hook call`过程是有否异常。
  * ####无异常情况下返回`hook call`的执行结果。
  * ####有异常情况下，抛出基础异常类`BaseException`的回溯结果
  
##总结
####断断续续坚持扒了一个多星期pluggy源码终于扒完了，以前听说看完优秀源码会觉得自己写的东西是垃圾，现在我相信了
