# ARC 和 MRC
### 一个正确的setter方法
```
- (void)setName:(NSString *)name {
    if (_name != name) {
        [_name release];
        _name = [name retain];
    }
}
```
苹果引入了引用计数来进行内存管理,在mrc环境下,谁调用,谁就要给引用计数加1,当不使用该对象的时候就要release操作,对引用计数减1.