# Activity 在横竖屏切换情况下的生命周期变化


# 概述

Activity 在横竖屏切换的时候,生命周期是不一样的,本地通过打印 log 的方式,看下区别.测试的机器是 Android6.0 .

<!-- more -->

# 不做任何配置的情况下

## 第一次启动

```console
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [null]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@de950fc
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onResume() called
```

------------------------------------------------------------------------

## 第一次切换成横屏

```console

D/LifeCircleActivity: onPause() called
D/LifeCircleActivity: onSaveInstanceState() called with: outState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onStop() called
D/LifeCircleActivity: onDestroy() called
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@266fbfb
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onRestoreInstanceState() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onResume() called

```



------------------------------------------------------------------------


## 再切换成竖屏

```console
D/LifeCircleActivity: onPause() called
D/LifeCircleActivity: onSaveInstanceState() called with: outState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onStop() called
D/LifeCircleActivity: onDestroy() called
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@7e6e82e
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onRestoreInstanceState() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onResume() called
```


------------------------------------------------------------------------

## 小结

默认情况下，每次旋转屏幕都会销毁当前的Activity对象，同时调用 onSaveInstanceState 方法，保存当前的界面状态；之后重新创建 Activity对象， onCreate 参数不为空，回调  onRestoreInstanceState 方法进行恢复。



# 配置 configChanges="orientation"

## 第一次启动

```console
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [null]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@de950fc
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onResume() called
```

------------------------------------------------------------------------

## 第一次切换成横屏

```console
D/LifeCircleActivity: onPause() called
D/LifeCircleActivity: onSaveInstanceState() called with: outState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onStop() called
D/LifeCircleActivity: onDestroy() called
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@266fbfb
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onRestoreInstanceState() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onResume() called
```

------------------------------------------------------------------------


## 再切换成竖屏

```console
D/LifeCircleActivity: onPause() called
D/LifeCircleActivity: onSaveInstanceState() called with: outState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onStop() called
D/LifeCircleActivity: onDestroy() called
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@7e6e82e
D/LifeCircleActivity: onStart() called 
D/LifeCircleActivity: onRestoreInstanceState() called with: savedInstanceState = [Bundle[{android:viewHierarchyState=Bundle[{android:views={16908290=android.view.AbsSavedState$1@80a47f5, 2131296581=android.view.AbsSavedState$1@80a47f5, 2131296815=android.view.AbsSavedState$1@80a47f5}}], key=x}]]
D/LifeCircleActivity: onResume() called
```



------------------------------------------------------------------------

## 小结

配置 orientation 的情况下，和默认情况一致。

------------------------------------------------------------------------


# 配置 configChanges="orientation|screenSize"

根据官方的介绍，这个两个值，在api大于13 之后，应该一起使用
## 第一次启动

```console
D/LifeCircleActivity: onCreate() called with: savedInstanceState = [null]Activity对象的地址:cn.steve.activitylifecycle.LifeCircleActivity@de950fc
D/LifeCircleActivity: onStart() called
D/LifeCircleActivity: onResume() called
```

------------------------------------------------------------------------

## 第一次切换成横屏

```console
D/LifeCircleActivity: onConfigurationChanged() called with: newConfig = [{1.0 ?mcc?mnc zh_CN ldltr sw360dp w640dp h336dp 320dpi nrml long land finger -keyb/v/h -nav/h s.11 themeChanged=0 themeChangedFlags=0}]
```

------------------------------------------------------------------------


## 再切换成竖屏

```console
D/LifeCircleActivity: onConfigurationChanged() called with: newConfig = [{1.0 ?mcc?mnc zh_CN ldltr sw360dp w360dp h616dp 320dpi nrml long port finger -keyb/v/h -nav/h s.12 themeChanged=0 themeChangedFlags=0}]
```

------------------------------------------------------------------------

## 小结
当配置了 screenSize 。则不会再销毁重建了，而是回调 onConfigurationChanged 方法。



# 总结

在不做配置默认的情况下,Activity 是被销毁,然后重新启动的.但是在 manifest 中进行相应的配置之后,就表示 Activity 自行处理配置的更改,将阻止 Activity 的销毁重新启动,而是保持运行状态,并且回调 onConfigurationChanged 方法.官方的建议是万不得已的情况下才能使用.


# 参考
* [AndroidDeveloper](https://developer.android.com/guide/topics/manifest/activity-element.html?hl=zh-cn)
* [处理运行时变更](https://developer.android.com/guide/topics/resources/runtime-changes?hl=zh-cn)

