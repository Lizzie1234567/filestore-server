通过docker可以快速部署小规模Ceph集群的流程，可用于开发测试。
以下的安装流程是通过linux shell来执行的；假设你只有一台机器，装了linux(如Ubuntu)系统和docker环境，那么可以参考以下步骤安装Ceph:




## docker部署ceph集群 原创
2023最新版！！！！！！
https://blog.51cto.com/niuben/6367453?articleABtest=1

只用看这个链接的内容就可以啦

```bash
以下是链接内容：
1. 创建Ceph专用网络
sudo docker network create --driver bridge --subnet 172.20.0.0/16 ceph-network

2. 拉取搭建用镜像
sudo docker pull ceph/daemon:latest-luminous

3. 搭建mon节点
sudo docker run -d --name ceph-mon --network ceph-network --ip 172.20.0.10 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_IP=172.20.0.10 -e MON_NAME=ceph-mon -e CEPH_PUBLIC_NETWORK=172.20.0.0/16 -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/log/ceph/:/var/log/ceph/ ceph/daemon:latest-luminous mon
4. 搭建osd节点
sudo docker exec ceph-mon ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring

修改配置文件以兼容etx4硬盘

sudo vi /etc/ceph/ceph.conf

在文件最后添加：

osd max object name len = 256
osd max object namespace len = 64

分别启动三个容器来模拟集群

sudo docker run -d --privileged=true --name ceph-osd-1 --network ceph-network --ip 172.20.0.11 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=ceph-mon -e MON_IP=172.20.0.10 -e OSD_TYPE=directory -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/lib/ceph/osd/1:/var/lib/ceph/osd -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-luminous osd
sudo docker run -d --privileged=true --name ceph-osd-2 --network ceph-network --ip 172.20.0.12 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=ceph-mon -e MON_IP=172.20.0.10 -e OSD_TYPE=directory -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/lib/ceph/osd/2:/var/lib/ceph/osd -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-luminous osd
sudo docker run -d --privileged=true --name ceph-osd-3 --network ceph-network --ip 172.20.0.13 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=ceph-mon -e MON_IP=172.20.0.10 -e OSD_TYPE=directory -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/lib/ceph/osd/3:/var/lib/ceph/osd -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-luminous osd


5. 搭建mgr节点
sudo docker run -d --privileged=true --name ceph-mgr --network ceph-network --ip 172.20.0.14 -e CLUSTER=ceph -p 7000:7000 --pid=container:ceph-mon -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ ceph/daemon:latest-luminous mgr

开启管理界面，访问地址 http://127.0.0.1:7000/

sudo docker exec ceph-mgr ceph mgr module enable dashboard

6. 搭建rgw节点
登录后复制 
sudo docker exec ceph-mon ceph auth get client.bootstrap-rgw -o /var/lib/ceph/bootstrap-rgw/ceph.keyring
sudo docker run -d --privileged=true --name ceph-rgw --network ceph-network --ip 172.20.0.15 -e CLUSTER=ceph -e RGW_NAME=ceph-rgw -p 7480:7480 -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-luminous rgw

7.检查Ceph状态
sudo docker exec ceph-mon ceph -s

8. 测试添加rgw用户
sudo docker exec ceph-rgw radosgw-admin user create --uid="test" --display-name="test user"


## 下面是2019版本，已经过期了，不用参考



# 要用root用户创建, 或有sudo权限
# 注: 建议使用这个docker镜像源:https://registry.docker-cn.com
# 1. 修改docker镜像源
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
EOF
# 重启docker
systemctl restart docker
# 2. 创建Ceph专用网络
docker network create --driver bridge --subnet 172.20.0.0/16 ceph-network
docker network inspect ceph-network
# 3. 删除旧的ceph相关容器
docker rm -f $(docker ps -a | grep ceph | awk '{print $1}')
# 4. 清理旧的ceph相关目录文件，加入有的话
rm -rf /www/ceph /var/lib/ceph/  /www/osd/
# 5. 创建相关目录及修改权限，用于挂载volume
mkdir -p /www/ceph /var/lib/ceph/osd /www/osd/
chown -R 64045:64045 /var/lib/ceph/osd/
chown -R 64045:64045 /www/osd/
# 6. 创建monitor节点
docker run -itd --name monnode --network ceph-network --ip 172.20.0.10 -e MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph ceph/mon
# 7. 在monitor节点上标识3个osd节点
docker exec monnode ceph osd create
docker exec monnode ceph osd create
docker exec monnode ceph osd create
# 8. 创建OSD节点
docker run -itd --name osdnode0 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd/0:/var/lib/ceph/osd/ceph-0 ceph/osd 
docker run -itd --name osdnode1 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd/1:/var/lib/ceph/osd/ceph-1 ceph/osd
docker run -itd --name osdnode2 --network ceph-network -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=monnode -e MON_IP=172.20.0.10 -v /www/ceph:/etc/ceph -v /www/osd/2:/var/lib/ceph/osd/ceph-2 ceph/osd
# 9. 增加monitor节点，组件成集群
docker run -itd --name monnode_1 --network ceph-network --ip 172.20.0.11 -e MON_NAME=monnode_1 -e MON_IP=172.20.0.11 -v /www/ceph:/etc/ceph ceph/mon
docker run -itd --name monnode_2 --network ceph-network --ip 172.20.0.12 -e MON_NAME=monnode_2 -e MON_IP=172.20.0.12 -v /www/ceph:/etc/ceph ceph/mon
# 10. 创建gateway节点
docker run -itd --name gwnode --network ceph-network --ip 172.20.0.9 -p 9080:80 -e RGW_NAME=gwnode -v /www/ceph:/etc/ceph ceph/radosgw
# 11. 查看ceph集群状态
sleep 10 && docker exec monnode ceph -s
# 12. 创建用户
docker exec -it gwnode radosgw-admin user create --uid=user1 --display-name=user1
```
