# Cloudarchitecture Test

## Building your image
You'll need to build your Docker images and upload it to a docker repository. You cannot build the images on the server ("cloud").
Let's take the server-cloud app. The container will be ready for production so we need to build all necessary things before the actual image.
```bash
cd server-cloud
npm run build # this will build the project
docker build -t haargeeel/node-server:v0.0.1 . # and this the container
docker push haargeeel/node-server:v0.0.1
```

This can be tested then on your machine:
```bash
# we run this in ./cloudarchitecture
# first we start our db
docker run -d -p 27017:27017 --name mongo1 --network test -v $PWD/mongo-cloud/mongo1:/data/db mvertes/alpine-mongo

# now the app
docker run -d -p 3000:3000 --name app --network test haargeeel/node-server:v0.0.1
```
If the container wasn't on your machine already, it would be downloaded automatically.

## Local with docker-compose

## Docker Swarm

### Without proxy

Swarm is natively supported in Docker. For a good example we need let's say three servers. Either with virtual machines locally or just three "real" servers.
In this example one server has to be the swarm manager. All the commands for deploying or scaling would go there. The other two will be swarm worker.
On the soon to be swarm manager, we initialize the swarm
```bash
docker swarm init --advertise-addr <MANAGER-IP>
```
From the swarm-tutorial the expected result:
```bash
$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
After executing the `join` command we got here, we have a swarm with 3 servers.
We need the app to communicate with the db. Therefor we put them in the same network.
```bash
docker network create --driver overlay test
```
All processes in the network `test` can communicate via hostnames, which are the same as the name we provide in the next step.
```bash
# first the db
docker service create --name mongo1 --network test --mount type=bind,source=/data/db,target=/data/db mvertes/alpine-mongo
# then we check docker service ls until the mongo process is running
docker service ls

ID            NAME    REPLICAS  IMAGE                         COMMAND
a1v2w92iqgjo  mongo1  1/1       mvertes/alpine-mongo

# 1/1 replicas are deployed so we can deploy the app
docker service create -p 3000:3000 --name app --network test haargeeel/node-server:v0.0.2

docker service ls
ID            NAME    REPLICAS  IMAGE                         COMMAND
a1v2w92iqgjo  mongo1  1/1       mvertes/alpine-mongo
e6as45w157t7  app     1/1       haargeeel/node-server:v0.0.2
```
We can check on any of the servers if the process is running there.
```bash
# on one of the worker machines
docker ps
CONTAINER ID        IMAGE                         COMMAND                 CREATED             STATUS              PORTS                  NAMES
c5d1c9f294f5        mvertes/alpine-mongo:latest   "/root/run.sh mongod"   4 minutes ago       Up 4 minutes        27017/tcp, 28017/tcp   mongo1.1.cijsmt2oin128hh6fxn8ts3hy
```

A small taste of the orchestration features would be scaling the app.
```bash
docker service scale app=5
app scaled to 5

# let's check the services
docker service ls
ID            NAME    REPLICAS  IMAGE                         COMMAND
a1v2w92iqgjo  mongo1  1/1       mvertes/alpine-mongo
e6as45w157t7  app     4/5       haargeeel/node-server:v0.0.2
```

Now we could visit any of the 3 servers on the port 3000 to see the test app.

### with proxy
If we use a proxy/load balancer like nginx in front of the app we don't need to open the ports of the app. But we add it to another network, which is shared with the proxy service.
```bash
docker network create --driver overlay proxy
docker service create --name app \
--network test \
--network proxy \
haargeeel/node-server:v0.0.2

docker service create --name proxy \
-p 80:80 -p 443:443 \
--mount type=bind,source={letsencrypt .well-known position},target={letsencrypt .well-known position} \
--mount type=bind,source=/etc/letsencrypt,target=/etc/letsencrypt \
--network proxy \
haargeeel/nginx:v0.0.4
```
In this case we use our own nginx container. The only special thing about is its own `nginx.conf` and `sites-available` folder. This container is prepared to use a secure connections already.

### Setting up a docker registry
For self hosting all our docker images we can use our own docker registry. Therefor we need a domain, tls certificate and a server.
```bash
# some preparation for the secure connection to the server
cd /etc/letsencrypt/live/domain.example.com/
cp privkey.pem domain.key
cat cert.pem chain.pem > domain.crt
chmod 777 domain.crt
chmod 777 domain.key

# for authentication we make an auth file
mkdir auth
docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > auth/htpasswd

docker run -d -p 3003:5000 --restart=always --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v /etc/letsencrypt/live/domain.example.com:/certs \
  -v /opt/docker-registry:/var/lib/registry \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2


# pushing to the registry
docker login domain:3003 # testuser - testpassword
docker tag {image} domain:3003/myimage
docker push domain:3003/myimage
```

## Kubernetes

### From scratch on Digital Ocean Servers

On all the servers we need:
- Etcd
- Flannel
- Docker
- Kubernetes

One of the servers has to be the master. It must be able to connect to the other
servers via ssh.
```bash
# creating ssh key
ssh-keygen -t rsa

# ...
# adding it to the authorized_keys on the other servers
```

We should add all the servers to their `/etc/host` files

Now we install etcd. It needs the ports
- 2379
- 2380
to be open.

For etcd we need golang:
```bash
sudo apt-get install build-essential bison git
# install go version manager
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
# avoid restarting session
source /root/.gvm/scripts/gvm
# install go
gvm install go1.7 -B
# create kubernetes binary folder
mkdir -p /opt/kubernetes && cd /opt/kubernetes

gvm use go1.7
export GOPATH=`pwd`
```

Now we can download etcd:
```bash
ETCD_VER=v3.0.15
DOWNLOAD_URL=https://github.com/coreos/etcd/releases/download
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
mkdir bin
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz -C bin --strip-components=1
```

Next is initialising etcd. We need to create a `/etc/systemd/system/etcd.service` file:
```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
[Service]
Type=notify
Environment=ETCD_DATA_DIR=/var/lib/etcd
ExecStart=/opt/kubernetes/bin/etcd -name {SERVER_NAME} \
  -advertise-client-urls=http://{SERVER_IP}:2379 \
  -listen-client-urls=http://{SERVER_IP}:2379 \
  -listen-peer-urls=http://{SERVER_IP}:2380 \
  -initial-advertise-peer-urls=http://{SERVER_IP}:2380 \
  -initial-cluster={SERVER_NAME_1}=http://{SERVER_IP_1}:2380,{SERVER_NAME_2}=http://{SERVER_IP_2}:2380,{SERVER_NAME_3}=http://{SERVER_IP_3}:2380 \
  -initial-cluster-state=new \
  -initial-cluster-token=catsonthemoon
Restart=always
RestartSec=10s
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
```

We do this on each server. In this example we have 3 servers.

```bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

After doing this on all servers we can do a healthy check:
```bash
$GOPATH/bin/etcdctl -endpoints=http://{SERVER_IP}:2379 cluster-health
```

Next we look at flannel.
```bash
FLANNEL_VERSION=v0.6.2
curl -L https://github.com/coreos/flannel/releases/download/${FLANNEL_VERSION}/flannel-${FLANNEL_VERSION}-linux-amd64.tar.gz -o flannel-${FLANNEL_VERSION}-linux-amd64.tar.gz
tar zxvf flannel-${FLANNEL_VERSION}-linux-amd64.tar.gz -C bin
# create flannel directory for config
mkdir -p /opt/kubernetes/flannel
```
_config.json_
```
{
  "Network": "172.17.0.0/16",
  "SubnetLen": 24,
    "Backend": {
    "Type": "vxlan",
    "VNI": 1,
    "Port": 8472
  }
}
```
Use the config file in the cluster:
```bash
$GOPATH/bin/etcdctl -endpoints=http://{SERVER_IP}:2379/ set {CLUSTER_NAME}/network/config < config.json
```
For flannel we also add a service file
_/etc/systemd/system/flannel.service_:
```
[Unit]
Description=flanneld
After=etcd.service
[Install]
WantedBy=multi-user.target
[Service]
ExecStart=/opt/kubernetes/bin/flanneld \
 -public-ip={PUBLIC_IP} \
 -ip-masq=true \
 -etcd-endpoints=http://{SERVER_IP_1}:2379,http://{SERVER_IP_2}:2379,http://{SERVER_IP_3}:2379 \
 -etcd-prefix="{CLUSTER_NAME}/network"
Restart=always
RestartSec=10s
LimitNOFILE=65535
```

```bash
systemctl daemon-reload
systemctl start flannel
systemctl enable flannel
```
With _ifconfig_ we can check the ip address and try to access it from another server.

Docker

We change 
_/etc/systemd/system/docker.service_:
```
[Unit]
Description=Docker with flannel
Requires=flannel.service
After=flannel.service
[Service]
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/docker daemon --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU} -s overlay
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
Delegate=yes
MountFlags=slave
[Install]
WantedBy=multi-user.target
```

### Using Google Cloud

lala
