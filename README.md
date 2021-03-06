背景
--

阿里搬砖头比赛说好是Client端线程级的同步阻塞请求，结果一帮人用了协程来完成这件事。其实吧，我想说就算用协程来完成，其实本质也和异步差不多（就网络通讯层），不过却激发了我的好奇心，因为比赛的结果是1G极限，只用了3秒！

3秒…如果我们将题目往对我有利的思考方向改变下，不再是Client端线程级的同步阻塞，只要求Server端请求应答同步即可。即：Server端在没收到一个请求之前，不能提前将下一个应答发出！如果我将题目修改为这样，我也不见得非常有信心能将时间控制在3秒以内！

所以这里立了个小项目来验证不同的实现方式下，对网络传输、CPU使用的开销。项目起名为镭射，取汇集小请求将能量集中爆发之意。

打算挑战下极限搬运的。

思路参考
----

穿越nat传输完整的文件，而且能自动重传丢失报文，采用UDP协议，在跨大洲之间进行完整的大文件（超过2GB）传输。 
主要解决思路和技术点如下： 
对每个报文进行文件任务标志。可以同时传输多个文件。 
发送文件内容之前，先发送文件名。然后返回文件名收到后的确认，然后再开始传输。 
对发送报文进行计数，每隔1000个报文（某个时间间隔），接收端发送一个计数报文，报告丢包率。采取降低发送速度等措施。 
长期丢包率在99%以上，尝试提高发送速度一倍。如果丢包率超过10%就降速。 
文件传输中断，有中断信息。接收端长期没有接受到文件报文，则终止任务。 
发送完毕后，接收端发送完毕报文，结束发送。 
10000个报文组成一个块的概念，接收端每收到10000个（某个数字）报文，发送接受块成功的消息。整个块不会重复发送。如果块出现某个报文发送失败，则重新发送报文。 
每个UDP报文，大小是1464字节，其中64个字节用来标示任务。 
采用java编写，在web端使用，在tomcat启动时，也就是servlet的contextlistener启动的时候，启动守护进程，监听udp的80端口。是否与servlet进行协同。为了支持子网，客户端只负责发送和上传数据，不能下载数据。所以计数和控制，可能需要tcp的参与。

首先是报文，为了降低开发工作量。 
先使用udp报文的48个字节。剩下24个还没想好。 
首先将48个字节分为6个long类型。 
第一个long，标识版本和报文类型。 
比如1表示是数据传输 
2表示是重传通知等等。 
第二个long，是uuid的high位 
第三个long，是uuid的low位。 
一起标识一个文件。 
第四个long是包序号。 
第五个long是包内报文序号 
第六个long是下面的报文长度。 
第七个long是在被传输文件里面的位移量 
第八个long是在传输文件里面的写入长度

开始为了测试方便，将udp报文的数据长度设为1024，总长度是1024+64=1088 
将缓冲窗口设为512KB，即在内存开辟一个512KB的缓冲器，当收满512个UDP报文时，写内存中的数据写入目标文件。如果512个报文在单位时间内，没有收满，说明有丢包，则选择丢失的报文重传。丢包判断待定。在UDT协议里面是采用等待4倍的RTT时间判断的，在4倍的RTT时间后未到达的报文都被判断为丢失。 
目前完成了udp的服务启动。数据传输窗口的相关开发。其实只要保证窗口内的512KB数据传输成功即可。每512KB作为一个window，然后返回确认信息。客户端确认接到512KB完成的确认信息后，再去发送下一个512KB。确认信息里面可以加入crc校验或者MD5校验。客户端判断一下是否正确，决定是否重传。如果客户端决定，继续发送，则直接发送报文即可。服务器端，接到新的报文，如果是新报文ID，则将旧512KB数据写入实体文件，然后更新window，将收到的第一个报文写入window的相应位置。如果客户端没有收到服务器端的window结束的确认报文，则等待，而不发送新的报文。服务器端等待4个RTT时间，再发送确认报文，最多重试5次。如果5次客户端都没收到确认报文，不发送新数据，服务器没有接受新数据，则服务器端认为网络中断。停止任务。将任务写入临时文件。等待客户端下一次重传唤醒。websocket已经实现了udp，估计这个程序的上传客户端可以直接拿javascript来写了。比较理想了。之前的思路有点问题，关于客户端停止发送这一点，会导致传输效率变低，应该是客户端继续发送，直到5个RTT时间后，没有收到上个window结束信号之后，再停止发送。 
UDP之所以比TCP效率高，本质原因是TCP的一个bug，TCP将RTT时间和带宽联系在一起，片面认为RTT时间越长，带宽越低，而真实情况是，RTT和带宽没有直接关系。
