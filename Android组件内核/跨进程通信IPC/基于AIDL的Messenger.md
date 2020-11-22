## 四、Messenger

### 4.1 说明

* 通过Messenger可以在不同的进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松的实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。只能传输Bundle支持的数据格式。

### 4.1 Messenger实例

* 服务端进程：创建一个Service来处理客户端的连接请求；创建一个Handler并通过它来创建一个Messenger对象；在Service的onBind中返回这个Messenger对象底层的Binder即可。在清单文件注册一下。

* 客户端进程：首先绑定服务端的Service；绑定成功后用服务端返回的IBinder对象创建一个Messager，通过这个Messenger就可以向服务器发送消息了，发送类型必须为Message对象； 如果需要服务端能够回应客户端，在客户端需要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replayTo参数就可以回应客户端

* 查看实例代码请点击 [IPC之Messenger的demo核心代码](https://github.com/nullWolf007/ToolProject/tree/master/IPC之Messenger的demo核心代码) 