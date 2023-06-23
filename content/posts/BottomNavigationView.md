---
title: BottomNavigationView 的使用
date: 2017-02-18 23:02:58
tags: [BottomNavigationView]
categories: [Mobile,Android]
---
# why
BottomNavigationView 这个概念很早之前就被提出,之后出一个第三方库.但是一直未有官方的支持,今天正好看到有官方支持,记录一下.

<!-- more -->


# what
[BottomNavigationView](https://material.io/guidelines/components/bottom-navigation.html)  是 material design 中的设计的实现，这种设计很早就出现了。

# how
1. 添加依赖
```groovy
compile 'com.android.support:design:25.1.1'
```
2. 在menu下新建 menu配置文件，顺序就是 bottom bar 显示的顺序
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
            android:id="@+id/recents"
            android:title="Recents"
            android:icon="@drawable/ic_recents"/>
    <item
            android:id="@+id/favorites"
            android:title="Favorites"
            android:icon="@drawable/ic_favorites"/>
    <item
            android:id="@+id/nearby"
            android:title="Nearby"
            android:icon="@drawable/ic_nearby"/>
</menu>
```

3. 在 drawable 和 drawable-v21下新建item的背景文件。
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- in drawable for lower then 21 API-->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true" android:drawable="@android:color/white"/>
    <item android:drawable="@android:color/transparent"/>
</selector>
<?xml version="1.0" encoding="utf-8"?>
<!--in drawable-v21 folder, for greater or equal then 21 API-->
<ripple
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:color="@android:color/white">
</ripple>
```

4. 在 layout 下新建布局文件，用来添加 BottomNavigationView
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <android.support.design.widget.BottomNavigationView
            android:id="@+id/bottomNavigationView"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:background="@android:color/holo_blue_bright"
            app:itemBackground="@drawable/navigationbar_item_bg"
            app:itemIconTint="@android:color/white"
            app:itemTextColor="@android:color/white"
            app:menu="@menu/menu_bottom_navigation_view"/>

</RelativeLayout>
```

5. 在 Activity 或者 fragment 中添加监听即可。
```java
bottomNavigationView = (BottomNavigationView) findViewById(R.id.bottomNavigationView);
this.bottomNavigationView.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
    @Override
    public boolean onNavigationItemSelected(@NonNull MenuItem item) {
        Toast.makeText(BottomNavigationViewActivity.this, item.getTitle(), Toast.LENGTH_SHORT).show();
        return true;
    }
});
```
#　Ｃonclusion
官方的这个支持，可扩展性不强，也没什么特别的新意，可参考学习第三方库[BottomBar](https://github.com/roughike/BottomBar/).









