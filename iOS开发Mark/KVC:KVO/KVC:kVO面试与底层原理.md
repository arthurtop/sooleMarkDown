# iOS一道复合型面试题与底层原理

## 0. 引言

我们常常吐槽面试的难度，甚至出现了 **“面试造火箭，开发拧螺丝”** 说法。作为客户端开发人员，面试直接让你现场手撸一个红黑树，难度是很大的，除非你专门准备过。

但常见的考点我们是需要知道的。有时考点可能被包装了一下，可能没法一下就看出来，但看破考点之后就会有恍然大悟的感觉。因为本质还是一样的，都是新瓶装旧酒。就像原来的理科考试题，包装一个新的场景，让你解决这个场景下的一个问题，但理论知识都是学过的。

好了，下面废话不多说，进入我们的问题。

## 1. 面试题

### 1.1 题目

我们从热身开始，慢慢深入：

- 面试题1

  现有一个继承于NSObject的实例对象，需要在不直接修改方法实现的情况下，改变一个方法的行为，你会怎么做？

  > 不直接修改方法实现，指的是不直接修改.m文件中方法的内部实现

  这一道题比较简单，其实问的就是 **Runtime** 的 **Method Swizzling** 。可能答出来之后，还会问几个 **Method Swizzling** 相关的深入问题。下面难度升级。

- 面试题2

  问题1，如果使用 **Method Swizzling** 技术，相当于修改了类对象中方法选择器和IMP实现的对应关系。这将导致继承自这个类的所有子类和实例对象都影响，如何控制受影响的范围，或者说如何让方法的行为改变只对这个实例对象生效？

  这个题难度上升了，但是不是有一种脱离生产的感觉，为了面试你而出的一道题？

  我们对这个问题包装一下，让它看起来更接地气，同时问题也再升级一点。

- 面试题3

  现有一个视图，我们需要扩大一下它的响应范围。如果使用 **Method Swizzling** 技术，受影响的范围会比较大。当然，也可以选择继承一个子类来实现。但如果现在实例已经创建了，还是同样的需求，你会如何实现？

  现在问题开始接近生产了。一般来说，修改响应范围涉及到 **响应链和事件传递** 的知识点。

  - 如果可以继承，当然可以选择复写两个方法来解决。
    - **- hitTest:withEvent:**
    - **- pointInside:withEvent:**

  现在限制了**继承并创建子类实例** 的方案，只能选择其他办法。

  - 如果回答 **Method Swizzling** 技术，又涉及到影响范围问题，可能需要加开关、加扩大响应范围记录的变量等，则又涉及到 **关联对象** 相关的问题。

  现在同样也限制了 **Method Swizzling** 方案，还有什么办法呢？

  答案还是 **Runtime** 技术。但这个会涉及到2个 **Runtime** 考点：**消息发送与转发** 以及 **isa-swizzling** 。

  - **消息发送与转发**：主要是 **objc_msgSend** 之后的方法查找流程。如果继续深入问，会到 **消息转发** 相关的考点。
  - **isa-swizzling** ：常见于 **KVO** 原理考点，但其实说到 **isa-swizzling** 肯定会伴随着 **消息发送与转发** 问题。因为修改了`isa`的指向，执行 **objc_msgSend** 时的查找流程会发生变化。

其实，从第1问到第3问，问的核心都是 **isa-swizzling** ，但通过层层包装可能涉及到 **多个知识点**，变成一道复合型面试题。

### 1.2 示例

我们来写一个例子：

```
@interface Person : NSObject
@property (nonatomic, strong, nullable) NSString *firstName;
@property (nonatomic, strong, nullable) NSString *lastName;
@end

@implementation Person
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    person.firstName = @"Tom";
    person.lastName = @"Google";
    
    NSLog(@"person full name: %@ %@", person.firstName, person.lastName);
}
@end
复制代码
```

现在要在创建了`person`实例后，修改`lastName`的返回值，将其固定返回 **Apple** 。

```
@interface Person : NSObject
@property (nonatomic, strong, nullable) NSString *firstName;
@property (nonatomic, strong, nullable) NSString *lastName;
@end

@implementation Person
@end

NSString *demo_getLastName(id self, SEL selector)
{
    return @"Apple";
}

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *person = [[Person alloc] init];
    person.firstName = @"Tom";
    person.lastName = @"Google";
    
    NSLog(@"person full name: %@ %@", person.firstName, person.lastName);
    
    // 1.创建一个子类
    NSString *oldName = NSStringFromClass([person class]);
    NSString *newName = [NSString stringWithFormat:@"Subclass_%@", oldName];
    Class customClass = objc_allocateClassPair([person class], newName.UTF8String, 0);
    objc_registerClassPair(customClass);
    // 2.重写get方法
    SEL sel = @selector(lastName);
    Method method = class_getInstanceMethod([person class], sel);
    const char *type = method_getTypeEncoding(method);
    class_addMethod(customClass, sel, (IMP)demo_getLastName, type);
    // 3.修改修改isa指针(isa swizzling)
    object_setClass(person, customClass);
    
    NSLog(@"person full name: %@ %@", person.firstName, person.lastName);
    
    Person *person2 = [[Person alloc] init];
    person2.firstName = @"Jerry";
    person2.lastName = @"Google";
    NSLog(@"person2 full name: %@ %@", person2.firstName, person2.lastName);
}
@end
// 输出
person full name: Tom Google
person full name: Tom Apple
person2 full name: Jerry Google
复制代码
```

从输出结果可以看到，我们使用 **isa-swizzling** 将`person`对象`lastName`的行为改变了，而`person2`对象没有受到影响。

我们一般知道 **isa-swizzling** 是 **KVO** 的底层原理，但不能只知道拿来做 **KVO** 。

我想通过这个面试题，介绍一种如何在日常开发中使用 **isa-swizzling** 的思路。

下面是 **KVO** 原理，如果你非常自信已经熟悉这部分内容，可以不看了~

**如果觉得这个面试题对你有所帮助，给我点个赞吧~** 👍🏻

![img](https://user-gold-cdn.xitu.io/2020/4/30/171ca690426b6532?imageView2/0/w/1280/h/960/ignore-error/1)

## 2. 由浅入深探索KVO

我们再回到应用这个原理的 **KVO** 上。

### 2.1 KVO应用

给大家再出一道简单的关于KVO日常应用的题。

```
@interface Person : NSObject
@property (nonatomic, strong, nullable) NSString *firstName;
@property (nonatomic, strong, nullable) NSString *lastName;
@property (nonatomic, strong, readonly) NSString *fullName;
@end

@implementation Person
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}
@end
复制代码
```

如何在修改`firstName`或`lastName`时，执行通知`fullName`变化了。如果你的思路是，在`firstName`或`lastName`的set方法中手动调用 **willChangeValueForKey:** 和 **didChangeValueForKey:** ，那么强烈建议阅读此部分。

#### 2.1.1 自动通知

```
// 调用set方法
[account setName:@"Savings"];

// 使用KVC forKey或forKeyPath
[account setValue:@"Savings" forKey:@"name"];
[document setValue:@"Savings" forKeyPath:@"account.name"];

// 使用 mutableArrayValueForKey: 检索关系代理对象
Transaction *newTransaction = <#Create a new transaction for the account#>;
NSMutableArray *transactions = [account mutableArrayValueForKey:@"transactions"];
[transactions addObject:newTransaction];
复制代码
```

**示例**

```
@interface ViewController ()
@property (nonatomic, strong) Person *person;
@property (nonatomic, strong) NSMutableArray<Person *> *people;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    // 非集合
    self.person = [[Person alloc] init];
    [self.person addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
    self.person.name = @"Tom";
    [self.person setValue:@"Jerry" forKey:@"name"];
    [self setValue:@"Tom" forKeyPath:@"person.name"];
    // 集合
    self.people = [NSMutableArray array];
    Person *person0 = [[Person alloc] init];
    person0.name = @"Tom";
    [self.people addObject:person0];
    Person *person1 = [[Person alloc] init];
    person1.name = @"Jerry";
    [self.people addObject:person1];
    NSString *key = @"people";
    [self addObserver:self forKeyPath:key options:NSKeyValueObservingOptionNew context:nil];
    Person *person2 = [[Person alloc] init];
    person2.name = @"Frank";
    NSMutableArray *people = [self mutableArrayValueForKey:key];
    [people addObject:person2];
    NSLog(@"People: \n%@", self.people);
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) {
        NSLog(@"new name: %@", change[NSKeyValueChangeNewKey]);
    } else if ([keyPath isEqualToString:@"people"]) {
        NSLog(@"new array: %@", change[NSKeyValueChangeNewKey]);
        NSArray<Person *> *people = change[NSKeyValueChangeNewKey];
        NSLog(@"new person: %@", people.firstObject.name);
    }
}
@end
// 输出
new name: Tom
new name: Jerry
new name: Tom
new array: (
    "<Person: 0x60000276cc20>"
)
new person: Frank
People: 
(
    "Person name: Tom",
    "Person name: Jerry",
    "Person name: Frank"
)
复制代码
```

#### 2.1.2 手动通知

手动通知提供了更自由的方式去决定什么时间，什么方式去通知观察者。想要使用手动通知必须实现 **automaticallyNotifiesObserversForKey:** (或者 **automaticallyNotifiesObserversOf<Key>** )方法。在一个类中同时使用自动和手动通知是可行的。对于想要手动通知的属性，可以根据它的keyPath返回NO，而其对于其他位置的keyPath，要返回父类的这个方法。

```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO;
    } else {
        return [super automaticallyNotifiesObserversForKey:key];
    }
}
// 或者
+ (BOOL)automaticallyNotifiesObserversOfName {
    return NO;
}
复制代码
```

##### 一对一关系

```
- (void)setOpeningBalance:(double)theBalance {
     if (theBalance != _openingBalance) {
        [self willChangeValueForKey:@"openingBalance"];
        _openingBalance = theBalance;
        [self didChangeValueForKey:@"openingBalance"];
     }
}
复制代码
```

如果一个操作会导致多个属性改变，需要嵌套通知：

```
- (void)setOpeningBalance:(double)theBalance {
     [self willChangeValueForKey:@"openingBalance"];
     [self willChangeValueForKey:@"itemChanged"];
     _openingBalance = theBalance;
     _itemChanged = _itemChanged + 1;
     [self didChangeValueForKey:@"itemChanged"];
     [self didChangeValueForKey:@"openingBalance"];
}
复制代码
```

##### 一对多的关系

必须注意不仅仅是这个key改变了，还有它改变的类型以及索引。

```
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
     [self willChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"transactions"];
     // Remove the transaction objects at the specified indexes.
     [self didChange:NSKeyValueChangeRemoval valuesAtIndexes:indexes forKey:@"transactions"];
}
复制代码
```

#### 2.1.3 键之间的依赖

在很多种情况下一个属性的值依赖于在其他对象中的属性。如果一个依赖属性的值改变了，这个属性也需要被通知到。

##### 一对一关系

```
@interface Person : NSObject
@property (nonatomic, strong, nullable) NSString *firstName;
@property (nonatomic, strong, nullable) NSString *lastName;
@property (nonatomic, strong, readonly) NSString *fullName;
@end
复制代码
```

可以重写 **keyPathsForValuesAffectingValueForKey:** 方法。也可以通过实现 **keyPathsForValuesAffecting<Key>** 方法来达到前面同样的效果，这里的 **<Key>** 就是属性名，不过第一个字母要大写。

```
@implementation Person
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
    if ([key isEqualToString:@"fullName"]) {
        NSArray *affectingKeys = @[@"lastName", @"firstName"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
// 或者
+ (NSSet *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"lastName", @"firstName", nil];
}
@end
复制代码
```

##### 一对多关系

**keyPathsForValuesAffectingValueForKey:** 方法不能支持一对多关系。

举个例子，比如你有一个`Department`对象，和很多个`Employee`对象。而`Employee`有一个`salary`属性。你可能希望`Department`对象有一个`totalSalary`的属性，依赖于所有的`Employee`的`salary` 。

注册`Department`成为所有`Employee`的观察者。当`Employee`被添加或者被移除时进行计算。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if (context == totalSalaryContext) {
        [self setTotalSalary:[self valueForKeyPath:@"employees.@sum.salary"]];
    }
    else
    // deal with other observations and/or invoke super...
}
 
- (void)setTotalSalary:(NSNumber *)newTotalSalary {
    if (totalSalary != newTotalSalary) {
        [self willChangeValueForKey:@"totalSalary"];
        _totalSalary = newTotalSalary;
        [self didChangeValueForKey:@"totalSalary"];
    }
}
 
- (NSNumber *)totalSalary {
    return _totalSalary;
}
复制代码
```

### 2.2 实现细节

#### 2.2.1 isa-swizzling

KVO的实现用了一种叫 **isa-swizzling** 的技术。

当一个对象的一个属性注册了观察者后，被观察对象的`isa`指针的就指向了一个系统为我们生成的中间类，而不是我们自己创建的类。在这个类中，系统为我们重写了被观察属性的**setter**方法。

通过 **object_getClass(id obj)** 方法可以获得实例对象真实的类(`isa`指针的指向)。

```
@interface Person : NSObject
@property (nonatomic, strong, nullable) NSString *name;
@end
@implementation Person
@end
  
@interface ViewController ()
@property (nonatomic, strong) Person *p1;
@property (nonatomic, strong) Person *p2;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    self.p1 = [[Person alloc] init];
    self.p2 = [[Person alloc] init];  
    self.p1.name = @"Tom";
    
  	NSLog(@"before kvo --- p2: %s", object_getClassName(self.p2));
    [self.p2 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
    NSLog(@"after  kvo --- p2: %s", object_getClassName(self.p2));
    
    self.p2.name = @"Jerry";
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) {
        NSLog(@"new name: %@", change[NSKeyValueChangeNewKey]);
    }
}
@end
// 输出
before kvo --- p2: Person
after  kvo --- p2: NSKVONotifying_Person
new name: Jerry
复制代码
```

我们在`p2`实例对象被键值观察的前后打印其`isa`指针(实际使用的类)。

从结果中我们可以看到`isa`指针指向了一个中间类`NSKVONotifying_Person`。

苹果的KVO中间类的命名规则是在类名前添加`NSKVONotifying_`，如果我们的类叫`Son`KVO之后的中间类为`NSKVONotifying_Son`。

#### 2.2.2 IMP

我们再看一下KVO前后的函数方法的地址是否一样。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    self.p1 = [[Person alloc] init];
    self.p2 = [[Person alloc] init];
    self.p1.name = @"Tom";
    
    NSLog(@"before kvo --- p1: %p p2: %p", [self.p1 methodForSelector:@selector(setName:)], [self.p2 methodForSelector:@selector(setName:)]);
    [self.p2 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
    NSLog(@" after  kvo --- p1: %p p2: %p", [self.p1 methodForSelector:@selector(setName:)], [self.p2 methodForSelector:@selector(setName:)]);
    
    self.p2.name = @"Jerry";
}
// 输出
before kvo --- p1: 0x10ccee670 p2: 0x10ccee670
after  kvo --- p1: 0x10ccee670 p2: 0x7fff258e454b
复制代码
```

我们看到监听之间两个实例对象的 **setName:** 方法的函数地址相同，KVO之后`p2`实例对象的 **setName:** 方法地址变了。

我们可以查看一下这个方法地址：

```
(lldb) image lookup -a 0x7fff258e454b
      Address: Foundation[0x000000000006954b] (Foundation.__TEXT.__text + 422667)
      Summary: Foundation`_NSSetObjectValueAndNotify
复制代码
```

这个是`Foundation`框架中的一个私有方法 **_NSSetObjectValueAndNotify** 。



![Foundation __NSSetObjectValueAndNotify](https://user-gold-cdn.xitu.io/2020/7/10/173372b8804089b5?imageView2/0/w/1280/h/960/ignore-error/1)



可以看到 **_NSSetObjectValueAndNotify** 还是调用了 **willChangeValueForKey:** 和 **didChangeValueForKey:** 来进行手动通知的。

### 2.3 自定义KVO

下面我们根据KVO的实现细节，仿写一个 **非常简化版** 的KVO。

```
NSString *ObserverKey = @"SetterMethodKey";
// 根据方法名获取Key
NSString *getKeyForSetter(NSString *setter) {
    NSRange range = NSMakeRange(3, setter.length - 4);
    NSString *key = [setter substringWithRange:range];
    NSString *letter = [[key substringToIndex:1] lowercaseString];
    key = [key stringByReplacingCharactersInRange:NSMakeRange(0, 1) withString:letter];
    return key;
}
// 实现一个setter和通知函数
void _MySetObjectValueAndNotify(id self, SEL selector, NSString *name) {
    // 1.调用父类的方法
    struct objc_super superClass = {
        self,
        class_getSuperclass([self class])
    };
    objc_msgSendSuper(&superClass, selector, name);
    // 2.通知观察者
    NSObject *observer = objc_getAssociatedObject(self, &ObserverKey);
    NSString *selectorName = NSStringFromSelector(selector);
    NSString *key = getKeyForSetter(selectorName);
    objc_msgSend(observer, @selector(observeValueForKeyPath:ofObject:change:context:), key, self, @{NSKeyValueChangeNewKey: name}, nil);
}

@implementation Person
- (void)snx_addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void *)context {
    // 1.创建一个子类
    NSString *oldName = NSStringFromClass([self class]);
    NSString *newName = [NSString stringWithFormat:@"CustomKVO_%@", oldName];
    Class customClass = objc_allocateClassPair([self class], newName.UTF8String, 0);
    objc_registerClassPair(customClass);
    // 2.修改修改isa指针
    object_setClass(self, customClass);
    // 3.重写set方法
    NSString *selectorName = [NSString stringWithFormat:@"set%@:", keyPath.capitalizedString];
    SEL sel = NSSelectorFromString(selectorName);
    class_addMethod(customClass, sel, (IMP)_MySetObjectValueAndNotify, "v@:@");
    // 4.绑定观察者
    objc_setAssociatedObject(self, &ObserverKey, observer, OBJC_ASSOCIATION_ASSIGN);
}
@end
复制代码
```

> **重要**
>
> 使用**objc_msgSendSuper**时，可能编译器会报错：
>
> **Too many arguments to function call, expected 0, have 3**
>
> 解决办法：在**Build Setting**修改**Enable Strict Checking of objc_msgSend Calls**为**No**。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    self.p1 = [[Person alloc] init];
    self.p2 = [[Person alloc] init];
    self.p1.name = @"Tom";
    
    NSLog(@"before kvo --- p2: %s", object_getClassName(self.p2));
    NSLog(@"before kvo --- p1: %p p2: %p", [self.p1 methodForSelector:@selector(setName:)], [self.p2 methodForSelector:@selector(setName:)]);
//    [self.p2 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
    [self.p2 snx_addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
    NSLog(@"after  kvo --- p2: %s", object_getClassName(self.p2));
    NSLog(@"after  kvo --- p1: %p p2: %p", [self.p1 methodForSelector:@selector(setName:)], [self.p2 methodForSelector:@selector(setName:)]);
    
    self.p2.name = @"Jerry";
}
// 输出
before kvo --- p2: Person
before kvo --- p1: 0x103514460 p2: 0x103514460
after  kvo --- p2: CustomKVO_Person
after  kvo --- p1: 0x103514460 p2: 0x103513f90
new name: Jerry
```