# 《高级计算机系统结构》课程Project

王重熙 201928013229111

## 1 实验内容

### 1.1 实验目标

使用开源的EDA工具链，探索完成小规模芯片从RTL代码设计到GDS的全流程，包括：架构设计、前端RTL设计、综合、后端物理设计（包括布局、 时钟树综合、布线、时序分析等），其中RTL设计可以自行设计，也可以使用开源芯片的RTL设计。

开源EDA工具使用OpenROAD-flow，Github仓库见https://github.com/The-OpenROAD-Project/OpenROAD-flow 。

### 1.2 OpenROAD-flow的流程

OpenROAD-flow从RTL代码到GDS版图的全部流程如下图所示，RTL设计的Verilog代码及其它设计和约束文件经过综合（Synthesis）、布图规划（Floorplanning）、布局（Placement）、时钟树综合（Clock Tree Synthesis）、布线（Routing）等过程得到最终的GDS版图：

![OpenROAD-flow flow](./image/flow.png)

- 逻辑综合（Logic Synthesis）

    逻辑综合是OpenROAD工作流程的第一步，RTL设计经过逻辑综合得到描述逻辑门连接的门级网表netlist，OpenROAD-flow使用开源的Yosys + ABC工具进行逻辑综合。

- 物理设计（Physical Design）

    物理设计的目的是从逻辑电路（门级网表）得到芯片制造所需要的版图，版图描述了晶体管和晶体管在不同层中的布局，以及导线互联，芯片的版图以GDSII文件格式保存，称为GDS版图。

    从门级网表netlist到GDS版图需要经过以下几步：

    - 布图规划（FloorPlanning）

        布图规划是为了确定芯片主要模块的位置，主要确定以下几项内容：

        1. 芯片的面积；

        2. 门和电线的尺寸；

        3. 输入引脚和输出引脚的位置；

        4. 电力输送网络PDN（Power Delivery Network）；

        5. 芯片中IP块的位置以及与IO引脚的连接等。

        OpenROAD-flow使用verilog2def、ioPlacer、MacroPlacer、PDN和tapcell等开源工具完成布图规划。

    - 布局（Placement）

        布局是确定不同电路元件具体位置的步骤，将电路元件（cell）对应到布图上的详细坐标。布局分为全局布局（Global Placement）和详细布局（Detailed Placement）：全局布局将所有cell分布到相应位置，可能带有一些重叠；详细布局将每个cell移动到附近的不重叠的合法位置。

        OpenROAD-flow使用RePIAce、Resizer和OpenDP等开源工具完成布局。

    - 时钟树综合（Clock Tree Synthesis）

        时钟树综合是将时钟信号均匀分布到电路中的过程，尽可能减少clock skew和延迟。

        OpenROAD-flow使用TritonCTS和OpenDP完成时钟树综合。

    - 布线（Routing）

        每个cell的位置在布局中被确定下来，布线则是确定cell间连线的路径。

        OpenROAD-flow使用FastRoute工具进行全局布线，使用TritonRoute工具进行详细布线。

    - 完成物理设计

        OpenROAD最后将DEF文件转换为GDS版图。


## 2.1 实验环境搭建

### 2.1.1 安装CentOS 7虚拟机

OpenROAD-flow支持MacOS、Windows、Linux等多种操作系统，不过开发者仅在CentOS 7上完成了测试，因此我在Windows上使用VMWare搭建了CentOS 7虚拟机作为本次实验的平台，VMWare的版本是VMWare Workstation 15 Player，CentOS 7的镜像是CentOS 7 x86_64 18.04。

我参照https://blog.csdn.net/babyxue/article/details/80970526 完成了CentOS 7虚拟机的搭建，分配了8G DDR4-2400内存，40G硬盘空间，CPU是8核Core i7-7700HQ。

### 2.1.2 安装OpenROAD-flow

参照https://github.com/The-OpenROAD-Project/OpenROAD-flow 安装OpenROAD-flow，一共有三种安装方法：
- Option1 Installing build exports，下载pre-build的安装包，不需要自己编译；
- Option2 Building the tools using docker，使用docker搭建；
- Option3 Building the tools locally，不使用docker，直接在本地环境中搭建。

我从Option1开始尝试，结果Option1安装失败；之后使用Option2安装成功。

#### 2.1.2.1 Option1

OpenROAD-flow第一种安装方法分为四步：

1. 首先使用git clone克隆OpenROAD-flow的仓库：

    git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD-flow.git

2. 直接从下面的网址下载pre-build好的安装包：

    https://github.com/The-OpenROAD-Project/OpenROAD-flow/releases/tag/alpha2

3. 将下载的tar解压到`OpenROAD-flow/tools/OpenROAD`文件夹

4. 使用下面的命令更新环境变量：

    source setup_env.sh

安装完毕后，使用以下三条命令测试安装的正确性：

    yosys -h
    openroad -h
    TritonRoute -h

问题1

- 问题描述：

    当使用` git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD-flow.git`复制OpenROAD仓库时，产生下面的错误：

        Cloning into 'src/FastRoute'...
        remote: Enumerating objects: 227, done.
        remote: Counting objects: 100% (227/227), done.
        remote: Compressing objects: 100% (126/126), done.
        error: RPC failed; result=18, HTTP code = 200iB | 6.00 KiB/s
        fatal: The remote end hung up unexpectedly
        fatal: early EOF
        fatal: index-pack failed
        Clone of 'https://github.com/The-OpenROAD-Project/FastRoute.git' into submodule path 'src/FastRoute' failed
        Failed to recurse into submodule path 'tools/OpenROAD'

- 原因分析：

    子模块FastRoute克隆失败，在克隆FastRoute时下载速度只有几KB/s，首先怀疑是访问github时产生的网络问题。

    进入OpenROAD-flow目录，使用`git submodule update --init --recursive`尝试再次更新子目录，发现克隆FastRoute子模块时再次失败；但是奇怪的是，根据`.gitmodules`文件中找到的子模块链接，直接使用命令`git clone --recursive https://github.com/The-OpenROAD-Project/FastRoute.git`就可以克隆成功。

- 解决方法：

    在`tools/OpenROAD/src`中直接执行`git clone --recursive https://github.com/The-OpenROAD-Project/FastRoute.git`，即可将FastRoute子模块克隆下来，之后返回OpenROAD-flow目录继续使用`git submodule update --init --recursive`完成OpenROAD-flow的下载。

    后续还有几个子模块包括OpenDB、replace等都遇到了同样的问题，解决方法也相同。

问题2

- 问题描述:

    安装完毕后，运行`yosys -h`、`openroad -h`和`TritonRoute -h`报错`command not found...`，查看`setup_env.sh`，发现加入`$PATH`的三个路径分别是`${modroot}/build/OpenROAD/src`、`${modroot}/build/TritonRoute`和`${modroot}/build/yosys/bin`，然而从https://github.com/The-OpenROAD-Project/OpenROAD-flow/releases 下载下来的安装包按照README规定的解压位置解压后，对应的可执行文件分别位于`${modroot}/OpenROAD/build/src`、`${modroot}/OpenROAD/build/src/TritonRoute`和`${modroot}/OpenROAD/build/src/yosys`。

    修改`setup_env.sh`文件，将正确的路径加入到`$PATH`后，运行`yosys -h`和`openroad -h`均报错`error while loading shared libraries: libtcl8.5.so: cannot open shared object file: No such file or directory`。

- 原因分析：

    首先是release、README和master branch不匹配的问题，在Alpha2 prebuilt binaries for CentOS 7发布之后OpenROAD-flow的master branch有200多次提交，虽然没有定位到具体的哪一个commit，但是`setup_env.sh`和OpenROAD-flow的目录结构应该有改动，造成了安装的失败。此外`yosys -h`和`openroad -h`提示找不到共享库，可能是发布的OpenROAD-flow prebuilt版本并不是完全免安装的，依然需要安装一些dependency的库，但是README中并没有说明。

- 解决方法：

    无。Option1安装失败，release或README需要更新。接下来尝试Option2。

#### 2.1.2.1 Option2

OpenROAD-flow第二种安装方法分为三步：

1. 使用git clone克隆OpenROAD-flow的仓库，这一步与Option1中相同

2. 确保docker守护程序（docker daemon）运行且`docker`命令被添加到了`$PATH`中，之后使用docker进行build：

    ./build_openroad.sh

3. 在docker容器中打开交互式shell：

    docker run -it -u $(id -u ${USER}):$(id -g ${USER}) openroad/flow bash

OpenROAD-flow的仓库在尝试Option1的安装方法时已经克隆到了本地。

我参照https://www.cnblogs.com/yufeng218/p/8370670.html 在CentOS中安装了docker，docker版本号为docker-ce-19.03.9-3。由于docker daemon最初没有运行，使用下面的命令将docker daemon加入到开机启动项中：

    systemctl enable docker

由于网络问题，我在执行`./build_openroad.sh`曾多次出错，主要发生在`git clone https://github.com/The-OpenROAD-Project/abc.git`和`yum install`，这些错误再次运行`./build_openroad.sh`即可解决；此外下面问题3和问题4中的错误需要修改Dockerfile。

问题3

- 问题描述：

    运行`./build_openroad.sh`创建docker镜像，报错：

        Loaded plugins: fastestmirror, ovl
        Cannot open: https://centos7.iuscommunity.org/ius-release.rpm. Skipping.
        Error: Nothing to do
        The command '/bin/sh -c yum group install -y "Development Tools"     && yum install -y https://centos7.iuscommunity.org/ius-release.rpm     && yum install -y wget centos-release-scl devtoolset-8     devtoolset-8-libatomic-devel tcl-devel tcl tk libstdc++ tk-devel pcre-devel     python36u python36u-libs python36u-devel python36u-pip &&     yum clean -y all &&     rm -rf /var/lib/apt/lists/*' returned a non-zero code: 1

- 原因分析：

    在浏览器中打开https://centos7.iuscommunity.org/ius-release.rpm ，发现连接到了一个github的issue：https://github.com/iusrepo/announce/issues/18 ，现在https://centos7.iuscommunity.org/ius-release.rpm 的链接不再维护，现在维护的是https://repo.ius.io/ius-release-el6.rpm 和https://repo.ius.io/ius-release-el7.rpm 。

- 解决方法：

    将`OpenROAD-flwo/tools/OpenROAD/Dockerfile`第6行的`https://centos7.iuscommunity.org/ius-release.rpm`修改为`https://repo.ius.io/ius-release-el7.rpm`，再次运行`./build_openroad.sh`即可。

问题4

- 问题描述：

    解决问题3后， 再次运行`./build_openroad.sh`报错：

        https://hkg.mirror.rackspace.com/epel/7/x86_64/repodata/ea5cfe85e50491bfba7524d64236cd8a954f1b59e82813857e2f2e90d5de5a45-updateinfo.xml.bz2: [Errno 14] curl#7 - "Failed connect to hkg.mirror.rackspace.com:443; Connection refused"
        Trying other mirror.
        No package git2u available.
        Error: Nothing to do
        The command '/bin/sh -c yum -y remove git && yum install -y git2u' returned a non-zero code: 1

- 原因分析：

    在Google中搜索相关问题，发现一个类似的Github issue：https://github.com/iusrepo/git216/issues/5 ，issue中提到`yum --enablerepo=ius-archive install git2u`可以正确安装git2u。

- 解决方法：

    将`OpenROAD-flow/tools/OpenROAD/Dockerfile`第25行的`RUN yum -y remove git && yum install -y git2u`修改为`RUN yum -y remove git && yum install -y --enablerepo=ius-archive git2u`，之后再次运行`./build_openroad.sh`成功。

安装完毕后运行`docker images`，可以看到列表中出现openroad、openroad/flow、openroad/yosys和openroad/tritonroute等四个镜像。

使用`docker run -it openroad/flow bash`在docker容器中打开交互式shell，运行`source setup_env.sh`更新环境变量，运行`yosys -h`、`openroad -h`和`TritonRoute -h`测试安装的正确性，安装成功！

## 3 OpenROAD-flow工具链demo从RTL到GDS全流程

如`OpenROAD-flow/flow/Makefile`第1行到第48行所示，OpenROAD-flow有数十个demo，支持的平台包括nangate45、tsmc65lp、gf14、invecas12、sw130等等。

OpenROAD-flow默认只安装了nangate45的平台，因此我只测试了使用nangate45的12个demo，包括aes、black_parrot、bp_be_top、bp_fe_top、bp_multi_top、dynamic_node、gcd、ibex、jpeg、swerv、swerv_wrapper和tinyRocket。

### 3.1 gcd

OpenROAD-flow通过`DESIGN_CONFIG`这个变量来指定具体的设计，demo中默认的是最简单的gcd模块，只需要大约250个cell，我首先完成完成`gcd.v`在nangate45平台上从verilog到GDS版图的全流程：

    docker run -it openroad/flow bash
    cd flow
    make

问题5

- 问题描述：

    `make`报错：

        /usr/bin/time: cannot run klayout: No such file or directory
        0:00.00elapsed 73%CPU 360memKB
        make: *** [results/nangate45/gcd/6_1_merged.gds] Error 127

- 原因分析：

    怀疑是klayout未安装成功。

- 解决方法：

    参照`OpenROAD-flow/Dockerfile`第六行，在openroad/flow的docker容器中运行`yum localinstall https://www.klayout.org/downloads/CentOS_7/klayout-0.26.4-0.x86_64.rpm -y`。

手动安装klayout后`make`成功，得到gcd的GDS版图。

### 3.2 其它的11个demo

在完成gcd从RTL到GDS的全流程后，说明OpenROAD-flow工具链安装正确且运行无误，接下来我又完成了其它的11个demo从RTL到GDS的全流程：

    docker run -it openroad/flow bash
    cd flow

    make DESIGN_CONFIG=./designs/nangate45/aes.mk
    make DESIGN_CONFIG=./designs/nangate45/black_parrot.mk
    make DESIGN_CONFIG=./designs/nangate45/bp_be_top.mk
    make DESIGN_CONFIG=./designs/nangate45/bp_fe_top.mk
    make DESIGN_CONFIG=./designs/nangate45/bp_multi_top.mk
    make DESIGN_CONFIG=./designs/nangate45/dynamic_node.mk
    make DESIGN_CONFIG=./designs/nangate45/ibex.mk
    make DESIGN_CONFIG=./designs/nangate45/jpeg.mk
    make DESIGN_CONFIG=./designs/nangate45/swerv.mk
    make DESIGN_CONFIG=./designs/nangate45/swerv_wrapper.mk
    make DESIGN_CONFIG=./designs/nangate45/tinyRocket.mk

其中swerv_wrapper出现bug，bug详细信息见问题6，其余均完成从RTL到GDS的全流程。

问题6

- 问题描述：

    运行`make DESIGN_CONFIG=./designs/nangate45/swerv_wrapper.mk`报错：

        make: *** [results/nangate45/swerv_wrapper/3_2_place_resized.def] Error 9

- 原因分析：

    在docker容器中打开`OpenROAD-flow/flow/logs/nangate45/swerv_wrapper`文件夹查看所有产生的log文件，发现log文件只产生到`3_2_resizer.log`，说明只进行到了使用Resizer工具进行详细布局这一步。打开`OpenROAD-flow/flow/logs/nangate45/swerv_wrapper/3_2_resizer.log`，发现总负时序裕量TNS（Total Negative Slack）和最差负时序裕量WNS（Worst Hold Slack）均为负值：

        ==========================================================================
        report_tns
        --------------------------------------------------------------------------
        tns -2654963.25

        ==========================================================================
        report_wns
        --------------------------------------------------------------------------
        wns -346.61

    最差负时序裕量WNS出现在这条路径上：

        ==========================================================================
        report_checks
        --------------------------------------------------------------------------
        Startpoint: _253006_ (rising edge-triggered flip-flop clocked by core_clock)
        Endpoint: _260932_ (recovery check against rising-edge clock core_clock)
        Path Group: **async_default**
        Path Type: max

        Delay    Time   Description
        ---------------------------------------------------------
        0.00    0.00   clock core_clock (rise edge)
        0.00    0.00   clock network delay (ideal)
        0.00    0.00 ^ _253006_/CK (DFFR_X1)
        0.08    0.08 v _253006_/Q (DFFR_X1)
        0.04    0.12 v _233754_/Z (BUF_X1)
        0.04    0.15 ^ _179879_/ZN (INV_X1)
        0.03    0.18 ^ _226933_/ZN (OR2_X1)
        0.03    0.21 ^ _226934_/ZN (AND2_X1)
        66.08   66.28 ^ _250135_/Z (BUF_X1)
        275.18  341.46 ^ _260932_/RN (DFFR_X1)
             341.46   data arrival time

        10.00   10.00   clock core_clock (rise edge)
        0.00   10.00   clock network delay (ideal)
        0.00   10.00   clock reconvergence pessimism
              10.00 ^ _260932_/CK (DFFR_X1)
        -15.14   -5.14   library recovery time
              -5.14   data required time
        ---------------------------------------------------------
              -5.14   data required time
            -341.46   data arrival time
        ---------------------------------------------------------
            -346.61   slack (VIOLATED)

    因此判断可能是时序违例，达不到要求的时钟频率。在docker容器中打开swerv_wrapper的设计文件`OpenROAD-flow/flow/designs/nangate45/swerv_wrapper.mk`，发现设定的是时钟频率是100MHz：

        export CLOCK_PERIOD = 10.000

    将时钟改为50MHz，即`export CLOCK_PERIOD = 20.000`，清除上一次的数据`make clean_all make DESIGN_CONFIG=./designs/nangate45/swerv_wrapper.mk`，重新运行`make DESIGN_CONFIG=./designs/nangate45/swerv_wrapper.mk`，依然出现相同的错误，且TNS和WNS完全一致，没有任何改变。

- 解决方法：

    无。swerv_wrapper的RTL到GDS全流程失败，时序违例且违例得非常离谱（346.61 ns），应该不是正常设计会出现的bug，没有找到bug原因。

### 3.3 demo总结

我使用OpenROAD-flow工具链完成了自带的demo中nangate平台上11个demo的RTL到GDS全流程，唯一失败的demo是swerv_wrapper，失败原因见问题6。

在`OpenROAD-flow/flow/reports/nangate45/design_name/`中，可以看到具体每一个设计的RTL到GDS全流程的报告，下表总结了这11个成功的demo的详细数据，包括

| Name         | DIE_AREA (um)      | CORE_AREA (um)            | CLOCK_PERIOD (ns) | # Cells | Power (mW) | Design Area (um^2) | Ultilization (%) |
| ------------ | ------------------ | ------------------------- | ----------------- | ------- | ---------- | ------------------ | ---------------- |
| aes          | 0 0  620.15  620.6 | 10.07 11.2  610.27  610.8 |  5                |  19571  | 1.59E-03   |  1438128           | 100              |
| black_parrot | 0 0 2200.01 2199.4 | 10.07 11.2 2189.94 2189.6 |  5.6              | 218035  | 5.52E+02   | 20003624           | 100              |
| bp_be_top    | 0 0 1550.02 1342.6 | 10.07 11.2 1540.14 1332.8 |  5.6              |  72715  | 2.92E+02   |  8534981           | 106              |
| bp_fe_top    | 0 0  999.97  799.4 | 10.07  9.8  989.9   789.6 |  5.6              |  45088  | 3.21E+02   |  3556117           | 116              |
| bp_multi_top | 0 0 1550.02 1342.6 | 10.07 11.2 1540.14 1332.8 |  5.6              | 132400  | 8.53E+02   |  9117903           | 113              |
| dynamic_node | 0 0  450.17  450   | 10.07 11.2  440.29  440.2 | 15                |  16512  | 2.26E-03   |   737122           | 100              |
| gcd          | 0 0  100.13  100.8 | 10.07 11.2   90.25   91   | 10                |    431  | 2.89E-05   |    25593           | 100              |
| ibex         | 0 0  600.08  599.8 | 10.07 11.2  590.01  590   | 10                |  39075  | 3.20E-03   |  1341146           | 100              |
| jpeg         | 0 0 1200.04 1199.8 | 10.07  9.8 1189.97 1190   |  4                |  87227  | 8.41E-03   |  5570072           | 100              |
| swerv        | 0 0 1550.02 1342.6 | 10.07 11.2 1540.14 1332.8 | 10                | 130858  | 1.03E-02   |  8088562           | 100              |
| tinyRocket   | 0 0  924.92  799.4 | 10.07  9.8  914.85  789.6 |  5.6              |  38636  | 5.62E+01   |  2830570           | 100              |


Github上开源的RISCV处理器核（语言为verilog）

- OPenV/mriscv
- Hummingbird E200
- PicoRV32
- SERV
- biRISC-V
- DarkRISCV
- SSRV
- Tinyriscv


## 参考文献
- https://blog.csdn.net/babyxue/article/details/80970526 ，CentOS安装
- https://github.com/The-OpenROAD-Project/OpenROAD-flow ，OpenROAD-flow仓库
- https://openroad.readthedocs.io/en/latest/user/getting-started.html ，OpenROAD Getting Started
- https://blog.abdelrahmanhosny.me/tech/eda/2019-12-06-getting-started-with-openroad-1/ ，OpenROAD Getting Started
- https://blog.abdelrahmanhosny.me/tech/eda/2019-12-06-getting-started-with-openroad-2/ ，OpenROAD Getting Started
- https://www.cnblogs.com/yufeng218/p/8370670.html ，docker安装
- https://blog.csdn.net/qq_29999343/article/details/78294604 ，docker设置开机启动项