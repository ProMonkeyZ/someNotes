### performselector

在objective-c中,真正的方法调用只有在运行时才能知道,编译时不能确定最后终将调用那个方法,所有OC的方法调用除了使用

**[类名/对象 + 方法名]**

调用以外还可以使用performselector方法进行调用,传入SEL(方法选择器).
```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
    
}];
[self performSelector:@selector(viewWillAppear:) onThread:thread withObject:nil waitUntilDone:NO];
```
还可以使用runtime库实现相同的功能.使用时需要导入头文件:
> #import <objc/message.h>

调用带参数的msgSend方法的时候需要在buildSetting里面设置"Eable Strict Checking of objc_msgSend Calls"为NO
```
- (void)customPerformSelectorIn:(SEL)aSelector withObject:(id)arg {
    objc_msgSend(self, aSelector);
}
```