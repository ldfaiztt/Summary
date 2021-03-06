Android性能优化
===

### 控制业务处理时间

由于人眼无法感知超过每秒 60 帧的画面更新频率，因此将 60FPS 作为 App 性能的衡量标准。
这就意味这每一帧要在 16MS 内处理完（16MS = 1000 / 60），否则就会发生卡顿现象。
如某个操作耗费 24MS，系统在得到 VSYNC 信号时无法正常渲染，就发生丢帧现象，用户在 32ms 内看到的会是同一帧画面。


### 刷新频率与帧率

1. Refresh Rate：表示屏幕在一秒内刷新屏幕的次数，这取决于硬件的固定参数，如 60Hz。
2. Frame Rate：表示 GPU 在一秒内绘制操作的帧数，例如 30fps，60fps。

GPU 会获取图形数据进行渲染，然后硬件负责把渲染后的内容呈现到屏幕上，而刷新频率和帧率无法一直保持相同的节奏。
如果发生帧率与刷新频率不一致的情况，就会容易出现撕裂现象，画面上下两部分显示内容发生断裂，来自不同的两帧数据发生重叠。

- [理解图像渲染里面的双重与三重缓存机制一](http://source.android.com/devices/graphics/index.html)
- [理解图像渲染里面的双重与三重缓存机制二](http://article.yeeyan.org/view/37503/304664)

通常，帧率超过刷新频率只是理想情况，在超过 60fps 的情况下，
GPU 所产生的帧数据会因为等待 VSYNC 的刷新信息而被 Hold 住，
这样能够保持每次刷新都有实际的新的数据可以显示。但更多的情况是帧率小于刷新频率。
在这种情况下，某些帧显示的画面内容就会与上一帧的画面相同。糟糕的事情是，帧率从超过 60fps 突然掉到 60FPS 以下，这样就会发生卡顿掉帧的不顺滑的情况。

### 自定义 View 优化

非可见的 UI 组件进行绘制更新会导致 Overdraw，Android 系统会通过避免绘制那些完全不可见的组件来尽量减少 Overdraw。
但对于自定义的 View (重写了 onDraw 方法)，Android 系统无法检测具体在 onDraw 里面会执行什么操作，系统无法监控并自动优化，也就无法避免 Overdraw。
但可通过如下方法进行优化：

1. 通过 canvas.clipRect() 指定一块矩形区域，只有在该区域内才会被绘制，其他的区域会被忽视。
2. 使用 canvas.quickreject() 判断是否和某个矩形相交，从而跳过那些非矩形区域内的绘制操作。

### 避免频繁 GC

Android 对 JVM 的 GC 机制进行了优化，为一个三级 Generation 内存模型，最近分配的对象会存放在 Young Generation 区域，
当这个对象在这个区域停留的时间达到一定程度，它会被移动到 Old Generation 区域，最后到 Permanent Generation 区域。
每一个级别的内存区域都有固定的大小，此后不断有新的对象被分配到此区域，当这些对象总的大小快达到这一级别内存区域的阀值时，会触发 GC 的操作，以便腾出空间来存放其他新的对象。

GC 时所有线程都为暂停状态，Generation 中的对象数量相同的情况下，Young Generation GC 操作时间最短，Old Generation 其次，Permanent Generation 最长。

通常来说，单个的 GC 并不会占用太多时间，但是大量不停的 GC 操作则会显著占用帧间隔时间(16ms)。
如果在帧间隔时间里面做了过多的 GC 操作，那么自然其他类似计算，渲染等操作的可用时间就变得少了。

导致 GC 频繁原因：

1. 大量的对象被创建又在短时间内马上被释放。
2. 瞬间产生大量的对象会严重占用Young Generation 的内存区域，当达到阀值，剩余空间不够的时候，也会触发GC。
即使每次分配的对象占用了很少的内存，但是他们叠加在一起会增加 Heap 的压力，从而触发更多其他类型的GC。
这个操作有可能会影响到帧率，并使得用户感知到性能问题。

查看内存抖动：

1. 在 Memory Monitor 里面查看到短时间发生了多次内存的涨跌，很有可能发生了内存抖动。
2. 通过 Allocation Tracker 来查看在短时间内，同一个栈中不断进出的相同对象，，也是内存抖动的典型信号之一。

建议：

1. 免在 For 循环里分配对象占用内存。
2. 避免在自定义 View 中的 onDraw 方法。
3. 对于那些无法避免需要创建对象的情况，可考虑对象池模型，通过对象池来解决频繁创建与销毁的问题，注意结束使用之后，手动释放对象池中的对象。

### 内存泄露定位

内存泄漏指程序不再使用的对象无法被 GC 识别，这样就导致这个对象一直留在内存当中，占用了宝贵的内存空间。
显然，这还使得每级 Generation 的内存区域可用空间变小，GC 就会更容易被触发，从而引起性能问题。

示例：某个 Activity 退出时，意思有内存泄露，如何定位？

1. 在 Activity 处于前台时，用 Heap Tool 获取一份当前状态的内存快照。
2. 创建一个几乎不占内存的空白 Activity 用来给前一个 Activity 进行跳转。
3. 在跳转到该空白 Activity 时候主动调用 System.gc() 方法触发 GC 操作。
4. 如果 Activity 的内存没有完全正确释放，使用 Alocation Track Tool 来仔细查找具体的可疑对象。
5. 从空白 Activity 开始监听，启动到观察 Activity，然后再回到空白 Activity 结束监听，分析内存泄漏的原因。

### 电量消耗优化

1. 减少唤醒屏幕次数及持续时间，使用 WakeLock 处理唤醒问题，能够正确执行唤醒操作并根据设定及时关闭操作进入睡眠状态。
2. 非必须立即执行的操作，例如上传歌曲，图片处理等，可使用 JobScheduler API 接口等到设备处于充电状态或者电量充足的时候再进行。[【参考这里】](http://hukai.me/android-training-course-in-chinese/background-jobs/scheduling/index.html)
3. 将零散的网络请求打包进行一次操作，避免过多的无线信号引起的电量消耗。关于网络请求引起无线信号的电量消耗，[【参考这里】](http://hukai.me/android-training-course-in-chinese/connectivity/efficient-downloads/efficient-network-access.html)
4. 设置选项中可查看指定 App 的电量消耗数据。
5. Android 5.0 支持通过 Battery Historian Tool 查看详细的电量消耗。程序被唤醒的频率，由谁唤醒，持续多长时间。

### 相关工具

##### 常用工具

1. 使用 HierarchyViewer 查找 Activity 中布局是否过于复杂，如重叠元素过多，动画执行次数过多。
2. 使用开发者选项中的 Show GPU Overdraw 观察是否有过度绘制情况。蓝色，淡绿，淡红，深红代表了 4 种不同程度的 Overdraw 情况，应尽量减少红色 Overdraw，看到更多的蓝色区域。
3. 使用 TraceView 观察 CPU 执行情况，排查性能瓶颈。

##### GPU渲染工具

可通过 android 内置工具来调试性能问题：

1. 打开开发者选项。
2. 选择 Profile GPU Rendering。
3. 选择 On screen as bars。

此时，可在屏幕上看到 GPU 绘制信息，分别关于 StatusBar，NavBar，激活的程序 Activity 区域的 GPU Rending 信息。
随着界面的刷新，界面上会滚动显示垂直的柱状图来表示每帧画面所需要渲染的时间，柱状图越高表示花费的渲染时间越长。
界面上有一根绿色的横线，代表16ms，确保每一帧花费的总时间都低于该横线，即可避免卡顿。

每一条柱状线都包含三部分：

1. 蓝色代表测量绘制 Display List 的时间
2. 红色代表 OpenGL 渲染 Display List 所需要的时间
3. 黄色代表 CPU 等待 GPU 处理的时间。


##### Android Studio 提供的性能分析工具

1. Memory Monitor：查看 app 所占用内存，及发生 GC 的时间，看短时间内是否有大量 GC。
2. Allocation Tracker：使用此工具来追踪内存的分配，前面有提到过。
3. Heap Tool：查看当前内存快照，便于对比分析哪些对象有可能是泄漏了的，请参考前面的Case。
