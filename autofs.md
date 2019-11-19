# autofs

将挂载信息写入到/etc/fstab文件中，可实现开机自动挂载。如果远程共享资源过多，则会给网络带宽和服务器的硬件资源带来很大负载。如果挂载的资源长期不使用，也会造成服务器资源的浪费。

* 当客户端在有使用NFS文件系统的需求时才让系统自动挂载。
* 当NFS文件系统使用完毕后，让NFS自动卸载。 

autofs自动挂载服务是一种Linux系统守护进程，当检测到用户视图访问一个尚未挂载的文件系统时，会自动挂载该文件系统。

如果把挂载信息都写入到autofs服务的主配置文件中，会使主配置文件臃肿不堪，不利于管理和维护。因此在autofs的主配置文件中按照“挂载目录的上层目录 子配置文件”的格式填写，具体的挂载信息写入到子配置文件中，方便日后管理和维护。

1. 在主配置文件里添加如下内容

   ~~~shell
   vim /etc/auto.master
   /media /etc/cdrom.misc
   ~~~

2. 子配置文件中添加如下内容

子配置文件按照**“挂载目录 挂载文件类型及权限 :设备名称”**的格式进行填写

~~~shell
vim /etc/cdrom.misc
cdrom -fstype=iso9660,ro,nosuid,nodev :/dev/cdrom
~~~

挂载目录为/media/cdrom，-fstype=ios9660表示以光盘格式挂载，ro、nosuid及nodev是挂载使用的权限，/dev/cdrom是挂载的设备名称。

## 挂载NFS网络文件系统

~~~shell
vim /etc/samba.misc
nfsdata -fstype=nfs 192.168.2.211:/nfsdata
~~~

