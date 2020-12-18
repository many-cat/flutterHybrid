### 学习索引
1. https://coding.imooc.com/class/321.html

# Flutter 与Android 通信

### 集成方式

1. 分页面嵌入
2. 单页面内混合嵌入

### 集成步骤

1. 创建Flutter module
2. 添加Flutter module 依赖
3. 在 kotlin/switch 中调用Flutter module
4. 编写Dart 代码

### Flutter Native 通信场景

1. 初始化Flutter时Native向Dart传递数据
2. 两者相互单向传递数据：Native发送数据给Dart，Dart发送数据给Native
3. 两者双向传递数据：Dart发送数据给Native，然后Native回传数据给Dart

![https://www.devio.org/io/flutter_app/img/blog/Flutter-Dart-Native-Communication.png](https://www.devio.org/io/flutter_app/img/blog/Flutter-Dart-Native-Communication.png)

### Flutter Native 通信机制

Flutter和Native的通信是通过Channel来完成的。
消息使用Channel（平台通道）在客户端（UI）和主机（平台）之间传递，如下图所示：

![https://www.devio.org/io/flutter_app/img/blog/PlatformChannels.png](https://www.devio.org/io/flutter_app/img/blog/PlatformChannels.png)

- Flutter中消息的传递是完全异步的；

[Channel 所支持类型的对照表](https://www.notion.so/275769e3f3d141c892aeb9a623698bf5)

### Flutter 定义了三种不同类型的Channel

1. BasicMessageChannel: 用于传递字符串和半结构化的信息，持续通信，收到消息后可以回复此消息，如：Native 将遍历到的文件信息陆续传递到Dart，再比如：Flutter将服务端陆续获取到的信息交给Native加工，Native处理完返回等；
2. MethodChannel: 用于传递方法调用一次性通信；如Flutter调用Native拍照
3. EventChannel：用于数据流的通信，持续通信，收到消息后无法回复此消息，通长用于Native向Dart的通信，例如：手机电量变化，网络连接变化，陀螺仪，传感器等
4. 这三种类型的Channel都是全双工通道，即A<=>B，Dart可以主动发送消息给platform端，并且platform接受到消息后可以做出回应，同样platform端可以主动发送消息给Dart端，dart端接收数据后返回给platform端。

### BasicMessageChannel

Native端：

```java
/**
 * 构造方法原型
 * messenger - 消息信使，是消息的发送与接收的工具；
 *
 * name - Channel的名字，也是其唯一标识符；
 * 
 * codec - 消息的编解码器，它有几种不同类型的实现;
 *     - BinaryCodec - 最为简单的一种Codec，因为其返回值类型和入参的类型相同，均为二进制格式（Android中为ByteBuffer，iOS中为NSData）。
 *       实际上，BinaryCodec在编解码过程中什么都没做，只是原封不动将二进制数据消息返回而已。
 *       或许你会因此觉得BinaryCodec没有意义，但是在某些情况下它非常有用，
 *       比如使用BinaryCodec可以使传递内存数据块时在编解码阶段免于内存拷贝
 *     - StringCodec - 用于字符串与二进制数据之间的编解码，其编码格式为UTF-8；
 *     - JSONMessageCodec - 用于基础数据与二进制数据之间的编解码，其支持基础数据类型以及列表、字典。
 *       其在iOS端使用了NSJSONSerialization作为序列化的工具，而在Android端则使用了其自定义的JSONUtil与StringCodec作为序列化工具；
 *     - StandardMessageCodec - 是BasicMessageChannel的默认编解码器，其支持基础数据类型、二进制数据、列表、字典，其工作原理；
 */
BasicMessageChannel(BinaryMessenger messenger, String name, MessageCodec<T> codec)

/**
 * setMessageHandler
 *
 * handler - 消息处理器，配合BinaryMessenger完成消息的处理；
 * 在创建好BasicMessageChannel后，如果要让其接收Dart发来的消息，则需要调用它的setMessageHandler方法为其设置一个消息处理器。
 */
 void setMessageHandler(BasicMessageChannel.MessageHandler<T> handler)

/**
 * BasicMessageChannel.MessageHandler原型
 *
 * onMessage - 用于接受消息，var1是消息内容，var2是回复此消息的回调函数；
 */
public interface MessageHandler<T> {
    void onMessage(T var1, BasicMessageChannel.Reply<T> var2);
}

/**
 * send方法原型 - 在创建好BasicMessageChannel后，如果要向Dart发送消息，可以调用它的send方法向Dart传递数据
 *
 * message  - 要传递给Dart的具体信息
 *
 * callback - 消息发出去后，收到Dart的回复的回调函数,
 */
void send(T message)
void send(T message, BasicMessageChannel.Reply<T> callback)
```

```java
//具体实现
public class BasicMessageChannelPlugin implements BasicMessageChannel.MessageHandler<String>, BasicMessageChannel.Reply<String> {
    private final Activity activity;
    private final BasicMessageChannel<String> messageChannel;

    static BasicMessageChannelPlugin registerWith(BinaryMessenger messenger, Activity activity) {
        return new BasicMessageChannelPlugin(messenger, activity);
    }

    private BasicMessageChannelPlugin(BinaryMessenger messenger, Activity activity) {
        this.activity = activity;
        this.messageChannel = new BasicMessageChannel<>(messenger, "BasicMessageChannelPlugin", StringCodec.INSTANCE);
        //设置消息处理器，处理来自Dart的消息
        messageChannel.setMessageHandler(this);
    }

    @Override
    public void onMessage(String s, BasicMessageChannel.Reply<String> reply) {//处理Dart发来的消息
        reply.reply("BasicMessageChannel收到：" + s);//可以通过reply进行回复
        if (activity instanceof IShowMessage) {
            ((IShowMessage) activity).onShowMessage(s);
        }
        Toast.makeText(activity, s, Toast.LENGTH_SHORT).show();
    }

    /**
     * 向Dart发送消息，并接受Dart的反馈
     *
     * @param message  要给Dart发送的消息内容
     * @param callback 来自Dart的反馈
     */
    void send(String message, BasicMessageChannel.Reply<String> callback) {
        messageChannel.send(message, callback);
    }

    @Override
    public void reply(String s) {

    }
}
```

Dart端：

```dart

/**
* 构造方法原型
* name - Channel的名字，要和Native端保持一致；
* 
* codec - 消息的编解码器，要和Native端保持一致，有四种类型的实现具体可以
*/
const BasicMessageChannel(this.name, this.codec);

/**
 * setMessageHandler方法原型 - 在创建好BasicMessageChannel后，如果要让其接收Native发来的消息，则需要调用它的setMessageHandler方法为其设置一个消息处理器
 *
 *  handler(T message) - 消息处理器，配合BinaryMessenger完成消息的处理；
 */
void setMessageHandler(Future<T> handler(T message))

/**
 * send方法原型 - 在创建好BasicMessageChannel后，如果要向Native发送消息，可以调用它的send方法向Native传递数据。
 *
 * message - 要传递给Native的具体信息；
 *
 * Future<T> - 消息发出去后，收到Native回复的回调函数；
 */
Future<T> send(T message)
```

```dart
import 'package:flutter/services.dart';
...
static const BasicMessageChannel _basicMessageChannel = const BasicMessageChannel('BasicMessageChannelPlugin', StringCodec());
...
//使用BasicMessageChannel接受来自Native的消息，并向Native回复
_basicMessageChannel
    .setMessageHandler((String message) => Future<String>(() {
          setState(() {
            showMessage = message;
          });
          return "收到Native的消息：" + message;
        }));
//使用BasicMessageChannel向Native发送消息，并接受Native的回复
String response;
    try {
       response = await _basicMessageChannel.send(value);
    } on PlatformException catch (e) {
      print(e);
    }
...
```

### MethodChannel

Native端：

```java
//构造方法原型
/**
 * messenger - 消息信使，是消息的发送与接收的工具；
 *
 * name - Channel的名字，也是其唯一标识符；
 * 
 * codec - 用作MethodChannel的编解码器；
 */
//会构造一个StandardMethodCodec.INSTANCE类型的MethodCodec
MethodChannel(BinaryMessenger messenger, String name)
//或
MethodChannel(BinaryMessenger messenger, String name, MethodCodec codec)

/**
 * 在创建好MethodChannel后，需要调用它的setMessageHandler方法为其设置一个消息处理器，以便能加收来自Dart的消息。
 * 
 * handler - 消息处理器，配合BinaryMessenger完成消息的处理；
 */
setMethodCallHandler(@Nullable MethodChannel.MethodCallHandler handler)

/**
 * onMethodCall - 用于接受消息，
 *   call是消息内容，它有两个成员变量,String类型的call.method表示调用的方法名,Object 类型的call.arguments表示调用方法所传递的入参；
 *   result是回复此消息的回调函数提供了result.success，result.error，result.notImplemented方法调用
 */
public interface MethodCallHandler {
    void onMethodCall(MethodCall var1, MethodChannel.Result var2);
}
```

```java
public class MethodChannelPlugin implements MethodCallHandler {
    private final Activity activity;

    /**
     * Plugin registration.
     */
    public static void registerWith(BinaryMessenger messenger, Activity activity) {
        MethodChannel channel = new MethodChannel(messenger, "MethodChannelPlugin");
        MethodChannelPlugin instance = new MethodChannelPlugin(activity);
        channel.setMethodCallHandler(instance);
    }

    private MethodChannelPlugin(Activity activity) {
        this.activity = activity;
    }

    @Override
    public void onMethodCall(MethodCall call, Result result) {
        switch (call.method) {//处理来自Dart的方法调用
            case "send":
                showMessage(call.arguments());
                result.success("MethodChannelPlugin收到：" + call.arguments);//返回结果给Dart
                break;
            default:
                result.notImplemented();
        }
    }

    /**
     * 展示来自Dart的数据
     *
     * @param arguments
     */
    private void showMessage(String arguments) {
        if (activity instanceof IShowMessage) {
            ((IShowMessage) activity).onShowMessage(arguments);
        }
        Toast.makeText(activity, arguments, Toast.LENGTH_SHORT).show();
    }
}
```

Dart 端：

```dart
/**
 *  name - Channel的名字，要和Native端保持一致；
 *
 *  codec - 消息的编解码器，默认是StandardMethodCodec，要和Native端保持一致；
 */
const MethodChannel(this.name, [this.codec = const StandardMethodCodec()])

/**
 *  method：要调用Native的方法名；
 *
 *  [ dynamic arguments ]：调用Native方法传递的参数，可不传；
 */
Future<T> invokeMethod<T>(String method, [ dynamic arguments ])
```

```dart
import 'package:flutter/services.dart';
...
static const MethodChannel _methodChannelPlugin = const MethodChannel('MethodChannelPlugin');
...
String response;
    try {
		response = await _methodChannelPlugin.invokeMethod('send', value);
    } on PlatformException catch (e) {
      print(e);
    }
...
```

### EventChannel

Native 端：

```java
/**
 * messenger - 消息信使，是消息的发送与接收的工具；
 * name - Channel的名字，也是其唯一标识符;
 * codec - 用作EventChannel的编解码器；
 */
//会构造一个StandardMethodCodec.INSTANCE类型的MethodCodec
EventChannel(BinaryMessenger messenger, String name)
//或
EventChannel(BinaryMessenger messenger, String name, MethodCodec codec)

/**
 * 在创建好EventChannel后，如果要让其接收Dart发来的消息，则需要调用它的setStreamHandler方法为其设置一个消息处理器。
 * 
 * handler - 消息处理器，配合BinaryMessenger完成消息的处理；
 */
void setStreamHandler(EventChannel.StreamHandler handler)

public interface StreamHandler {
    /**
     *  Flutter Native监听事件时调用
     *  
     *  args是传递的参数
     * 
     *  eventSink是Native回调Dart时的会回调函数
     *  eventSink提供success、error与endOfStream三个回调方法分别对应事件的不同状态；
     */
    void onListen(Object args, EventChannel.EventSink eventSink);

    /**
     *  Flutter取消监听时调用
     */
    void onCancel(Object o);
}
```

```java
public class EventChannelPlugin implements EventChannel.StreamHandler {
    private EventChannel.EventSink eventSink;

    static EventChannelPlugin registerWith(BinaryMessenger messenger) {
        EventChannelPlugin plugin = new EventChannelPlugin();
        new EventChannel(messenger, "EventChannelPlugin").setStreamHandler(plugin);
        return plugin;
    }

    void send(Object params) {
        if (eventSink != null) {
            eventSink.success(params);
        }
    }

    @Override
    public void onListen(Object args, EventChannel.EventSink eventSink) {
        this.eventSink = eventSink;
    }

    @Override
    public void onCancel(Object o) {
        eventSink = null;
    }
}
```

Dart端：

```dart
/**
 * name - Channel的名字，要和Native端保持一致;
 *
 * codec - 消息的编解码器，默认是StandardMethodCodec，要和Native端保持一致；
 */
const EventChannel(this.name, [this.codec = const StandardMethodCodec()]);

/**
 * arguments - 监听事件时向Native传递的数据；
 *
 * codec - 消息的编解码器，默认是StandardMethodCodec，要和Native端保持一致；
 */
Stream<dynamic> receiveBroadcastStream([ dynamic arguments ])

```

```dart
// 初始化一个广播流用于从channel中接收数据，它返回一个Stream接下来需要调用Stream的listen方法来完成注册，
// 另外需要在页面销毁时调用Stream的cancel方法来取消监听；

import 'package:flutter/services.dart';
...
static const EventChannel _eventChannelPlugin = EventChannel('EventChannelPlugin');
StreamSubscription _streamSubscription;
  @override
  void initState() {
    _streamSubscription=_eventChannelPlugin
        .receiveBroadcastStream()
        .listen(_onToDart, onError: _onToDartError);
    super.initState();
  }
  @override
  void dispose() {
    if (_streamSubscription != null) {
      _streamSubscription.cancel();
      _streamSubscription = null;
    }
    super.dispose();
  }
  void _onToDart(message) {
    setState(() {
      showMessage = message;
    }); 
  }
  void _onToDartError(error) {
    print(error);
  }
...
```

## 实例讲解

1. 初始化Flutter时Native向Dart传递数据

    ![https://www.devio.org/io/flutter_app/img/blog/init-data-to-dart.gif](https://www.devio.org/io/flutter_app/img/blog/init-data-to-dart.gif)

在Flutter的API中提供了Native在初始化Dart页面时传递数据给Dart的方式，这种传递数据的方式比上文中所讲的其他几种传递数据的方式发生的时机都早。

1. Android向Flutter传递初始化数据**`initialRoute`**

 2 . Flutter允许我们在初始化Flutter页面时向Flutter传递一个String类型的**`initialRoute`**参数，从参数名字它是用作路由名的，但是既然Flutter给我们开了这个口子，那可以搞点事情，传递点我们想传的其他参数呢，比如：

```dart
findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        String inputParams = paramInput.getText().toString().trim();
        startActivity(
            FlutterActivity
                .withNewEngine()
                .initialRoute("route1")
                .build(MainActivity.this)
        );
    }
});
```

然后在Flutter module通过如下方式获取：

```dart
import 'dart:ui';//要使用window对象必须引入

String initParams=window.defaultRouteName;
//序列化成Dart obj 敢你想干的
...
```

2. Native到Dart的通信(Native发送数据给Dart)

在Flutter 中Native向Dart传递消息可以通过BasicMessageChannel或EventChannel都可以实现：

1. 通过BasicMessageChannel的方式

    ![https://www.devio.org/io/flutter_app/img/blog/native-to-dart-BasicMessageChannel.gif](https://www.devio.org/io/flutter_app/img/blog/native-to-dart-BasicMessageChannel.gif)

1. 通过EventChannel的方式

    ![https://www.devio.org/io/flutter_app/img/blog/native-to-dart-EventChannel.gif](https://www.devio.org/io/flutter_app/img/blog/native-to-dart-EventChannel.gif)

以上就是使用不同Channel实现Native到Dart通信的原理及方式，接下来我们来看一下实现Dart到Native之间通信的原理及方式。

3. Dart到Native的通信(Dart发送数据给Native)

在Flutter 中Dart向Native传递消息可以通过BasicMessageChannel或MethodChannel都可以实现：

1. 通过BasicMessageChannel的方式:

    ![https://www.devio.org/io/flutter_app/img/blog/dart-to-native-BasicMessageChannel.gif](https://www.devio.org/io/flutter_app/img/blog/dart-to-native-BasicMessageChannel.gif)

 

1. 通过MethodChannel的方式

    ![https://www.devio.org/io/flutter_app/img/blog/dart-to-native-MethodChannel.gif](https://www.devio.org/io/flutter_app/img/blog/dart-to-native-MethodChannel.gif)
