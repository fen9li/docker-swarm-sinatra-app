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

### Deploy helloworld stack

```sh
[fli@docker101 docker-swarm-sinatra-app]$ pwd
/home/fli/test/docker-swarm-sinatra-app
[fli@docker101 docker-swarm-sinatra-app]$

[fli@docker101 docker-swarm-sinatra-app]$ tree
.
├── docker-cloud.yml
├── Dockerfile
├── LICENSE
├── README.md
└── simple-sinatra-app
    ├── config.ru
    ├── Dockerfile
    ├── Gemfile
    ├── Gemfile.lock
    └── helloworld.rb

1 directory, 9 files
[fli@docker101 docker-swarm-sinatra-app]$

[fli@docker101 docker-swarm-sinatra-app]$ docker stack deploy --compose-file docker-cloud.yml helloworld
Creating network helloworld_default
Creating service helloworld_web
[fli@docker101 docker-swarm-sinatra-app]$

[fli@docker101 docker-swarm-sinatra-app]$ docker stack ls
NAME                SERVICES
helloworld          1
[fli@docker101 docker-swarm-sinatra-app]$ docker stack ps helloworld
ID                  NAME                IMAGE                              NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
m8hqphw6lpth        helloworld_web.1    fen9li/simple-sinatra-app:latest   docker103.fen9.li   Running             Running 13 seconds ago
7abku4lx47l1        helloworld_web.2    fen9li/simple-sinatra-app:latest   docker101.fen9.li   Running             Running 13 seconds ago
[fli@docker101 docker-swarm-sinatra-app]$
```

### Test  

> Test from docker101

```sh
[fli@docker101 ~]$ curl http://docker101.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$ curl http://docker102.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$ curl http://docker103.fen9.li
Hello World![fli@docker101 ~]$
[fli@docker101 ~]$
```

> Test from docker102

```sh
[fli@docker102 ~]$ curl http://docker101.fen9.li
Hello World![fli@docker102 ~]$
[fli@docker102 ~]$ curl http://docker102.fen9.li
Hello World![fli@docker102 ~]$
[fli@docker102 ~]$ curl http://docker103.fen9.li
Hello World![fli@docker102 ~]$
[fli@docker102 ~]$
```

> Test from docker103

```sh
[fli@docker103 ~]$ curl http://docker101.fen9.li
Hello World![fli@docker103 ~]$
[fli@docker103 ~]$ curl http://docker102.fen9.li
Hello World![fli@docker103 ~]$
[fli@docker103 ~]$ curl http://docker103.fen9.li
Hello World![fli@docker103 ~]$
[fli@docker103 ~]$
```

### Remove helloworld Stack

```sh
[fli@docker101 docker-swarm-sinatra-app]$ docker stack rm helloworld
Removing service helloworld_web
Removing network helloworld_default
[fli@docker101 docker-swarm-sinatra-app]$ docker stack ls
NAME                SERVICES
[fli@docker101 docker-swarm-sinatra-app]$
```
