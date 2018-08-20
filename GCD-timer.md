### 使用GCD创建一个timer
```
// 定义回调block
typedef void(^nomalVoidHandler)(dispatch_source_t timer);

+ (void)dispatchTimerWithTarget:(id)target
                       interval:(NSTimeInterval)timeInterval
                        handler:(nomalVoidHandler)handler {
    // 创建一个队列,用以执行timer
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    // 创建一个timer
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER,0, 0, queue);
    // 设置timer执行频率和实行次数
    dispatch_source_set_timer(timer, dispatch_walltime(NULL, 0), (uint64_t)(timeInterval *NSEC_PER_SEC), 0);
    // 设置timer执行回调
    __weak __typeof(target) weaktarget  = target;
    dispatch_source_set_event_handler(timer, ^{
        if (!weaktarget)  {
            dispatch_source_cancel(timer);
        } else {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (handler) {
                    handler(timer);
                }
            });
        }
    });
    // 启动定时器
    dispatch_resume(timer);
}
```