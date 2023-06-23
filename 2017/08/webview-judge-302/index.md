# webview中关于服务端重定向的判断


# 背景
由于 H5 页面打开都比较慢,为了减少返回时候的刷新,所以有时候需要多开,就是每个 url 都是在一个新的 activity 中打开.一般的处理方式是在 shouldOverrideUrlLoading() 方法中进行处理,这个方法按照 SDK 中的说明是当 url 发生变化时就会回调,当遇到服务端重定向的时候,就会出现一个空白页面的情况,所以我们需要判断是否是重定向,对于重定向进行特殊的处理.

<!-- more -->

# 困难

```java

public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
        return shouldOverrideUrlLoading(view, request.getUrl().toString());
}

```
在 Android7.0 ,新增了 WebResourceRequest 接口,接口中有个判断是否是重定向的方法,但对于低版本的该如何判断呢?
可以做如下判断.

```java
WebViewClient webViewClient = new WebViewClient() {

        private boolean mLoaded = false;

        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            if (mLoaded){
                //not redirect
            }else {
                // is redirect
            }
            return true;

        }

        @Override
        public void onPageFinished(WebView webView, String url) {
            mLoaded = true;
        }

    };
```

因为能回调 onPageFinished 方法,都是有渲染页面的操作,说明页面是有内容的,对于服务端重定向而言,是没有内容的,所以就不会走 onPageFinished 方法,通过这个间接的判断页面是否是重定向的.目前还未发现有什么不对的地方.


# 参考
1. [stackoverflow](https://stackoverflow.com/questions/3852414/in-a-webview-is-there-a-way-for-shouldoverrideurlloading-to-determine-if-it-is-c/45477862#45477862)

