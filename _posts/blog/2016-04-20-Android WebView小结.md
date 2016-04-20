---
layout: post
title: Android WebView小结
description: Android WebView 注意事项
category: blog
---
      
     
### 需要注意的点
1. webView.loadUrl(url)方法参数url一般要加协议头（http或者https）
2. myWebView.setWebViewClient(new WebViewClient())，要设置这个WebViewClient，不然后跳转到外部浏览器