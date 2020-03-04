## 协程asyncio

 并发（concurrency）的一种方式 。

带来的不是真正的并行。Python多线程也不能带来真正的并行。

一个协程可以放弃执行，把机会让给其他协程（yield form 或await）。

### 定义

```
async def语句
```

```
async def do_some_work(x):
    print("Waiting " + str(x))
    await asyncio.sleep(x)
```

协程可以：

- 等待一个future结束
- 等待另一个协程（产生一个结果或引发一个异常）
- 产生一个结果给正在等待他的协程
- 引发一个异常给正在等待他的协程

asyncio.sleep也是一个协程

### 运行

调用协程函数，协程不会运行，只是返回一个协程对象。

要让这个协程对象运行有两种方式：

- 在一个运行的协程中用 await 等待它
- 通过 ensure_future 函数计划他的运行

Loop运行了协程才可以运行。

```
# 协程对象交给loop
loop = asyncio.get_event_loop()
loop.run_until_complete(do_some_work(3))
```

run_until_complete()是一个阻塞调用， 直到协程运行结束后它才返回。

#### run_until_complete()

**参数：**future

ensure_future 函数把协程对象包装成了 future。

```
loop.run_until_complete(asyncio.ensure_future(do_some_work(3)))
```

但实际操作上 run_until_complete() 内部会做检查，会自动将 协程对象 封装为 future。

### 回调函数

**需求场景：**

协程是一个 IO 的读写操作，等它读完数据后，我们希望得到通知，以便下一步数据的处理。

往future 里添加回调。

```
def done_callback(futu):
    print('Done')

futu = asyncio.ensure_future(do_some_work(3))
futu.add_done_callback(done_callback)

loop.run_until_complete(futu)
```

### 多个协程

将多个协程交给loop，需要借助 **asyncio.gather 函数**

```
loop.run_until_complete(asyncio.gather(do_some_work(1), do_some_work(3)))
```

先将**协程存在列表里**

```
coros = [do_some_work(1), do_some_work(3)]
loop.run_until_complete(asyncio.gather(*coros))
```

协程之间是并发进行的。

### run_until_complete和run_forever

 `run_until_complete` 来运行 loop ，等到 future 完成，`run_until_complete` 也就返回了。 

```
async def do_some_work(x):
    print('Waiting ' + str(x))
    await asyncio.sleep(x)
    print('Done')
loop = asyncio.get_event_loop()
coro = do_some_work(3)
loop.run_until_complete(coro)

输出：
Waiting 3
<等待三秒钟>
Done
<程序退出>
```

 而`run_forever` 会一直运行，直到 `stop`被调用 。

但是如果 `run_forever` 不返回，`stop` 永远也不会被调用。所以，只能在协程中调 `stop` 。

```
# loop里面单个协程的程序退出
async def do_some_work(loop, x):
    print('Waiting ' + str(x))
    await asyncio.sleep(x)
    print('Done')
    loop.stop()
```

loop里面多个协程的程序退出，需要用到 gather 将多个协程合并为一个future， 并添加 回调， 然后在回调里面去停止loop。

```
# loop里面多个协程的程序退出
async def do_some_work(loop, x):
    print('Waiting ' + str(x))
    await asyncio.sleep(x)
    print('Done')

def done_callback(loop, futu):
    loop.stop()

loop = asyncio.get_event_loop()

futus = asyncio.gather(do_some_work(loop, 1), do_some_work(loop, 3))
futus.add_done_callback(functools.partial(done_callback, loop))

loop.run_forever()
```

### loop.close

loop只要不关闭就还可以再运行。

如果关闭了就不能再运行了。

```
loop.run_until_complete(do_some_work(loop, 1))
loop.close()
loop.run_until_complete(do_some_work(loop, 3))  # 此处异常
```

### gather和wait

功能相似

### timer

python没有原生支持timer，不过可以用 asyncio.sleep来模拟。

```
async def timer(x, cb):
    futu = asyncio.ensure_future(asyncio.sleep(x))
    futu.add_done_callback(cb)
    await futu

t = timer(3, lambda futu: print('Done'))
loop.run_until_complete(t)
```