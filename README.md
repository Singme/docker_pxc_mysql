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
** 注意 **    
由于同时创建多个数据卷，和连续运行pxc多个镜像会导致容器运行闪退，所以这边采用每次创建一个数据卷，就映射到容器内，然后运行镜像，pxc容器运行间隔2分钟以上后才能创建下一个。      
> docker volume create --name v1    

可以通过``` docker volume ls ```查看数据卷，``` docker volume inspect v1 ```查看数据卷信息，``` docker volume rm v1 ```删除数据卷。       

6.运行第一个pxc镜像     
```     
docker run -d -p 3355:3306 --name=node1 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -e CLUSTER_NAME=cluster1 -e XTRABACKUP_PASSWORD=123456 --privileged -net=pxc_net --ip 172.19.0.2 pxc 
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
** 5.因为停止了node1节点容器，再重启发现无法启动该容器了。**      

#### 节点意外宕机，数据恢复   
后续完善    
