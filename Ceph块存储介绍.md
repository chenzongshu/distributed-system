
## 一、 前置知识


块设备：

块设备将信息存储在固定大小的块中，每个块都能进行编址。块设备的基本特征是每个块都能区别于其它块而读写。块设备也是底层设备的抽象，块设备上未建立文件系统时，也称之为裸设备。


块设备与ceph的联系：

client想要把数据存储到ceph的集群中时，他必须要有一个读写的目标，能够在本地知道这个目标。这里讲的是块存储，当然这个读写的目标要是一个块设备才行，需要将这个块设备与ceph关联起来，这个块设备通常成为rbd设备。

rbd的全称应该是Rados Block Device。rbd是由ceph进行整理物理资源并且向外提供的RAODS形式的块设备，这样的块设备在客户端同其他类型的块设备使用方法相同


OSD是如何存储数据：

OSD其实是建立在文件系统之上的，当你使用一个块设备进行部署OSD节点时，部署工具会默认格式化osd为xfs，当然你也可以预先格式为想要的文件系统（ext4等）。数据到了OSD层次时，这时可以把这个请求变成一个文件的操作，最后交给了xfs文件系统，最终组织到了磁盘上。


二、rbd到osd的映射：

1、客户端的使用rbd设备，

在客户端使用rbd设备时，一般有两种方法。


第一种 是kernel rbd。就是创建了rbd设备后，把rbd设备map到内核中，形成一个虚拟的块设备，这时这个块设备同其他通用块设备一样，一般的设备文件为/dev/rbd0，后续直接使用这个块设备文件就可以了，可以把/dev/rbd0格式化后mount到某个目录，也可以直接作为裸设备使用。这时对rbd设备的操作都通过kernel rbd操作方法进行的。


第二种是librbd方式。就是创建了rbd设备后，这时可以使用librbd、librados库进行访问管理块设备。这种方式不会map到内核，直接调用librbd提供的接口，可以实现对rbd设备的访问和管理，但是不会在客户端产生块设备文件。


2、Ceph数据映射：


180430_9Sz8_2460844.png

客户想要创建一个rbd设备前，必须创建 一个pool，需要为这个pool指定pg的数量，在一个pool中的pg数量是不一定的，同时这个pool中要指明保存数据的副本数量3个副本。再在这个pool中创建一个rbd设备rbd0，那么这个rbd0都会保存三份，在创建rbd0时必须指定rbd的size，对于这个rbd0的任何操作不能超过这个size。之后会将这个块设备进行切块，每个块的大小默认为4M，并且每个块都有一个名字，名字就是object+序号。将每个object通过pg进行副本位置的分配(pg map 到osd的过程会在下一节讲述)，pg会寻找3个osd，把这个object分别保存在这三个osd上。osd上实际是把底层的disk进行了格式化操作，一般部署工具会将它格式化为xfs文件系统。最后对于object的存储就变成了存储一个文件rbd0.object1.file。


3、客户端写数据到OSD：

假设这次采用的是librbd的形式，使用librbd创建一个块设备，这时向这个块设备中写入数据，在客户端本地同过调用librados接口，然后经过pool，rbd，object、pg进行层层映射,在PG这一层中，可以知道数据保存在哪3个OSD上，这3个OSD分为主从的关系，也就是一个primary OSD，两个replica OSD。客户端与primay OSD建立SOCKET 通信，将要写入的数据传给primary OSD，由primary OSD再将数据发送给其他replica OSD数据节点。

总结剖析这个写数据过程，第一部分客户端处理对rbd读写的请求，经过librbd与librados库可知道数据保存在哪些OSD上，客户端与primary OSD建立通信，传输请求，再由primary OSD 发送给其他replica OSD。

180525_dkRe_2460844.png


三、PG选择OSD的过程

pg选择osd过程也就是要选择保存副本的空间，每个osd只能保存一个副本。也就是一个object保存到一个pg中，由pg找到3个osd，每个osd保存一份object。

查看crush.map:

ceph osd getcrushmap -o crush.map
crushtool -d crush.map >> crush.txt

pg 到OSD的映射的过程算法叫做crush 算法，这个算法是一个伪随机的过程，他可以从所有的OSD中，随机性选择一个OSD集合，但是同一个PG每次随机选择的结果是不变的，也就是映射的OSD集合是固定的。

1 Crush因子

OSDMap管理当前ceph中所有的OSD，OSDMap规定了crush算法的一个范围，在这个范围中选择OSD结合。那么影响crush算法结果的有两种因素，一个就是OSDMap的结构，另外一个就是crush rule。

OSDMap其实就是一个树形的结构，叶子节点是device（也就是osd），其他的节点称为bucket节点，这些bucket都是虚构的节点，可以根据物理结构进行抽象，当然树形结构只有一个最终的根节点称之为root节点，中间虚拟的bucket节点可以是数据中心抽象、机房抽象、机架抽象、主机抽象等如图。
