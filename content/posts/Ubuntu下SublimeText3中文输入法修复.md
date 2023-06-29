---
title: Ubuntu下SublimeText3中文输入法修复
date: 2016-10-07 14:09:09
tags: [Ubuntu,SublimeText3,输入法]
categories: [ComputerFoundation,OS]
---
## 概述
本文记录在 ubuntu 如何修复 Sublime Text 不能输入中文的问题.

<!-- more -->

## 问题
在 ubuntu 下安装完 Sublime Text之后,发现不能输入中文,也不能正常的切换成中文输入法.google 了一圈之后,看到网上的解决方案,记录下来,方便日后使用.

## 解决方案

### 1. 编译一段C代码.保存以下的代码,并将文件命名为 `sublime-imfix.c`
```C
/*
sublime-imfix.c
Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
By Cjacker Huang

gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
LD_PRELOAD=./libsublime-imfix.so subl
*/
##include 
##include 
typedef GdkSegment GdkRegionBox;

struct _GdkRegion
{
  long size;
  long numRects;
  GdkRegionBox *rects;
  GdkRegionBox extents;
};

GtkIMContext *local_context;

void
gdk_region_get_clipbox (const GdkRegion *region,
            GdkRectangle    *rectangle)
{
  g_return_if_fail (region != NULL);
  g_return_if_fail (rectangle != NULL);

  rectangle->x = region->extents.x1;
  rectangle->y = region->extents.y1;
  rectangle->width = region->extents.x2 - region->extents.x1;
  rectangle->height = region->extents.y2 - region->extents.y1;
  GdkRectangle rect;
  rect.x = rectangle->x;
  rect.y = rectangle->y;
  rect.width = 0;
  rect.height = rectangle->height;
  //The caret width is 2;
  //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
  if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
        gtk_im_context_set_cursor_location(local_context, rectangle);
  }
}

//this is needed, for example, if you input something in file dialog and return back the edit area
//context will lost, so here we set it again.

static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
{
    XEvent *xev = (XEvent *)xevent;
    if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
       GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
       if(GDK_IS_WINDOW(win))
         gtk_im_context_set_client_window(im_context, win);
    }
    return GDK_FILTER_CONTINUE;
}

void gtk_im_context_set_client_window (GtkIMContext *context,
          GdkWindow    *window)
{
  GtkIMContextClass *klass;
  g_return_if_fail (GTK_IS_IM_CONTEXT (context));
  klass = GTK_IM_CONTEXT_GET_CLASS (context);
  if (klass->set_client_window)
    klass->set_client_window (context, window);

  if(!GDK_IS_WINDOW (window))
    return;
  g_object_set_data(G_OBJECT(context),"window",window);
  int width = gdk_window_get_width(window);
  int height = gdk_window_get_height(window);
  if(width != 0 && height !=0) {
    gtk_im_context_focus_in(context);
    local_context = context;
  }
  gdk_window_add_filter (window, event_filter, context);
}
```


### 2. 因为要编译C代码,所以需要安装C的编译环境
打开命令行,安装编译环境
```shell
sudo apt-get install build-essential
sudo apt-get install libgtk2.0-dev
```

### 3. 编译如上说的C代码,会在当前目录下生成`libsublime-imfix.so`文件,并将`libsublime-imfix.so`复制到`/opt/sublime_text/`目录下.

```shell
gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
```

### 4. 修改`/usr/share/applications/sublime_text.desktop`

```shell
vim sublime_text.desktop
```


然后修改对应的内容
```shell
[Desktop Entry]
[...]
Exec=env LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text %F
[...]

[Desktop Action Window]
[...]
Exec=env LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text -n
[...]

[Desktop Action Document]
[...]
Exec=env LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text --command new_file
[...]
```


## 参考
* [ 完美解决 Linux 下 Sublime Text 中文输入 ](https://www.sinosky.org/linux-sublime-text-fcitx.html)
