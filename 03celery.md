## celery

管理分布式任务队列的工具。

框架图：

 ![celery架构图](https://upload-images.jianshu.io/upload_images/9419034-87ea2c0bc2cf1b8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

- Producer：调用了Celery提供的API、函数或者装饰器而产生任务并交给任务队列处理的都是任务生产者。
- Celery Beat：任务调度器，Beat进程会读取配置文件的内容，周期性地将配置中到期需要执行的任务发送给任务队列。
- Broker：消息代理，又称消息中间件，接受任务生产者发送过来的任务消息，存进队列再按序分发给任务消费方（通常是消息队列或者数据库）。Celery目前支持RabbitMQ、Redis、MongoDB、Beanstalk、SQLAlchemy、Zookeeper等作为消息代理，但适用于生产环境的只有RabbitMQ和Redis。
- Celery Worker：执行任务的消费者，通常会在多台服务器运行多个消费者来提高执行效率。
- Result Backend：任务处理完后保存状态信息和结果，以供查询。Celery默认已支持Redis、RabbitMQ、MongoDB、Django ORM、SQLAlchemy等方式。

客户端和消费者之间传输数据需要序列化和反序列化。

 ![celery序列化](https://upload-images.jianshu.io/upload_images/9419034-d4ee3804d2031aa5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

### 进阶用法

```

# 装饰器将一个正常的函数修饰成了一个 celery  task对象。
@app.task  #普通函数装饰为 celery task
def add(x, y):
    return x + y
```

首先可以让修饰的函数成为 task对象 的绑定方法。 == 被修饰的函数成为了 实例方法。这时可以调用 self 获取当前 task 实例的很多状态及属性。

其次我们可以自己复写 类来做一些自定义的操作。

#### 根据任务状态执行不同操作

```
# tasks.py
class MyTask(Task):
    def on_success(self, retval, task_id, args, kwargs):
        print 'task done: {0}'.format(retval)
        return super(MyTask, self).on_success(retval, task_id, args, kwargs)
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print 'task fail, reason: {0}'.format(exc)
        return super(MyTask, self).on_failure(exc, task_id, args, kwargs, einfo)
 
@app.task(base=MyTask)
def add(x, y):
    return x + y
```

#### 绑定任务为实例方法

```
from celery.utils.log import get_task_logger
 
logger = get_task_logger(__name__)
@app.task(bind=True)
def add(self, x, y):
    logger.info(self.request.__dict__)
    return x + y
```

#### 任务状态回调

|  参数   |     说明     |
| :-----: | :----------: |
| PENDING |  任务等待中  |
| STARTED |  任务已开始  |
| SUCCESS | 任务执行成功 |
| FAILURE | 任务执行失败 |
|  RETRY  | 任务将被重试 |
| REVOKED |   任务取消   |

当 耗时时间较长的任务进行 想得知它的实时进度时，需要自定义一个任务状态来说明进度并手动更新状态，从而告诉回调当前任务的进度。

```
from celery import Celery
import time
 
@app.task(bind=True)
def test_mes(self):
    for i in xrange(1, 11):
        time.sleep(0.1)
        self.update_state(state="PROGRESS", meta={'p': i*10})
    return 'finish'
```

```

# trigger.py
from task import add,test_mes
import sys
 
def pm(body):
    res = body.get('result')
    if body.get('status') == 'PROGRESS':
        sys.stdout.write('\r任务进度: {0}%'.format(res.get('p')))
        sys.stdout.flush()
    else:
        print '\r'
        print res
r = test_mes.delay()
print r.get(on_message=pm, propagate=False)
```

#### 定时、周期任务

只需要再配置中配置好周期任务，运行一个周期任务触发器(beat)即可：

```
# celery_config.py
from datetime import timedelta
from celery.schedules import crontab
 
CELERYBEAT_SCHEDULE = {
    'ptask': {
        'task': 'tasks.period_task',
        'schedule': timedelta(seconds=5),
    },
}
 
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

定时任务涉及到 datetime 需要再配置中 加入时区信息，默认 utc

```
CELERY_TIMEZONE = 'Asia/Shanghai'
```

```
# tasks.py
# 增加要被周期执行的任务
app = Celery('tasks', backend='redis://localhost:6379/0', broker='redis://localhost:6379/0')
app.config_from_object('celery_config')
 
@app.task(bind=True)
def period_task(self):
    print 'period task done: {0}'.format(self.request.id)
```

#### 链式任务

几个子任务组成，尽量不要以同步阻塞的方式调用子任务，而用异步回调的方式进行链式任务的调用。

链式任务中前一个任务的返回值 默认是下一个任务的输入值之一。

不想让返回值做默认参数可以用 si() 或者 s(immutable=True) 的方式调用 。

 `s()` 是方法 `celery.signature()` 的快捷调用方式。

signature 具体作用就是生成一个包含调用任务及其调用参数与其他信息的对象，个人感觉有点类似偏函数的概念：先不执行任务，而是把任务与任务参数存起来以供其他地方调用。 

```
# 方法一
def update_page_info(url):
    # fetch_page -> parse_page -> store_page
    chain = fetch_page.s(url) | parse_page.s() | store_page_info.s(url)
    chain()
 
@app.task()
def fetch_page(url):
    return myhttplib.get(url)
 
@app.task()
def parse_page(page):
    return myparser.parse_document(page)
 
@app.task(ignore_result=True)
def store_page_info(info, url):
    PageInfo.objects.create(url=url, info=info)
```

```
# 方法二
fetch_page.apply_async((url), link=[parse_page.s(), store_page_info.s(url)])
```

#### 调用任务

实际上， delay 只是 apply_async 的快捷方式，二者作用相同。只是 apply_async 可以进行更多的任务属性设置。

####  AsyncResult

用来储存任务执行信息和执行结果。

配合 gevent 一起使用就可以 像写js promise 写回调：

```
import gevent.monkey
monkey.patch_all()
 
import time
from celery import Celery
 
app = Celery(broker='amqp://', backend='rpc')
 
@app.task
def add(x, y):
    return x + y
 
def on_result_ready(result):
    print('Received result for id %r: %r' % (result.id, result.result,))
 
add.delay(2, 2).then(on_result_ready)
```

这种promise写法 现在只能用在 backend 是 RPC 或 redis 时。并且独立使用时需要引入猴子补丁，可能会影响其他代码。

这个特性结合 tornado/twisted 等异步框架更合适。