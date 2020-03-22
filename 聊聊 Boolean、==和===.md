# 聊聊 Boolean、==和===
业务开发，踩坑多次，本文将从 V8 源码分析 Boolean、==和===。
## Boolean
## ==
## ===




[V8 源码](https://cs.chromium.org/chromium/src/v8/?g=0)可在浏览器中查看，这个网站在代码浏览与检索方面的功能十分强大，可以快速的查看 C++ 变量的定义和引用。缺点是不能查看 V8 某个版本或某个 git tag 的源码，但依然强烈推荐。如果想要查看 V8 某个 tag 的源码，可以访问 [v8.git](https://chromium.googlesource.com/v8/v8.git) 。如果想要在本地编译 V8 源码，在参考 [V8 官方文档](https://v8.dev/docs/build-gn)的基础上，还要注意墙的问题，浏览器能访问 google 不代表终端也能访问 google，终端也需要设置代理。








