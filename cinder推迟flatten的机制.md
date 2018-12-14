## cinder推迟flatten的机制
openstack提供的解决方案:添加了
```
from eventlet import tpool self.volume = tpool.Proxy(self.volume)
```
把创建volume的任务丢到了线程池，主进程继续向下走。
Time-consuming operations like flattening and copying volumes blocks eventlet loop and all cinder-volume service hangs until it finished. It makes cinder-volume services unavailable for a while.

This patch moves all RBDVolume calls to a separate python thread which doesn't block eventlet loop.

这里有个问题，为什么用tpool呢。

## eventlet中的tpool与greenpool
greenthread 池提供了一定数量的备用 greenthread ，有效限制了孵化 greenthread 过多导致的内存不足，当池子中没有足够的空闲 greenthread 时，孵化过程被暂停，只有当先前工作中的 greenthread 完成当前工作，才能为下一个任务做孵化准备。

thread进程的多进程都是同一个id，而tpool的则是不同的进程。
```
In [1]: from eventlet import tpool
In [2]:  import thread
In [3]: def my_func(thread_ident):
    ...:     print ("now raw thrad id:",  thread_ident)
    ...:
In [4]: tpool.execute(my_func, thread.get_ident())
('now raw thrad id:', 140736349889472)
In [5]: def my_func(thread_ident):
    ...:     print ("now raw thrad id:",  thread_ident, thread.get_ident())
In [6]: tpool.execute(my_func, thread.get_ident())
('now raw thrad id:', 140736349889472, 123145477103616)
In [7]: tpool.execute(my_func, thread.get_ident())
('now raw thrad id:', 140736349889472, 123145481310208)
In [8]: tpool.execute(my_func, thread.get_ident())
('now raw thrad id:', 140736349889472, 123145485516800)
```

## tpool
当直接调用myfunc（）时，它不仅阻止了此协程，而且还阻止了其他协程运行，同时调用tpool.execute（）将阻止当前协程，但不知何故会产生其他协程可以运行。是这样吗？否则，我不明白tpool如何有用。

与eventlet.spawn(tpool.execute, fun, ...)相结合，它甚至不会阻塞调用者协程。也许你会发现这是一个有用的组合。
