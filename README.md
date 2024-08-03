# 消息队列
这是一个C语言消息队列实现。内部使用get队列和put队列分离的机制，实现高并发。由于使用严格的加锁机制，消息严格保序。
## 接口
### 创建队列
**msgqueue_t \*msgqueue_create(size_t maxlen, int linkoff);**  

@maxlen: put队列的最大长度。由于内部使用双队列，默认模式下实际未消费的消息数可能达到两倍maxlen。  

@linkoff: 指定一个消息拉链的偏移量。每一条进入消息队列的消息，用户需要在这个偏移位置保留一个指针的空间，用于内部拉链。linkoff可以是正数，还可以是负数或0。通过指定拉链偏移的方法，模块内部无需再为消息分配和释放内存了。    

函数返回一个msgqueue_t \*类型的消息队列对象。或返回NULL表示失败，同时设置errno。  

### 发送消息
**void msgqueue_put(void \*msg, msgqueue_t \*queue);**  

@msg: 消息指针。注意 **(char \*)msg + linkoff**的位置，应该预留一个指针空间。  

@queue: 消息队列对象。  

函数无返回值。在默认的堵塞模式下，如果put队列达到maxlen，则操作堵塞。而在非堵塞模式下，操作立即返回，消息被发送。  

### 接收消息
**void \*msgqueue_get(msgqueue_t \*queue);**

@queue: 消息队列对象。  

函数取出并返回最早写入一条消息。如果没有消息，在堵塞模式下将进行进行等待。在非堵塞模式下，返回NULL。  

### 设置非堵塞模式
**void msgqueue_set_nonblock(msgqueue_t \*queue);**  

@queue: 消息队列对象。  

函数将消息队列设置为非堵塞。所有put或get操作，不会堵塞。在非堵塞模式下，队列长度没有限制。  

### 设置堵塞模式
**void msgqueue_set_block(msgqueue_t \*queue);**  

@queue: 消息队列对象。  

函数将消息队列设置为默认的堵塞模式。  

### 销毁消息队列
**void msgqueue_destroy(msgqueue_t \*queue);**

@queue: 要销毁的消息队列对象。  

函数销毁消息队列。如果有消息还没有被接收，用户需要自行处理。  

## 示例
~~~c
#include <assert.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include "msgqueue.h"

int main(void)
{
    struct msg
    {     
        char str[64];
        void *link;
    };

    msgqueue_t *mq = msgqueue_create(10, 64/* offset of 'link' field*/);
    struct msg *in = (struct msg *)malloc(sizeof *in);
    strcpy(in->str, "hello");
    msgqueue_put(in, mq);

    struct msg *out;
    out = msgqueue_get(mq);
    printf("%s\n", out->str);

    assert(in == out);
    free(in);
    return 0;
}
~~~

# 线程池
线程池是基于消息队列实现的。利用了消息队列高并发高性能的特征，具备非常高的调度性能。  
## 接口
### 创建线程池
**thrdpool_t \*thrdpool_create(size_t nthreads, size_t stacksize);**  

@nthreads: 线程的数量。  

@stacksize: 线程的堆栈大小。0表示使用默认值。  

函数成功返回thrdpoo_t \*类型的线程池对象。失败返回NULL并设置errno。  

### 提交任务
**int thrdpool_schedule(const struct thrdpool_task \*task, thrdpool_t \*pool);**  

@task: struct thrdpool_task类型任务指针，该结构包含了任务的函数入口和上下文。定义如下：  
~~~c
struct thrdpool_task
{
    void (*routine)(void *);
    void *context;
};
~~~
@pool: 线程池对象。  

函数返回0表示成功，返回-1表示提交失败，并设置errno。用户可以安全的在线程池的任务里调用该函数提交一个新的任务，哪怕此刻有另一个线程正在销毁线程池。  

### 增加线程
**int thrdpool_increase(thrdpool_t \*pool);**

@pool:  线程池对象。  

函数返回0表示成功，此时线程池增加了一个线程。返回-1表示失败，并设置errno。  

### 减少线程
**int thrdpool_decrease(thrdpool_t \*pool);**

@pool:  线程池对象。  

函数返回-1表示失败，并设置errno。当函数返回0代表成功，存在三种情况：
* 当前有空闲线程，那么会有一个空闲线程退出。
* 所有线程都正在工作，第一个进入空闲的线程将会退出。
* 逻辑上线程数已经为负数，下一个被增加的线程会直接退出。

注意函数不会等待线程退出完毕，而总是立刻返回。

### 判断当前线程是否为线程池里的线程

**int thrdpool_in_pool(thrdpool_t \*pool);**  

@pool:  线程池对象。  

如果当前线程是线程池线程，返回1，否则返回0。  

### 退出当前线程

**void thrdpool_exit(thrdpool_t \*pool);**

@pool:  线程池对象。  

如果当前线程是线程池中的线程，立即退出并且线程池减少一个线程。如果调用线程不是在线程池中，无任何行为。

### 销毁线程池
**void thrdpool_destroy(void (\*pending)(const struct thrdpool_task \*), thrdpool_t \*pool);**  

@pending: 用于返回未执行任务的回调函数。当pending为NULL，未执行的任务直接丢弃。  

@pool: 要销毁的线程池对象。  

函数销毁线程池，并通过pending回调函数返回还没有被执行的任务。在一个线程任务里调用本函数是一个合法行为，这种情况下线程池正常被销毁，调用者线程不会立刻中断，而是正常运行至routine结束。  


