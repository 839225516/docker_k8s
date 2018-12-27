### CentOS7配置支持AUFS文件系统
CentOS7 默认不支持aufs文件系统, 要使用, 就必须自己去安装内核

1. 添加yum源
```shell
# 进入repo目录
cd /etc/yum.repos.d/
wget https://yum.spaceduck.org/kernel-ml-aufs/kernel-ml-aufs.repo
# 安装
yum install kernel-ml-aufs -y
```

2. 修改内核启动
``` shell
vim /etc/default/grub
# 修改参数, 表示启动时选择第一个内核
###################################
GRUB_DEFAULT=0
###################################


# 重新生成grub.cfg
grub2-mkconfig -o /boot/grub2/grub.cfg


# 重启
reboot
```

    GRUB_DEFAULT=saved   saved表示下次启动时默认启动上次的内核
    GRUB_DEFAULT=0    0表示启动时选择第一个内核

3. 查看是否支持
```shell
cat /proc/filesystems
```
