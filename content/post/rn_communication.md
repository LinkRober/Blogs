+++
keywords = ["React Native"]
tags = ["React Native"]
categories = ["development"]
date = "2017-10-25T00:00:00Z"
title = "React Native和Native间的通信"
draft = false
comments = true
showPagination = true
autoThumbnailImage = true

coverImage = '../../../rn_image.jpg'
showTags = true
showSocial = true
showDate = true

+++
<!-- thumbnailImage = 'rn_image.jpeg'
thumbnailImagePosition = 'bottom' -->

本系列文章作为学习RN期间的总结

- [React Native如何集成到现有项目中](https://linkrober.github.io/bookshelf/2017/10/react-native%E5%A6%82%E4%BD%95%E9%9B%86%E6%88%90%E5%88%B0%E7%8E%B0%E6%9C%89%E9%A1%B9%E7%9B%AE%E4%B8%AD/)
- React Native和Native间的通信
- [RCTRootView、RCTBridge做了什么](https://linkrober.github.io/bookshelf/2017/10/rctrootviewrctbridge%E5%81%9A%E4%BA%86%E4%BB%80%E4%B9%88/)



<!--more-->

#### Native -> RN

###### Properties

`properties`是RN和Native间通信最简单的方式。我们只需要在初始化`RCTRootView`的时候，将需要的数据通过方法入参的形式传入，它是一个字典类型。

```
- (instancetype)initWithBundleURL:(NSURL *)bundleURL
                       moduleName:(NSString *)moduleName
                initialProperties:{{< hl-text red >}}(NSDictionary *)initialProperties{{< /hl-text >}}
                    launchOptions:(NSDictionary *)launchOptions
```
在RN中我们用来承接`initialProperties`参数的就是每个js类的`props`。如果我们需要在Native中修改`props`属性，就要对`RCTRootView`的`appProperties`进行重新赋值，然后RN会对视图进行重新渲染，当且仅当两次赋值不一样的时候才会重新渲染。

```
rootView.appProperties = @{@"content" : imageList};
```

我们也可以使用

```
- (instancetype)initWithBridge:(RCTBridge *)bridge
                    moduleName:(NSString *)moduleName
             initialProperties:(NSDictionary *)initialProperties NS_DESIGNATED_INITIALIZER;
```
在初始化bridge时以代理的方式将bundleURL传入：

```
- (instancetype)initWithDelegate:(id<RCTBridgeDelegate>)delegate
                   launchOptions:(NSDictionary *)launchOptions;


 @protocol RCTBridgeDelegate <NSObject>

/**
 * The location of the JavaScript source file. When running from the packager
 * this should be an absolute URL, e.g. `http://localhost:8081/index.ios.bundle`.
 * When running from a locally bundled JS file, this should be a `file://` url
 * pointing to a path inside the app resources, e.g. `file://.../main.jsbundle`.
 */
- (NSURL *)sourceURLForBridge:(RCTBridge *)bridge;

@optional 
```

{{< alert warning >}}
需要注意的是，赋值操作必需要放在Native的主线程。而且js中的`componentWillReceiveProps`和`componentWillUpdateProps`并不会因为重新渲染而被再次调用，你只能在`componentWillMount`方法中访问新的`props`
{{< /alert >}}


###### RCTEventEmitter

我们可以使用`RCTEventEmitter`类来让js监听native的事件，facebook的文档中是这样描述`RCTEventEmitter`的：

```
/**
 * RCTEventEmitter is an abstract base class to be used for modules that emit
 * events to be observed by JS.
 */
```
它是一个模块的抽象基类，用来发送被JS监听的事件。我们通过

```
- (void)sendEventWithName:(NSString *)name body:(id)body
```
方法来完成事件的发送。入参是事件名称和需要携带的参数（Dictionary）。
每一个`RCTRootView`都有一个`RCTBridge`属性，它是RN中非常重要的一个类，负责Native和JS之间的通信。通过`RCTBridge`的:

```
/**
 * Retrieve a bridge module instance by name or class. Note that modules are
 * lazily instantiated, so calling these methods for the first time with a given
 * module name/class may cause the class to be sychronously instantiated,
 * potentially blocking both the calling thread and main thread for a short time.
 */
- (id)moduleForName:(NSString *)moduleName;
```
方法，我们可以拿到每个`RCTRootView`对应的`RCTEventEmitter`实例（它是通过lazy load的方式初始化的），因为这里的`moduleName`参数可以是类名也可以是自定义的模块名称，所以在项目中如果出现多个实例，需要在初始化通过view的`reactTag`属性来区分。该属性声明在分类`UIView+React.h`中。

```
RCT_EXPORT_MODULE();

+ (EventEmitterManger *)mangerWithRootView:(RCTRootView *)rootView {
    return [rootView.bridge moduleForName:NSStringFromClass(self)];
}
```
这样我们就拿到目标RootView对应的`RCTEventEmitter`对象。接下来我们需要为事件定义一个identifier，重写

```
/**
 * Override this method to return an array of supported event names. Attempting
 * to observe or send an event that isn't included in this list will result in
 * an error.
 */
- (NSArray<NSString *> *)supportedEvents;
```
方法，把你方法的identifer作为数组的一个元素返回。这样就可以使用上面所说的方法进行消息的发送了。
到这里Native端的代码已经完成了，下面看在js中如何设置。
我们需要在对应的js中引入一个`NativeModules`、`NativeEventEmitter`模块，然后获取监听模块的manger

```
var raToRnManger = NativeModules.EventEmitterManger
```
在js对象的声明周期方法`componentWillMount`中设置监听，接下来就等native发送消息了。

```
componentWillMount(){
         var raToRnMangerEmitter = new NativeEventEmitter(raToRnManger)
          const subscription = raToRnMangerEmitter.addListener("EventReminder",
              (reminder) => {
                  console.log("test")
                    this.setState({
                        name:"B"
                    })
              }
          );
    }
```

#### RN -> Native


###### RCTBridgeModule

在Native端定义RN模块的时候，让其遵循`<RCTBridgeModule>`协议，在实现文件中暴露出你想要从RN向Native发送的消息

```
RCT_EXPORT_METHOD(exampleMethod:(NSString *)name)
{
    NSLog(@"%@",name);
}
```
如果你想要该方法参数支持闭包，可以使用RN内置的闭包类型`RCTResponseSenderBlock`

```
RCT_EXPORT_METHOD(exampleCallbackMethod:(RCTResponseSenderBlock)callback)
{
    NSDictionary *param = @{@"letter":@"B"};
    callback(@[[NSNull null], param]);
}
```

在js端，通过`NativeModules`模块拿到Native中对应的模块
```
var manger = NativeModules.ExportMethodManger
```
如果RN中某个事件被触发了，直接通过manger向Native发送消息，消息名称正如你在Native中声明的那样

```
onPressLearnMore(){
        manger.exampleMethod("##############Learn More############")
    }
```

在Native项目中嵌入RN，你绝对不想要在js中再写一套网络请求吧。这时候就会遇到用RN展示UI，而数据请求放在在Native中做的情况。我们可以使用`XMLHttpRequest`这类js网络框架，但是这里想说的是`async/await`语法，它在ES6中被提供。从命名上一眼就能看出其目的，异步方法等待消息返回。

```
async updateEvents(){
        try {
            var events = await manger.findEvents();
            this.setState({
                letter:events[0].letter
            });
          } catch (e) {
            console.error(e);
          }
    }
```
这样，如果Native中`findEvents`是一个异步的方法就可以在js中等待Native的回调了。





