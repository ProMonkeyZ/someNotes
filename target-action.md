target-action是消息传递机制中的一种方式.多用于UIButton的点击事件绑定,我们可以给NSObject自定义一个target-action对.我们可以给NSObject添加方法:
```
- (void)addTarget:(id)target action:(SEL)selector;
```
添加属性进行记录:
```
@interface LNModel ()

@property(nonatomic, weak) id target;
@property(nonatomic, assign) SEL selector;

@end
```
方法实现:
```
@implementation LNModel

- (void)addTarget:(id)target action:(SEL)selector {
    _target = target;
    _selector = selector;
}

/**
 到达一定条件之后就通过performSelector的方式来调用"选择子"进行方法调用.
 */
- (void)runAction {
    IMP imp = [self.target methodForSelector:self.selector];
    void (*func)(id,SEL) = (void *)imp;
    func(self.target,self.selector);
}

@end
```
#### 注意:
需要注意,在runAction方法中我们为什么不适用```[self.target performSelector:self.selector];```的方式进行方法调用,原因在于ARC模式下编译会出现一个```"performSelector may cause a leak because its selector is unknown"```的warning,意思是编译器无法检查实际上是否有这个选择子,但是为什么会造成内存泄露?

在ARC下调用一个方法,runtime需要知道对于方法的方绘制应该怎样处理.返回值可能有void,string,id等各种类型.ARC一般根据返回值类型进行对应处理,一共有如下戏中情况:
1. 直接忽略(void,int)
2. 先retain,等用不到的时候再release.
3. 不retain,用不到的时候直接release(int, copy这一类的方法)
4. 设么也不做,默认返回值在返回前后是始终有效的(一直到最近的release pool结束为止)

**调用```performSelector:```的时候,系统会默认返回值是个对象,单页不会retain或release,也就是默认采用第四种做法.所以如果这个选择子所对应的方法属于前三种,都有可能会造成内存泄露.**