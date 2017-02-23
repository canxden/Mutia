## Block基础
### 什么是Block
1. Block是一个`数据类型`,  Block类型的变量专门用以`存储一段代码`, 这段代码可以有`返回值`, 也可以有`参数`.
2. 在声明Block变量的时候, 指定了其`参数`和`返回值`, 就`必须存储`这样格式的代码段, 别的格式的代码段是`无法存储`的.

### 如何声明Block类型的变量
- 声明block变量的时候, 要指定这个block变量中存储的代码段的`返回值`和`参数`
- 声明block变量的语法:

```objc
返回值 (^block变量名称)(参数列表);
void (^blockName)(int a, NSString b,...)
```

### 如何初始化Block类型的变量
- 初始化block变量的原理: 写一段符合block变量要求的代码. 把这段代码存储到这个block变量中.
- 书写一个block代码段的语法格式:

```objc
^返回值类型(参数列表){ 代码 }; // 结尾要加;
```

- 如果代码段标识了有返回值类型, 那么在代码段结束之前, 必须要用`return`返回指定类型的值.

```objc
^int(int num1, int num2) { 
    return num1 + num2; 
};
```

> block变量中只能存储和这个block要求相同的代码段

### 如何使用Block类型的变量
- 与调用函数的方式类似

```objc
// 有参数传参数, 有返回值接返回值
返回值类型 returnValue = block变量名(实参);
// 定义block变量
int (^sumBlock)(int, int);

// 初始化block变量
sumBlock = ^int(int num1, int num2) { 
    return num1 + num2; 
};

// 使用block变量
int res = sumBlock(num1, num2);
```

* * *

## Block进阶
### 为什么用block
> block主要流行起来就是因为GCD, 其原因就是, 程序员当前所处的控制器, 并不能有效并且全面的了解当前内核使用情况, 我们需要将我们需要处理的我们需要处理的代码, 发送给另一个有效且全面了解当前内核使用情况的对象去处理代码, 所以其实Block, 直白来说, 就是相当于把自己需要处理的事务, 交给一个能力比自身更适合的对象去处理.

###block的本质
我们都知道程序, 在block外部的零时变量, 不使用__block修饰, 就无法在block中真正修改. 
我们可以看到如下代码. 

```objc
#pragma mark - source code

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int val = 10; // 不使用__block修饰
        void (^block)(void) = ^{
            NSLog(@"%d", val); // 内部使用val
        };
        block();
    }
    return 0;
}

#pragma mark - clang -rewrite-objc

// MARK: 1.Block被转换成一个结构体
struct __main_block_impl_0 {
    struct __block_impl impl; // 实现接口
    struct __main_block_desc_0* Desc; // block描述
    int val; // val变成结构体的一员
  
    // MARK: 2.val变成参数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _val, int flags=0) : val(_val) {
        impl.isa = &_NSConcreteStackBlock;// 指向栈中的block
        impl.Flags = flags;
        impl.FuncPtr = fp; // 函数指针 即__main_block_func_0
        Desc = desc; // 描述结构体 即__main_block_desc_0
    }
    
};

// MARK: 3.Block的执行函数 -> 参数是指向Block结构体的指针
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int val = __cself->val; // 执行过程实际上是已经将复制到结构体中的val设置成了当前val, 并且执行其后代码
  NSLog((NSString *)&__NSConstantStringImpl__val_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_41daf1_mi_0, val);
}

// MARK: 4.Block的描述
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};


int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        int val = 10;
        void (*block)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, val);
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}

```

通过上述代码, 我们可以了解
block中的数据实际上被转换成一个结构体, 
然后对应的block代码段, 实际上就是`__main_block_func_0`里面的代码.

- isa指针：指向表明该block类型的类。

- flags：按bit位表示一些block的附加信息，比如判断block类型、判断block引用计数、判断block是否需要执行辅助函数等。

- reserved：保留变量，我的理解是表示block内部的变量数。

- FuncPtr：函数指针，指向具体的block实现的函数调用地址。

- desc：block的附加描述信息，比如保留变量数、block的大小、进行copy或dispose的辅助函数指针。

- val：因为block有闭包性，所以可以访问block外部的局部变量。这些variables就是复制到结构体中的外部局部变量或变量的地址。

然后执行block调用的函数`__main_block_func_0`, 即此代码是运行block时执行的代码.
我们可以看到取到val的方式是通过取到block数组中的val来获取的, 也就是说, 实际上val已经被block复制了一份, 这里如果我们不使用__block修饰局部变量, 那么局部变量在MRC下未用copy存储在栈区, 用了copy或者在ARC下,存储在堆区

block有几种不同的类型，每种类型都有对应的类，上述中isa指针(impl.isa = &_NSConcreteStackBlock)就是指向这个类。这里列出常见的三种类型：

_NSConcreteGlobalBlock：全局的静态block，不会访问任何外部变量，不会涉及到任何拷贝，比如一个空的block。例如：

```objc
#include int main()
{
    ^{ printf("Hello, World!\n"); } ();
    return 0;
}
```

---
> block内部执行转换

```objc
#import typedef void(^BlockA)(void);
__attribute__((noinline))
void runBlockA(BlockA block) {
    block();
}
void doBlockA() {
    BlockA block = ^{
        // Empty block
    };
    runBlockA(block);
}
```

```objc
#import __attribute__((noinline))
void runBlockA(struct Block_layout *block) {
    block->invoke();
}
void block_invoke(struct Block_layout *block) {
    // Empty block function
}
void doBlockA() {
    struct Block_descriptor descriptor;
    descriptor->reserved = 0;
    descriptor->size = 20;
    descriptor->copy = NULL;
    descriptor->dispose = NULL;
    struct Block_layout block;
    block->isa = _NSConcreteGlobalBlock;
    block->flags = 1342177280;
    block->reserved = 0;
    block->invoke = block_invoke;
    block->descriptor = descriptor;
    runBlockA(&block);
}
```

_NSConcreteStackBlock：保存在栈中的block，当函数返回时被销毁。例如：

```objc
#include int main()
{
    char a = 'A';
    ^{ printf("%c\n",a); } ();
    return 0;
}
```

_NSConcreteMallocBlock：保存在堆中的block，当引用计数为0时被销毁。该类型的block都是由_NSConcreteStackBlock类型的block从栈中复制到堆中形成的。例如下面代码中，在exampleB_addBlockToArray方法中的block还是_NSConcreteStackBlock类型的，在exampleB方法中就被复制到了堆中，成为_NSConcreteMallocBlock类型的block：

```objc
void exampleB_addBlockToArray(NSMutableArray *array) {
    char b = 'B';
    [array addObject:^{
            printf("%c\n", b);
    }];
}
void exampleB() {
    NSMutableArray *array = [NSMutableArray array];
    exampleB_addBlockToArray(array);
    void (^block)() = [array objectAtIndex:0];
    block();
}
```

总结一下：

- _NSConcreteGlobalBlock类型的block要么是空block，要么是不访问任何外部变量的block。它既不在栈中，也不在堆中，我理解为它可能在内存的全局区。代码段

- _NSConcreteStackBlock类型的block有闭包行为，也就是有访问外部变量，并且该block只且只有有一次执行，因为栈中的空间是可重复使用的，所以当栈中的block执行一次之后就被清除出栈了，所以无法多次使用。

- _NSConcreteMallocBlock类型的block有闭包行为，并且该block需要被多次执行。当需要多次执行时，就会把该block从栈中复制到堆中，供以多次执行。

### `__block`和`__weak`

####如果你需要修改局部变量需要添加__block

```objc
 __block int multiplier = 7;
 int (^myBlock)(int) = ^(int num) {
    multiplier ++;//这样就可以了
    return num * multiplier;
};
```

如此一来, multiplier是创建在栈区, 但是block的isa指针已经指向_NSConcreteStackBlock,也就是在堆区开辟了空间, 并且会在操作multiplier的时候已经拷贝一份到栈区, 当我们修改的时候, 实际上是修改了拷贝到堆区的数据

####如果局部变量是数组或者指针的时候只复制这个指针
> 两个指针指向同一个地址,block只修改指针上的内容

```objc
NSMutableArray *mArray = [NSMutableArray arrayWithObjects:@"a",@"b",@"abc",nil];
    NSMutableArray *mArrayCount = [NSMutableArray arrayWithCapacity:1];
    [mArray enumerateObjectsWithOptions:NSEnumerationConcurrent usingBlock: ^(id obj,NSUInteger idx, BOOL *stop){
        [mArrayCount addObject:[NSNumber numberWithInt:[obj length]]];
    }];
   
    NSLog(@"%@",mArrayCount);
```

因为block仅仅是引用了mArray的地址, 并没有修改mArray的地址, 只是读取地址并且修改地址内的空间, 是允许的.

####为什么系统不给block内部修改栈区局部变量
> block执行的时机并不仅仅是在局部变量生命周期的那段作用域中, 因为block也是一个数据类型, 他可以被当做参数传递, 所以执行时, 可能局部变量已经不存在了, 因为栈区的数据, 在过其作用域就被销毁了.

```objc
__weak __typeof(&*self)weakSelf =self; 
//等同于
__weak UIViewController *weakSelf =self;
```

####强引用与__block
如果使用`__block`来修饰, self会被强引用, 因为self指针已经被拷贝一份到堆区空间, 并且retainCount++. 而使用`__weak`只是一个弱引用, 并不会引起强引用

扩展:NSTimer注意避免循环引用的地方,需要找个合适的时机和地方来 invalidate timer
1、如果你通过引用来访问一个实例变量，self会被retain。
2、如果你通过值来访问一个实例变量，那么变量会被retain

> block的copy是在self.block的setter方法实现的?

### 大总结
1. block内部的代码决定了block代码存在的区域
    1. 未涉及局部变量或者外部变量的block, 存储在全局区(执行的代码存储在代码段)
    2. 涉及局部变量或者外部变量的block, MRC下存储在栈区, MAR+copy或ARC下都存储在堆区(连执行的代码都存储在堆区,不存储在代码段)
    
2. block允许读取局部变量, 但是不允许写入局部变量, 指针类型的变量, 修改的仅仅是指针指向的值, 而不是指针的值. 所以可以更改

3. block为什么用copy, 因为在MRC下, 不用copy, 拥有局部变量的代码, 存储在栈区, 当作用域结束, block也因为作用域的原因, 被弹栈了, 也就是被销毁了. 如果使用了copy, 那么block就被copy 到堆区了, 所以MRC下使用copy, 可以解决此问题, ARC下创建出来就在堆区, 不使用copy也行(可能系统内部帮我们copy到了堆区), 所以可以使用strong修饰.

4. `__block`的其实就是将局部变量, 拷贝到了堆区, 所以作用域结束, 实际上block修改的只是堆区的备份.



