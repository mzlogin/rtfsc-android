# Android 源码分析 —— 从 Toast 出发

![](./assets/toast.png)

（图 from Android Developers）

Toast 是 Android 开发里较常用的一个类了，有时候用它给用户弹提示信息和界面反馈，有时候用它来作为辅助调试的手段。用得多了，自然想对其表层之下的运行机制有所了解，所以在此将它选为我的第一个 RTFSC Roots。

本篇采用的记录方式是先对它有个整体的了解，然后提出一些问题，再通过阅读源码，对问题进行一一解读而后得出答案。

本文使用的工具与源码：Chrome、插件 insight.io、GitHub 项目 [aosp-mirror/platform_frameworks_base](https://github.com/aosp-mirror/platform_frameworks_base)

## Toast 印象

首先我们从 Toast 类的 [官方文档](1) 和 [API 指南](2) 中可以得出它具备如下特性：

1. Toast 不是 View，它用于帮助创建并展示包含一条小消息的 View；

2. 它的设计理念是尽量不惹眼，但又能展示想让用户看到的信息；

3. 被展示时，浮在应用界面之上；

4. 永远不会获取到焦点；

5. 大小取决于消息的长度；

6. 超时后会自动消失；

7. 可以自定义显示在屏幕上的位置（默认左右居中显示在靠近屏幕底部的位置）；

8. 可以使用自定义布局，也只有在自定义布局的时候才需要直接调用 Toast 的构造方法，其它时候都是使用 makeText 方法来创建 Toast；

9. Toast 弹出后当前 Activity 会保持可见性和可交互性；

10. 使用 cancel 方法可以立即将已显示的 Toast 关闭，让未显示的 Toast 不再显示；

11. Toast 也算是一个「通知」，如果弹出状态消息后期望得到用户响应，应该使用 Notification。

不知道你看到这个列表，是否学到了新知识或者明确了以前不确定的东西，反正我在整理列表的时候是有的。

## 提出问题

根据以上特性，再结合平时对 Toast 的使用，提出如下问题来继续本次源码分析之旅（大致由易到难排列，后文用 小 demo 或者源码分析来解答）：

1. Toast 的超时时间具体是多少？

2. 能不能弹一个时间超长的 Toast？

3. Toast 能不能在非 UI 线程调用？

4. 应用在后台时能不能 Toast？

5. Toast 数量有没有限制？

6. `Toast.makeText(…).show()` 具体都做了些什么？

## 解答问题

### Toast 的超时时间

用这样的一个问题开始「Android 源码分析」，真的好怕被打死……大部分人都会嗤之以鼻：Are you kidding me? So easy. 各位大佬们稍安勿躁，阅读大型源码不是个容易的活，让我们从最简单的开始，一点一点建立自信，将这项伟大的事业进行下去。

面对这个问题，我的第一反应是去查 Toast.LENGTH_LONG 和 Toast.LENGTH_SHORT 的值，毕竟平时都是用这两个值来控制显示长/短 Toast 的。

在 Toast 类源码 platform/frameworks/base/core/java/android/widget/Toast.java 文件中能看到它们俩的定义是这样的：

```java
/**
 * Show the view or text notification for a short period of time.  This time
 * could be user-definable.  This is the default.
 * @see #setDuration
 */
public static final int LENGTH_SHORT = 0;

/**
 * Show the view or text notification for a long period of time.  This time
 * could be user-definable.
 * @see #setDuration
 */
public static final int LENGTH_LONG = 1;
```

啊哦~原来它们只是两个 Flag，并非确切的时间值。

[1]: https://developer.android.com/reference/android/widget/Toast.html
[2]: https://developer.android.com/guide/topics/ui/notifiers/toasts.html
