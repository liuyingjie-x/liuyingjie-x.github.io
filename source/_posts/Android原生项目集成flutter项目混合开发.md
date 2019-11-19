# Android原生项目集成flutter项目混合开发

方案挑选：

目前主要有两种集成方式：

1、**源码集成**：就是谷歌官方提供的方案（ https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps ）

2、 **产物集成**:  Flutter项目单独开发，开发完成后发布成安卓以aar包，iOS的framework形式，原生项目依赖flutter输出的制品就可以了，具体可以看闲鱼的文章，他们就是这么干的，借用网上的一张图

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025170029.png)

两种方案各有优劣，鉴于我们项目参与人数众多，如果采用源码集成的方式，所有人都得配相应的环境和了解相关的知识，显然成本较高，所有我们选择了第二种方式，专门几个人去做flutter项目然后以AAR包的方式集成到项目中来

这篇文章里我两种都尝试一下，当然我是安卓这边的尝试

## 方式一、源码集成的方式，项目里直接引入flutter module

直接从AS的图形界面导入已经新建的flutter module尝试失败

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025132954.png)



这样导入flutter module失败



**正确打开方式：**

1、使用Android Studio来创建Flutter Module（放在原生的同级目录） 

> 1、 依次点击左上角的File --> New --> New Flutter Project 
>
> 2、然后选择Flutter Module。

如图：

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025133531.png)

2、在项目的根目录settings.gradle文件中配置：

```java
include ':app'
include ':flutter_module'
setBinding(new Binding([gradle: this]))
evaluate(new File(        
    settingsDir.parentFile,        
    '主项目名称/flutter_module名称/.android/include_flutter.groovy'))
```



3、在app下的gradle文件中添加

```java
 implementation project(':flutter') 
```

完成这三步就顺利完成在原生项目中集成flutter module了



安卓原生界面跳转到flutter界面有两种方式，一种是使用**flutterView**，另一种是使用 **FlutterFragment** ，这里我选择了前者

先新建一个activity，FlutterViewActivty（随意命名），在oncreate方法中加入以下代码

```kotlin
  private fun initView() {
        val flutterView: View = Flutter.createView(this, lifecycle, "flutter Route1")
        val layoutParams = FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
        addContentView(flutterView, layoutParams)
    }
```

`Flutter.createView()`方法返回的是一个**FlutterView**，它继承自View，我们可以把它当做一个普通的View，调用`addContentView()`方法将这个View添加到Activity的contentView中。我们注意到`Flutter.createView()`方法的第三个参数传入了"flutter Route1"字符串，表示路由名称，它确定了Flutter中要显示的Widget，接下来需要在之前创建好的Flutter Module中编写逻辑了，修改**main.dart**文件中的代码

```dart
Widget _widgetForRoute(String route) {
  switch (route) {
    case "flutter Route1":
      return Scaffold(
        appBar: AppBar(
          title: Text("test native"),
        ),
        body: Center(
          child: Text('flutter 页面,route=$route'),
        ),
      );
    default:
      return Center(
        child: Text('Unknown route: $route', textDirection: TextDirection.ltr),
      );
  }
}
```

在runApp()方法中通过window.defaultRouteName可以获取到我们在Flutter.createView()方法中传入的路由名称，即"flutter Route1"，之后编写了一个_widgetForRoute()方法，根据传入的route字符串显示相应的Widget。

最后在Mainacivity中加入跳转事件

```java
switch (v.getId()) {
            case R.id.btn_flutter:
                Intent intent = new Intent(MainActivity.this, FlutterViewActivity.class);
                startActivity(intent);
                break;
        }
```

大功告成！！

但是发现原生的标题栏还在，在style中加入透明标题栏样式

```groovy
<style name="FlutterTheme" parent="AppTheme.NoActionBar">
        <item name="android:windowTranslucentStatus">true</item>
    </style>
```

顺利完成，如图，点击跳转的时候会有黑屏一下的现象，不过没事，这是debug情况才会有的，debug有一些调试的预加载，打了release包之后就没有这个现象了

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025174333.png)



## 方案二、AAR方式集成

 **1、Android 原工程需要使用 java 8 编译** 

 在 工程的 build.gradle 里面，android {} 下修改 

```groovy
android {
  //...
  compileOptions {
    sourceCompatibility 1.8
    targetCompatibility 1.8
  }
}
```

 **2、生成 aar 文件** 

 通过集成 arr 的方式需要先生成 Flutter Module 的 arr，在 Flutter Module 的根目录下运行以下命令

```groovy
flutter build aar
```

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025174822.png)

 **3、使用 aar** 

 主工程的 build.gradle repositories 闭包 里面加入 

```groovy
  flatDir {
            dirs 'libs'   // aar目录
        }
```

然后把AAR包复制到libs目录下面， app 的 build.gradle 里面添加依赖 

```groovy
implementation (name:'flutter-release-old',ext:'aar')
```

跳转的代码和方式一一样不用动，编译运行之后成功

![](https://raw.githubusercontent.com/liuyingjie-x/BlogImage/master/20191025175320.png)