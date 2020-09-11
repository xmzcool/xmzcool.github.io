---
layout:     post
title:      System Generator使用指南
subtitle:   Using Guide
date:       2020-09-11
author:     MZ
header-img: img/post-bg-SG.jpg
catalog: true
tags:
    - Using Guide

---

# System Generator使用指南

> 学习前人，启发后人。

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

<img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-11_20-07-15.png" style="zoom:67%;" />

添加一个Gateway In和一个Gateway Out模块到model中，再添加一个Digital FIR Filter模块。按照加法器输出->Gateway In->Digital FIR Filter->Gateway Out的顺序依次连接。那么首先在simulink工程中添加信号输入。生成1MHz和9MHz正弦波，两路信号的叠加：

<img src="https://s1.ax1x.com/2020/09/11/wNokQg.png" alt="wNokQg.png" style="zoom:50%;" />

在设计中加入Gateway In模块，这个模块是Simulink到FPGA之间的数据转换。将Sample period设置为“1/20e6”（20MHz采样率），完成**连续时间到离散时间的转换**；设置OutType完成**数据格式的转换**。这里保持为默认的二进制带符号数补码、定点数的设置。Quantization中可以设置量化方式为Truncate（截断）或者round（四舍五入）。

<img src="https://s1.ax1x.com/2020/09/11/wNo56g.png" alt="wNo56g.png" style="zoom:60%;" />

添加Digital FIR Filter模块，FIR滤波器的系数可以使用fir1等MATLAB函数设计，也可以使用FDATOOL工具设计。选中“Use FDA Tool as coefficient source”，点击“FDA Tool”按钮，会弹出FDATOOL窗口，设置采样率为20Mhz，通带截止频率1.5MHz，阻带截止频率8.5Mhz，通带衰减0.01dB，阻带衰减100dB，点击“Design”设计后退出。

<img src="https://s1.ax1x.com/2020/09/11/wNTygU.png" alt="wNTygU.png" style="zoom:50%;" />

添加System Generator模块，这个block是配置与FPGA相关的系统参数。**这个block的配置会应用到Gateway In和Gateway Out之间的所有模块中**。双击打开，切换到Clock标签，FPGA clock period设置为50ns，Simulink system period设置为1/20e6（都是20Mhz）。Perfor analysis设置为Post Sythesis，Analyzer type设置为Resource，在系统综合后会进行资源使用情况的分析。

<img src="https://s1.ax1x.com/2020/09/11/wN7gRf.png" alt="wN7gRf.png" style="zoom:60%;" />

### 3.3 运行仿真

然后添加Gateway out进行系统仿真，整个系统框架如下和仿真结果如下。可以看到经过滤波后，9Mhz的频率分量被滤除，设计符合预期。

<img src="https://s1.ax1x.com/2020/09/11/wNqPKO.png" alt="wNqPKO.png" style="zoom:50%;" />

仿真验证功能正确后，需要将设计导出到FPGA中，这个步骤仍然要借助System Generator这个block。双击打开，切换到Compilation标签下：这里可以设置使用的开发板（Board，只能选择Xilinx官方开发板）、FPGA芯片（Part），也可以设置导出设计的硬件描述语言（Verilog或VHDL）。

<img src="https://s1.ax1x.com/2020/09/11/wNqgRx.png" alt="wNqgRx.png" style="zoom:50%;" />

点击“Generate”，System Generator会将Gateway In和Gateway Out之间的模块导出到FPGA中。运行结束后，根据前面的设置，弹出了资源分析报告如下。

<img src="https://s1.ax1x.com/2020/09/11/wNqzwQ.png" alt="wNqzwQ.png" style="zoom:67%;" />

### 3.4 Vivado调用

在slx同文件夹下，生成netlist文件夹。其中sysgen子文件夹包含了导出的Verilog或VHDL设计文件；ip子文件夹是设计导出的IP核形式；ip_catalog子文件夹包含一个调用该IP核的Vivado的示例工程。

<img src="https://s1.ax1x.com/2020/09/11/wNLv1x.png" alt="wNLv1x.png" style="zoom:67%;" />

用Vivado打开ip_catalog下的工程，顶层模块代码如下：

```verilog
//Copyright 1986-2019 Xilinx, Inc. All Rights Reserved.
//--------------------------------------------------------------------------------
//Tool Version: Vivado v.2019.1 (win64) Build 2552052 Fri May 24 14:49:42 MDT 2019
//Date        : Fri Sep 11 20:56:02 2020
//Host        : XMZ running 64-bit major release  (build 9200)
//Command     : generate_target temp_bd_wrapper.bd
//Design      : temp_bd_wrapper
//Purpose     : IP block netlist
//--------------------------------------------------------------------------------
`timescale 1 ps / 1 ps

module temp_bd_wrapper
   (clk,
    gateway_in,
    gateway_out);
  input clk;
  input [15:0]gateway_in;
  output [35:0]gateway_out;

  wire clk;
  wire [15:0]gateway_in;
  wire [35:0]gateway_out;

  temp_bd temp_bd_i
       (.clk(clk),
        .gateway_in(gateway_in),
        .gateway_out(gateway_out));
endmodule

```

可以看到，temp_bd是调用System Generator导出的IP核的子模块。16Bits输入数据经过滤波后得到36Bits的输出结果。运行RTL ANALYSIS，打开RTL视图，找到底层：

![wNxgWd.png](https://s1.ax1x.com/2020/09/11/wNxgWd.png)

可以看到其本质上仍然是调用了FIR Compiler IP核来实现数字滤波，只不过我们是在Simulink中完成的设计。在其它工程中可以像示例工程一样调用这个System Generator导出的IP核，来完成特定的DSP系统功能。

理论上经过Simulink中的仿真，已经可以确定设计的正确性。若我们在System Generator block的Compilation标签下选中“Create testbench”。在点击Generate导出设计时，软件会根据选择的硬件描述语言生成对应的testbench（在netlist/sysgen文件夹下）。在生成的Vivado示例工程中会自动添加这个testbench文件。在这个testbench中包含4个子模块：时钟生成模块xlclk、测试数据输入模块xltbsource、模块数据输出模块xltbsink和IP核设计文件temp_bd_wrapper。最后在顶层模块中调用4个子模块，组成一个完整的测试平台。直接运行仿真，Vivado中的仿真结果如下图所示：

![wU9Dcd.png](https://s1.ax1x.com/2020/09/11/wU9Dcd.png)

仿真结果与上一部分完全一致。这是因为System Generator工具在生成testbench文件时将simulink环境中接入到Gateway In block的数据存储到dat文件中，在testbench中调用。而我们自己编写testbench时需要设计M文件产生信号，再用HDL语言设计仿真过程。sysgen节省了一部分验证工作量。

### 3.5 资源分析与时序分析

System Generator集成了时序分析和资源分析功能，以确保在simulink中设计的DSP系统导出到FPGA环境中能够正确运行。其本质上仍然是在后台调用Vivado进行分析，System Generator只是读取了分析结果并显示出来。

设计完成并且Simulink运行完毕后，打开System Generator这个block，切换到Clock标签下：

<img src="https://s1.ax1x.com/2020/09/11/wN7gRf.png" alt="wN7gRf.png" style="zoom:60%;" />

Perform analysis中可以设置

- None：不进行分析；

- Post Synthesis：综合后进行分析；

- Post Implementation：实现后进行分析。

Analyzer type中可以设置

- “Resource”（进行资源分析）

- “Timing”（进行时序分析）

注意在Perform analysis设置好，并且导出设计到FPGA后，可以切换Analyzer type，并且点击右边的“Launch”查看两种报告。**但修改设计和修改Perform analysis参数后必须重新运行才能生成最新的报告**。

综合后和实现后的资源分析和时序分析分别如下。时序报告中可以看到每一条时钟路径的情况（本设计中只有一条）；资源报告中可以看到每一个模块的资源消耗（本设计只有一个模块）。可以看到综合后与实现后的资源消耗情况、时序情况有些差异。

- System Generator集成的静态时序分析功能提供了如下特性：

  点击每一列的指标名称，可以选择升序/降序排列；

  时序不满足时，相应的路径Slack值为负数，且显示为红色；

  交叉定位功能：时序报告中选中某一路径，Simulink模型中对应的部分会高亮显示（时序满足为绿色；不满足为红色），这可以帮助设计者更快的找到和修改时序错误。

  如下图：

  <img src="https://s1.ax1x.com/2020/09/11/wUClUf.png" alt="wUClUf.png" style="zoom:50%;" />

  时序不满足时，可以考虑修改设计（如增加一些单元的Latency，以资源换速度），或者更换综合策略/实现策略。

  在System Generator block的Compilation标签下可以设置“Synthesis strategy”和“Implementation strategy”。列表中有几种Vivado提供的策略，也可以在Vivado中添加好用户自定义的策略，在System Generator中调用。

- 资源分析

  System Generator集成的资源分析功能提供了如下特性：

  点击每一列的指标名称，可以选择升序/降序排列；

  BRAMs（包括RAMB36E、FIFO36E、RAMB18E、FIFO18E0资源）；DSPs（包括DSP48E、DSP48E1、DSP48E2资源）；Registers（包括寄存器和触发器，以FD和LD开头的资源）；LUTs（所有的查找表）；

  交叉定位功能：资源报告中选中某一模块，Simulink模型中对应的部分会高亮显示（黄色）如下图：

  <img src="https://s1.ax1x.com/2020/09/11/wUPPMj.png" alt="wUPPMj.png" style="zoom:50%;" />

  资源消耗超过FPGA芯片总资源时，可以考虑修改设计，或者更换综合策略/实现策略。这里与时序分析部分相同。

### 3.6 小结

在System Generator中完成DSP系统设计与直接在Vivado环境下进行DSP系统设计相比，有三个优点：

- 更强大、更方便的仿真环境；

- 系统级设计角度，无需关心RTL设计细节以及一些IP核的具体使用方法。
- 在设计的过程中查看时序和资源问题，节省设计时间。

## 4. 常见模块以及参数介绍

## 5. 设计过程中的思考

- System Generator进行系统建模到完成一般包含以下四个步骤：
  1. 用数学语言描述算法。
  2. 在设计环境中实现算法，开始时使用双精度。
  3. 把双精度算法转换为定点算法。
  4. 把设计转换成有效的硬件。