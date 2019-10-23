系统环境为 Debian 9.11
源码版本 3.7

## Ice的编译

1. 下载 ice-3.7 源码

   我们只编译 cpp 版本，编译前有必要看下 Readme.txt 文件

2. 安装依赖的包
   lmdb, mcpp

3. 修改 根目录下的 config/Make.rules 文件

    1. prefix 设置库的安装路径
       
       我们使用默认的 /opt/Ice-$(version) 路径
    
    2. 因为我们的目的是调试和跟踪， 设置 OPTIMIZE ?= no
    
    3. 编译配置, 可以通过 如下命令查看所有支持的配置
       
       `make print V=supported-configs`
    
        其中 cpp11 前缀的是支持 c++11 的版本,在此我们使用 cpp11-shared 配置
   
4. 在 .bashrc 中设置 ICE_HOME 环境变量为我们的安装路径
    
   ICE_HOME=/opt/Ice-3.7.3/

5. 进入到 cpp 文件夹中, 执行如下命令进行安装
   
   sudo make CXXFLAGS=-std=c++11 CONFIGS=cpp11-shared V=1 install
   
   

## Ice的调试

1. 下载 ice-demos-3.7 

2. 到 cpp11 中 执行 `make Ice/hello` 编译 hello 的例子

3. gdb 调试

   1. gdb ./server
   
   2. 设置 调试源码路径
   
      set substitute-path src/Ice/ObjectAdapterI.cpp /home/liuyanan/playground/ice-3.7/cpp/

   3. 设置断点(带路径)
   
      `b src/Ice/ObjectAdapterI.cpp:163`

   4.  输入r开始调试 
   