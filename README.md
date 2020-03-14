# iOS-Interview
> 面试知识点整理

- [UI视图相关](#UI视图相关)
  - [UIView跟CALayer](#UIView跟CALayer)
  - [事件传递与视图响应链](#事件传递与视图响应链)
  - [图像显示原理](#图像显示原理)
  - [UI卡顿掉帧原因](#UI卡顿掉帧原因)
  - [绘制原理&异步绘制](#绘制原理&异步绘制)
  - [离屏渲染](#离屏渲染)
- [内存管理](#内存管理)
  - [内存布局](#内存布局)
  - [内存管理方案](#内存管理方案)
  - [ARC&MRC](#ARC&MRC)
  - [弱引用](#弱引用)
  - [自动释放池](#自动释放池)
- [Objective-C语言特性](#Objective-C语言特性)
  - [分类](#分类)
  - [扩展](#扩展)
  - [代理](#代理)
  - [通知](#通知)
  - [KVO](#KVO)
  - [KVC](#KVC)
- [Block](#Block)
  - [Block介绍](#Block介绍)
  - [Block判空操作](#Block判空操作)
  - [截获变量](#截获变量)
  - [__block修饰符](#__block修饰符)
  - [Block内存管理](#Block内存管理)
  - [Block循环引用](#Block循环引用)
- [多线程](#多线程)
  - [GCD](#GCD)
  - [NSOperation](#NSOperation)
  - [NSThread](#NSThread)
  - [多线程与锁](#多线程与锁)
- [Runtime](#Runtime)
  - [Runtime数据结构](#Runtime数据结构)
  - [对象,类对象,元类对象](#对象,类对象,元类对象)
  - [消息传递](#消息传递)
  - [消息转发](#消息转发)
  - [Method-Swizzing](#Method-Swizzing)
  - [动态添加方法](#动态添加方法)
  - [动态方法解析](#动态方法解析)
   
- [Runloop](#Runloop)
  - [概念](#概念)
  - [Runloop数据结构](#Runloop数据结构)
  - [Runloop与NSTimer](#Runloop与NSTimer)
  - [Runloop与多线程](#Runloop与多线程)
  
  
- [网络](#网络)
  - [HTTP协议](#HTTP协议)
  - [HTTPS与网络安全](#HTTPS与网络安全)
  - [TCP、UDP传输层协议](#TCP、UDP传输层协议)
  - [DNS解析](#DNS解析)
  
  
- [设计模式](#设计模式)
  - [设计原则](#设计原则)
      - [单一职责原则](#单一职责原则)
      - [依赖倒置原则](#依赖倒置原则)
      - [开闭原则](#开闭原则)
      - [里氏替换原则](#里氏替换原则)
      - [接口隔离原则](#接口隔离原则)
      - [迪米特原则](#迪米特原则)
  - [设计模式](#设计模式)
      - [责任链模式](#责任链模式)
      - [桥接模式](#桥接模式)
      - [适配器模式](#适配器模式)
      - [单例模式](#单例模式)
      - [命令模式](#命令模式)
      
- [框架及架构](#框架及架构)
  - [图片缓存框架](#图片缓存框架)
  - [客户端架构](#客户端架构)
      
## UI视图相关

#### UIView跟CALayer
- 单一原则
- UIView为CALayer提供内容以及负责处理触摸事件,参与响应链
- CALayer负责显示内容的Content
#### 事件传递与视图响应链
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
```
###### 事件传递示例图

- 我们点击屏幕产生触摸事件，系统将这个事件加入到一个由UIApplication管理的事件队列中，UIApplication会从消息队列里取事件分发下去，首先传给UIWindow
- 在UIWindow中就会调用hitTest:withEvent:方法去返回一个最终响应的视图
- 在hitTest:withEvent:方法中就回去调用pointInside: withEvent:去判断当前点击的point是否在UIWindow范围内，如果是的话，就会去遍历它的子视图来查找最终响应的子视图
- 遍历的方式是使用倒序的方式来遍历子视图，也就是说最后添加的子视图会最先遍历，在每一个视图中都会去调用它的hitTest:withEvent:方法，可以理解为是一个递归调用
- 最终会返回一个响应视图，如果返回视图有值，那么这个视图就作为最终响应视图，结束整个事件传递；如果没有值，那么就会将UIWindow作为响应者
---
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92.png)

---

###### `hitTest:withEvent:`系统实现示例图

- 首先会判断当前视图的hiden属性、是否可以交互以及透明度是否大于0.01，如果满足条件则进入下一步，否则返回nil
- 调用pointInside: withEvent:方法来判断这个点是否在当前视图范围内，如果满足条件则进入下一步，否则返回nil

- 然后以倒序的方式遍历它的子视图，在每个子视图中去调用hitTest:withEvent:方法，如果有一个子视图返回了一个最终的响应视图，那么就将这个视图返回给调用方；如果全部遍历完成都没有找到一个最终的响应视图，因为点击位置在当前视图范围内，就将当前视图作为最终响应视图返回

---
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/hitTest.png)

---

###### 视图响应者链
- 与事件传递过程相反
- 如果hitTest:withEvent:找到了第一响应者initial view，但是该响应者没有处理该事件，那么事件会沿着响应者链向上传递：第一响应者 -> 父视图 -> 视图控制器，如果传递到最顶级视图还没处理事件，那么就传递给UIWindow去处理，若window对象也不处理那么就交给UIApplication处理，如果UIApplication对象还不处理，就丢弃该事件（但是并不会引起崩溃）

#### 图像显示原理
###### 图像显示原理
- CPU和GPU两个硬件是通过总线链接起来的
- 在CPU中输出的结果是位图，经由总线在合适的时机上传给GPU
- GPU拿到位图之后，会做相应位图的图层渲染，包括文理的合成之后会把结果放到帧缓冲区（Frame Buffer）当中
- 由视频控制器根据VSync信号在指定时间之前去提取在帧缓存区当中的显示内容，最终显示到手机屏幕上

###### UIView显示原理
- 当创建一个UIView控件之后，显示部分是由 CALayer 来负责的
- CALayer当中有一个contents属性，就是我们最终要绘制到屏幕上的位图
- 比如说我们创建的是一个UILable，contents里面最终放置的结果就是关于hello word的文字位图
- 然后系统会在合适的时机回调一个drawRect：方法，在此基础上可以绘制一些自定义想要绘制的内容
- 绘制好的位图，最终会由Core Animation框架提交给GPU部分的OpenGL（ES）渲染管线进行最终的位图的渲染，包括文理的合成，然后显示到屏幕上面


---
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%9B%BE%E5%83%8F%E6%98%BE%E7%A4%BA%E5%8E%9F%E7%90%86.png)

---
#### UI卡顿掉帧原因

- 如果要保持界面滑动流畅那就需要保证60FPS,那么就需要在16.7ms的时间内,CPU和GPU协同完成产生一帧的数据
- 在这个时间内CPU和GPU没有来得及生产出一帧缓冲，那么在下一次`VSnc`信号之前, 那么这一帧会被丢弃，显示器就会保持不变，继续显示上一帧内容，这就将导致导致画面卡顿
---
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%8D%A1%E9%A1%BF%E6%8E%89%E5%B8%A7.png)

---

###### 优化方案
- CPU
  - 对象创建、调整、销毁（可以放在子线程）
  - 预排版（布局计算，文本计算）
  - 预渲染（文本等异步绘制，图片编码等）
- GPU
  - 纹理渲染（避免离屏渲染、CPU异步绘制机制减轻GPU压力）
  - 视图混合（减轻层级复杂度）
#### 绘制原理&异步绘制

###### UIView绘制流程
- 当我们调用[UIView setNeedsDisplay]这个方法时，系统没有立即进行绘制工作，而是立刻调用CALayer的同名方法，并且会在当前layer上打上一个标记，然后会在当前runloop将要结束的时候调用[CALayer display]这个方法，然后进入真正绘制过程
- 在[CALayer display]这个方法的内部实现中会判断这个layer的delegate是否响应displayLayer:这个方法，
  - 不响应，进入到**系统绘制**流程中；
  - 响应，为我们提供**异步绘制**的入口

![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/UIView%E7%BB%98%E5%88%B6%E5%8E%9F%E7%90%86.png)

###### 系统绘制
- 在`CALayer`内部会先创建`backing store`(可以理解为`CGContext`)，一般在`drawRect`:方法中通过上下文堆栈当中取出栈顶的`context`,也就是上下文

- 然后这个`layer`会判断是否有代理
  - 如果没有代理，那么就会调用`[CALayer drawInCotext:]`；
  - 如果有代理，会调用代理的`drawLayer:inContext:`方法，然后做当前视图的绘制工作（这一步是发生在系统内部的），然后在一个合适的时机给与我们`[UIView drawRect:]`方法的回调
- 然后无论是哪个分支，最终都会由`CALayer`上传对应的`backing store`(位图)给`GPU`，然后就结束了系统默认的绘制流程

![CALayer%E7%BB%98%E5%88%B6.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/CALayer%E7%BB%98%E5%88%B6.png)
###### 异步绘制
- 异步绘制需要系统的`[layer.delegate displayLayer:]`,这个过程需要代理负责生成对应的`bitmap`,同时设置`bitmap`为`layer.contents`属性的值
- 假如在某一个时机调用了`[view setNeedsDisplay]`这个方法，系统会在当前`runloop`将要结束的时候调用`[CALyer display]`方法，然后如果我们这个layer的代理实现了`[view displayLayer]`这个方法
- 然后会通过子线程的切换，我们在子线程中去做一个位图的绘制，主线程可以去做一些其他的操作
- 在子线程中第一步先通过`CGBitmapContextCreate()`方法来创建一个位图的上下文，然后我们通过`CoreGraphic API`可以做当前UI控件的一些绘制工作，最后我们再通过`CGBitmapContextCreateImage()`这个函数来根据当前所绘制的上下文来生成一张`CGImage`图片

- 最后回到主线程来提交这个位图，设置`layer`的`contents`属性，这样就完成了一个UI控件的异步绘制过程
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%BC%82%E6%AD%A5%E7%BB%98%E5%88%B6.png)
#### 离屏渲染

###### 含义
> 离屏渲染指的是GPU在当前屏幕缓冲区以外开辟了一个缓冲区进行渲染操作

###### 坏处
- 创建了新的缓冲区
- 上下文频繁切换

###### 原因
- 圆角(和maskToBounds一起使用)
- 图层蒙版(mask)
- 阴影(shadows)
- 光栅化(shouldRasterize)

## 内存管理
#### 内存布局

![内存布局](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4.png)
- 栈(stack) -- ↓
  - 方法调用
- 堆(heap) -- ↑
  - 通过alloc分配的对象
- 未初始化数据(.bss)
  - 未初始化的全局变量 未初始化静态变量
- 已初始化数据(.data)
  - 已初始化的全局变量
- 代码段(.text)
  - 程序代码

#### 内存管理方案

> 不同场景不同管理方案

- **Tagged Poingter** 
  - 64bit开始引入`Tagged Poingter `优化小对象的存储:如`NSNumber`,`NSDate`,`NSString`
  - 在没有使用Tagged Pointer之前， NSNumber等对象需要动态分配内存、维护引用计数等，NSNumber指针存储的是堆中NSNumber对象的地址值
  - 使用Tagged Pointer之后，NSNumber指针里面存储的数据变成了：Tag + Data，也就是将数据直接存储在了指针中
  - 当指针不够存储数据时，才会使用动态分配内存的方式来存储数据
  - objc_msgSend能识别Tagged Pointer，比如NSNumber的intValue方法，直接从指针提取数据，节省了以前的调用开销


- **NONPOINTER_ISA**

  ![NONPOINTER_ISA](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/NONPOINTER_ISA.png)
  - arm64架构下是一个64bit
    - 0 :indexed 0是纯`isa`指针
    - 1 :has_assoc 是否关联对象 
    - 2 :has_cxx_dtor 是否使用过c++
    - 3->15 + 16->31 + 32->35 共33位来表示内存地址
    - 36->41 magic
    - 42 :weakly_referenced 弱引用
    - 43 :deallocating 是否在进行dealloc操作
    - 44 :has_sidetable_rc 当前存储引用计数是否到达上限
    - 45-63 extra_rc 额外引用计数
- **散列表**

> 在64bit中，引用计数可以直接存储在优化过的isa指针中，也可能存储在SideTable类中
> 

  ![SideTables().png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/SideTables().png)

  > 为什么不是一个`SideTable`,而是多个组成的`SideTables`?
  > 

  > 如果是一张大表的话,如果要操作某个对象的引用计数值进行修改,由于对象可能是在不同对象中创建的,所以进行操作时需要加锁操作才能保证数据安全,存在效率问题
  > 
  
    > 解决办法 
  > 

  > 分离锁计数方案,分拆成8个,假如对象A B分别在不同表的话,可以并发进行操作加锁处理
  > 

  > 怎么实现快速分流?通过对象的指针,如何快速定位到其属于哪张`SideTable`表?
  > 

  > `SideTables`本质是一张`Hash`表,里面有64张具体的`SideTable`,存储不同对象的引用计数表和弱引用表,通过对象的指针通过`hash`函数可以得到具体在`SideTables`中具体位置;`f(ptr) = (uintptr_t) % array.count`
  > 

  - `SideTables()`
    - `spinlock_t` 自旋锁
      - `spinlock_t` 是一种忙等的锁
      - 适用于轻量访问

    - `RefcountMap`引用计数表
      - `hash`表
      - `size_t`代表对象的引用计数值,需向右偏移俩位来计算
      ![RefcountMap.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/RefcountMap.png)
    - `weak_table_t`弱引用表
      - `hash`表
      ![%E5%BC%B1%E5%BC%95%E7%94%A8%E8%A1%A8.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%BC%B1%E5%BC%95%E7%94%A8%E8%A1%A8.png)

#### ARC&MRC

- MRC
  - **`alloc`**
    - 经过一系列的调用,最终调用了C函数的`calloc`
    - 此时并没有设置引用计数为1
  - **`retain`**
    - **`SideTable &table = SideTables()[this]`** //通过当前对象的指针,通过`hash`函数计算在`SideTables`找到对应的`sizeTable`
    - **`size_t &refcntStorage = table.refcnts[this]`**
    - **`refcntStorage += SIDE_TABLE_RC_ONE`**  // 此处加1,并不是实际的1,而是一个偏移量,实际是4
  - **`release`**
    - **`SideTable &table = SideTables()[this]`** //通过当前对象的指针,通过hash函数计算在SideTables找到对应的sizeTable
    - **`RefcountMap::iterator it = table.fefcnts.find(this)`**
    - **`refcntStorage -= SIDE_TABLE_RC_ONE`**
  - **`retainCount`****
    - **`SideTable & t**able = SideTables()[this]` **
    - **`sizt_t refcnt_result = 1`****
    - **`RefcountMap::iterator it = table.fefcnts.find(this)`**
    - **`refcnt_result += it->secont >> SIDE_TABLE_RC_SHIFT`
  - **`dealloc`**

    ![dealloc.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/dealloc.png)
- ARC
  - `ARC`编译器在对应的位置为我们插入`retain`,`release`操作;
  - `ARC`是`LLVM`和`Runtime`协作的结果
  - 禁止手动调用`retain`,`release`,`retainCount`,`dealloc`
  - `weak`,`strong`

#### 弱引用

> `id __weak obj1 = obj;` -- > `objc_initWeak(&obj1,obj)`
> 
![%E6%B7%BB%E5%8A%A0weak%E5%8F%98%E9%87%8F.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%B7%BB%E5%8A%A0weak%E5%8F%98%E9%87%8F.png)
>  一个被声明为__weak对象指针,会经过如下方法 添加
- `objc_initWeak()`
  - 传入了弱引用变量的地址`&obj1`和被修饰的弱引用变量`obj`
- `storeWeak()`
- `weak_register_no_lock()`

> 问:系统是怎样把一个weak变量添加到他所对应的的弱引用表中的?

> 答:一个被声明为__weak的对象指针,经过编译器编译之后,会调用`objc_initWeak()`经过一系列函数调用栈,最终在`weak_register_no_lock()`中进行弱引用变量添加,添加的位置是通过一个hash算法进行位置查找,如果查找位置有当前对象的所对应的的弱引用数组,那么就将新弱引用变量加到数组里面,如果没有,就创建一个弱引用数组,将第0个设置为weak,其他位置置为nil


> 问:当一个weak释放时,怎么置为nil?

> 答:当一个对象被`dealloc`之后,在`dealloc`内部实现当中会调用`weak_clear_no_lock()`,在函数实现当中会根据当前对象的指针查找所对应的的弱引用表,把当前对象所对应的弱引用拿出来是一个数组,遍历置为nil
#### 自动释放池

- 是以栈为结点通过双向链表的形式组合而成的
- 是和线程一一对应的
> 问: `AutoreleasePool`实现原理

> 在当次runloop将要结束的时候调用`AutoreleasePoolPage::pop()`

> 问: `AutoreleasePool`嵌套使用

> 多次嵌套就是多次插入哨兵对象


> 问: 什么情况下使用

> 在for循环中alloc图片等数据消耗较大的场景手动插入autoreleasepool
> 

- `@autoreleasepool{}` 编译后转换为

  ```
   void *ctx = objc_autoreleasePoolPush();
   { 代码 };
   objc_autoreleasePoolPop(ctx);
  ```
  - `objc_autoreleasePoolPush()`
    - 内部会调用`c++`函数`AutoreleasePoolPage::push(void)`
  - `objc_autoreleasePoolPop(ctx)`
    - 内部会调用`c++`函数`AutoreleasePoolPage::pop(ctx)`
    - 一次pop相当于一次批量的pop操作
- `AutoreleasePoolPage`的数据结构

  ```
  class AutoreleasePoolPage{
    id *next;
    AutoreleasePoolPage *const parent;
    AutoreleasePoolPage *child;
    pthread_t const thread;
  }
  ```
  - `id *next` 用来作为游标指向栈顶添加新的`autoreleasepool`对象的下一个位置
  - `parent`和`child`双向链表
  - `thread` 和线程一一相关
  - 每个AutoreleasePoolPage对象占用4096字节内存，除了用来存放它内部的成员变量，剩下的空间用来存放autorelease对象的地址
  - 当当前的`AutoreleasePoolPage`页再加入一个`autoreleasepool`对象就要满时(next指针马上指向栈顶时),新建立一个下一页的`page`对象,与当前页链表链接完成后,新`page`页的`next`指针会被初始化在栈底,然后继续向栈顶添加对象
  - 每当进行`push`操作,runtime会向当前的`page`add一个哨兵对象nil,objc_autoreleasePoolPush的返回值正是这个哨兵对象的地址，被objc_autoreleasePoolPop(哨兵对象)作为入参:
  - 根据传入的哨兵对象地址找到哨兵对象所处的page
  - 在当前page中，将晚于哨兵对象插入的所有autorelease对象都发送一次- release消息，并向回移动next指针到正确位置

#### 循环引用

- 自循环引用
- 相互循环引用
- 多循环引用

- __weak
- __block
  - MRC下,不会增加引用计数,避免循环引用
  - ARC下,会被强引用,需要手动解环
- __unsafe_unretained
  - 修饰对象不会增加引用计数,避免循环引用
  - 会产生悬垂指针
- NSTimer的循环引用问题




## Objective-C语言特性

#### 分类
###### 分类做了什么事情
- 声明私有方法
- 分解体积庞大的类文件
- Framework私有方法公开化

###### 分类的特点
- 运行时决议 编译的时候宿主类没有分类的方法，运行时runtime才真实的添加到宿主类
- 可以为系统类添加分类 
 
###### 分类中都可以添加
- 实例方法
- 类方法
- 协议
- 属性 分类中定义的属性只生成set get方法 并没有实例变量
 
###### 分类的结构体
  ```
  struct category_t {
    const char *name;  // 分类的名称 如Person+Student
    classref_t cls;    // 宿主类 如Person   
    struct method_list_t *instanceMethods; // 实例方法列表
    struct method_list_t *classMethods;    // 类方法列表
    struct protocol_list_t *protocols;     // 协议列表
    struct property_list_t *instanceProperties;  // 实例属性列表
    
    method_list_t *methodsForMeta(bool isMeta){
        if(isMeta) return classMethods;
        else return instanceMethods;
    }
    property_list_t *propertiesForMeta(bool isMeta){
        if(isMeta) return nil;
        eles return instanceProperties;
    }
  }
  ```
###### 分类的调用栈

![%E5%88%86%E7%B1%BB%E8%B0%83%E7%94%A8%E6%A0%88.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%88%86%E7%B1%BB%E8%B0%83%E7%94%A8%E6%A0%88.png)

- `remethodizeClass`
> **将传入的宿主类(假设为`Person`类)判断是否为元类对象(取决于添加的方法是实例方法还是类方法),然后取出宿主类中为完成整合的所有分类(假设有`Person+A`,`Person+B`,`Person+C`等),拼接到宿主类上**
> 
  ```
    static void remethodizeClass (Class cls){
        category_list *cats;
        bool isMeta;
        runtimeLock.assertWriting();
        // 判断当前类是否为元类对象,取决于添加的方法是实例方法还是类方法
        // 以实例方法添加逻辑为例,所以假设isMeta = NO;
        isMeta = cls->isMetaClass();
        // 获取cls中未完成整合的所有分类
        if((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))){
            if(PrintConnecting){

            }
            // 将分类cats拼接到cls上
            attachCategories(cls, cats, true);
            free(cats);
        }
    }
  ```
- `attachCategories`   

> **传入的分类的数组`cats`是按照顺序编译的,假设为`Person+A`和`Person+B`;在这个方法中会根据`cats`数组总数,然后进行倒序遍历,最先访问最后编译的分类,先访问`Person+B`,将其方法列表协议列表等添加到新的二维数组中,当调用方法时,会先查找新的二维数组中的`Person+B`的对应的方法,这也就解释了分类方法覆盖的原因.  接下来会根据传入的宿主类的名字来获取宿主类的`rw`数据,此时将刚才的包含分类的二维数组拼接到宿主类中**
  ```
    static void attachCategories (Class cls, category_list *cats, bool flush_caches){
        // 非空判断
        if(!cats) return;
        // 打印相关 可忽略
        if(PrintReplacedMethods) printReplacements(cls, cats);
        // 以实例方法添加逻辑为例,所以假设isMeta = NO;
        bool isMeta = cls->isMetaClass();
        /*
         二维数组
         [[method_t,method_t],[method_t],[method_t,method_t,method_t],...]
         */
        method_list_t **mlists = (method_list_t*)malloc(cats->count *sizeof(*mlists));
        property_list_t **proplists = (property_list_t*)malloc(cats->count *sizeof(*proplists));
        protocol_list_t **protolists = (protocol_list_t*)malloc(cats->count *sizeof(*protolists));
        int mcount = 0;
        int propcount = 0;
        int protocount = 0;
        int i = cats->count; //宿主类分类的总数
        bool fromBundle = NO;
        while(i--){ // 倒叙遍历,最先访问最后编译的分类
            // 获取一个分类
            auto& entry = cats-list[i];
            // 获取该分类的方法列表
            method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
            if(mlist){
                // 最后编译的分类最先添加的分类数组里面
                mlists[mcount++] = mlist;
                fromBoundle |= entry.hi->isBoundle();
            }
            // 属性列表同方法列表规则一样
            property_list_t *proplist = entry.cat->methodsForMeta(isMeta);
            if(proplist){
                proplists[propcount++] = proplist;
            }
            // 协议列表同方法列表规则一样
            protocol_list_t *protolist = entry.cat->protocols;
            if(protolist){
                protolists[propcount++] = protolist;
            }
        }
        // 获取当前宿主类当中的rw数据,其中包含宿主类的方法列表信息
        auto rw = cls->data();
        // 主要针对 分类中关于内存管理相关方法下 一些特殊处理 (自己实现set方法)
        prepareMethodLists(cls, mlists, mcount, NO, fromBundle)
        /*
         rw 代表类
         methods 代表类的方法列表
         attachLists 将含有mcount个元素的mlist拼接到rw的methods上
         */
        rw->methods.attachLists(mlists,mcount);
        free(mlists);
        if(flush_caches && mcount > 0) flushCaches(cls);
        rw->properties.attachLists(proplists,propcount);
        free(proplists);
        rw->protocols.attachLists(protolists,protocount);
        free(protolists);
    }
  ```
- `attachLists`  

> **根据传入的包含分类的二维数组个数(假设是A,B,C)和原来的个数(假设是X,Y),重新分配(增大)内存(5),然后将原来的元素进行内存移动,将原来的移动到尾部,将传入的二维数组拷贝到对应的位置,结果由原来的`[X,Y]`变为`[A,B,C,X,Y]`, 这就是分类会"覆盖"宿主类方法的原因**
  ```
  void attachLists (list *const *addedLists, unit32_t addedCount){
    if (addedCount == 0) return;
    if (hasArray()) {
        // 列表中原有元素总数  oldCount = 2
        unit32_t oldCount = array()->count;
        // 拼接之后的元素总数
        unit32_t newCount = oldCount + addedCount;
        // 根据新总数重新分配内存
        setArray((array_t *)realloc(array(),array_t::byteSize(newCount)));
        // 重新设置元素总数
        array()->count = newCount;
        /*
         内存移动
         [[],[],[],[原来的第一个元素],[原来的第二个元素]]
         */
        memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
        /*
         内存拷贝
         [
            A ----> [addedLists中的第一个元素],
            B ----> [addedLists中的第二个元素],
            C ----> [addedLists中的第三个元素],
            [原来的第一个元素],
            [原来的第二个元素]
         ]
         这就是分类会"覆盖"宿主类方法的原因
         宿主类的方法还存在m,由于消息发送机制,根据selecter选择器根据名字查找,一旦找到,就返回imp,分类位于数组前面,所以会覆盖
        */
        memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
    }
  }
  ```
  
###### 结论
- 分类添加的方法会"覆盖"原类方法
- 同名分类方法谁能生效取决于编译顺序
- 名字相同的分类会引起编译报错 

#### 关联对象

###### 给分类添加`成员变量`

- 获取关联对象
  - **`id objc_getAssociatedObject(id object, const void *key)`**
- 设置关联对象 
  - **`void objc_setAssociatedObject(id object, const void *key,id value,objc_AssociationPolicy policy)`**
- 删除关联对象
  - **`void objc_removeAssociatedObjects(id object)`**

###### 位置
- 成员变量添加的位置并未添加到原宿主数组
- 关联对象由`AssociationsManager`管理在`AssociationHashMap`中
- 所有对象的关联内容都在一个相同的全局容器中
![%E5%85%B3%E8%81%94%E5%AF%B9%E8%B1%A1.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%85%B3%E8%81%94%E5%AF%B9%E8%B1%A1.png)

###### 源码分析

> 以设置关联对象方法为例 
> - 获取`AssociationsManager`管理的全局容器`AssociationHashMap`
> - 将传入的`value`和`policy`处理为`new_value`
> - 根据传入的要被关联的对象`object`,对其指针地址按位取反;作为全局容器`AssociationHashMap`中的某一对象的`object_key`
> - 根据这个`object_key`查找对应的`ObjectAssociationMap`结构的`map`,
>   - 找的到`map`,根据传入的`key`获取`ObjectAssociationMap`中的`ObjectAssociation`对象
>     - 如果有`ObjectAssociation`对象,将`new_value`覆盖值
>     - 如果没有`ObjectAssociation`对象,将传入的`value`和`policy`封装成一个`ObjectAssociation`对象,封住的对象作为新创建的`ObjectAssociationMap`对应`key(传入的)`的`value`
>   - 找不到`map`,创建一个`ObjectAssociationMap`,作为全局容器`AssociationHashMap`对应这个`object_key`的`ObjectAssociationMap_value`,然后将传入的`value`和`policy`封装成一个`ObjectAssociation`对象,封住的对象作为新创建的`ObjectAssociationMap`对应`key(传入的)`的`value`
> 
```
/// 关联对象的最终方法,参数是透传过来的
/// @param object 要被关联的对象 比如要给Person的分类Person+A关联,那么object就是Person+A
/// @param key  要关联的值的对应的key
/// @param value 要给Person+A关联的值
/// @param policy 内存管理策略 copy retain等
void _objc_set_associative_reference(id object, void *key, id value, uintptr_t policy){
    ObjcAssociation old_association(0, nil);
    // 根据策略值来对value进行加工
    id new_value = value ? acquireValue(value, policy) : nil;
    // 关联对象管理类 C++实现的一个类 主要管理的是AssociationHashMap
    AssociationsManager manager;
    // 获取manager维护的一个HashMap,是一个全局的容器
    AssociationHashMap &associations(manager.associations());
    // 对object对象指针地址按位取反来作为全局容器中的某一对象的唯一标识
    disguised_ptr_t disguised_object = DISGUISE(object);
    if (new_value) {
        // 根据对象的指针查找一个对应的ObjectAssociationMap结构的map
        AssociationHashMap::iterator i = associations.find(disguised_object)
        // 关联过
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                // 如果有值就替换
                old_association = j->second;
                j->seceond = ObjectAssociation(policy ,new_value);
            }else{
                // 没值就设置
                (*refs)[key] = ObjectAssociation(policy ,new_value);
            }
        }else{
            // 没找到 需要创建一个新的
            ObjectAssociationMap *refs = new ObjectAssociationMap;
            associations[disguised_object] = refs;
            (*refs)[key] = ObjectAssociation(policy ,new_value);
            // 设置有了关联对象的标识
            object->setHasAssociatedObjects();
        }
    }else{
        AssociationHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                old_association = j->second;
                refs->erase(j);
            }
        }
    }
}
```
###### 生成的数据结构
```
// 全局容器
{    //A分类添加的实例变量
    "0x4927298472":{                ----key:被关联的对象A        value:ObjectAssociationMap
        "@selector(text)":{         ----key:被关联对象传入的key   value:ObjectAssociation   @selector(text)---->@property (retain) NSString *text;
            "value":"hellow",
            "policy":"retain"
        },
        "@selector(title)":{      @selector(title)---->@property (copy) NSString *title;
            "value":"a object",
            "policy":"copy"
        }
    },
    //B分类添加的实例变量
    "0x3428154292": {
        "@selector(backgroundColor)": {
            "value":"oxff55ff",
            "policy":"retain"
        }
    }
    
}
```
#### 扩展
###### 扩展用法
- 声明私有属性
- 声明私有方法
- 声明私有成员变量
###### 分类扩展区别
- 扩展特点:
  - 编译时决议,分类是运行时决议
  - 只有声明,没有实现,多数情况下寄生于宿主的.m中
  - 不能为系统类添加扩展

#### 代理
###### 含义
- 是一种软件设计模式,代理模式
- iOS当中以@protocol形式实现
- 传递方式一对一

###### 遇到的问题
- 一般声明为weak以规避循环引用

#### 通知
###### 含义
- 使用观察者模式来实现的用于跨层传递消息的机制
- 传递方式为一对多

###### 实现机制
- 创建一个类`NSNotificationCenter`,维护一个表`Notification_Map`,`key`为`notificationName`,`value`为`Observers_List`
- `Observers_List`里面应该有`Observer`及 接受 发送通知的相关方法
#### KVO
###### 什么是KVO
- KVO是Objective-C对观察者模式的又一实现
- 使用了isa混写(isa-swizzling)技术实现KVO
- KVC设置value能生效
- 通过成员变量直接赋值不生效,手动添加KVO才会生效

#### KVC
- Key-value coding缩写
- 破坏了面向对象的编程思想

#### 属性关键字

- 读写性
- 原子性
- 内存管理


## Runtime

#### Runtime数据结构
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E7%B1%BB%E5%92%8C%E5%AF%B9%E8%B1%A1%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)
###### **objc_object**

- `id` 类型
- `isa_t` 共用体
  - 指针型`isa` : `isa`64位的值代表`Class`地址
  - 非指针型`isa`:  `isa`64位的部分值代表`Class`地址
- 关于`isa`操作相关方法
  - 对象指向类对象: 实例------>Class
  - 类对象指向元类对象: Class------>MetaClass
- 弱引用相关
- 关联对象相关
- 内存管理

###### **objc_class**

- `Class`类型,继承于`objc_object`
- `Class` `superClass`
- `cache_t` `cache` 方法缓存
  - 特点:
    - 用于**快速**查找方法执行函数
    - 可**增量扩展**的**哈希表**结构
    - 是**局部性原理**的最佳应用
  - 结构
    - `bucket_t`
      - `key`---->`selector`
      - `IMP`---->无类型的函数指针
- `class_data_bits_t` `bits`
    - `class_data_bits_t`是对 `class_rw_t`的封装
    - `class_rw_t`代表了类相关的读写信息,是对`class_ro_t`的封装
      - `class_ro_t`
      - `protocols` 二维数组
      - `properties` 二维数组
      - `methods` 二维数组
    - `class_ro_t`类的只读信息
      - `name` 类名 `NSClassFromString(aClassName)`
      - `ivars` 成员变量 --- 一维数组
      - `properties` 属性 --- 一维数组
      - `protocols` 遵循的协议 --- 一维数组
      - `methodList` 方法 --- 一维数组

- `method_t` 
  - `SEL name` 函数名称
  - `const char *types`  函数返回值和参数
  - `IMP imp`  函数体

- `Type Encodings`
  - `const char *types`  返回值+参数1+参数2+....
  - `-(void)aMethod`-->`v@:`-->`void`+`id`+`SEL`

#### **对象,类对象,元类对象**
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%AF%B9%E8%B1%A1.png)

#### **消息传递**

- `void objc_msgSend(id self, Sel op, ...)`
- `void objc_msgSendSuper(struct objc_super *super, Sel op, ...)`

![消息传递](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%B5%81%E7%A8%8B.png)

###### **缓存查找**

> 可以理解为: 给定值是SEL,目标值是对应的`bucket_t`中的`IMP`
- 使用的是哈希查找
- 根据给定的`SEL`通过一个函数来映射出对应的`bucket_t`在数组当中的位置
- `f(key)`就是`SEL`跟`mask`做`&`操作,而`mask`也是`cache_t`的一个成员变量
![缓存查找](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E7%BC%93%E5%AD%98%E6%9F%A5%E6%89%BE.png)

###### **当前类中查找**

- 对于`已排序好的`列表,采用`二分查找法`查找方法相应的执行函数
- 对于`未排序的`列表,采用`一般遍历法`查找方法相应的执行函数

###### **父类中查找**

- 如图

![父类查找](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E7%88%B6%E7%B1%BB%E6%9F%A5%E6%89%BE.png)

#### **消息转发**


![消息转发](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%B5%81%E7%A8%8B.png)
- `+ (BOOL)resolveInstanceMethod:(SEL)sel`
  - 告诉系统要不要解决当前传入的方法的实现
- `- (id)forwardingTargetForSelector:(SEL)aSelector`
  - 告诉系统这次的方法调用应该由哪个对象来处理
- `- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`和`forwardInvocation`
  - 是否返回方法签名,进而处理

#### **Method-Swizzling**

![交换方法](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%96%B9%E6%B3%95%E4%BA%A4%E6%8D%A2.png)
#### **动态添加方法**

- 添加时机是在`resolveInstanceMethod`方法中添加
- 添加方法是 `class_addMethod(Class cls, SEL name, IMP imp,
    const char *types)`
- `performSelector: `

#### **动态方法解析**

- `@dynamic`
  - 动态运行时语言将函数决议推迟到运行时
  - 编译时语言在编译器进行函数决议


## Block

#### Block介绍
- Block是将`函数`及其`执行上下文`封装起来的`对象`

怎么样来理解这段话呢,我们通过源码来进行分析

- 创建一个`MCBlock`文件,在其`.m`中写如下代码

  ```
  - (void) method {
      int multiplier = 6;
      int(^Block)(int) = ^int(int num){
          return num *multiplier;
      };
      Block(2);
  }
  ```
- `clang -rewrite-objc MCBlock.m` 通过clang编译器编辑生成`MCBlock.cpp`文件

  ```
   // _I_MCBlock_method I代表是当前类`MCBlock`类的实例方法
   static void _I_MCBlock_method(MCBlock * self, SEL _cmd) {
      int multiplier = 6;
      int(*Block)(int) = ((int (*)      (int))&__MCBlock__method_block_impl_0((void *)__MCBlock__method_block_func_0, &__MCBlock__method_block_desc_0_DATA, multiplier));
      ((int (*)(__block_impl *, int))((__block_impl *)Block)->FuncPtr)((__block_impl *)Block, 2);
  }
    ``` 

- `__MCBlock__method_block_impl_0`

  ```
  struct __MCBlock__method_block_impl_0 {
    struct __block_impl impl;
    struct __MCBlock__method_block_desc_0* Desc;
    int multiplier;
    __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int _multiplier, int flags=0) : multiplier(_multiplier) {
      impl.isa = &_NSConcreteStackBlock;
      impl.Flags = flags;
      impl.FuncPtr = fp;
      Desc = desc;
    }
  };
  ```
- `_block_impl`

    ```struct __block_impl {
      void *isa;
      int Flags;
      int Reserved;
      void *FuncPtr;
    };
    ``` 

- `__MCBlock__method_block_func_0`

    ```
    static int __MCBlock__method_block_func_0(struct __MCBlock__method_block_impl_0 *__cself, int num) {
      int multiplier = __cself->multiplier; // bound by copy
      return num *multiplier;
    }
    ```
    
##### Block判空操作
> `Block`执行时需判空,否则`Crash`
> 
- 定义一个`Block`时,底层会将其转为`_block_impl`的结构体指针,而其成员结构上面我们也可以看到;当`block()`调用时,底层为函数的地址调用`block->FuncPtr`,去访问`FuncPtr`地址,而由于这个地址是无效的,所以会抛出异常

> 异常的`EXC_BADACCESS`,地址`address`是多少呢?
> 
- `EXC_BADACCESS(code=1,address=0x10)`,是因为这个`_block_impl`数据结构为:
  - 占8个字节的`isa`指针
  - 占4个字节的`Flags`int
  - 占4个字节的`Reserved`int

偏移量刚好为16所对应的`block`调用`FuncPtr`

#### 截获变量
- 局部变量(基本数据类型) 
  - 截获其值
  
  ```
  - (void) method {
      int multiplier = 6;
      int(^Block)(int) = ^int(int num){
          return num *multiplier;
      };
      multiplier = 4;
      Block(2);
  }
  // 12
  ```
- 局部变量(对象类型)    
  - 连同其所有权修饰符一起截获
- 静态局部变量
  - 指针形式

  ```
  - (void) method {
      static int multiplier = 6;
      int(^Block)(int) = ^int(int num){
          return num *multiplier;
      };
      multiplier = 4;
      Block(2);
  }
  // 8
  ```
- 全局变量
  - 不截获
- 静态全局变量
  - 不截获

**源码分析**
```
// 全局变量
int global_var = 4;
// 静态全局变量
static int static_global_var = 5;

- (void)method{
    // 基本数据类型局部变量
    int var = 1;
    // 对象类型局部变量
    __unsafe_unretained id unsafe_obj = nil;
    __strong id strong_obj = nil;
    // 局部静态变量
    static int static_var = 3;
    
    void(^Block)(void) = ^{
        NSLog(@"局部变量<基本数据类型> var %d",var);
        NSLog(@"局部变量<__unsafe_unretained 对象类型> var %@",unsafe_obj);
        NSLog(@"局部变量<__strong 对象类型> var %@",strong_obj);
        NSLog(@"局部 静态变量 %d",static_var);
        NSLog(@"全局变量 %d",global_var);
        NSLog(@"静态全局变量 %d",static_global_var);
    };
    Block();
}
```
**clang**后
```
int global_var = 4;

static int static_global_var = 5;

struct __MCBlock__method_block_impl_0 {
  struct __block_impl impl;
  struct __MCBlock__method_block_desc_0* Desc;
  // 截获局部变量的值
  int var;
  // 连同所有权修饰符一起截获
  __unsafe_unretained id unsafe_obj;
  __strong id strong_obj;
  // 以指针的形式截获局部变量
  int *static_var;
  // 全局变量,全局静态变量不截获
  __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int _var, __unsafe_unretained id _unsafe_obj, __strong id _strong_obj, int *_static_var, int flags=0) : var(_var), unsafe_obj(_unsafe_obj), strong_obj(_strong_obj), static_var(_static_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
#### __block修饰符

###### 使用场景 
- 一般情况下,对被截获的变量进行`赋值`操作的时候需要加`__block`修饰符

  ```
  __block NSMutableArray *array = nil;
  void(^Block)(void) = ^{
    array = [NSMutableArray array]; // 赋值
  } ;
  Block();
  ```
- 使用情况下不需要

  ```
  NSMutableArray *array = [NSMutableArray array];
  void(^Block)(void) = ^{
    [array addObject:@123]; // 使用 
  } ;
  Block();
  ```
  
###### 做了什么

- 使用`__block`修饰符的变量变成了对象
  ```
  - (void) method {
      __block int multiplier = 6;
      int(^Block)(int) = ^int(int num){
          return num *multiplier;
      };
      multiplier = 4;
      Block(2);
  }
  // 8
  ```
  
  ```
   Struct _Block_byref_multiplier_0{
      void *_isa;
      _Block_byref_multiplier_0 *_forwarding;
      int _flags;
      int size;
      int multiplier;
   }
  ```  
  `multiplier = 4;` `==>` `(multiplier.__forwarding ->multiplier) = 4;`
  这一步操作,`multiplier`已经变成了对象,通过`multiplier`对象中同类型的`__forwarding`找到对象(由于这是栈上的block,此时__forwarding指针指向的是自己),进行赋值操作
  

#### Block内存管理
###### 种类
- 栈 `_NSConcreteStackBlock`
- 堆 `_NSConcreteMallocBlock`
- 全局 `_NSConcreteGlobalBlock`
###### `copy`操作
- 栈 堆上生成block
- 堆 什么都不做
- 全局 增加引用计数

###### `__forwarding`指针作用

一个在栈上,被`__block`修饰的变量,经过`copy`操作,`__forwarding`指向了堆上的`__block`变量,而堆上的`__block`变量中的`__forwarding`指向自己

```
{
    __block int multiplier = 10;
    _blk = ^int(int num){
        return num * multiplier;
    };
    multiplier = 6;
    [self executeBlock];
}
```
```
  - (void)executeBlock{
      int result = _blk(4);
      NSLog(@"%d",result);
  }
  
  // 24;
```
- 栈上创建了局部变量`multiplier`,并且用`__block`修饰,此时`multiplier`变成了一个对象
- `_blk`变量作为当前对象的一个成员变量,当对其进行赋值操作的时候,会对其进行`copy`操作,此时堆上会生成一个`_blk`副本
- `multiplier = 6;`不是对`multiplier`变量进行赋值,而是通过`multiplier`对象`__forwarding`指针对其成员变量`multiplier`进行赋值
- 当第一段代码执行完后: 如果`_blk`没有进行`copy`操作,`multiplier = 6;`修改的就是栈上对应的`__block`修饰的变量
- 如果`_blk`进行`copy`操作,`multiplier = 6;`通过栈上的`multiplier`的`__farwarding`指针找到堆上的`__block`变量对应的副本进行修改
- 第二段代码调用堆上的`_blk`,入参是4,所以是24


#### Block循环引用

- 自循环

  ```
   {
      _array = [NSMutableArray arrayWithObjects:@"block", nil];
      _strBlk = ^NSString *(NSString *num){
          return [NSString stringWithFormat:@"hello %@",_array[0]];
      };
      _strBlk(@"hello");
  }
  ```
  - `_strBlk`作为当前对象的一个成员变量,一般copy修饰,所以`self`持有`_strBlk`
  - `_strBlk`表达式中使用了`_array`,由于`Block`截获变量时将其权限修饰符的特性,所以`_strBlk`中有一个`__strong`指针的`self`;
  - `__weak`解决: `__weak NSArray *weakArray = _array`
- 大环循环

  ```
  {
    __block MCBlock *blockSelf = self;
    _blk = ^int(int num){
        return num *blockSelf.var;
    };
    _blk(3);
  }
  ```
  - `MRC`下不会产生循环引用
  - `ARC`下会产生循环引用,引起内存泄露
    - 对象持有`_blk`
    - `blk`持有`__block`变量
    - `__block`变量持有原对象(__farwarding指针指向自己)
    - 解决办法如下,但是有一个弊端,就是如果不执行这段代码的话,闭环永远存在
    ```
    {
      __block MCBlock *blockSelf = self;
      _blk = ^int(int num){
          int result = num *blockSelf.var;
          blockSelf = nil;
          return result;
      };
      _blk(3);
    }
    ```

## 多线程

#### GCD
###### 同步 异步 串行 并发
- 组合1 同步分派任务到串行队列
  > `dispatch_sync(serial_queue,^{//任务})`
  > 
    ```
    // 头条面试题  死锁
    - (void)viewDidLoad{
        dispatch_sync(dispatch_get_main_queue(),^{
        [self doSomething];
        }); 
    }
    // 原因:队列引起的循环等待
    ```
     ```
    // 改为异步
    - (void)viewDidLoad{
        dispatch_async(dispatch_get_main_queue(),^{
        [self doSomething];
        }); 
    }
    // 原因:队列引起的循环等待 主队列不管同步还是异步方式添加都要在主线程中进行处理
    ```
     ```
    // 改为自定义串行队列
    - (void)viewDidLoad{
        dispatch_sync(serialQueue,^{
        [self doSomething];
        }); 
    }
    // 正常运行
    ```
- 组合2 异步分派任务到串行队列
  > `dispatch_async(serial_queue,^{//任务})`

    ```
    - (void)viewDidLoad{
        dispatch_async(dispatch_get_main_queue(),^{
          [self doSomething];// 刷新UI
        }); 
    }
    ```
- 组合3 同步分派任务到并发队列
  > `dispatch_sync(concurrent_queue,^{//任务})`
  
    ```
    // 美团面试
    - (void)viewDidLoad{
        NSLog(@"1");
        dispatch_sync(global_queue,^{
          NSLog(@"2");
            dispatch_sync(global_queue,^{
              NSLog(@"3");
          }); 
          NSLog(@"4");
        }); 
        NSLog(@"5");
    }
    // 12345
    ```
- 组合4 异步分派任务到并发队列
  > `dispatch_async(concurrent_queue,^{//任务})`

    ```
    // 腾讯
     - (void)viewDidLoad{
        dispatch_async(global_queue,^{
          NSLog(@"1");
          [self performSelector:@selector(printLog) withObject:nil afterDelay:0]; 
          NSLog(@"3");
        }); 
    }
    -(void)printLog{NSLog(@"2");}
    // 13
    // 异步方式分派到global_queue队列,子线程执行没有开启runloop
    ```
###### dispatch_barrier_async() 
- 用来实现多读单写
###### dispatch_group_t

#### NSOperation
- 优点
  - 添加任务依赖
  - 任务执行状态的控制
    - isReady
    - isExecuting
    - isFinished
    - isCancelled
    - 重写了`main`方法,底层控制变更任务执行完成的状态及任务退出状态
    - 重写了`start`方法,自行控制任务状态
  - 最大并发量

- 面试题 
  - 系统是怎么样移除一个isFinished = YES的NSOperation? 通过KVO
#### NSThread
###### 启动流程
- start()
- 创建pthread
- main()
- [target performSelector:selector]
- exit()
###### 常驻线程
- 见runloop
#### 多线程与锁

###### @synchronized
- 创建单例对象使用,保证多线程环境下创建的对象是唯一的
###### atomic
- 属性关键字 对被修饰的对象进行原子操作(不负责使用)
  - @property(atomic)NSMutableArray *arr;
  - self.array = [NSMutableArray array];  可以
  - [self.array addObject:obj]  不可以
###### OSSpinLock 
- 自旋锁
- 循环等待访问,不释放当前资源
- 用于轻量级数据访问,例如简单的int值的+1/-1操作,系统源码层面引用计数简单的+1/-1操作
###### NSRecursiveLock

###### NSLock
- 面试题(蚂蚁金服)
  ```
  - (void)methodA{
      [lock lock];
      [self methodB];
      [lock unlock];
  }

  - (void)methodB{
      [lock lock];
      // 操作逻辑
      [lock unlock];
  }
  
  // 死锁
  // 某一线程调取lock方法获取到这个锁,然后又对同一把锁再次调用`lock`方法,已经获取到锁,再次去获取锁,造成死锁
  // 使用递归锁
  ```
###### dispatch_semaphore_t

- dispatch_semaphore_create(1)
  - 源码

  ```
  struct semaphore{
    int value;
    List<thread>;
  }
  ```
- dispatch_semaphore_wait(semaphore,DISPATCH_TIME_FOREVER);
  -  源码
  ```
  {
    S.value = S.value - 1;
    if S.value < 0 then Block(S.list);阻塞是一个主动行为
  }
  ``` 

- dispatch_semaphore_signal(semaphore)
  -  源码
  ```
  {
    S.value = S.value + 1;
    if S.value <= 0 then wakeup(S.list);唤醒是一个被动行为
  }
  ``` 
## Runloop

#### 概念
> Runloop是通过内部维护的`事件循环`来对`事件/消息进行管理`的一个对象
> 
- 没有消息需要处理时,休眠以避免资源占用
  - 用户态 --> 内核态

- 有消息需要处理时,立刻被唤醒
  - 内核态 --> 用户态

#### Runloop数据结构
> NSRunloop是对CFRunloop的封装,提供了面向对象的API
> 
![Runloop.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/Runloop.png)
##### CFRunloop

```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    __CFPort _wakeUpPort;
    Boolean _unused;
    volatile _per_run_data *_perRunData;          
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFTypeRef _counterpart;
};
```
- **pthread:** 代表线程,runloop跟线程是一一对应关系
- **currentMode:** CFRunLoopMode数据结构
- **modes**: 一个包含CFRunLoopMode类型的集合   (NSMutableSet<CFRunLoopMode *>)
- **commonModes:** 一个包含NSString类型的集合    (NSMutableSet<NSString *>)
- **commonModeItems:** 包含多个元素的集合 (NSMutableSet<observer,timer,source *>)
##### CFRunloopMode

```
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
};

```
- **name:** 模式的名称
  - 如NSDefaultRunLoopMode,通过这样一个名称来切换对应的模式;
  - commonModes里面都是名称字符串，通过这些名称来支持多种模式(NSDefaultRunloopMode)
- **sources0:** 集合类型的数据结构(NSMutableSet)
- **sources1:** 集合类型的数据结构(NSMutableSet)
- **observers:** 数组类型的数据结构(NSMutableArray)
- **timers:** 数组类型的数据结构(NSMutableArray)
##### Source/Timer/Observer
###### CFRunLoopSource
- source0
  - 需要手动唤醒线程
- source1
  - 具备唤醒线程能力

###### CFRunLoopTimer
- 基于事件的定时器
- 和NSTimer是`toll-free bridged`

###### CFRunLoopObserver
> 观测时间点
- kCFRunLoopEntry (runloop启动)
- kCFRunLoopBeforeTimers (将要对timer进行观察处理)
- kCFRunLoopBeforeSources (将要对sources进行观察处理)
- kCFRunLoopBeforeWaiting (将要进入休眠 用户态-->内核态)
- kCFRunLoopAfterWaiting (内核态-->用户态)
- kCFRunLoopeExit (runloop退出)
##### CommonMode特殊性
- CommonMode不是实际存在的一种Mode
- 是同步Source/Timer/Observer到多个Mode中的一种技术方案
##### 事件循环的实现机制
###### void CFRunLoopRun()

![runloop事件循环](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/runloop%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E6%9C%BA%E5%88%B6.png)
- 1.即将进入Runloop,发通知
- 2.将要处理Timer/Source0事件, 发通知
- 3.处理source0事件
- 4.如果有source1事件处理,todo跳转8
- 5.没事件处理,线程将要休眠,发通知
- 6.休眠,等待唤醒
  - Source1唤醒
  - Timer事件唤醒
  - 外部手动唤醒
- 7.线程刚被唤醒,发通知
- 8.处理唤醒时收到的消息,处理完后回到2
- 9.即将推出RunLoop,发通知


##### Runloop的核心

![runloop核心](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/runloop%E7%9A%84%E6%A0%B8%E5%BF%83.png)
- `main()`函数经过一系列调用会调用为系统的`mach_msg()`函数,这样就发生了系统调用
- 经过系统调,当前线程就把控制权交给了内核态,`mach_msg()`函数在一定条件下会返回给调用方,这个触发返回的逻辑就是唤醒线程的逻辑(source1,手动唤醒等)
#### Runloop与NSTimer
###### void CFRunLoopAddTimer(runLoop,timer,commonMode)

#### Runloop与多线程
###### 关系
- 一一对应
- 自己创建的线程没有RunLoop
###### 怎么实现一个常驻线程
- 为当前线程开启一个RunLoop
- 向该RunLoop中添加一个Port/Source等维护RunLoop的事件循环
- 启动该RunLoop



## 网络

#### HTTP协议
> 超文本传输协议
> 
##### 请求/响应报文

###### 请求报文
![请求报文](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E8%AF%B7%E6%B1%82%E6%8A%A5%E6%96%87.png)
- 请求行
  - 方法: `GET` `POST`等
  - URL:请求地址
  - 协议版本:HTTP1.1
- 请求头
  - 头部字段以`key:value`形式组合在一起,多个首部字段构成首部字段区域
- 实体主体
  - 一般`GET`请求没有实体主体,`POST`请求有请求主体

###### 响应报文

![响应报文](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%93%8D%E5%BA%94%E6%8A%A5%E6%96%87.png)
- 响应行
  - 版本
  - 状态码:`100` `200` `303` `404` `500`
  - 短语:状态码的描述
- 请求头
  - 头部字段以`key:value`形式组合在一起,多个首部字段构成首部字段区域
- 实体主体

###### 请求方式

- GET
- POST
- HEAD
- PUT
- DELETE
- OPTIONS


###### GET POST区别
**一般的回答是:**
- `GET`请求参数是以`?`拼接到`URL`后面的,而`POST`请求的参数一般是放在`Body`里面的
- `GET`的参数长度限制是`2048`个字符,而`POST`请求一般没有限制
- `GET`请求不太安全,`POST`请求是比较安全的

**但从语义的角度来比较的话,应该是这样的:**
  - GET: 获取资源 `安全的` `幂等的` `可缓存的`
  - POST: 处理资源 `不安全的` `不幂等的` `不可缓存的`


**语义对应的解释:**
- 安全性:
  - 不应该引起`server`端任何状态变化;如`GET`请求多次去`Server`端获取数据,不会引起`Server`端的状态变化
  - 安全请求包括: `GET`,`HEAD`,`OPTIONS`

- 幂等性:
  - 同一个请求方法执行多次跟执行一次的效果完全相同;如`GET`请求多次去`Server`端获取数据,执行的效果是完全相同的;此处并不是指的是获取的数据完全相同(如多次`GET`请求中间有`POST`数据有可能发生改变),指的是`GET`请求这个方法 `执行的效果`
  - 幂等性请求包括: `GET`,`PUT`,`DELETE`
- 可缓存性:
  - 请求是否可以缓存;当我们在发起一个`HTTP`请求的过程中,传递的链路我们是不确定的,虽然说在一条`TCP`链接上,但是网络路径在接触或者通过网关包括一些代理到底`Server`端,在这上面会涉及到方方面面内容,往往对于一些代理服务器会有缓存,而这种缓存性是官方的一种规范,可以遵守也可以不遵守,大多数情况下会遵守,所以在`GET`请求会有相对应的缓存
  - 可缓存的请求有`GET`,`HEAD`
##### 链接建立流程

![HTTP建立流程](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/HTTP%E9%93%BE%E6%8E%A5%E5%BB%BA%E7%AB%8B%E6%B5%81%E7%A8%8B.png)
###### 三次握手
- 为什么是3次握手?(超时)
  - Client第一次发送syn1,超时了,Server没收到
  - Client启动超时重发策略,发送了syn2,Server收到以后,建立链接;
  - 此时第一次发送syn1的也收到,Server以为要再次建立链接
###### 四次挥手
- Client-->FIN-->Server
- Server-->ACK-->Client(半关闭状态)
- Server-->FIN+ACK-->Client
- Client-->ACK-->Server

##### HTTP特点
###### 无连接

- 含义: HTTP链接有建立链接和释放链接的一个过程
- 补偿: HTTP的持久链接
![持久链接](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%8C%81%E4%B9%85%E9%93%BE%E6%8E%A5.png)
  - 涉及到的请求头部字段
    - Connection: keep-alive (客户端期许持久链接)
    - time:20 (持久链接持续时间)
    - max:10 (当前链接最大的http请求和响应对)
  - HTTP的持久链接好处
    - 提升网络请求效率,减少`TCP`连接建立的数量 
  - 怎么判断一个请求是否结束
    > 在一个tcp连接里面发送了多次http请求,怎么区分前一个请求结束了,后一个请求开始
    > 
    - Content-length: 1024 (请求报文跟响应报文都有头部字段,根据服务端响应的数据大小,客户端根据接收数据字节数是否到达1024)
    - chunked: 通过post请求时,server端返回给客户端的数据是需要通过多次响应来返回给客户端这些数据,可以根据http响应报文中头部字段chunked来判断http请求是否结束;当有多个块儿通过http的TCP链接传输给客户端时,每个报文都会带有chunked字段,最后一个是空的chunked


###### 无状态
- 含义:server端对事务处理没有记忆能力
- 补偿: Cookie / Session


#### HTTPS与网络安全

###### HTTPS跟HTTP有什么样的区别

> HTTPS = HTTP + SSL/TLS
> 

###### HTTPS连接建立流程

![HTTPS链接流程](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/HTTPS%E9%93%BE%E6%8E%A5%E6%B5%81%E7%A8%8B.png)
- Client向Server发送TLS版本,支持的算法及一个随机数C
- Server向Client发送商定的加密算法,随机数S,server证书
- Client证书验证(验证公钥)
- Client组装会话秘钥(随机数C+随机数S+客户端产生的预主秘钥)
- Client通过Server的公钥对预主秘钥进行加密传输
- Server通过私钥解密得到预主秘钥
- 组装会话秘钥(随机数C+随机数S+解密得到预主秘钥)
- Client发送加密握手消息
- Server返回加密的握手消息

###### HTTPS连接使用了哪些加密手段?为什么?
- 连接建立过程中使用非对称加密,耗时但是安全
- 后续通信过程使用对称加密
#### TCP、UDP传输层协议

##### TCP 传输控制协议
###### 特点
- 面向连接 (三次握手,四次挥手)
- 可靠传输
  - 特点
    - 无差错
    - 不丢失
    - 不重复
    - 按序到达 (滑动窗口协议)
  - 实现:停止等待协议
    - 无差错情况
    - 超时重传
    - 确认丢失
    - 确认迟到
  ![可靠传输](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%8F%AF%E9%9D%A0%E4%BC%A0%E8%BE%93.png)

- 面向字节流
  - 不管发送方提交给TCP的缓冲是多大的数据,对TCP本身来说它会根据一个实际的情况来进行划分一次性传递的字节数

  ![面向字节流](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E9%9D%A2%E5%90%91%E5%AD%97%E8%8A%82%E6%B5%81.png)
- 流量控制 
  - 滑动窗口协议
    > 发送方的窗口由接收方确定，目的在于控制发送速度，以免接受方缓存不够大，而导致溢出。假如接收方告诉发送方，建议滑动窗口为6，（否则太大一次发送太大，我接收缓冲区太小，处理不过来），然后发送方可以一次发送6个数据帧，假设已经发送了4,5,6，但是没收到关联的ACK，7,8,9则等待发送，如果此时发送端收到4号ACK，则窗口向右收缩，此时窗口就“滑动”了
    
  - 后退n针协议 
    > 发送方一次发送比如说10个帧，前两个针都返回了对应的ACK，数据帧2出现了错误，这时发送方被迫重新发送2-8这七个帧，这就是回退n帧协议。但是如果接收方已经接收到了3-8帧，只是丢失了2帧，全部重传太浪费网络条件了，所以有时候会选择重传丢失的的帧。这就是选择重传协议。

  ![滑动窗口](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3.png)
- 拥塞控制
  > 拥塞控制就是防止过多的数据注入到网络中，这样使得网络中的路由器或者链路不至于过载。
  - 慢开始 拥塞避免
    > 慢开始是指发送方先设置cwnd=1，一次发送一个报文段，随后每经过一个传输轮次，拥塞串口cwnd就加倍，其实增长并不慢，以指数形式增长。还要设定一个慢开始门限，当cwnd>门限值，改用拥塞避免算法。拥塞避免算法使cwnd按线性规律缓慢增长。当网络发生延时，门限值减半，拥塞窗口执行慢开始算法。
  - 快恢复 快重传
    >  当接收方收到失序的报文段，按照快重传，需要尽快发送对未收到的报文段的重复确认。快恢复是指当拥塞串口达到门限值，不直接开启慢启动算法，而是快恢复，快恢复就是收到三个重复的确认（可看作是网络已经拥塞了），此时并不执行慢开始算法，而是执行快恢复，就是新的门限值是原来的一半，直接进入拥塞避免阶段。
    
    
  ![慢开始](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%85%A2%E5%BC%80%E5%A7%8B.png)

###### 功能
- 复用 分用(多端口复用)
- 差错检测


##### UDP 用户数据包协议 
###### 特点
- 无连接
- 尽最大努力交付
  - 不保证可靠传输
- 面向报文 既不合并也不拆分
  
  ![面向报文](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E9%9D%A2%E5%90%91%E6%8A%A5%E6%96%87.png)

###### 功能

- 复用 分用(多端口复用)

  ![复用](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%A4%8D%E7%94%A8%20%E5%88%86%E7%94%A8.png)
- 差错检测

  ![差错检测](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%B7%AE%E9%94%99%E6%A3%80%E6%B5%8B.png)
#### DNS解析
> 域名到IP地址的映射,DNS解析请求采用UDP数据报,且明文,端口号53
> 
- 递归查询
   > 我去给你问一下
- 迭代查询
  > 我告诉你谁可能知道
  > 
- DNS劫持问题
  - httpDNS
    - IP直连 `http://119.29.29.29?dn=www.xxx.com&ip=172.12.134.108`
    - 由原来的使用DNS协议向DNS服务器的53端口进行请求转换为使用HTTP协议向DNS服务器的80端口进行请求
  - 长连接
   ![长连接](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E9%95%BF%E8%BF%9E%E6%8E%A5.png)
- DNS解析转发
  - 例如移动网络下向移动DNS解析
  - 移动DNS解析为了节省资源转给电信DNS
  - 电信DNS通过权威DNS解析对应电信对应的服务器返回
  - 这样的话相当于移动网络访问电信服务器,跨网访问,请求缓慢
#### Session/Cookies
> HTTP协议无状态特点的补偿
> 
##### Session
> Session主要用来记录用户的状态,区分用户;状态保存在服务器端
> 

##### Cookies
> Cookie主要用来记录用户的状态,区分用户;状态保存在客户端
> 客户端发送的cookie是在http请求报文的Cookie首部字段中
> 服务器设置http响应报文的Set-Cookie首部字段
> 
###### 怎么修改cookie?
- 新cookie覆盖旧cookie
- 覆盖规则:name,path,domain需要跟原cookie一致
###### 怎么删除cookie?
- 新cookie覆盖旧cookie
- 覆盖规则:name,path,domain需要跟原cookie一致
- 设置cookie的expries = 过去的某一时间点(cookie失效),或设置maxAge = 0

###### 怎么保证cookie安全?
- 对cookie进行加密处理
- 只在https上携带cookie
- 设置cookie为httpOnly,防止跨站脚本攻击



## 设计模式
#### 设计原则
##### 单一职责原则
> 一个类只负责一件事儿
- CALayer 和 UIView
##### 依赖倒置原则
> 抽象不应该依赖具体实现,具体实现可以依赖于抽象
> 
- 数据的增删改查依赖定义抽象的接口
- 接口数据具体实现是数据库还是plist,文件等等不关心
##### 开闭原则
> 对修改关闭,对扩展开放
> 
- 设计类的时候考虑后续迭代扩展的需求,成员变量定义尽量谨慎,避免反复修改
- 子类继承
##### 里氏替换原则
> 父类可以被子类无缝替换,且原有功能不受任何影响
> 
- KVO机制
##### 接口隔离原则
> 使用多个专门的协议 而不是一个庞大臃肿的协议
> 协议中的方法应该尽量少
- UITableView
##### 迪米特原则
> 一个对象应当对其他对象有尽可能少的了解
> 高内聚 低耦合
#### 设计模式
##### 责任链模式 
###### 场景
![%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F.png)
###### 类构成
![%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F%E7%B1%BB%E6%9E%84%E6%88%90.png](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F%E7%B1%BB%E6%9E%84%E6%88%90.png)
```
#import <Foundation/Foundation.h>

@class BusinessObject;
typedef void(^CompletionBlock)(BOOL handled);
typedef void(^ResultBlock)(BusinessObject *handler, BOOL handled);

@interface BusinessObject : NSObject

// 下一个响应者(响应链构成的关键)
@property (nonatomic, strong) BusinessObject *nextBusiness;
// 响应者的处理方法
- (void)handle:(ResultBlock)result;

// 各个业务在该方法当中做实际业务处理
- (void)handleBusiness:(CompletionBlock)completion;
@end

```
```
#import "BusinessObject.h"

@implementation BusinessObject

// 责任链入口方法
- (void)handle:(ResultBlock)result
{
    CompletionBlock completion = ^(BOOL handled){
        // 当前业务处理掉了，上抛结果
        if (handled) {
            result(self, handled);
        }
        else{
            // 沿着责任链，指派给下一个业务处理
            if (self.nextBusiness) {
                [self.nextBusiness handle:result];
            }
            else{
                // 没有业务处理, 上抛
                result(nil, NO);
            }
        }
    };
    
    // 当前业务进行处理
    [self handleBusiness:completion];
}

- (void)handleBusiness:(CompletionBlock)completion
{
    /*
     业务逻辑处理
     如网络请求、本地照片查询等
     */
}
```
- 这样做就可以根据产品经理的需求,来调整业务之间的指向,来动态调整业务的顺序
- 进一步的调整为服务端动态下发,本地对任务定义任务号及一一对应的类名,作为`key value`形式的通过后端下发,本地根据`Class`的反射来解析出具体的类,根据数组的顺序调整责任链的下一个响应者来动态调整
###### 示例
**[Responder](https://github.com/tutu279737146/DesignPatten/tree/master/Responder)**

##### 桥接模式
###### 场景--业务解耦问题
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E4%B8%9A%E5%8A%A1%E8%A7%A3%E8%80%A6.png)
###### 类构成
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F%E7%B1%BB%E6%9E%84%E6%88%90.png)
###### 示例
**[Bridge](https://github.com/tutu279737146/DesignPatten/tree/master/Bridge)**
##### 适配器模式
###### 场景--一个现有类需要适应变化的问题
> 错误: 对原有类增加实力变量或者方法;

###### 类构成
![avatar](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E9%80%82%E9%85%8D%E5%99%A8%E6%A8%A1%E5%BC%8F%E7%B1%BB%E6%9E%84%E6%88%90.png)
###### 示例
**[Adapter](https://github.com/tutu279737146/DesignPatten/tree/master/Adapter)**

##### 单例模式

###### 示例
```
+ (id)sharedInstance
{
    // 静态局部变量
    static Mooc *instance = nil;
    // 通过dispatch_once方式 确保instance在多线程环境下只被创建一次
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 创建实例
        instance = [[super allocWithZone:NULL] init];
    });
    return instance;
}

// 重写方法【必不可少】
+ (id)allocWithZone:(struct _NSZone *)zone{
    return [self sharedInstance];
}

// 重写方法【必不可少】
- (id)copyWithZone:(nullable NSZone *)zone{
    return self;
}
```
##### 命令模式

###### 场景
> 行为参数化
> 降低代码重合度
###### 示例
**[Command](https://github.com/tutu279737146/DesignPatten/tree/master/Command)**


## 框架及架构
#### 图片缓存框架
> 如何设计一个图片缓存框架?
> 
##### 框架构成

![图片缓存](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%9B%BE%E7%89%87%E7%BC%93%E5%AD%98%E6%A1%86%E6%9E%B6%E7%9A%84%E7%BB%93%E6%9E%84.png)
##### Manager
> 负责调度各个模块
> 
- 读写
  - 以图片URL单向hash值作为key,多级缓存提高查找效率
  - 内存
  - 磁盘
  - 网络
##### 内存模块
###### size
- 数据结构--队列
- 小于10kb 50张
- 小于100kb 20张
- 大于100kb 10张
###### 淘汰策略
- 以队列先进先出方式进行淘汰
- LRU
  - 定时检查(落地难)
  - 提高检查触发频率(图片读写时,前后台切换时)

##### 磁盘模块

###### 存储方式
###### size
###### 淘汰策略

##### 网络模块

###### 最大并发量

###### 超时重试
###### 请求的优先级

##### CodeManager
###### 压缩
###### 解码
- 方式

> 应用策略模式对不同图片格式进行解码
- 时机

> 磁盘读取后  网络请求返回后
> 
#### 客户端架构

![客户端架构](https://raw.githubusercontent.com/tutu279737146/BlogImages/master/Images/%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)
- 独立App的通用层(时长统计,崩溃统计,网络第三方库)
  - 放于任何App中都能起到支撑作用
- 通用业务层 (当前业务相关比如imageView的封装)
- 中间层 (解耦)
- 业务A 业务B 业务C 业务D ....

##### 解耦
- openURL
- 依赖注入
  - 中间层,业务C将自己注入到中间层,业务A去中间层获取所依赖的方法等等
  - 注册protocol
