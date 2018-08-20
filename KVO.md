#### 1.定义及使用
***KVO(Key Value Observ)*** ,说起来就是键值对监听.使用方法:

先定义一个类:

```
@interface LNModel : NSObject

@property(nonatomic, assign) NSInteger age;
@property(nonatomic, copy) NSString *firstName;
@property(nonatomic, strong) NSString *lastName;

@end

@implementation LNModel

- (NSString *)description {
    return [NSString stringWithFormat:@"firstName:%@,lastName:%@,age:%zd",self.firstName,self.lastName,self.age];
}

@end

```
下面我们在VC里面监听这个类对象的age属性的变化:
```
@interface ViewController ()

@property(nonatomic, strong) LNModel *model;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self KVOTest];
}

- (void)KVOTest {
    [self.model addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:@"LNModel age"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if ([keyPath isEqualToString:@"age"]) {
        NSLog(@"%@",object);
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.model.age ++;
}

- (LNModel *)model {
    if (!_model) {
        _model = [LNModel new];
        _model.firstName = @"张";
        _model.lastName = @"三丰";
        _model.age = 11;
    }
    return _model;
}

@end
```
打印结果如下:

```
2018-08-09 10:29:20.841105+0800 APPCodeTest[5522:229889] firstName:张,lastName:三丰,age:12
2018-08-09 10:29:21.041945+0800 APPCodeTest[5522:229889] firstName:张,lastName:三丰,age:13
2018-08-09 10:29:21.222183+0800 APPCodeTest[5522:229889] firstName:张,lastName:三丰,age:14
2018-08-09 10:29:21.424226+0800 APPCodeTest[5522:229889] firstName:张,lastName:三丰,age:15
```
#### 2.实现原理
***KVO(Key Value Observ)*** ,本身是由runtime实现的.在我们对一个类对象的属性对象进行监听的时候.系统运行时代码会为我们***动态***的生成一个该类的派生类.在这个派生类中重写所有基类中被观察属性的setter.派生类在setter内实现真正的 **通知机制**.

接下来我们稍微修改一下上面监听到age变化之后的代码,用runtime中的方法获取一下监听到变化的object的真正的类:
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    Class cls = object_getClass(object);
    NSLog(@"%@",cls);
    if ([keyPath isEqualToString:@"age"]) {
        NSLog(@"%@",object);
    }
}
```
打印结果如下:
```
2018-08-09 10:45:04.228725+0800 APPCodeTest[5987:283694] NSKVONotifying_LNModel
2018-08-09 10:45:04.229197+0800 APPCodeTest[5987:283694] firstName:张,lastName:三丰,age:12
2018-08-09 10:46:38.591659+0800 APPCodeTest[5987:283694] NSKVONotifying_LNModel
2018-08-09 10:46:38.592059+0800 APPCodeTest[5987:283694] firstName:张,lastName:三丰,age:13
2018-08-09 10:46:39.256102+0800 APPCodeTest[5987:283694] NSKVONotifying_LNModel
2018-08-09 10:46:39.256664+0800 APPCodeTest[5987:283694] firstName:张,lastName:三丰,age:14
```
可以看到我们监听到变化的对象的所属类已经不是我们定义的**LNModel**,而是系统帮我们创建的派生类**NSKVONotifying_LNModel**了.
#### 实现一个KVO,添加监听的方法
给NSObject些个分类,增加
> -(void)ln_addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void *)context;

方法.

下面是具体的实现

```
- (void)ln_addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void *)context {
    // 判断当前类是否有需要监听的属性,并且存在setter
    SEL setterSelector = NSSelectorFromString(setterForGetter(keyPath));
    Method setter = class_getInstanceMethod([self class], setterSelector);
    if (!setter) {
        NSLog(@"当前类没有要监听的该属性,---%@",keyPath);
        return;
    }
    // 如果有setter的话,就要用runtime开始创建一个集成自当前类的派生类了.
    
    // 获取当前类的类名字符串
    Class class = object_getClass(self);
    NSString *selfClassName = NSStringFromClass(class);
    
    // 判断当前class名称是否是派生类的类名
    if (![selfClassName hasPrefix:kLNKVOPrefix]) {
        class = [self makeKVOClassWithClassName:selfClassName];
        object_setClass(self, class);
    }
    
    // 判断当前class(这个class指的是我们自己派生出来的类)是否有setter,如果没有就给他加上setter
    if (![self hasSelector:setterSelector]) {
        const char *types = method_getTypeEncoding(setter);
        class_addMethod(class, setterSelector, (IMP)kvo_setter, types);
    }
    
    // 记录当前的监听者
    [self.observers addObject:observer];
}
```