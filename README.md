# docker配置踩坑记录

## 1. 方法一：partner.io

在https://hub.docker.com/r/nvidia/cuda上找到所需要的镜像，我选择的tag是`10.0-cudnn7-devel-ubuntu18.04`。然后打开http://192.168.1.51:9000/#/containers，在左边栏中选择Containers，再点击+Add container。接下来进行container的配置

<center>
<img border:0; display:inline; src="1.png" width=70%/>
</center>

Name随意填写，Image在前面选的tag前加上nvidia/cuda:，如我的Image应填`nvidia/cuda:10.0-cudnn7-devel-ubuntu18.04`。在Manual network port publishing处点击`+publish a new network port`，并在host中填写`10022`或`10024`或`10026`……(若前一个端口已被占用则依次递推)，映射到container的`22`端口。Entry Point填写`/bin/bash`，Console选择 `Interactive & TTY (-i -t)`。

点选Volumes，在其中添加映射，以下为例:

<center>
<img border:0; display:inline; src="4.png" width=70%/>
</center>

在Runtime & Resources中将Runtime选为nvidia，最后点击Deploy the container即可。

最后启动container，打开它的console。



## 1. 方法二：命令行创建docker

```bash
docker run -it --name xxx -p 10022:22 -v /media/cita/mydata:/media/mydata:rw --runtime=nvidia --shm-size="4g" pytorch/pytorch:1.4-cuda10.1-cudnn7-devel /bin/bash
```



## 2.  更换源、安装并配置SSH

在console中由于是root权限，直接运行以下语句：

```bash
apt-get update
# apt-get upgrade
apt-get install vim
cp /etc/apt/sources.list /etc/apt/sources.list.bak
vim /etc/apt/sources.list
```

删除文件中的所有内容，替换为

```bash
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```

接着更新源，安装SSH服务器端

```bash
apt-get update
# apt-get install openssh-client
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
# /etc/init.d/ssh start
service ssh start
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

~~打开profile文件~~

``` bash
# sudo vim ~/.bashrc
```

~~在文件末尾处添加~~

```bash
# export PATH=/home/username/anaconda3/bin:$PATH
```

在终端中执行

```bash
source ~/.bashrc
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
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATH
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



==接下来是分支配置或问题解决方案(非必须)==

### A.  SSH连接github(可能加快git clone速度)

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
ssh-agent bash
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



### D.  matplotlib等出图问题

#### Solution 1  安装Xming

#### Solution 2  配置display环境变量

该方法主要仅适合使用Mobaxterm作为SSH软件的用户。

具体方法如下：

<center>
<img border:0; display:inline; src="3.png" width=90%/>
</center>

注意右上角红框，将鼠标停在按钮上方可以看到出现Current DISPLAY={Your IPv4 address}:X.Y

故做以下操作，在bash中执行

```bash
sudo vim ~/.bashrc
```

在末尾加上

```bash
export DISPLAY={Your IPv4 address}:X.Y
```

然后使DISPLAY环境变量生效

```bash
source ~/.bashrc
```

这时即可正常出图了。

==注意:==

1)

运行py文件时还可能提示`UserWarning: Matplotlib is currently using agg, which is a non-GUI backend, so cannot show the figure`，此时的解决方法是

安装tkinter

```bash
sudo apt-get install python3-tk
```

然后在py文件中`import matplotlib.pyplot as plt`的前面加上

```python
import matplotlib
matplotlib.use('tkagg')
```



### E.  tensorflow报错问题

若运行时报错提示

`Could not create cudnn handle: CUDNN_STATUS_INTERNAL_ERROR`

则改变文件以下部分

```python
config = tf.ConfigProto(gpu_options=tf.GPUOptions(allow_growth=True))
sess = tf.Session(config = config)
```



### F. vim永久显示行号

```bash
vim ~/.vimrc
```

在打开的文件中最后一行输入

```bash
set number
```

以后用vim打开文件都会显示行号了。
