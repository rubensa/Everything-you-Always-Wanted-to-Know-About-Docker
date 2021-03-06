![docker-swarm-banner](../media/swarm.png?raw=true)

# docker-swarm
*Note: before reading this you should read the talk on [docker-compose](../docker-compose/README.md), first.*

To demonstrate the concepts behind docker-swarm, we're going to create a cluster on our local machines. For all practical intents and purposes though this could be done across many physical machines.

*Note: a "cluster" is not a composition. A cluster is sort of like a composition of hosts, who are hosting potentially many compositions, or just a single container or whatever. Sort of.*

## Setup
Before we can get started, we need to actually setup our cluster. Don't worry, this is really easy though.

### Create a discovery token
First, we need to create a "discovery token". This token is used to a secure cluster, as well as enables discovery/agent registration. To do this, we're going run a container on a docker-machine that'll generate this for us.

```bash
# save the token generated by this container's output
echo "DISCOVERY_TOKEN=$(docker run swarm create)" >> ~/.bash_profile  
source ~/.bash_profile
```

### Create our swarm manager
Now we need our swarm manager. This will be in the form of a docker-machine (rather than a container).

```bash
docker-machine create \
        -d virtualbox \
        --swarm \
        --swarm-master \
        --swarm-discovery token://$DISCOVERY_TOKEN \
        master
```

We can use this docker-machine to control all of the others containers running on other machines in our cluster.

### Create some nodes to be in our swarm
With a master in place, we can add some worker nodes to our cluster now.

**Note:** This will be faster if you do it in 3 separate terminals.

```bash
# terminal 1
docker-machine create \
   -d virtualbox \
   --swarm \
   --swarm-discovery token://$DISCOVERY_TOKEN \
   worker-1
```

```bash
# terminal 2
docker-machine create \
   -d virtualbox \
   --swarm \
   --swarm-discovery token://$DISCOVERY_TOKEN \
   worker-2
```

```bash
# terminal 3
docker-machine create \
   -d virtualbox \
   --swarm \
   --swarm-discovery token://$DISCOVERY_TOKEN \
   worker-3
```

## Do stuff with swarm!
Now that our cluster in place, we can do stuff with it! So let's connect our local docker client to our pretend remote master and see some info about our cluster.

```bash
eval $(docker-machine env --swarm master)
docker info
```

This should have output *similar* to this

```bash
Containers: 5
Images: 4
Role: primary
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 4
 master: 192.168.99.110:2376
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.0.7-boot2docker, operatingsystem=Boot2Docker 1.7.1 (TCL 6.3); master : c202798 - Wed Jul 15 00:16:02 UTC 2015, provider=virtualbox, storagedriver=aufs
 worker-1: 192.168.99.111:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.0.7-boot2docker, operatingsystem=Boot2Docker 1.7.1 (TCL 6.3); master : c202798 - Wed Jul 15 00:16:02 UTC 2015, provider=virtualbox, storagedriver=aufs
 worker-2: 192.168.99.112:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.0.7-boot2docker, operatingsystem=Boot2Docker 1.7.1 (TCL 6.3); master : c202798 - Wed Jul 15 00:16:02 UTC 2015, provider=virtualbox, storagedriver=aufs
 worker-3: 192.168.99.113:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.0.7-boot2docker, operatingsystem=Boot2Docker 1.7.1 (TCL 6.3); master : c202798 - Wed Jul 15 00:16:02 UTC 2015, provider=virtualbox, storagedriver=aufs
CPUs: 4
Total Memory: 4.086 GiB
Name: 00555c88a225
```

What this enables us to do is to treat our cluster as if it were one big docker host. A sysadmin (or a piece of software, e.g. puppet or salt stack) can issue docker commands, using the standard docker client as if it were all local and docker-swarm handles the rest. Let's see an example.

```bash
docker run hello-world    # run a simple container once
docker run hello-world    # run it again
docker run hello-world    # and again...
docker ps -a              # look at all of our containers (across the swarm)
```

Basically what we did above is run a simple hello-world container 3 times. What's interesting about this is the output from `docker ps -aq`

```bash
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS                      PORTS                                     NAMES
9b62f1e8b1c0        hello-world         "/hello"               6 seconds ago        Exited (0) 5 seconds ago                                              worker-3/nostalgic_yalow
d09da7ad3f53        hello-world         "/hello"               20 seconds ago       Exited (0) 20 seconds ago                                             worker-1/kickass_turing
e5e54e9eac59        hello-world         "/hello"               25 seconds ago       Exited (0) 24 seconds ago                                             worker-2/condescending_bohr
5a4fe0cf94cb        swarm:latest        "/swarm join --adver"   About a minute ago   Up About a minute           2375/tcp                                  worker-3/swarm-agent
43ca47f7cd64        swarm:latest        "/swarm join --adver"   2 minutes ago        Up 2 minutes                2375/tcp                                  worker-2/swarm-agent
42e9399881da        swarm:latest        "/swarm join --adver"   2 minutes ago        Up 2 minutes                2375/tcp                                  worker-1/swarm-agent
e0a4a3cbc83d        swarm:latest        "/swarm join --adver"   5 minutes ago        Up 5 minutes                2375/tcp                                  master/swarm-agent
00555c88a225        swarm:latest        "/swarm manage --tls"   5 minutes ago        Up 5 minutes                2375/tcp, 192.168.99.110:3376->3376/tcp   master/swarm-agent-master
```

As we can see above, there are a couple of containers running across our swarm, so let's break it down. We can basically divide everything into two groups:

1. Swarm stuff (i.e. an agent or agent-master running on the host to do all of the orchestration magic stuff)
2. hello-world stuff

Let's just ignore the swarm stuff as that's not super interesting. What we care about is all the work we can do with these clusters. So, with that in mind, if we look at the containers that were running the hello-world image, we can see that they were automatically distributed across the swarm, by looking at the container name being prefixed with its host (e.g. `worker-3/nostalgic_yalow`).

## Fault tolerance

So if a master fails what happens? Well, swarm has a high-availability feature for that but we're not going to go into it here.
