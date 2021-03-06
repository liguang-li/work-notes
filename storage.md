* SAN: Storage area network 存储区域网络
* NAS: Network attached storage 网络附加存储

NAS不一定是盘阵，一台普通的主机就可以做NAS,只要它自己有磁盘和文件系统，而且对外提供访问其文件系统的接口（如NFS,CIFS等），它就是一台NAS。

NFS(Network File System): Linux/Unix

Cifs(Common internet file system): Windows

* SAN是一个网络上的磁盘

* NAS是一个网络上的文件系统

根据SAN的定义，可知SAN其实是指一个网络，但是这个网络里包含着各种各样的元素，主机、适配器、网络交换机、磁盘阵列前端、盘阵后端、磁盘等。

NAS必须具备的物理条件有两条，第一，NAS必须可以访问卷或者物理磁盘；第二，NAS必须具有接入以太网的能力，也就是必须具有以太网卡。

![image-20191119164136070](/home/wrsadmin/.config/Typora/typora-user-images/image-20191119164136070.png)

![image-20191119164155307](/home/wrsadmin/.config/Typora/typora-user-images/image-20191119164155307.png)

NAS架构的路径在虚拟目录层和文件系统层通信的时候，用以太网和TCP/IP协议代替了内存，这样做不但增加了大量的CPU指令周期（TCP/IP逻辑和以太网卡驱动程序），而且使用了低速传输介质（内存速度要比以太网快得多）

SAN方式下，路径中比NAS方式多了一次FC访问过程，但是FC的逻辑大部分都由适配卡上的硬件完成，增加不了多少CPU的开销，而且FC访问的速度比以太网高，所以我们很容易得出结论，如果后端磁盘没有瓶颈，那么除非NAS使用快于内存的网络方式与主机通信，否则其速度永远无法超越SAN架构

如果后端磁盘有瓶颈，那么NAS用网络代替内存的方法产生的性能降低就可以忽略。比如，在大量随记小块I/O、缓存命中率极低的环境下，后端磁盘系统寻到瓶颈达到最大，此时前端的I/O指令都会处于等待状态，所以就算路径首段速度再快，也无济于事。此时，NAS系统不但不比SAN慢，而且由于其优化的并发I/O设计和基于文件访问而不是簇块访问的特性，反而可能比SAN性能高。 

* NAS的成本比SAN低很多。前端只使用以太网接口即可，FC适配卡以及交换机的成本相对以太网卡和交换机来说非常高的。
* NAS可以解决主机服务器上的CPU和内存资源。NAS适用于cpu密集的应用环境。
* NAS由于利用了以太网，所以可扩展性很强，且容易部署。
* NAS设备一般都提供多种协议访问数据，而SAN只能使用SCSI协议访问。
* NAS可以在一台盘阵上实现多台客户端的共享访问，包括同时访问某个目录或文件。而SAN方式下，除非所有的客户端都安装了专门的集群管理软件，否则不能将某个lun共享，强制共享会损坏数据。
* 经过特别优化的NAS系统，可以同时并发处理大量客户端的请求，提供比SAN方式更方便的访问方法。
* 多台主机可以同时挂接NFS上的目录，那么相当于减少了整个系统中文件系统的处理流程，由原来的多个并行处理转化成了NFS上的单一实例，简化了系统冗余度。

