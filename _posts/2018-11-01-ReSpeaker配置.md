---
 layout:     post
 title:      语音识别配置
 subtitle:   ReSpeaker搭配叮当
 date:       2018-10-27
 author:     Bob
 header-img: img/post-bg-rwd.jpg
 catalog: true
 tags:
     - HardWare
---

### 配置ReSpeaker



### 配置叮当

```shell

#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

#高级选项开启 Expand Filesystem ，reroot -f生效
sudo raspi-config 

# 换更新源
cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo nano /etc/apt/sources.list
deb http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main non-free contrib
deb-src http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main non-free contrib
sudo apt-get update
sudo apt-get upgrade


# 安装若干工具：
sudo apt-get update
sudo apt-get install -y vim git-core python-dev bison libasound2-dev libportaudio-dev python-pyaudio libatlas-base-dev python-pymad cmake uuid-dev fswebcam libav-tools

# 安装叮当

# 把 dingdang-robot 项目拉取下来：
git clone https://github.com/wzpan/dingdang-robot.git dingdang
cd dingdang
#mkdir temp

# 如果需要修改分枝
# 查看所有分支
git branch -r

#使用的是 ReSpeaker 2-Mics Array HAT 作为麦克风阵列开发板
git checkout respeaker

# 之后安装必须的 pypi 库：
sudo apt-get install -y python-setuptools
sudo easy_install pip
mkdir ~/.pip
echo -e "[global]\nindex-url = http://mirrors.aliyun.com/pypi/simple/\n[install]\ntrusted-host = mirrors.aliyun.com" > ~/.pip/pip.conf

# 更新 pypi
sudo pip install --upgrade setuptools
sudo pip --default-timeout=10000 install -r client/requirements.txt 
cd ..

# 安装 Sox，支持 mp3 格式的音频
sudo apt-get install -y sox libsox-fmt-mp3

# 安装 TaskWarrior,用于日程提醒。
sudo apt-get install -y taskwarrior

# 安装 Sphinxbase/Pocketsphinx
# Stretch 已经包含了 PocketSphinx 的源，可以先装预编译的版本：
sudo apt-get install -y pocketsphinx

# 预编译的版本没有包含 Python 的接口，所以还得拉源码构建一次。
# 安装 sphinxbase
cd sphinxbase-0.8/
./configure --enable-fixed
make
sudo make install
cd ..

# 安装 pocketsphinx
cd pocketsphinx-0.8/
./configure
make
sudo make install
cd ..

# 安装 CMUCLMTK
sudo apt-get install subversion autoconf libtool automake gfortran g++ --yes
# 如果是打包下载的，可跳过此部分下载
svn co https://svn.code.sf.net/p/cmusphinx/code/trunk/cmuclmtk/
cd cmuclmtk/
./autogen.sh && make && sudo make install
cd ..

# 编译安装 OpenFST：
cd openfst-1.4.1/
sudo ./configure --enable-compact-fsts --enable-const-fsts --enable-far --enable-lookahead-fsts --enable-pdt
sudo make install
cd ..

# 编译安装 M2M：
cd m2m-aligner-1.2/
sudo make
sudo cp m2m-aligner /usr/local/bin/m2m-aligner
cd ..

# 编译安装 MITLMT：
cd mitlm-0.4.1/
sudo ./configure
sudo make install
cd ..

# 编译安装 Phonetisaurus：
cd is2013-conversion/phonetisaurus/src
sudo make
sudo cp ../../bin/phonetisaurus-g2p /usr/local/bin/phonetisaurus-g2p
cd ../../../

# 词汇模型
#  g014b2b.zip
#  vocabularies.zip

# 第三方插件安装
mkdir ~/.dingdang
cd ~/.dingdang
git clone http://github.com/dingdang-robot/dingdang-contrib contrib
sudo pip --default-timeout=10000 install -r contrib/requirements.txt

```

参考资料

[叮当](https://github.com/dingdang-robot/dingdang-robot/wiki/install)

[Amazon echo](https://www.hackster.io/idreams/build-your-own-amazon-echo-using-a-rpi-and-respeaker-hat-7f44a0)

[Respeaker](http://wiki.seeedstudio.com/ReSpeaker_2_Mics_Pi_HAT/)

[Back Img](http://conanwhf.github.io/2016/08/25/rpi-cloneimg/)
