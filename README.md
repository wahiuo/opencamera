# opencamera

<img src="https://user-images.githubusercontent.com/31297314/139537513-4340dd17-7fb7-44e5-8fd5-cd53914eb6c8.jpg">

**opencamera:** 一个关于相机的技术栈教学。

包含fpga设计，linux驱动和应用程序。灵感来自于[openwifi项目](https://github.com/open-sdr/openwifi)，它大而全面非常好，但是读起来很容易只见树木不见森林。opencamera是一个非常小的但是有广度也有深度的项目，侧重于教学。

**支持的平台：** zynq7000 + sensor（eg.ov5640）  
**状态：** 1.1Done  
**vivadolicense:** webpack不需要

<img src="https://user-images.githubusercontent.com/31297314/139534114-a2978922-db15-4a46-b55b-d1a971c51129.jpg">

**（1）MipiDphy部分。** 接收“时钟hs-lp”和“数据hs-lp”，发送分频后的字节时钟和解串后的字节数据。

**（2）MipiCsi2部分。** 先接收dphy发送的data，sync，valid，active信号放到LM层交替排列，再送到LLP层进行长短包判断。支持raw10和rgb565格式。

**（3）Line2Frame部分。** 统计帧数并将数据转发给后续模块。

**（4）BayerInterpolate部分。** 将bayer数据插值输出rgb。支持rgb565和argb8888格式。

**（5）arm部分。** 处理帧缓冲队列，以及使用iic指令配置相机。

**（6）datamover部分。** 将数据从pl送到ps的ddr里（临时缓存），或从ps的ddr里送到pl里（vga显示）。

**（7）两个AxiLiteSlave2datamover部分。** 将axil的多个32比特命令字，组合成72比特命令字发送给datamover，主要用来交互图像帧请求。

**（8）VgaDisplay部分。** 将图像进行VGA显示。

# （1-4）部分

<img src= "https://user-images.githubusercontent.com/31297314/139534130-35c57940-54a6-471d-b187-badf069f6dd7.jpg">

# 一，MipiDphy部分

（1）z7系列需要外挂电阻网络或mc20901芯片来拆分LVDS（hs）和LVCMOS（lp）电平，从而得到上图中的MipiDphy输入。

（2）Scnn模块得到分频时钟。使用BUFIO来得到串行时钟336MHz，使用BUFR四分频来得到字节时钟84MHz。

（3）Sfen模块得到解串字节数据。使用ISERDESE2原语获得并行数据，主要是同步序列的匹配。

# 二，MipiCsi2部分

（1）LM层实现同步fifo，用来将多条lane的数据对齐。

（2）LLP层实现异步fifo，用来将对齐后的数据包解析成具体的格式，目前支持raw10和rgb565。

# 三，Line2Frame部分

（1）接收ps发送的帧数索引，并与本模块内部的帧数索引比较。

（2）如果不同说明需要获取新帧请求，否则不会接收mipi传输过来的数据。

# 四，BayerInterpolate部分

（1）运行在100MHz时钟下使用malvar插值算法输出。参考论文“high_quality_linear_interpolation(malvar2004).pdf”。

（2）输入40bit@100MHz是四个raw10，输出64bit@100MHz是四个rgb565，或128bit@100MHz是四个argb8888。

（3）使用五个行缓冲构成计算窗口。可以支持480x640或1080x1920等分辨率的计算。

# （5-8）部分

<img src= "https://user-images.githubusercontent.com/31297314/139534142-22822413-ec23-4fa7-9aa0-706a966eecad.jpg">

# 五，arm部分

（1）通过EMIO扩展gpio和iic来配置相机。

（2）GP0接口用来连接外设的axil型接口。HP0和HP1接口用来连接datamover的数据接口，构成上下传输通路。

（3）linux平台下的v4l2驱动实现中，主要加入了mipi驱动以及dma驱动等。v4l2缓冲管理的核心可以精简为四个队列，为了帮助理解，此项目额外在裸机版驱动中模拟了这部分内容。

<img src= "https://user-images.githubusercontent.com/31297314/139534493-df4c4386-990d-4602-9aea-e647bb117890.jpg" width="640">

# 六，datamover部分

（1）mm位宽是ps端存储映射，注意如果是64比特那ps软件里必须八字节对齐，否则传输错误。（编程上预先分配一个大空间然后取模偏移计算，同理如果是128比特那ps软件里是十六字节对齐）

（2）stream位宽，s2mm是自动64或128比特，为了四个rgb565或argb8888。mm2s是16比特，为了rgb565一个时钟一个像素。

# 七，两个AxiLiteSlave2datamover

（1）处理ps端发送的帧传输请求，与datamover交互72比特命令字和8比特状态字。

（2）当tvalid和tready同时为高时，交互命令计数自动加一，相当于帧索引了。

# 八，VgaDisplay

（1）使用axis接收数据，tlast信号用来帧同步复位。

（2）标准vga时序，支持480x640@60fps或1080x1920@60fps，数据格式支持rgb565或argb8888。

<img src= "https://user-images.githubusercontent.com/31297314/139534156-21b86fe4-c690-45b4-ad4a-1dac6997d641.jpg" width="640">
<img src= "https://user-images.githubusercontent.com/31297314/139534163-696e7a8c-bf97-45e0-bdc2-b56f877b745e.jpg" width="640">
