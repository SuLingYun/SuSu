# Centos7.9升级内核版本【三种方法】

## 方法一、YUM在线升级

### 1.查看内核版本

```sh
uname -r
```

### 2.导入ELRepo软件仓库的公共秘钥

```sh
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

### 3.安装ELRepo软件仓库的yum源

```sh
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
```

### 4.查看有哪些内核版本可供安装

```sh
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```

### 5.开始安装

```sh
yum --enablerepo=elrepo-kernel install kernel-ml -y  //安装的是主线版本，该版本比较激进，慎重选择。

yum --enablerepo=elrepo-kernel install kernel-lt -y  //安装的长期稳定版本，稳定可靠。
```

### 6.设置 GRUB 默认的内核版本

```sh
查看系统启动方式属于哪种：
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS

BIOS方式启动：
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg  //查看系统可用内核
grub2-set-default 0 　　//初始化页面的第一个内核将作为默认内核
grub2-mkconfig -o /boot/grub2/grub.cfg　　//重新创建内核配置

UEFI方式启动：
cat /boot/efi/EFI/centos/grub.cfg |grep "menuentry 'CentOS Linux"  //查看系统可用内核
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg  //重新创建内核配置
grub2-set-default 'CentOS Linux (5.17.4-1.el7.elrepo.x86_64) 7 (Core)'  //配置默认内核
```

### 7.删除旧内核

```sh
rpm -qa |grep kernel  //查看系统安装的内核
yum -y remove   //根据需要删除对应的内核
```

<br/>

## 方法二、离线下载内核rpm包升级

```sh
https://elrepo.org/linux/kernel/el7/x86_64/RPMS/  //下载需要的内核
rpm -ivh kernel-lt-5.4.95-1.el7.elrepo.x86_64.rpm kernel-lt-devel-5.4.95-1.el7.elrepo.x86_64.rpm  //安装内核包
参照方法一设置默认内核
```

<br/>

## 方法三、编译安装、并制作RPM包

### 1.kernel官网下载内核

```sh
https://www.kernel.org/
longterm:	5.15.35	2022-04-20	[tarball]  //一般下载这个长期稳定版
```

### 2.上传并解压内核文件

```sh
tar -Jxf linux-5.15.35.tar.xz  //想要解压 tar.xz 文件，使用tar -xf命令，加上压缩包名字即可。
```

### 3.安装依赖包

```sh
yum -y install elfutils-libelf-devel
yum -y install openssl-devel
yum -y groupinstall "development tools"  //安装开发工具包组
yum -y install ncurses-devel  //安装ncurse-devel包 (make menuconfig 文本界面窗口依赖包)
```

### 4.由于编译要求gcc版本必须大于5.1.0，所以需要升级gcc版本

```sh
安装centos-release-scl
yum install centos-release-scl -y

安装devtoolset，注意，如果想安装7.版本的，就改成devtoolset-7-gcc，以此类推
yum install devtoolset-8-gcc* -y

激活对应的devtoolset，所以你可以一次安装多个版本的devtoolset，需要的时候用下面这条命令切换到对应的版本
scl enable devtoolset-8 bash

查看gcc版本
gcc -v
```

补充：这条激活命令只对本次会话有效，重启会话后还是会变回原来的4.8.5版本，要想随意切换可按如下操作。

首先，安装的devtoolset是在 /opt/rh 目录下的。

每个版本的目录下面都有个 enable 文件，如果需要启用某个版本，只需要执行

```sh
source ./enable
```

所以要想切换到某个版本，只需要执行

```sh
source /opt/rh/devtoolset-8/enable
```

直接替换旧的gcc

旧的gcc是运行的 /usr/bin/gcc，所以将该目录下的gcc/g++替换为刚安装的新版本gcc软连接，免得每次enable

```sh
mv /usr/bin/gcc /usr/bin/gcc-4.8.5

ln -s /opt/rh/devtoolset-8/root/bin/gcc /usr/bin/gcc

mv /usr/bin/g++ /usr/bin/g++-4.8.5

ln -s /opt/rh/devtoolset-8/root/bin/g++ /usr/bin/g++

gcc --version

g++ --version
```

### 5.从 /boot 目录将现有版本的内核编译config配置文件拷过来到放到新的内核源码解压目录内并重命名为.config的隐藏文件(这个文件保存了在安装系统时内核所安装的模块配置信，否则需要重新手动指定每一个模块的编译配置)

```sh
cd linux-5.15.35/
cp /boot/config-$(uname -r) ./.config
或者使用
cp /boot/config-3.10.0-1160.el7.x86_64 ./.config
```

### 6.运行 make menuconfig，开启文本界面的编译选项菜单窗口，可以对内核加载的模块编译选项进行调整，如修改编译后的内核名称、新添加之前系统缺少的模块等

```sh
cd linux-5.15.35/
make menuconfig
General setup --->local version -append to kernel release  //修改内核名称_zhiming.su@qq.com
File systems --->DOS/FAT/NT Filesystems --->NTFS file system support  //新添加NTFS文件系统支持模块
```

### 7.编译内核，PS：时间较长，根据硬件决定，一般1个小时

```sh
make -j `cat /proc/cpuinfo | grep 'model name'|wc -l`
或者
make -j $(nproc)
-j参数指定编译时使用CPU核心数
```

### 8.编译安装模块

```sh
编译完成后执行make modules_install 安装内核模块
make modules_install
```

### 9.安装内核核心文件

```sh
make install
```

### 10.制作成内核RPM文件

```sh
yum -y install rpmdevtools  //安装RPM打包工具

cd linux-5.15.35/

make rpm-pkg  //同时构建源和二进制RPM软件包【一般用这个】

或者

make binrpm-pkg  //仅构建二进制RPM软件包
```

### 11.根据方法一调整默认内核文件

### 12.打包的RPM文件移植到其它服务器安装

```sh
上传以下3个RPM文件到新服务器
rpm -ivh kernel-4.19.239_SuLingYun-1.x86_64.rpm  //安装顺序1
rpm -ivh kernel-devel-4.19.239_SuLingYun-1.x86_64.rpm  //安装顺序2
rpm -ivh kernel-headers-4.19.239_SuLingYun-1.x86_64.rpm  //安装顺序3

查看系统可用内核
cat /boot/efi/EFI/centos/grub.cfg |grep "menuentry 'CentOS Linux"  //UEFI模式
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg    //BOIS模式

配置默认内核
grub2-set-default 'CentOS Linux (4.19.239_SuLingYun) 7 (Core)'  //UEFI模式
grub2-set-default 0  //BOIS模式 　　
```