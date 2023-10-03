# （五）零基础学懂FPGA中的串口通信（UART）
# 0 致读者

此篇为专栏 **《FPGA学习笔记》** 的第五篇，记录我的学习FPGA的一些开发过程和心得感悟，刚接触FPGA的朋友们可以先去此专栏置顶 [《FPGA零基础入门学习路线》](http://t.csdnimg.cn/T0Qw2)来做最基础的扫盲。

本篇内容基于笔者实际开发过程和正点原子资料撰写，将会详细讲解此FPGA实验的全流程，**诚挚**地欢迎各位读者在评论区或者私信我交流！

**UART** 的英文全称是 **Universal Asynchronous Receiver/Transmitter**，即**通用异步收发器**，串口是**串行接口**的简称，两者组合起来就是**通用异步串行通信接口**， 它包括了 RS232、 RS499、 RS423、 RS422 和 RS485 等接口标准规范和总线标准规范， 因此串口广泛应用于嵌入式、工业控制等领域。 本文我们**将使用 ZYNQ 开发板上的 UART 串口完成上位机与 ZYNQ PL 的通信**。


本文的工程文件**开源地址**如下（基于ZYNQ7020，大家 **clone** 到本地就可以直接跑仿真，如果要上板请根据自己的开发板更改约束即可）：

> [https://github.com/ChinaRyan666/FPGA-UART](https://github.com/ChinaRyan666/FPGA-UART)
# 1 实验任务

本文实验任务是**上位机**通过串口调试助手**发送数据**给 ZYNQ 7020 开发板， ZYNQ 7020 开发板 **PL 端**通过 USB_UART 串口**接收**数据并**将接收到的数据发送给上位机**，完成**串口数据环回**。**UART**通信波特率：**115200**，停止位：**1**，数据位：**8位**，无校验位。




# 2 UART 串口简介

**通信方式**在日常的应用中一般分为**串行通信（serial communication）**和**并行通信（parallel communication）**。首先我们了解下什么是**并行通信**， 并行通信是指多比特数据同时通过并行线进行传送， 一般以字或字节为单位并行进行传输。这种传输方式用的通信线多、成本高，故不宜进行远距离通信，因此并行通信一般用于**近距离**的通信，通常传输距离小于 30 米。

我们再来了解下**串行通信**的特点。 串行通信是指数据在一条数据线上，一比特接一比特地按顺序传送的方式，这一点与并行通信是不同的。这里我们以传输一个字节（ 8 位）数据为例，在**并行通信**中， 一个字节的数据是在 8 条并行传输线上同时由源地传送到目的地；而在**串行通信**中， 因为数据是在一条传输线上一位接一位地顺序传送的，所以一个字节的数据要分 8 次进行传送。 

如果我们以 T 为一个时间单位的话，那么并行通信发送一个字节的数据只需要 1T 的时间，而串行通信需要 8T 的时间，由此可以总结出**串行通信的的特点**：一是节省传输线，大大降低了使用成本，二是数据传送速度慢，这一点在大位宽的数据传输上尤为明显。综上可知，串行通信主要应用于长距离、低速率的通信场合。本次实验我们主要讲解下串行通信。

串行通信一般有 2 种通信方式： **同步串行通信（ synchronized serial communication）** 和**异步串行通信（asynchronous serial communication）**。 同步串行通信需要通信双方在**同一时钟**的控制下同步传输数据； 异步串行通信是指具有不规则数据段传送特性的串行数据传输。 在常见的通信总线协议中， **I2C， SPI 属于同步通信而 UART 属于异步通信**。

>同步通信的通信双方必须先建立同步，即双方的时钟要调整到同一个频率，收发双方不停地发送和接收连续的同步比特流。异步通信在发送字符时，发送端可以在任意时刻开始发送字符，所以，**在 UART 通信中，数据起始位和停止位是必不可少的。**

**UART** 是一种采用异步串行通信方式的通用异步收发传输器（universal asynchronous receiver-transmitter），它**在发送数据时将并行数据转换成串行数据来传输，在接收数据时将接收到的串行数据转换成并行数据。**

UART 串口通信需要**两根信号线**来实现， 一根用于**串口发送**，另外一根负责**串口接收**，如下图所示。 对于 PC 来说它的 TX 要和对于 FPGA 来说的 RX 连接， 同样 PC 的 RX 要和 FPGA 的 TX 连接，如果是两个 TX 或者两个 RX 连接那数据就不能正常被发送出去或者接收到，所以这里大家不要弄混。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2a2f4495c21442848eb7f5f33e2100a1.png)

**UART** 在发送或接收过程中的一帧数据由 4 部分组成，**起始位**、**数据位**、**奇偶校验位**和**停止位**，如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a807fe60970f41c2b8e6d6e57f1f0dbc.png)

>**起始位：** 当不传输数据时， UART 数据传输线通常保持高电压电平。若要开始数据传输，发送 UART 会将传输线从高电平拉到低电平并保持 1 个波特率周期。当接收 UART 检测到高到低电压跃迁时，便开始以波特率对应的频率读取数据帧中的位。

>**数据帧：** 数据帧包含所传输的实际数据。如果使用奇偶校验位，数据帧长度可以是 5 位到 8 位。如果不使用奇偶校验位，数据帧长度可以是 9 位。在大多数情况下，数据以最低有效位优先方式发送。

>**奇偶校验：** 奇偶性描述数字是偶数还是奇数。通过奇偶校验位，接收 UART 判断传输期间是否有数据发生改变。电磁辐射、不一致的波特率或长距离数据传输都可能改变数据位。接收 UART 读取数据帧后，将计数值为 1 的位，检查总数是偶数还是奇数。如果奇偶校验位为 0（偶数奇偶校验），则数据帧中的 1 或逻辑高位总计应为偶数。如果奇偶校验位为 1（奇数奇偶校验），则数据帧中的 1 或逻辑高位总计应为奇数。当奇偶校验位与数据匹配时， UART 认为传输未出错。但是，如果奇偶校验位为 0，而总和为奇数，或者奇偶校验位为 1，而总和为偶数，则 UART 认为数据帧中的位已改变。

>**停止位：** 为了表示数据包结束，发送 UART 将数据传输线从低电压驱动到高电压并保持 1 到 2 位时间。

**UART** 通信过程中的**数据格式**及**传输速率**是可设置的，为了正确的通信，**收发双方应约定并遵循同样的设置**。**数据位**可选择为 5、6、7、8 位，其中 8 位数据位是最常用的，在实际应用中一般都选择 8 位数据位；**校验位**可选择奇校验、偶校验或者无校验位；**停止位**可选择 1 位（默认），1.5 或 2 位。串口通信的速率用**波特率**表示，它表示每秒传输二进制数据的位数，单位是 bps （位/秒），常用的波特率有 9600、 19200、 38400、57600 以及 115200 等。

>那么什么是波特率呢？ **波特率：即每秒传输的位数(bit)。一般选波特率都会有 9600， 19200， 115200 等选项**。其实意思就是每秒传输这么多个比特位数(bit)。在信息传输通道中，携带数据信息的信号单元叫作码元（因为串口是 1bit 进行传输的，所以其码元就代表一个二进制数），每秒通过信号传输的码元数称为码元的传输速率，简称“波特率”，常用符号“Baud”表示，其单位为“波特每秒”（Bps）。串口常见的波特率有 4800、 9600、 115200 等，此处我们选用 **115200** 的波特率进行讲解。

通信信道每秒传输的信息量称为**位传输速率**，简称 **“比特率”** ，其单位为 “每秒比特数”（bps）。比特率可由波特率计算得出，公式为**比特率=波特率×单个调制状态对应的二进制位数**。

>如果使用的是 115200 的波特率，其串口的比特率为 115200Bps×1bit = 115200bps， 由计算得串口发送或者接收 1bit 数据的时间为一个波特，即 1/115200s。

在设置好数据格式及传输速率之后， **UART 负责完成数据的串并转换， 而信号的传输则由外部驱动电路实现**。 电信号的传输过程有着不同的电平标准和接口规范， 针对异步串行通信的接口标准有 **RS232**、 **RS422**、**RS485** 等， 它们定义了接口不同的电气特性，如 **RS-232** 是**单端输入输出**，而 **RS-422/485** 为**差分输入输出**等。

**RS-232** 标准的串口最常见的接口类型为 **DB9**， 样式如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/382d14d2579340a19b0f808ed37ea031.png)



工业控制领域中用到的工控机一般都配备多个串口， 很多老式台式机也都配有串口。但是笔记本电脑以及较新一点的台式机都没有串口，它们一般通过 **USB 转串口线**（如下图所示）来实现与外部设备的串口通信。

![在这里插入图片描述](https://img-blog.csdnimg.cn/78fdbd3b7c1848e5b80626d6438e4324.png)

**DB9** 接口定义以及各引脚功能说明如下图所示，我们一般只用到其中的 **2（RXD）、3（TXD）、5（GND）** 引脚，其他引脚在普通串口模式下一般不使用，如果大家想了解，可以自行搜索下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a265314a341f4a9dae1f823da83688f6.png)

**由于传统的 DB9 接口体积较大**，会占用开发板过多空间，现在的开发板上大多数采用的是 **USB TYPE_C** 接口，样式如图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4509a758ff4b43a58c65da8abd379e4b.png)


**另一端直接和电脑 USB 相连**。连接示意图如图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd469dc441794aeb80a10970e851acbd.png)







# 3 程序设计

## 3.1 总体模块设计

首先我们分析下本次设计中都需要哪些**功能模块**。由实验任务可知，我们需要通过串口来接收上位机发出的数据，所以我们需要一个串口接收模块（**uart_rx**） ，该模块用于将串口接收端口（**uart_rxd**）上的**串行数据解析成并行数据**， 并将解析完成的并行数据（**uart_rx_data**）作为模块的输出信号来供其它模块使用。

>需要注意的是，**数据并不是一直有效的**，所以为了告诉其它模块何时输出的 **uart_rx_data** 数据是有效的，我们还需要输出一个用于表示串口数据接收完成的信号，该信号可以说明当前的输出的 **uart_rx_data** 数据是完整且有效的， 这里我们将该信号命名位 **uart_rx_done**。

有了串口接收模块模块后，我们还需要一个能将数据发给上位机的串口发送模块（**uart_tx**），该模块用于将 **uart_rx** 模块解析完成的并行数据数据（**uart_rx_data**） 转成串行数据，并通过串口发送端口（**uart_txd**）发回上位机， 所以我们需要将 **uart_rx_data** 数据传递给 **uart_tx** 模块， 即 **uart_tx** 模块需要一个用于接收 **uart_rx_data** 数据的输入端口，这里我们将该端口命名为 **uart_tx_data**，且位宽与 **uart_rx_data** 相等。 

除此之外，因为 **uart_tx** 模块并不需要实时运行，所以我们需要输入一个使能信号（**uart_tx_en**） 来控制 **uart_tx** 模块在什么时候工作。又因为我们需要在 **uart_rx** 模块解析完上位机发来的数据后，就将该数据发送回上位机，因此我们可以将 **uart_rx** 模块输出的 **uart_rx_done** 信号作为 **uart_tx** 模块的使能信号。 

>值得一提的是，在一些复杂的工程项目， **uart_tx** 模块需要发送的数据可能是其它功能模块内部的某个数据，所以为了**避免在发送数据的过程中因内部数据发生改变而造成的误码和丢码现象**，我们通常会让 **uart_rx** 模块输出一个忙信号（**uart_tx_busy**），以此来告诉其它功能模块不要在 **uart_tx** 模块运行期间（即发送数据的过程中）更新需要通过串口发送的数据。

综上，我们可以绘制出如下所示的**系统框图：**

![在这里插入图片描述](https://img-blog.csdnimg.cn/19020f032a3248d3aee4bf87a2f51b88.png)


由系统总体框图可知， 本次实验包括三个模块， 分别是**顶层模块**、 **串口接收模块**和**串口发送模块。** 

**串口接收模块**从串口接收端口来接收**上位机**发送的串行数据，并在一帧数据接收结束后给出**通知信号**。**串口发送模块**将接收到的数据通过**串口发送端口**发送出去并同时给出一个**反馈信号**。

**顶层模块端口与功能描述**如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/4113944c30eb494f86d856e1204b7257.png)

**代码编写的思维导图**（非常重要！后面的代码如果阅读吃力可以回来看看思路）如图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/faf4696ce009465eb22b3ff2200df532.jpeg)







## 3.2 串口接收模块设计


首先介绍**串口接收模块**的设计，串口接收模块我们的输入信号主要有系统时钟信号、系统复位信号与串口接收端口。当我们将一帧的数据接收完成后，那么要告诉下级模块已经将一帧数据接收完成了，所以输出为接收完成标志和串口接收数据信号。**模块接口框图**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/6b37359a21614ccc8b45fd9d2d6cbb1a.png)



**串口接收模块端口与功能描述**如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/378e16b480c14d7784f3823ff5641d05.png)

### 3.2.1 绘制波形图

在绘制波形图之前， 我们**首先要确定串口通信的数据格式及波特率**。在这里我们选择串口比较常用的一种模式， 数据位为 8 位， 停止位为 1 位，无校验位，波特率为 115200bps。则传输一帧数据的时序图如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/b7d5e37ae87042d6809a2e8a8a88b11d.png)

如果使用的是 **115200** 的波特率，其串口的比特率为 **115200Bps×1bit = 115200bps**。 由计算得串口发送或者接收 **1bit** 数据的时间为一个波特，即 **1/115200s**，如果用 **50MHz（周期为 20ns）**的系统时钟来计数，需要计数的个数为 **cnt = (1s×10^9)ns / 115200bit)ns / 20ns ≈ 434** 个系统时钟周期，**即每位数据之间的间隔要在 50MHz的时钟频率下计数 434 次**，这样我们就需要一个**至少为 9 位**的波特率计数器来计数，这里设为 16 位是为了其他波特率的使用（为了模块的通用性）。

**串口是串行发送的，而我们的输出数据是并行输出的**，所以还需要一个 **4 位的接收数据计数器（rx_cnt）**。我们在系统时钟**计数器计数到一半时去采集数据**，这时候的数据采集是最正确的。波形图如下所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/fbfa778163dd45deaef30370a198c1c7.png)

由上图可知，我们**将进来的串口接收数据打拍，这个操作是为了消除亚稳态**，那么什么是亚稳态呢？

>亚稳态是由于**违背了触发器的建立和保持时间**而产生的。寄存器采样需要满足一定的建立时间（setup）和保持时间（holdup），而**异步电路没有办法保证建立时间（ setup）和保持时间（holdup），所以会出现亚稳态**。下图为两个不同时钟的寄存器的示意图，寄存器 B 采样可能采样到寄存器 A 输出的任意状态，包括 Q 的信号跳变沿。


![在这里插入图片描述](https://img-blog.csdnimg.cn/ca8d357f58f94f9ab851ca591ba94b63.png)

>但是寄存器的 D 端信号**需要满足建立时间 Tsu 和保持时间 Th 要求**，否则就会出现 Q 端采样到不确定的值的状态，就是俗称的**亚稳态**。

采样时采样到不同的电平状态（**因为信号跳变沿是一个信号上升的过程，时钟在信号跳变沿采样会采样到不同的电平**），那么会导致不同决断时间，决断时间指的是最终逻辑从不确定值到为 0 或者为 1 的持续时间，也就是无法预测该单元的输出电平，并且这种不确定的输出电平可以沿信号通道上的各个触发器级联式传播下去。亚稳态会导致这种严重的后果，那么解决亚稳态有哪些方法呢？解决亚稳态有以下几种方式：

>**单 bit 信号：** 直接**多级寄存器同步法**，一般采用 2-3 级寄存器进行同步处理，这个 2-3 级寄存器也称作同步器，在 ASIC 设计中，一般都有提供专用的同步器库，因为同步器要求多级寄存器位置靠的越近越好，靠的越近，亚稳态消失的概率就越大。 FPGA 设计中，直接使用 2-3 级寄存器进行同步处理即可。

>**多 bit 信号：** **异步 FIFO** 或者使用**多次握手同步方法**。在握手协议中，异步的 REQ/ACK 也需要使用单 bit 同步技术进行同步处理，异步 FIFO 也是如此。

由波形图可知，**当检测到串口接收数据的下降沿时**，表示接收过程的开始，即 **start_flag** 信号拉高，此时系统时钟计数器（**clk_cnt**） 开始计数。当系统时钟计数器记到最大值（433）时接收数据计数器（**rx_cnt**） 开始计数，当接收数据计数器大于等于 1 后，计数器每计数一次，将打拍后的数据 **uart_rxd_d2** 寄存在 **rxdata** 中。当接收数据计数器记到 9 时，把 **rxdata** 寄存的数据 **rxdata** 给到输出端口 **uart_data**，同时拉高接收完成标志信号 **uart_done**。

>这里为什么要记到 9 呢，因为我们本次的传输协议是 1bit 起始位， 8bit数据位， 1bit 停止位，总共 10bit，而 0~9 总共是计了 10 个数。






### 3.2.2 编写代码

从上面波形的绘制与讲解我们可以很容易看出串口接收模块中各个信号的逻辑关系，接下来我们就可以开始编写串口接收模块的代码了， **串口接收模块的详细代码**如下所示：


```
module uart_rx(
    input               clk         ,  //系统时钟
    input               rst_n       ,  //系统复位，低有效

    input               uart_rxd    ,  //UART接收端口
    output  reg         uart_rx_done,  //UART接收完成信号
    output  reg  [7:0]  uart_rx_data   //UART接收到的数据
    );

//parameter define
parameter CLK_FREQ = 50000000;               //系统时钟频率
parameter UART_BPS = 115200  ;               //串口波特率
localparam BAUD_CNT_MAX = CLK_FREQ/UART_BPS; //为得到指定波特率，对系统时钟计数BPS_CNT次

//reg define
reg          uart_rxd_d0;
reg          uart_rxd_d1;
reg          uart_rxd_d2;
reg          rx_flag    ;  //接收过程标志信号
reg  [3:0 ]  rx_cnt     ;  //接收数据计数器
reg  [15:0]  baud_cnt   ;  //波特率计数器
reg  [7:0 ]  rx_data_t  ;  //接收数据寄存器

//wire define
wire        start_en;

//*****************************************************
//**                    main code
//*****************************************************
//捕获接收端口下降沿(起始位)，得到一个时钟周期的脉冲信号
assign start_en = uart_rxd_d2 & (~uart_rxd_d1) & (~rx_flag);

//针对异步信号的同步处理
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) begin
        uart_rxd_d0 <= 1'b0;
        uart_rxd_d1 <= 1'b0;
        uart_rxd_d2 <= 1'b0;
    end
    else begin
        uart_rxd_d0 <= uart_rxd;
        uart_rxd_d1 <= uart_rxd_d0;
        uart_rxd_d2 <= uart_rxd_d1;
    end
end

//给接收标志赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        rx_flag <= 1'b0;
    else if(start_en)    //检测到起始位
        rx_flag <= 1'b1; //接收过程中，标志信号rx_flag拉高
    //在停止位一半的时候，即接收过程结束，标志信号rx_flag拉低
    else if((rx_cnt == 4'd9) && (baud_cnt == BAUD_CNT_MAX/2 - 1'b1))
        rx_flag <= 1'b0;
    else
        rx_flag <= rx_flag;
end        

//波特率的计数器赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        baud_cnt <= 16'd0;
    else if(rx_flag) begin     //处于接收过程时，波特率计数器（baud_cnt）进行循环计数
        if(baud_cnt < BAUD_CNT_MAX - 1'b1)
            baud_cnt <= baud_cnt + 16'b1;
        else 
            baud_cnt <= 16'd0; //计数达到一个波特率周期后清零
    end    
    else
        baud_cnt <= 16'd0;     //接收过程结束时计数器清零
end

//对接收数据计数器（rx_cnt）进行赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        rx_cnt <= 4'd0;
    else if(rx_flag) begin                  //处于接收过程时rx_cnt才进行计数
        if(baud_cnt == BAUD_CNT_MAX - 1'b1) //当波特率计数器计数到一个波特率周期时
            rx_cnt <= rx_cnt + 1'b1;        //接收数据计数器加1
        else
            rx_cnt <= rx_cnt;
    end
    else
        rx_cnt <= 4'd0;                     //接收过程结束时计数器清零
end        

//根据rx_cnt来寄存rxd端口的数据
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        rx_data_t <= 8'b0;
    else if(rx_flag) begin                           //系统处于接收过程时
        if(baud_cnt == BAUD_CNT_MAX/2 - 1'b1) begin  //判断baud_cnt是否计数到数据位的中间
           case(rx_cnt)
               4'd1 : rx_data_t[0] <= uart_rxd_d2;   //寄存数据的最低位
               4'd2 : rx_data_t[1] <= uart_rxd_d2;
               4'd3 : rx_data_t[2] <= uart_rxd_d2;
               4'd4 : rx_data_t[3] <= uart_rxd_d2;
               4'd5 : rx_data_t[4] <= uart_rxd_d2;
               4'd6 : rx_data_t[5] <= uart_rxd_d2;
               4'd7 : rx_data_t[6] <= uart_rxd_d2;
               4'd8 : rx_data_t[7] <= uart_rxd_d2;   //寄存数据的高低位
               default : ;
            endcase  
        end
        else
            rx_data_t <= rx_data_t;
    end
    else
        rx_data_t <= 8'b0;
end        

//给接收完成信号和接收到的数据赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) begin
        uart_rx_done <= 1'b0;
        uart_rx_data <= 8'b0;
    end
    //当接收数据计数器计数到停止位，且baud_cnt计数到停止位的中间时
    else if(rx_cnt == 4'd9 && baud_cnt == BAUD_CNT_MAX/2 - 1'b1) begin
        uart_rx_done <= 1'b1     ;  //拉高接收完成信号
        uart_rx_data <= rx_data_t;  //并对UART接收到的数据进行赋值
    end    
    else begin
        uart_rx_done <= 1'b0;
        uart_rx_data <= uart_rx_data;
    end
end

endmodule
```

### 3.2.3 代码讲解


串口接收模块程序中 34 至 45 行是一个经典的**边沿检测电路**，通过检测串口接收端 **uart_rxd** 的下降沿来捕获起始位。一旦检测到起始位，输出一个时钟周期的脉冲 **start_en**，并进入串口接收过程。串口接收状态用 **rx_flag** 来标志， **rx_flag** 为高标志着串口接收过程正在进行，此时启动系统时钟计数器 **clk_cnt** 与接收数据计数器 **rx_cnt**。仿真波形如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/bd25bcac88b344328f7fc034c29e8561.png)

由第 13 行的公式 **BAUD_CNT_MAX = CLK_FREQ/UART_BPS** 可知， **BAUD_CNT_MAX** 为当前波特率下，串口传输一位所需要的系统时钟周期个数。因此 **baud_cnt** 从零计数到 **BAUD_CNT_MAX - 1** 时，串口刚好完成一位数据的传输。由于接收数据计数器 **rx_cnt** 在每次 **baud_cnt** 计数到 **BAUD_CNT_MAX - 1** 时加 1，因此由 **rx_cnt** 的值可以判断串口当前传输的是第几位数据。仿真波形如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/fa6b9b7996384a158916767ad5b05257.png)

第 89 行至第 111 行就是根据 **baud_cnt** 的值将 **uart** 接收端口的数据寄存到接收数据寄存器对应的数据位，从而实现接收数据的串并转换。其中第 93 行选择 **baud_cnt** 计数至 **（BAUD_CNT_MAX/2）- 1、** 时寄存接收端口数据，是因为**计数到数据中间时的采样结果最稳定**。仿真波形如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2b06d318b4df4231927c3c65c24bc28f.png)


程序中需要额外注意的地方是**串口接收过程结束条件的判定**，由第 54 行可知，在计数到停止位中间时，标志位 **rx_flag** 就已经拉低。这样做是因为虽然此时一帧数据传输还没有完成（停止位只传送到一半），但是数据位已经寄存完毕。而在连续接收数据时，**提前半个波特率周期结束接收过程可以为检测下一帧数据的起始位留出充足的时间**。仿真波形如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/3186d99be6dc40d0893d44e3da5e028c.png)

**串口的整个接收过程的仿真图**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/87d6497a1e0747af950706162fe54568.png)

当检测到串口接收端 **uart_rxd** 的下降沿后，在整个接收过程中 **rx_flag** 保持为高电平，同时 **rx_cnt** 对串口数据进行计数。当 **rx_cnt** 计数到 9 时， 且 **baud_cnt** 计数到停止位的中间时， 说明串口数据接收完成， 此时拉高 **uart_rx_done** 信号，同时将接收到的数据赋值给 **uart_rx_data**。从图中可以看到，接收模块能够正确接收串口数据并完成串并转换。




## 3.3 串口发送模块设计


**串口发送模块**我们的输入信号主要有系统时钟信号、系统复位信号以及发送使能信号和待发送数据，输出信号主要有发送忙状态标志和串口发送端口。**模块接口框图**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/19f80900dcfb477181b5970cb589de70.png)

**串口发送模块端口与功能描述**如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c988e99319854ffe93f6ba89a5d4375c.png)


### 3.3.1 绘制波形图

串口发送模块与串口接收模块异曲同工， **串口接收模块是进行串转并， 串口发送模块是并转串**，因此串口发送模块也需要一个 16 位的系统时钟计数器（**baud_cnt**）和 4 位的发送数据计数器（**tx_cnt**）。

当串口发送模块接收到串口接收模块发送过来的高电平发送使能（**uart_tx_en**）时，拉高发送忙状态标志（**uart_tx_busy**）同时寄存待发送的数据（**tx_data_t**）。 在整个发送过程中发送忙状态标志保持高电平，**tx_cnt** 对串口数据进行计数，同时 **tx_data_t** 的各个数据位依次通过串口发送端 **uart_txd** 发送出去。当 **tx_cnt** 计数到 9 时，串口数据发送完成，开始发送停止位。在一个波特率周期的停止位发送完成后，串口发送过程结束， **uart_tx_busy** 信号拉低，表明串口发送模块进入空闲状态。

**波形图**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b5296427ea2c4ac898c1f8c8e666e16a.png)

>从波形图中大家是否发现了在停止位的时候我将发送忙状态标志（**uart_tx_busy**）提前给拉低了 **1/16** 个波特率周期（根据经验反复验证后得到的最佳参数），这是为了**确保发送模块发送数据的时间略小于接收模块接收数据的时间**，否则当连续传输大量数据时，发送数据的时间会不断累积，最终导致在做串口环回实验时丢失数据。串口接收模块传输过来的待发送数据为 **8’h55**，最后我们发送给上位机的数据也是 **8’h55**。

### 3.3.2 编写代码

从上面波形的绘制与讲解我们可以很容易看出串口发送模块中各个信号的逻辑关系，接下来我们就可以开始编写串口发送模块的代码了， **串口发送模块的详细代码**如下所示：


```
module uart_tx(
    input               clk         , //系统时钟
    input               rst_n       , //系统复位，低有效
    input               uart_tx_en  , //UART的发送使能
    input     [7:0]     uart_tx_data, //UART要发送的数据
    output  reg         uart_txd    , //UART发送端口
    output  reg         uart_tx_busy  //发送忙状态信号
    );

//parameter define
parameter CLK_FREQ = 50000000;               //系统时钟频率
parameter UART_BPS = 115200  ;               //串口波特率
localparam BAUD_CNT_MAX = CLK_FREQ/UART_BPS; //为得到指定波特率，对系统时钟计数BPS_CNT次

//reg define
reg  [7:0]  tx_data_t;  //发送数据寄存器
reg  [3:0]  tx_cnt   ;  //发送数据计数器
reg  [15:0] baud_cnt ;  //波特率计数器

//*****************************************************
//**                    main code
//*****************************************************

//当uart_tx_en为高时，寄存输入的并行数据，并拉高BUSY信号
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) begin
        tx_data_t <= 8'b0;
        uart_tx_busy <= 1'b0;
    end
    //发送使能时，寄存要发送的数据，并拉高BUSY信号
    else if(uart_tx_en) begin
        tx_data_t <= uart_tx_data;
        uart_tx_busy <= 1'b1;
    end
    //当计数到停止位结束时，停止发送过程
    else if(tx_cnt == 4'd9 && baud_cnt == BAUD_CNT_MAX - BAUD_CNT_MAX/16) begin
        tx_data_t <= 8'b0;     //清空发送数据寄存器
        uart_tx_busy <= 1'b0;  //并拉低BUSY信号
    end
    else begin
        tx_data_t <= tx_data_t;
        uart_tx_busy <= uart_tx_busy;
    end
end

//波特率的计数器赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        baud_cnt <= 16'd0;
    //当处于发送过程时，波特率计数器（baud_cnt）进行循环计数
    else if(uart_tx_busy) begin
        if(baud_cnt < BAUD_CNT_MAX - 1'b1)
            baud_cnt <= baud_cnt + 16'b1;
        else 
            baud_cnt <= 16'd0; //计数达到一个波特率周期后清零
    end    
    else
        baud_cnt <= 16'd0;     //发送过程结束时计数器清零
end

//tx_cnt进行赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        tx_cnt <= 4'd0;
    else if(uart_tx_busy) begin             //处于发送过程时tx_cnt才进行计数
        if(baud_cnt == BAUD_CNT_MAX - 1'b1) //当波特率计数器计数到一个波特率周期时
            tx_cnt <= tx_cnt + 1'b1;        //发送数据计数器加1
        else
            tx_cnt <= tx_cnt;
    end
    else
        tx_cnt <= 4'd0;                     //发送过程结束时计数器清零
end

//根据tx_cnt来给uart发送端口赋值
always @(posedge clk or negedge rst_n) begin
    if(!rst_n) 
        uart_txd <= 1'b1;
    else if(uart_tx_busy) begin
        case(tx_cnt) 
            4'd0 : uart_txd <= 1'b0        ; //起始位
            4'd1 : uart_txd <= tx_data_t[0]; //数据位最低位
            4'd2 : uart_txd <= tx_data_t[1];
            4'd3 : uart_txd <= tx_data_t[2];
            4'd4 : uart_txd <= tx_data_t[3];
            4'd5 : uart_txd <= tx_data_t[4];
            4'd6 : uart_txd <= tx_data_t[5];
            4'd7 : uart_txd <= tx_data_t[6];
            4'd8 : uart_txd <= tx_data_t[7]; //数据位最高位
            4'd9 : uart_txd <= 1'b1        ; //停止位
            default : uart_txd <= 1'b1;
        endcase
    end
    else
        uart_txd <= 1'b1;                    //空闲时发送端口为高电平
end

endmodule
```

### 3.3.3 代码讲解


为了确保环回实验的成功， 在程序的 36 行我们将 **uart_tx_busy** 提前 **1/16** 个停止位拉低。 尽管串口发送数据只是接收数据的反过程，理论上在传输的时间上是一致的， 但是考虑到我们模块里计算波特率会有较小的偏差，并且串口对端的通信设备（如电脑等）收发数据的波特率同样可能会出现较小的偏差，因此为了确保环回实验的成功，这里将发送模块的停止位略微提前结束。

需要说明的是，较小偏差的波特率在串口通信时是允许的，同样可以保证数据可靠稳定的传输。 **仿真波形**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/3f28b1b82440441e88518bdb8d0fb95a.png)



当检测到 **uart_tx_en** 使能时，将 **uart_tx_data** 端口上的待发送数据寄存到 **tx_data_t** 中，并进入串口发送过程，整个串口发送过程的**仿真图**如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a144ebfb8aed49ff891b589490b5e907.png)


## 3.4 顶层模块编写


实验工程的各子功能模块均已讲解完毕， 接下来我们就需要根据总体模块设计中的讲解来将各个子模块**例化**到顶层模块中。

### 3.4.1 编写代码

在顶层模块中完成了对其余各个子模块的例化。 本次实验我们需要**设置 2 个参数变量**，分别是系统时钟频率 **CLK_FREQ** 与串口波特率**UART_BPS**，大家使用时可以根据不同的系统时钟频率以及所需要的串口波特率设置这两个变量。 我使用的开发板上的系统时钟为 **50MHz**，所以这里将 **CLK_FREQ** 参数设为 **50000000**。

在前文我们说过本次实验的波特率设为 **115200**，所以这里的参数 **UART_BPS** 就设**115200**。我们可以尝试将串口波特率 **UART_BPS** 设置为其他值（如 **9600**），在模块例化时会将这个变量传递到串口接收与发送模块中，从而实现不同速率的串口通信。

顶层模块 **uart_loopback** 的代码如下：


```
module uart_loopback(
    input            sys_clk  ,   //外部50MHz时钟
    input            sys_rst_n,   //系外部复位信号，低有效
    
    //UART端口    
    input            uart_rxd ,   //UART接收端口
    output           uart_txd     //UART发送端口
    );

//parameter define
parameter CLK_FREQ = 50000000;    //定义系统时钟频率
parameter UART_BPS = 115200  ;    //定义串口波特率

//wire define
wire         uart_rx_done;    //UART接收完成信号
wire  [7:0]  uart_rx_data;    //UART接收数据

//*****************************************************
//**                    main code
//*****************************************************

//串口接收模块
uart_rx #(
    .CLK_FREQ  (CLK_FREQ),
    .UART_BPS  (UART_BPS)
    )    
    u_uart_rx(
    .clk           (sys_clk     ),
    .rst_n         (sys_rst_n   ),
    .uart_rxd      (uart_rxd    ),
    .uart_rx_done  (uart_rx_done),
    .uart_rx_data  (uart_rx_data)
    );

//串口发送模块
uart_tx #(
    .CLK_FREQ  (CLK_FREQ),
    .UART_BPS  (UART_BPS)
    )    
    u_uart_tx(
    .clk          (sys_clk     ),
    .rst_n        (sys_rst_n   ),
    .uart_tx_en   (uart_rx_done),
    .uart_tx_data (uart_rx_data),
    .uart_txd     (uart_txd    ),
    .uart_tx_busy (            )
    );
    
endmodule
```


# 4 仿真验证

## 4.1 编写 TestBench


我们接下来先对代码进行仿真，因为本章实验我们有**系统时钟**、 **系统复位**和**串口接收端口**这三个输入信号，所以仿真文件也只需要编写这三个信号的激励即可。在之前的博客中已经讲解过时钟和复位的激励怎么编写，这里不再讲解，不会的朋友可以翻看我之前写的博客。

当复位拉高后，再经过 **1000ns** 后，我们将 **uart_rxd** 拉低，表示已经开始发送起始位了，经过 **8680ns（434*20ns=8680，这个是 115200 波特率的一个波特率周期）**后，发送 **bit 0** 位，循坏 8 次后， **8bit** 数据已经全部发送完成了，后续就是开始发送停止位。 

**TestBench 代码**如下：


```
`timescale  1ns/1ns   //仿真的单位/仿真的精度

module tb_uart_loopback();

//parameter define
parameter  CLK_PERIOD = 20;//时钟周期为20ns

//reg define
reg            sys_clk  ;  //时钟信号
reg            sys_rst_n;  //复位信号
reg            uart_rxd ;  //UART接收端口

//wire define
wire           uart_txd ;  //UART发送端口

//*****************************************************
//**                    main code
//*****************************************************

//发送8'h55  8'b0101_0101
initial begin
    sys_clk <= 1'b0;
    sys_rst_n <= 1'b0;
    uart_rxd <= 1'b1;
    #200
    sys_rst_n <= 1'b1;  
    #1000
    uart_rxd <= 1'b0;   //起始位
    #8680
    uart_rxd <= 1'b1;   //D0
    #8680
    uart_rxd <= 1'b0;   //D1
    #8680
    uart_rxd <= 1'b1;   //D2
    #8680
    uart_rxd <= 1'b0;   //D3
    #8680
    uart_rxd <= 1'b1;   //D4
    #8680
    uart_rxd <= 1'b0;   //D5
    #8680
    uart_rxd <= 1'b1;   //D6
    #8680
    uart_rxd <= 1'b0;   //D7 
    #8680
    uart_rxd <= 1'b1;   //停止位
    #8680
    uart_rxd <= 1'b1;   //空闲状态   
end

//50Mhz的时钟，周期则为1/50Mhz=20ns,所以每10ns，电平取反一次
always #(CLK_PERIOD/2) sys_clk = ~sys_clk;

//例化顶层模块
uart_loopback  u_uart_loopback(
    .sys_clk      (sys_clk  ), 
    .sys_rst_n    (sys_rst_n),
    .uart_rxd     (uart_rxd ),
    .uart_txd     (uart_txd )
    );

endmodule
```


## 4.2 代码仿真


接下来打开 **Modelsim** 软件对代码进行仿真，**仿真的波形**如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c4b18fb1c9b34e88ae7978750d0f1951.png)


从上图中我们可以看出当 **uart_rxd** 接收到数据 **8’h55** 后，经过一段时间后 **uart_txd** 发送出数据 **8’h55**，与我们的实验任务是一致的。 至此本节设计已经完成，接下来就进入**下载验证**部分了。












# 5 下载验证

## 5.1 引脚约束

在仿真验证完成后，接下来对引脚进行分配，并上板验证。 本实验中的**管脚分配**（根据自己的开发板原理图来分配）如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/301a125a80844b849c20f7dff5f37721.png)

对应的 **XDC 约束语句**如下所示：

```
create_clock -period 20.000 -name sys_clk [get_ports sys_clk]
set_property -dict {PACKAGE_PIN U18 IOSTANDARD LVCMOS33} [get_ports sys_clk]
set_property -dict {PACKAGE_PIN N16 IOSTANDARD LVCMOS33} [get_ports sys_rst_n]
set_property -dict {PACKAGE_PIN T19 IOSTANDARD LVCMOS33} [get_ports uart_rxd]
set_property -dict {PACKAGE_PIN J15 IOSTANDARD LVCMOS33} [get_ports uart_txd]
```

**Vivado** 软件中 **IO Planning** 界面如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/48cc2c57fe1d4b509fe98b3fa03c0f62.png)

## 5.2 上板验证

编译工程并生成比特流文件后， 我们需要准备一根的 **USB Type-C** 线，将 **USB** 接口一端插入电脑上的 **USB** 口， 另一端与开发板上的 **USB_UART** 接口相连接，接着分别连接 **JTAG** 接口和电源线，并打开电源开关。

注意上位机第一次使用 **USB 转串口线**与 **FPGA** 开发板连接时，需要安装 **USB 串口驱动**，我使用的开发板需安装 **CH340** 驱动（根据自己的开发板原理图为准，驱动资源网上非常多）。

开发板电源打开后，将本次实验的 **bit 文件**下载到开发板中，接下来打开串口助手。 串口助手是上位机中用于辅助串口调试的小工具，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7a61731752c4b7da9fff37c91ded1ab.png)


在串口助手中选择与开发板相连接的 **CH340** 虚拟串口，具体的端口号（我这里是 **COM5**）需要根据实际情况选择， 可以在计算机**设备管理器**中查看，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/cb7cdce899de4595b105305a25f0bb3b.png)




在串口助手中**设置波特率为 115200，数据位为 8，停止位为 1，无校验位**， 最后确认打开串口。

串口打开后，在发送文本框中输入数据 **“55”** 并点击发送，可以看到串口助手中接收到数据 **“55”** ，如下图所示。**串口助手接收到的数据与发送的数据一致**，说明程序所实现的串口数据环回功能**验证成功！**

![在这里插入图片描述](https://img-blog.csdnimg.cn/7e04d71d9a84437da83f206028ba61fe.png)










# 6 总结

到这里，本博客关于 **FPGA UART** 的讲解就完毕，通过实验，相信读者对于串口传输的基本知识和概念已经了解，大家需要重点理解以下两个知识点：

1. 了解**亚稳态**产生的原因以及怎么消除亚稳态；
2. 掌握**串并转换**，因为串并转换是接口中很常用的一种方法。

希望对您有所帮助，有兴趣的朋友可以进一步联系我交流。



微博：沂舟Ryan ([@沂舟Ryan 的个人主页 - 微博 ](https://weibo.com/u/7619968945))

GitHub：[ChinaRyan666](https://github.com/ChinaRyan666)

微信公众号：沂舟无限进步

如果对您有帮助的话请点赞支持下吧！



>**认识到有差距但不要低下头去，是你走出迷茫的第一步。**

