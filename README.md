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

[fli@docker101 ~]$ docker --version
Docker version 17.09.0-ce, build afdb6d4
[fli@docker101 ~]$
```

### Add user to docker group
Add user name to docker group, logout, login ...
```sh
[root@docker101 ~]# usermod -aG docker fli
... ...

Using username "fli".
fli@192.168.200.101's password:
Last login: Sat Oct 14 22:11:59 2017 from 192.168.200.1
[fli@docker101 ~]$ id
uid=1000(fli) gid=1000(fli) groups=1000(fli),995(docker) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[fli@docker101 ~]$
```

## Create simple-sinatra-app docker image 

Base image: [ruby:2.4-onbuild](https://hub.docker.com/_/ruby/) 

### Prepare 
```sh
[fli@docker101 ~]$ mkdir test
[fli@docker101 ~]$ cd test/
[fli@docker101 test]$ git clone https://github.com/fen9li/docker-swarm-sinatra-app.git
[fli@docker101 test]$ cd docker-swarm-sinatra-app/
[fli@docker101 docker-swarm-sinatra-app]$
[fli@docker101 docker-swarm-sinatra-app]$ git clone -b develop https://github.com/fen9li/simple-sinatra-app.git
Cloning into 'simple-sinatra-app'...
remote: Counting objects: 8, done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 8 (delta 1), reused 8 (delta 1), pack-reused 0
Unpacking objects: 100% (8/8), done.
[fli@docker101 docker-swarm-sinatra-app]$

[fli@docker101 docker-swarm-sinatra-app]$ \cp Dockerfile simple-sinatra-app/
[fli@docker101 docker-swarm-sinatra-app]$ cd simple-sinatra-app/
[fli@docker101 simple-sinatra-app]$ tree
.
├── config.ru
├── Dockerfile
├── Gemfile
└── helloworld.rb

0 directories, 4 files
[fli@docker101 simple-sinatra-app]$
```

### Generate Gemfile.lock, build simple-sinatra-app image and push to docker hub
```sh
[fli@docker101 simple-sinatra-app]$ docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.4 bundle install
... ...
[fli@docker101 simple-sinatra-app]$ 

[fli@docker101 simple-sinatra-app]$ tree
.
├── config.ru
├── Dockerfile
├── Gemfile
├── Gemfile.lock
└── helloworld.rb

0 directories, 5 files
[fli@docker101 simple-sinatra-app]$

[fli@docker101 simple-sinatra-app]$ docker build -t fen9li/simple-sinatra-app .
... ...
[fli@docker101 simple-sinatra-app]$ 
[fli@docker101 simple-sinatra-app]$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
fen9li/simple-sinatra-app   latest              fbd175a49e26        8 seconds ago       702MB
ruby                        2.4-onbuild         4e688c69ff1b        3 days ago          684MB
ruby                        2.4                 eebb1381c2aa        3 days ago          684MB
[fli@docker101 simple-sinatra-app]$

[fli@docker101 simple-sinatra-app]$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: fen9li
Password:
Login Succeeded
[fli@docker101 simple-sinatra-app]$ docker push fen9li/simple-sinatra-app
The push refers to a repository [docker.io/fen9li/simple-sinatra-app]
... ...
[fli@docker101 simple-sinatra-app]$ 
```

## Setup Swarm

### Initial Swarm on docker101 
```sh
[fli@docker101 ~]$ docker swarm init --advertise-addr $(hostname -i)
Swarm initialized: current node (yclttfs3fqiggcef2y5es6pik) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2xx9qgtocre0zp5gddtmea9gi7zbcnoizk6a5kh0gdegr2hmaq-a2beod2aozb0vw2bxqp949ym9 192.168.200.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

[fli@docker101 ~]$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
yclttfs3fqiggcef2y5es6pik *   docker101.fen9.li   Ready               Active              Leader
[fli@docker101 ~]$
```

### Join docker102 & docker103 to Swarm

```sh
[fli@docker102 ~]$ docker swarm join --token SWMTKN-1-2xx9qgtocre0zp5gddtmea9gi7zbcnoizk6a5kh0gdegr2hmaq-a2beod2aozb0vw2bxqp949ym9 192.168.200.101:2377
This node joined a swarm as a worker.
[fli@docker102 ~]$

... ...

[fli@docker103 ~]$ docker swarm join --token SWMTKN-1-2xx9qgtocre0zp5gddtmea9gi7zbcnoizk6a5kh0gdegr2hmaq-a2beod2aozb0vw2bxqp949ym9 192.168.200.101:2377
This node joined a swarm as a worker.
[fli@docker103 ~]$

```

### Double check on docker101
```sh
[fli@docker101 ~]$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
yclttfs3fqiggcef2y5es6pik *   docker101.fen9.li   Ready               Active              Leader
59p1wfpfkimlmtczqioprlz8n     docker102.fen9.li   Ready               Active
okvzrtn4educaqj0186pj4pwm     docker103.fen9.li   Ready               Active
[fli@docker101 ~]$

```

## Deploy simple-sinatra-app service to Swarm

### Deploy as per below specification

> Service name: simple-sinatra-app

> Expose binding port: 80

> Replicas: 2

> Detach: true

> Image: fen9li/simple-sinatra-app

```sh
[fli@docker101 ~]$ docker service create --name simple-sinatra-app --publish 80:4567 --replicas 2 --detach=true fen9li/simple-sinatra-app
z0gbpcq3fedpifjebzcddoodp
[fli@docker101 ~]$

[fli@docker101 ~]$ docker service ls
ID                  NAME                 MODE                REPLICAS            IMAGE                              PORTS
z0gbpcq3fedp        simple-sinatra-app   replicated          1/2                 fen9li/simple-sinatra-app:latest   *:80->4567/tcp
[fli@docker101 ~]$ docker service ps simple-sinatra-app
ID                  NAME                   IMAGE                              NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
9tw7qwpqfoko        simple-sinatra-app.1   fen9li/simple-sinatra-app:latest   docker102.fen9.li   Running             Preparing 2 minutes ago
88ordjfio2y1        simple-sinatra-app.2   fen9li/simple-sinatra-app:latest   docker101.fen9.li   Running             Running 2 minutes ago
[fli@docker101 ~]$
```

> Note: service instance on docker102's CURRENT STATE is 'Preparing ...'. It needs time to download image from docker hub. Keep watching its status until its CURRENT STATE is 'Running ...'.

```sh
[fli@docker101 ~]$ docker service ps simple-sinatra-app
ID                  NAME                   IMAGE                              NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
9tw7qwpqfoko        simple-sinatra-app.1   fen9li/simple-sinatra-app:latest   docker102.fen9.li   Running             Running 3 minutes ago
88ordjfio2y1        simple-sinatra-app.2   fen9li/simple-sinatra-app:latest   docker101.fen9.li   Running             Running 8 minutes ago
[fli@docker101 ~]$

```  

### Test by Using curl or web browser 

```sh
[fli@docker101 ~]$ curl http://docker101.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$ curl http://docker102.fen9.li
Hello World![fli@docker101 ~]$

```

### Scale up to 3 instances

> Same as what happened to docker102, it takes time to download image from docker hub. Keep watching till its CURRENT STATE is 'Running ...'

```sh
[fli@docker101 ~]$ docker service scale simple-sinatra-app=3
simple-sinatra-app scaled to 3
Since --detach=false was not specified, tasks will be scaled in the background.
In a future release, --detach=false will become the default.
[fli@docker101 ~]$ docker service ps simple-sinatra-app
ID                  NAME                   IMAGE                              NODE                DESIRED STATE       CURRENT STATE                  ERROR               PORTS
9tw7qwpqfoko        simple-sinatra-app.1   fen9li/simple-sinatra-app:latest   docker102.fen9.li   Running             Running 9 minutes ago
88ordjfio2y1        simple-sinatra-app.2   fen9li/simple-sinatra-app:latest   docker101.fen9.li   Running             Running 15 minutes ago
y8kn8g814re6        simple-sinatra-app.3   fen9li/simple-sinatra-app:latest   docker103.fen9.li   Running             Preparing about a minute ago
[fli@docker101 ~]$

```

> Service are scaled up to 3 instances successfully.

```sh
[fli@docker101 ~]$ docker service ps simple-sinatra-app
ID                  NAME                   IMAGE                              NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
9tw7qwpqfoko        simple-sinatra-app.1   fen9li/simple-sinatra-app:latest   docker102.fen9.li   Running             Running 16 minutes ago
88ordjfio2y1        simple-sinatra-app.2   fen9li/simple-sinatra-app:latest   docker101.fen9.li   Running             Running 22 minutes ago
y8kn8g814re6        simple-sinatra-app.3   fen9li/simple-sinatra-app:latest   docker103.fen9.li   Running             Running about a minute ago
[fli@docker101 ~]$
```

### Test again

```sh
[fli@docker101 ~]$ curl http://docker101.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$ curl http://docker102.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$ curl http://docker103.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$

```
