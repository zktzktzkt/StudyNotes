# AccessibilityService 

## 1 AccessibilityService基本介绍

### 1.1 AccessibilityService 使用方法

+ 第一步

创建一个类继承自AccessibilityService,复写onAccessibilityEvent、onInterrupt方法,然后在相应的方法添加事件的处理逻辑即可

+ 第二步

配置服务的配置xml文件,类似如下

```
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/accessibility_service_description"
    android:packageNames="com.tencent.mm"
    android:accessibilityEventTypes="typeNotificationStateChanged|typeWindowStateChanged|typeWindowsChanged|typeWindowContentChanged"
    android:accessibilityFlags="flagDefault|flagReportViewIds"
    android:accessibilityFeedbackType="feedbackVisual"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="true"
    android:settingsActivity="com.stdnull.accessibility_demo.MainActivity"
    />
```

+ 第三步

在manifest中注册服务

```
<service android:name=".MyAccessibilityService"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_service_config" />
        </service>
```

### 1.2 Serive配置文件配置说明

+ android:description-关于该accessibility service 目的或者行为的描述

+ android:packageNames-指定该accessibility service需要接收事件的包名,不写代表接受所有应用的事件

+ android:accessibilityEventTypes-指定该service需要接受的事件类型,例如typeAllMask表示接受所有事件,具体事件解释见[AccessibilityEvent文档](https://developer.android.com/reference/android/view/accessibility/AccessibilityEvent.html)

+ android:accessibilityFlags-附加的额外标记,用于获取额外的事件信息

+ android:accessibilityFeedbackType-服务反馈类型

+ android:notificationTimeout-接受相同事件的最小时间周期

+ android:canRetrieveWindowContent-指示服务能否获取到当前活跃Window的内容

+ android:settingsActivity-指示一个组件名，用户可通过该界面来配置服务

## 2 AccessibilityService常用API

### 2.1 AccessibilityService中常用API

+ public void onAccessibilityEvent(AccessibilityEvent event)

接收到event的callback,在此处接收到配置中的指定事件,并处理

+ public void onInterrupt()

中断accessibility feedback的callback(没有发现用处)

+ protected void onServiceConnected()

Service的生命周期方法,标志Service被系统启动

+ public AccessibilityNodeInfo getRootInActiveWindow()

获取当前活跃的Window的根布局节点

### 2.2 相关类

+ AccessibilityNodeInfo

对于Window中视图View的封装,通过该节点获取到当前视图层级中节点的信息,从而获取、执行相应的操作

+ AccessibilityServiceInfo

用于描述Service的配置行为,可以通过setServiceInfo方法动态的设置给AccessibilityService

最后一个简单的[抢红包插件的demo](https://github.com/stdnull/demo/tree/master/accessibility_demo)