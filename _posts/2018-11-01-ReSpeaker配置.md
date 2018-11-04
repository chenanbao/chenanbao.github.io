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

```shell

#删除之前配置
rm /home/pi/.asoundrc

#装驱动
git clone https://github.com/respeaker/seeed-voicecard
cd seeed-voicecard
sudo ./install.sh 
sudo reboot

#查看声卡
aplay -l

#调音
alsamixer

arecord -d 3 temp.wav  # 测试录音3秒
aplay temp.wav # 播放录音看看效果

#保存当前调音配置
sudo alsactl --file=asound.state store
sudo cp asound.state /var/lib/alsa/

sudo apt-get install sox  # 用于播放音乐
sudo apt-get install libsox-fmt-mp3 # 添加 sox 的 mp3 格式支持

```


### 配置叮当

```shell

#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

#sudo passwd root
#sudo passwd --unlock root
#su root

#高级选项开启 Expand Filesystem ，reroot -f生效
sudo raspi-config 

# 换更新源
cp /etc/apt/sources.list /etc/apt/sources.list.bak

sudo nano /etc/apt/sources.list
deb http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main non-free contrib
deb-src http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main non-free contrib

sudo nano /etc/apt/sources.list.d/raspi.list
deb http://mirror.tuna.tsinghua.edu.cn/raspberrypi/ stretch main ui
deb-src http://mirror.tuna.tsinghua.edu.cn/raspberrypi/ stretch main ui


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
cd ~/dingdang/
sudo pip --default-timeout=10000 install -r client/requirements.txt 
cd ..

# 安装 Sox，支持 mp3 格式的音频
sudo apt-get install -y sox libsox-fmt-mp3

# 安装 TaskWarrior,用于日程提醒。
#sudo apt-get install -y taskwarrior
cd $HOME
wget https://taskwarrior.org/download/task-2.5.1.tar.gz
tar xzvf task-2.5.1.tar.gz
cd task-2.5.1
cmake -DCMAKE_BUILD_TYPE=release . -DENABLE_SYNC=OFF
make
sudo make install

touch /home/pi/.taskrc


# 安装 Sphinxbase/Pocketsphinx
wget http://downloads.sourceforge.net/project/cmusphinx/sphinxbase/0.8/sphinxbase-0.8.tar.gz
tar -zxvf sphinxbase-0.8.tar.gz
cd sphinxbase-0.8/
./configure --enable-fixed
make
sudo make install
wget http://downloads.sourceforge.net/project/cmusphinx/pocketsphinx/0.8/pocketsphinx-0.8.tar.gz
tar -zxvf pocketsphinx-0.8.tar.gz
cd pocketsphinx-0.8/
./configure
make
sudo make install

#pocketsphinx_continuous  测试安装

# 安装 CMUCLMTK
sudo apt-get install subversion autoconf libtool automake gfortran g++ --yes
# 如果是打包下载的，可跳过此部分下载
svn co https://svn.code.sf.net/p/cmusphinx/code/trunk/cmuclmtk/
cd cmuclmtk/
./autogen.sh && make && sudo make install
cd ..

# download
wget http://distfiles.macports.org/openfst/openfst-1.4.1.tar.gz
wget https://github.com/mitlm/mitlm/releases/download/v0.4.1/mitlm_0.4.1.tar.gz
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/m2m-aligner/m2m-aligner-1.2.tar.gz
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/phonetisaurus/is2013-conversion.tgz

tar -xvf m2m-aligner-1.2.tar.gz
tar -xvf openfst-1.4.1.tar.gz
tar -xvf is2013-conversion.tgz
tar -xvf mitlm_0.4.1.tar.gz

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

# sound
alsamixer  #调节声音大小
amixer cset numid=3 1 #3.5mm
sudo aplay /usr/share/sounds/alsa/Front_Center.wav
omxplayer Downloads/Hotel\ California.mp3 
```

参考资料

[叮当](https://github.com/dingdang-robot/dingdang-robot/wiki/install)

[Amazon echo](https://www.hackster.io/idreams/build-your-own-amazon-echo-using-a-rpi-and-respeaker-hat-7f44a0)

[Respeaker](http://wiki.seeedstudio.com/ReSpeaker_2_Mics_Pi_HAT/)

[Back Img](http://conanwhf.github.io/2016/08/25/rpi-cloneimg/)
