JetPack分类有四种，分别是Architecture、Foundationy、Behavior、UI。

![img](https:////upload-images.jianshu.io/upload_images/20285170-11b52f3aa91c0be7.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

------

①Architecture（架构组件）

  Architecture指的是架构组件，帮助开发者设计稳健、可测试且易维护的应用。架构组件可以说是对应用开发帮助最大的组件，本系列也是围绕着架构组件进行讲解。

  DataBinding：以声明方式将可观察数据绑定到界面元素，通常和ViewModel配合使用。

  Lifecycle：用于管理Activity和Fragment的生命周期，可帮助开发者生成更易于维护的轻量级代码。

  LiveData: 在底层数据库更改时通知视图。它是一个可观察的数据持有者，与常规observable不同，LiveData是生命周期感知的。

  Navigation:处理应用内导航。

  Paging:可以帮助开发者一次加载和显示小块数据，按需加载部分数据可减少网络带宽和系统资源的使用。

  Room:友好、流畅的访问SQLite数据库。它在SQLite的基础上提供了一个抽象层，允许更强大的数据库访问。

  ViewModel: 以生命周期的方式管理界面相关的数据，通常和DataBinding配合使用，为开发者实现MVVM架构提供了强有力的支持。

  WorkManager: 管理Android的后台的作业，即使应用程序退出或设备重新启动也可以运行可延迟的异步任务。

