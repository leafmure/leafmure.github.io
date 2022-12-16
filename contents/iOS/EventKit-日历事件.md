---
title: EventKit 日历事件
date: 2018-12-19 13:39:24
categories: iOS
tags:
- EventKit
keywords: 日历事件,EventKit,event,reminders
description:
images: "/postCover/EventKit-日历事件.png"
---

#### EventKit 框架
此框架可以访问手机系统日历和提醒事项APP的数据，用此框架可以添加日历事件和提醒事项。

#### info.plist 配置
在 info.plist 添加访问日历权限和提醒实现的key：Privacy - Calendars Usage Description、Privacy - Reminders Usage Description，目前只使用到日历事件的操作。
<!-- more -->
#### EKEventStore
EKEventStore 类似一个数据库，管理着日历事件和提醒事项，我们通过它来操作日历事件和提醒事项。
首先我们创建一个 EKEventStore：
```Objective-C
EKEventStore *eventStore = [[EKEventStore alloc] init];
```
获取授权，entityType 是授权实体类型，completion 是授权状态回调
```Objective-C
/*
typedef NS_ENUM(NSUInteger, EKEntityType) {
EKEntityTypeEvent,  // 日历事件
EKEntityTypeReminder  // 提醒事项
};
*/
- (void)requestAccessToEntityType:(EKEntityType)entityType completion:(EKEventStoreRequestAccessCompletionHandler)completion NS_AVAILABLE(10_9, 6_0);

// 获取日历事件授权
[eventStore requestAccessToEntityType:EKEntityTypeEvent completion:^(BOOL granted, NSError *error){
//  granted 是否授权
//  error 授权失败原因
if (granted) {
// 授权成功
} else {
// 授权失败
}
}];
```

#### EKSource
日历源中拥有多个日历对象，将日历对象进行划分。
```Objective-C

typedef NS_ENUM(NSInteger, EKSourceType) {
EKSourceTypeLocal,  // 本地日历源
EKSourceTypeExchange,  // Exchange协议远程源，网络联网同步数据。
EKSourceTypeCalDAV, // CalDAV协议远程源，网络联网同步数据，iCloud 便是这这类。
EKSourceTypeMobileMe, // 我的iphone
EKSourceTypeSubscribed,  // 订阅
EKSourceTypeBirthdays  // 生日
};

@interface EKSource : EKObject

// 源标识
@property(nonatomic, readonly) NSString        *sourceIdentifier;
// 源类型
@property(nonatomic, readonly) EKSourceType     sourceType;
// 源标题
@property(nonatomic, readonly) NSString        *title;

// 日历源中的日历对象数组
@property(nonatomic, readonly) NSSet<EKCalendar *> *calendars NS_DEPRECATED(NA, NA, 4_0, 6_0);


// 根据实体类型获取源中相对应类型的日历对象
- (NSSet<EKCalendar *> *)calendarsForEntityType:(EKEntityType)entityType NS_AVAILABLE(10_8, 6_0);

@end

```
日历源也是通过 EKEventStore 这个管理者获取的， EKEventStore 有个属性
```Objective-C
// 所有日历源
@property (nonatomic, readonly) NSArray<EKSource *> *sources;

// 根据 identifier 获取日历源
- (nullable EKSource *)sourceWithIdentifier:(NSString *)identifier;
```
我们可以根据名称和类型获取我们要的日历源
```Objective-C
for (EKSource *source in eventStore.sources) {

// 获取 iCloud 源
if (source.sourceType == EKSourceTypeCalDAV && [source.title isEqualToString:@"iCloud"]) {

calendarSource = source;
}
}
```

#### EKCalendar
EKCalendar 日历，添加的日历事件需要选择一个日历对象，日历对象可以通过 EKEventStore 来获取，也可以获取对应的日历源来获取。
```Objective-C
// 所有的日历对象
@property(nonatomic, readonly) NSArray<EKCalendar *> *calendars;

// 通过实体类型来获取相对应的日历对象
- (NSArray<EKCalendar *> *)calendarsForEntityType:(EKEntityType)entityType;

// 一般有指定的默认使用的日历对象，有可能没有，需小心使用
- (nullable EKCalendar *)defaultCalendarForNewReminders;
```
当然，有的时候你想添加一个由自己命名的日历对象：
```Objective-C

// 事件数据管理者
EKEventStore *eventStore = [[EKEventStore alloc] init];
// 获取日历源
EKSource *calendarSource = [self findSource];

// 创建事件日历
EKCalendar *calendar = [EKCalendar calendarForEntityType:EKEntityTypeEvent eventStore:eventStore];
calendar.title = title;
calendar.source = calendarSource;

NSError *err;
[self.eventStore saveCalendar:calendar commit:YES error:&err];
if (err) {

NSLog(@"添加事件日历失败");
}
```

#### EKEvent
EKEvent 是日历事件，显示在下拉框的日程中。
##### 添加日历事件
```Objective-C
// 事件数据管理者
EKEventStore *eventStore = [[EKEventStore alloc] init];
EKEvent *event  = [EKEvent eventWithEventStore:eventStore];
event.title = title; // 标题
event.location = content; // 内容
event.startDate = startDate; // 事件开始时间
event.endDate = endDate; // 事件结束事件
event.allDay = allDay; // 提示是否为 全天事件
event.notes = note; // 备注

// 添加提醒，-1800: 开始前半小时提醒，单位秒，可添加多个
[event addAlarm:[EKAlarm alarmWithRelativeOffset:-1800]];

// 每天重复
EKRecurrenceRule *rule = [[EKRecurrenceRule alloc]initRecurrenceWithFrequency:EKRecurrenceFrequencyDaily interval:1 end:[EKRecurrenceEnd recurrenceEndWithEndDate:endDate]];
event.recurrenceRules = @[rule];

// 设置事件所属的日历对象
[event setCalendar:calendar];

// 添加日历事件
NSError *err;
[eventStore saveEvent:event span:EKSpanThisEvent error:&err];
if (err) {
NSLog(@"添加失败");
} else {
NSLog(@"已添加到系统日历中");
}

// 添加日历事件
- (BOOL)saveEvent:(EKEvent *)event span:(EKSpan)span commit:(BOOL)commit error:(NSError **)error;

// 添加提醒事项
// commit 为YES时，马上提交更改，一次处理一个事件。为 NO 时，不会马上提交改动，可以连续提交多个，在调用 - (BOOL)commit:(NSError **)error 方法后提交所有改动。
- (BOOL)saveReminder:(EKReminder *)reminder commit:(BOOL)commit error:(NSError **)error;
```
##### 重复规则
```Objective-C
// 此枚举类型用于指示对重复事件进行更改的范围。EKSpanThis指示更改应仅应用于此事件，EKSpanFutureEvents指示更改应应用于此事件以及模式中的所有未来事件。
typedef NS_ENUM(NSInteger, EKSpan) {
EKSpanThisEvent,
EKSpanFutureEvents
};

// 重复频率类型
typedef NS_ENUM(NSInteger, EKRecurrenceFrequency) {
EKRecurrenceFrequencyDaily, 
EKRecurrenceFrequencyWeekly,
EKRecurrenceFrequencyMonthly,
EKRecurrenceFrequencyYearly
};

// 完整初始化方法
- (instancetype)initRecurrenceWithFrequency:(EKRecurrenceFrequency)type
interval:(NSInteger)interval 
daysOfTheWeek:(nullable NSArray<EKRecurrenceDayOfWeek *> *)days
daysOfTheMonth:(nullable NSArray<NSNumber *> *)monthDays
monthsOfTheYear:(nullable NSArray<NSNumber *> *)months
weeksOfTheYear:(nullable NSArray<NSNumber *> *)weeksOfTheYear
daysOfTheYear:(nullable NSArray<NSNumber *> *)daysOfTheYear
setPositions:(nullable NSArray<NSNumber *> *)setPositions
end:(nullable EKRecurrenceEnd *)end;

type：重复频率类型
interval：间隔多少天/周/月/年，必须大于0，为1时，表示每天/周/月/年
daysOfTheWeek：EKRecurrenceFrequencyWeekly类型有效，一周中的哪几天触发
daysOfTheMonth：EKRecurrenceFrequencyMonthly类型有效，一个月中的哪几号触发，取值范围：-31～31，必须大于0，-1表示一个月的最后一天
monthsOfTheYear：EKRecurrenceFrequencyYearly类型有效，一年中哪几个月触发，取值范围：1～12，-1表示一年的最后一月
weeksOfTheYear：EKRecurrenceFrequencyYearly类型有效，一年中的哪几周触发，取值范围：-53～53，-1表示一年的最后一周
daysOfTheYear：EKRecurrenceFrequencyYearly类型有效，一年中的哪几天触发，取值范围：-366～366，-1表示一年的最后一天
setPositions：此属性对于具有有效的daysOfTheWeek、daysOfTheMonth、weeksOfTheYear或monthsOfTheYear属性的规则有效，它允许您指定一组序号，以帮助您从所选事件集中选择哪些对象
包括。例如：将daysOfTheWeek 设置为 Monday-Friday，并在数组中包含-1的值将指示重复范围（月、年等）的最后一个工作日，取值范围：-366～366。
end：结束规则
```
##### 查询事件
```Objective-C
// 通过事件标识获取
- (nullable EKEvent *)eventWithIdentifier:(NSString *)identifier;

// 通过 predicate筛选规则获取
- (NSArray<EKEvent *> *)eventsMatchingPredicate:(NSPredicate *)predicate;

// 日历事件 predicate 筛选规则 创建
// 根据开始时间和结束时间，从多个或一个日历对象筛选出符合的日历事件
- (NSPredicate *)predicateForEventsWithStartDate:(NSDate *)startDate endDate:(NSDate *)endDate calendars:(nullable NSArray<EKCalendar *> *)calendars;

// 提醒事项 predicate 筛选规则 创建
// 筛选出多个或一个日历对象中的提醒事项
- (NSPredicate *)predicateForRemindersInCalendars:(nullable NSArray<EKCalendar *> *)calendars;
// 根据开始时间和结束时间，筛选多个或一个日历对象中进行的提醒事项
- (NSPredicate *)predicateForIncompleteRemindersWithDueDateStarting:(nullable NSDate *)startDate ending:(nullable NSDate *)endDate calendars:(nullable NSArray<EKCalendar *> *)calendars;
// 根据开始时间和结束时间，筛选多个或一个日历对象中完成的提醒事项
- (NSPredicate *)predicateForCompletedRemindersWithCompletionDateStarting:(nullable NSDate *)startDate ending:(nullable NSDate *)endDate calendars:(nullable NSArray<EKCalendar *> *)calendars;
```

##### 删除事件
```Objective-C
[eventStore removeEvent:event span:EKSpanFutureEvents error:&removeError];
// 删除日历事件方法，span 看重复规则那一节
// commit 为YES时，马上提交更改，一次处理一个事件。为 NO 时，不会马上提交改动，可以连续提交多个，在调用 - (BOOL)commit:(NSError **)error 方法后提交所有改动。
- (BOOL)removeEvent:(EKEvent *)event span:(EKSpan)span error:(NSError **)error;
- (BOOL)removeEvent:(EKEvent *)event span:(EKSpan)span commit:(BOOL)commit error:(NSError **)error;

// 提醒事项
- (BOOL)removeReminder:(EKReminder *)reminder commit:(BOOL)commit error:(NSError **)error;
```

#### 后言
EventKit 并未提供修改事件的方法，所以你需获取保存事件，删除 EventStore 中该事件，然后再次添加。
