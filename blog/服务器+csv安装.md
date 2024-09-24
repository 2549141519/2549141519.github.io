点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# Ubuntu装服务器
### 下载 
去Ubuntu官网找到20.04的镜像文件，下载 iso文件(server版本 放入u盘)

### 安装
1. 使用新创建的可引导USB驱动器引导系统 按住F12键 选择英文
![!\[Alt text\](image.png)](../blogImg/服务器+csv1.png)
2. 键盘布局选择默认
![!\[Alt text\](image.png)](../blogImg/服务器+csv2.png)
3. 手动填入IP地址
![!\[Alt text\](image.png)](../blogImg/服务器+csv3.png)
4. 代理不填 默认 接下来都选择默认 到用户名阶段 创建用户密码
![!\[Alt text\](image.png)](../blogImg/服务器+csv4.png)

### 网络配置
装好之后 配置网络
```
$ sudo vi /etc/netplan/00-installer-config.yaml
// 编辑如下，把其他网卡删掉了
// 配置信息需要根据实际情况修改
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno3:
      dhcp4: no
      dhcp6: no
      addresses: [your.ip/24]
      gateway4: your.gateway
      nameservers:
        addresses: [your.dns1, your.dns2]
```

配置生效 ``sudo netplan apply``

### 替换镜像源 能apt-install
```
sudo cp /etc/apt/source.list /etc/apt/source.list.origin
sudo vi /etc/apt/source.list
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse

```

### 安装梯子
这个我就不写了 有需要可由我来装

### 安装环境
```
sudo apt-get update -y
sudo apt-get install -y policycoreutils-python-utils
sudo apt install policycoreutils selinux-utils selinux-basics
sudo apt-get install -y semanage-utils
```
装服务器应该默认装了ssh 现在就可以连接了
![!\[Alt text\](image.png)](../blogImg/服务器+csv5.png)

# csv安装
### 测试服务器
测试一下之前装的服务器成不成功
```
$ sudo chmod a+w /opt
$ cd /opt/
$ git clone https://gitee.com/anolis/hygon-devkit.git
$ mv hygon-devkit hygon
$ sudo cp /opt/hygon/bin/hag /usr/bin
```

### 更换内核
我们需要把虚拟机内核和服务器内核更换成适合csV架构的内核 先下载更换主机内核
```
$ sudo chmod a+w /opt
$ cd /opt/
$ git clone https://gitee.com/anolis/hygon-devkit.git
$ mv hygon-devkit hygon
$ sudo cp /opt/hygon/bin/hag /usr/bin
```
### 导入证书
```
sudo apt-get install -y ca-certificates
# 如果已经根据 1-安装操作系统和内核部分安装了hygon-devkit，可以在以下目录找到hag
$ ls -l /opt/hygon/bin/hag
# 如果没有安装hygon-devkit，可以使用以下命令下载
$ mkdir -p /opt/hygon/bin/
$ cd /opt/hygon/bin/
$ wget https://gitee.com/anolis/hygon-devkit/raw/master/bin/hag
$ chmod +x hag

$ cd /opt/hygon/bin
$ sudo ./hag general get_id
$ sudo ./hag general hgsc_version
```
以下命令确认整数是否导入成功
```
$ cd /opt/hygon/bin
$ sudo ./hag csv platform_status
```
可以看到是成功的
![!\[Alt text\](image.png)](../blogImg/服务器+csv6.png)

### 安装软件
1. 安装依赖 ``sudo /opt/hygon/csv/install_csv_sw.sh``
后续还会缺依赖 不知道为啥 可能服务器新装的把 不过缺什么 apt-install即可
2. 安装qemu，用于启动虚拟机
```
$ git clone https://gitee.com/anolis/hygon-qemu.git
$ cd hygon-qemu/
$ sudo /opt/hygon/csv/build_qemu.sh
```
3. 安装grub
```
$ wget https://ftp.gnu.org/gnu/grub/grub-2.06.tar.gz
$ tar -xvf grub-2.06.tar.gz
$ cd grub-2.06/
$ sudo /opt/hygon/csv/build_grub.sh
```

4. 安装edk2
```
$ git clone https://gitee.com/anolis/hygon-edk2.git
$ cd hygon-edk2/
$ git submodule update --init
$ sudo /opt/hygon/csv/build_edk2.sh
```
5. 安装devit
```
$ cd /opt/hygon/csv/
$ sudo ./make_vm_img.sh
$ git clone -b master https://github.com/guanzhi/GmSSL.git
$ sudo ./build_devkit.sh
```

6. 运行虚拟机 
```
 sudo qemu-system-x86_64 -name csv-vm --enable-kvm -cpu host -m 6114 -hda /opt/hygon/csv/vm.qcow2 -drive if=pflash,format=raw,unit=0,file=/opt/hygon/csv/OVMF_CODE.fd,readonly=on -qmp tcp:127.0.0.1:2222,server,nowait -vnc 0.0.0.0:0 -object sev-guest,id=sev0,policy=0x1,cbitpos=47,reduced-phys-bits=5 -machine memory-encryption=sev0 -netdev bridge,br=virbr0,id=net0 -device virtio-net-pci,netdev=net0,romfile= -nographic
```
这个命令需要改一下 我在虚拟机更新内核的时候磁盘空间不够了 这个虚拟机 他用的是用户模式 只能出不能进 所以需要用2222端口做ssh转发才能外部连接

我成功配置好了
![!\[Alt text\](image.png)](../blogImg/服务器+csv7.png)

7. 内核复制到虚拟机
```
root@ubuntu:~# scp ubuntu@10.10.11.115:/opt/hygon/kernel.tgz ./
ubuntu@10.10.11.115's password: 
kernel.tgz                                      100%  112   147.7KB/s   00:
```

8. 发现磁盘无法分区的话 先kill虚拟机进程 然后变大qemu的磁盘空间大小
```
sudo qemu-img resize /opt/hygon/csv/vm.qcow2 +20G

```

9. 成功安装

![!\[Alt text\](image.png)](../blogImg/服务器+csv8.png)
对了 虚拟机镜像也要替换 








