# Deploy simple-sinatra-app to Swarm

This project deploys a [simple-sinatra-app](https://github.com/fen9li/simple-sinatra-app.git) to a Swarm. 

## The Swarm
This demo Swarm has 3 nodes, 1 manager node acts as leader and 2 worker nodes. All nodes are CentOS 7 hosts and should be able to see each other.     

| hostname           | ip address          | role                |
| ------------------ | ------------------- | ------------------- |
| docker101.fen9.li  | 192.168.200.101/24  | Manager / Leader	 | 
| docker102.fen9.li  | 192.168.200.102/24  | Worker		 |	
| docker103.fen9.li  | 192.168.200.103/24  | Worker              |

Following procedure prepares docker101, repeats same process to prepare docker102 & docker103 accordingly.

### Configure hostname, networking and /etc/hosts file
```sh
hostnamectl set-hostname docker101.fen9.li
nmcli con add con-name ens33-fix type ethernet ifname ens33 autoconnect yes ip4 192.168.200.101/24 gw4 192.168.200.2
nmcli con mod ens33-fix ipv4.dns 192.168.200.2

echo '192.168.200.101 docker101.fen9.li	docker101' >> /etc/hosts
echo '192.168.200.102 docker102.fen9.li docker102' >> /etc/hosts
echo '192.168.200.103 docker103.fen9.li docker103' >> /etc/hosts
```

### Configure firewall
```sh
[root@docker101 ~]# vi /etc/firewalld/services/docker-swarm-manager.xml
[root@docker101 ~]# cat /etc/firewalld/services/docker-swarm-manager.xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>docker swarm manager</short>
  <description>Docker swarm manager </description>
  <port protocol="tcp" port="2377"/>
  <port protocol="tcp" port="7946"/>
  <port protocol="udp" port="7946"/>
  <port protocol="udp" port="4789"/>
</service>
[root@docker101 ~]# 

firewall-cmd --permanent --add-service={http,docker-swarm-manager}
firewall-cmd --reload

[root@docker101 ~]#
```

### Prepare docker
```sh
yum -y update
yum -y install wget tree git

wget https://download.docker.com/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl start docker
systemctl enable docker

[root@docker101 ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2017-10-14 22:16:49 AEDT; 1min 12s ago
     Docs: https://docs.docker.com
 Main PID: 1504 (dockerd)
   CGroup: /system.slice/docker.service
           ├─1504 /usr/bin/dockerd
           └─1510 docker-containerd -l unix:///var/run/docker/libcontainerd/d...

Oct 14 22:16:47 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:47.5..."
Oct 14 22:16:48 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:48.5..."
Oct 14 22:16:48 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:48.6..."
Oct 14 22:16:48 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:48.6..."
Oct 14 22:16:49 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:49.0..."
Oct 14 22:16:49 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:49.3..."
Oct 14 22:16:49 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:49.3...e
Oct 14 22:16:49 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:49.3..."
Oct 14 22:16:49 docker101.fen9.li dockerd[1504]: time="2017-10-14T22:16:49.3..."
Oct 14 22:16:49 docker101.fen9.li systemd[1]: Started Docker Application Con....
Hint: Some lines were ellipsized, use -l to show in full.
[root@docker101 ~]#

[root@docker101 ~]# docker --version
Docker version 17.09.0-ce, build afdb6d4
[root@docker101 ~]#
```

### Add user to docker group

```sh
usermod -aG docker fli
```
