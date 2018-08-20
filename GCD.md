### 多线程
多线程一般有四种方式:
1. pthread
2. nsthread
3. nsoperation
4. gcd

现在经常使用的是gcd技术进行多线程的实现,这套框架是苹果专为多核硬件开发的.
### 队列和线程
在使用gcd的时候要明确两个概念:**队列**,**任务**. 队列有两种:
1. 串行队列
> 使用串行队列的时候遵守FIFO原则,任务按顺序执行.所以串行队列只在一个线程里面顺序执行任务.
* 异步将block加入串行队列
```
    dispatch_queue_t serialQueue = dispatch_queue_create("com.zln", DISPATCH_QUEUE_SERIAL);
    dispatch_async(serialQueue, ^{
        NSLog(@"block 1, %@",[NSThread currentThread]);
    });
    dispatch_async(serialQueue, ^{
        NSLog(@"block 2, %@",[NSThread currentThread]);
    });
    dispatch_async(serialQueue, ^{
        NSLog(@"block 3, %@",[NSThread currentThread]);
    });
    dispatch_async(serialQueue, ^{
        NSLog(@"block 4, %@",[NSThread currentThread]);
    });
    NSLog(@"mainthread, %@",[NSThread currentThread]);
```
输出:
```
2018-08-16 10:39:33.164870+0800 APPCodeTest[1676:108747] block 1, <NSThread: 0x604000070680>{number = 3, name = (null)}
2018-08-16 10:39:33.164872+0800 APPCodeTest[1676:108569] mainthread, <NSThread: 0x604000073440>{number = 1, name = main}
2018-08-16 10:39:33.165097+0800 APPCodeTest[1676:108747] block 2, <NSThread: 0x604000070680>{number = 3, name = (null)}
2018-08-16 10:39:33.165246+0800 APPCodeTest[1676:108747] block 3, <NSThread: 0x604000070680>{number = 3, name = (null)}
2018-08-16 10:39:33.165661+0800 APPCodeTest[1676:108747] block 4, <NSThread: 0x604000070680>{number = 3, name = (null)}

```
通过输出可以看出来,用异步方法加入的block不会阻塞主线程任务.因为使用异步函数的时候,即使是串行队列也会新开线程.
* 同步将block加入串行队列
```
    dispatch_queue_t serialQueue = dispatch_queue_create("com.zln", DISPATCH_QUEUE_SERIAL);
    dispatch_sync(serialQueue, ^{
        NSLog(@"block 1, %@",[NSThread currentThread]);
    });
    dispatch_sync(serialQueue, ^{
        NSLog(@"block 2, %@",[NSThread currentThread]);
    });
    dispatch_sync(serialQueue, ^{
        NSLog(@"block 3, %@",[NSThread currentThread]);
    });
    dispatch_sync(serialQueue, ^{
        NSLog(@"block 4, %@",[NSThread currentThread]);
    });
    NSLog(@"mainthread, %@",[NSThread currentThread]);
```
输出结果:
```
2018-08-16 10:42:48.712633+0800 APPCodeTest[1749:119081] block 1, <NSThread: 0x604000260200>{number = 1, name = main}
2018-08-16 10:42:48.712841+0800 APPCodeTest[1749:119081] block 2, <NSThread: 0x604000260200>{number = 1, name = main}
2018-08-16 10:42:48.713002+0800 APPCodeTest[1749:119081] block 3, <NSThread: 0x604000260200>{number = 1, name = main}
2018-08-16 10:42:48.713134+0800 APPCodeTest[1749:119081] block 4, <NSThread: 0x604000260200>{number = 1, name = main}
2018-08-16 10:42:48.713399+0800 APPCodeTest[1749:119081] mainthread, <NSThread: 0x604000260200>{number = 1, name = main}
```
通过同步方式加入的任务不会新开线程,就在当前线程执行,而且会阻塞当前线程,必须是一个任务执行完毕之后才会继续执行下一个任务.
2. 并行队列
> 使用并行队列,队列中任务没有固定的执行顺序.所以并行队列是需要使用多条线程,具体有多少线程,取决于UNIX内核.
* 使用并行队列异步方式加入block,查看任务执行顺序:
```
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.zln.CONCURRENT", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(concurrentQueue, ^{
        NSLog(@"block 1, %@",[NSThread currentThread]);
    });
    dispatch_async(concurrentQueue, ^{
        NSLog(@"block 2, %@",[NSThread currentThread]);
    });
    dispatch_async(concurrentQueue, ^{
        NSLog(@"block 3, %@",[NSThread currentThread]);
    });
    dispatch_async(concurrentQueue, ^{
        NSLog(@"block 4, %@",[NSThread currentThread]);
    });
    NSLog(@"thread, %@",[NSThread currentThread]);
```
输出结果:
```
2018-08-16 11:40:15.694871+0800 APPCodeTest[2942:236819] thread, <NSThread: 0x6000000681c0>{number = 1, name = main}
2018-08-16 11:40:15.694871+0800 APPCodeTest[2942:236910] block 1, <NSThread: 0x60000026af40>{number = 3, name = (null)}
2018-08-16 11:40:15.694871+0800 APPCodeTest[2942:236911] block 4, <NSThread: 0x60000026ad40>{number = 6, name = (null)}
2018-08-16 11:40:15.694907+0800 APPCodeTest[2942:236912] block 3, <NSThread: 0x60000026ae80>{number = 5, name = (null)}
2018-08-16 11:40:15.694909+0800 APPCodeTest[2942:236908] block 2, <NSThread: 0x600000066080>{number = 4, name = (null)}
```
* 使用并行队列同步方式加入block:
```
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.zln.CONCURRENT", DISPATCH_QUEUE_CONCURRENT);
    dispatch_sync(concurrentQueue, ^{
        NSLog(@"block 1, %@",[NSThread currentThread]);
    });
    dispatch_sync(concurrentQueue, ^{
        NSLog(@"block 2, %@",[NSThread currentThread]);
    });
    dispatch_sync(concurrentQueue, ^{
        NSLog(@"block 3, %@",[NSThread currentThread]);
    });
    dispatch_sync(concurrentQueue, ^{
        NSLog(@"block 4, %@",[NSThread currentThread]);
    });
    NSLog(@"thread, %@",[NSThread currentThread]);
```
输出结果:
```
2018-08-16 11:41:59.817925+0800 APPCodeTest[2990:243588] block 1, <NSThread: 0x60400006cd40>{number = 1, name = main}
2018-08-16 11:41:59.818111+0800 APPCodeTest[2990:243588] block 2, <NSThread: 0x60400006cd40>{number = 1, name = main}
2018-08-16 11:41:59.818296+0800 APPCodeTest[2990:243588] block 3, <NSThread: 0x60400006cd40>{number = 1, name = main}
2018-08-16 11:41:59.818746+0800 APPCodeTest[2990:243588] block 4, <NSThread: 0x60400006cd40>{number = 1, name = main}
2018-08-16 11:41:59.818892+0800 APPCodeTest[2990:243588] thread, <NSThread: 0x60400006cd40>{number = 1, name = main}
```