# 阅读源码工具选用

之前我写过一篇 [搭建大型源码阅读环境——使用 OpenGrok][1]，给大家介绍了一款开源的源码阅读工具的安装方法，实际到目前为止，OpenGrok 仍是我最喜爱的源码阅读工具之一。

本次 Android 源码阅读之旅，我也将继续使用 OpenGrok，但同时在阅读 framework 层源码时会使用 Chrome 插件 [Insight.io for GitHub][2] 来打开 GitHub 上的 framework 镜像 [android/platform_frameworks_base][3]，它是一款强大的辅助工具，目录树、文件搜索、方法列表、引用查询等等都不在话下，要感受它的妙处可以安装起来体验，你值得拥有，先来几张图感受一下：

目录树、文件搜索：

![](assets/file-tree-and-search.png)

方法列表、引用查询、弹出注释：

![](assets/outline-and-refrerence.png)

[1]: http://mazhuang.org/2016/12/14/rtfsc-with-opengrok/
[2]: https://chrome.google.com/webstore/detail/insightio-for-github/pmhfgjjhhomfplgmbalncpcohgeijonh
[3]: https://github.com/android/platform_frameworks_base
