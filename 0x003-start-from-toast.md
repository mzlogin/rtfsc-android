# Android 源码分析 —— 从 Toast 出发

![](./assets/toast.png)

（图 from Android Developers）

Toast 是 Android 开发里较常用的一个类了，有时候用它给用户弹提示信息和界面反馈，有时候用它来作为辅助调试的手段。用得多了，自然想对其表层之下的运行机制有所了解，所以在此将它选为我的第一个 RTFSC Roots。

本篇采用的记录方式是先对它有个整体的了解，然后提出一些问题，再通过阅读源码，对问题进行一一解读而后得出答案。

本文使用的工具与源码为：Chrome、插件 insight.io、GitHub 项目 [aosp-mirror/platform_frameworks_base][3]

**目录**

<!-- vim-markdown-toc GFM -->

* [Toast 印象](#toast-印象)
* [提出问题](#提出问题)
* [解答问题](#解答问题)
    * [Toast 的超时时间](#toast-的超时时间)
    * [能不能弹一个时间超长的 Toast？](#能不能弹一个时间超长的-toast)
    * [Toast 能不能在非 UI 线程调用？](#toast-能不能在非-ui-线程调用)
    * [应用在后台时能不能 Toast？](#应用在后台时能不能-toast)

<!-- vim-markdown-toc -->

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

10. 使用 `cancel` 方法可以立即将已显示的 Toast 关闭，让未显示的 Toast 不再显示；

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

面对这个问题，我的第一反应是去查 `Toast.LENGTH_LONG` 和 `Toast.LENGTH_SHORT` 的值，毕竟平时都是用这两个值来控制显示长/短 Toast 的。

文件 [platform_frameworks_base/core/java/android/widget/Toast.java][4] 中能看到它们俩的定义是这样的：

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

啊哦~原来它们只是两个 flag，并非确切的时间值。

既然是 flag，那自然就会有根据不同的 flag 来设置不同的具体值的地方，于是使用 insight.io 点击 `LENGTH_SHORT` 的定义搜索一波 `Toast.LENGTH_SHORT` 的引用，在 [aosp-mirror/platform_frameworks_base][3] 里一共有 50 处引用，但都是调用 `Toast.makeText(...)` 时出现的。

继续搜索 `Toast.LENGTH_LONG` 的引用，在 [aosp-mirror/platform_frameworks_base][3] 中共出现 42 次，其中有我们想找的：

文件 [platform_frameworks_base/services/core/java/com/android/server/notification/NotificationManagerService.java][5] 里

```java
long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
```

在同一文件里能找到 `LONG_DELAY` 与 `SHORT_DELAY` 的定义：

```java
static final int LONG_DELAY = PhoneWindowManager.TOAST_WINDOW_TIMEOUT;
static final int SHORT_DELAY = 2000; // 2 seconds
```

点击查看 `PhoneWindowManager.TOAST_WINDOW_TIMEOUT` 的定义：

文件 [platform_frameworks_base/services/core/java/com/android/server/policy/PhoneWindowManager.java][6]

```java
/** Amount of time (in milliseconds) a toast window can be shown. */
public static final int TOAST_WINDOW_TIMEOUT = 3500; // 3.5 seconds
```

于是，我们可以得出 **结论：Toast 的长/短超时时间分别为 3.5 秒和 2 秒。**

*Tips: 也可以通过分析代码里的逻辑，一层一层追踪用到 `LENGTH_SHORT` 和 `LENGTH_LONG` 的地方，最终得出结论，而这里是根据一些合理推断来简化追踪过程，更快达到目标，这在一些场景下是可取和必要的。*

### 能不能弹一个时间超长的 Toast？

注：这里探讨的是能否直接通过 Toast 提供的公开 API 做到，网络上能搜索到的使用 Timer、反射、自定义等方式达到弹出一个超长时间 Toast 目的的方法不在讨论范围内。

我们在 Toast 类的源码里看一下跟设置时长相关的代码：

文件 [platform_frameworks_base/core/java/android/widget/Toast.java][4]

```java
...

    /** @hide */
    @IntDef({LENGTH_SHORT, LENGTH_LONG})
    @Retention(RetentionPolicy.SOURCE)
    public @interface Duration {}

...

    /**
     * Set how long to show the view for.
     * @see #LENGTH_SHORT
     * @see #LENGTH_LONG
     */
    public void setDuration(@Duration int duration) {
        mDuration = duration;
        mTN.mDuration = duration;
    }

...

    /**
     * Make a standard toast that just contains a text view.
     *
     * @param context  The context to use.  Usually your {@link android.app.Application}
     *                 or {@link android.app.Activity} object.
     * @param text     The text to show.  Can be formatted text.
     * @param duration How long to display the message.  Either {@link #LENGTH_SHORT} or
     *                 {@link #LENGTH_LONG}
     *
     */
    public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        return makeText(context, null, text, duration);
    }

...
```

其实从上面 `setDuration` 和 `makeText` 的注释已经可以看出，duration 只能取值 `LENGTH_SHORT` 和 `LENGTH_LONG`，除了注释之外，还使用了 `@Duration` 注解来保证此事。`Duration` 自身使用了 `@IntDef` 注解，它用于限制可以取的值。

文件 [platform_frameworks_base/core/java/android/annotation/IntDef.java][7]

```java
/**
 * Denotes that the annotated element of integer type, represents
 * a logical type and that its value should be one of the explicitly
 * named constants. If the {@link #flag()} attribute is set to true,
 * multiple constants can be combined.
 * ...
 */
```

不信邪的我们可以快速在一个 Demo Android 工程里写一句这样的代码试试：

```java
Toast.makeText(this, "Hello", 2);
```

Android Studio 首先就不会同意，警告你 `Must be one of: Toast.LENGTH_SHORT, Toast.LENGTH_LONG`，但实际这段代码是可以通过编译的，因为 `Duration` 注解的 `Retention` 为 `RetentionPolicy.SOURCE`，我的理解是该注解主要能用于 IDE 的智能提示警告，编译期就被丢掉了。

但即使 duration 能传入 `LENGTH_SHORT` 和 `LENGTH_LONG` 以外的值，也并没有什么卵用，别忘了这里设置的只是一个 flag，真正计算的时候是 `long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;`，即 duration 为 `LENGTH_LONG` 时时长为 3.5 秒，其它情况都是 2 秒。

所以我们可以得出 **结论：无法通过 Toast 提供的公开 API 直接弹出超长时间的 Toast。**（如节首所述，可以通过一些其它方式实现类似的效果）

### Toast 能不能在非 UI 线程调用？

这个问题适合用一个 demo 来解答。

我们创建一个最简单的 App 工程，然后在启动 Activity 的 onCreate 方法里添加这样一段代码：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        Toast.makeText(MainActivity.this, "Call toast on non-UI thread", Toast.LENGTH_SHORT)
                .show();
    }
}).start();
```

啊哦~很遗憾程序直接挂掉了。

```
11-07 13:35:33.980 2020-2035/org.mazhuang.androiduidemos E/AndroidRuntime: FATAL EXCEPTION: Thread-77
    java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
        at android.os.Handler.<init>(Handler.java:197)
        at android.os.Handler.<init>(Handler.java:111)
        at android.widget.Toast$TN.<init>(Toast.java:324)
        at android.widget.Toast.<init>(Toast.java:91)
        at android.widget.Toast.makeText(Toast.java:238)
        at org.mazhuang.androiduidemos.MainActivity$1.run(MainActivity.java:27)
        at java.lang.Thread.run(Thread.java:856)
```

从崩溃报告里我们能得到的关键信息是 `Can't create handler inside thread that has not called Looper.prepare()`，那我们在 toast 前面加一句 `Looper.prepare()` 试试？这次不崩溃了，但依然不弹出 Toast，毕竟，这个线程在调用完 `show()` 方法后就直接结束了，至于为什么调用 Toast 的线程结束与否会对 Toast 的显示隐藏等起影响，在本文的后面的章节里会进行分析。

从崩溃提示来看，Android 并没有限制在非 UI 线程里使用 Toast，只是线程是一个有 Looper 的线程。于是我们尝试构造如下代码，发现可以成功从非 UI 线程弹出 toast 了：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        final int MSG_TOAST = 101;
        final int MSG_QUIT = 102;

        Looper.prepare();

        final Handler handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {

                switch (msg.what) {
                    case MSG_TOAST:
                        Toast.makeText(MainActivity.this, "Call toast on non-UI thread", Toast.LENGTH_SHORT)
                                .show();
                        sendEmptyMessageDelayed(MSG_QUIT, 4000);
                        return;

                    case MSG_QUIT:
                        Looper.myLooper().quit();
                        return;
                }

                super.handleMessage(msg);
            }
        };

        handler.sendEmptyMessage(MSG_TOAST);

        Looper.loop();
    }
}).start();
```

至于为什么 `sendEmptyMesageDelayed(MSG_QUIT, 4000)` 里的 delayMillis 我设成了 4000，这里卖个关子，感兴趣的同学可以把这个值调成 0、1000 等等看一下效果，会有一些意想不到的情况发生。

到此，我们可以得出 **结论：可以在非 UI 线程里调用 Toast，但是得是一个有 Looper 的线程。**

ps. 上面这一段演示代码让人感觉为了弹出一个 Toast 好麻烦，也可以采用 Activity.runOnUiThread、View.post 等方法从非 UI 线程将逻辑切换到 UI 线程里执行，直接从 UI 线程里弹出。

*知识点：这里如果对 Looper、Handler 和 MessageQueue 有所了解，就容易理解多了，预计下一篇对这三剑客进行讲解。*

### 应用在后台时能不能 Toast？

[1]: https://developer.android.com/reference/android/widget/Toast.html
[2]: https://developer.android.com/guide/topics/ui/notifiers/toasts.html
[3]: https://github.com/aosp-mirror/platform_frameworks_base
[4]: https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/widget/Toast.java
[5]: https://github.com/aosp-mirror/platform_frameworks_base/blob/master/services/core/java/com/android/server/notification/NotificationManagerService.java
[6]: https://github.com/aosp-mirror/platform_frameworks_base/blob/master/services/core/java/com/android/server/policy/PhoneWindowManager.java
[7]: https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/annotation/IntDef.java
