[TOC]

## Socket解析

#### 参考转载

* [socket技术详解（看清socket编程）](https://blog.csdn.net/panker2008/article/details/46502783?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.control)
* [TCP三次握手详解-深入浅出(有图实例演示)](https://blog.csdn.net/jun2016425/article/details/81506353)
* [Socket通信原理](https://juejin.cn/post/6844904022567026695#heading-4)

### 一、前提知识

#### 1.1 TCP/IP协议

* TCP/IP（Transmission Control Protocol/Internet Protocol，传输控制协议/网际协议）是指能够在多个不同网络间实现信息传输的协议簇。TCP/IP协议不仅仅指的是[TCP](https://baike.baidu.com/item/TCP/33012) 和[IP](https://baike.baidu.com/item/IP/224599)两个协议，而是指一个由[FTP](https://baike.baidu.com/item/FTP/13839)、[SMTP](https://baike.baidu.com/item/SMTP/175887)、TCP、[UDP](https://baike.baidu.com/item/UDP/571511)、IP等协议构成的协议簇， 只是因为在TCP/IP协议中TCP协议和IP协议最具代表性，所以被称为TCP/IP协议。

#### 1.2 五层体系结构

![五层体系结构](..\..\..\images\设计思想解读开源框架库\网络访问框架\网络基础\五层体系结构.png)

#### 1.3 TCP/IP所处的体系结构

* TCP/IP在每层都有自己的协议

![TCPIP所处体系结构](..\..\..\images\设计思想解读开源框架库\网络访问框架\网络基础\TCPIP所处体系结构.png)

* 应用层：TFTP，HTTP，SNMP，FTP，SMTP，DNS，Telnet 等等

* 传输层：TCP，UDP

* 网络层：IP，ICMP，OSPF，EIGRP，IGMP等

* 数据链路层：SLIP，CSLIP，PPP，MTU等

#### 1.4 TCP的三次握手

![TCP三次握手详细流程](..\..\..\images\设计思想解读开源框架库\网络访问框架\网络基础\TCP三次握手详细流程.png)

* 第一步：客户端发送SYN消息，其序号为J
* 第二步：服务器端收到SYN消息，发送ACK消息和SYN消息，其中ACK的序号为J+1，SYN序号为K
* 第三步：客户端收到后，发送ACK消息，其序号为K+1

#### 1.5 TCP的四次挥手

![TCP四次回收详细流程](..\..\..\images\设计思想解读开源框架库\网络访问框架\网络基础\TCP四次回收详细流程.png)

* 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
* 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
* 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
* 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
* 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
* 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。

#### 1.6 Socket

##### 1.6.1 那么Socket在哪呢？

* 从图中我们就可以知道Socket在哪了

![Socket在哪呢](..\..\..\images\设计思想解读开源框架库\网络访问框架\网络基础\Socket在哪呢.png)

##### 1.6.2 Socket是什么？

* Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议

#### 1.7 Socket和Http的异同

* Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议。
* Http协议：简单的对象访问协议，对应于应用层。Http协议是基于TCP链接的。
* Http是基于Socket（Tcp Socket）来实现的，不过由于其功能需求，Http连接一般设计为短连接的形式
* Socket连接一般指的是其长连接的形式，但是使用Socket可以实现各种（短连接，长连接，无连接等）

### 二、Socket基本实现步骤

#### 2.1 基于TCP协议

##### 2.1.1 服务器端

* **服务端**首先声明一个`**ServerSocket**`对象并且指定端口号
* 然后调用Serversocket的**`accept()`**方法接受客户端的数据。accept()方法在没有数据进行接受时处于堵塞状态。（`Socket socket = serversocket.accept()`），
* 一旦接受数据，通过**`inputstream`**读取接受的数据。

##### 2.1.2 客户端

* **客户端**创建一个Socket对象，执行服务器端的ip地址和端口号（`Socket socket = new Socket("172.168.10.108", 8080);`）
* 创建一个InputStream用户读取要发送的文件。
*  获取Socket的OutputStream对象用于发送数据（`OutputStream outputstream = socket.getOutputStream();`），
* 将要发送的数据写入到**`outputstream`**
* 调用outputStream.flush()，把数据发送给服务端

#### 2.2 基于UDP协议

##### 2.2.1 服务器端

* **服务器**端首先创建一个**`DatagramSocket`**对象，并且指定监听端口。
* 接下来创建一个空的DatagramSocket对象用于接收数据（`byte data[] = new byte[1024]; DatagramSocket packet = new DatagramSocket(data, data.length);`），
* 使用`DatagramSocket`的**`receive()`**方法接受客户端发送的数据，`receive()`与`serversocket`的`accept()`方法**类似**，在没有数据进行接受时处于堵塞状态。

##### 2.2.2 客户端

* **客户端**也创建个`DatagramSocket`对象，并且指定监听的端口。
* 接下来创建一个**`InetAddress`**对象，这个对象类似于一个网络的发送地址（`InetAddress serveraddress = InetAddress.getByName("172.168.1.120")`）。
* 定义要发送的一个字符串，创建一个`DatagramPacket`对象，并指定要将该数据包发送到网络对应的那个地址和端口号，
* 最后使用DatagramSocket的对象的**`send()`**发送数据。

### 三、Android基于Sokcet简单实现

#### 3.1 基于TCP

##### 3.1.1 客户端

```java
protected void connectServerWithTCPSocket() {

        Socket socket;
        try {// 创建一个Socket对象，并指定服务端的IP及端口号
            socket = new Socket("192.168.1.32", 1989);
            // 创建一个InputStream用户读取要发送的文件。
            InputStream inputStream = new FileInputStream("e://a.txt");
            // 获取Socket的OutputStream对象用于发送数据。
            OutputStream outputStream = socket.getOutputStream();
            // 创建一个byte类型的buffer字节数组，用于存放读取的本地文件
            byte buffer[] = new byte[4 * 1024];
            int temp = 0;
            // 循环读取文件
            while ((temp = inputStream.read(buffer)) != -1) {
                // 把数据写入到OuputStream对象中
                outputStream.write(buffer, 0, temp);
            }
            // 发送读取的数据到服务端
            outputStream.flush();

            /** 或创建一个报文，使用BufferedWriter写入,看你的需求 **/
//            String socketData = "[2143213;21343fjks;213]";
//            BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(
//                    socket.getOutputStream()));
//            writer.write(socketData.replace("\n", " ") + "\n");
//            writer.flush();
            /************************************************/
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

```

##### 3.1.2 服务器端

```java
public void ServerReceviedByTcp() {
        // 声明一个ServerSocket对象
        ServerSocket serverSocket = null;
        try {
            // 创建一个ServerSocket对象，并让这个Socket在1989端口监听
            serverSocket = new ServerSocket(1989);
            // 调用ServerSocket的accept()方法，接受客户端所发送的请求，
            // 如果客户端没有发送数据，那么该线程就停滞不继续
            Socket socket = serverSocket.accept();
            // 从Socket当中得到InputStream对象
            InputStream inputStream = socket.getInputStream();
            byte buffer[] = new byte[1024 * 4];
            int temp = 0;
            // 从InputStream当中读取客户端所发送的数据
            while ((temp = inputStream.read(buffer)) != -1) {
                System.out.println(new String(buffer, 0, temp));
            }
            serverSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

#### 3.2 基于UDP

##### 3.2.1 客户端发送数据

```java
protected void connectServerWithUDPSocket() {
        
        DatagramSocket socket;
        try {
            //创建DatagramSocket对象并指定一个端口号，注意，如果客户端需要接收服务器的返回数据,
            //还需要使用这个端口号来receive，所以一定要记住
            socket = new DatagramSocket(1985);
            //使用InetAddress(Inet4Address).getByName把IP地址转换为网络地址  
            InetAddress serverAddress = InetAddress.getByName("192.168.1.32");
            //Inet4Address serverAddress = (Inet4Address) Inet4Address.getByName("192.168.1.32");  
            String str = "[2143213;21343fjks;213]";//设置要发送的报文  
            byte data[] = str.getBytes();//把字符串str字符串转换为字节数组  
            //创建一个DatagramPacket对象，用于发送数据。  
            //参数一：要发送的数据   参数二：数据的长度  
            //参数三：服务端的网络地址  参数四：服务器端端口号 
            DatagramPacket packet = new DatagramPacket(data, data.length ,serverAddress ,10025);  
            socket.send(packet);//把数据发送到服务端。  
        } catch (SocketException e) {
            e.printStackTrace();
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }  
    }
```

##### 3.2.2 客户端接受数据

```java
public void ReceiveServerSocketData() {
        DatagramSocket socket;
        try {
            //实例化的端口号要和发送时的socket一致，否则收不到data
            socket = new DatagramSocket(1985);
            byte data[] = new byte[4 * 1024];
            //参数一:要接受的data 参数二：data的长度
            DatagramPacket packet = new DatagramPacket(data, data.length);
            socket.receive(packet);
            //把接收到的data转换为String字符串
            String result = new String(packet.getData(), packet.getOffset(),
                    packet.getLength());
            socket.close();//不使用了记得要关闭
            System.out.println("the number of reveived Socket is  :" + flag
                    + "udpData:" + result);
        } catch (SocketException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

```

##### 3.2.3 服务器端

```java
public void ServerReceviedByUdp(){
        //创建一个DatagramSocket对象，并指定监听端口。（UDP使用DatagramSocket）  
        DatagramSocket socket;
        try {
            socket = new DatagramSocket(10025);
            //创建一个byte类型的数组，用于存放接收到得数据  
            byte data[] = new byte[4*1024];  
            //创建一个DatagramPacket对象，并指定DatagramPacket对象的大小  
            DatagramPacket packet = new DatagramPacket(data,data.length);  
            //读取接收到得数据  
            socket.receive(packet);  
            //把客户端发送的数据转换为字符串。  
            //使用三个参数的String方法。参数一：数据包 参数二：起始位置 参数三：数据包长  
            String result = new String(packet.getData(),packet.getOffset() ,packet.getLength());  
        } catch (SocketException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }  
    }

```






