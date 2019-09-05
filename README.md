[TOC]

# docker容器配置(踩坑记录)

## 1.  partner.io

在https://hub.docker.com/r/nvidia/cuda上找到所需要的镜像，我选择的tag是`10.0-cudnn7-devel-ubuntu18.04`。然后打开http://192.168.1.51:9000/#/containers，在左边栏中选择Containers，再点击+Add container。接下来进行container的配置

<center>
<img border:0; display:inline; src="1.png" width=70%/>
</center>

Name随意填写，Image在前面选的tag前加上nvidia/cuda:，如我的Image应填`nvidia/cuda:10.0-cudnn7-devel-ubuntu18.04`。在Manual network port publishing处点击`+publish a new network port`，并在host中填写`10022`或`10024`或`10026`……(若前一个端口已被占用则依次递推)，映射到container的`22`端口。Entry Point填写`/bin/bash`，Console选择 `Interactive & TTY (-i -t)`。

在Runtime & Resources中将Runtime选为nvidia，最后点击Deploy the container即可。

最后启动container，打开它的console。



## 2.  安装并配置SSH

在console中由于是root权限，直接运行以下语句：

```bash
apt-get update
apt-get upgrade
apt-get install vim
apt-get install openssh-client
apt-get install openssh-server
vim /etc/ssh/sshd_config
```

然后在将sshd_config中的以下PermitRootLogin修改为：

```bash
PermitRootLogin yes
```

保存后回到bash，用`passwd`指令为root用户设置密码，让另一端的用户能登录。

然后再执行

```bash
/etc/init.d/ssh start
```

执行以下语句确认ssh-server是否启动：

```bash
ps -e|grep ssh
```

假若看到了sshd那么ssh-server已正常启动。





## 3.  远程连接+基础配置

在你的另一台电脑终端上运行

```bash
ssh root@192.168.1.51 -p 10024
```

并输入密码，即可实现正常SSH连接。



### 3.1  用户配置

首先执行：

```bash
adduser username
apt-get install sudo
vim /etc/sudoers
```

在`root ALL=(ALL) ALL`下面加上`username (ALL)=(ALL) ALL`，强制保存退出username用户即可使用sudo。

```bash
su username
cd ~
```



### 3.2  Anaconda3配置(若用tf-docker则无需该步)

配置Anaconda3

```bash
wget https://repo.anaconda.com/archive/Anaconda3-2019.07-Linux-x86_64.sh
bash Anaconda3-2019.07-Linux-x86_64.sh
```

打开profile文件

``` bash
sudo vim /etc/profile
```

在文件末尾处添加

```bash
export PATH=/home/username/anaconda3/bin:$PATH
```

回到bash中，执行

```bash
source /etc/profile
```

再输入python，出现Anaconda, Inc.则表示成功，也可输入`echo $PATH`查看已有的环境变量。



### 3.3  Cuda环境变量完善

由于默认的镜像里安装了Cuda10.0但还未配置好系统环境变量，故需要手动配置。

在bash中输入

```bash
sudo vim ~/.bashrc
```

在该文件末尾输入

```bash
export CUDA_HOME=/usr/local/cuda
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATHs
# export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/username/anaconda3/lib 此句会带来问题，删去
```

最后回到bash执行

```bash
source ~/.bashrc
```

检查环境变量是否生效，可在bash中执行

```bash
nvcc -V
```

若出现回应，则说明Cuda环境变量已配置好。



### ~~3.4  Cudnn安装及配置(有Cuda的镜像无需此步)~~

~~为适应Cuda 10.0，并保证Tensorflow等能正常运行，选择了Cudnn v7.4.2 Library for Linux。~~

<center>
<img border:0; display:inline; src="2.png" width=80%/>
</center>

~~在终端中执行~~

```bash
# tar -xvzf cudnn-10.0-linux-x64-v7.4.2.24.tgz
# sudo cp cuda/include/cudnn.h /usr/local/cuda/include/
# sudo cp -d cuda/lib64/libcudnn* /usr/local/cuda/lib64/
# sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*
```

~~可以用以下指令来验证cudnn安装的版本：~~

```bash
# cat /usr/local/cuda/include/cudnn.h | grep CUDNN_MAJOR -A 2
```



### 3.5  安装git

配置git

```bash
sudo apt-get install git
git config --global user.name "your name"
git config --global user.email "your_email@example.com"
```



==接下来是分支配置(非必须)==

### A.  SSH连接github(加快git clone速度)

在bash中输入

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

一直按确认键选择默认即可。接着输入

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

将SSH密钥添加到GitHub帐户，Title随意取，将/home/username/.ssh/id_rsa.pub的内容粘贴到Key中。

回到bash中，输入以下语句测试连接是否成功：

```bash
ssh -T git@github.com
```

接下来就可以在git clone过程中使用SSH方式，下载速度会比https快一些。



### B.  配置locate

```bash
sudo apt-get install mlocate
sudo updatedb
```

之后即可使用locate定位文件。



### C.  Tensorflow docker与pycharm连接问题

tensorflow docker与pycharm配合使用时会出现SFTP连接不上的问题，解决方案有两个(目前推荐第二个，第一个不能解决根本问题)：

1. 

```bash
vim /etc/ssh/sshd_config
```

将其中的以下行

```
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

改为

```
Subsystem       sftp    internal-sftp
```

2. 

删去/etc/bash.bashrc中的所有内容，并使其生效

```bash
sudo cp /etc/bash.bashrc /etc/bash.bashrc.backup
sudo vim /etc/bash.bashrc
source /etc/bash.bashrc
```



**PS:**

- 如果运行时出现路径前一堆乱码的情况时，打开右上角的Edit Configurations，将Script path调好，注意运行不要再使用Ctrl+Shift+F10，直接Run当前文件，即可正常运行。

- pycharm运行的py文件中matplotlib打印不出图片时，首先要`sudo apt-get install python3-tk`，然后再`sudo vim ~/.bashrc`，在其中加上DISPLAY环境变量:





## 4.  框架分支

接下来是深度学习框架的安装，根据需要安装对应的框架即可，尽量不要同时安装多个。

### I.  配置caffe

有两种方法(推荐第二种，第一种中的caffe无法在使用Anaconda3中的python时import)：

1. 直接在终端输入

```bash
sudo apt-get install caffe-cuda
```

2. 下载源码进行编译

首先我们要从GitHub的远端下载caffe的源码

```bash
git clone git@github.com:BVLC/caffe.git
```

如果前文没有配置SSH连接GitHub，那么请使用下面这句替代上一句

```bash
git clone https://github.com/BVLC/caffe.git
```

在bash输入以下语句安装caffe依赖包

```bash
sudo apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev
 
sudo apt-get install libhdf5-serial-dev protobuf-compiler
 
sudo apt-get install --no-install-recommends libboost-all-dev
 
sudo apt-get install libopenblas-dev liblapack-dev libatlas-base-dev
 
sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev
```

把Makefile.config.example复制得Makefile.config，修改成符合系统环境的情况。以如下环境为例

- Ubuntu 18.04 LTS
- RTX 2080 + Nvidia-Driver 430.40
- Cuda 10.0 + Cudnn v7.4.2
- Anaconda3

给出对应的Makefile.config示例

```makefile
## Refer to http://caffe.berkeleyvision.org/installation.html
# Contributions simplifying and improving our build system are welcome!

# cuDNN acceleration switch (uncomment to build with cuDNN).
USE_CUDNN := 1

# CPU-only switch (uncomment to build without GPU support).
# CPU_ONLY := 1

# uncomment to disable IO dependencies and corresponding data layers
# USE_OPENCV := 0
# USE_LEVELDB := 0
# USE_LMDB := 0
# This code is taken from https://github.com/sh1r0/caffe-android-lib
# USE_HDF5 := 0

# uncomment to allow MDB_NOLOCK when reading LMDB files (only if necessary)
#	You should not set this flag if you will be reading LMDBs with any
#	possibility of simultaneous read and write
# ALLOW_LMDB_NOLOCK := 1

# Uncomment if you're using OpenCV 3
OPENCV_VERSION := 3

# To customize your choice of compiler, uncomment and set the following.
# N.B. the default for Linux is g++ and the default for OSX is clang++
# CUSTOM_CXX := g++

# CUDA directory contains bin/ and lib/ directories that we need.
CUDA_DIR := /usr/local/cuda
# On Ubuntu 14.04, if cuda tools are installed via
# "sudo apt-get install nvidia-cuda-toolkit" then use this instead:
# CUDA_DIR := /usr

# CUDA architecture setting: going with all of them.
# For CUDA < 6.0, comment the *_50 through *_61 lines for compatibility.
# For CUDA < 8.0, comment the *_60 and *_61 lines for compatibility.
# For CUDA >= 9.0, comment the *_20 and *_21 lines for compatibility.
CUDA_ARCH := # -gencode arch=compute_20,code=sm_20 \
		# -gencode arch=compute_20,code=sm_21 \
		-gencode arch=compute_30,code=sm_30 \
		-gencode arch=compute_35,code=sm_35 \
		-gencode arch=compute_50,code=sm_50 \
		-gencode arch=compute_52,code=sm_52 \
		-gencode arch=compute_60,code=sm_60 \
		-gencode arch=compute_61,code=sm_61 \
		-gencode arch=compute_61,code=compute_61

# BLAS choice:
# atlas for ATLAS (default)
# mkl for MKL
# open for OpenBlas
# BLAS := atlas
BLAS := open
# Custom (MKL/ATLAS/OpenBLAS) include and lib directories.
# Leave commented to accept the defaults for your choice of BLAS
# (which should work)!
# BLAS_INCLUDE := /path/to/your/blas
# BLAS_LIB := /path/to/your/blas

# Homebrew puts openblas in a directory that is not on the standard search path
# BLAS_INCLUDE := $(shell brew --prefix openblas)/include
# BLAS_LIB := $(shell brew --prefix openblas)/lib

# This is required only if you will compile the matlab interface.
# MATLAB directory should contain the mex binary in /bin.
# MATLAB_DIR := /usr/local
# MATLAB_DIR := /Applications/MATLAB_R2012b.app

# NOTE: this is required only if you will compile the python interface.
# We need to be able to find Python.h and numpy/arrayobject.h.
# PYTHON_INCLUDE := /usr/include/python2.7 \
#		/usr/lib/python2.7/dist-packages/numpy/core/include
# Anaconda Python distribution is quite popular. Include path:
# Verify anaconda location, sometimes it's in root.
ANACONDA_HOME := $(HOME)/anaconda3
PYTHON_INCLUDE := $(ANACONDA_HOME)/include \
		 $(ANACONDA_HOME)/include/python3.7m \
		 $(ANACONDA_HOME)/lib/python3.7/site-packages/numpy/core/include

# Uncomment to use Python 3 (default is Python 2)
PYTHON_LIBRARIES := boost_python3 python3.6m
# PYTHON_INCLUDE := /usr/include/python3.5m \
#                 /usr/lib/python3.5/dist-packages/numpy/core/include

# We need to be able to find libpythonX.X.so or .dylib.
# PYTHON_LIB := /usr/lib
PYTHON_LIB := $(ANACONDA_HOME)/lib

# Homebrew installs numpy in a non standard path (keg only)
# PYTHON_INCLUDE += $(dir $(shell python -c 'import numpy.core; print(numpy.core.__file__)'))/include
# PYTHON_LIB += $(shell brew --prefix numpy)/lib

# Uncomment to support layers written in Python (will link against Python libs)
WITH_PYTHON_LAYER := 1

# Whatever else you find you need goes here.
INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include /usr/include/hdf5/serial
LIBRARY_DIRS := $(PYTHON_LIB) /usr/local/lib /usr/lib /usr/lib/x86_64-linux-gnu/hdf5/serial

# If Homebrew is installed at a non standard location (for example your home directory) and you use it for general dependencies
# INCLUDE_DIRS += $(shell brew --prefix)/include
# LIBRARY_DIRS += $(shell brew --prefix)/lib

# NCCL acceleration switch (uncomment to build with NCCL)
# https://github.com/NVIDIA/nccl (last tested version: v1.2.3-1+cuda8.0)
# USE_NCCL := 1

# Uncomment to use `pkg-config` to specify OpenCV library paths.
# (Usually not necessary -- OpenCV libraries are normally installed in one of the above $LIBRARY_DIRS.)
# USE_PKG_CONFIG := 1

# N.B. both build and distribute dirs are cleared on `make clean`
BUILD_DIR := build
DISTRIBUTE_DIR := distribute

# Uncomment for debugging. Does not work on OSX due to https://github.com/BVLC/caffe/issues/171
# DEBUG := 1

# The ID of the GPU that 'make runtest' will use to run unit tests.
TEST_GPUID := 0

# enable pretty build (comment to see full commands)
Q ?= @
```

此外还要还需将Makefile==(注意不是Makefile.config)==中的这一行

```makefile
NVCCFLAGS += -ccbin=$(CXX) -Xcompiler -fPIC $(COMMON_FLAGS)
```

替换为

```makefile
NVCCFLAGS += -D_FORCE_INLINES -ccbin=$(CXX) -Xcompiler -fPIC $(COMMON_FLAGS)
```

最后回到终端，执行以下语句获取caffe的python依赖库

```bash
cd /home/username/caffe/python
for req in $(cat requirements.txt); do pip install $req; done
```

以下语句目的是回到caffe文件夹下，执行完整个编译过程。

```bash
cd ..
make clean
make all -j8 # -j8表示调动CPU的8个线程，加快执行该指令的速度
make pycaffe -j8
make test -j8
make runtest -j8
make distribute
```



#### 分支I  PSPNet

下载Matlab R2015b_glnxa64.iso和Crack文件。

挂载ISO镜像文件

```bash
sudo mkdir /media/matlab
sudo mount -o -loop R2015b_glnxa64.iso /media/matlab
cd /media/matlab
sudo ./install
sudo cp /[Your crack directory]/Matlab_R2015b/Matlab_2015b_Linux64_Crack/R2015b/bin/glnxa64/* /usr/local/MATLAB/R2015b/bin/glnxa64
```

首次运行matlab要用root权限（否则无法写文件），采用不联网激活，找到Crack中相应的激活文件*.lic，导入激活。

```bash
cd /usr/local/MATLAB/R2015b/bin
sudo ./matlab
```

卸载ISO镜像

```bash
sudo umount /media/matlab
```

回到PSPNet主目录中，

```bash
cd ~/PSPNet
make matcaffe
make mattest
```



#### 分支II  py-faster-rcnn

安装依赖包

```bash
pip install cython
pip install easydict
sudo apt-get install python-opencv
```

克隆py-faster-rcnn项目

```bash
git clone --recursive git@github.com:rbgirshick/py-faster-rcnn.git
```

编译内部模块

```bash
cd faster-rcnn/lib
make
cd ../caffe-fast-rcnn
cp Makefile.config.example Makefile.config
```

将Makefile.config配置为符合当前系统环境，后

```bash
make -j8 && make pycaffe
```

此时一般会遇到与==cudnn==有关的问题，解决办法是将已编译的新版本caffe中以下三个部分复制覆盖到faster-rcnn自带的caffe中。

```bash
cp ~/caffe/src/caffe/layers/cudnn* ~/faster-rcnn/caffe-fast-rcnn/src/caffe/layers/
cp ~/caffe/include/caffe/layers/cudnn* ~/faster-rcnn/caffe-fast-rcnn/include/caffe/layers/
cp ~/caffe/include/caffe/util/cudnn* ~/faster-rcnn/caffe-fast-rcnn/include/caffe/util/
```

之后

```bash
make test -j8
make runtest -j8
make distribute
```

在第一步通常会遇到与==test_smooth_L1_loss_layer.cpp==有关的问题，解决方法为注释掉出问题的include行。



### II.  配置tensorflow



### III.  配置pytorch



