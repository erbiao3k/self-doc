当我们从MongoDB中删除文档或集合时，MongoDB并不会将已经占用了的磁盘空间释放，它会一直维护已经占用了磁盘空间的数据文件，尽管数据文件中可能存在大大小小的空记录列表（empty record list）。当客户端程序再次插入文档时，MongoDB会从空记录列表中分配存储空间给新文档。那么为了更加有效的使用磁盘空间，我们需要对mongodb的数据文件做碎片整理以及未使用空间的回收。思想无非两种：
  1、对原数据进行重组
  2、仅将数据复制出来，形成仅数据的完整备份

### 以下介绍几种常用的实施方法：

	1、compact
	2、db.repairDatabase()
	3、secondary节点重同步
	2、db.copyDatabase()


### 一、[compat](https://docs.mongodb.com/v3.4/reference/command/compact/index.html)
官网对该命令的定义：对集合中的所有数据和索引进行重写和碎片整理。
###### 使用方法
```
use yourdatabase;
db.runCommand({ compact : 'yourCollection' });
```
###### 注意事项
```
1、在执行命令前请保证你有比较新的备份
2、在使用MMAPv1存储引擎的MongoDB上compact需要数据文件所在分区至少有2G的空闲空间
3、在使用WiredTiger存储引擎的MongoDB上，compact命令将重写集合和索引，且释放未使用的空间，但使用MMAPv1存储引擎的MongoDB上，该命令只对集合的数据文件进行碎片整理并重新创建其索引。不会释放空间，在使用MMAPv1存储引擎的MongoDB上回收空间，建议使用第三种方法“secondary节点重同步”
4、使用MMAPv1存储引擎的MongoDB中的Capped Collections，是无法被压缩的，但使用WiredTiger存储引擎的MongoDB在执行compact时会进行压缩。
5、在副本集上运行该命令时，要分别在每个节点执行
6、该命令只能在mongod实例上执行，不能再mongos实例上运行。也就是说针对分片集群的compact操作要分别在每个分片节点上执行。
7、一般该命令运行在secondary节点上，在执行时，会强制节点进入RECOVERING状态，RECOVERING状态的实例读写操作将被阻塞
8、再碰到特殊情况要停止运行该命令时，可通过db.currentOp()查询进程信息，然后通过db.killOp()干掉进程
9、compact可能会增加数据文件的总大小和数量，尤其是第一次运行时。但这不会增加总集合使用的磁盘空间，因为存储大小是数据库文件中分配的数据量，而不是文件系统上文件的大小/数量
10、使用MMAPv1存储引擎的MongoDB中的Capped Collections，是无法被压缩的，但使用WiredTiger存储引擎的MongoDB在执行compact时会进行压缩。
```
### 二、[db.repairDatabase()](https://docs.mongodb.com/v3.4/reference/command/repairDatabase/)
官网该命令的定义：通过丢无效或损坏的数据老重建数据库和索引。类似于文件系统修复命令fsck。所以此命令主要是用于修复数据。
###### 使用方法
```
use yourdatabase;
db.repairDatabase();
```
###### 注意事项
```
1、db.repairDatabase()主要用于修复数据。若你拥有数据的完整副本，且有权限访问，请使用第三种方法“secondary节点重同步”
2、在执行命令前请保证你有比较新的备份
3、此命令会完全阻塞数据库的读写，谨慎操作
4、此命令执行需要数据文件所在位置有等同于所有数据文件大小总和的空闲空间再加上2G
5、在使用MMAPv1存储引擎的secondary节点上执行该命令可以压缩集合数据
6、在使用WiredTiger存储引擎的MongoDB库上执行不会有压缩的效果
7、再碰到特殊情况要停止运行该命令时，可通过db.currentOp()查询进程信息，然后通过db.killOp()干掉进程
8、非常消耗时间
```
### 三、[secondary节点重同步](https://docs.mongodb.com/manual/tutorial/resync-replica-set-member/)
主要思想就是：删除secondary节点中指定数据，使之与primary重新开始数据同步。当副本集成员数据太过陈旧，也可以使用重新同步。数据的重新同步与直接复制数据文件不同，MongoDB会只同步数据，因此重同步完成后的数据文件是没有空集合的，以此实现了磁盘空间的回收。
###### 使用方法
  首先必须确保数据有完整的备份。

	1、若是primary节点，先强制将之变为secondary节点，否则跳过此步骤：
		rs.stepdown(120)；
	2、然后在primary上删除secondary节点：
		rs.remove("IP:port");
	3、删除secondary节点dbpath下的所有文件。
	4、将节点重新加入集群，然后使之自动进行数据的同步：
		rs.add("IP:port");
	5、等数据同步完成后，循环1-4的步骤可以将集群中所有节点的磁盘空间释放

	针对一些特殊情况，不能下线secondary节点的，可以新增一个节点到副本集中，然后secondary就自动开始数据的同步了。
	总的来说，重同步的方法是比较好的，第一基本不会阻塞副本集的读写，第二消耗的时间相对前两种比较短

### 四、[db.copyDatabase()](https://docs.mongodb.com/manual/reference/method/db.copyDatabase/)
	mongodb还支持在线复制数据：db.copyDatabase("from","to","IP:port"),此种方法也能释放空间，因为db.copyDatabase复制的数据，而不是表示在磁盘中的数据文件。但，该命令在4.0版本起被弃用；3.x版本还能继续使用
	如：
		db.copyDatabase("sourceDB","DistDB");
		将源库sourceDB。拷贝为DistDB。

	当然，该命令支持远程复制。
	该命令的完整语法为：
	db.copyDatabase(<源数据库名称>, <目标数据库名称>, <源mongodb的IP：port>, <源数据库连接需要的账户>,<密码>, <mechanism>)

	以上：命令必须在目标数据库服务器上执行。若源数据库与目标数据库存在于一个MongoDB服务器，<源mongodb的IP：port>, <源数据库连接需要的账户>,<密码>都可省略。<mechanism>是身份验证类型，可选的。

###### 注意事项
```
1、db.copyDatabase()不会阻塞源数据库和目标数据库数据的读写，因此可能会出现两份数据不一致的情况
2、db.copyDatabase()复制索引数据会锁定数据库，此操作也会对其他数据库产生影响
3、db.copyDatabase()不要在mongos实例中使用
4、db.copyDatabase()不要用于复制包含分片集合的数据库
5、在4.0版中更改：db.copyDatabase()仅支持SCRAM进行身份验证fromhost，<mechanism>选项。
6、某些不同版本的MongoDB间不支持此种复制方法，详见链接：https://docs.mongodb.com/manual/reference/method/db.copyDatabase/
```
除此之外，还有一些方法，像使用导入/导出的方法（mongodump/mongorestore），这种方法在数据量非常大的情况是不适用的，因为导入导出的方法使用的全量的形式，要保证有足够的空闲空间来存放导入的数据。
