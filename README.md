# fuck-keil-guide（现代化STM32软件开发指南）

## 前言

换用开源工具链的理由很明显。Keil这种生态割裂的专有软件对代码规范化和系统化管理的危害有目共睹。

- 计划做的框架要简化调车的过程，但是最后权衡利弊却还是只能用C，因为Keil对C++支持稀烂。软件工程、设计模式在C语言的框架下完全施展不开，虽然代码的抽象程度有进步但是远不如专门的面向对象语言来的直观。

- 队里推广了git版本管理，但是Keil的工程文件不是人类可读的，而且时不时就会变一下，污染commit记录。试想如果未来要进行code review，这样完全不能让人接受。

- 编辑器难用。补全迟缓，没有自动格式化，搜索功能和记事本一样原始。

- 没有哪个CI/CD平台支持Keil这一套东西。

- 它整个流程所有轮子都是自己造的（工具链、构建系统、编辑器等等）。在当今开源的大环境下，所有人都在使用和完善那套开源的工具，Keil这种专有软件的生态规模和开源生态根本不能比。找到个第三方库发现它支持CMake，你却用不了，全靠复制粘贴；行吧，用CMake，又发现CMake不支持那个年久失修的ARMCC编译器。

只是因为“熟悉”，或者调试起来方便就守着Keil一直用绝对是因小失大。看看广工的CI/CD流水线给他们省了多少沟通和调试时间：一套代码给所有车用，仿真改都不用改直接就能跑。要跟上时代享受这些新技术带来的便利，就必须抛弃Keil，转向几乎所有C/C++开发者都在广泛应用的开源工具链。

本文会简要介绍使用ARMGCC+开源工具链进行现代化STM32软件开发的方法。进一步地，在介绍完这一套工具链的工作方式之后，会分章节介绍如何使用各种IDE简化开发过程。

只要学会了这一套，肯定就会意识到工具的组合是无穷无尽的，也就不在乎到底是用什么编辑器写代码，或者到底用什么调试器调试了。

*多嘴一句，CMSIS-DSP等一系列CMSIS库原生支持的就是用CMake构建。ARM的工程师都不用Keil你为什么还用？*

## 前置知识

**CMake+GCC**

这两样可以不熟但是一定要会用，本文假设读者熟悉桌面平台上使用CMake管理的C/C++工程构建过程，不会提平台无关的关于CMake的一些常见问题。

不会也没关系，学起来很简单。找个CMake教程看看就好。

## 复习一下：C/C++开发的一般过程

无论在什么平台上构建C/C++代码，都是经过`源代码->目标文件->可执行文件`三步。编译器把源代码(`.c`)编译成目标文件(`.o/.obj`)，链接器把代码编译出的一大堆目标文件(`.o/.obj`)和库(`.a/.lib/.so/.dll`)链接在一起，生成一个或者几个可执行文件(`/.bin/.elf/.exe`)或库(`.a/.lib/.so/.dll`)。

![](assets/1.jpg)

这是C/C++代码构建的一般过程，隐藏在IDE、构建系统等等一系列工具背后的本质。无需在意程序到底是在哪个IDE里点那个"build"按钮编译出来的，那只是个“壳”。只要有编译器和链接器，就可以弄出一个能跑的程序。

观察各种C/C++工具链，都可以看见编译器和链接器这两样东西。构建C/C++代码，就是依次把源代码送进这两样东西，然后他们俩吐出一个可执行文件。

MSVC (Visual Studio): `cl` `link`

![](assets/2.png)

GCC: `gcc` `g++` `ld`

![](assets/3.png)

还有Keil老登自带的ARMCC: `armcc` `armlink`

![](assets/4.png)

项目庞杂的情况下，就要一个一个文件编译，再把一大堆.o文件拷到一起链接。这是构建系统存在的原因，`CMake`和`make`会帮你干这个事。在C/C++项目根目录下经常可以看到`CMakeLists.txt`文件，我们根据需要编写这个文件，`CMake`根据这个文件生成`Makefile`，`make`根据`Makefile`自动编译和链接。这样就不用折磨自己，build一次要拷半个小时文件，再敲半个小时命令了。

IDE做的就是把这个过程打包在一起，做一个图形界面把它们都装起来。IDE里的"build"按钮，其实就是调用`CMake`和`make`，然后把编译器和链接器的输出显示在一个窗口里。IDE里的调试功能，其实就是调用`GDB`，然后把`GDB`的输出解析到编辑器窗口里，看起来就像是一行一行地调试代码。

## 类比到嵌入式开发

嵌入式平台上的C/C++开发和上述过程一模一样，只不过桌面平台直接运行调试就好，而嵌入式平台需要把可执行文件烧录到目标设备上，然后用调试器调试。

**所以，整个核心开发流程一共只需要五个软件：编译器、链接器、构建系统，加上烧录工具和调试器。**

下面，我们在STM32平台上走一遍标准的C/C++开发流程。

### 需要用到的工具

#### 交叉工具链(编译器和链接器)

用[GNU Arm Embedded Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)。从名字就能看出来，它就是嵌入式版的GCC。还有一个正在日渐强大的`LLVM/Clang`，本文不介绍，可以自己探索。

![](assets/5.png)

↑ 它长这个样子，眼熟么。

#### 构建系统

用[`CMake`](https://cmake.org/download/)+[`Ninja`](https://github.com/ninja-build/ninja/releases)。`Ninja`是一个比`make`编译速度快很多的替代品，两者用法基本一样。

#### 烧录工具

[`OpenOCD`](http://openocd.org/getting-openocd/)启动！

#### 调试器

最原始、最通用的方法是用`GDB`连接`OpenOCD`的`GDB Server`。很多IDE的调试功能底层就是这样。然而因为这种方法基本就是准备给IDE套壳的，在命令行里面手搓很不方便，所以本文不介绍。在分章节介绍不同IDE的环节，会推荐不同开发环境下最好用的调试方法。

参考阅读：[【Linux】GDB保姆级调试指南（什么是GDB？GDB如何使用？）](https://blog.csdn.net/weixin_45031801/article/details/134399664)

## 上工！

搜搜资料，把这些工具都安装好；我们来试试什么IDE都不用，只用CubeMX和命令行就让STM32跑起来。

### 生成工程

用CubeMX生成工程，注意要选择生成`STM32CubeIDE`的工程。

![](assets/14.png)

### 写代码

随便写点代码，比如点个灯啥的

### 写CMakeLists.txt

在这个生成好的工程的根目录下按照CubeMX里的配置编写`CMakeLists.txt`。因为手搓太麻烦，所以我们使用一套写好的cmake脚本[[patrislav1/cubemx.cmake](https://github.com/patrislav1/cubemx.cmake)]来解析.ioc文件。如果想学怎么手写CMakeLists.txt，可以看[这篇教程](https://github.com/MaJerle/stm32-cube-cmake-vscode)。

这里简要说一下这套脚本最基本的用法，也可以直接去原仓库看README。

1. 安装python。装过的话就不用装了。

2. 进到工程根目录，Clone这个仓库

    ```shell
    git clone https://github.com/patrislav1/cubemx.cmake
    ```

3. 把仓库里的`CMakeLists-example.txt`复制到工程根目录，重命名为`CMakeLists.txt`

    然后把这个文件改成这样：（注意看我写的中文注释）

    ```cmake
    cmake_minimum_required(VERSION 3.16)

    # Possible values: openocd, pyocd, stlink, blackmagic. stlink is default
    set(CMX_DEBUGGER "openocd")
    set(OPENOCD_CFG "${CMAKE_CURRENT_SOURCE_DIR}/openocd.cfg")
    # 1. ↑选一种调试器，去掉这两行的注释

    include(cubemx.cmake/cubemx.cmake)

    # 2. 设置工程名
    project(gongchengming)

    add_executable(${PROJECT_NAME})
    cubemx_target(
        TARGET ${PROJECT_NAME}
        # 3. 设置.ioc文件路径
        IOC "${CMAKE_CURRENT_LIST_DIR}/Template.ioc"
    )
    target_compile_options(${PROJECT_NAME} PRIVATE -Og -Wall -g -gdwarf-2)

    # 注意一下这两个宏，根据工程设置的不同，有时候会报错，其中一个要去掉
    # Depending on the project setup, sometimes one of these symbols must be omitted. (Cannot be reliably determined from the .ioc file)
    target_compile_definitions(${PROJECT_NAME} PRIVATE USE_LL_DRIVER USE_HAL_DRIVER)
    ```

4. 开始build。在工程根目录下新建一个`build`文件夹，进入这个文件夹，依次运行`cmake`和`ninja`。（这个算法组的同学熟;))）

    ```shell
    mkdir build
    cd build
    cmake -G Ninja -DCMAKE_TOOLCHAIN_FILE="../cubemx.cmake/arm-gcc.cmake" .. # -G指定一下用Ninja，-DCMAKE_TOOLCHAIN_FILE指定一下工具链配置文件。这两样不能少。
    ninja
    ```

    如果一切顺利，会输出构建进度和RAM/FLASH的使用情况，最后在`build`文件夹里生成一个`.elf`文件。我们现在把这个文件烧录到STM32上。

### 烧录

可以直接在命令行里调用`OpenOCD`进行烧录，不过这套cmake脚本还包含`OpenOCD`的烧录脚本，所以直接用就完了。如果想学怎么在命令行里用`OpenOCD`烧录，可以看[这篇教程](https://cloud.tencent.com/developer/article/1840792)。

1. 在工程根目录下新建一个`openocd.conf`。这个文件怎么写看教程。

    要用CMSIS-DAP调试器烧录FLASH大小为128KB的STM32F4的话，这个文件是这样的：

    ```
    adapter driver cmsis-dap
    transport select swd

    set FLASH_SIZE 0x20000
    source [find target/stm32f4x.cfg]

    # download speed = 10MHz
    adapter speed 10000
    ```

2. 执行烧录的target

    ```shell
    ninja flash
    ```

    等烧录完，按一下板子上的reset键，程序就跑起来了。完工。

### 调试

由于说过的原因，这里不介绍。实在想学的话上面有教程。

## 使用IDE简化开发过程

进行到这里，我们已经可以不用IDE，只用围绕GNU工具铺开的这套开源工具就完成STM32开发。这个非常重要，因为这样解决了用第三方库的问题，也让CI/CD成为可能，还让我们可以随便选一个壳（IDE）来写代码。由于开源生态过于强大，CMake已经是当今C/C++项目的标配，没有哪个当代的C/C++ IDE不支持CMake；我们随意选一个IDE，看看怎么把上面介绍的过程装进它的图形界面里；顺便，介绍一下不同IDE下最好用的调试方法。

- [CLion](./clion.md)

- [VSCode](./vscode.md)

- [Visual Studio](./vs2022.md)

还有很多其他IDE可以选，但是没必要，这三个是最好用的。想整活的话下面确实还有几个抽象的（我感觉Visual Studio已经够抽象了）：

- Eclipse

- Qt Creator

- Code::Blocks

- Vim/Emacs

## 完

至此，整个开发流程已经打通。现在，我们有了很多以前不能有的能力。

- 不需要任何额外配置就可以C++/C混合编译。

- `GNU Arm Embedded Toolchain`的开发和维护非常活跃，可以随意使用现代C++20甚至C++23的新功能。

- 由于本质上整个工程是一个CMake工程，很多现代软件开发的方法都可以直接拿来用。模块化、CI/CD等等做起来都变得很方便。

- 可以随意选择代码编辑器，VSCode、CLion等软件强大的功能会成倍提高开发效率，还会让代码变得更整洁易读。
