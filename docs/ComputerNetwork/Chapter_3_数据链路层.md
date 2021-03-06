## 使用点对点信道的数据链路层

### 基本概念

+ 链路

链路（link）是指从一个结点到**相邻**结点的<u>一段物理线路</u>（有线或无线），中间没有任何的交换结点。

+ 数据链路

在链路的基础上，增设实现各类通信协议的软硬件设施，即构成数据链路（data link）。

+ 帧

帧（frame）是点对点信道数据链路层所使用的一种协议数据单元。

### 主要任务

数据链路层主要负责三个任务：

#### 封装成帧

将网络层交下来的IP数据报添加首部和尾部封装成帧，如下图所示：

![](pics/Screenshot%202020-11-26%20145037.png)

MTU指最大传输单元（Maximum Transfer Unit），用于规定数据部分的长度上限。

首部和尾部的一个重要作用就是**帧定界**，通常使用特殊的帧定界符来区分每个帧，如首部设置SOH（Start Of Header）用于指示帧的开始，尾部设置EOT（End Of Transmission）指示帧的结束。帧定界的作用在传输发生差错时更为明显，因为不具备首部或尾部的不完整帧必然首先被丢弃。

#### 透明传输

将封装好的帧发送到目的主机的数据链路层，同时**无论帧中使用什么样的比特组合的数据，都能够按照原样没有差错地通过数据链路层**，这就是“透明”的含义所在。

为了防止一些与控制字符具有相同比特组合的数据不被接收端错误解析为控制字符，需要在它们之前加上转义字符，之后再由接收端移除转义字符后解析数据，从而实现透明传输。

#### 差错校验

检测收到的帧是否有差错，有则丢弃，无则提取出IP数据报上交至网络层。

**比特差错**

顾名思义，比特差错是指数据在传输过程中因为噪声等缘故，导致原来比特位的1和0发生变化。在一段时间内，传输错误的比特占所传输的比特总数的比率称为**误码率**，它与信噪比有着密切的关系。目前在数据链路层广泛使用的差错校验技术为CRC（Cycle Redundancy Check，循环冗余检验）。

CRC的原理如下：

在发送端将数据以 k 个比特为一组进行划分，每一组 M 后面添加 n 位冗余码（被称作帧检验序列FCS，Frame Check Sequence），然后构成一个帧发送出去。n 位冗余码的计算方式为：用二进制的**模2运算**进行 2<sup>n</sup> × M 的运算（相当于在 M 后面添加 n 个0），之后用这个 k + n 位的数**除以**收发双方<font color=red>事先商定</font>的 n + 1 位除数 P，得出一个 n 位余数 R，这个余数就是最后要添加的冗余码。

假设要发送的数据组 M 为`101001`，商定的除数 P 为`1101`，那么FCS的计算过程如下图所示：

![](pics/Screenshot%202020-11-26%20155947.png)

如果传输的帧没有差错，将 k + n 位的数据组（如上面例子中的101001001）除以除数 P 后得到的余数必定为0。

**传输差错**

CRC只能对比特差错进行校验，对于帧的<u>丢失、重复以及失序</u>问题，就需要在CRC的基础上增加**帧编号**、**确认**和**重传机制**。目前的做法是：

+ 对于通信质量良好的线路，数据链路层可以只做比特差错校验，发生传输差错时交由上层协议完成改正差错的任务；

+ 对于通信质量较差的线路，由数据链路层使用确认和重传机制，向上层提供可靠传输的服务。

### 点对点协议PPP

#### 特点

PPP协议是**用户计算机连接ISP**时所用到数据链路层协议。除了实现数据链路层的三大基本任务之外，还需要满足以下需求：

>简单

接收到帧之后做CRC校验，通过则上交，未通过则丢弃。简单的协议可以提高不同厂商在不同实现上的**互操作性**。

>支持多种网络层协议

PPP协议必须能够**在同一条物理线路上同时支持多种网络层协议**的运行。

>支持多种类型链路

PPP协议必须能在包括串行、并行、同步、异步、低速、高速、动态、非动态等类型的链路上运行。

>检测连接状态

PPP协议必须提供一种机制，可以及时地自动检测出链路是否处于正常工作状态。

>网络层地址协商

PPP协议必须提供一种机制，可以使通信的两个网络层实体能够通过协商来获知或配置彼此的网络层地址。

>数据压缩协商

PPP协议必须提供一种方法来协商数据的压缩算法，但不要求将数据压缩算法进行标准化。

此外，PPP协议还具有不支持多点线路、支持全双工链路的特点。它的组成可分为三个部分：

（1）一个将IP数据报封装到串行链路的方法；

（2）一个用来建立、配置和测试数据链路连接的链路控制协议LCP（Link Control Protocol）；

（3）一套网络控制协议NCP（Network Control Protocol）。

#### 帧结构

PPP协议的帧结构如下图所示：

![](pics/Screenshot%202020-11-27%20130146.png)

+ F表示Flag，十六进制数，是用于定界的标志字段。
+ A表示Address，十六进制数，是一个地址字段。
+ C表示Control，十六进制数，是一个控制字段。
+ 协议字段用于标识PPP帧的信息部分属于哪个协议。

为了确保透明传输，需要对信息部分的某些特定比特组合进行处理，一种是采用**字节填充**的方式，通常在异步传输的时候使用：

（1）信息字段中每存在一个0x7E字节，需要将其转换为2字节序列（0x7D，0x5E）;

（2）信息字段中每存在一个0x7D字节，需要将其转换为2字节序列（0x7D, 0x5E）;

（3）信息字段中存在ASCII码控制字符时，需要在其前面加上0x7D，并改变该控制字符的编码。

还有一种是**零比特填充**，通常在同步传输的时候使用：发送端在发送前对信息字段进行扫描，只要发现有5个连续的1，就在后面补上0，接收端则是反向操作，从而避免帧边界的误判。

#### 工作状态

下图是PPP链路的工作状态转换图：

![](pics/Screenshot%202020-11-27%20133746.png)

注意：
+ PPP链路的起始和终止状态一直是“链路静止”
+ 在LCP配置协商阶段，一方要发送LCP配置请求帧，另一方则回应配置确认帧/否认帧/拒绝帧。<u>确认帧表示所有选项都接受，否认帧表示理解所有选项但不接受，拒绝帧表示部分选项无法理解或接受需要协商</u>
+ 鉴别所用到的协议有口令鉴别协议PAP（Password Authentication Protocol）和口令握手鉴别协议CHAP（Challenge-Handshake Authentication Protocol）等
+ NCP根据网络层的不同协议互相交换网络层特定的网络控制分组
+ 在链路打开阶段，两个PPP端点还可以发送回送请求/响应LCP分组检查链路状态

从上面的流程可以看出，PPP协议已不是纯粹的数据链路层协议，它还包含了物理层和网络层的内容。

## 使用广播信道的数据链路层

### 局域网和以太网

### 使用集线器的星型拓扑

### 信道利用率

### MAC层

### 扩展以太网

### 高速以太网