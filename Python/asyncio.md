通过`async/await`语法来声明`协程`是编写`asyncio`应用的推荐方式。
```python
	import asyncio
	
	async def main():
		print('hello')
		await asynico.sleep(1)
		print('world')
	asyncio.ruin (main())

hello
world
```
ps:简单地调用一个协程并不会使其被调度执行
```
	main()
<coroutine object main at ......>
```

要真正运行一个协程，asyncio提供了三种主要机制：
+ `asyncio.run()`函数用来运行最高层次的入口点“main（）”函数
+ 使用`await`等待一个协程。

```python
import asyncio
import time 

async def say_after(delay, what):
	await asyncio.sleep(delay)
	print(what)

async def main():
	print(f'started at {time.strftime('%X')}')
	
	await say_after(1, 'hello')
	await say_after(2, 'world')
	
	print(f'finished at {time.strftime('%x')}')

asynico.run(main())
```
output:
```
started at 17:13:52
hello
world
finished at 17:13:55
```
+ asyncio.create_task()函数用来并发运行作为asyncio任务的多个协程
```python
async def main():
	task1 = asyncio.create_task(say_after(1, 'hello'))
	
	task2 = asyncio.create_task(say_after(2, 'world'))
	
	print(f'started at {time.strftime('%x')}')
	
	await task1
	await task2
	
	print(f'finished at {time.strftime('%x')}')
```
output
```
started an 17:14:32
hello
world
finished at 17:14:34
```
比上一种方法快了一秒（并发）
### 可等待对象
如果一个对象可以在`await`语句中使用，那么他就是可等待对象。
许多asyncio API都被设计为接受可等待对象。
可等待对象有三种主要类型：协程，任务和Future
*协程*
 + 协程函数：定义形式为`async def`的函数
 + 协程对象：调用`协程函数`所返回的对象
asyncio也支持旧时的`基于生成器`的协程
*任务*
任务被用来“并行的”调用协程
当一个协程通过`anyncio.create_task()`等函数被封装为一个任务，该协程会被自动调度执行
*Futures*
Future是一种特殊的底层级可等待对象，表示一个异步操作的最终结果。
当一个Future对象被等待，这意味着协程将保持等待知道该Future对象在其他地方操作完毕。
在asyncio中需要Future对象以便允许通过async/await使用基于回调的代码。
通常情况下没有必要在应用层级的代码中创建Future对象。
