<!--[metadata]>
+++
title = "Get started with multi-host networking"
description = "Use overlay for multi-host networking"
keywords = ["Examples, Usage, network, docker, documentation, user guide, multihost, cluster"]
[menu.main]
parent = "smn_networking"
weight=-3
+++
<![end-metadata]-->     

# Get started with multi-host networking

This article uses an example to explain the basics of creating a mult-host
network. Docker Engine supports this out-of-the-box through the `overlay`
network.  Unlike `bridge` networks overlay networks require some pre-existing
conditions before you can create one. These conditions are:

* Access to a key-value store. Engine supports Consul, Etcd, Zookeeper (Distributed store), and BoltDB (Local store) key-value stores.
* A cluster of hosts with connectivity to the key-value store.
* A properly configured Engine `daemon` on each host in the cluster.

You'll use Docker Machine to create both the the key-value store server and the host cluster. This example creates a Swarm cluster.

## Prerequisites

Before you begin, make sure you have a system on your network with Docker Engine  and Docker Machine installed. The example also relies on Virtualbox. If you installed on a Mac or Windows with Docker Toolbox, you have all of these installed already.

If you have not already done so, make sure you upgrade Docker Engine and Docker Machine to the latests versions.


## Step 1: Set up a Consul key-store


1. Log into a system prepared with the prerequisite Docker Engine, Docker Machine, and Virtualbox software.

2. Provision a virtualbox machine called `mh-consul`.  

				$ docker-machine create -d virtualbox mh-consul

		You'll configure this machine as the key-value store that every `overlay`
		network requires. Engine supports Consul, Etcd, Zookeeper (Distributed store),
		and BoltDB (Local store) key-value stores.

		Provision adds Docker Engine to the machine. This means rather than installing
		Consul manually, you can create an instance using the [consul from Docker
		Hub](https://hub.docker.com/r/progrium/consul/).

3. Start a `progrium/consul` container running on the `mh-consul` machine.

			docker $(docker-machine config mh-consul) run -d \
				-p "8500:8500" \
				-h "consul" \
				progrium/consul -server -bootstrap

	 You passed the `docker run` command the connection configuration using a bash
	 expansion `$(docker-machine config mh-consul)`.  The client started a
	 `progrium/consul` image running in the machine. The server is called `consul`
	 and is listening port `8500`.

4. Set your local environment to the `mh-consul` machine.

			$  eval "$(docker-machine env mh-consul)"

5. Run the `docker ps` command to see the `consul` container.

			$ docker ps
			CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
			4d51392253b3        progrium/consul     "/bin/start -server -"   25 minutes ago      Up 25 minutes       53/tcp, 53/udp, 8300-8302/tcp, 0.0.0.0:8500->8500/tcp, 8400/tcp, 8301-8302/udp   admiring_panini

Keep your terminal open and move onto the next step.


## Step 2: Create a Swarm cluster

In this step, you use `docker-machine` to provision the hosts for your network. At this point, you won't actually created the network. You'll create several virtual hosts in Virtualbox. One of the host will act as th Swarm master, you'll create that first. As you create each host, you'll pass the Engine on that host options which are needed to create the `overlay` network.

1. Create a swarm master.

			docker-machine create \
			-d virtualbox \
			--swarm \
			--swarm-image="swarm" \
			--swarm-master \
			--swarm-discovery="consul://$(docker-machine ip mhs-consul):8500" \
			--engine-opt="cluster-store=consul://$(docker-machine ip mhs-consul):8500" \
			mhs-demo0

	At creation time, you supply the Engine `daemon` with the ` --cluster-store` option. This option tells the Engine which key-value store to use with the `overlay` network. The bash expansion `$(docker-machine ip mhs-consul)` resolves to the IP address of the Consul server you created in "STEP 1".

2. Create another host and add it to the Swarm cluster.

		docker-machine create \
			-d virtualbox \
			--swarm \
			--swarm-image="swarm:1.0.0" \
			--swarm-discovery="consul://$(docker-machine ip mhs-consul):8500" \
			--engine-opt="cluster-store=consul://$(docker-machine ip mhs-consul):8500" \
		        mhs-demo1

3. List your machines to confirm they are all up and running.

		$ docker-machine ls
		NAME         ACTIVE   DRIVER       STATE     URL                         SWARM
		default               virtualbox   Running   tcp://192.168.99.100:2376   
		mhs-consul            virtualbox   Running   tcp://192.168.99.103:2376   
		mhs-demo0             virtualbox   Running   tcp://192.168.99.104:2376   mhs-demo0 (master)
		mhs-demo1             virtualbox   Running   tcp://192.168.99.105:2376   mhs-demo0

Leave your terimanl open and go onto the next STEP.

## STEP 3: Create the overlay Network

To create an overlay network

1. Set your docker environment to the Swarm master.

			$ eval $(docker-machine env --swarm mhs-demo0)

2. Use the `docker info` command to view the Swarm.

			$ docker info
			Containers: 3
			Images: 2
			Role: primary
			Strategy: spread
			Filters: affinity, health, constraint, port, dependency
			Nodes: 2
			mhs-demo0: 192.168.99.104:2376
			└ Containers: 2
			└ Reserved CPUs: 0 / 1
			└ Reserved Memory: 0 B / 1.021 GiB
			└ Labels: executiondriver=native-0.2, kernelversion=4.1.10-boot2docker, operatingsystem=Boot2Docker 1.9.0-rc1 (TCL 6.4); master : 4187d2c - Wed Oct 14 14:00:28 UTC 2015, provider=virtualbox, storagedriver=aufs
			mhs-demo1: 192.168.99.105:2376
			└ Containers: 1
			└ Reserved CPUs: 0 / 1
			└ Reserved Memory: 0 B / 1.021 GiB
			└ Labels: executiondriver=native-0.2, kernelversion=4.1.10-boot2docker, operatingsystem=Boot2Docker 1.9.0-rc1 (TCL 6.4); master : 4187d2c - Wed Oct 14 14:00:28 UTC 2015, provider=virtualbox, storagedriver=aufs
			CPUs: 2
			Total Memory: 2.043 GiB
			Name: 30438ece0915

3. Create your `overlay` network.

			$ docker network create -d overlay my-net

		You only need to create the network on a single host in the cluster. In this case, you used the Swarm master but you could easily have run it on any host in the cluster.

4. Check that the network is running on both hosts:

			$ docker network ls
			$ eval $(docker-machine env mhs-demo1)
			$ docker network ls


##  Run an application on your Network

Once your network is created, you can start a container on any of the hosts and it automatically is part of the network.

1. Start an Nginx server on `mhs-demo0`.

		$ docker run -itd --name=web --net=my-net --env="constraint:node==mhs-demo0" nginx

2. Start a Busybox web server on `msh-demo1`

		$ docker run -it --rm --net=my-net --env="constraint:node==mhs-demo1" busybox wget -O- http://web

## Related information

* [Docker Swarm overview](https://docs.docker.com/swarm)
* [Docker Machine overview](https://docs.docker.com/machine)
