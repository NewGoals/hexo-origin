---
title: ubuntu16.04安装caffe
commends: false
date: 2019-12-30 15:02:09
categories:
    - caffe
tags:
    - caffe
---

<center>为什么要使用caffe？很多人可能会这样问我。caffe有高效的cuda/C++实现、Matlab/Python接口、独特的神经网络描述方式、清晰的代码框架...都不是，有时候哪有这么多的理由，只是初见而已，就喜欢上了这个框架。</center>

<!--more-->

- - - - - - -

> 至于为什么要是用ubuntu而不使用centos等等其他linux的发行版，完全是看个人喜好了，我也在centos环境下安装过，甚至还在windows下安装过。linux环境的安装都大同小异，网上寻找较为靠谱的教程跟着安装即可。至于windows环境的安装，我不太推荐。一个是微软有官方版本的caffe，也可以去bvlc的github下载源码,装好之后虽然能在windows环境下运行，但是很多caffe自带的bash工具无法使用，windows的命令我也不是很熟，所以在这里不做推荐。这篇博客里我也只记录一些我觉得比较重要的点，这些内容全靠我在安装之后的回想，可能会有遗漏，仅供参考。

## Ubuntu换源
如果你已经换过国内源了，或者是觉得官方源对你来说没有影响，那么此条内容可以跳过。
```
cd /etc/apt     # 进入apt目录
sudo cp sources.list sources.list.bak   # 将原来的源进行备份
sudo vim sources.list      # 由于我们已经做好了备份，直接将里面内容替换即可，如果是桌面版也可以用gedit，较为方便
```
我比较推荐阿里云的源，当然也可以换成其他的，网上有很多
### 阿里云源
``` bash
# deb cdrom:[Ubuntu 16.04 LTS _Xenial Xerus_ - Release amd64 (20160420.1)]/ xenial main restricted
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
```
### 更新源
```
sudo apt-get update
```

## 安装caffe需要的依赖包
> **ProtoBuffer** 由Google开发的一种可以实现内存与非易失存储介质交换的协议接口，通俗点来讲就是硬盘等存储文件，参见[protobuffer_C++接口文档][protobuffer]

> **Boost** 被称为C++准标准库，是一个功能强大，构造精巧，跨平台，开源且免费的库，caffe主要用来解决字符串的运算（或者是我只接触到这里？）。

> **GFLAGS** caffe主要用来进行命令行参数解析的作用，与用户进行交互。

> **GLOG** 由Google开发的用于记录应用程序日志的实用库，提供基于C++标准输入输入流形式的接口，记录时可以选择不同的日志级别，方便将重要日志和普通日志分开。

> **BLAS** caffe主要调用其中的方法做矩阵，向量的计算，最常用的BLAS实现有Intel MKL,ATLAS,OpenBLAS，caffe可以选择其中的任意一种，进入Makefile.config将BLAS有关设置改为BLAS := open即可，表示我们使用OpenBLAS。

> **HDF5** 是美国国家高级计算中心(NCSA)为了满足各种领域研究需求而研制的一种能高效存储和分发科学数据的新型数据格式。caffe的训练模型经常保存为hdf5格式。

> **OpenCV** 世界上最流行的开源计算机视觉库。caffe主要使用opencv来完成一些图像存取和预处理功能。

> **LMDB和LEVELDB** 闪电般的内存映射型数据库管理器(LMDB)，在caffe中的作用主要是提供数据管理。所有的数据都要通过一定的预处理来将其转化为LMDB类型才能被caffe的网络读取。LEVELDB库是caffe早期版本使用的数据存储方式，由Google开发，虽然现在大部分历程都已经使用LMDB代替了LEVELDB，但是为了与以前的版本兼容，仍然将这个库增添到依赖中。当然，我肯定会推荐使用LMDB而避免使用LEVELDB。

> **Snappy** 用来压缩和解压缩的C++库，旨在提供较高的压缩速度和合理的压缩率。
```
sudo apt-get install git
sudo apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler
sudo apt-get install --no-install-recommends libboost-all-dev
sudo apt-get install libopenblas-dev liblapack-dev libatlas-base-dev
sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev
```
如果你不喜欢使用ubuntu的apt-get工具自动安装的话，以上的所有依赖包均可以自行下载安装编译。

## 安装Nvidia显卡驱动
我不太推荐CPU版本的caffe，因为计算力过于低下。如果你使用的是ubuntu桌面版，那么直接进入[Nvidia官网](https://www.nvidia.com/Download/index.aspx)下载相对应的显卡驱动即可，如果使用的服务器或者是只有命令行的版本，可以使用apt-get的方法来安装，但是考虑到有些显卡的不支持，我推荐使用另一台可以正常访问Nvidia官网的电脑记录下载地址使用wget安装，同样的命令在安装后面的cudnn可能会遇到困难，因为cudnn的下载是需要用户登录的，可能需要在wget命令中填写cookies。

如果原先有驱动但是不太适合的话，可以先进行卸载，进入驱动所在目录
``` bash
sudo chmod 777 *.run     # 给执行权限
sudo ./NVIDIA-Linux-x86_64-390.59.run --uninstall      # 使用自带的工具进行卸载
```
### 禁用nouveau
将ubuntu自带显卡驱动加入黑名单。虽然我没遇到过类似的问题，但是如果你碰到了类似于显卡驱动无法安装的情况，不妨试试这种方法
```
sudo vim /etc/modprobe.d/blacklist-nouveau.conf
```
进入后增加如下一条命令，将nouveau加入黑名单
```
blacklist nouveau
```
更新后重启电脑使其生效
```
sudo update-initramfs -u
```
进入显卡驱动安装目录
``` bash
sudo chmod 777 NVIDIA-Linux-x86_64-xxx.xx.run      # 主要是给执行权限。将xxx.xx换成安装的驱动版本即可，或者是tab键补全
./ NVIDIA-Linux-x86_64-xxx.xx.run
```
按照提示完成安装，重启后使用nvidia-smi查看驱动版本及显卡当前状态，有结果表明安装成功。

## 安装CUDA
CUDA是一种由NVIDIA推出的通用并行计算架构，该架构使GPU能够解决复杂的计算问题。同样的，桌面版用户可以直接去[CUDA官网][cuda]安装，具体细节可以参考网上的资料或者是[CUDA官方文档][cudaDocument]，强烈推荐根据官方文档中的步骤进行安装，官方文档中有着非常详细的说明，根据自己的系统选择合适的安装方式即可。

如果你已经确定了自己的安装版本，那么进入下载页面根据自己的系统选择合适的安装包，假设选择的是runfile，那么下载完成后，根据下载页面的提示，进入该目录执行以下命令：
``` bash
sudo sh cuda_xxx_xxx_linux.run      # tab补全命令即可
```
按照提示安装重启即可。

重启后进入用户根目录下，编辑\~/.bashrc文件或者是/etc/bashrc文件，使得打开shell时添加环境变量，根据操作系统不同，\~/.bashrc和\~/.profile是用户目录下的环境变量设定，/etc/bashrc和/etc/profile是全局环境变量设定，具体设置的区别可以自行了解，这里我们只更改当前用户的环境变量设置。
```
sudo vim ~/.bashrc      # 编辑.bashrc
```
在末尾添加路径和库(假设我们安装的是cuda10.1)，x86_64-linux-gnu两个动态库貌似是ubuntu16.04单独增加的？不管怎样，也把它们添加进来。":"是分隔符，可以将这些路径写到一起，但是这样较为方便。
``` bash
export PATH=/usr/local/cuda-9.0/bin$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/usr/lib:$LD_LIBRARY_PATH
```
使用source命令执行该脚本
```
source ~/.bashrc
```
检测cuda是否安装成功
```
cd /usr/local/cuda-10.1/samples/1_Utilities/deviceQuery
sudo make
./deviceQuery
```
正如其名，devicequery是cuda自带的一个样例，用来检测设备，如果能够正确识别到你的所有GPU，那么cuda安装成功。

## 安装cuDNN
使用GPU计算已经比CPU快很多了，但是依旧不满足计算力，那么可不可以在不改变硬件的条件下，使得GPU的计算力得到进一步提升呢？当然可以。为了达到更高的性能，我们可以借助专业加速库cuDNN。我们需要通过[网站][cudnn]首先注册为CUDA开发者才能获得下载权限，这个是支持社交账号登陆的。Ubuntu桌面版的用户当然可以直接登录下载，要注意你的cudnn与CUDA版本是对应的，下载相应的linux版本。cuda的版本应以你实际下载的版本为准，nvidia-smi工具查询到的结果只是他建议的cuda版本，与实际下载的版本无关。如果是服务器或者是纯命令行的用户，会发现wget下载时会遇到一些问题，但是可以通过wget实现cookie欺骗来进行下载，有需要的可以自行了解。

下载完成后将其解压，得到一个cuda文件夹，进入该文件夹执行以下命令，因为cudnn归根到底是cuda的库，所以我们将其头文件和动态库都复制到/usr/local/cuda下。/usr/local/下的cuda是一个软链接，指向我们真正下载的cuda-10.1文件夹。因此我们在配置时尽量使用cuda而不是cuda-10.1，以后安装更换cuda版本时，直接改变cuda的指向即可。
```
sudo cp lib64/lib* /usr/local/cuda/lib64/
sudo cp include/cudnn.h /usr/local/cuda/include/
```
进入/usr/local/cuda/lib64文件夹下，更改动态库(假设我们安装的是cudnn 7.6.5)，具体什么原因不太明白。
``` bash
sudo rm -f libcudnn.so libcudnn.so.7
sudo ln -s libcudnn.so.7.6.5 libcudnn.so.7
sudo ln -s libcudnn.so.7 libcudnn.so
sudo ldconfig   # 使动态库生效
```
所以现在的链接关系是libcudnn.so->libcudnn.so.7->libcudnn.so.7.6.5

## 下载caffe
使用git下载caffe的github源码，make编译时是根据Makefile.config来编译的，我们复制一份Makefile.config.example并将其命名为Makefile.config，将原来的作为备份。
```
git clone https://github.com/bvlc/caffe.git
cd caffe/
cp Makefile.config.example Makefile.config
```

## Makefile.config文件配置
主要还是看编译过程中错误怎么报，根据报错来解决比较好，我只是提供一个我自己的配置方法，但是在不同的设备和版本上运行的时候仍有可能报错。

使用vim编辑Makefile.config文件，cudnn前面的注释取消
``` conf
# cuDNN acceleration switch (uncomment to build with cuDNN).
USE_CUDNN := 1
```
使用`pkg-config --modversion opencv`查看opencv版本，如果是opencv3的话，将opencv前面的注释取消
``` conf
# Uncomment if you're using OpenCV 3
OPENCV_VERSION := 3
```
根据使用的cuda版本注释相应的ARCH，我使用的是10.1，所以删掉了所有的*_20和*_21，就成了下面这个样子
``` conf
# CUDA architecture setting: going with all of them.
# For CUDA < 6.0, comment the *_50 through *_61 lines for compatibility.
# For CUDA < 8.0, comment the *_60 and *_61 lines for compatibility.
# For CUDA >= 9.0, comment the *_20 and *_21 lines for compatibility.
CUDA_ARCH := -gencode arch=compute_30,code=sm_30 \
                -gencode arch=compute_35,code=sm_35 \
                -gencode arch=compute_50,code=sm_50 \
                -gencode arch=compute_52,code=sm_52 \
                -gencode arch=compute_60,code=sm_60 \
                -gencode arch=compute_61,code=sm_61 \
                -gencode arch=compute_61,code=compute_61
```
BLAS我们选择OpenBlas，将其改为open
``` conf
# BLAS choice:
# atlas for ATLAS (default)
# mkl for MKL
# open for OpenBlas
BLAS := open
```
在命令行状态下使用`sudo find / -name hdf5.h` 和 `sudo find / -name hdf5_hl.h `查看hdf5.h和hdf5_hl.h头文件路径，将其添加到Makfile.config中，一般而言应为/usr/inlucde/hdf5/serial，动态库也要单独的添加，同样的使用`sudo find / -name libhdf5.so`，也可以得知它并不在一般的/usr/lib或者/usr/local/lib中，因此也要单独添加
``` conf
# Whatever else you find you need goes here.
# Whatever else you find you need goes here.
INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include /usr/include/hdf5/serial
LIBRARY_DIRS := $(PYTHON_LIB) /usr/local/lib /usr/lib /usr/lib/x86_64-linux-gnu/hdf5/serial
```

## Makefile文件配置
找到LIBRARIES，将我们之前通过apt-get下载的依赖库添加进去
``` makefile
LIBRARIES +=glog gflags protobuf boost_system boost_filesystem m hdf5_serial_hl hdf5_serial
```
找到NVCCFALGS，增加预处理和编译时需要的宏-D_FORCE_INLEINES，貌似可以防止nvcc编译出错？不太明白，先添加进去，防止出一些莫名其妙的错误
```
NVCCFLAGS += -D_FORCE_INLINES -ccbin=$(CXX) -Xcompiler -fPIC $(COMMON_FLAGS)
```


## 添加动态链接库路径
感觉有点无关紧要，Makefile.config中会单独指明路径。编辑/etc/ld.so.conf，添加以下路径
``` 
include /etc/ld.so.conf.d/*.conf
include /usr/lib
include /usr/local/lib
```
执行`ldconfig`使其生效

## caffe编译
设置完成后使用`make all -j`编译，一般来说按照我上面的设置不会遇到问题，如果还是出现了各种各样的问题，按照报的具体错误去研究解决，以下为我安装过程中的出现的一些错误，可供参考。出现错误不要紧，`make clean`后解决问题在编译即可
### "#warning "math_functions.h is an internal header file and must not be used directly. This file will be removed in a future CUDA release. Please use cuda_runtime_api.h or cuda_runtime.h instead.""
根据提示，进入/src/caffe/util/math_functions.cu文件中，用cuda_runtime.h头文件代替原来的math_functions.h，高版本的cuda都会用cuda_runtime.h来代替，也可以选择忽视该问题
``` c
//#include <math_functions.h>  // CUDA's, not caffe's, for fabs, signbit
#include <cuda_runtime.h>  //cuda10.1 support,not math_functions.h
```

### "/usr/bin/ld: cannot find -lhdf5_hl和/usr/bin/ld: cannot find -lhdf5"
原因在于找不到hdf5动态库文件，可以使用`sudo find / -name libhdf5.so`，ubuntu16.04 LTS是将该动态库文件放在/usr/lib/x86_64-linux-gnu/hdf5/serial下的，因此我们需在Makefile.config中指明编译时的路径
```
LIBRARY_DIRS := $(PYTHON_LIB) /usr/local/lib /usr/lib /usr/lib/x86_64-linux-gnu/hdf5/serial
```

## 编译完成
运行`make all -j`编译不报错即为编译完成，也可以运行`make runtest -j`来进行测试，测试过程可能较为漫长，请耐心等待。如果没有错误会显示PASSED
```
[----------] Global test environment tear-down
[==========] 2207 tests from 285 test cases ran. (270650 ms total)
[  PASSED  ] 2207 tests.
```
到此为止caffe安装编译完成，编译好的文件会放到caffe根目录的build文件夹下，其他接口的编译暂不作说明，感兴趣的可以自己去网络上查找资料。

附上贾扬清博士的caffe介绍：http://caffe.berkeleyvision.org/


[protobuffer]: https://developers.google.com/protocol-buffers/docs/cpptutorial
[cuda]: https://developer.nvidia.com/cuda-downloads
[cudaDocument]: https://docs.nvidia.com/cuda/
[cudnn]: https://developer.nvidia.com/rdp/cudnn-download
