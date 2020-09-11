# System Generator使用指南

## 1. 概括

System Generator是Xilinx公司进行数字信号处理开发的一种设计工具，它通过将Xilinx开发的一些模块嵌入到Simulink的库中，可以在Simulink中进行定点仿真，可以设置定点信号的类型，这样就可以比较定点仿真与浮点仿真的区别。并且可以生成HDL文件，或者网表，可以在ISE中进行调用。或者直接生成比特流下载文件。能够加快DSP系统的开发进度。（摘自百度百科）

简单说，Sysgen工具可以对设计的fpga部分进行simulink仿真，加快了关于信号处理的fpga模块的开发进度。

## 2. 运行环境配置

System Generator软件不需要单独安装，在安装Vivado时会有相关的选项，注意勾选就可以了。但是System Generator软件需要同MATLAB一同使用。官方文档UG973有推荐System Generator和Matlab搭配的版本。查询地址：https://www.xilinx.com/support/documentation-navigation/see-all-versions.html?xlnxproducttypes=Design%20Tools&xlnxdocumentid=UG973。查找如下应用进行配置。

<img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-11_11-07-3.png" style="zoom:60%;" />

自用是vivado2019.1搭配Matlab2019a。



<img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-11_11-07-38.png" style="zoom:67%;" />

因为不是推荐配置，所以apply的时候会有warming，目前自用未发现大问题。配置成功后进入sysgen需要从sysgen软件入口进入，不要从matlab入口进入，否则无法向Simulink中添加Block。

## 3. 简单入门

> 设计一个简单的数字滤波器

### 3.1 新建工程

运行sysgen后打开simulink，新建一个空白model，保存名为temp。

<img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-11_11-07-4.png" style="zoom:50%;" />

### 3.2 添加模块

Simulink中的仿真模型为连续时间系统，数据格式多种多样；而FPGA中为离散时间系统，数据必须用一定的位数进行量化。两者之间必须要进行从连续到离散的转换、数据格式的转换，否则无法进行正确的FPGA设计。Xilinx Blockset中提供了相应的解决方案。



添加一个Gateway In和一个Gateway Out模块到model中，再添加一个Digital FIR Filter模块。按照加法器输出->Gateway In->Digital FIR Filter->Gateway Out的顺序依次连接。