# Profiler的使用教程

### 参考链接

* [利用 Android Profiler 测量应用性能](https://developer.android.com/studio/profile/android-profiler?hl=zh_cn)
* 详细请参考上述网站，安卓官网介绍的很详细

## 一、Android Profiler整体界面介绍

![Profiler整体界面](..\..\images\工具使用\AS自带工具\Profiler整体界面.png)

* 说明：下面的①到⑤对应上图的红色1到5

① Android Profiler 显示当前正在分析的进程和设备。

② 在 **Sessions** 窗格中，选择要查看的会话，或启动一个新的分析会话。     

③ 使用缩放按钮控制要查看时间轴范围，或使用 **Attach to live** 按钮跳转到实时更新。

④ 事件时间轴显示与用户输入相关的事件，包括键盘 Activity、音量控制更改和屏幕旋转。

⑤ 共享时间轴视图，包括 CPU、内存、网络和耗电量图表。

## 二、Memory Profiler

### 2.1 预览

![MemoryProfiler概览](..\..\images\工具使用\AS自带工具\MemoryProfiler概览.png)

①用于强制执行垃圾回收事件的按钮。

②用于[捕获堆转储](https://developer.android.com/studio/profile/memory-profiler?hl=zh_cn#capture-heap-dump)的按钮。

**注意**：只有在连接到搭载 Android 7.1（API 级别 25）或更低版本的设备时，才会在堆转储按钮右侧显示用于[记录内存分配](https://developer.android.com/studio/profile/memory-profiler?hl=zh_cn#record-allocations)的按钮。

③一个下拉菜单，用于指定分析器捕获内存分配的频率。选择适当的选项可帮助您[在分析时提高应用性能](https://developer.android.com/studio/profile/memory-profiler?hl=zh_cn#performance)。    

④用于放大/缩小时间轴的按钮。

⑤用于跳转到实时内存数据的按钮。

⑥事件时间轴，显示活动状态、用户输入事件和屏幕旋转事件。

⑦内存使用量时间轴，它包括以下内容：

- 一个堆叠图表，显示每个内存类别当前使用多少内存，如左侧的 y 轴以及顶部的彩色键所示。
- 一条虚线，表示分配的对象数，如右侧的 y 轴所示。
- 每个垃圾回收事件的图标。

 ### 2.2 如何计算内存

![内存计算](..\..\images\工具使用\AS自带工具\内存计算.png)

* **Java**：从Java或Kotlin代码分配的对象的内存
* **Native**：从C或C++代码分配的对象的内存。即使您的应用中不使用C++，您也可能会看到此处使用的一些原生内存，因为 Android 框架使用原生内存代表您处理各种任务，如处理图像资源和其他图形时，即使您编写的代码采用 Java 或 Kotlin 语言。
* **Graphics**：图形缓冲区队列向屏幕显示像素（包括 GL 表面、GL 纹理等等）所使用的内存。（请注意，这是与 CPU 共享的内存，不是 GPU 专用内存。）
* **Stack**：您的应用中的原生堆栈和 Java 堆栈使用的内存。这通常与您的应用运行多少线程有关。
* **Code**：您的应用用于处理代码和资源（如 dex 字节码、经过优化或编译的 dex 代码、.so 库和字体）的内存。
* **Others**：您的应用使用的系统不确定如何分类的内存。
* **Allocated**：您的应用分配的 Java/Kotlin 对象数。此数字没有计入 C 或 C++ 中分配的对象。

### 2.3 查看内存分配

内存分配为您显示内存中的每个 Java 对象和 JNI 引用是如何分配的。具体而言，Memory Profiler 可为您显示有关对象分配的以下信息：

- 分配了哪些类型的对象以及它们使用多少空间。
- 每个分配的堆栈轨迹，包括在哪个线程中。
- 对象在何时被取消分配（仅当使用搭载 Android 8.0 或更高版本的设备时）。

具体操作步骤如下：在时间轴上拖动以选择要查看哪个区域的分配（如视频 1 中所示）。不需要开始记录会话，因为 Android 8.0 及更高版本附带设备内置分析工具，可持续跟踪您的应用分配。

![内存分配](..\..\images\工具使用\AS自带工具\内存分配.png)

**要检查分配记录，请按以下步骤操作：**

1. 浏览列表以查找堆计数异常大且可能存在泄漏的对象。为帮助查找已知类，点击**Class Name**列标题以按字母顺序排序。然后，点击一个类名称。此时右侧将出现**Instance View**窗格，显示该类的每个实例，如图 3 所示。    
   - 或者，您也可以快速找到对象，方法是点击 **Filter** 图标 ![img](https://developer.android.com/studio/images/buttons/profiler_filter.png?hl=zh_cn)，或按 Ctrl+F 键（在 Mac 上，按 Command+F 键），然后在搜索字段中输入类或软件包名称。如果从下拉菜单中选择 **Arrange by callstack**，还可以按方法名称搜索。如果要使用正则表达式，请勾选 **Regex** 旁边的框。如果您的搜索查询区分大小写，请勾选 **Match case** 旁边的框。
2. 在 **Instance View** 窗格中，点击一个实例。此时下方将出现 **Call Stack** 标签，显示该实例被分配到何处以及哪个线程中。
3. 在 **Call Stack** 标签中，右键点击任意行并选择 **Jump to Source**，以在编辑器中打开该代码。

**您可以使用已分配对象列表上方的两个菜单来选择要检查的堆以及如何组织数据。**

从左侧的菜单中，选择要检查的堆：

- **default heap**：当系统未指定堆时。
- **image heap**：系统启动映像，包含启动期间预加载的类。此处的分配保证绝不会移动或消失。
- **zygote heap**：写时复制堆，其中的应用进程是从 Android 系统中派生的。
- **app heap**：您的应用在其中分配内存的主堆。
- **JNI heap**：显示 Java 原生接口 (JNI) 引用被分配和释放到什么位置的堆。     

从右侧的菜单中，选择如何安排分配：

- **Arrange by class**：根据类名称对所有分配进行分组。这是默认选项。
- **Arrange by package**：根据软件包名称对所有分配进行分组。
- **Arrange by callstack**：将所有分配分组到其对应的调用堆栈。

**为了在分析时提高应用性能，Memory Profiler 在默认情况下会定期对内存分配进行采样。在运行 API 级别 26 或更高级别的设备上进行测试时，您可以使用 Allocation Tracking下拉菜单来更改此行为。可用选项如下：**

- **Full**：捕获内存中的所有对象分配。这是 Android Studio 3.2 及更低版本中的默认行为。如果您有一个分配了大量对象的应用，则可能会在分析时观察到应用的运行速度明显减慢。
- **Sampled**：定期对内存中的对象分配进行采样。这是默认选项，在分析时对应用性能的影响较小。在短时间内分配大量对象的应用仍可能会表现出明显的速度减慢。
- **Off**：停止跟踪应用的内存分配。

