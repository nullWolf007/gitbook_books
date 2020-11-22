## 六、View的绘制

   ### 6.1 绘制流程

   * View的绘制流程从ViewRoot的performTraversals方法开始，经过measure、layout和draw三个过程进行绘制。measure用来测量View的宽和高；layout用来确定View在父容器中放置位置；draw负责将View绘制在屏幕上。

   ## 6.2 MeasureSpec

   #### 6.2.1 说明

   * MeasureSpec代表一个32位int值，高2位代表SpecMode，低30位代表SpecSize。SpecMode是指测量模式，而SpecSize指在某种测量模式下的规格大小。

   #### 6.2.2 SpecMode

   * UNSPECIFIED

     > 父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量状态

   * EXACTLY

     > 父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的指。它对应于LayoutParams中的match_parent和具体的数值这两种模式

   * AT_MOST

     > 父容器指定一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现，它对应于LayoutParams中的wrap_content

