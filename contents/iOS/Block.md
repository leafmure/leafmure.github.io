---
title: Block 探究
date: 2018-11-28 14:37:57
categories: iOS
tags:
- block
keywords: block,block变量,__block,循环引用,block的实现,block的本质
description:
---
> Block是OSX Snow Leopard和iOS 4引入的C语言扩充功能，是一个带有自动变量（局部变量）的匿名函数，也被称为闭包。


### Block语法

#### Block的表达式语法
```Objective-C
^ 返回值类型 参数列表 表达式
^ int (int count) { return count + 1; }

// 可省略返回值类型，如果表达式中有返回值就使用返回值的类型
^(int count) { return count + 1; }

// 如果没有参数，参数列表也可以省略
^{ printf("hello Blocks\n"); }

```
<!-- more -->
#### Block 类型变量
Block类型变量与一般的C语言变量完全相同，可作为：自动变量、函数参数、静态变量、静态全局变量、全局变量。

声明Block类型变量的方式
- 使用Block语法将Block赋值为Block类型变量
```Objective-C
返回值 (^变量名)(参数列表) = ^参数列表 表达式
int (^block)(int) = ^(int count) { return count + 1; }
```
- 使用typedef来声明Block类型变量
```Objective-C
// 定义Block类型
typedef 返回值 (^变量名)(参数列表)
typedef int (^block_t)(int count);
// 声明block_t类型的Block变量
block_t block;
```

### Block的实现与本质

Blcok语法实际上是作为极普通的C语言源代码来处理的，可以在shell中通过clang命令将OC文件转换为源码文件。
```shell
clang -rewrite-objc 文件名
```
将以下Block语法转换为源代码形式
```Objective-C
int main(int argc, const char * argv[]) {
@autoreleasepool {

void (^block)(void) = ^{
printf("hello block!");
};

block();
}
return 0;
}
```
源码为：
```Objective-C
// Block的结构体实例的信息
struct __block_impl {
void *isa;
int Flags;
int Reserved;
void *FuncPtr;
};

// Block的结构体实例
struct __main_block_impl_0 {
// 结构体实例成员变量信息：结构体实例类型、函数等
struct __block_impl impl;
// 结构体实例信息：区域和大小
struct __main_block_desc_0* Desc;

// __main_block_impl_0结构体的构造方法
// 第一个参数 __main_block_func_0 是block变量的表达式部分转换的C语言函数指针
// 
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
// 实例类型名
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
// block的表达式函数
impl.FuncPtr = fp;
Desc = desc;
}
};

/*
* __main_block_func_0函数 是的Block实例的表达式部分转换的C语言函数
* __cself是指向Blcok对象自身的变量，就像Objective-C中的self
**/
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

printf("hello block!");
}

// 其描述的是 __main_block_impl_0结构体实例的区域和大小
static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

// main.m文件
int main(int argc, const char * argv[]) {
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
return 0;
}
```
跟着代码执行流程走，首先执行
```Objective-C
void (^block)(void) = ^{
printf("hello block!");
};
```
下面源代码将在栈上__main_block_impl_0 结构体实例的指针赋值给 __main_block_impl_0 结构体指针类型的变量block。
```Objective-C
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

// 去掉类型转换
struct __main_block_impl_0 tmp = __main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
struct __main_block_impl_0 *block = &tmp
```
__main_block_impl_0 构造函数的第一个参数 __main_block_func_0 是block变量的表达式部分转换的C语言函数指针，第二个参数 __main_block_desc_0_DATA中存储着 __main_block_impl_0 结构体实例大小。

声明好了block变量，就到了下一步，使用block：block()
```Objective-C
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

// 去掉类型转换
(*block->FuncPtr)(block);
```
此步骤就是简单的使用函数指针调用函数，所调用的函数就是上面源码的 __main_block_func_0 函数。

总结：由上可知Block实质就是Objective-C的对象。

### Block捕获自动变量
```Objective-C
int main () {
int val = 10;
void (^block)(void) = ^{
printf("val = %d", val);
};

val = 1;

block();
}
```
上面的示例的结果是：val = 10，由于Block捕获的是val的值，所以外部val变量的改动并不会影响到Block中的val变量，但是，如果我们试图在Block中改变val的值，由于Blcok捕获的是val变量的值而不是地址，所以在Block中无法修改自动变量val的值，因此编译器会报错，这一点在下节的“Block 是如何捕获自动变量”可明白。
```Objective-C
int val = 10;
void (^block)(void) = ^{ val = 1; };
```
如果我们只使值的话就不会有任何问题
```Objective-C
int val = 10;
void (^block)(void) = ^{ printf("val = %d", val); };

id multArray = [NSMutableArray alloc] init];
void (^block)(void) = ^{ 
[multArray addObject:@"use is ok"];
}
```
当然也有特殊情况，当我们在Block中只使用C语言的字符串字面变量数组时，由于Block捕获自动变量的方法没有实现对C语言数组的截获，所以在Block中使用C语言的字符串字面变量数组会出错。
```Objective-C
const char text[] = "hello";
void (^block)(void) = ^{
printf("%c\n",text[2]);
}

// 该用此做法就能解决问题
const char text = "hello";
void (^block)(void) = ^{
printf("%c\n",text[2]);
}
```

### Block 是如何捕获自动变量
将捕获自动变量的代码转换为源代码
```Objective-C
int main(int argc, const char * argv[]) {
@autoreleasepool {

int val = 10;
void (^block)(void) = ^{
printf("val = %d", val);
};

block();
}
return 0;
}
```
源码：
```Objective-C
struct __main_block_impl_0 {

struct __block_impl impl;
struct __main_block_desc_0* Desc;
int val;

__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _val, int flags=0) : val(_val) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

int val = __cself->val; // bound by copy

printf("val = %d", val);

}

static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, const char * argv[]) {
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

int val = 10;
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, val));

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
return 0;
}
```
从上面的代码可知道，Block将在表达式中用到的val自动变量作为成员变量追加到 __main_block_impl_0 结构体中
```Objective-C
struct __main_block_impl_0 {

struct __block_impl impl;
struct __main_block_desc_0* Desc;
int val;
};
```
跟着程序执行流程
```Objective-C
int val = 10;
void (^block)(void) = ^{
printf("val = %d", val);
};

// 源码：
int val = 10;
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, val));
```
定义 val变量并赋予初始值10，将val值通过 __main_block_impl_0 的构造方法对__main_block_impl_0追加的val成员变量进行初始化。
```Objective-C
// __main_block_impl_0 初始化
impl.isa = &_NSConcreteStackBlock;
impl.Flags = 0;
impl.FuncPtr = __main_block_func_0;
Desc = __main_block_desc_0_DATA;
val = 10;
```

使用block：block()，也就是执行
```Objective-C
^{ printf("val = %d", val); }

源码：
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

int val = __cself->val; // bound by copy

printf("val = %d", val);

}
```
在执行block的表达式时，用的val自动变量，是结构体追加的val自动变量，所以Block捕获自动变量并不是真的将变量直接作为成员变量，而是在结构体中追加一个名字相同的成员变量，在构造方法中将要捕获的自动变量值对追加的成员变量初始化。

### __block 说明符
在此之前试图在block表达式中改变捕获的自动变量的值时编译器会报错，解决方案有：
- 用__block 说明符修饰自动变量
- 静态变量
- 静态全局变量
- 全局变量
```Objective-C
int global_val = 1;
static int static_global_val = 2;
int main(int argc, const char * argv[]) {
@autoreleasepool {

static int static_val = 3;
__block int block_val = 4;
void (^block)(void) = ^{

global_val = 2;
static_global_val = 3;
static_val = 4;
block_val = 5;
};

block();

printf("global_val: %d \n", global_val);
printf("static_global_val: %d \n", static_global_val);
printf("static_val: %d \n", static_val);
printf("block_val: %d \n", block_val);
}
return 0;
}

// 结果：
global_val: 2 
static_global_val: 3 
static_val: 4 
block_val: 5 
```
转换的源码为：
```Objective-C
int global_val = 1;
static int static_global_val = 2;
struct __Block_byref_block_val_0 {
void *__isa;
__Block_byref_block_val_0 *__forwarding;
int __flags;
int __size;
int block_val;
};

struct __main_block_impl_0 {
struct __block_impl impl;
struct __main_block_desc_0* Desc;
int *static_val;
__Block_byref_block_val_0 *block_val; // by ref

__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_val, __Block_byref_block_val_0 *_block_val, int flags=0) : static_val(_static_val), block_val(_block_val->__forwarding) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

__Block_byref_block_val_0 *block_val = __cself->block_val; // bound by ref
int *static_val = __cself->static_val; // bound by copy

global_val = 2;
static_global_val = 3;
(*static_val) = 4;
(block_val->__forwarding->block_val) = 5;
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->block_val, (void*)src->block_val, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->block_val, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

static int static_val = 3;
__attribute__((__blocks__(byref))) __Block_byref_block_val_0 block_val = {(void*)0,(__Block_byref_block_val_0 *)&block_val, 0, sizeof(__Block_byref_block_val_0), 4};
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_val, (__Block_byref_block_val_0 *)&block_val, 570425344));

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

printf("global_val: %d \n", global_val);
printf("static_global_val: %d \n", static_global_val);
printf("static_val: %d \n", static_val);
printf("block_val: %d \n", (block_val.__forwarding->block_val));
}
return 0;
}
```
以下是对四种变量改变值的表达式函数
```Objective-C
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

__Block_byref_block_val_0 *block_val = __cself->block_val; // bound by ref
int *static_val = __cself->static_val; // bound by copy

global_val = 2;
static_global_val = 3;
(*static_val) = 4;
(block_val->__forwarding->block_val) = 5;
}
```
Block仅捕获了 block_val修饰的变量和staic_val，全局变量没有捕获，原因是block_val、staic_val作为局部变量，其作用域仅在 main 方法中，当离开作用域时变量会被废弃，所以Block需捕获block_val、staic_val变量以保证在表达式函数的使用，然而全局变量的作用域很广，所以Block无需捕获，所以看出Block只对需要捕获的变量进行捕获。

对于全局变量能在Block中修改值，如上面说的，它作用域很广，所以在Block表达式函数结束后也能保存修改的值。静态变量static_val是将其指针传递给 __main_block_impl_0 结构体的构造函数并保存，也就是说Block捕获的并不是 static_val的值，而是其指针（即内存地址），所以static_val能够在超出作用域之外使用，在Block结束后也能保存修改后的值。

最后就是block_val变量了，block_val 是使用__block 说明符修饰的变量，“__block 说明符”也被称为“__block 存储域类说明符”，block_val在添加上“__block 说明符”后，源码变换如下
```Objective-C
__block int block_val = 4;

// 转换后的源码：
__attribute__((__blocks__(byref))) __Block_byref_block_val_0 block_val = {(void*)0,(__Block_byref_block_val_0 *)&block_val, 0, sizeof(__Block_byref_block_val_0), 4};
//去掉类型转换后：
__Block_byref_block_val_0 block_val = { 0, &block_val, 0, sizeof(__Block_byref_block_val_0), 4};

// __Block_byref_block_val_0 结构体
struct __Block_byref_block_val_0 {
void *__isa;
__Block_byref_block_val_0 *__forwarding;
int __flags;
int __size;
int block_val;
};
```
在添加上“__block 说明符”后，block_val变成了 __Block_byref_block_val_0 结构体类型的自动变量，并且其结构体中含有一个相当于原自动变量block_val的成员变量，当对block_val变量赋值时
```Objective-C
__Block_byref_block_val_0 *block_val = __cself->block_val; 

(block_val->__forwarding->block_val) = 5;
```
Blcok的 __main_block_impl_0 结构体实例持有指向 block_val变量的__Block_byref_block_val_0 结构体实例的指针，而__Block_byref_block_val_0 结构体实例的成员变量 __forwarding 持有永远指向自身的指针，可以通过 __forwarding来访问成员变量 block_val，因此在Block结束后，block_val变量能保存改动后的值。这里还有一个疑问：block_val超出了block变量的作用域并且它是配置在栈上，为什么还能访问？这个问题在下节“Block变量和__block变量存储域”可以明白。

总结：要在Blcok中改变被捕获的自动变量的值的方式有：
- 以指针（内存地址）形式被Block捕获，Block保存其指针后，对于内容的更改便不会丢失
- 改变自动变量的存储方式，如__block， __Block_byref_block_val_0结构体实例中拥有相当于原自动变量的成员变量并拥有永远指向自己的指针的成员变量__forwarding，不过变量在栈上还是堆上都能访问。


### Block变量和__block变量存储域
上面所述可知，Block类型变量转换为 __main_block_impl_0 结构体类型变量，__block 变量转换为 __Block_byref_block_val_0结构体类型变量，所谓的结构体类型变量即栈上生成的结构体实例。

Block实质就是Objective-C对象，它有三种类型：_NSConcreteStackBlock、_NSConcreteGlobalBlock、_NSConcreteMallocBlock。
- _NSConcreteStackBlock：
它是在栈上的Block类型，只用到外部局部变量、成员属性变量，没有强指针引用，一旦脱离作用域时就会被销毁。
- _NSConcreteGlobalBlock：
它是在数据区域（.data区）的Block类型，只用到没有用到外界变量或只用到全局变量的block，只有在应用程序结束才会被销毁。
- _NSConcreteMallocBlock
它是在堆（内存块）里的Block类型，有强指针引用或copy修饰的成员属性引用，没有强指针引用即销毁。

![image](https://destinmure.github.io/pic/postImage/Block/psb.png)

_NSConcreteGlobalBlock 类型Block变量在超出作用域也能通过指针访问，而_NSConcreteStackBlock类型Block在作用域结束后就会被废弃，同样的 __block 类型变量也是如此，为解决这个问题，Block语法提供了将Block 和 __block变量从栈上复制到堆上，这样在Block变量作用域结束后，堆上的Block还能继续存在。复制在堆上的Block会将 _NSConcreteMallocBlock 类名写入Block 结构体实例 __main_block_impl_0 中的成员变量 ipml 的isa变量中。
```Objective-C
ipml.isa = &_NSConcreteMallocBlock;
```
而 __block 结构体实例的成员变量 __forwarding 拥有永远指向自身的指针， 可以实现无论 __block 变量配置在栈上还是堆上也可以访问 __block变量，因此即使__block变量在Block变量的作用域之外也能访问__block变量并保存更改后的值。

### Block 的copy方法和dispose方法
这是在MRC环境下的代码示例
```Objective-C
typedef void (^block_t)(id);
int main(int argc, const char * argv[]) {
@autoreleasepool {

id array = [[NSMutableArray alloc] init];
block_t block = [^(id obj) {

[array addObject:obj];
NSLog(@"array count = %ld", [array count]);
} copy];

block([[NSObject alloc] init]);
block([[NSObject alloc] init]);
block([[NSObject alloc] init]);
}
return 0;
}

// clang后的源码：
typedef void (*block_t)(id);

struct __main_block_impl_0 {
struct __block_impl impl;
struct __main_block_desc_0* Desc;
id array;
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj) {

id array = __cself->array; // bound by copy

((void (*)(id, SEL, ObjectType _Nonnull))(void *)objc_msgSend)((id)array, sel_registerName("addObject:"), (id)obj);
NSLog((NSString *)&__NSConstantStringImpl__var_folders_2k_lvtjwq3x1h746y288_mm11200000gn_T_main_580dfd_mi_0, ((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array, sel_registerName("count")));

}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

id array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));

block_t block = (block_t)((id (*)(id, SEL))(void *)objc_msgSend)((id)((void (*)(id))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, array, 570425344)), sel_registerName("copy"));

((void (*)(__block_impl *, id))((__block_impl *)block)->FuncPtr)((__block_impl *)block, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));
((void (*)(__block_impl *, id))((__block_impl *)block)->FuncPtr)((__block_impl *)block, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));
((void (*)(__block_impl *, id))((__block_impl *)block)->FuncPtr)((__block_impl *)block, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init")));

}
return 0;
}
```
上面代码中 Block捕获了一个 __strong 类型的变量 array，虽然源码中没有标识。在Objective-C中，C语言结构体不能含有 __strong 类型的变量，因为在Block从栈上复制到堆以及堆上的Block废弃时，编译器不知道何时进行C语言结构体的初始化和废弃操作，但是由于 Objective-C的运行库能够准确把握Block从栈上复制到堆以及堆上的Block废弃的时机，因此在 __main_block_desc_0增加了 copy、dispose成员变量并赋予__main_block_copy_0、__main_block_dispose_0函数，通过这个两个函数管理此类变量的复制和废弃。
```Objective-C
static struct __main_block_desc_0 {
size_t reserved;
size_t Block_size;
void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

// BLOCK_FIELD_IS_OBJECT 和 BLOCK_FIELD_IS_BYREY 用于分辨copy和dispose函数的对象类型是对象还是 __block变量。
// 在Block 从栈上复制到堆上时，会调用__main_block_copy_0 将 __strong 类型变量跟随Block复制到堆上
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

// 当堆上Block 废弃时，__main_block_dispose_0 将 __strong 类型变量废弃
static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}
```

在ARC下，编译器会适当地进行判断去自动执行copy，而在MRC下需要自己手动copy，看下面在ARC环境下的一个返回Block的函数：
```Objective-C
typedef int (^blk_t)(int);
blk_t func(int rate) {

return ^(int count) { return rate *count; };
};

ARC编译器转换简约后：
blk_t func(int race) {

blk_t tmp = &__func_block_impl_0(__func_block_func0, &__func_block_desc_0_DATA, rate);

tmp = objc_retainBlock(tmp);

return objc_autoreleaseReturnValue(tmp);
}

```
第一步，通过Block结构体构造函数生成配置在栈上的结构体实例并将其赋给Block 变量 tmp。第二步，objc_retainBlock()函数实质是_Block_copy函数（由objc4 运行库可知），将栈上的Block变量tmp复制到堆上。第三步，将堆上的Block变量tmp作为Objective-C对象注册到自动释放池中，然后返回改对象。虽说ARC下，编译器会适当地进行判断去自动生成“将Block 变量从栈上复制到堆上”的代码，但是在此之外的情况，我们需要自己手动copy，比如 alloc/new/copy/mutableCopy等方法其实都是将对象从栈上复制到堆上的操作。在ARC中有以下情况需要手动copy：
- 向方法或者函数的参数中传递Block之前时，当然如果方法或或者函数中有对Block进行复制的操作，那便不需要。
- NSArray类的 initWithObjects 实例方法

在ARC中有以下情况不需要手动copy：
- Block作为函数值返回时
- 将 Block赋值给 使用 __strong 修饰符的id类型或 Block类型变量
- Coccoa 框架的方法且方法名中含有 usingBlock 中使用时。
- Grand Central Dispatch 的API中

对于不同类型的Block进行copy操作后的情况
```Objective-C
Block 的类                   副本源的配置存储域                 复制结果
_NSConcreteStackBlock        栈                            从栈复制到堆
_NSConcreteGlobalBlock       程序的数据区域                   什么也不做
_NSConcreteMallocBlock       堆                            引用计数增加
```
不管Block配置在何处，用copy方法都不会引起任何问题，因此在不确定的时候可以调用copy方法，但是需注意从栈上复制到堆上十分消耗CPU资源。

以上只对Block进行了说明，那么对于__block 变量又是怎么处理的？

当Block从栈上复制到堆上时，其拥有的所有__block 变量也会随Block全部被复制到堆上。栈上的__block变量结构体中的成员变量 __forwarding的值会被替换成堆上的__block变量结构体的地址，堆上的__block变量结构体中的成员变量 __forwarding依然指向自身，因此，不管是在栈上我们都能通过__forwarding 访问同一个__block 变量。

![image](https://destinmure.github.io/pic/postImage/Block/psb-1.png)

### Block 循环引用
#### Block 是如何引起循环引用的
```Objective-C

#import <Foundation/Foundation.h>

typedef void (^printBlock_t)(void);

@interface TestCircleRetain : NSObject {
printBlock_t printBlock;
}
@property (nonatomic,weak) NSString *name;
- (void)test;

@end

#import "TestCircleRetain.h"

@implementation TestCircleRetain

- (instancetype)init {
self = [super init];
if (self) {
self.name = @"sss";

printBlock = ^{
NSLog(@"print: %@", _name);
};
}
return self;
}


- (void)test {

printBlock();
}

- (void)dealloc {

NSLog(@"TestCircleRetain dealloc");
}

int main(int argc, const char * argv[]) {
@autoreleasepool {

TestCircleRetain *testCircleRetain = [TestCircleRetain alloc] init];
}
return 0;
}


@end

```
运行后可发现TestCircleRetain实例类的dealloc方法没有调用，因为在TestCircleRetain实例方法中，Block里使用了 带有 __strong（强引用） 修饰的testCircleRetain（也就是self）对象的成员变量 name，所以Block会捕获testCircleRetain（而不是只捕获name，即使你用的是_name,和self.name并无差别），并且当Block赋值给成员变量printBlock时，Block由栈上复制到了堆上 ，因此 testCircleRetain 持有 printBlock，printBlock 持有 testCircleRetain，双方互相持有（强引用），没法销毁，故而没法执行dealloc()方法。不过上面的循环引用比较明显，编译器会发现并警告。

![image](https://destinmure.github.io/pic/postImage/Block/psb-2.png)

#### 如何避免循环引用

- 使用 __weak（弱引用）修饰的变量

上面的例子中，Block捕获了带有 __strong修饰的testCircleRetain对象，导致testCircleRetain持有的printBlock变量强引用了testCircleRetain自身引起循环引用，我们可以使用 __weak 修饰的变量并将testCircleRetain（即self）赋值使用来避免循环引用。
```Objective-C
__weak TestCircleRetain *weakSelf = self;
printBlock = ^{
NSLog(@"print: %@", weakSelf.name);
};
```

![image](https://destinmure.github.io/pic/postImage/Block/psb-3.png)

- 使用__block 变量来避免循环引用
```Objective-C
__block block_self = self;
printBlock = ^{
NSLog(@"print: %@", block_self.name);
block_self = nil;
};
```
使用__block修饰的变量并赋值self（testCircleRetain自身），它们之间的引用便变成了：testCircleRetain 引用了 printBlock，block_self 引用了 testCircleRetain和 printBlock。当printBlock执行后 循环引用便会打破，但是如果你不使用 printBlock的话，便会持续循环引用从而导致内存泄漏。

![image](https://destinmure.github.io/pic/postImage/Block/psb-4.png)
