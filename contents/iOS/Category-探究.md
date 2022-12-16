---
title: Category 探究
date: 2018-12-21 14:37:21
categories: iOS
tags:
- category
keywords: category,分类,实例变量,+ load,关联对象,runtime
description:
images: "/postCover/Category-探究.png"
---

### 简介
人类在进步，社会在发展，随着时间变化我们会遇到不同的问题，由此也诞生了对应的解决方法。我们编写的类也是如此，在随着业务的发展，原定类的方法也就不足以处理新的业务，因此我们开始想方设法去扩展类以处理新业务。在 Objective-C 2.0 提供的 category 特性，此特性可为已有类添加新的方法。除了为已有类添加新方法外，我们可以运用分类的特性去抽离出复杂类的业务，降低类的复杂度。
<!-- more -->
### 与 category 相似的 extension
extension(类扩展) 从代码看起来和 category 十分相像，代码如下:
```Objective-C
// extension
@interface Father ()
@property (nonatomic,strong) NSString *name;
- (void)say;

@end

// category
@interface Father (skill)
- (void)fastTalk;

@end
```
虽然代码看上去只有括号中有名称的不同，但其实它们是完全不同的，它们的不同之处如下：
- 在编译上，extension 在编译期时被确定，而 category 是在运行期被确定的。
- extension 一般是用来隐藏类的私有信息，而且你必须在类的源码文件处才能添加。而 category 不同，只要类存在，就能为其添加 category。
- category 不能添加实例变量，这是它与 extension 最大的区别。（category 能通过类关联添加属性，这在后面提到）

### category 实现和加载
category 的源码定义如下
```Objective-C
typedef struct objc_category *Category;

// objc 2.0
struct objc_category {
char *category_name                                      OBJC2_UNAVAILABLE;
char *class_name                                         OBJC2_UNAVAILABLE;
struct objc_method_list *instance_methods                OBJC2_UNAVAILABLE;
struct objc_method_list *class_methods                   OBJC2_UNAVAILABLE;
struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
}

// objc_category 的实现
struct category_t {
const char *name;
classref_t cls;
struct method_list_t *instanceMethods;
struct method_list_t *classMethods;
struct protocol_list_t *protocols;
struct property_list_t *instanceProperties;
// Fields below this point are not always present on disk.
struct property_list_t *_classProperties;

method_list_t *methodsForMeta(bool isMeta) {
if (isMeta) return classMethods;
else return instanceMethods;
}

property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```
从定义中我们可以看到：
- name：分类名称
- cls：分类所属类
- instance_methods：用于存储实例方法
- class_methods：用于存储类方法
- protocols：用于存储协议方法
- instanceProperties：用于存储添加的属性，不会生成实例变量，因此也不会生成 setter 和 getter 方法。
- _classProperties：用于存储添加的类的属性，该属性不会生成 setter 和 getter 方法。

#### 为什么存在 instanceProperties 和 _classProperties
_classProperties 类的属性，对于我们来说可能很陌生，在开发中很少使用，它的定义和我们常定义的属性差不多，只不过 setter 和 getter方法为 “+ 方法”（类方法）。

```Objective-C
// instance property
@property (nonatomic,strong) NSString *name;
- (NSString *)name;
- (void)setName:(NSString *)name

// class property
@property (class,nonatomic,strong) NSString *name;
+ (NSString *)name;
+ (void)setName:(NSString *)name
```
在前面说了：category 是不能添加实例变量的，我们可能就会疑惑了，为什么会有 instanceProperties 和 _classProperties，这我们就得明白成员变量、实例变量、属性的联系了。
```Objective-C
@interface Father : NSObject {

int age;
NSData *data;
}

@property (nonatomic,strong) NSString *name;

@end
```
##### 成员变量
在 @interface Father : NSObject {} 中声明的变量都是成员变量，编译器不会自动为成员变量声明 setter 和 getter 方法，因此我们无法通过 .语法糖访问成员变量，可通过 self -> 变量名或直接使用变量名，如：
```Objective-C
age = 23;
self -> age = 24;
```
##### 实例变量
实例变量又是什么呢，从本质上，实例变量就是成员变量，实例变量从名字上就能体现出其是类定义的变量。如 data 就是由 NSData 类定义的实例变量，而 age 是基本数据类型 int 变量，因此 age 不是实例变量。所以我们大概可以总结出：成员变量 = 实例变量 + 基本数据类型变量。

##### 属性
成员变量用于类内部，我们无法从类外部访问成员变量，因此诞生了“属性”，属性可以允许在内外部方法，在外部使用 类实例.属性 访问，在类内部使用 self.属性名或者 _变量名访问，但前提是为属性生成了实例变量以及实例变量的 setter 和 getter 方法。

在 iOS 5 前使用 @property 声明属性并声明属性的 setter 和 getter 方法，需在 .m 实现文件中使用 “@sythesize 属性名 = _属性名” 生成与属性对应的实例变量以及实现 setter 和 getter 方法，在 iOS 5 后使用 @property声明属性，编译器会自动为属性生成一个名称为 _属性名的实例变量并实现 setter 和 getter 方法。

##### @sythesize
@sythesize 除了为属性生成 setter 和 getter 方法外还能指定属性对应的实例变量的名称，如 @sythesize name，那么 name 属性的实例变量名不再是默认生成的 _name 而是 name，如果 @sythesize name = _name 的话，实例变量名依然为 _name。

##### 在 category 中添加属性
在 category 中添加属性后能编译成功，但一旦使用了属性，程序便会崩溃，原因为：未找到属性的 setter 或 getter 方法的实现，于是我们开始为属性手动实现 setter 或 getter 方法，我们会发现无法访问到属性。通过 runtime 的 class_copyPropertyList() 获取类的所有属性 以及 class_copyIvarList() 获取类的变量，我们可以发现，在属性列表中存在该属性，而在变量列表中却没有对应的实例变量，所以我们可知编译器未给属性生成对应的实例变量，因此手动实现 setter 和 getter 方法的想法破灭了。当我们试图在 category 创建成员变量时，编译器会报 Instance variables may not be placed in categories 错误。

正所谓上有政策下有对策，我们可以创建一个全局变量来保存属性值，就如这样:
```Objective-C
@interface NSObject (test)
@property (nonatomic,strong) NSString *name;
@end

NSString *_name;
@implementation NSObject (test)

- (NSString *)name {
return _name;
}

- (void)setName:(NSString *)name
{
_name = name;
}

@end
```
或者我们可以通过 runtime的关联对象 来实现属性的 setter 和 getter 方法
```Objective-C
@interface NSObject (test)
@property (nonatomic,strong) NSString *name;

@end

#import <objc/runtime.h>

@implementation NSObject (test)

- (NSString *)name {
return objc_getAssociatedObject(self, @selector(name));
}

- (void)setName:(NSString *)name
{
objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```
所以在 category 中无法添加实例变量，但是可以添加属性，但是不能添加类属性（后节分类的加载有提到）。

#### 编译器对 category 的处理
```Objective-C
@interface NSObject (Category)

- (void)printDescription;

@end

@implementation NSObject (Category)

- (void)printDescription
{
NSLog(@"%@",self.description);
}

@end
```
使用 clang -rewrite-objc 文件名.m 命令对上面代码转换成源码
```Objective-C
static struct /*_method_list_t*/ {
unsigned int entsize;  // sizeof(struct _objc_method)
unsigned int method_count;
struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_NSObject_$_Category __attribute__ ((used, section ("__DATA,__objc_const"))) = {
sizeof(_objc_method),
1,
{{(struct objc_selector *)"printDescription", "v16@0:8", (void *)_I_NSObject_Category_printDescription}}
};

extern "C" __declspec(dllimport) struct _class_t OBJC_CLASS_$_NSObject;

static struct _category_t _OBJC_$_CATEGORY_NSObject_$_Category __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
"NSObject",
0, // &OBJC_CLASS_$_NSObject,
(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_NSObject_$_Category,
0,
0,
0,
};
static void OBJC_CATEGORY_SETUP_$_NSObject_$_Category(void ) {
_OBJC_$_CATEGORY_NSObject_$_Category.cls = &OBJC_CLASS_$_NSObject;
}
#pragma section(".objc_inithooks$B", long, read, write)
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CATEGORY_SETUP[] = {
(void *)&OBJC_CATEGORY_SETUP_$_NSObject_$_Category,
};
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
&_OBJC_$_CATEGORY_NSObject_$_Category,
};
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };

```
我们可以看到：
- 首先，生成了方法列表对象 _OBJC_$_CATEGORY_INSTANCE_METHODS_NSObject_$_Category
- 然后，用方法列表对象初始化 _OBJC_$_CATEGORY_NSObject_$_Category (NSObject+Category)分类对象，在 OBJC_CATEGORY_SETUP_$_NSObject_$_Category 方法中将 _OBJC_$_CATEGORY_NSObject_$_Category 的 cls 指向 OBJC_CLASS_$_NSObject （NSObject 类）。
- 最后，将 NSObject+Category 分类 保存到 DATA段下的objc_catlist section里的 L_OBJC_LABEL_CATEGORY_$ 中，在运行期时用于分类的加载。

#### category 加载
Objective-C 运行时入口方法：
```Objective-C
// objc4-706  objc-os.mm
void _objc_init(void)
{
static bool initialized = false;
if (initialized) return;
initialized = true;

// fixme defer initialization until an objc-using image is found?
environ_init();
tls_init();
static_init();
lock_init();
exception_init();

_dyld_objc_notify_register(&map_2_images, load_images, unmap_image);
}

// objc4-680  objc-os.mm
void _objc_init(void)
{
static bool initialized = false;
if (initialized) return;
initialized = true;

// fixme defer initialization until an objc-using image is found?
environ_init();
tls_init();
static_init();
lock_init();
exception_init();

// Register for unmap first, in case some +load unmaps something
_dyld_register_func_for_remove_image(&unmap_image);
dyld_register_image_state_change_handler(dyld_image_state_bound,
1/*batch*/, &map_2_images);
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}

// objc4-437  objc-os.m
void _objc_init(void)
{
// fixme defer initialization until an objc-using image is found?
environ_init();
tls_init();
lock_init();
exception_init();

// Register for unmap first, in case some +load unmaps something
_dyld_register_func_for_remove_image(&unmap_image);
dyld_register_image_state_change_handler(dyld_image_state_bound,
1/*batch*/, &map_images);
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```
以上是三个版本的定义，最新版本 706 变得更简洁了。category 附加到类上面是在 map_images/map_2_images（images这里表示二进制文件(可执行文件或者动态链接库 .iso 文件)编译后的符号、代码） 的时候发生的，map_images/map_2_images 两个版本中，都在在方法里调用了 map_images_nolock 方法，在 map_images_nolock 函数中加载 images
```Objective-C
// Find all images with Objective-C metadata.
hCount = 0;
i = infoCount;
while (i--) {
const headerType *mhdr = (headerType *)infoList[i].imageLoadAddress;

hi = _objc_addHeader(mhdr);
if (!hi) {
// no objc data in this entry
continue;
}

hList[hCount++] = hi;


if (PrintImages) {
_objc_inform("IMAGES: loading image for %s%s%s%s%s\n", 
_nameForHeader(mhdr), 
mhdr->filetype == MH_BUNDLE ? " (bundle)" : "", 
_objcHeaderIsReplacement(hi) ? " (replacement)" : "",
_objcHeaderOptimizedByDyld(hi)?" (preoptimized)" : "",
_gcForHInfo2(hi));
}
}
```
在加载完 images 后，便在方法末尾调用 _read_images函数，我们在 _read_images函数 找到处理分类的相关代码
```Objective-C
// Discover categories. 
for (EACH_HEADER) {
category_t **catlist = 
_getObjc2CategoryList(hi, &count);
for (i = 0; i < count; i++) {
category_t *cat = catlist[i];
// Do NOT use cat->cls! It may have been remapped.
class_t *cls = remapClass(cat->cls);

// Process this category. 
// First, register the category with its target class. 
// Then, rebuild the class's method lists (etc) if 
// the class is realized. 
BOOL classExists = NO;
if (cat->instanceMethods ||  cat->protocols  
||  cat->instanceProperties) 
{
addUnattachedCategoryForClass(cat, cls, hi);
if (isRealized(cls)) {
remethodizeClass(cls);
classExists = YES;
}
if (PrintConnecting) {
_objc_inform("CLASS: found category -%s(%s) %s", 
getName(cls), cat->name, 
classExists ? "on existing class" : "");
}
}

if (cat->classMethods  ||  cat->protocols  
/* ||  cat->classProperties */) 
{
addUnattachedCategoryForClass(cat, cls->isa, hi);
if (isRealized(cls->isa)) {
remethodizeClass(cls->isa);
}
if (PrintConnecting) {
_objc_inform("CLASS: found category +%s(%s)", 
getName(cls), cat->name);
}
}
}
}

```
首先，通过 _getObjc2CategoryList(hi, &count) 获取的 catlist 就是在编译器编译时的 L_OBJC_LABEL_CATEGORY_$。获取到 category_t 列表后，开始遍历 catlist，将 instanceMethods（实例方法）、protocols（协议）、instanceProperties（属性）添加到类上，将 classMethods（类方法）、protocols（协议）添加到类的元类（meta class）上。在添加到元类时，有一段注释 /* ||  cat->classProperties */ 可见并不会将 classProperties （类属性）添加到元类上，不支持在分类给类添加类属性。

##### addUnattachedCategoryForClass 函数
```Objective-C
class_t *cls = remapClass(cat->cls);
addUnattachedCategoryForClass(cat, cls, hi);

static void addUnattachedCategoryForClass(category_t *cat, class_t *cls,
header_info *catHeader)
{
rwlock_assert_writing(&runtimeLock);

BOOL catFromBundle = (catHeader->mhdr->filetype == MH_BUNDLE) ? YES: NO;

// DO NOT use cat->cls! 
// cls may be cat->cls->isa, or cat->cls may have been remapped.
// 获取所有未进行处理的分类
NXMapTable *cats = unattachedCategories();
category_list *list;

// 根据 cls 获取该类未处理的分类
list = NXMapGet(cats, cls);
if (!list) {
list = _calloc_internal(sizeof(*list) + sizeof(list->list[0]), 1);
} else {
list = _realloc_internal(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
}
list->list[list->count++] = (category_pair_t){cat, catFromBundle};
NXMapInsert(cats, cls, list);
}
```
实现代码也比较简单，大致内容是：获取所有未进行处理的分类，如果当前类在 NXMapGet 中未拥有未处理的分类，那么 list 不存在，便会初始化 list。如果 list 存在，便会用 list 重新初始化并将容量加 1，然后将分类添加到 list 中，最后将分类列表插入到 NXMapTable 中。所以可见 addUnattachedCategoryForClass 函数是将类和分类做了一个关联存储。

##### remethodizeClass 函数
remethodizeClass 是去处理分类方法添加的入口
```Objective-C
// 删除无关分类的代码
static void remethodizeClass(struct class_t *cls)
{
category_list *cats;
BOOL isMeta;

rwlock_assert_writing(&runtimeLock);

isMeta = isMetaClass(cls);

// Re-methodizing: check for more categories
if ((cats = unattachedCategoriesForClass(cls))) {

BOOL vtableAffected = NO;

// Update methods, properties, protocols

attachCategoryMethods(cls, cats, &vtableAffected);

_free_internal(cats);

// Update method caches and vtables
flushCaches(cls);
if (vtableAffected) flushVtables(cls);
}
}
```
首先会执行 unattachedCategoriesForClass(cls)，以下是 unattachedCategoriesForClass 函数的实现
```Objective-C
/***********************************************************************
* unattachedCategoriesForClass
* Returns the list of unattached categories for a class, and 
* deletes them from the list. 
* The result must be freed by the caller. 
* Locking: runtimeLock must be held by the caller.
**********************************************************************/
static category_list *unattachedCategoriesForClass(class_t *cls)
{
rwlock_assert_writing(&runtimeLock);
return NXMapRemove(unattachedCategories(), cls);
}
```
根据注释和方法名，我们可以很清晰的知道 unattachedCategoriesForClass 做了什么，调用 NXMapRemove 函数，NXMapRemove 函数以 cls 为 key 从 NXMapTable 中删除类映射的分类列表并返回类的分类列表，最终 unattachedCategoriesForClass 将 NXMapRemove 函数得到分类列表返回。

在执行 unattachedCategoriesForClass 函数获得分类列表后，将分类列表传入 attachCategoryMethods 函数中，attachCategoryMethods 函数的实现如下
```Objective-C
attachCategoryMethods(class_t *cls, category_list *cats, 
BOOL *outVtablesAffected)
{
if (!cats) return;
if (PrintReplacedMethods) printReplacements(cls, cats);

BOOL isMeta = isMetaClass(cls);
method_list_t **mlists = _malloc_internal(cats->count * sizeof(*mlists));

// Count backwards through cats to get newest categories first
int mcount = 0;
int i = cats->count;
BOOL fromBundle = NO;
while (i--) {
method_list_t *mlist = cat_method_list(cats->list[i].cat, isMeta);
if (mlist) {
mlists[mcount++] = mlist;
fromBundle |= cats->list[i].fromBundle;
}
}

attachMethodLists(cls, mlists, mcount, fromBundle, outVtablesAffected);

_free_internal(mlists);

}
```
此函数中，**mlists 是个二维数组，遍历 cats 分类列表，将分类方法数据存入 **mlists 中，并将 mlists 存储到 class_rw_t 的方法列表中。最后将 mlists 传入 attachMethodLists 函数中，该函数的实现
```Objective-C
static void 
attachMethodLists(class_t *cls, method_list_t **lists, int count, 
BOOL methodsFromBundle, BOOL *outVtablesAffected)
{
rwlock_assert_writing(&runtimeLock);

BOOL vtablesAffected = NO;
size_t listsSize = count * sizeof(*lists);

// Create or extend method list array
// Leave `count` empty slots at the start of the array to be filled below.

if (!cls->data->methods) {
// no bonus method lists yet
cls->data->methods = _calloc_internal(1 + count, sizeof(*lists));
} else {
size_t oldSize = malloc_size(cls->data->methods);
cls->data->methods = 
_realloc_internal(cls->data->methods, oldSize + listsSize);
memmove(cls->data->methods + count, cls->data->methods, oldSize);
}

// Add method lists to array.
// Reallocate un-fixed method lists.

int i;
for (i = 0; i < count; i++) {
method_list_t *mlist = lists[i];
if (!mlist) continue;

// Fixup selectors if necessary
if (!isMethodListFixedUp(mlist)) {
mlist = _memdup_internal(mlist, method_list_size(mlist));
fixupMethodList(mlist, methodsFromBundle);
}

// Scan for vtable updates
if (outVtablesAffected  &&  !vtablesAffected) {
uint32_t m;
for (m = 0; m < mlist->count; m++) {
SEL sel = method_list_nth(mlist, m)->name;
if (vtable_containsSelector(sel)) vtablesAffected = YES;
}
}

// Fill method list array
cls->data->methods[i] = mlist;
}

if (outVtablesAffected) *outVtablesAffected = vtablesAffected;
}

```
首先判断类的方法总列表 methods 是否是为空，如果为空，以分类方法列表 lists 初始化 methods，如果 methods 不为空，便以原 methods 和 lists 初始化，通过 memmove 宏方法将 methods 容量扩容增加 count （即 lists 中的方法列表个数）并将原来的 methods 中的方法列表往后移 count 位，这样原来的方法列表就被移至到 methods 中的后面部分。

对于 methods 方法总列表的处理也就结束了，接下来便是开始添加方法列表至 methods 中。遍历 lists，对方法列表 mlist 中的方法进行表注册等处理，最后添加到 methods 中。

最后的添加是将分类方法列表添加到 methods 的前面，这样的添加顺序，使得即使 category 中存在和原来类一样的方法也不会覆盖掉原来类的方法。调用方法时，会从 methods 的前面开始遍历，当找到方法后就开始调用并终止查询，所以在后面存在一样的方法便不可能会被调用。

### category 与 + load 方法
创建以下文件，并调用 + load 方法
```Objective-C
// Father.h
@interface Father : NSObject
+ (void)load;

@end
// Father.m
@implementation Father
+ (void)load {
NSLog(@"[Father load]");
}

// Father+Category.h
@interface Father (Category)
+ (void)load;

@end
// Father+Category.m
@implementation Father (Category)

+ (void)load {
NSLog(@"[Category load]");
}

// Father+Category2.h
@interface Father (Category2)
+ (void)load;

@end
// Father+Category2.m
@implementation Father (Category2)

+ (void)load {
NSLog(@"[Category2 load]");
}
```
以编译文件顺序为：Father、Father+Category、Father+Category2，根据上节讲述的，我们可能会觉的打印的结果如下：
```Objective-C
[Category2 load]
```
但其实正确的结果如下：
```Objective-C
[Father load]
[Category load]
[Category2 load]
```
#### 从 objc_init 中看 + load 的处理
这是为什么呢？说好的只会执行一次 load？我们来看 runtime 入口方法 _objc_init
```Objective-C
void _objc_init(void)
{
// fixme defer initialization until an objc-using image is found?
environ_init();
tls_init();
lock_init();
exception_init();

// Register for unmap first, in case some +load unmaps something
_dyld_register_func_for_remove_image(&unmap_image);
dyld_register_image_state_change_handler(dyld_image_state_bound,
1/*batch*/, &map_images);
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```
在 map_images 执行完后，category 的方法也就添加到类上了，最后执行
```Objective-C
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
```
初始化 dyld 所需的依赖和注册 load_images 回调通知，当 iamge 被加载就会通知 runtime 处理，该函数的实现如下
```Objective-C

/***********************************************************************
* load_images
* Process +load in the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock and loadMethodLock
**********************************************************************/
__private_extern__ const char *
load_images(enum dyld_image_states state, uint32_t infoCount,
const struct dyld_image_info infoList[])
{
BOOL found;

recursive_mutex_lock(&loadMethodLock);

// Discover load methods
rwlock_write(&runtimeLock);
found = load_images_nolock(state, infoCount, infoList);
rwlock_unlock_write(&runtimeLock);

// Call +load methods (without runtimeLock - re-entrant)
if (found) {
call_load_methods();
}

recursive_mutex_unlock(&loadMethodLock);

return NULL;
}
```
通过 load_images_nolock 函数查询 images中是否有 + load 方法并收集 + load方法，该方法实现如下
```Objective-C
load_images_nolock(enum dyld_image_states state,uint32_t infoCount,
const struct dyld_image_info infoList[])
{
BOOL found = NO;
uint32_t i;

i = infoCount;
while (i--) {
header_info *hi;
for (hi = FirstHeader; hi != NULL; hi = hi->next) {
const headerType *mhdr = (headerType*)infoList[i].imageLoadAddress;
if (hi->mhdr == mhdr) {
prepare_load_methods(hi);
found = YES;
}
}
}

return found;
}
```
使用 while 循环遍历加载 iamges 中的 + load，其中处理便是 prepare_load_methods 这个函数，我们看看这个函数的实现
```Objective-C
void prepare_load_methods(header_info *hi)
{
size_t count, i;

rwlock_assert_writing(&runtimeLock);

class_t **classlist = 
_getObjc2NonlazyClassList(hi, &count);
for (i = 0; i < count; i++) {
class_t *cls = remapClass(classlist[i]);
schedule_class_load(cls);
}

category_t **categorylist = _getObjc2NonlazyCategoryList(hi, &count);
for (i = 0; i < count; i++) {
category_t *cat = categorylist[i];
// Do NOT use cat->cls! It may have been remapped.
class_t *cls = remapClass(cat->cls);
realizeClass(cls);
assert(isRealized(cls->isa));
add_category_to_loadable_list((Category)cat);
}
}
```
我们可以看到，首先被处理的是类的 + load 方法，将类和类的 + load  通过 schedule_class_load 函数中的 add_class_to_loadable_list 函数收集在 loadable_classes 中，然后才是处理分类中的 + load 方法，将分类和分类中的 + load方法收集在 loadable_categories 中。至此，+ load 收集准备工作也就完成了。

回到 load_images 函数， 如果有 + load 方法的话，那么 load_images_nolock 则会返回 YES，那么接下来便会执行 call_load_methods 函数，调用所有收集的 + load 方法，我们来看看它的实现
```Objective-C
void call_load_methods(void)
{
static BOOL loading = NO;
BOOL more_categories;

recursive_mutex_assert_locked(&loadMethodLock);

// Re-entrant calls do nothing; the outermost call will finish the job.
if (loading) return;
loading = YES;

do {
// 1. Repeatedly call class +loads until there aren't any more
while (loadable_classes_used > 0) {
call_class_loads();
}

// 2. Call category +loads ONCE
more_categories = call_category_loads();

// 3. Run more +loads if there are classes OR more untried categories
} while (loadable_classes_used > 0  ||  more_categories);

loading = NO;
}
```
在 do-while 循环中我们可以看到，首先执行的是类的+ load 方法，然后是分类的 + load 方法 。

#### 总结
runtime 会收集所有的 + load 方法并调用所有的 + load 方法，所以即使在分类中实现 + load 方法也不会覆盖类的+ load 方法。+ load 的调用顺序，类总是在分类之前调用，而分类里的 + load 方法调用顺序自然是以最后编译的分类的 + load 方法为先。

### 后言
本次源码版本是 objc4-437，附上下载入口 [源码下载](https://opensource.apple.com/tarballs/objc4/)。作为一只菜鸟，也有向往成为大神的梦想。阅读源码，有的时候让自己很是头疼，当慢慢反复阅读后，感觉豁然开朗。本文若有错误之处还望指出，谢谢！
