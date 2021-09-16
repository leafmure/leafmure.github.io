---
title: __weak 修饰符
date: 2018-12-19 12:03:55
categories: iOS
tags:
- __weak
keywords: __weak,weak,循环引用,SideTables
description:
images:
---

### SideTables
SideTables 是全局表，管理着对象的引用计数和weak引用指针，每个对象在此表中都有对应的一个 SideTable，让我们来看看 SideTables 源码定义
<!-- more -->
```
static StripedMap<SideTable>& SideTables() {
return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```
虽然看不懂，但从源码的定义可以看出 SideTables 是通过 StripedMap 来实现的，我们来看一下它的实现
```
template<typename T>
class StripedMap {

enum { CacheLineSize = 64 };

#if TARGET_OS_EMBEDDED // 嵌入式
enum { StripeCount = 8 };
#else
enum { StripeCount = 64 };
#endif

struct PaddedT {
T value alignas(CacheLineSize);
};

PaddedT array[StripeCount];

static unsigned int indexForPointer(const void *p) {
uintptr_t addr = reinterpret_cast<uintptr_t>(p);
return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}

public:
T& operator[] (const void *p) { 
return array[indexForPointer(p)].value; 
}
const T& operator[] (const void *p) const { 
return const_cast<StripedMap<T>>(this)[p]; 
}
};
```
从上可以看出，StripedMap 的容量大小为：StripeCount = 64，通过 indexForPointer 函数分配在 StripedMap的下标。public 中的应该是读存方法吧（猜测，哈哈。。），可以看到其和 Map 集合一样。

#### SideTable 
SideTable 的定义如下
```
typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,true> RefcountMap;

struct SideTable {
spinlock_t slock;
RefcountMap refcnts;
weak_table_t weak_table;
...
};
```
- slock：是用于在对 SideTable 操作时，对 SideTable 加锁防止其他访问。
- refcnts：对象的引用计数器，存储着对象被引用的记录。
- weak_table_t: 存放弱变量引用

##### DisguisedPtr
```
class DisguisedPtr {
uintptr_t value;

// 对指针进行伪装
static uintptr_t disguise(T* ptr) {
return -(uintptr_t)ptr;
}
// 恢复至原指针
static T* undisguise(uintptr_t val) {
return (T*)-val;
}
};

```
DisguisedPtr 对指针进行伪装的类，将指针强转为 uintptr_t （unsigned long）类型的负值，这样类似 leaks 这样的查内存泄漏的工具便无法跟踪到对象。

##### weak_table_t
```
struct weak_table_t {
weak_entry_t *weak_entries;
size_t    num_entries;
uintptr_t mask;
uintptr_t max_hash_displacement;
};
```
- weak_entries：存放对象与弱引用对象指针映射的弱引用条目数组
- num_entries：弱引用条目总数
- mask：可存储弱引用条目的容量
- max_hash_displacement：最大哈希偏移值

##### weak_entry_t
```
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
DisguisedPtr<objc_object> referent;
union {
struct {
weak_referrer_t *referrers;
uintptr_t        out_of_line_ness : 2;
uintptr_t        num_refs : PTR_MINUS_2;
uintptr_t        mask;
uintptr_t        max_hash_displacement;
};
struct {

weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
};
};
};
```
weak_entry_t 是一个弱引用条目，其映射了引用对象和其被弱引用的指针，referent 便是引用对象，union（联合体） 里存放着弱引用该对象的指针，union 里面的多个成员变量共享同一内存空间。union 中有两个结构体都是存储弱引用对象指针的集合。第一个结构体中 referrers 是一个可进行扩容的集合，而第二个结构体中 inline_referrers 是一个容量为 4 的数组，weak_entry_t 默认使用 inline_referrers 来保存弱引用指针，当此数组容量满后，会使用 referrers 接管保存工作。out_of_line_ness 便是描述存储的弱引用指针是否超出 inline_referrers 的容量。

### __weak 原理
```
NSString *aa = @"aa";
__weak NSString *test = aa;
```
上面代码在编译时，模拟的代码如下：
```
NSString *aa;
aa = @"aa"
NSString *test;
objc_initWeak(&obj, aa);
objc_destoryWeak(&obj);
```
#### __weak 变量创建

__weak 变量的创建入口是 objc_initWeak 这个函数，其实现是：
```
id objc_initWeak(id *location, id newObj)
{
if (!newObj) {
*location = nil;
return nil;
}

return storeWeak<false/*old*/, true/*new*/, true/*crash*/>
(location, (objc_object*)newObj);
}

```
如果 __weak 变量被赋予的对象是 nil 那么，将 __weak 变量置 nil，进入 objc_destoryWeak 销毁函数。storeWeak 函数是一个更新弱变量的函数，此函数有点长，我们分段讲述：
```
// 如果 HaveOld 为 true，则表明变量需要清理，变量可能为nil
// 如果 HaveNew 为 true，则表明有一个新值将赋予变量，这个新值可能为 nil
// 如果 CrashIfDeallocating 为 true，则表明新值 newObj 是释放了的对象（并不是说 newObj 为 nil）或者是一个不支持弱引用的对象。
// 如果 CrashIfDeallocating 为 false，则将新值 newObj 置nil 并 *location 弱变量赋值为 nil。 
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
assert(HaveOld  ||  HaveNew);
if (!HaveNew) assert(newObj == nil);

Class previouslyInitializedClass = nil;
id oldObj;
SideTable *oldTable;
SideTable *newTable;

// Acquire locks for old and new values.
// 为新旧值获取锁。
// Order by lock address to prevent lock ordering problems. 
// 按锁地址排序以防止锁排序问题
// Retry if the old value changes underneath us.
// 如果下面的旧值发生更改，请重试。
retry:
if (HaveOld) {
oldObj = *location;
oldTable = &SideTables()[oldObj];
} else {
oldTable = nil;
}
if (HaveNew) {
newTable = &SideTables()[newObj];
} else {
newTable = nil;
}
```
声明一个 previouslyInitializedClass 保存先前初始化的类，声明一个旧值对象 oldObj，一新一旧两个 SideTable（散列表）。从 objc_initWeak 传入的 HaveOld 为 false，HaveNew 为 true，因此将 oldTable 赋值为 nil，从 SideTables 获取 newObj 的 SideTable 赋值给 newTable。两个散列表处理好了后，因为当前是 __weak 变量的创建，处理的是新值，所以下面只给出新值有关的处理代码
```
// 给新旧散列表加锁
SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

// 通过确保没有弱引用对象具有未初始化的isa，防止弱引用机制和初始化机制之间的死锁。
if (HaveNew  &&  newObj) {
Class cls = newObj->getIsa();
if (cls != previouslyInitializedClass  &&  
!((objc_class *)cls)->isInitialized()) 
{
SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
_class_initialize(_class_getNonMetaClass(cls, (id)newObj));

// 如果这个类完成了+initialize，那最好。如果这个类仍在这个线程上运行+initialize（即+initialize，调用storeWeak，在其自身的实例上），我们可以继续，但是将显示为初始化和尚未初始化作为上述检查，设置 previouslyInitializedClass 以在重试时识别它
previouslyInitializedClass = cls;

goto retry;
}
}
```
此步骤为确保弱引用对象 newObj 初始化了，首先通过获取 newObj 的 isa 指针获取它的类，然后判断它的类是否初始化了，如果没有，便打开新旧散列表的锁，获取 newObj 的元类发送 +initialize 消息进行初始化。下面是 storeWeak 函数最后一部分：

```

// Assign new value, if any.
if (HaveNew) {
// 弱引用注册失败便返回 nil
newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, (id)newObj, location, CrashIfDeallocating);

// 设置refcount表中的弱引用位
if (newObj  &&  !newObj->isTaggedPointer()) {
newObj->setWeaklyReferenced_nolock();
}

// 不要在其他地方设置 *location，否则会有冲突。
*location = (id)newObj;
}
// 打开新旧散列表的锁
SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

// 弱引用处理完毕，返回新值
return (id)newObj;
```
首先，通过 weak_register_no_lock 函数将 __weak 变量对 newObj 的弱引用注册到 newObj的散列表的弱引用表中，如果注册成功则设置 newObj 的refcount表中的 __weak 变量对其的引用为弱引用，然后将新值赋给 __weak 变量。最后，所有操作完成了，打开新旧散列表的锁，返回新值赋给 __weak 变量。



#### 弱引用注册
__weak 变量引用对象时，需要将 __weak 变量的弱引用注册到被引用对象的弱引用表中，这一操作便由 weak_register_no_lock 函数完成。此函数的实现我们分两部分程呈现：
```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
id *referrer_id, bool crashIfDeallocating)
{
objc_object *referent = (objc_object *)referent_id;
objc_object **referrer = (objc_object **)referrer_id;

if (!referent  ||  referent->isTaggedPointer()) return referent_id;

// ensure that the referenced object is viable
// 确保引用的对象是可用的
bool deallocating;
if (!referent->ISA()->hasCustomRR()) {
deallocating = referent->rootIsDeallocating();
}
else {
BOOL (*allowsWeakReference)(objc_object *, SEL) = 
(BOOL(*)(objc_object *, SEL))
object_getMethodImplementation((id)referent, 
SEL_allowsWeakReference);
if ((IMP)allowsWeakReference == _objc_msgForward) {
return nil;
}
deallocating =
! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
}

if (deallocating) {
if (crashIfDeallocating) {
_objc_fatal("Cannot form weak reference to instance (%p) of "
"class %s. It is possible that this object was "
"over-released, or is in the process of deallocation.",
(void*)referent, object_getClassName((id)referent));
} else {
return nil;
}
}
```
上部分是为了确保弱引用的对象 referent（newObj对象）支持被弱引用。首先判断引用对象 referent 的 isa 中是否有自定义 retain 和 release 实现，如果没有，则调用 rootIsDeallocating() 函数检查 referent 是否在析构（即是否被释放）。

如果referent 的 isa 中有自定义 retain 和 release 实现，首先会获取 referent 中 SEL_allowsWeakReference 方法的实现，如果获取的是 _objc_msgForward 消息转发函数，那么表明该引用对象不支持弱引用。反之，便发送 SEL_allowsWeakReference 消息去判断该对象是否支持弱引用，如果支持则表示 referent 引用对象不在析构。

如果该引用对象在析构并且 crashIfDeallocating（控制引用对象析构是否需crash）为true，则crash。如果 crashIfDeallocating 为 false，则返回 nil 表示注册弱引用失败。

```
// now remember it and where it is being stored
weak_entry_t *entry;
if ((entry = weak_entry_for_referent(weak_table, referent))) {
append_referrer(entry, referrer);
} 
else {
weak_entry_t new_entry(referent, referrer);
weak_grow_maybe(weak_table);
weak_entry_insert(weak_table, &new_entry);
}

// Do not set *referrer. objc_storeWeak() requires that the 
// value not change.

return referent_id;
}
```
这下半部分就是将 __weak 变量的弱引用指针存储到被引用对象 newObj 的弱引用表中，完成注册。首先通过 weak_entry_for_referent 函数去查找 newObj 对应的 SideTable 的 weak_table 表中的对应 newObj 的弱引用条目 entry。如果不存在 entry，则用 newobj 和 __weak变量指针生成一个新的弱引用条目 new_entry。接下来执行 weak_grow_maybe 函数看 weak_table 是否需要扩容。

##### weak_grow_maybe 和 weak_resize
```
#define TABLE_SIZE(entry) (entry->mask ? entry->mask + 1 : 0)

static void weak_grow_maybe(weak_table_t *weak_table)
{
size_t old_size = TABLE_SIZE(weak_table);

// Grow if at least 3/4 full.
// 如果当前条目数已满容量的 3/4 则允许扩容
// 如果是初次的话扩容 64，之后以 2 倍增加
if (weak_table->num_entries >= old_size * 3 / 4) {
weak_resize(weak_table, old_size ? old_size*2 : 64);
}
}
```
如果 newObj 是第一次被引用，那么其对应的 weak_table 的容量 mask 应为 0，则 old_size = 0， weak_table 的弱引用条目总数自然也为 0。满足扩容条件，因此初次扩容为 64，执行 weak_resize(weak_table, 64)。

weak_resize 是对 weak_table 扩容的函数，其实现如下：
```
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
size_t old_size = TABLE_SIZE(weak_table);

weak_entry_t *old_entries = weak_table->weak_entries;
weak_entry_t *new_entries = (weak_entry_t *)
calloc(new_size, sizeof(weak_entry_t));

weak_table->mask = new_size - 1;
weak_table->weak_entries = new_entries;
weak_table->max_hash_displacement = 0;
weak_table->num_entries = 0;  // restored by weak_entry_insert below

if (old_entries) {
weak_entry_t *entry;
weak_entry_t *end = old_entries + old_size;
for (entry = old_entries; entry < end; entry++) {
if (entry->referent) {
weak_entry_insert(weak_table, entry);
}
}
free(old_entries);
}
}
```
创建一个 weak_entry_t 实例 old_entries 保存 weak_table 弱引用表中的弱引用条目列表，然后创建一个 64 格的新弱引用条目列表，接着更新设置 weak_table 的容量大小、弱引用条目列表、最大哈希位移数、条目总数。最后，如果旧弱引用条目列表 old_entries 存在数据，则将旧条目列表的数据插入 weak_table 新扩容的条目列表中并释放旧条目列表。

扩容完后，便开始将新创建的条目插入 weak_table 的条目列表 weak_entries 中。
##### weak_entry_insert
```
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
weak_entry_t *weak_entries = weak_table->weak_entries;
assert(weak_entries != nil);

size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
size_t index = begin;
size_t hash_displacement = 0;
while (weak_entries[index].referent != nil) {
index = (index+1) & weak_table->mask;
if (index == begin) bad_weak_table(weak_entries);
hash_displacement++;
}

weak_entries[index] = *new_entry;
weak_table->num_entries++;

if (hash_displacement > weak_table->max_hash_displacement) {
weak_table->max_hash_displacement = hash_displacement;
}
}
```
通过 hash_pointer(new_entry->referent) 和 weak_table->mask 的与运算决定新弱引用条目在 weak_entries 的初始下标，如果 weak_entries 中该下标中没被用，则将 new_entry 存放在此处，weak_table 的 num_entries 自增长 1。

如果该初始下标中已被存放了条目，则循环将 hash_pointer() 计算的 hash值 + 1 再次与 weak_table->mask 进行“与运算”并且哈希位移数自增加 1，如果没找到可存储的位置则会执行 bad_weak_table 报 “This may be a runtime bug or a memory error somewhere else.” 错误。

如果找到可用的位置，则 new_entry 将存放到 weak_entries 中。最后，如果哈希位移数大于 weak_table 中存储的最大哈希位移数，则更新 weak_table 中的 max_hash_displacement 值为 hash_displacement。 到此处，弱引用的注册也就完成了。

##### append_referrer
如果 newObj 是一个被其他变量弱引用的对象，那么能通过 weak_entry_for_referent 函数找到 newObj 对应的弱引用条目。将 __weak 变量的指针保存到弱引用条目的引用指针数组中完成注册，我们来看看这个过程是怎么样的
```
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
if (! entry->out_of_line()) {
// Try to insert inline.
for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
if (entry->inline_referrers[i] == nil) {
entry->inline_referrers[i] = new_referrer;
return;
}
}

// Couldn't insert inline. Allocate out of line.
weak_referrer_t *new_referrers = (weak_referrer_t *)
calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
// This constructed table is invalid, but grow_refs_and_insert
// will fix it and rehash it.
for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
new_referrers[i] = entry->inline_referrers[i];
}
entry->referrers = new_referrers;
entry->num_refs = WEAK_INLINE_COUNT;
entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
entry->mask = WEAK_INLINE_COUNT-1;
entry->max_hash_displacement = 0;
}
```
上部分，判断弱引用条目中存放的引用指针数超过了 inline_referrers 数组的容量。如果没有超过的话（有可能容量已满），则遍历 inline_referrers 找到空位置存放 new_referrer。如果 inline_referrers 容量已满，改用 entry 的 referrers 列表存放引用指针。首先，将 inline_referrers 中存放的引用指针加到 referrers 中，更新设置 num_refs、out_of_line_ness（是否超出了inline_referrers数组的容量）、mask、max_hash_displacement。接下来就进入下部分

```
assert(entry->out_of_line());

if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
return grow_refs_and_insert(entry, new_referrer);
}
```
如果 entry 的引用指针数达到了存放容量的 3/4，那么对 new_referrer 进行扩容并且插入 new_referrer。
#### grow_refs_and_insert
```
#define TABLE_SIZE(entry) (entry->mask ? entry->mask + 1 : 0)
static void grow_refs_and_insert(weak_entry_t *entry, 
objc_object **new_referrer)
{
assert(entry->out_of_line());

size_t old_size = TABLE_SIZE(entry);
size_t new_size = old_size ? old_size * 2 : 8;

size_t num_refs = entry->num_refs;
weak_referrer_t *old_refs = entry->referrers;
entry->mask = new_size - 1;

entry->referrers = (weak_referrer_t *)
calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
entry->num_refs = 0;
entry->max_hash_displacement = 0;

for (size_t i = 0; i < old_size && num_refs > 0; i++) {
if (old_refs[i] != nil) {
append_referrer(entry, old_refs[i]);
num_refs--;
}
}
// Insert
append_referrer(entry, new_referrer);
if (old_refs) free(old_refs);
}
```
这个函数和 weakTable 的扩容函数 weak_resize 一样，首先通过 old_size 计算出扩容的大小 new_size，然后创建一个 weak_referrer_t 实例 old_refs 存放 entry 中 referrers 列表的引用指针，然后对 entry 的 referrers 进行初始化扩容。最后，如果 old_refs 有数据（即原 entry 存在的引用指针），将引用指针通过 append_referrer 插入到扩容后的 referrers 中，此步骤为递归调用。插入的主要代码便是 append_referrer 的最后一部分，如下


```
size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
size_t index = begin;
size_t hash_displacement = 0;
while (entry->referrers[index] != nil) {
hash_displacement++;
index = (index+1) & entry->mask;
if (index == begin) bad_weak_table(entry);
}
if (hash_displacement > entry->max_hash_displacement) {
entry->max_hash_displacement = hash_displacement;
}
weak_referrer_t &ref = entry->referrers[index];
ref = new_referrer;
entry->num_refs++;
}
```
上方代码和 weakTable 的 weak_entry_insert 函数实现原理一样，便不再讲述。从上面分析下来，可见 weakTable 的weak_entries 和 entry 的referrers 一样是一个可以自动扩容的数组，而 entry 的inline_referrers 是一个不可扩容的数组。

#### __weak 变量销毁
__weak 变量销毁会调用 objc_destroyWeak 这个函数
```
void
objc_destroyWeak(id *location)
{
(void)storeWeak<true/*old*/, false/*new*/, false/*crash*/>
(location, nil);
}
```
其中实现也是通过 storeWeak 函数将 __weak 变量置为nil，下面只显示相关代码
```
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
assert(HaveOld  ||  HaveNew);
if (!HaveNew) assert(newObj == nil);

Class previouslyInitializedClass = nil;
id oldObj;
SideTable *oldTable;
SideTable *newTable;

retry:
if (HaveOld) {
oldObj = *location;
oldTable = &SideTables()[oldObj];
} else {
oldTable = nil;
}
if (HaveNew) {
newTable = &SideTables()[newObj];
} else {
newTable = nil;
}

SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

if (HaveOld  &&  *location != oldObj) {
SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
goto retry;
}

// Clean up old value, if any.
if (HaveOld) {
weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
}

SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

return (id)newObj;
}

```
从 objc_destroyWeak 函数传入的 HaveOld = true、HaveNew = false、CrashIfDeallocating = false，首先将 __weak 变量的内存指针指向 oldObj，将当前值变成旧值，对应的从 SideTables 取出对应的 SideTable 为 oldTable，将 newTable 赋值为 nil。然后，将 oldTable 和 newTable 加锁，如果旧值存在并且旧值 __weak 变量内存地址中的值和旧值不相等的话，那么需要重新执行第一步骤以保证销毁工作进行。接下来就是注销 oldObj 对应的 weak_table 中 __weak 变量的弱引用。最后，解开 oldTable 和 newTable 的锁，返回 nil，将 __weak 变量置 nil。

##### weak_unregister_no_lock
```
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
id *referrer_id)
{
objc_object *referent = (objc_object *)referent_id;
objc_object **referrer = (objc_object **)referrer_id;

weak_entry_t *entry;

if (!referent) return;

if ((entry = weak_entry_for_referent(weak_table, referent))) {
remove_referrer(entry, referrer);
bool empty = true;
if (entry->out_of_line()  &&  entry->num_refs != 0) {
empty = false;
}
else {
for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
if (entry->inline_referrers[i]) {
empty = false; 
break;
}
}
}

if (empty) {
weak_entry_remove(weak_table, entry);
}
}

// Do not set *referrer = nil. objc_storeWeak() requires that the 
// value not change.
}
```
首先，通过 weak_entry_for_referent 找到 weak_table 中的弱引用条目 entry，然后通过 remove_referrer 函数从 entry 的引用指针列表中删除 __weak变量指针。如果 entry 中没有引用指针了，那么便会执行 weak_entry_remove 从弱引用表 weak_table 中删除该弱引用条目。

##### remove_referrer
```
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
if (! entry->out_of_line()) {
for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
if (entry->inline_referrers[i] == old_referrer) {
entry->inline_referrers[i] = nil;
return;
}
}
_objc_inform("Attempted to unregister unknown __weak variable "
"at %p. This is probably incorrect use of "
"objc_storeWeak() and objc_loadWeak(). "
"Break on objc_weak_error to debug.\n", 
old_referrer);
objc_weak_error();
return;
}

size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
size_t index = begin;
size_t hash_displacement = 0;
while (entry->referrers[index] != old_referrer) {
index = (index+1) & entry->mask;
if (index == begin) bad_weak_table(entry);
hash_displacement++;
if (hash_displacement > entry->max_hash_displacement) {
_objc_inform("Attempted to unregister unknown __weak variable "
"at %p. This is probably incorrect use of "
"objc_storeWeak() and objc_loadWeak(). "
"Break on objc_weak_error to debug.\n", 
old_referrer);
objc_weak_error();
return;
}
}
entry->referrers[index] = nil;
entry->num_refs--;
}
```
如果，entry 的引用指针数不超过 inline_referrers 的容量，那么遍历 inline_referrers 找到引用指针的位置并置为nil。如果引用指针数不超过 inline_referrers 的容量，那么便得去 referrers 中找到引用指针置为nil并将 referrers 的长度减一。如果找不到便会调用 objc_weak_error()

##### weak_entry_remove
```
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
// remove entry
if (entry->out_of_line()) free(entry->referrers);
bzero(entry, sizeof(*entry));

weak_table->num_entries--;

weak_compact_maybe(weak_table);
}

```
此函数将弱引用条目 entry 从 weak_table 中删除，然后通过 weak_compact_maybe 去检查是否需要缩小 weak_table 的容量。

#### weak_compact_maybe
```
#define TABLE_SIZE(entry) (entry->mask ? entry->mask + 1 : 0)

static void weak_compact_maybe(weak_table_t *weak_table)
{
size_t old_size = TABLE_SIZE(weak_table);

// Shrink if larger than 1024 buckets and at most 1/16 full.
if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
weak_resize(weak_table, old_size / 8);
// leaves new table no more than 1/2 full
}
}
```
如果达到缩小容量大小的要求，便通过 weak_resize 函数调整容量为原来的 1/8。

