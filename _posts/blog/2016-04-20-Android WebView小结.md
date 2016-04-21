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


