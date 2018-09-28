# runtime-dictToModel
runtime 字典转模型

> 将后台JSON数据中的字典转成本地的模型，我们一般选用部分优秀的第三方框架，如SBJSON、JSONKit、MJExtension、YYModel等。但是，一些简单的数据，我们也可以尝试自己来实现转换的过程。


## 快速使用
当我们的请求到的数据不是很复杂, 也不希望引入第三方框架的时候, 可以使用下这个分类, 来实现字典转模型.

### 1.根据请求数据, 创建对应的模型类, 并根据字典中的键值对定义对应的属性
+ 创建模型原则: 从外层到内存, 一个类型字典对应一个模型
+ 示例程序中, 根据plist, 创建了三个类: ShopItem , AttrModel , ListItemModel 

![runtime-pic1.png](https://upload-images.jianshu.io/upload_images/126164-b08f62ddfa2ceaa4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

+ 注意: 定义的属性名和字典中的键名字一致.

### 2.在分类中导入最外层模型

![runtime-pic2.png](https://upload-images.jianshu.io/upload_images/126164-7232ec2b19a8c4ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.最外层类中导入 NSObject+EnumDict 分类

![runtime-pic3.png](https://upload-images.jianshu.io/upload_images/126164-9b1b653825878152.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.遵守分类协议 ModelDelegate, 实现协议方法

![runtime-pic4.png](https://upload-images.jianshu.io/upload_images/126164-7ed2752967e441d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 5.控制器中, 导入最外层模型 ShopItem.h , 解析数据遍历数组, 并字典转模型

![runtime-pic5.png](https://upload-images.jianshu.io/upload_images/126164-5a93c23ce6b12585.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 原理讲解
![runtime字典转模型的核心算法.png](https://upload-images.jianshu.io/upload_images/126164-e461f28bef5a32e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以往, 我们字典转模型,总是需要在模型类中定义一个静态方法或者对象方法,来字典转模型, 这样, 我们在不同的模型中, 都必须定义这样一个方法来完成字典转模型, 如果我们写的项目比较大, 模型比较多,这样字典转模型的效率就太低了,耦合性也比较高, 那我们如何做到字典转模型 与 模型类的彻底解耦呢?

我们可以创建一个 NSObject 的分类, 因为所有的类(NSProxy 除外)都继承自 NSObject, 那我们就可以用任意的类去调 NSObject 的这个分类方法, 子类可以任意调用父类方法

那么我们如何在这个分类方法中完成字典转模型呢?

这里就要用到**运行时**的概念了,

### 首先我们在分类中导入 <objc/runtime.h>这个框架, 然后进行第一步,获取属性列表

```
const char *kPropertyListKey = "SKPropertyListKey";

+ (NSArray *)sk_objcProperties
{
     /* 获取关联对象 */
    NSArray *ptyList = objc_getAssociatedObject(self, kPropertyListKey);

     /* 如果 ptyList 有值,直接返回 */
    if (ptyList) {
        return ptyList;
    }
     /* 调用运行时方法, 取得类的属性列表 */
    /* 成员变量:
     * class_copyIvarList(__unsafe_unretained Class cls, unsigned int *outCount)
     * 方法:
     * class_copyMethodList(__unsafe_unretained Class cls, unsigned int *outCount)
     * 属性:
     * class_copyPropertyList(__unsafe_unretained Class cls, unsigned int *outCount)
     * 协议:
     * class_copyProtocolList(__unsafe_unretained Class cls, unsigned int *outCount)
     */
    unsigned int outCount = 0;
    /**
     * 参数1: 要获取得类
     * 参数2: 类属性的个数指针
     * 返回值: 所有属性的数组, C 语言中,数组的名字,就是指向第一个元素的地址
     */
    /* retain, creat, copy 需要release */
    objc_property_t *propertyList = class_copyPropertyList([self class], &outCount);

    NSMutableArray *mtArray = [NSMutableArray array];

     /* 遍历所有属性 */
    for (unsigned int i = 0; i < outCount; i++) {
         /* 从数组中取得属性 */
        objc_property_t property = propertyList[i];
         /* 从 property 中获得属性名称 */
        const char *propertyName_C = property_getName(property);
         /* 将 C 字符串转化成 OC 字符串 */
        NSString *propertyName_OC = [NSString stringWithCString:propertyName_C encoding:NSUTF8StringEncoding];
        [mtArray addObject:propertyName_OC];
    }

     /* 设置关联对象 */
    /**
     *  参数1 : 对象self
     *  参数2 : 动态添加属性的 key
     *  参数3 : 动态添加属性值
     *  参数4 : 对象的引用关系
     */

    objc_setAssociatedObject(self, kPropertyListKey, mtArray.copy, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    /* 释放 */
    free(propertyList);
    return mtArray.copy;

}

```

### 其实上面这段代码,只有4句是最关键的

1.`/* 获取关联对象 */ NSArray *ptyList = objc_getAssociatedObject(self, kPropertyListKey);`
如果在程序运行的时候, 模型对象的属性是不会发生变化的, 我们在利用这个函数如果能获取到关联对象的属性列表, 就不用再走下面的代码去利用运行时再去获取属性列表了

2.`objc_property_t *propertyList = class_copyPropertyList([self class], &outCount);`
这句代码就是真正的利用运行时获取属性列表, 这个属性列表是 C 的结构体指针数组,我们必须将其遍历,并利用另外一个函数将取出结构体指针所指向的结构体中国的 C 字符串,也就是属性名称

3.`const char *propertyName_C = property_getName(property);`
获得C字符串后,我们只需要将其转换为 OC 字符串,加到可变数组中即可

4.`objc_setAssociatedObject(self, kPropertyListKey, mtArray.copy, OBJC_ASSOCIATION_RETAIN_NONATOMIC);`
设置属性列表, 就是把已经生成好的属性列表设置到一个类似于*属性*的东西储存起来, 下次 get 的时候,直接拿出来用即可,有点类似于*懒加载*.

### 获取属性列表之后, 我们就要进行字典转模型的操作了

首先我们要遍历参数字典, 如果我们获取得属性列表中包含了字典中的 key,就利用 KVC 方法赋值,然后就完成了字典转模型的操作

```
+ (instancetype)sk_objcWithDict:(NSDictionary *)dict
{
     /* 实例化对象 */
    id objc = [[self alloc]init];

     /* 使用字典,设置对象信息 */
     /* 1\. 获得 self 的属性列表 */
    NSArray *propertyList = [self  sk_objcProperties];

     /* 2\. 遍历字典 */
    [dict enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {

         /* 3\. 判断 key 是否字 propertyList 中 */
        if ([propertyList containsObject:key]) {
             /* 说明属性存在,可以使用 KVC 设置数值 */
            [objc setValue:obj forKey:key];
        }

    }];

     /* 返回对象 */
    return objc;
}

```

这样, 比如我在 ViewDidLoad 方法中, 自定义一个字典
然后我只需要一行代码就可以获取到模型对象,如下

```
- (void)viewDidLoad {
    [super viewDidLoad];
    /* 创建一个字典 */
    NSDictionary *dict = @{
                           @"name":@"小明",
                           @"age":@18,
                           @"title":@"master",
                           @"height":@1.7,
                           @"something":@"nothing"
                           };

    Person *person = [Person sk_objcWithDict:dict];
}

```

而此时, 模型类中,没有添加任何的构造方法,只有单纯的属性,这样就做到了彻底的解耦, 比如我现在再来一个学生(Student)类,我也无需添加构造方法,也同样只需要调用`-(instancetype)sk_objcWithDict:dict;`即可.


