# 核心要点
**务必理解下面三句官方描述**
> 1. **await** uses the yield from implementation with an extra step of validating its argument. [https://peps.python.org/pep-0492/](https://peps.python.org/pep-0492/)

> 2. The statement  RESULT = yield from EXPR is semantically equivalent to [https://peps.python.org/pep-0380/](https://peps.python.org/pep-0380/)

```python

_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:
                try:
                    _y = _m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:
            try:
                if _s is None:
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:
                _r = _e.value
                break
RESULT = _r

```
> 3. Any yield from chain of calls ends with a yield. every await is suspended by a yield somewhere down the chain of await calls. [https://peps.python.org/pep-0492/](https://peps.python.org/pep-0492/)


我的理解

1. await按照上面理解来展开执行，await chain会逐层展开，执行时会持续next/send把参数往下传递，直到遇到一个真实的yield表达式。然后挂起yield所在函数，把执行权和结果交给上层函数，逐层向上传递，直到最顶层，也就是Task.__step中的coro.send处。
2. 当再次调用coro.send时，还是从await chain(也就是上面展开中的隐藏yield)开始向下传递数输入的参数给到真实的yield表达式，然后执行yield后的下半部分逻辑
3. await本身不会挂起进程，只有遇到真实的yield才会挂起，没有真实yield就不会有隐藏yield执行。看上面的展开，遇到一个真实的yield，才会执行一次隐藏的yield，隐藏的yield总是和真实的yield依次执行，隐藏的yield的确会挂起进程，但仅仅起到代理真实yield的作用
4. async def定义的是native coroutine, 其中只能使用await。native coroutine是一个Awaitable对象
5. types.coroutune装饰的是gen_based_coroutine, 其中只能使用yield/yield from。types.coroutune返回的也是一个awaitable对象，把gen封装在一个Awaitble对象内部，用Awaitble对象来代替这个gen，所以可以用在await语句中。

详情参考：

- [https://peps.python.org/pep-0492/](https://peps.python.org/pep-0492/)
- [https://peps.python.org/pep-0380/](https://peps.python.org/pep-0380/)
- [https://peps.python.org/pep-0342/](https://peps.python.org/pep-0342/)
# 实验
可以用pdb调试看变化

- python -m pdb test.py
- 重点观察Task的切换 和 self._ready的内容
```python
import asyncio
import types
# from types import Awaitable

def task_print(s):
    print(f"{asyncio.current_task().get_name()}: {s}")


async def coro_without_await(s):
    """不带await语句的协程"""
    task_print("coro_without_await start...")
    task_print(s)
    task_print("coro_without_await end.")


class Awaitable(object):
    """可等待对象, 要求实现了__await__方法且返回生成器(也即包含yield语句)"""
    def __await__(self):
        task_print("Awaitable.__await__ start...")
        task_print("the 'await' does nothing to pause the current task")
        task_print("before 'yield'")
        yield None
        task_print("after 'yield'")
        task_print("Awaitable.__await__ continue..., cause the task 'coro_without_await' completed")
        task_print("Awaitable.__await__ end.")

def gen_with_yield():
    """普通生成器"""
    task_print("gen_with_yield start...")
    task_print("before 'yield'")
    yield None # None or a Future object 
    task_print("after 'yield'")
    task_print("gen_with_yield end.")


async def coro_with_await():
    """带有await语句的协程"""
    task_print("coro_with_await start...")
    task_print("the 'await' does nothing to pause the current task")
    await Awaitable()
    task_print("coro_with_await continue..., cause Awaitable.__await__ end. leaving an '__await__' does nothing to pause the task")
    task_print("coro_with_await end.")


@types.coroutine
def coro_with_types_coroutine():
    """装饰的出来的协程(要求是有yield语句)，本质是底层封装了一个可等待对象"""
    task_print("coro_with_types_coroutine start...")
    task_print("before 'yield'")
    yield None # None or a Future object are the only legitimate yields to the task in asyncio
    task_print("after 'yield'")
    task_print("coro_with_types_coroutine continue..., cause the task 'coro_without_await' completed")
    task_print("coro_with_types_coroutine end.")

@types.coroutine
def coro_with_types_coroutine_yield_from():
    """装饰的出来的协程(要求是有yield语句)"""
    task_print("coro_with_types_coroutine_yield_from start...")
    task_print("before 'yield'")
    # yield from Awaitable().__await__()
    yield from gen_with_yield()
    task_print("after 'yield'")
    task_print("coro_with_types_coroutine_yield_from continue..., cause the task 'coro_without_await' completed")
    task_print("coro_with_types_coroutine_yield_from end.")


# async def coro_with_yield():
#    """错误的函数"""
#     task_print("coro_with_yield start...")
#     task_print("before 'yield'")
#     yield None # None or a Future object 
#     task_print("after 'yield'")
#     task_print("coro_with_yield end.")

async def main():
    task_print("main start...")
    asyncio.create_task(coro_without_await("'yield' does pause the current task, allowing the task 'coro_without_await' to run"))
    await coro_with_await()
    task_print("main continue...., cause 'coro_with_await' end. leaving an 'coro' does nothing to pause the task")

    task_print("---------------------------")
    asyncio.create_task(coro_without_await("'yield' does pause the current task, allowing the task 'coro_without_await' to run"))
    await Awaitable()

    task_print("---------------------------")
    asyncio.create_task(coro_without_await("'yield' does pause the current task, allowing the task 'coro_without_await' to run"))
    res = await coro_with_types_coroutine()
    assert res is None

    task_print("---------------------------")
    asyncio.create_task(coro_without_await("'yield' does pause the current task, allowing the task 'coro_without_await' to run"))
    await coro_with_types_coroutine_yield_from()

    asyncio.create_task(coro_without_await("main task finished, loop will handle this task, cause 'types.coroutine  yield' is like 'sync yield' "))
    for y in coro_with_types_coroutine():
        task_print(f"")
    task_print("main end.")


asyncio.run(main())

```
# 核心函数分析
## asyncio.run

1. 启动 asyncio.run(main) 
- [https://github.com/python/cpython/blob/main/Lib/asyncio/runners.py](https://github.com/python/cpython/blob/main/Lib/asyncio/runners.py)
- **此函数一般作为asyncio程序的主入口点**
- main必须是coroutine
2. 创建事件循环 loop = events.new_event_loop()
- **创建全局策略为 _event_loop_policy**,  linux下默认为_UnixDefaultEventLoopPolicy
- 基于策略类返回一个loop，linux下默认为_UnixSelectorEventLoop
3. 设置事件循环 events.set_event_loop(loop)
- 设置全局策略的loop为**_event_loop_policy._local._loop**
4. 开启事件循环 loop.run_until_complete(main)

## BaseEventLoop.run_until_complete

1. 调用BaseEventLoop.run_until_complete(future)
- future为协程类型，async def func(), func为function类型，func()为coroutine类型
2. **创建一个task，加入事件循环，tasks.ensure_future(future, loop=self)**
- loop.create_task(coro_or_future)
- task = Task.__init__(coro, loop)，task为future对象的子类
   - 添加进_ready以便调用，self.call_soon(callback=Task.__step, context=Task._context)
      - handle = events.Handle(callback, args, loop, context)
      - **self._ready.append(handle)**
   - 全局_all_tasks.add(task)
3. future.add_done_callback(_run_until_complete_cb)，添加完成时统一回调函数
4. **self.run_forever()**
5. return future.result()
## BaseEventLoop.run_forever

1. 设置running的loop，events._set_running_loop(self)
2. 死循环调用 self._run_once()
## BaseEventLoop._run_once
遍历处于ready的handle，handle是对task.__step封装，

1. 定时调度任务 准备工作
- 清理取消的调度任务中的handle
2. **io事件加入ready**
- 计算select超时时间，取self._scheduled[0]，也即最小值，不能超过调度任务的超时时间
- 获取ready io事件，event_list = self._selector.select(timeout)
   - selector为selectors.DefaultSelector()
- self._process_events(event_list)
   - self._add_callback(reader/writer)
   - **self._ready.append(handle)**
3. **定时到达任务加入ready**
- while self._scheduled:
   - if handle._when >= end_time: break
   - **self._ready.append(handle)**
4. 开始遍历本次self._ready
- handle._run

## events.Handle

1. handle._run()
2. self._context.run(self._callback, *self._args)
3. 开始调用注册的回调函数 _callback(args)， 比如 task.__step

## Task.__step

1. 执行协程，result = coro.send()
- 执行coro中的await chain，await不会挂起，直到遇到第一个yield
- yield返回值和控制权给await chain直到result = coro.send()
2. 判断result
- result = None，会call_soon(__step)继续加入事件循环，下次从await chain进入执行yield的下半部分
- result 为 future对象
   - 添加回调函数result.add_done_callback(self.__wakeup, context=self._context)
   - future set_result时，会call_soon(callback)把callback加入事件循环
   - self.__wakeup会唤起等待这个future对象完成的task继续执行 task.__step
- 如果触发StopIterationawait
   - 将结果从StopIteration取出，task/future.set_result设置结果完成，将回调加入事件循环
   - 并赋值给tas继续执行coro await后面的语句，直到结束或遇到另一个await chain+yield

## Task.__wakeup
查看 await对象的执行结果结果情况

1. 执行 future.result()
- 如果未完成，返回异常。**应该完成，因为只有future完成时才会把__wakeup加入到事件循环中**
2. 如果future对象正常完成， 会执行self.__step()
- result = coro.send()，这次从await chain开始执行到yield的后半逻辑，

执行完后面会触发StopIteration

- await会将结果从StopIteration取出并赋值，继续执行coro await后面的语句，直到结束或遇到另一个await chain+yield



