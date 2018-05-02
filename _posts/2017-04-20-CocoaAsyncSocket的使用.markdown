---
layout: post
title: CocoaAsyncSocket的使用
date: 2017-04-20
---

#### 前言
最近项目中涉及到socket通信这块；所以有幸有时间大概看了一下这一块；目前还在实现阶段，因此现在还不能去些具体的实现过程；现在只大概描述一下这几天看的资料和自己的一点心得吧；等项目实现之后会将具体的实现流程写出来以供大家参考；

#### Socket通信基础
Socket起源于Unix，而Unix/[Linux](http://lib.csdn.net/base/linux)基本哲学之一就是“一切皆文件”，都可以用“打开open –> 读写read/write –> 关闭close”模式来操作。我的理解就是Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）；

套接字（socket）是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：```连接使用的协议，本地主机的IP地址，本地进程的协议端口，远地主机的IP地址，远地进程的协议端口。```因为TCP协议+端口号可以唯一标识一台计算机中的进程;

创建Socket连接时，可以指定使用的传输层协议，Socket可以支持不同的传输层协议（TCP或UDP），当使用TCP协议进行连接时，该Socket连接就是一个TCP连接。

由于通常情况下Socket连接就是TCP连接，因此Socket连接一旦建立，通信双方即可开始相互发送数据内容，直到双方连接断开。但在实际网络应用中，客户端到服务器之间的通信往往需要穿越多个中间节点，例如路由器、网关、防火墙等，大部分防火墙默认会关闭长时间处于非活跃状态的连接而导致 Socket 连接断连，因此需要通过轮询告诉网络，该连接处于活跃状态。

#### TCP/IP协议
再者就是已经被讲烂了的TCP/IP协议了，但是我们大多数人对其还不是很熟悉，只是整天在各种资料上看到听到，但是我们似乎从来都没有用心去细看；但是这里还是不得不说🤐；首先就是最基本的TCP的三次握手：
第一次握手：客户端发送syn包（syn=j）到服务器，并进入SYN_SEND状态，等待服务器确认；
第二次握手：服务器收到syn包，确认客户端的syn(ack=j+1)，同时自己也发送一个syn包（syn=k），即syn+ack包，此时服务器进入SYN_RECV状态；
第三次握手：客户端收到服务器的syn+ack包，向服务器发送确认包ack(ack=k+1),此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手；

![安全的tcp连接.png](http://upload-images.jianshu.io/upload_images/1867963-d5c724d8d5c96725.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### CocoaAsyncSocket 分类
CocoaAsyncSocket 主要分GCDAsyncSocket.h 和 GCDAsyncUdpSocket.h这两个类；前者是针对TCP进行通信的，而后者是针对UDP进行通信；

##### UDP广播接收
```
NSError *error;
//绑定本地端口，端口号和后台协商指定；
[self.asyncUdpSocket bindToPort:8989 error:&error];
if (error) {SALog(@"绑定端口:%@",error);return;}

//启用广播；
[self.asyncUdpSocket enableBroadcast:YES error:&error];
if (error) {SALog(@"启用广播:%@",error);return;}

//开启接收数据,不开启的话会接收不到数据；
[self.asyncUdpSocket beginReceiving:&error];
if (error) {SALog(@"开启接收数据:%@",error);return;}

//重复发送广播
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(broadcast) userInfo:nil repeats:YES];
[timer fire];

- (void)broadcast{
    NSString *msg = @"Hello world!";
    [self.socket sendData:[msg dataUsingEncoding:NSUTF8StringEncoding] toHost:@"192.168.0.31" port:9999 withTimeout:-1 tag:100];
}
```
/** 在广播的时候IP地址192.168.0.31是一个广播地址，这个地址可以通过自己手机的IP地址和子网掩码相与进行计算出来，实在不知道可以放一个所有地址的广播地址255.255.255.255 。 这样只要在这个局域网中的所有设备都可以接收这个广播 */

然后设置代理，监听接收的数据：
```
- (GCDAsyncUdpSocket *)asyncUdpSocket{
    if (!_asyncUdpSocket) {
        _asyncUdpSocket = [[GCDAsyncUdpSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_global_queue(0, 0)];
    }
    return _asyncUdpSocket;
}

代理方法接收数据
- (void)udpSocket:(GCDAsyncUdpSocket *)sock didReceiveData:(NSData *)data fromAddress:(NSData *)address withFilterContext:(id)filterContext{
    SALog(@"UDP接收数据……………………………………………………");
    if (!self.isConnect) {
        NSString *msg = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        NSArray *msgArray = [msg componentsSeparatedByString:@"|"];
        NSData *msgData = [msgArray.lastObject dataUsingEncoding:NSUTF8StringEncoding];
        NSError *error;
        NSDictionary *dataDict = [NSJSONSerialization JSONObjectWithData:msgData options:NSJSONReadingMutableContainers error:&error];
        if (!error) {
            self.serverHost = dataDict[@"ip"];
            self.port = [dataDict[@"port"] intValue];
            //连接广播的IP地址
            [self openConnection];
        }
    }
}

- (void)udpSocket:(GCDAsyncUdpSocket *)sock didNotConnect:(NSError *)error{
    SALog(@"断开连接");
}

- (void)udpSocket:(GCDAsyncUdpSocket *)sock didSendDataWithTag:(long)tag{
    SALog(@"发送的消息");
}

- (void)udpSocket:(GCDAsyncUdpSocket *)sock didConnectToAddress:(NSData *)address{
    SALog(@"已经连接");
}

- (void)udpSocketDidClose:(GCDAsyncUdpSocket *)sock withError:(NSError *)error{
    SALog(@"断开连接");
}

- (void)udpSocket:(GCDAsyncUdpSocket *)sock didNotSendDataWithTag:(long)tag dueToError:(NSError *)error{
    SALog(@"没有发送数据");
}

```
注意事项：
（1）监听的端口(绑定端口)和发送的目的端口要一致。否则接收不到数据；
（2）如果要进行广播数据，必须只能使用[socket bindToPort:9999 error:&error] 方法来绑定端口，不能绑定IP地址并且启用广播，否则不能广播数据。在发送广播的时候可以绑定地址也可以设置为所有地址的广播地址；
（3）该实例没有被销毁前，只能被创建一次，因为端口正在被使用，不能被重复创建，可以用一个懒加载优化下。广播发送之后就会一直发送除非断开连接；
（4）注意 NSTimer 的销毁防止内存泄露。

##### 心跳包
###### 心跳包机制
  跳包之所以叫心跳包是因为：它像心跳一样每隔固定时间发一次，以此来告诉服务器，这个客户端还活着。事实上这是为了保持长连接，至于这个包的内容，是没有什么特别规定的，不过一般都是很小的包，或者只包含包头的一个空包。   
  
  在TCP的机制里面，本身是存在有心跳包的机制的，也就是TCP的选项：SO_KEEPALIVE。系统默认是设置的2小时的心跳频率。但是它检查不到机器断电、网线拔出、防火墙这些断线。而且逻辑层处理断线可能也不是那么好处理。一般，如果只是用于保活还是可以的。   心跳包一般来说都是在逻辑层发送空的echo包来实现的。下一个定时器，在一定时间间隔下发送一个空包给客户端，然后客户端反馈一个同样的空包回来，服务器如果在一定时间内收不到客户端发送过来的反馈包，那就只有认定说掉线了。   其实，要判定掉线，只需要send或者recv一下，如果结果为零，则为掉线。但是，在长连接下，有可能很长一段时间都没有数据往来。理论上说，这个连接是一直保持连接的，但是实际情况中，如果中间节点出现什么故障是难以知道的。更要命的是，有的节点（防火墙）会自动把一定时间之内没有数据交互的连接给断掉。在这个时候，就需要我们的心跳包了，用于维持长连接，保活。   在获知了断线之后，服务器逻辑可能需要做一些事情，比如断线后的数据清理呀，重新连接呀……当然，这个自然是要由逻辑层根据需求去做了。  
  
   总的来说，心跳包主要也就是用于长连接的保活和断线处理。一般的应用下，判定时间在30-40秒比较不错。如果实在要求高，那就在6-9秒。
   
###### 实现方法：

   由应用程序自己发送心跳包来检测连接是否正常，大致的方法是：服务器在一个 Timer事件中定时向客户端发送一个短小精悍的数据包，
   然后启动一个低级别的线程，在该线程中不断检测客户端的回应， 如果在一定时间内没有收到客户端的回应，即认为客户端已经掉线；同
   样，如果客户端在一定时间内没 有收到服务器的心跳包，则认为连接不可用。
   
   心跳检测步骤：
   ```
   1.客户端每隔一个时间间隔发生一个探测包给服务器
   2.客户端发包时启动一个超时定时器
   3.服务器端接收到检测包，应该回应一个包
   4.如果客户机收到服务器的应答包，则说明服务器正常，删除超时定时器
   5.如果客户端的超时定时器超时，依然没有收到应答包，则说明服务器挂了
   ```
   
##### GCDAsyncSocket
   ```
   - (instancetype)init
   {
   if (self = [super init]) {
   _asyncSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
   }
   return self;
   }
   
   - (void)dealloc
   {
   _asyncSocket.delegate = nil;
   _asyncSocket = nil;
   }
   //连接主机
   - (void)connectWithHost:(NSString *)hostName port:(int)port
   {
   
   }
   
   - (void)disconnect
   {
       [_asyncSocket disconnect];
   }
   
   - (BOOL)isConnected
   {
       return [_asyncSocket isConnected];
   }
   
   - (void)readDataWithTimeout:(NSTimeInterval)timeout tag:(long)tag
   {
   
   }
   
   - (void)writeData:(NSData *)data timeout:(NSTimeInterval)timeout tag:(long)tag
   {
   
   }
   
   #pragma mark -
   #pragma mark GCDAsyncSocketDelegate method
   - (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(NSError *)err
   {
   
   }
   //连接成功
   - (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(uint16_t)port
   {
   }
   
   - (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag
   {
       //一直请求读取否则会接收不到数据
       [sock readDataWithTimeout:-1 tag:tag];
   }
   
   - (void)socket:(GCDAsyncSocket *)sock didWriteDataWithTag:(long)tag
   {
       [sock readDataWithTimeout:-1 tag:tag];
   }
   ```
   
#### 数据粘包的处理
   在使用socket通信的时候是避免不了会遇到在接收数据的时候出现数据粘包的问题，因此在客户端处理数据粘包的问题是不可避免的；本项目中对于数据粘包的处理是和后台协商制定包尾结束符,对接收到的数据进行便利拼接成完成的数据包以供解析;
