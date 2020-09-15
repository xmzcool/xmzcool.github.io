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

## 4. 常见参数介绍

- **Precision**

  XILINX Blockset库中的模块在仿真计算时按任意精度定点数进行，大部分模块的计算精度可由开发人员定义，包括位数(Number of Bits)和小数位(Binary Point)，如下图所示。

  ![](https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-15_16-50-23.png)

  在默认情况下，模块的输出为全精度(Full Precision)，提供了足够的精度以保证结果不会出错。大部分模块具有用户自定义精度(User-Defined Percision)选项，开发人员可自行确定数据的位数和小数位。

- **Arithmetic Type**

  在模块的参数对话框中可以定义无符号位或带符号位(二进制补码)数作为模块的输出信号的数据类型。

- **Number of bits**

  定点数以何种数据格式进行存储由其位数、小数位和运算类型决定，System Generator支持的最大位数为4096。小数位必须介于0到位数之间。

- **Overflow & Quantization**

  一旦开发人员定义了数据的精度，就会产生由溢出和量化带来的误差。当一数值超过了开发人员定义的数据所能表示的范围，就产生了溢出误差；当开发人员定义的数据的小数部分无法完全表示实际输入的小数数值时，就出现量化误差。

  对于开发人员定义的任意精度数据，System Generator都针对溢出和量化误差提供了多种处理方式。

  当发生溢出时，开发人员有三种处理方式，当选择Saturate模式时，将数据饱和在正的最大值或负的最小值;当选择Wrap模式时，作绕回处理，即最大值加1结果是最小值，最小值减1得到最大值:当选择Flag as error模式时，将数据溢出标记到Simulink的错误报告中。

  当出现量化误差时，开发人员有两种处理方式，当选择Round模式时，将数据四舍五入到最接近的开发人员定义精度可表示的数值上;当选择Truncate模式时，直接将开发人员定义精度无法表示部分数值丢弃。

- **Latency**

  许多XILINX Blockset库中的模块都具有延迟选项，定义该模块的输出信号延迟多少个采样周期后输出。System Generator中没有专门的模块或结构用于实现流水线，**给模块的输出添加延迟可以理解为在其输出端上存在有移位寄存器**，用以调节模块的处理时间，**使得不同的模块具有相同的处理时间**，实现流水线，提高系统处理速度。

- **Provide synchronous reset port**

  选中该选项，System Generator自动激活模块的复位端(rst)。当复位信号有效时，模块立即恢复到初始状态，复位信号必须为布尔量(Boolean)信号。

- **Provide enable port**

  选中该选项，System Generator 自动激活模块的使能端(en)。当使能信号无效时，模块保持当前状态，直到使能信号或复位信号有效时模块开始动作。复位信号的优先级高于使能信号，同样，使能信号也必须为布尔量(Boolean)信号。

- **Sample period**

  System Generator根据特定的采样率对数据流进行处理，通常每个模块按照采样率对输入信号进行采样并按照采样率输出信号，唯有Up Sample和Down Sample两个模块例外，分别用于实现提高、降低采样率。

- **Specify explicit sample period**

  如果选中该选项，开发人员可以按设计需要指定采样周期。该选项经常用于构成反馈回路的设计中，在反馈回路中，System Generator不能确定默认的采样率，因为回路的存在使得比较器的输入信号的采样率依据于尚未确定的反馈信号的采样率。在这种情况下，System Generator需要开发人员指定采样周期。

- **Use behavioral HDL(otherwise use core)**

  如果选中该选项，由MCode(MATLAB的M文件所用代码)仿真来生成行为级硬件描述语言，否则由IP核构成硬件描述语言。

- **Use pre-defined core placement information**

  如果选中该选项，生成的IP核包括相关的布局信息，通常使得最终得到的硬件具有较高的处理速度。也正因为IP核的布局会受到相应的约束，将影响到后续布局布线阶段的处理结果。

- **Placement style**

  当IP核布局信息选项被选中后，开发人员才可以设定布局风格选项。该选项用于直接设定模块在硬件上的布局。选中矩形(Rectangular shape)选项，FPGA 上相关查找表(LUTs)分布比较松散;选中三角(Triangular packing)选项，FPGA 上相关查找表(LUTs)分布比较紧凑。

- **Define FPGA area for resource estimation / FPGA area (slices, FFs, BRAMs,LUTs, IOBs, emb. mults, TBUFs)**

  Resource Estimator模块需要调用这部分信息，用于估算Resource Estimator模块所在的System Generator设计模型需要多少FPGA硬件资源。

  如果在System Generator 设计中包含有Resource Estimator模块，开发人员可以手动定义任意功能模块的FPGA资源使用量。如果没有输入任何参数，Resource Estimator 模块会自动计算相关数据并给出所需资源总量。

  如果开发人员希望自行定义设计中的每一功能 模块的FPGA资源使用量，就必须选中Define FPGA area for resource estimation 选项，Resource Estimator模块才会在处理时严格按照开发人员设定的参数进行计算;否则Resource Estimator模块的计算结果会覆盖开发人员设定的任意参数。

  一共有七个参数需要输入FPGA area栏，在填写时需要注意其先后顺序。七个参数依次表示本模块slices、触发器(FFs)、块状RAM(BRAMs)、查找表(LUTs)、IO模块(IOBs)、嵌入式乘法器(emb. mults)、三态缓存器(TBUFs)的使用量。Resource Estimator 模块进行处理时仅考虑需要占用硬件资源的模块，略过不占用硬件资源的模块，如Clock Enable Probe模块等。虽然Slice和LUT、FF是有关联的(每个Slice中包含有两个LUT和两个FF)，但在计算使用资源量时需要分别计算，因为在一个Slice中，不一定同时用到LUT和FF,并且不同的设计会有不同的结果。

- **Pipeline for maximum performance**

  在使用IP核构成设计模块时可在其内部构成流水线结构以提高处理速度，选中该选项System Generator可以最大限度利用模块的延迟时间构成流水线结构。

## 5. 常见模块介绍

只有XILINX模块才会在FPGA硬件中实现。

### 5.1 

### 5.2 Sysgen中的信号类型

为了提高硬件仿真精度，Sysgen中的模块采用了任意精度的定点数，最大支持4096位定点数，而在simulink中的连续时间的信号必须经过Gateway In模块采样成FPGA设计中的信号使用，输出经Gateway Out转变为simulink中的信号才可以实现两者互联。Sysgen共有如下3种信号类型：

![](https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-12_10-47-36.png)

- Boolean型，一些使能信号，开关信号要求输入为bool型。显示为Fix_N_n，N表示数据位宽，n表示小数位宽。
- Fixed-point型，定点数，可以指定数据位宽和小数位宽，其中可以分为以下两种：
  - Signed：有符号型，规则遵循2‘s comp，也就是补码。
  - Unsigned：无符号型。
- Floating-point型，浮点数，可以分为以下三种
  - Single：单精度浮点数，数据位宽32位，1（符号位，sign bit） + 8（指数位，exponent width） + 24（有效精度，significand precision）。
  - Double：双精度浮点数，数据位宽64位，1（符号位，sign bit） + 11（指数位，exponent width） + 53（有效精度，significand precision）。
  - Costom：用户自定义指数位和小数位位宽。

FPGA设计可以考虑使用**定点数据类型**（Fixed-point）或**浮点数据类型**（Floating-point），但后者消耗的资源几乎是前者的十几倍甚至更多，设计中通常都采用定点数据格式。

**与信号类型相关的模块：**

System Generator中有两个与此相关的block：Convert和Reinterpret，都可以进行数据的转换。

- **Reinterpret**

  这个block可以完成以下数据转换功能：

  - 将无符号数转换为带符号数；
  - 将带符号数转换为无符号数；
  - 通过重新规定小数点位置来定义数据范围。

  需要注意的是，“**转换**”在这个block中的含义更接近于其英文直译“**重新解释**”。事实上，数据在经过该block后，其位宽与每一位的值都没有发生任何改变，变化的只有其所表示的“**意义**”。一个二进制数是无符号数还是带符号数、小数点在哪一位仅仅取决于设计者如何规定和看待它。而Reinterpret改变的便是这种**“规定和看待”方式**。

  比如，“1100”这个数，当视作**UFix_4_0**（无符号定点数、4Bits位宽、小数部分0bit）时，其值为12；当视作**Fix_4_2**（带符号定点数、4Bits位宽、小数部分2Bits）时，其值为-1。因为reinterpret实现的只是一种意义上的转换，因此其在转换为FPGA设计后，**不会消耗任何资源**。

  既然reinterpret的输出和输入完全相同，那么加入此模块有什么作用？

  - 从FPGA设计转换到Simulink环境中时会按设定的“意义”解析数据格式；
  - 完成不同格式数据之间的的拼接。

  <img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-12_11-10-10.png" style="zoom:60%;" />

  选中“Force Arithmetic Type”后，输出数据格式的“意义”将转换为（没有选中，则输出与输入的表征意义相同）：无符号数（**Unsigned**）、带符号数二进制补码（**Signed(2’s comp)**）、浮点数（**Floating-point**）。

  选中“Force Binary Point”后，可以重新规定输出数据的小数点位置。比如设置为31时，表明数据中的低31Bits为小数部分。

- **Convert**

  该block不仅可以完成数据类型的转换，还具有如下特性：

  - 重新设置数据的量化、溢出方式；
  - 重新设置定点数格式（进行数据截位、转换）。

  <img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-12_11-19-14.png" style="zoom: 50%;" />

  其中大部分参数设置方法与Gateway In模块的设置完全相同，这里只讲述两者不同的地方。

  1. Convert模块的量化方式可以配置为“**Round(unbiased: even values)**”，这是针对“四舍五入”量化方式的缺点所作的改进:

  - 传统的四舍五入所有的中间值（如1.5、2.5）都会向更大的值量化，即**不是完全对称**的，这样会导致一组数据量化后平均值**高于**量化前的平均值。
  - unbiased: even values在处理中间值时会向更接近的偶数量化。比如1.5会量化为2；2.5仍然会量化为2（因为二者最接近的偶数都是2）。这样量化规则在整体上会呈现出**对称性**。

  2. 选中“**Provide enable port**”后，block会增加一个en使能管脚，只有当使能有效时convert block才会执行数据转换功能，否则将保持当前状态不变。

  3. **Latency**设置了Convert输出数据要经过多少个**采样时钟周期**的延时。注意不要混淆“采样时钟周期”和“FPGA时钟周期”的概念，在过采样系统中，一个采样时钟周期可能等于多个FPGA时钟周期。

  Convert在导出到FPGA中时会以Floating-Point IP核的方式实现，“Implementation”标签下可以设置Latency在FPGA中的实现方式：

  ●选中“Pipeline for maximum performance”时，**Latency以流水线的方式实现**，即在计算过程中增加中间级寄存器，以更多的资源实现更快的计算速度和更大的数据吞吐量。

  ●未选中“Pipeline for maximum performance”时，**Latency以在IP核末尾增加一级移位寄存器的方式实现**，这样只是单纯的实现了延时功能。

### 5.3 Sysgen中的时钟

相关时钟概念见5.2章节。下面介绍一下时钟相关的模块。

下面举个例子：添加如下几个模块并连线。

![](https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-15_15-14-12.png)

如上图，输入信号的速率在Gateway In模块中设置为1，而Up Sample分别设置为2、3、4(用两个速率为2 Up Sample的模块级联而成)，那么Simulink的系统时钟周期应该为1、1/2、 1/3、 1/4 的最大公因子1/12。信号时钟输出如下：

![](https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-15_15-30-08.png)

- Clock Probe模块，在仿真中该模块被用来测试**系统速率运行**的时钟信号。**该模块仅用来仿真**，输出类型是双精度浮点数。当生成硬件时，该模块将被忽略。

- Up sample和Down sample模块：可以进行采样率变换，SystemGenerator中可以设置不同颜色代表不同的采样率。可以点击Display-〉Sample Time中设置。双击设置可以看到：

  <img src="https://gitee.com/xmzcool/bloglmage/raw/master/img/Snipaste_2020-09-15_16-12-57.png" style="zoom:67%;" />

  Simpling rate是指升采样倍数，要求为整数。

  Copy samples是指升采样后的结果是持续的，还是只在升采样后的第一个clk出现，其余时间为低电平。

- Clock Enable Probe模块，输出为时钟使能信号，输入为模型中的任意信号，输出是与该信号的采样率同频率的脉冲布尔量(Boolean)信号，一般用于多采样率系统。从上图可以看出时钟使能信号都是相应采样周期的周期脉冲。当生成硬件时，该模块将被忽略。

## 6. 设计过程中的思考

### 6.1 设计流程概括

System Generator进行系统建模到完成一般包含以下四个步骤：

1. 用数学语言描述算法。
2. 在设计环境中实现算法，开始时使用双精度。
3. 把双精度算法转换为定点算法。
4. 把设计转换成有效的硬件。

### 6.2 Simulink系统时间和硬件时钟频率

在Simulink中没有明确的时钟概念，FPGA的时钟信号周期也不等于仿真的时钟周期，而是由外部的时钟源信及内部DCM或者PLL共同决定的。在Sysgen中可以在System Generator模块的参数设置中为simulink系统周期和FPGA时钟频率指定具体的值，从而把simulink采样周期和FPGA硬件时钟周期联系起来。

举个例子来说明，假设一个设计中有多个模块具有不同的采样周期，比如分别为2s和3s两个采样周期，那么Simulink的系统周期应设为其最大公因子即1s，如果FPGA系统时钟周期是10ns，那么Simuink系统周期、2s和3s两个采样周期分别对应FPGA期间实现时的10ns、20ns和30ns。**另一种做法是将Simulink系统周期就定为FPGA的系统时钟周期，省去了时钟周期间的换算。**

## 7. 常见错误

### 7.1 生成报告错误

错误提示：Parsing of timing analysis data didn't complete successfully. Exiting the program。

解决方法：在Simulink中右键单击xilinx令牌，悬停在"Xilinx工具"选项上。几秒钟后，Simulink会有 点颤抖，用户界面中的一些按钮可能会重新加载。