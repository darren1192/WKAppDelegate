# WKAppDelegate

[![Version](https://img.shields.io/cocoapods/v/SHRMAppDelegate.svg?style=flat)](http://cocoapods.org/pods/SHRMAppDelegate)
[![Pod License](http://img.shields.io/cocoapods/l/SHRMAppDelegate.svg?style=flat)](https://opensource.org/licenses/MIT)
![iOS 6.0+](https://img.shields.io/badge/iOS-6.0%2B-blue.svg)
![](https://img.shields.io/badge/language-objc-orange.svg)
![ARC](https://img.shields.io/badge/ARC-orange.svg)


#### 注：在`v0.0.3`版本引入`NSObject+AppEventModule`分类，用以解决`NSObject`的`performSelector:`函数最多支持2个参数的问题。
#### 注：`v0.0.7`版本进行了prefix变更，`SHRM`变更为`WK`。


## Link
* Blog : [iOS 代码优化 -- AppDelegate模块化瘦身](https://juejin.im/post/5c62caf6e51d457fc905dd75)

## 介绍
**一个对APPDelegate深度解耦的逻辑，教你实现APPDelegate模块化拆分，原本上千行的代码可以简化到10行内，使用方便，极少的代码浸入。支持iOS6+**
![image text](https://user-gold-cdn.xitu.io/2019/2/12/168e20a0d3b1bb53?imageslim)

**AppDelegate简化后的样子：**

```objc
#import "WKAppDelegate.h"
@interface AppDelegate : WKAppDelegate
@property (strong, nonatomic) UIWindow *window;
@end
```
```objc
#import "MainViewController.h"
@implementation AppDelegate
@synthesize window;
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
[super application:application didFinishLaunchingWithOptions:launchOptions];
[self initMainController];
return YES;
}

- (void)initMainController {
self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
self.window.backgroundColor = [UIColor whiteColor];
self.window.rootViewController = [[MainViewController alloc] init];
[self.window makeKeyAndVisible];
}
@end
```
**你没看错，引入框架后，你的AppDelegate只有这些代码！怎么做的？看demo。**

## 特性

- 搭积木式设计，业务插件化分离。
- `AppDelegate.m`内的代码精简到10行内。
- 原有的功能模块拆分为单独的子模块，模块独立，模块间无耦合。
- 每个模块都拥有自己的生命周期，目前支持模块的初始化->销毁。
- 模块可自定义执行的优先级，也就是说可以自定义原`AppDelegate`中不同业务功能的加载顺序。
- 模块扩展性高，易于维护，新需求只需新增模块即可，顺便在模块内部管理下该模块的生命周期。
- 代码可读性强，每个模块只负责该模块内的业务。
- 模块支持插拔，需要就拖到项目中，不需要删除也不影响项目build！
- 其他特性demo中自己挖掘。

## 安装

### 1.CocoaPods
1. 在 Podfile 中添加 `pod 'WKAppDelegate', '~> 0.0.7'`。
2. 执行 `pod install` 或 `pod update`。
3. 在AppDelegate中导入 `<WKAppDelegate.h>`并继承。


### 2.手动导入

下载WKAppDelegate文件夹，将WKAppDelegate文件夹拖入到你的工程中。

## 用法

- `WKAppDelegate`为对外暴漏的主体类，内部实现了所有的App生命周期方法，当然你也可以指定实现哪一类方法，或者不需要框架实现，只需要在你的`AppDelegate`中重写即可。
- `WKAppEventModuleManager`是模块的管理类，管理着所有注册过的模块。
- `WKBaseAppEventModule`是所有模块需要继承的基类，为所有模块提供公共方法，包括模块执行顺序，模块生命周期管理等，可自行扩充。
- `WKAppEventAnnotation`为模块注册专用类，每个模块都需要注册的，为什么注册？不注册怎么管理呢。
- `NSObject+AppEventModule`为NSObject分类，扩充了`NSObject`的`performSelector:`函数，用以支持多参传递。

### 1.`AppDelegate`继承`WKAppDelegate`，让`WKAppDelegate`来接管App的生命周期函数。
### 2.创建插件，继承`WKBaseAppEventModule`：

```objc
#import "WKBaseAppEventModule.h"

@interface testMudule : WKBaseAppEventModule

@end
```
```objc
WKAppEventMod(testMudule)

@implementation testMudule

// 该插件执行顺序
- (NSInteger)moduleLevel {
return 1;
}

//该插件需要在didFinishLaunchingWithOptions:生命周期函数中做些操作
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
//插件初始化
[self initMudule];
//插件销毁，父类实现
[self destroyModule];
return YES;
}

- (void)initMudule {
NSLog(@"testMudule init");
}

@end
```
**插件概念：每个插件都为独立的业务，比如程序启动后的数据库处理、定位处理等都可以定义为一个插件，让插件继承`WKBaseAppEventModule`插件就会被管理。如果想在程序启动的时候执行该插件就在该插件里面重写`didFinishLaunchingWithOptions:`即可，如上。如果想在程序进入后台执行该插件的一些业务那么只需要重写`applicationDidEnterBackground:`即可，然后在该函数里面进行业务处理。**

### 3.如果需要自己实现App周期函数只需要在你的AppDelegate重写即可。

### 4.NSObject+AppEventModule分类

提供`performSelector:`函数传递多个参数的解决方案，基于`NSMethodSignature`和`NSInvocation`实现。核心代码如下：

```objc
- (id)performSelector:(SEL)selector
           withObject:(id)p1
           withObject:(id)p2
           withObject:(id)p3 {
    NSMethodSignature *sig = [self methodSignatureForSelector:selector];
    if (sig) {
        NSInvocation* invo = [NSInvocation invocationWithMethodSignature:sig];
        [invo setTarget:self];
        [invo setSelector:selector];
        [invo setArgument:&p1 atIndex:2];
        [invo setArgument:&p2 atIndex:3];
        [invo setArgument:&p3 atIndex:4];
        [invo invoke];
        if (sig.methodReturnLength) {
            return [self handleReturnType:sig invo:invo];
        } else {
            return nil;
        }
    } else {
        return nil;
    }
}
```

```objc
- (id)handleReturnType:(NSMethodSignature *)sig invo:(NSInvocation *)aInvo
{
    int booResult = strcmp([sig methodReturnType], @encode(BOOL));
    int floatResult = strcmp([sig methodReturnType], @encode(float));
    int intResult = strcmp([sig methodReturnType], @encode(int));
    
    id anObject = nil;
    if (booResult == 0)
    {
        BOOL result;
        [aInvo getReturnValue:&result];
        anObject = [NSNumber numberWithBool:result];
    }
    else if (floatResult == 0)
    {
        float result;
        [aInvo getReturnValue:&result];
        anObject = [NSNumber numberWithFloat:result];
    }
    else if (intResult)
    {
        int result;
        [aInvo getReturnValue:&result];
        anObject = [NSNumber numberWithInt:result];
    }
    return anObject;
}
```

### 5.详细使用请参照demo，如有疑问欢迎issue，欢迎star。

## License

WKAppDelegate is available under the MIT License. See the [LICENSE](https://github.com/GitWangKai/WKAppDelegate/blob/master/LICENSE) file for more info.

