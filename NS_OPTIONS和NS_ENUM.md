### 含义
都是代表枚举.

1. NS_OPTIONS的枚举项类型是NSUInteger(无符号整型)类型的,只能使用1<<0, 1<<1, 1<<2这样2的N次方的定义方式.
2. NS_ENUM的枚举项是NSInteger类型的.

### 区别
1. NS_OPTIONS定义位移枚举,NS_ENUM定义通用枚举.位移枚举是可以在用的时候给一个属性赋值多个枚举值:
```
UISwipeGestureRecognizer *swipeGR = [[UISwipeGestureRecognizer alloc] init];
// 这里几个枚举项同时存在表示它的方向同时包含1.向下2.向左3.向右
swipeGR.direction = UISwipeGestureRecognizerDirectionDown | UISwipeGestureRecognizerDirectionLeft | UISwipeGestureRecognizerDirectionRight;
```
2. 之所以两个枚举不能直接使用成一个,而且还要枚举值多选.是因为按照C++模式编译,所以C++模式下定义了NS_OPTIONS宏以保证不出现类型转换.