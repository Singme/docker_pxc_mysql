### Docker搭建MySQL的PXC集群  
#### PXC  
PXC，全称是Percona Xtradb Cluster,是基于Galera插件的MySQL集群。galera cluster关注的是数据的一致性，对待事物的行为时，要么在所有节点上执行，要么都不执行，这就保证了MySQL集群的数据一致性。数据同步是双向的，在PXC里任何一个节点都是可读可写的。      
Replication集群是常用MySQL集群的另一种方案，数据同步时单向的，master负责写，然后异步复制给slave，如果slave写入数据，不会复制给master。这种异步复制方式，主从数据无法保证一致性。     
#### docker创建mysql集群  
1.docker的官方仓库下载pxc的镜像    
> docker pull percona/percona-xtradb-cluster    

2.pxc的镜像名太长，重新给镜像改名   
> docker tag percona/percona-xtradb-cluster pxc    

3.``` docker images ``` 查看镜像，可以看到多了pxc镜像，但还是有原来percona/percona-xtradb-cluster的镜像，删除多余镜像   
> docker rmi percona/percona-xtradb-cluster     

4.处于安全，需要给pxc集群实例创建docker内部网络，这边创建内部网段并分配具体ip。还可以通过 ``` docker network ls ```查看有哪些创建好的网络，然后通过``` docker network inspect 网络名 ```查看网络信息，也可以通过``` docker network rm 网络名 ```删除不需要的网络。           
> docker network create --subnet 172.19.0.0/16 pxc_net      

5.创建docker数据卷，一旦生成docker容器，不要在容器内保存业务的数据，要把数据放到宿主机上，可以把宿主机的一个目录映射到容器内，如果容器出现问题，只需要把容器删除，重现建立一个新的容器把目录映射给新的容器。      
**注意**    
由于同时创建多个数据卷，和连续运行pxc多个镜像会导致容器运行闪退，所以这边采用每次创建一个数据卷，就映射到容器内，然后运行镜像，pxc容器运行间隔2分钟以上后才能创建下一个。      
> docker volume create --name v1    

可以通过``` docker volume ls ```查看数据卷，``` docker volume inspect v1 ```查看数据卷信息，``` docker volume rm v1 ```删除数据卷。       

6.运行第一个pxc镜像     
```     
docker run -d -p 3355:3306 --name=node1 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -e CLUSTER_NAME=cluster1 -e XTRABACKUP_PASSWORD=123456 --privileged --net=pxc_net --ip 172.19.0.2 pxc 
```     
命令说明：     
-d: 代表创建的容器在后台运行     
-p: 端口映射 宿主机端口:容器端口   
--name=node1 节点名称node1     
-v: 路径映射   
-e MYSQL_ROOT_PASSWORD=123456 指定mysql的root账号密码   
-e CLUSTER_NAME=cluster1 执行名称为cluster1  
-e XTRABACKUP_PASSWORD=123456 指定mysql数据同步时用的密码    
--privileged 给最高的权限    
--net=pxc_net  使用的内部网段    
--ip 172.19.0.2 分发的ip地址    
pxc 镜像名称pxc     

运行第一个容器2分钟后，通过``` docker ps ```或是```docker logs node1 ```查看node1容器是否创建成功，之后创建数据卷v2，再运行node2容器，跟node1运行的命令多了CLUSTER_JOIN=node1 指定与节点node1进行数据同步。     
```   
docker run -d -p 3356:3306 --name=node2 -v:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -e CLUSTAR_NAME=cluster1 -e XTRABACKUP_PASSWORD=123456 -e CLUSTAR_JOIN=node1 --privileged --net=pxc_net --ip 172.19.0.3 pxc
```   

#### 测试数据同步     
1.进入各节点的数据库容器    
```    
docker exec -it node1 /bin/bash
mysql -uroot -p123456   
```   
在节点node1创建数据库node1_test      
2.进入node2节点数据库容器，查看是否有node1_test数据库     
3.可以在node3节点数据库，创建node3_test数据库，然后分别到各节点数据库查看是否数据同步      
4.``` docker stop node1 ```停止node1节点容器，然后再各节点测试创建数据库，检查是否会数据同步，通过验证关闭node1并不影响其他节点数据的同步。     
**5.因为停止了node1节点容器，再重启发现无法启动该容器了。**      
6.再停止node2节点容器，也是无法重新启动容器。      

#### 节点意外宕机，数据恢复   
因为集群节点异常宕机会记录一个同步信息点，异常节点全挂了，节点间各自记录的信息和集群内部信息都不统一导致启动不成功。     
**解决方式**       
1.直接通过``` docker start node1 ```或者任何一个节点是启动不了的，原因是集群之前的同步机制造成的，启动任何一个节点，该节点都会去其他节点同步数据，其它节点仍处于宕机状态，所以该节点启动失败，这也是pxc集群的强一致性的表现，解决方式是：      
删除所有节点 ``` docker rm node1 node2 node3 node4 ``` 和数据卷中的grastate.dat文件    
```     
rm -rf /var/lib/docker/volumes/v1/_data/grastate.dat
rm -rf /var/lib/docker/volumes/v2/_data/grastate.dat
```    
之后重新执行集群创建的命令即可，因为数据都在数据卷中，集群重新启动后数据仍然都在。     
**注意**    
1.node3创建数据库test_node3，然后停止node3容器   
2.node1创建数据库test_node1，然后停止node1容器     
3.node2创建数据库test_node2，然后停止node2,node4容器       
4.通过删除grastate.dat文件方式，重新创建集群，查看数据    
进入所有节点容器数据库查看数据都只有 test_node1,test_node3，因为强一致性，在CLUSTER_JOIN=node1,指定节点数据同步节点node1宕机，所以后续的数据没有同步上，数据丢失了。      
**mac进入和退出LinuxKit**   
由于Mac版docker是创建了虚拟环境LinuxKit,所以首先进入docker的LinuxKit终端     
```    
cd ~/Library/Containers/com.docker.docker/Data/vms/0/
screen tty  # 命令登录终端，遇到空白命令行，直接回车即可
退出终端是 先按 Ctrl+A 然后按 Ctrl+K 
```    
#### 集群热备份和恢复       
由于上面提到的方式，数据会丢失，所以采用xtrabackup方案备份。       
1.依据上面注意点，继续操作，先进入node4容器,以root用户进入系统    
```       
docker exec -it --user root node4 /bin/bash
# 先确定系统是否有 apt-get 或者 yum
yum update  # 因为这边有yum，所以更新一下
yum install percona-xtrabackup  # 一般percona-xtrabackup已经在更新列表中，没有就选择安装   
# 备份数据
innobackupex --user=root --password=123456 /data/back/full
```       
2.进行全量恢复      
**注意：pxc集群数据库可以热备份，但是不能热还原。为了避免恢复过程中的数据同步，所以采用空白的MySQL还原数据，然后再建立pxc集群。**       
先停止所有容器，然后删除容器node1,node2...，删除存储卷v1,v2...   
然后先只创建node1容器          
``` docker exec -it --user root node1 /bin/basn ```进入node1容器，安装 percona-xtrabackup      
```    
# 未提交事务回滚
innobackupex --user=root --password=1234 --apply-back /data/backup/full/2019-11-19_06_05_57/
# 执行冷还原
innobackupex --user=root --password=1234 --copy-back /data/backup/full/2019-11-19_06_05_57/

# 退出容器，重启node1节点 
docker stop node1
docker start node1
```   
3.进入node1节点容器数据库，查看数据，然后重新建立其他节点。



