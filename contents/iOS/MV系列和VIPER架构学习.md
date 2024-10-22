---
title: MV系列和VIPER架构学习
date: 2018-11-28 14:08:59
categories: iOS
tags: 
- 架构学习
keywords: MV系列架构,VIPER架构,架构学习
description:
---
由于个人对一些流行的架构的分辨和其各模块的分工很模糊，所以通过网上查询整理关于架构模式的知识，并且根据自己的了解写了一个小[demo](https://github.com/Leafmure/ArchitectureDemo)。
<!-- more -->
### 架构
iOS 下的架构设计常用模式有：MVC、MVP、MVVM以及VIPER，架构设计是将项目中的视图、数据模型、逻辑等进行角色定义，将任务分摊给相对应的角色。

### 为什么要使用架构
当随着业务的增加，项目的复杂度也随之在提升。这个时候会出现某些类中的代码量巨大并且分工模糊，里面的内聚性很高，牵一发而动全身。尤其是当出现bug进行调试的时候，由于分工模糊没办法快速定位bug的发生处，如果bug所处类中复杂度过高，阅读代码也比较耗费时间，一般会出现的情况如下(即使你使用的是Apple的MVC架构)：
1. UIViewController中含有大量的代码：视图的创建、逻辑处理、建立数据模型等
2. 数据在UIViewController中进行获取并读存
3. UIView除了视图的布局不处理其他事情，当然也可能没有UIView类，都在UIViewController中
4. Model仅仅是一个数据结构，只有属性
5. 无法进行单元测试

### 如何选择框架
一个好的架构应该具备的特点：
#### 分布性
虽然当开发工作越多的时候，大脑越能习惯复杂的思维，但是思维能力是有限的，所以我们在弄清一些功能原理时，应遵循“单一功能原则”(一个方法做一件事，一个功能类只做本份的事情)将功能划分给不同的角色实体以保持负载均衡，降低项目复杂度。

#### 易测性
测试可以在增加新特性或者重构后提前发现在运行时会出现的问题，所以易测性可以减少代码的bug出现率。

#### 易用性
易用性没有固定的答案，一般都有个共同点就是维护成本较。代码量少是一种，代码越少，bug就越少，所带来的维护成本也少。

### 常用架构介绍
#### MVC
- Model：数据的结构保存和操作处理
- View：界面UI展示
- Controller：View 和 Model之间的协调器

##### Apple 所推荐的MVC

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/applemvc.webp)

上图所表示的意思：View和Model之间没有任何直接的联系，Controller作为View 和 Model之间的协调器，所以处理逻辑的代码放在Controller。

但是事实是这样的：

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/事实mvc.webp)

由于View混杂在Controller的生命周期中，所以View和ViewController几乎是绑定在一起的。加上处理逻辑代码也编写在Controller中，造成了Controller臃肿，因此也被戏称 “Massive View Controller”。即使可以将一部分逻辑放在Model中处理，但是由于View和Controller的紧密性，对于View的减压毫无办法，之前所说“即使使用的是Apple的MVC架构也会出现原因”便是因为如此。

##### 我们常用的MVC

我们可能会看到这样的代码
```swift
var userCell = tableView.dequeueReusableCellWithIdentifier("identifier") as UserCell
userCell.configureWithUser(user)
```
这里View直接调用了Model，这违反了MVC的原则，但是这样的情况让人不觉得有什么不对的地方，因为向Cell中传递一个Model对象可以避免将Cell的设置代码放在Controller中，减少Controller的体积。因此我们用的MVC便成了下面的示意：

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/常用mvc.webp)

##### MVC特点分析
- 分布性：View和Model确实是分开的，但Controller和View紧密耦合
- 易测性：因为Controller和View紧密耦合需创造性的模拟View以及其生命周期，所以只能对Model测试。
- 易用性：与其他模式相比代码量少，是每个编程人员入门学习的一种架构，所以维护起来也比较容易。

#### MVP

- Model：数据的结构保存和操作处理
- View/Controller：Controller被定义为View，界面UI展示
- Presenter（展示器）：处理逻辑以及View的动作事件，将数据展示到View上

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/mvp.webp)

MVP与Aplle的MVC很像，区别在于：虽然View是和Controller紧密耦合的，但是MVP的协调器Presenter并没有对ViewController的生命周期做任何改变，因此View可以很容易的被模拟出来。在Presenter中根本没有和布局有关的代码，但是它却负责更新View的数据和状态。在MVP中Controller被定义为View，于是View是和Controller紧密耦合的关系便不在是测试的阻碍了，但是因为Presenter的编写降低了一些开发速度。

##### MVP特点分析
- 分布性：将主要任务分摊到Presenter和Model，View的功能很少
- 易测性：因为划分的比较明确，所以测试性非常好
- 易用性：虽然代码量比MVC模式多，但是MVP划分的十分清晰

#### MVVM

- Model：数据的结构保存和操作处理
- View/Controller：Controller被定义为View，界面UI展示
- ViewModel：读取Model，处理逻辑，绑定View，将Model的改变更新到自身并更新View状态

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/mvvm.webp)

MVVM架构模式MV系列中最新的模式，它和MVP很相似，都是将Controller定义为View并且View和Model没有直接联系。ViewModel读取Model数据进行逻辑处理并绑定View，所以Model改变的同时，ViewMode会将Model的改变更新到自身并更新View。

ViewModel和View的绑定有两种选择：
- 基于KVO的绑定库如 RZDataBinding 和 SwiftBond
- 完全的函数响应式编程，比如像ReactiveCocoa、RxSwift或者 PromiseKit

##### MVVM特点分析
- 分布性：MVVM的View要比MVP中的View承担的责任多。因为前者通过ViewModel的设置绑定来更新状态，而后者只监听Presenter的事件但并不会对自己有什么更新
- 易侧性：ViewModel不知道关于View的任何事情，这允许我们可以轻易的测试ViewModel。同时View也可以被测试，但是由于属于UIKit的范畴，对他们的测试通常会被忽略。
- 易用性：代码量和MVP的差不多，但是在实际开发中，我们必须把View中的事件指向ViewModel并且手动的来更新View，如果使用绑定的话，MVVM代码量将会小的多。

#### VIPER
- View：Controller被定义为View，界面UI展示
- Interactor（交互器）：包括关于数据和网络请求的业务逻辑，例如创建一个实体（数据），或者从服务器中获取一些数据。为了实现这些功能，需要使用服务、管理器，但是它们并不被认为是VIPER架构内的模块，而是外部依赖。
- Presenter（展示器）：包含UI层面的业务逻辑以及在交互器层面的方法调用
- Entity（实体）：普通的数据对象，不属于数据访问层次，因为数据访问属于交互器的职责。
- Router（路由器）：用来连接VIPER的各个模块，负责视图的跳转

![image](https://destinmure.github.io/pic/postImage/MV系列和VIPER架构学习/viper.webp)

VIPER和MV系列在任务分摊不同：
- Model（实体） 作为最小的数据结构转换到交互器中
- Controller/Presenter/ViewModel的UI展示方面的职责移到了Presenter中，但是并没有数据转换相关的操作
- VIPER是第一个通过路由器实现明确的地址导航模式

##### VIPER特点分析
- 分布性：任务划分十分清晰
- 易侧性：由于角色划分十分清晰，所以可测试性极佳
- 易用性：由于必须为很小功能的类写出大量的接口，维护成本比较高

### 总结

没有完全适用的架构模式，我们应该根据项目实情来选择架构模式，一个项目中可以根据模块复杂程度选择适用的架构模式，并非一个项目只能用一种架构模式。业务比较简单的模块可以使用MVC模式，如果业务升级了，我们可切换至其他的架构模，这样可以避免架高炮射蚊子。以上都是基于自己目前理解的整理，关于VIPER的Router部分理解还是很模糊，后续了解再更新或调整。
