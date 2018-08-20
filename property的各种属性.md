# property的各种属性
## @Property (xxx, xxx) class objec; // 一般性的属性定义目前都这么写
### atomic/nonatomic:
* **atomic**: 操作是原子的,意味着仅仅有一个线程访问实例变量的时候.atomic是线程安全的,至少在存取器上是安全的.他是一个默认的特性,现在很少直接使用,因为效率较低.这跟ARM平台和内补锁机制有关.
* **nonatomic**: 跟atomic恰恰相反,意味操作是非原子性,可以允许多个线程访问.但是不能保证在多线程环境下的安全性,在明知道仅有一个线程或者单线程环境下被广泛使用.

### strong/weak:
* **strong**: 对引用对象保持强引用,即拥有该对象的全部所有权.在setter中对对象引用计数+1:

```
-(void)setName:(NSString*)_name{  
     //首先推断是否与旧对象一致，假设不一致进行赋值。  
     //由于假设是一个对象的话，进行if内的代码会造成一个极端的情况：当此_name的retain为1时。使此次的set操作让实例name提前释放。而达不到赋值目的。  
     if ( _name != name){  
          [_name release];  
          _name = [name retain];  
     }
}
```
* **weak**: 对引用对象的引用计数不做+1操作,只是简单的赋值.

```
-(void)setName:(NSString*)name{
     _name = name;
}
```
举个🌰:
> 把引用的对象想象成一条狗,他要跑掉(**dealloc**),但是strong修饰的主人是手里切切实实拿了一条狗链子拴着这条狗的.只有当所有的狗链子都解开的时候这条狗才能跑掉,而weak修改的主人在其实是没有拿到狗链的,在其他有狗链的主人还有最后一条链子没有断开之前weak修饰的主人向狗发送"***停下***"的命令时,狗还是会听,但是当所有的狗链都断开的时候,weak修饰的主人将不能再对这条狗发送指令

### readonly:
当我们要为一个类声明一个只读属性的时候,需要使用readonly来对属性进行修饰;在.h文件中我们需要做如下定义.

```@property(nonatomic, readonly) NSString *name;```

但是仅仅是这样的定义还不够,我们还需要在.m中使用:
```@synthesize name = _name;```帮助我们自动生成setter和getter