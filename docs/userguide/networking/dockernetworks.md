<!--[metadata]>
+++
title = "Docker container networking"
description = "How do we connect docker containers within and across hosts ?"
keywords = ["Examples, Usage, network, docker, documentation, user guide, multihost, cluster"]
[menu.main]
parent = "smn_networking"
weight = -5
+++
<![end-metadata]-->

# Understand Docker container networks

To build web applications that act in concert but do so securely. Networks by definition provide complete isolation for containers. So, it is important to have control over the networks your applications run on. Docker container networks give you that control.

This section provides an overview of the default networking behavior that Docker Engine delivers natively.  It describes the resources required to create networks on a single host or across a cluster of hosts.

## Default Networks

When you install Docker, it creates three networks automatically. You can list these networks using the `docker network ls` command:

```
$ docker network ls
NETWORK ID          NAME                DRIVER
7fca4eb8c647        bridge              bridge
9f904ee27bf5        none                null
cf03ee007fb4        host                host
```

Historically, these three networks are  part of Docker's implementation. When you run a container you can use the `--net` flag to specify which network you want to run a container on. These three networks are still available to you.

The `bridge` network represents the `docker0` network present in all Docker installations. The Docker daemon launches each new container this network by default. You can see this bridge as part of the hosts network stack by doing an `ifconfig` on the host.

```
ubuntu@ip-172-31-36-118:~$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:47:bc:3a:eb  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:47ff:febc:3aeb/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1100 (1.1 KB)  TX bytes:648 (648.0 B)
...

The `none` network adds a container to a container-specific network stack.  That container lacks a network interface. Attaching to such a container and looking at it's stack you see this:

```
ubuntu@ip-172-31-36-118:~$ docker attach nonenetcontainer

/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ #
```
**Note**: You can detach from the container and leave it running with `CTRL-p CTRL-q`.

The `host` network adds a container on the hosts network stack. You'll find the network configuration inside the container is identical to the host.

With the exception of the the `bridge` network, you really don't need to interact with these default networks. While you can list and inspect these networks, you cannot remove them. They are required by your Docker installation.  
However, you can add to the list of networks by creating new ones yourself. Before you learn more about creating your own networks, it is worth looking at the `default` network a bit.


### The default bridge0 network in detail
The default `bridge` network is present on all Docker hosts. First, inspect the `bridge` network:

```
$ sudo docker network inspect bridge
{
    "name": "bridge",
    "id": "7fca4eb8c647e57e9d46c32714271e0c3f8bf8d17d346629e2820547b2d90039",
    "driver": "bridge",
    "containers": {}
}
```

Start two new containers running `busybox`, `container1` and `container2` respectively.

```
$ sudo docker run -itd --name=container1 busybox
f2870c98fd504370fb86e59f32cd0753b1ac9b69b7d80566ffc7192a82b3ed27

$ sudo docker run -itd --name=container2 busybox
bda12f8922785d1f160be70736f26c1e331ab8aaf8ed8d56728508f2e2fd4727
```

And then inspect the `bridge` network again. You can recognize both newly launched containers by their ids inside the network.

```
$ sudo docker network inspect bridge
{
    "name": "bridge",
    "id": "7fca4eb8c647e57e9d46c32714271e0c3f8bf8d17d346629e2820547b2d90039",
    "driver": "bridge",
    "containers": {
        "bda12f8922785d1f160be70736f26c1e331ab8aaf8ed8d56728508f2e2fd4727": {
            "endpoint": "e0ac95934f803d7e36384a2029b8d1eeb56cb88727aa2e8b7edfeebaa6dfd758",
            "mac_address": "02:42:ac:11:00:03",
            "ipv4_address": "172.17.0.3/16",
            "ipv6_address": ""
        },
        "f2870c98fd504370fb86e59f32cd0753b1ac9b69b7d80566ffc7192a82b3ed27": {
            "endpoint": "31de280881d2a774345bbfb1594159ade4ae4024ebfb1320cb74a30225f6a8ae",
            "mac_address": "02:42:ac:11:00:02",
            "ipv4_address": "172.17.0.2/16",
            "ipv6_address": ""
        }
    }
}
```
`docker network inspect` command above shows all the connected containers and its network resources on a given network

Containers in this default network are able to communicate with each other using container names.  First `attach` to your running `container` and investigate its  configuration:

```
$ sudo docker attach container1

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1382 (1.3 KiB)  TX bytes:258 (258.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Then use `ping` for about 3 seconds to test the connectivity of the containers on this `bridge` network.

```
/ # ping -w3 container2
PING container2 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.125 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.130 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.172 ms

--- container2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.125/0.142/0.172 ms
```

Finally, use the `cat` command to check the `container1` network configuration:

```

/ # cat /etc/hosts
172.17.0.2      f2870c98fd50
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      container1
172.17.0.2      container1.bridge
172.17.0.3      container2
172.17.0.3      container2.bridge
```
Exit `container1` or start a new terminal. Then, attach to `container2` and repeat these three commands.

```
$ sudo docker attach container2

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # ping container1 -w3
PING container1 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.277 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.179 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.130 ms
--- container1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.113/0.174/0.277 ms
/ # cat /etc/hosts
172.17.0.3      bda12f892278
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      container1
172.17.0.2      container1.bridge
172.17.0.3      container2
172.17.0.3      container2.bridge
/ #

```

The default `docker0` bridge network supports the use of port mapping and  `docker run --link` to allow communications between containers in the `docker0` network. These techniques are cumbersome to set up and prone to error. While they are still available to you as techniques, it is better to avoid them and define your own networks instead.

## User-defined networks

Users can create their own user-defined networks that better isolate containers.  Docker provides some default **network drivers** for use defining new networks.  You can create a new **bridge network** or **overlay network**. You can also create a **network plugin** written to your own specifications.

The next few sections describe each of these drivers in greater detail.

### A bridge network

The easiest user-defined network to create is a `bridge` network. This network is similar to the historical, default `docker0` network. There are some added features and some old features that aren't available.

```
$ docker network create -d bridge isolated_nw
8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7

$ docker network inspect isolated_nw
{
    "name": "isolated_nw",
    "id": "8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7",
    "driver": "bridge",
    "containers": {}
}

$ docker network ls
NETWORK ID          NAME                DRIVER
9f904ee27bf5        none                null
cf03ee007fb4        host                host
7fca4eb8c647        bridge              bridge
8b05faa32aeb        isolated_nw         bridge

```

After you create the network, you can launch containers on it using  the `docker run --net=<NETWORK>` option.

```
$ docker run --net=isolated_nw -itd --name=container3 busybox
777344ef4943d34827a3504a802bf15db69327d7abe4af28a05084ca7406f843

$ docker network inspect isolated_nw
{
    "name": "isolated_nw",
    "id": "8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7",
    "driver": "bridge",
    "containers": {
        "777344ef4943d34827a3504a802bf15db69327d7abe4af28a05084ca7406f843": {
            "endpoint": "c7f22f8da07fb8ecc687d08377cfcdb80b4dd8624c2a8208b1a4268985e38683",
            "mac_address": "02:42:ac:14:00:01",
            "ipv4_address": "172.20.0.1/16",
            "ipv6_address": ""
        }
    }
}
```

The containers you launch into this network must reside on the same Docker host.  Each container in the network can immediately communicate with other containers in the network. Though, the network itself isolates the containers from external networks.

![](images/bridge_network.png)

Within of a user-defined bridge network, linking is not supported. You can expose and publish container ports on containers in this network. This is useful if you want make a portion of the `bridge` network available to an outside network.

![](images/network_access.png)

A bridge network is useful in cases where you want to run a relatively small network on a single host. You can, however, create significantly larger networks by creating an `overlay` network.


### An overlay network

Docker's `overlay` network driver supports multi-host networking natively
out-of-the-box. This support is accomplished with the help of `libnetwork`, a
built-in VXLAN-based overlay network driver, and Docker's `libkv` library.

The `overlay` network requires a valid key-value store service. Currently, Docker's supports Consul, Etcd, Zookeeper (Distributed store) and
BoltDB (Local store). Before creating a network you must install and configure your chosen key-value store service.  The Docker hosts that you intend to network and the service must be able to communicate.

![](images/key_value.png)

Each host in the network must run a Docker Engine instance. The easiest way to provision the hosts are with Docker Machine.

![](images/engine_on_net.png)

Using Docker Swarm you can quickly create a cluster which includes a discovery service as well.

To create an overlay network, you configure options on  the `daemon` on each Docker Engine for use with `overlay` network. There are two options to set:

| Option                           | Description                                               |
|----------------------------------|-----------------------------------------------------------|
| `--cluster-store=PROVIDER://URL` | Describes the location of the KV service.                 |
| `--cluster-advertise=HOST_IP`    | Advertises containers created by the HOST on the network. |

Create an `overlay` network on one of the machines in the Swarm.

        $ docker network create -d overlay my-multi-host-network

This results in a single network spanning multiple hosts. The `overlay` network provides complete isolation for the containers.

![](images/overlay_network.png)

Then, on each host, launch containers making sure to specify the network name.

        $ docker run run -itd --net=mmy-multi-host-network busybox

Once launched, each container has access to all the containers in the network regardless of which Docker host the container was launched on.

![](images/overlay-network-final.png)

If you would like to try this for yourself, see the [Getting started for overlay](get-started-overlay.md).

### Docker Network Plugins

You can [extend](../../extend) Docker's networking by using a Network Plugin 
or writing your own. For more information see the [Docker Network Plugin](../../extend/plugin_network.md)

Once you have installed a Network Plugin, you use the new driver in the `docker
network create` command. For example,

    $ docker network create -d weave mynet

You can then connect your containers to this network. For example,

    $ docker run -it --net mynet busybox

## Legacy links

Before the Docker network feature, you could use the Docker link feature to allow containers to discover each other and securely transfer information
about one container to another container. With the introduction of Docker networks, you can still create links but they are only supported on the default `bridge` network named `bridge` and appearing in your network stack as `docker0`.

While links are still supported in this limited capacity, you should avoid them in preference of Docker networks. The link feature is expected to be deprecated and removed in a future release.

## Related information

- [Work with network commands](work-with-networks.md)
- [Get started with overlay networks](get-started-overlay.md)
- [Managing Data in Containers](dockervolumes.md)
- [Investigate the libnetwork project](https://github.com/docker/libnetwork/blob/master)
