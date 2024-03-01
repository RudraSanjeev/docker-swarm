#### Docker-swarm:

- install three instance of ubuntu (at least 3 required - 1 for master, 2-worker)
- add the script to preinstalled docker in

```bash
#!/bin/bash

# Update and remove existing Docker-related packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
    sudo apt-get remove -y $pkg
done

# Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add the current user to the docker group
sudo usermod -aG docker $USER


```

- make sure icmp is enable in security group to ping one another
- install docker in it
- check whether docker swarm active or not

```bash
docker info | head -50
```

- In the master instance run this cmd
- init docker swarm

```bash
docker swarm init
```

- then paste swarm join with token in the both worker instance
- insure port 2377 is accepting in inbound rule of instance

```bash
docker swarm join --token <MANAGER_TOKEN>
```

- you will get messages like
  `This node joined a swarm as a worker.`

#### How to get joining token {worker | manager}:

- may be next day you need the token to add more worker in the cluster for that you need token for that

```bash
docker swarm join-token worker
```

- Now your cluster is bigger you need to add master for that

```bash
docker swarm join-token manager
```

- to see the cluster

```bash
docker node ls
```

#### to leave cluster:

- to remove any worker
- go to that worker and run

```bash
docker swarm leave
```

**To remove from the manager instance**:

```bash
# if worker already leave the cluster
docker node rm <worker_name> | <worker_id>

# if not then forcefully
docker node rm -f <worker_name> | <worker_id>

```

Note:

- both manager and worker don't know whether master remove it or worker leave
- need to run cmd above in both to update

#### How to promote worker or demote manager:

- to inspect any worker

```bash
docker node inspect worker1
```

**To promote any worker**

```bash
docker node promote woker1 worker2
```

**To demote any worker**

```bash
docker node demote woker1 worker2
```

#### How to run service on manager:

- this cmd refresh output at every 2 min
- run this cmd to both workers

```bash
watch docker contianer ls
```

#### service cmd:

```bash
# create
docker service create -d alpine ping <master_ip>

# inspect
docker service inspect <container_id> | <container_name>

# logs
docker service logs <container_id> | <container_name>
```

#### To create 4 replicas of same service:

```bash
docker service create --replicas 4 -d alpine pint <master_ip>
```

###### Imp Cmd:

```bash
docker container ps # this will show all running container on instance or node
docker service ls  # this will show all services - only on master node

docker service ps <service_id> # this will log which replica on which node

```

#### Concept of persisting replicas:

- Now suppose any container is removed from the worker1 | worker2 | or even from the manager then amnager immediately create new one to maintain **4 replicas**
- try out by yourself - same cmd as we use in docker

#### How to scale up and scale down:

```bash
# scale | down
docker service scale <service_id>=<no of replicas>

# multiple service up & down
docker service scale <service_id>=<no of replicas> <service_id>=<no of replicas>

```

- till now we did ping wit alpine you can do whatever service you want like web , reddis , micoservices etc.

#### Port Binding:

```bash
# create nginx service
docker service create -d -p 8080:80 nginx


```

- let's suppose it is assign to worker2
  then you can access it from publicip of worker2:8080
  manager and worker1 as well
- this is the concept of overlay

#### How to run single instance or may be multiple to each instance in a cluster

- **mode**

```bash

docker create --mode=global apine <ip>
```

now suppose you create new worker then automatically one replica will run on that instance as well

###### Use Cases:

- may be any antivirus or monitoring tool that we need to run on every single instance.

#### How to create all replicas on Manager or worker only

```bash
# on manager
docker service create --replicas=3 --constraint="node.role==manager" alpine ping <ip>
# on worker
docker service create --replicas=3 --constraint="node.role==worker" alpine ping <ip>
```

##### How to create container on a single node let's say called worker1

- for that we need to create label first

```bash
docker node update --help | grep label # to find cmd
docker node update --label-add="ssd=true" worker1

# now we can create a service on worker1 only
docker create --replicas=3 --constraint="node.labels.ssd==true" -d alpine ping <ip>


```

- now if we also attach label to worker2 then does some load from worker1 shift to worker2
  ans: **No**

##### 2nd way to add label

- In worker2
- go to /etc/docker
- create a file named daemon.json
- open it in vim editor & insert data like that

```vim
{ "labels" : ["name=sanjeev"]}

```

- we can create labels in **node level** and **engine level**

#### Now to use this label we need to specify like this

```bash
docker create --replicas=3 --constraint="engine.labels.ssd==true" -d alpine ping <ip>

```

##### How to pause any node :

- from manager

```bash
docker node update --avalability=pause worker2
```

- once a worker is on pause mode then it don't receive any task from the manager.

#### reserve cpu/memory | limit cpu/memory:

```bash
docker service create --help | grep reserve
docker service create --help | grep limit
```

- for ex- if you have to create a container which can take only 300 mb memory then use limit
- and limit or reserve cpu use reserve and limit accordingly.
