<!--[metadata]>
+++
title = "Work with network commands"
description = "How do we connect docker containers within and across hosts ?"
keywords = ["commands, Usage, network, docker, cluster"]
[menu.engine]
parent = "smn_networking"
weight=-4
+++
<![end-metadata]-->

# Work with network commands

This article provides examples of the network subcommands you can use to interact with Docker networks and the containers in them. The commands are available through the Docker Engine CLI.  These commands are:

* `docker network create`
* `docker network connect`
* `docker network ls`
* `docker network rm`
* `docker network disconnect`
* `docker network inspect`

While not required, it is a good idea to read [Understanding Docker
network](dockernetworks.md) before trying the examples in this section. The
examples for the rely on a `bridge` network so that you can try them
immediately.  If you would prefer to experiment with an `overlay` network see
the [Getting started with multi-host networks](get-started-overlay.md) instead.

## Create networks

When you install Docker Engine it creates a `bridge` network automatically.
This network corresponds to the `docker0` bridge that Engine has traditionally
relied on. In addition to this network, you can create your own `bridge` or `overlay` network.  

A `bridge` network resides on a single host running an instance of Docker Engine.  An `overlay` network can span multiple hosts running their own engines. If you run `docker network creat`e and supply only a network name, it creates a bridge network for you.

```bash
$ docker network create simple-network
b8626a65d142e9468fc292866825c4e1eb81e81bf3ef6827f9de5de9ac10b939
$ docker network inspect simple-network
[
    {
        "name": "simple-network",
        "id": "b8626a65d142e9468fc292866825c4e1eb81e81bf3ef6827f9de5de9ac10b939",
        "scope": "local",
        "driver": "bridge",
        "ipam": {
            "driver": "default",
            "config": [
                {}
            ]
        },
        "containers": {},
        "options": {}
    }
]
```

Unlike `bridge` networks `overlay` networks require some pre-existing conditions
before you can create one. These conditions are:

* Access to a key-value store. Engine supports Consul Etcd Zookeeper (Distributed store) and BoltDB (Local store) key-value stores.
* A cluster of hosts with connectivity to the key-value store.
* A properly configured Engine `daemon` on each host in the cluster.

The `docker daemon` options that support the `overlay` network are:

* `--cluster-store`
* `--cluster-store-opt`
* `--cluster-advertise`

It is also a good idea, though not required, that you install Docker Swarm on
to manage the cluster. Swarm provides sophisticated discovery and server
management that can assist your implementation.

When you create a network, Engine creates a non-overlapping subnetwork for the network by default. You can override this default and specify a subnetwork directly using the the `--subnet` option. On a `bridge` network you can only create a single subnet. An `overlay` network supports multiple subnets.

In addition to the `--subnetwork` option, you also specify the `--gateway` `--ip-range` and `--aux-address` options.

```bash
docker network create -d overlay
  --subnet=192.168.0.0/16 --subnet=192.170.0.0/16
  --gateway=192.168.0.100 --gateway=192.170.0.100
  --ip-range=192.168.1.0/24
  --aux-address a=192.168.1.5 --aux-address b=192.168.1.6
  --aux-address a=192.170.1.5 --aux-address b=192.170.1.6
  my-multihost-newtork
```

Be sure that your subnetworks do not overlap. If they do, the network create fails and Engine returns an error.

See the ../../reference/commandline/network_

## Connect containers

Docker containers can dynamically connect to one or more networks. These
networks can be backed the same or different network drivers. Once connected,
the containers can communicate using only another container's IP address or
name.

For `overlay` networks or custom plugins that support multi-host
connectivity, containers connected to the same multi-host network but launched
from different Engines can also communicate in this way.

Create two containers for this example:

```
$ docker run -itd --name=container1 busybox
f2870c98fd504370fb86e59f32cd0753b1ac9b69b7d80566ffc7192a82b3ed27

$ docker run -itd --name=container2 busybox
bda12f8922785d1f160be70736f26c1e331ab8aaf8ed8d56728508f2e2fd4727
```

Then create a isolated, `bridge` network to test with.

```bash
$ docker network create -d bridge isolated_nw
8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7
```

Connect `container2` to the network and then `inspect` the network to verify the connection:

```
$ docker network connect isolated_nw container2
$ docker network inspect isolated_nw
{
    "name": "isolated_nw",
    "id": "8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7",
    "driver": "bridge",
    "containers":
        "777344ef4943d34827a3504a802bf15db69327d7abe4af28a05084ca7406f843": {
            "endpoint": "2ac11345af68b0750341beeda47cc4cce93bb818d8eb25e61638df7a4997cb1b",
            "mac_address": "02:42:ac:14:00:02",
            "ipv4_address": "172.20.0.2/16",
            "ipv6_address": ""
        }
    }
}
```


You can see that the Engine automatically assigns an IP address to `container2`. If you had specified a `--subnetwork` when creating your network, the network would have used that addressing. Now, start a third container and connect it to the network on launch using the `docker run` command's `--net` option:

$ docker run --net=isolated_nw -itd --name=container3 busybox
777344ef4943d34827a3504a802bf15db69327d7abe4af28a05084ca7406f843

Now inspect the network resources used by `container2` and `container3`.

```bash
$ docker inspect --format='{{.NetworkSettings.Networks}}' container2
[bridge isolated_nw]
$ docker inspect --format='{{.NetworkSettings.Networks}}' container3
[isolated_nw]
```

You should find `container2` is belongs to two networks.  The `bridge` network which it joined by default when you launched it and the `isolated_nw` which you later connected it to. In the case of container3, you connected it on launch to the `isolated_nw` so it never is connected to `bridge`.

Use the `docker attach` command to connect to the running `container2` and examine its networking stack:

```bash

$ docker attach container2

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:21 errors:0 dropped:0 overruns:0 frame:0
          TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1586 (1.5 KiB)  TX bytes:1460 (1.4 KiB)

eth1      Link encap:Ethernet  HWaddr 02:42:AC:14:00:02
          inet addr:172.20.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe14:2/64 Scope:Link
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
/ # cat /etc/hosts
172.17.0.3	8845961414ca
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	container1
172.17.0.2	container1.bridge
172.19.0.3	container3
172.19.0.3	container3.isolated_nw
```

**Note**: You can detach from the container and leave it running with `CTRL-p CTRL-q`.

In this case, `container1` is attached to both networks and so can talk to `container2` and `container3`. But `container3` and `container1` are not in the same
network and cannot communicate. Test, this now by attaching to `container3`.

```bash
$ docker attach container3

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:14:00:01
          inet addr:172.20.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe14:1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:24 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1944 (1.8 KiB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # ping -w 4 container2.isolated_nw
PING container2.isolated_nw (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.217 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.150 ms
64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.188 ms
64 bytes from 172.20.0.2: seq=3 ttl=64 time=0.176 ms
--- container2.isolated_nw ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.150/0.182/0.217 ms
/ # ping -w 2 container2
PING container2 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.120 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.109 ms
--- container2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.109/0.114/0.120 ms

/ # ping container1
ping: bad address 'container1'

/ # ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
^C
--- 172.17.0.2 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

/ # ping 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes
^C
--- 172.17.0.3 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

```

To connect a container it must be running. If you stop a container and inspect a network it belongs to, you won't see that container.  The `docker network inspect` command only shows running containers.

## Disconnecting containers

You can disconnect a container from a network using the `docker network disconnect` command.

```
$ docker network disconnect isolated_nw container2

$ docker inspect --format='{{.NetworkSettings.Networks}}' container2
[bridge]

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

Once a container is disconnected from a network, it cannot communicate with
other containers connected to that network. In this example, `container2` can no longer  talk to `container3` on the `isolated_nw` network.

```
$ docker attach container2

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:26 errors:0 dropped:0 overruns:0 frame:0
          TX packets:23 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1964 (1.9 KiB)  TX bytes:1838 (1.7 KiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # ping container3
PING container3 (172.20.0.1): 56 data bytes
^C
--- container3 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

The `container2` still has full connectivity to the bridge network

/ # ping container1
PING container1 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.119 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.174 ms
^C
--- container1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.119/0.146/0.174 ms
/ #

```

## Remove a network

When all the containers in a network stops or disconnected, you can remove a network.

```bash
$ docker network inspect isolated_nw
{
    "name": "isolated_nw",
    "id": "8b05faa32aeb43215f67678084a9c51afbdffe64cd91e3f5bb8267475f8bf1a7",
    "driver": "bridge",
    "containers": {}
}

$ docker network rm isolated_nw
```

List all your networks to verify the `isolated_nw` was removed:

```
$ docker network ls
NETWORK ID          NAME                DRIVER
9f904ee27bf5        none                null
cf03ee007fb4        host                host
7fca4eb8c647        bridge              bridge
```

## Related information

* [network create](../../reference/commandline/network_create.md)
* [network inspect](../../reference/commandline/network_inspect.md)
* [network connect](../../reference/commandline/network_connect.md)
* [network disconnect](../../reference/commandline/network_disconnect.md)
* [network ls](../../reference/commandline/network_ls.md)
* [network rm](../../reference/commandline/network_rm.md)
