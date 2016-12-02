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

## Kubernetes
