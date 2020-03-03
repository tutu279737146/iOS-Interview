# iOS-Interview
> 面试知识点整理

- [UI视图相关](#UI视图相关)
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
- [GCD](#GCD)
- [Runtime](#Runtime)
- [Runloop](#Runloop)
- [网络](#网络)

## 内存管理
#### 内存布局
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

- 不同场景不同管理方案
- TaggedPoingter 小对象如`NSNumber`
- NONPOINTER_ISA
  - arm64架构
  - 0-64  1是0否
    - 0 :indexed 0是纯`isa`指针
    - 1 :has_assoc 是否关联对象 
    - 2 :has_cxx_dtor 是否使用过c++
    - 3->15 + 16->31 + 32->35 共33位来表示内存地址
    - 36->41 magic
    - 42 :weakly_referenced 弱引用
    - 43 :deallocating 是否在进行dealloc操作
    - 44 :has_sidetable_rc 当前存储引用计数是否到达上限
    - 45-63 extra_rc 额外引用计数
- 散列表
  - `SideTables()`
    - `spinlock_t` 自旋锁
      - `spinlock_t` 是一种忙等的锁
      - 适用于轻量访问
    - `RefcountMap`引用计数表
      - `hash`表
      - `ptr` ----> `Disguise(objc_object)` ----> `size_t`
      - `size_t`
        - 0: `weakly_referenced`
        - 1: `deallocating`
        - 2-63: 实际的引用计数值 向右偏移俩位来计算
    - `weak_table_t`弱引用表
      - `hash`表
      - `对象指针(key)` ----> `Hash函数` ----> `weak_entry_t(value)`

#### ARC&MRC

- MRC
  - `alloc`
    - 经过一系列的调用,最终调用了C函数的`calloc`
  - `retain`
    - `SideTable & table = SideTables()[this]` //通过当前对象的指针,通过hash函数计算在SideTables找到对应的sizeTable
    - `size_t &refcntStorage = table.refcnts[this]`
    - `refcntStorage += SIDE_TABLE_RC_ONE`
  - `release`
    - `SideTable & table = SideTables()[this]` //通过当前对象的指针,通过hash函数计算在SideTables找到对应的sizeTable
    - `RefcountMap::iterator it = table.fefcnts.find(this)`
    - `refcntStorage -= SIDE_TABLE_RC_ONE`
  - `retainCount`
    - `SideTable & table = SideTables()[this]` 
    - `sizt_t refcnt_result = 1`
    - `RefcountMap::iterator it = table.fefcnts.find(this)`
    - `refcnt_result += it->secont >> SIDE_TABLE_RC_SHIFT`
  - `autoRelease`
  - `dealloc`
    - `_objc_rootDealloc()`
    - `rootDealloc()` 作如下判断
      - `nonpointer_isa`
      - `weakly_referenced`
      - `has_assoc`
      - `has_cxx_dtor`
      - `has_sidetable_rc`
    - 结果为YES 调用`C函数free()` 
    - 结果为NO 调用`object_dispose()`
      - `objc_destructInstance()` 作如下判断
        - `hasCxxDtor`
        - 结果为YES 调用 `objc_cxxDestruct()`
        - 结果为NO 调用 `hasAssociatedObjects`
          - 结果为YES, `_objc_remove_associations()`
          - 结果为NO, `clearDeallocating()`
            - `sidetable_clearDeallocating()`
            - `weak_clear_no_lock()` 将指向该对象的弱引用指针置为nil
            - `table.refcnts.erase()` 引用技术表清除引用计数
      - `C函数free()`
- ARC
  - `ARC`是`LLVM`和`Runtime`协作的结果
  - 禁止手动调用`retain`,`release`,`retainCount`,`dealloc`
  - `weak`,`strong`

#### 弱引用

> `id __weak obj1 = obj;` -- > `objc_initWeak(&obj1,obj)`
>  一个被声明为__weak对象指针,会经过如下方法 添加
- `objc_initWeak()`
- `storeWeak()`
- `weak_register_no_lock()`

> 问:系统是怎样把一个weak变量添加到他所对应的的弱引用表中的?
> 答:一个被声明为__weak的对象指针,经过编译器编译之后,会调用`objc_initWeak()`经过一系列函数调用栈,最终在`weak_register_no_lock()`中进行弱引用变量添加,添加的位置是通过一个hash算法进行位置查找,如果查找位置有当前对象的所对应的的弱引用数组,那么就将新弱引用变量加到数组里面,如果没有,就创建一个弱引用数组,将第0个设置为weak,其他位置置为nil


> 问:当一个weak释放时,怎么置为nil?
> 答:当一个对象被`dealloc`之后,在`dealloc`内部实现当中会调用`weak_clear_no_lock()`,在函数实现当中会根据当前对象的指针查找所对应的的弱引用表,把当前对象所对应的弱引用拿出来是一个数组,遍历置为nil
#### 自动释放池

- 是以栈为接点通过双向链表的形式组合而成的
- 是和线程一一对应的
> 问: `AutoreleasePool`实现原理
> 在当次runloop将要结束的时候调用`AutoreleasePoolPage::pop()`

> 问: `AutoreleasePool`嵌套使用
> 多次嵌套就是多次插入哨兵对象


> 问: 什么情况下使用
> 在for循环中alloc图片等数据消耗较大的场景手动插入autoreleasepool
> 
- `@autoreleasepool{}` 编译后转换为
  -  `void *ctx = objc_autoreleasePoolPush()`
  -  `{}`
  -  `objc_autoreleasePoolPop(ctx)`

- `AutoreleasePoolPage`
  - `id *next`
  - `AutoreleasePoolPage *const parent`
  - `AutoreleasePoolPage *child`
  - `pthread_t const thread`
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
    const char *name;     // 分类的名称
    classref_t cls;       // 宿主类
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
- `_objc_init`
- `map_2_images`
- `map_images_nolock`
- `_read_images`
- `_remethodizeClass`

###### 源码分析
- `remethodizeClass`
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

- `id objc_getAssociatedObject(id object, const void *key)`
- `void objc_setAssociatedObject(id object, const void *key,id value,objc_AssociationPolicy policy)`
- `void objc_removeAssociatedObjects(id object)`

###### 位置
- 成员变量添加的位置并未添加到原宿主数组
- 关联对象由`AssociationsManager`管理在`AssociationHashMap`中
- 全局容器


###### 源码分析

- 以设置关联对象方法为例`void objc_setAssociatedObject(id object, const void *key,id value,objc_AssociationPolicy policy)
`
- 将传入的`value`和`policy`封装为一个`ObjcAssociation`这样一个结构
- `ObjcAssociation`再和`key`(`@selector(text)`)映射为一个`ObjcAssociationMap`,此处`key`作为了方法名
- `ObjcAssociationMap`再跟`object`(`DISGUISE(obj)`)映射为一个全局的容器
```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy){
    objc_setAssociatedObject_non_gc(object, key, value, policy);
}
void objc_setAssociatedObject_non_gc(id object, const void *key, id value, objc_AssociationPolicy policy){
    _objc_set_associative_reference(object, (void *)key, value, policy);
}

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
###### 数据结构
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

#### 数据结构
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
    - 用于快速查找方法执行函数
    - 可增量扩展的哈希表结构
    - 是局部性原理的最佳应用
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

###### **对象,类对象,元类对象**
- 一张图片

###### **消息传递**

- `void objc_msgSend(id self, Sel op, ...)`
- `void objc_msgSendSuper(struct objc_super *super, Sel op, ...)`

###### **缓存查找**

> 给定值是SEL,目标值是对应的`bucket_t`中的`IMP`
> 
- cache_key_t ----> f(key)---->bucket_t
- f(key) = key &mask

###### **当前类中查找**

- 排序好的,采用二分查找法查找相应的执行函数
- 未排序的,采用一般遍历法查找

###### **父类中查找**
- curClass ----> superClass

###### **消息转发**
- `resolveInstanceMethod`
- `forwardingTargetForSelector`
- `methodSignatureForSelector`和`forwardInvocation`

###### **Method-Swizzling**
- `load`
###### **动态添加方法**

- `class_addMethod(Class cls, SEL name, IMP imp,
    const char *types)`
- `performSelector: `

###### **动态方法解析**

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
