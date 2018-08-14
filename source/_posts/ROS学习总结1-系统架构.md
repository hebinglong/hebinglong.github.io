---
title: ROS学习总结1:系统架构
date: 2018-08-14 13:30:04
tags:
---
# ROS系统架构

​	声明：本博客大部分摘抄自 **中国大学MOOC---《机器人操作系统入门》 课程讲义**，编写此博客主要是为了整理自己在学习ros时的思路并结合其他的一些优秀博文作为整理。

​	ROS系统的架构主要可以分为两个层级，分别是**文件系统级(The Filesystem level)** 和**计算图级(The Computation)**.

​	文件系统级主要根据ROS工作空间中文件的**物理组织形式**来展开，分别介绍有哪些目录，每个目录中的某个文件是干什么的；计算图级是以ROS整个控制系统中各个模块的**逻辑组织形式**来展开的，介绍ROS中的基本组件以及组件之间建立联系的方式。

## 文件系统级

### 目录结构

在ROS的官方wiki里面，一个标准的Catkin工作空间应该是下列的样子：

```
workspace_folder/         -- WORKSPACE 工作空间
  src/                    -- SOURCE SPACE 源文件空间
    CMakeLists.txt        -- The 'toplevel' CMake file
    package_1/
      CMakeLists.txt
      package.xml
      ...
    package_n/
      CATKIN_IGNORE       -- Optional empty file to exclude package_n from being processed
      CMakeLists.txt
      package.xml
      ...
  build/                  -- BUILD SPACE 编译空间
    CATKIN_IGNORE         -- Keeps catkin from walking this directory
  devel/                  -- DEVELOPMENT SPACE (set by CATKIN_DEVEL_PREFIX) 开发空间
    bin/
    etc/
    include/
    lib/
    share/
    .catkin
    env.bash
    setup.bash
    setup.sh
    ...
  install/                -- INSTALL SPACE (set by CMAKE_INSTALL_PREFIX) 安装空间
    bin/
    etc/
    include/
    lib/
    share/
    .catkin             
    env.bash
    setup.bash
    setup.sh
    ...
```

- 工作空间（一般也叫catkin工作空间）：一个包含功能包、可编辑源文件或编译包的文件夹。当你向同时编译不同的功能包时非常方便，并且可以用来保存本地开发包。
  - 源文件空间：src/: ROS的catkin软件包（源代码包） 
  - 编译空间 build/: catkin（CMake） 的缓存信息和中间文件 
  - 开发空间 devel/: 生成的目标文件（包括头文件，动态链接库，静态链接库，可执行文件等） 、环
    境变量 

### 开发工作流

在编译过程中，它们的工作流程如图 ：

|      src/       | -->  |           bulid/           | -->  |  devel/  |
| :-------------: | :--: | :------------------------: | :--: | :------: |
| package源代码包 | -->  | cmake&catkin缓存和中间文件 | -->  | 目标文件 |

​	**后两个路径由catkin系统自动生成、管理，我们日常的开发一般不会去涉及，而主要用到的是src文件夹，我们写的ROS程序、网上下载的ROS源代码包都存放在这里。**在编译时，catkin编译系统会递归的查找和编译 src/ 下的每一个源代码包。因此你也可以把几个源代码包放到同一个文件夹下。

### Catkin编译系统

​	上面介绍了工作空间，那么为什么叫catkin工作空间，catkin到底是什么呢？

​	对于源代码包，我们只有编译才能在系统上运行。而Linux下的编译器有gcc、g++，随着源文件的增加，直接用gcc/g++命令的方式显得效率低下，人们开始用Makefile来进行编译。然而随着工程体量的增大，Makefile也不能满足需求，于是便出现了Cmake工具。CMake是对make工具的生成器，是更高层的工具，它简化了编译构建过程，能够管理大型项目，具有良好的扩展性。对于ROS这样大体量的平台来说，就采用的是CMake，并且ROS对CMake进行了扩展，于是便有了Catkin编译系统。

​	简单来讲就是catkin可以生成CMakeLists.txt，cmake工具根据CMakeLists.txt可以生成Makefile，make工具根据Makefile调用gcc等编译器，经过编译、链接等阶段生成最终的可执行文件。所以catkon是比cmake抽象层度更高的编译系统。

#### catkin的特点

Catkin是基于CMake的编译构建系统，具有以下特点： 

- Catkin沿用了包管理的传统像 find_package() 基础结构, pkg-config 
- 扩展了CMake，例如 
  - 软件包编译后无需安装就可使用 
  - 自动生成 find_package() 代码， pkg-config 文件 
  - 解决了多个软件包构建顺序问题 

#### catkin软件包的组成

一个Catkin的软件包（package） 必须要包括两个文件： 

- package.xml: 包括了package的描述信息 
  - name, description, version, maintainer(s), license 
  - opt. authors, url's, dependencies, plugins, etc... 
- CMakeLists.txt: 构建package所需的CMake文件 
  - 调用Catkin的函数/宏 
  - 解析 package.xml 
  - 找到其他依赖的catkin软件包 
  - 将本软件包添加到环境变量 

#### catkin工作原理

catkin编译的工作流程如下： 

1. 首先在工作空间 catkin_ws/src/ 下递归的查找其中每一个ROS的package。 

2. package中会有 package.xml 和 CMakeLists.txt 文件，Catkin(CMake)编译系统依据 CMakeLists.txt 文件,从而生成 makefiles (放在 catkin_ws/build/ )。 

3. 然后 make 刚刚生成的 makefiles 等文件，编译链接生成可执行文件(放在 catkin_ws/devel )。

**也就是说，Catkin就是将 cmake 与 make 指令做了一个封装从而完成整个编译过程的工具。catkin有比较突出的优点，主要是： 操作更加简单 、一次配置，多次使用 、跨依赖项目编译 。**



### Package软件包

通过前面catkin的介绍，我们知道 Package是catkin编译的基本单元。我们调用catkin的编译命令的对象就是一个个package，也就是说，ROS里面，所有的源代码（无论是cpp还是python）必须组织成Package的形式才可以利用catkin工具链进行编译。一个package可以编译出多个目标文件（ROS可执行程序、动态静态库、头文件等）。

#### package组织结构

一个package下常见的文件、路径有： 

```
├── CMakeLists.txt #(必须) 定义package的包名、依赖、源文件、目标文件等编译规则，是package不可少的成分
├── package.xml #(必须)描述package的包名、版本号、作者、依赖等信息，是package不可少的成分 
├── src/ #存放ROS的源代码，包括C++的源码和(.cpp)以及Python的module(.py) 
├── include/ #存放C++源码对应的头文件 
├── scripts/ #存放可执行脚本，例如shell脚本(.sh)、Python脚本(.py) 
├── msg/ #存放自定义格式的消息(.msg) 
├── srv/ #存放自定义格式的服务(.srv) 
├── models/ #存放机器人或仿真场景的3D模型(.sda, .stl, .dae等) 
├── urdf/ #存放机器人的模型描述(.urdf或.xacro) 
├── launch/ #存放launch文件(.launch或.xml) 
```

​	其中定义package的是 CMakeLists.txt 和 package.xml ，这两个文件是package中必不可少
的。catkin编译系统在编译前，首先就要解析这两个文件。这两个文件就定义了一个package。 

​	通常ROS文件组织都是按照以上的形式，这是约定俗成的命名习惯，建议遵守。以上路径中，**只有 CMakeLists.txt 和 package.xml 是必须的**，其余路径根据软件包是否需要来决定。 

#### CMakeLists.txt文件

​	CMakeLists.txt 原本是Cmake编译系统的规则文件，而Catkin编译系统基本沿用了CMake的编译风格，只是针对ROS工程添加了一些宏定义。所以在写法上，catkin的 CMakeLists.txt 与CMake的基本一致。 这个文件直接规定了这个package要依赖哪些package，要编译生成哪些目标，如何编译等等流程。所以 CMakeLists.txt 非常重要，它指定了由源码到目标文件的规则，catkin编译系统在工作时首先会找到每个package下的 CMakeLists.txt ，然后按照规则来编译构建。 

​	CMakeLists.txt 的基本语法都还是按照CMake，而Catkin在其中加入了少量的宏，总体的结构如下： 

| -cmake基本函数           | -功能                                                |
| ------------------------ | ---------------------------------------------------- |
| cmake_minimum_required() | CMake的版本号                                        |
| project()                | 项目名称                                             |
| find_package()           | 找到编译需要的其他CMake/Catkin package               |
| add_library()            | 生成库                                               |
| add_executable()         | 生成可执行二进制文件                                 |
| add_dependencies()       | 定义目标文件依赖于其他目标文件，确保其他目标已被构建 |
| target_link_libraries()  | 链接                                                 |
| install()                | 安装至本机                                           |



| -catkin新加宏         | -功能                                                        |
| --------------------- | ------------------------------------------------------------ |
| catkin_python_setup() | 打开catkin的Python Module的支持                              |
| add_message_files()   | 添加自定义Message/Service/Action文件                         |
| add_service_files()   |                                                              |
| add_action_files()    |                                                              |
| generate_message()    | 生成不同语言版本的msg/srv/action接口                         |
| catkin_package()      | catkin新加宏，生成当前package的cmake配置，供依赖本包的其他软件包调用 |
| catkin_add_gtest()    | 生成测试                                                     |

#### package.xml文件

​	package.xml 也是一个catkin的package必备文件，它是这个软件包的描述文件，在较早的ROS版本(rosbuild编译系统)中，这个文件叫做 manifest.xml ，用于描述pacakge的基本信息。如果你在网上看到一些ROS项目里包含着 manifest.xml ，那么它多半是hydro版本之前的项目了。 

​	pacakge.xml 包含了package的名称、版本号、内容描述、维护人员、软件许可、编译构建工具、编译依赖、运行依赖等信息。实际上 rospack find 、 rosdep 等命令之所以能快速定位和分析出package的依赖项信息，就是直接读取了每一个pacakge中的 package.xml 文件。它为用户提供了快速了解一个pacakge的渠道。 

​	pacakge.xml 遵循xml标签文本的写法，由于版本更迭原因，现在有两种格式并存（format1与format2） ，不过区别不大。在新版本（format2） 中，包含的标签为： 

| -标签               | -标签作用                                      |
| ------------------- | ---------------------------------------------- |
| pacakge             | 根标记文件                                     |
| name                | 包名                                           |
| version             | 版本号                                         |
| description         | 内容描述                                       |
| maintainer          | 维护者                                         |
| license             | 软件许可证                                     |
| buildtool_depend    | 编译构建工具，通常为catkin                     |
| depend              | 指定依赖项为编译、导出、运行需要的依赖，最常用 |
| build_depend        | 编译依赖项                                     |
| build_export_depend | 导出依赖项                                     |
| exec_depend         | 运行依赖项                                     |
| test_depend         | 测试用例依赖项                                 |
| doc_depend          | 文档依赖项                                     |

#### launch文件

机器人是一个系统工程，通常一个机器人运行操作时要开启很多个node，对于一个复杂的机器人的启动操作应该怎么做呢？当然，我们并不需要每个节点依次进行rosrun，ROS为我们提供了一个命令能一次性启动master和多个node。该命令是： 

```
$ roslaunch pkg_name file_name.launch
```

​	roslaunch命令首先会自动进行检测系统的roscore有没有运行，也即是确认节点管理器是否在运行状态中，如果master没有启动，那么roslaunch就会首先启动master，然后再按照launch的规则执行。launch文件里已经配置好了启动的规则。 所以 roslaunch 就像是一个启动工具，能够一次性把多个节点按照我们预先的配置启动起来，减少我们在终端中一条条输入指令的麻烦。 

launch文件同样也遵循着xml格式规范，是一种标签文本，它的格式包括以下标签： 

| -标签      | -标签功能                    |
| ---------- | ---------------------------- |
| launch     | 根标签                       |
| node       | 需要启动的node及其参数       |
| include    | 包含其他launch               |
| machine    | 指定运行的机器               |
| env-loader | 设置环境变量                 |
| param      | 定义参数到参数服务器         |
| rosparam   | 启动yaml文件参数到参数服务器 |
| arg        | 定义变量                     |
| remap      | 设定参数映射                 |
| group      | 设定命名空间                 |
| /launch    | 根标签                       |

### Package命令行工具

| -命令             | -作用                            |
| ----------------- | -------------------------------- |
| rospack           | 获取信息或者在系统中查找工作空间 |
| catkin_create_pkg | 创建一个新的功能包               |
| catkin_make       | 编译工作空间                     |
| rosdep            | 安装功能包的系统依赖             |
| rqt_dep           | 查看包的依赖关系图               |
| roscd             | 更改目录 类Linux的 cd命令        |
| rosed             | 编辑文件                         |
| roscp             | 从一些功能包复制文件             |
| rosd              | 列出功能包的目录                 |
| rosls             | 功能包下的文件                   |

### 软件元包Metapackage

​	在一些ROS的教学资料和博客里，你可能还会看到一个Stack（功能包集） 的概念，它指的是将多个功能接近、甚至相互依赖的软件包的放到一个集合中去。但Stack这个概念在Hydro之后就取消了，取而代之的就是Metapackage。尽管换了个马甲，但它的作用没变，都是把一些相近的功能模块、软件包放到一起。 其中 moveit  就是运动规划相关的（主要是机械臂） 功能包集。 

​	总结：metapackage是一些只有一个文件的特定包，它是package.xml。它不包含其他文件，如代码等。



### 其他常见文件类型

| -文件类型          | -文件作用                                                    |
| ------------------ | ------------------------------------------------------------ |
| lunch文件          | launch文件一般以.launch或.xml结尾，它对ROS需要运行程序进行了打包，通过一句命令来启动。一般launch文件中会指定要启动哪些package下的哪些可执行程序，指定以什么参数启动，以及一些管理控制的命令。 launch文件通常放在软件包的 launch/ 路径中中。 |
| msg/srv/action文件 | ROS程序中有可能有一些自定义的消息/服务/动作文件，为程序的发者所设计的数据结构，这类的文件以.msg , .srv , .action 结尾，通常放在package的 msg/ , srv/ , action/ 路径下。 |
| urdf/xacro文件     | urdf/xacro文件是机器人模型的描述文件，以.urdf或.xacro结尾。它定义了机器人的连杆和关节的信息，以及它们之间的位置、角度等信息，通过urdf文件可以将机器人的物理连接信息表示出来。并在可视化调试和仿真中显示。 |
| yaml文件           | yaml文件一般存储了ROS需要加载的参数信息，一些属性的配置。通常在launch文件或程序中读取.yaml文件，把参数加载到参数服务器上。通常我们会把yaml文件存放在 param/ 路径下 。 |
| dae/stl文件        | dae或stl文件是3D模型文件，机器人的urdf或仿真环境通常会引用这类文件，它们描述了机器人的三维模型。相比urdf文件简单定义的性状，dae/stl文件可以定义复杂的模型，可以直接从solidworks或其他建模软件导出机器人装配模型，从而显示出更加精确的外形。 |
| rviz文件           | rviz文件本质上是固定格式的文本文件，其中存储了RViz窗口的配置（显示哪些控件、视角、参数） 。通常rviz文件不需要我们去手动修改，而是直接在RViz工具里保存，下次运行时直接读取。 |

## 计算图级

​	通过上述文件系统级的描述，我们知道一个完整的ROS体系里面，最重要的是src/目录下面的多个package。这些package通过catkin编译系统生成一个个的可执行文件，多个可执行文件与ROS本身构成了完整的控制系统。那么问题就来了，我们知道，在操作系统层面上，每一个可执行文件其实就是一个进程，进程之间的通讯有socket、管道等，那么在ROS上构建的这些package是如何联系在一起的呢？接下来我们要讲的就是ROS的通讯架构。

​	**ROS的通信架构是ROS的灵魂，也是整个ROS正常运行的关键所在。**

ROS的通讯方式有以下四种：

- Topic 主题
- Service 服务
- Parameter server 参数服务器
- Action 动作

 ### Node & Master

​	在正式介绍ROS的通讯方式之前，需要先理解两个在ROS中相当重要的概念：“节点 Node”与“节点管理器 Master”。

​	在ROS的世界里，最小的进程单元就是节点（node） 。一个软件包里可以有多个可执行文件，可执行文件在运行之后就成了一个进程(process)，这个进程在ROS中就叫做节点。 从程序角度来说，node就是一个可执行文件（通常为C++编译生成的可执行文件、Python脚本）被执行，加载到了内存之中；从功能角度来说，通常一个node负责者机器人的某一个单独的功能。 

​	由于机器人的元器件很多，功能庞大，因此实际运行时往往会运行众多的node，负责感知世界、控制运动、决策和计算等功能。那么如何合理的进行调配、管理这些node？这就要利用ROS提供给我们的节点管理器master, master在整个网络通信架构里相当于管理中心，管理着各个node。node首先在master处进行注册，之后master会将该node纳入整个ROS程序中。node之间的通信也是先由master进行“牵线”，才能两两的进行点对点通信。当ROS程序启动时，第一步首先启动master，由节点管理器处理依次启动node。 

### Topic 主题

#### Topoc命令

| -命令                            | -作用                                                        |
| :------------------------------- | :----------------------------------------------------------- |
| rostopic bw topic_name           | 显示主题所使用的带宽                                         |
| rostopic echo topic_name         | 将消息输出到屏幕                                             |
| rostopic find message_type       | 按照类型查找主题                                             |
| rostopic hz topic_name           | 显示主题的发布频率                                           |
| rostopic info topic_name         | 输出活动主题、发布的主题、主题订阅者和服务的信息             |
| rostopic list                    | 输出活动主题的列表                                           |
| rostopic pub topic_name type arg | 将数据发布到主题。它允许我们直接从命令行中对任意主题创建和发布数据 |
| rostopic type topic_name         | 输出主题的类型，或者说主题中发布的消息类型                   |





### Service 服务

### Parameter server 参数服务器

### Action 动作

## 博客TODO





----ROS中常用的工具介绍

分别讲述ROS中4种通讯模型（消息 服务 参数管理器 动作）

---分析package中关键文件

分析cmakelist例子

分析package.xml例子

分析luach文件例子



----使用ros提供的通讯模型

创建一个自定义mesg（编写msg文件）

创建一个自定义的service（srv文件）

创建一个action（编写action文件）



*使用Bag

可视化工具的运用
使用TF
URDF与xacro
MoveIt