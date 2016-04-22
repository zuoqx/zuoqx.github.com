---
layout: post
title: Android WebView小结
description: Android WebView 注意事项
category: blog
---
      

### 需要注意的点
1. webView.loadUrl(url)方法参数url一般要加协议头（http或者https）
2. myWebView.setWebViewClient(new WebViewClient())，要设置这个WebViewClient，不然后跳转到外部浏览器
3. 注入到Web页面JS的context里的Java对象(通过webview.addJavascriptInterface()方法)，运行在另外的线程里，而不是在创建它的线程
4. `webview.loadUrl("javascript:alert()")` 调用JS方法
5. 在API 17及以上，注入到web里的方法要加`@JavascriptInterface`注解
6. 使用`addJavascriptInterface()`方法，将允许JavaScript控制你的应用，这样存在安全问题。Google的建议是，除非加载到WebView里的HTML和JavaScript全部是你自己的，否则，就不要使用`addJavascriptInterface()`。并且不要允许用户在你的WebView里打开外部链接，而是应该使用默认浏览器
7. `WebChromeClient`和`WebViewClient`的区别
    
    * `WebChromeClient`: 当那些会影响浏览器UI的事发生时，会被调用。比如进度改变、JS的alert弹出等
    * `WebViewClient`: 当那些会引起浏览器内容渲染的事发生时，会被调用。比如发生错误、表单提交、以及拦截Url等

8. `WebSettings`有各种关于WebView的设置
9. ` WebSettings.setBuiltInZoomControls(boolean)`方法，可以添加页面缩放功能。但是：**如果WebView的宽或高设置为`WRAP_CONTENT`，将会导致无法预测的行为，应该避免这样做**
10. `shouldInterceptRequest`回调方法里，可以加载本地缓存的图片
11. 自Android 4.4起，Android中的WebView开始基于Chromium，性能大幅提升
12. 子线程更新WebView，使用` runOnUiThread()`方法：

        runOnUiThread(newRunnable(){
            @Override
            publicvoid run(){
            // Code for WebView goes here
            }
        });

13. 不要在UI线程中去等待JavaScript 的回调，应该在子线程里，或者在API 19里调用方法`evaluateJavascript() `,**但必须是UI线程调用**，例如：

        WebView.evaluateJavascript(script, new ValueCallback() {
            @Override
            public void onReceiveValue(String value) {
            //TODO
            }
        });

### WebView缓存

WebView的缓存可以分为**页面缓存**和**数据缓存**。

1. 页面缓存
   
    页面缓存是指加载一个网页时的html、JS、CSS等页面或者资源数据产生的缓存。这些缓存资源是由于浏览器的行为而产生，开发者只能通过配置HTTP响应头影响浏览器的行为才能间接地影响到这些缓存数据。            

2. 数据缓存分为两种：**AppCache**和**DOM Storage（Web Storage）**。他们是因为页面开发者的直接行为而产生。所有的缓存数据都由开发者直接完全地掌控。

    `AppCache 使我们能够有选择的缓冲web浏览器中所有的东西，从页面、图片到脚本、css等等。尤其在涉及到应用于网站的多个页面上的CSS和JavaScript文件的时候非常有用。其大小目前通常是5M，需要手动开启（setAppCacheEnabled）。并设置路径（setAppCachePath）和容量（setAppCacheMaxSize，但在API 18调用没有作用，系统统一管理缓存大小）`

    `DOM Storage 存储一些简单的用key/value,分为Session Storage和Local Storage两种。分别用于会话级别的存储（页面关闭即消失）和本地化存储（除非主动删除，否则数据永远不会过期）。`
    > 手动开启DOM Storage（setDomStorageEnabled），设置存储路径（setDatabasePath)
    
    > 如果需要清除Local Storage的话，仅仅删除Local Storage的本地存储文件是不够的，内存里面有缓存数据。如果再次进入页面，Local Storage中的缓存数据同样存在。需要杀死程序运行的当前进程再重新启动才可以。

3. 优化：主要从缓存方面考虑：缓存网页、数据，不如图片等，或者把一些资源html、js保存到本地，然后需要时加载等