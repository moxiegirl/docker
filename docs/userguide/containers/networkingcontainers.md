<!--[metadata]>
+++
title = "Networking containers"
description = "How to manage data inside your Docker containers."
keywords = ["Examples, Usage, volume, docker, documentation, user guide, data,  volumes"]
[menu.engine]
smn_containers_eng
weight = -3
+++
<![end-metadata]-->


 containers

If you are working your way through the user guide, you just built and ran a simple application. You've also built in your own images. This section teaches you now to network your containers.

## Name a container

You've already seen that each container you create has an automatically
created name; indeed you've become familiar with our old friend
`nostalgic_morse` during this guide. You can also name containers
yourself. This naming provides two useful functions:

*  You can name containers that do specific functions in a way
   that makes it easier for you to remember them, for example naming a
   container containing a web application `web`.

*  Names provide Docker with a reference point that allows it to refer to other
   containers. There are several commands that support this and you'll use one in a exercise later.

You name your container by using the `--name` flag, for example launch a new container called web:

    $ docker run -d -P --name web training/webapp python app.py

Use the `docker ps` command to see the check the name:

    $ docker ps -l
    CONTAINER ID  IMAGE                  COMMAND        CREATED       STATUS       PORTS                    NAMES
    aed84ee21bde  training/webapp:latest python app.py  12 hours ago  Up 2 seconds 0.0.0.0:49154->5000/tcp  web

You can also use `docker inspect` with the container's name.

    $ docker inspect web
    [
    {
        "Id": "3ce51710b34f5d6da95e0a340d32aa2e6cf64857fb8cdb2a6c38f7c56f448143",
        "Created": "2015-10-25T22:44:17.854367116Z",
        "Path": "python",
        "Args": [
            "app.py"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
      ...

Container names must be unique. That means you can only call one container
`web`. If you want to re-use a container name you must delete the old container
(with `docker rm`) before you can reuse the name with a new container. Go ahead and stop and them remove your `web` container.

    $ docker stop web
    web
    $ docker rm web
    web


## Launch a container on the default network

Docker includes support for networking containers through the use of **network drivers**.  By default, Docker provides two network drivers for you, the `bridge` and the `overlay` driver. You can also write a network driver plugin so that you can create your own drivers but that is an advanced task.

Every installation of the Docker Engine automatically includes three default networks. You can list them:

    $ docker network ls
    NETWORK ID          NAME                DRIVER
    18a2866682b8        none                null                
    c288470c46f6        host                host                
    7b369448dccb        bridge              bridge  

The network named `bridge` is a special network. Unless you tell it otherwise, Docker always launches your containers in this network. Try this now:

    $ docker run -itd --name=networktest ubuntu
    74695c9cea6d9810718fddadc01a727a5dd3ce6a69d09752239736c030599741

Inspecting the network is an easy way to find out the container's IP address.

    [
      {
          "name": "bridge",
          "id": "7b369448dccbf865d397c8d2be0cda7cf7edc6b0945f77d2529912ae917a0185",
          "scope": "local",
          "driver": "bridge",
          "ipam": {
              "driver": "default",
              "config": [
                  {
                      "subnet": "172.17.0.1/16",
                      "gateway": "172.17.0.1"
                  }
              ]
          },
          "containers": {
              "74695c9cea6d9810718fddadc01a727a5dd3ce6a69d09752239736c030599741": {
                  "endpoint": "3141aa83b33e60adae944d155fbd32a62e4b9eccc0893d8b34d6809877aa51dc",
                  "mac_address": "02:42:ac:11:00:02",
                  "ipv4_address": "172.17.0.2/16",
                  "ipv6_address": ""
              }
          },
          "options": {
              "com.docker.network.bridge.default_bridge": "true",
              "com.docker.network.bridge.enable_icc": "true",
              "com.docker.network.bridge.enable_ip_masquerade": "true",
              "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
              "com.docker.network.bridge.name": "docker0",
              "com.docker.network.driver.mtu": "9001"
          }
      }
    ]

You can remove a container from a network by simply disconnecting the container. To do this, you supply both the network name and the container name. You can also container id.  In this example, tho, the name is faster.

    $ docker network disconnect bridge networktest

While you can disconnect a container from a network, you cannot every remove the  builtin `bridge` network named `bridge`. Networks are natural ways to isolate containers from other containers or other networks. So, as you get more experienced with Docker, you'll want to create your own networks.

## Create your own bridge network

Docker Engine natively supports both bridge networks and overlay networks. A bridge network is limited to a single host running Docker Engine.  An overlay network can include multiple hosts and is a more advanced topic. For this example, you'll create a bridge network:  

    $ docker network create -d bridge my-bridge-network

The `-d` flag tells Docker to use the `bridge` driver for the new network. You could have left this flag off as `bridge` is the default value for this flag. Go ahead and list the networks on your machine:

    $ docker network ls
    NETWORK ID          NAME                DRIVER
    7b369448dccb        bridge              bridge              
    615d565d498c        my-bridge-network   bridge              
    18a2866682b8        none                null                
    c288470c46f6        host                host

If you inspect the network, you'll find that it has nothing in it.

    $ docker network inspect my-bridge-network
    [
      {
          "name": "my-bridge-network",
          "id": "615d565d498c771fa43f7d715119304e32d71b3486bd364406ccc981e96599fc",
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

## Add containers to a network

To build web applications that act in concert but do so securely. Networks by definition provide complete isolation for containers.  You can add containers to a network when you first launch a container.

Launch a container running a PostgreSQL database and pass it the `--net=my-bridge-network` flag to connect it to your new network:

    $ docker run -d --net=my-bridge-network --name db training/postgres

If you inspect your `my-bridge-network` you'll see it has a container attached. You can also inspect your container to see where it is connected:

    $ docker inspect --format='{{.NetworkSettings.Networks}}' db
    [my-bridge-network]

Now, go ahead and start your by now familiar web application. This time leave off the `-P` flag and also don't specify a network.

    $ docker run -d --name web training/webapp python app.py

Which network is your `web` application running under? Inspect the application and you'll find it is running in the default `bridge` network.

    $ docker inspect --format='{{.NetworkSettings.Networks}}' web
    [bridge]

Then, get the IP address of your `web`

    $ docker inspect --format='{{.NetworkSettings.IPAddress}}' web
    172.17.0.2

Now, open a shell to your running `db` container:

    $ docker exec -it db bash
    root@a205f0dd33b2:/# ping 172.17.0.2
    ping 172.17.0.2
    PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
    ^C
    --- 172.17.0.2 ping statistics ---
    44 packets transmitted, 0 received, 100% packet loss, time 43185ms

After a bit, use CTRL-C to end the `ping` and you'll find the ping failed. That is because the two container are running on different networks. You can fix that.

Docker networking allows you to attach a container to as many networks as you like. You can also attach an already running container.  Go ahead and attach your running `web` app to the `my-bridge-network`.

    $ docker network connect my-bridge-network Web

Open a shell into the `db` application again and try the ping command. This time just use the container name `web` rather than the IP Address.

    $ docker exec -it db bash
    root@a205f0dd33b2:/# ping web
    PING web (172.19.0.3) 56(84) bytes of data.
    64 bytes from web (172.19.0.3): icmp_seq=1 ttl=64 time=0.095 ms
    64 bytes from web (172.19.0.3): icmp_seq=2 ttl=64 time=0.060 ms
    64 bytes from web (172.19.0.3): icmp_seq=3 ttl=64 time=0.066 ms
    ^C
    --- web ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2000ms
    rtt min/avg/max/mdev = 0.060/0.073/0.095/0.018 ms

The `ping` shows it is contacting a different IP address, the address on the `my-bridge-network` which is different from its address on the `bridge` network.

## Next steps

Now that you know
